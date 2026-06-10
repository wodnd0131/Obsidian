## `@ElementCollection`

`Set<UserRole>` 같은 **컬렉션을 별도 테이블로 분리**하는 어노테이션.

`@OneToMany`는 반대쪽에 `@Entity`가 있어야 하는데, `UserRole` 같은 단순 enum/값 타입은 엔티티가 아니라서 `@ElementCollection` 씀.

```sql
-- user 테이블
user_id | email | ...

-- user_roles 테이블 (자동 생성)
user_id | roles
1       | USER
1       | ADMIN
```

---

## `FetchType.EAGER`

`User` 조회할 때 `user_roles`도 **즉시 JOIN해서 같이 가져옴**.

```sql
SELECT u.*, ur.roles
FROM user u
LEFT JOIN user_roles ur ON u.id = ur.user_id
WHERE u.id = 1
```

LAZY면 `user.getRoles()` 호출 시점에 쿼리 추가 발생 — JWT에 roles 담으려면 로그인 시점에 바로 필요하니까 EAGER가 맞아.

---

## `@Enumerated(EnumType.STRING)`

```java
enum UserRole { USER, ADMIN }
```

|EnumType|DB 저장값|위험성|
|---|---|---|
|ORDINAL (기본)|0, 1|enum 순서 바뀌면 데이터 깨짐|
|STRING|"USER", "ADMIN"|순서 변경 무관, 안전|

---

## `@CollectionTable`

자동 생성되는 테이블 이름/FK 컬럼명 명시적으로 지정.

```java
@CollectionTable(name = "user_roles", joinColumns = @JoinColumn(name = "user_id"))
//               ↑ 테이블명 지정       ↑ FK 컬럼명 지정
```

안 쓰면 JPA가 `user_roles`, `user_id` 같은 이름으로 자동 생성하긴 하는데, 명시하는 게 나중에 헷갈리지 않아서 좋음.