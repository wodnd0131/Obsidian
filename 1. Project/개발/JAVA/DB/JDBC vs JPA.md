
# 한 줄 정리

**JDBC** — Java에서 SQL을 직접 DB에 보내는 저수준 API 
**JPA** — 객체와 테이블을 매핑해서 SQL을 자동 생성해주는 ORM 명세

---

## 같은 작업, 두 방식 비교

```java
// JDBC — SQL을 직접 작성
Connection conn = dataSource.getConnection();
PreparedStatement ps = conn.prepareStatement(
    "SELECT * FROM product WHERE id = ?"
);
ps.setLong(1, productId);
ResultSet rs = ps.executeQuery();

if (rs.next()) {
    Product p = new Product();
    p.setId(rs.getLong("id"));
    p.setName(rs.getString("name"));
    p.setStock(rs.getInt("stock"));
}
// 연결 닫기, 예외 처리 등 직접 관리
```

```java
// JPA — SQL 없음, 객체로 바로
Product product = em.find(Product.class, productId);
// 끝. JPA가 SELECT * FROM product WHERE id=? 생성해서 실행
```

---

## 계층 관계

```
[ 내 코드 ]
    ↓
[ JPA / Hibernate ]   ← SQL 자동 생성, 객체-테이블 매핑
    ↓
[ JDBC ]              ← 실제 SQL을 DB에 전송
    ↓
[ DB ]
```

JPA가 JDBC를 대체하는 게 아니다. **JPA는 내부적으로 JDBC를 쓴다.** 추상화 레이어가 하나 더 얹힌 것.

---

## 각각 언제 쓰나

**JPA가 유리한 경우**

- 단순 CRUD, 연관관계 탐색이 많은 비즈니스 로직
- 객체 중심으로 설계할 때

**JDBC(또는 MyBatis)가 유리한 경우**

- 복잡한 통계 쿼리, 대용량 배치
- JPA로 표현하기 어려운 SQL (윈도우 함수, 다중 조인 등)
- 쿼리 튜닝이 중요한 상황 — JPA가 생성하는 SQL을 믿기 어려울 때

실무에서는 JPA + JPQL/Native Query를 기본으로 쓰고, 복잡한 조회는 QueryDSL이나 직접 JDBC로 내려가는 방식을 혼용하는 경우가 많다.

---

이어서 깊게 볼 게 있다면 **JPA의 영속성 컨텍스트(1차 캐시, dirty checking, flush 타이밍)** 가 실무에서 가장 자주 문제가 되는 지점이다. 앞서 나온 낙관적 락의 version 체크 타이밍도 거기서 결정된다.