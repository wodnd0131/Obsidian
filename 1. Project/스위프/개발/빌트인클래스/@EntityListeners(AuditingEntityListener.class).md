JPA Auditing 기능이야. 엔티티의 생성/수정 시간, 생성자/수정자를 **자동으로 채워주는** 리스너.

---

### 동작 방식

```java
@EntityListeners(AuditingEntityListener.class)
@Entity
public class User {

    @CreatedDate
    private Instant createdAt;

    @LastModifiedDate
    private Instant updatedAt;
}
```

`save()` 호출할 때 자동으로 값 넣어줌. 직접 `createdAt = Instant.now()` 안 해도 됨.

---

### 활성화 조건

`@EnableJpaAuditing` 을 설정 클래스에 붙여야 작동함:

```java
@EnableJpaAuditing
@SpringBootApplication
public class Application { ... }
```

이거 빠지면 `@CreatedDate` 있어도 null로 들어감.

---

### 자주 쓰는 어노테이션 4개

|어노테이션|역할|
|---|---|
|`@CreatedDate`|최초 저장 시각|
|`@LastModifiedDate`|마지막 수정 시각|
|`@CreatedBy`|생성자 (별도 설정 필요)|
|`@LastModifiedBy`|수정자 (별도 설정 필요)|

`@CreatedBy` / `@LastModifiedBy`는 `AuditorAware` 빈 등록이 추가로 필요해.