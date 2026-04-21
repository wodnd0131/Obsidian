#OOP/Entitiy #Status/보충 
# Entity는 가변적인가?

**1. Problem First**

"엔티티는 가변이 자연스럽다"는 주장의 근거는 보통 이거다:

> 도메인 객체는 상태가 바뀐다. 주문은 `PENDING → CONFIRMED`로 바뀌고, 계좌 잔액은 증가하고 감소한다. 이걸 불변으로 모델링하면 오히려 부자연스럽지 않냐?

그런데 이 주장이 전제하는 게 있다: **"상태 변화 = 필드 변경"**. 이게 틀린 전제다.

```java
// 안티패턴: 가변 엔티티, setter 열어둠
public class Order {
    private OrderStatus status;
    private List<OrderItem> items;

    public void setStatus(OrderStatus status) {
        this.status = status; // 외부에서 아무 상태나 꽂을 수 있다
    }

    public List<OrderItem> getItems() {
        return items; // 참조 노출 → 외부에서 items.add(...) 가능
    }
}

// 어디선가
order.setStatus(OrderStatus.CANCELLED); // 도메인 규칙 없이 상태 전환
order.getItems().clear();               // 엔티티가 모르는 사이 내부 변경
```

이 코드의 문제는 "가변"이 아니라 **"제어되지 않는 가변"** 이다.

---

**2. Mechanics — 무엇이 "엔티티"인가**

DDD 맥락에서 엔티티의 정의부터 정확히 짚어야 한다.

> _"An object primarily defined by its identity is called an Entity."_ — Eric Evans, _Domain-Driven Design_, p. 89

엔티티의 본질은 **동일성(identity)** 이다. `id`가 같으면 필드 값이 달라도 같은 객체다. 이건 가변/불변과 **직교하는 개념**이다.

VO(Value Object)와의 차이:

| |Entity|Value Object|
|---|---|---|
|동등성 기준|identity (id)|모든 필드값|
|생명주기|있음|없음|
|가변성|규칙에 따라 허용|불변 권장|

JVM 레벨에서 보면:

- 엔티티는 힙에 올라간 객체다. 필드를 바꿔도 참조(주소)는 같다.
- JPA 같은 ORM이 엔티티를 가변으로 다루는 이유는 **더티 체킹(dirty checking)** 때문이다 — 필드가 바뀌어야 `UPDATE` 쿼리를 생성한다. 이건 **인프라 제약**이지 도메인 설계 원칙이 아니다.

---

**3. 공식 근거**

**Eric Evans (DDD)**:

> _"The key is to keep the Entity focused on its identity and the integrity of its lifecycle."_ — _Domain-Driven Design_, p. 92
> 
> (엔티티의 핵심은 동일성과 생명주기 무결성에 집중하는 것이다.)

가변을 허용하되, **도메인이 그 변화를 통제해야 한다**는 게 Evans의 입장이다.

**Vaughn Vernon (IDDD)**:

> _"Favor immutability where possible. When an Entity must change, model the change explicitly as a behavior."_ — _Implementing Domain-Driven Design_, p. 228
> 
> (가능한 한 불변을 선호하라. 엔티티가 변해야 한다면, 그 변화를 행위로 명시적으로 모델링하라.)

**Effective Java Item 17 (Bloch)**:

> _"Classes should be immutable unless there's a very good reason to make them mutable."_
> 
> (매우 타당한 이유가 없다면 클래스는 불변이어야 한다.)

Bloch는 Entity를 직접 언급하진 않지만, 이 원칙을 도메인 객체에 적용하면 — 기본값은 불변, 가변은 **정당화가 필요한 예외**.

이 부분은 공식 자료라기보다 커뮤니티 관행 수준이지만: Spring 공식 문서는 JPA 엔티티에 setter를 쓰지 말고 **의미있는 비즈니스 메서드**를 쓰라고 권장한다.

---

**4. 트레이드오프**

**가변 엔티티 (제어된)**

- ✅ JPA 더티 체킹과 자연스럽게 맞음
- ✅ 상태 전환 표현이 직관적
- ❌ 불변식(invariant) 보호가 설계 규율에 달림 — 누군가 setter 하나 추가하면 무너짐
- ❌ 멀티스레드 환경에서 외부 동기화 필요

**불변 엔티티**

- ✅ 불변식이 타입 시스템으로 강제됨
- ✅ 스레드 안전, 부수효과 없음
- ❌ JPA와 충돌 — 프록시 생성, 더티 체킹 모두 가변을 전제
- ❌ 상태 전환마다 새 객체 생성 → 영속성 컨텍스트와 정합성 유지가 복잡해짐

현실적으로: **순수 도메인 레이어에서는 불변 지향, 인프라(JPA)와 맞닿는 레이어에서는 제어된 가변** 이 트레이드오프를 가장 잘 다루는 구조다.

---

**5. 안티패턴 검증**

**개념 자체의 오남용 — "엔티티는 가변이니까 setter 전부 열어도 된다"**

```java
// 안티패턴
public class Account {
    private Money balance;
    public void setBalance(Money balance) { this.balance = balance; }
}

// 올바른 방향
public class Account {
    private Money balance;

    public void deposit(Money amount) {
        if (amount.isNegative()) throw new IllegalArgumentException(...);
        this.balance = this.balance.add(amount);
    }

    public void withdraw(Money amount) {
        if (amount.isGreaterThan(balance)) throw new InsufficientFundsException(...);
        this.balance = this.balance.subtract(amount);
    }
}
```

`setBalance`는 도메인 규칙을 우회한다. `deposit`/`withdraw`는 규칙을 강제한다. 둘 다 가변이지만 전자는 안티패턴이다.

**잘못된 적용 방식 — 불변 강제로 JPA 쓰는 경우**

```java
// 현실에서 무너지는 패턴
@Entity
public class Order {
    private final Long id;
    private final OrderStatus status;
    // JPA는 기본 생성자 + 필드 변경 전제 → 이 구조는 프록시/더티체킹 전부 깨짐
}
```

이건 "불변이 좋다"는 원칙을 인프라 제약 무시하고 적용한 잘못된 구현이다.

**논쟁: 엔티티의 불변성**

- **불변 지지측 (Vernon, 일부 함수형 DDD)**: 이벤트 소싱과 결합하면 엔티티 자체를 불변으로 유지할 수 있다. 상태 전환은 새 이벤트를 append하는 것이지 필드를 수정하는 게 아니다.
- **실용주의 측 (대부분의 JPA 기반 팀)**: 인프라 제약을 무시한 순수주의는 실용성이 없다. 가변을 인정하되 setter 대신 도메인 행위로 통제하면 충분하다.

어느 쪽도 **"setter 열어두는 가변"은 틀렸다**는 데는 동의한다.

---

**6. 이어지는 개념**

이 논쟁의 뿌리와 확장:

```
엔티티 가변성
    ├── Tell, Don't Ask          ← setter 대신 행위 메서드로 가는 원칙
    ├── 도메인 불변식(Invariant)  ← 엔티티가 보호해야 할 규칙
    ├── Aggregate Root           ← 엔티티 가변성의 진짜 제어 단위
    ├── 이벤트 소싱              ← 엔티티 불변화의 극단적 해법
    └── JPA vs 순수 도메인 분리  ← 인프라 제약과 도메인 원칙의 충돌 지점
```

다음에 팔 곳은 **Aggregate Root**다. "엔티티는 가변이어도 된다"는 말이 성립하려면, 그 가변성을 **Aggregate Root가 통제**한다는 전제가 있어야 한다. 그게 빠진 가변 엔티티 논의는 절반짜리다.