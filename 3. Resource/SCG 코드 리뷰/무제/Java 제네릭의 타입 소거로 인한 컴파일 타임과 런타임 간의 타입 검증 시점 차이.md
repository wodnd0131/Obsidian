

기존 ReservationDao.save(ReservationRequest)가 하던 "DTO를 받아 Time/Theme을 조회하고 Reservation을 구성해 저장한다"는 책임을, 이름만 같은 save 메서드로 JpaRepository에 그대로 옮기려 하신 것으로 보입니다.

하지만, 다음 에러가 발생합니다.

org.springframework.dao.InvalidDataAccessApiUsageException: roomescape.reservation.DTO.ReservationRequest is not an entity

이는 save() 호출에서 ReservationRequest이라는 DTO 대신
JpaRepository<Reservation, Long> 으로 지정한 Reservation 엔티티를 파라미터로 사용하면 해결됩니다.

 public Reservation save(Reservation newReservation);

Spring Data JPA는 JpaRepository<T, ID>의 제네릭 타입 T를 기준으로 작동하기 때문에, 상속받은 메서드들도 모두 이 타입을 파라미터로 사용해야 합니다.

여기서 컴파일도 되고 앱도 정상 기동되고 있지만, 실제 예약 생성 요청이 들어올 때만 에러가 터지는 이유가 뭘까요?

이는 후술한 제너릭 타입과 관련된 이유입니다.
하지만, 이는 참고만 하면 좋을 것 같아요.

현 미션에서는 T가 엔티티에 매핑되고, 매핑된 값을 기준으로 기본 메서드를 활용 해야한다 정도로 충분합니다.😄

---
런타임 예외만 발생하는 이유는, 
JpaRepository가 제너릭 타입(`T`)을 통해 엔티티 형태를 지정받고, 이를 기준으로 작동하기 때문입니다.

`JpaRepository<Reservation, Long>` 는 `CrudRepository<T, ID>` 라는 부모 인터페이스를 가지고,
`<S extends T> S save(S entity)`라는 기본 save 메서드를 가지고있습니다.

**CrudRepository에서 상속받은 save 메서드**
```java
public interface CrudRepository<T, ID> {
    <S extends T> S save(S entity);
}
```
> T는 엔티티, `<S extends T>`의 s는 T 이하 자식 클래스

`JpaRepository<Reservation, Long>`는 이 `CrudRepository`를 상속받으므로, 
`T = Reservation`이 대입되어 다음과 같이 해석됩니다:
```java
public Reservation save(Reservation entity);
```

그런데 Java 제네릭은 컴파일 후 **메서드 시그니처에서만** 타입 정보가 사라집니다(타입 소거).
따라서 바이트코드상 메서드는 다음과 같이 표현됩니다.

```java
public Object save(Object entity);
```

**하지만** Spring Data는 애플리케이션 시작 시 리플렉션으로 `JpaRepository<Reservation, Long>`의 제네릭 정보를 읽어내고, "`save()`는 `Reservation`을 받아야 한다"는 걸 미리 알고 구현체를 생성합니다.

따라서 **실제 호출 시점**에 `ReservationRequest`가 전달되면, 기대하던 타입과 맞지 않으므로 예외가 발생합니다.

쉽게 플로우로 보자면,

`public Reservation save(ReservationRequest newReservation);`
위 코드를 컴파일 이후 다음과 같이 인식합니다.
-> `public Object save(Object newReservation);`

하지만 Spring Data는 `ReservationRepository`를 분석할 때 이미 
리플렉션으로 `JpaRepository<Reservation, Long>`에서 `T = Reservation`을 파악해뒀습니다.

따라서 Spring Data의 구현체는
"이 save 메서드는 Reservation을 받아야 한다"고 미리 알고 있습니다.

그런데 실제로 호출할 때 ReservationRequest가 들어오니까?

=> 실제 실행 시 InvalidDataAccessApiUsageException 발생

때문에, 실행되어 값이 들어오기 전까지는 예외로 인식하지 못합니다.