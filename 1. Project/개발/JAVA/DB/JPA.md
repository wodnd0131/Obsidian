# JPA — 내부 동작과 설계 근거

---

## 1. Problem First

### 시나리오 A: JDBC 직접 사용 시대

```java
public Order findOrderWithItems(Long orderId) {
    Connection conn = dataSource.getConnection();
    
    // Order 조회
    PreparedStatement orderStmt = conn.prepareStatement(
        "SELECT * FROM orders WHERE id = ?"
    );
    orderStmt.setLong(1, orderId);
    ResultSet orderRs = orderStmt.executeQuery();
    
    Order order = new Order();
    if (orderRs.next()) {
        order.setId(orderRs.getLong("id"));
        order.setStatus(orderRs.getString("status"));
        // 컬럼 하나 추가될 때마다 여기도 수정
    }
    
    // Item 조회 (별도 쿼리)
    PreparedStatement itemStmt = conn.prepareStatement(
        "SELECT * FROM order_items WHERE order_id = ?"
    );
    // ... 반복
    
    // 연결 닫기 (예외 나면 누수)
    conn.close();
}
```

**문제의 본질:**

- 객체 모델과 테이블 모델 간 변환 코드를 전부 수동으로 작성
- `Order`에 필드 하나 추가 = SQL + 매핑 코드 + 모든 관련 쿼리 수정
- 트랜잭션, 연결 관리, 예외 처리가 비즈니스 로직과 뒤섞임
- **이것이 Martin Fowler가 명명한 "Object-Relational Impedance Mismatch"다**

> 📎 **근거:** Martin Fowler — _"OrmHate"_, martinfowler.com/bliki/OrmHate.html

---

### 시나리오 B: JPA를 쓰면서 N+1이 터진 경우

```java
// 주문 목록 조회
List<Order> orders = orderRepository.findAll(); // 쿼리 1번

for (Order order : orders) {
    // order.getItems()가 LAZY라면 여기서 쿼리가 터진다
    System.out.println(order.getItems().size()); // 주문 수만큼 쿼리 발생
}

// 주문이 100개면 → 쿼리 101번
// 주문이 10,000개면 → 쿼리 10,001번
```

JPA를 쓴다고 성능 문제가 사라지지 않는다. **내부 동작을 모르면 JDBC보다 더 심각한 문제를 만든다.**

---

### 시나리오 C: 트랜잭션 경계를 잘못 이해한 경우

```java
@Service
public class OrderService {

    public void processOrder(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        order.setStatus(OrderStatus.PROCESSING); // 변경
        // save() 호출 없음
        // 트랜잭션도 없음
        // → DB 반영 안 됨. 조용히 사라짐.
    }
}
```

JPA의 Dirty Checking은 **트랜잭션 안에서만** 작동한다. 트랜잭션 없이 엔티티를 변경하면 아무 일도 일어나지 않는다.

---

## 2. Mechanics

### 2-1. JPA 전체 구조 — 계층별 역할

```
┌──────────────────────────────────────┐
│         애플리케이션 코드              │
├──────────────────────────────────────┤
│     Spring Data JPA Repository       │  ← 쿼리 메서드 자동 생성
├──────────────────────────────────────┤
│      JPA (javax.persistence API)     │  ← 표준 명세 (JSR-338)
├──────────────────────────────────────┤
│    Hibernate (JPA 구현체)             │  ← 실제 동작
│  ┌─────────────────────────────────┐ │
│  │      SessionFactory             │ │  ← 애플리케이션 생명주기
│  │  ┌──────────────────────────┐   │ │
│  │  │   Session (EntityManager) │   │ │  ← 요청 생명주기
│  │  │   ┌──────────────────┐   │   │ │
│  │  │   │  1차 캐시         │   │   │ │  ← 핵심 동작의 근거
│  │  │   │  (영속성 컨텍스트) │   │   │ │
│  │  │   └──────────────────┘   │   │ │
│  │  └──────────────────────────┘   │ │
│  └─────────────────────────────────┘ │
├──────────────────────────────────────┤
│         JDBC                         │
├──────────────────────────────────────┤
│         DB                           │
└──────────────────────────────────────┘
```

---

### 2-2. 영속성 컨텍스트 — JPA 모든 동작의 근거

영속성 컨텍스트(Persistence Context)는 `EntityManager`가 관리하는 **엔티티 인스턴스의 1차 캐시**다.

```
영속성 컨텍스트 내부 구조:

Map<EntityKey, Object> firstLevelCache
    EntityKey = (클래스타입, PK값)
    
Map<EntityKey, Object[]> snapshotCache  ← Dirty Checking용 원본 스냅샷
```

**엔티티의 4가지 상태:**

```
[비영속 - New]
Order order = new Order();  // JPA가 전혀 모름

      em.persist(order)
            ↓
[영속 - Managed]             // 1차 캐시에 있음, 변경 추적됨
      
      em.detach(order)       트랜잭션 종료
      또는 em.close()   ←────────────────────
            ↓
[준영속 - Detached]          // 1차 캐시에서 제거, 변경 추적 안 됨

      em.remove(order)
            ↓
[삭제 - Removed]             // DELETE 예약됨
```

**`em.find()` 호출 시 실제 흐름:**

```java
Order order1 = em.find(Order.class, 1L);
// → 1차 캐시 확인: 없음 → SQL SELECT 실행 → 결과를 1차 캐시에 저장

Order order2 = em.find(Order.class, 1L);
// → 1차 캐시 확인: 있음 → SQL 없이 캐시에서 반환

System.out.println(order1 == order2); // true — 같은 인스턴스
```

**같은 트랜잭션 안에서 같은 PK로 조회하면 항상 동일 인스턴스를 반환한다.** 이것이 JPA의 **동일성 보장**이다.

> 📎 **근거:** JPA 2.2 명세 (JSR-338) §3.1 — _"EntityManager"_, §3.2 — _"Persistence Context Lifetime"_

---

### 2-3. Dirty Checking 내부 동작

```java
@Transactional
public void updateOrder(Long orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    // 이 시점에 스냅샷 저장:
    // snapshotCache[Order#1] = ["PENDING", 99.99, ...]
    
    order.setStatus(OrderStatus.COMPLETED); // 변경만 함, save() 없음
    
    // 트랜잭션 커밋 직전 flush 발생:
    // 현재 상태: ["COMPLETED", 99.99, ...]
    // 스냅샷:    ["PENDING",   99.99, ...]
    // → 다르다 → UPDATE SQL 자동 생성
}
```

**flush 시점의 정확한 순서:**

```
트랜잭션 커밋
    ↓
EntityManager.flush() 호출
    ↓
영속성 컨텍스트의 모든 Managed 엔티티 순회
    ↓
현재 상태 vs 스냅샷 필드별 비교
    ↓
차이 있음 → UPDATE SQL 생성 → JDBC 배치에 추가
    ↓
INSERT/UPDATE/DELETE 순서로 SQL 실행
    ↓
DB 커밋
```

**Dirty Checking이 전체 필드를 UPDATE하는 이유:**

```sql
-- JPA 기본 동작
UPDATE orders
SET status = ?, total_price = ?, customer_id = ?, created_at = ?
WHERE id = ?
-- status만 바꿨어도 모든 컬럼을 UPDATE

-- 이유: PreparedStatement를 재사용하기 위해
-- 같은 엔티티 타입은 항상 동일한 SQL 구조 → 파싱/실행계획 재사용
```

**변경된 필드만 UPDATE하려면:**

```java
@Entity
@DynamicUpdate // Hibernate 전용 — 변경된 컬럼만 UPDATE
public class Order { ... }

// 주의: 실행계획 재사용 불가 → 고트래픽 환경에서 오히려 역효과 가능
```

> 📎 **근거:** Hibernate 공식 문서 §3.5 — _"Flushing"_, Hibernate `@DynamicUpdate` Javadoc

---

### 2-4. N+1 문제 — 발생 원인과 해결

**왜 N+1이 발생하는가:**

```java
@Entity
public class Order {
    @OneToMany(fetch = FetchType.LAZY) // 기본값
    private List<OrderItem> items;
}

// JPQL은 SELECT o FROM Order o
// → Hibernate가 생성하는 SQL:
// SELECT * FROM orders                    ← 1번
// SELECT * FROM order_items WHERE order_id = 1  ← N번
// SELECT * FROM order_items WHERE order_id = 2
// ...
```

**LAZY 로딩의 내부 동작:**

```java
Order order = em.find(Order.class, 1L);
// items 필드: 실제 List가 아닌 HibernateProxy 객체가 들어있음

List<OrderItem> items = order.getItems();
// 아직 SQL 안 나감 — proxy 반환

items.size(); // 또는 items.get(0)
// 이 시점에 SELECT * FROM order_items WHERE order_id = 1 실행
// → 프록시가 실제 데이터로 초기화됨
```

**해결책 1 — Fetch Join:**

```java
// JPQL
@Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items WHERE o.id IN :ids")
List<Order> findWithItems(@Param("ids") List<Long> ids);

// 생성되는 SQL:
// SELECT DISTINCT o.*, i.*
// FROM orders o
// INNER JOIN order_items i ON i.order_id = o.id
// WHERE o.id IN (?, ?, ...)
// → 쿼리 1번으로 해결
```

**해결책 2 — `@EntityGraph`:**

```java
@EntityGraph(attributePaths = {"items", "items.product"})
@Query("SELECT o FROM Order o WHERE o.customerId = :customerId")
List<Order> findByCustomerIdWithItems(@Param("customerId") Long customerId);
```

**해결책 3 — `@BatchSize`:**

```java
@Entity
public class Order {
    @OneToMany
    @BatchSize(size = 100) // IN 절로 묶어서 조회
    private List<OrderItem> items;
}

// 생성되는 SQL:
// SELECT * FROM order_items WHERE order_id IN (1, 2, 3, ..., 100)
// N+1 → N/100 + 1 로 감소
```

**Fetch Join의 한계 — 페이지네이션과 함께 쓸 수 없다:**

```java
// DANGEROUS
@Query("SELECT o FROM Order o JOIN FETCH o.items")
Page<Order> findAll(Pageable pageable);

// Hibernate 경고:
// HHH90003004: firstResult/maxResults specified with collection fetch;
// applying in memory
// → 전체 데이터를 메모리에 올린 뒤 페이지네이션 → OOM 가능
```

**컬렉션 Fetch Join + 페이지네이션 해결책:**

```java
// 1단계: ID만 페이지네이션으로 조회
@Query("SELECT o.id FROM Order o")
Page<Long> findAllIds(Pageable pageable);

// 2단계: ID로 Fetch Join
@Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items WHERE o.id IN :ids")
List<Order> findWithItemsByIds(@Param("ids") List<Long> ids);
```

> 📎 **근거:** Hibernate 공식 문서 §15.3 — _"Fetching strategies"_, Hibernate HHH90003004 경고 공식 설명

---

### 2-5. 트랜잭션과 영속성 컨텍스트의 관계

**Spring에서의 기본 동작:**

```
@Transactional 메서드 시작
    ↓
EntityManager 생성 (영속성 컨텍스트 시작)
    ↓
메서드 실행 (엔티티 조회, 변경)
    ↓
메서드 종료
    ↓
flush() → Dirty Checking → SQL 실행
    ↓
commit()
    ↓
EntityManager 종료 (영속성 컨텍스트 소멸)
    ↓
엔티티는 Detached 상태가 됨
```

**`@Transactional` 없이 Repository를 호출하면:**

```java
// Repository 메서드 자체에 @Transactional이 있어서 실행은 됨
Order order = orderRepository.findById(1L).orElseThrow();
// findById 트랜잭션 종료 → order는 즉시 Detached

order.setStatus(COMPLETED);
// Detached 상태 변경 → Dirty Checking 대상 아님

orderRepository.save(order);
// save()는 merge()를 호출 → 새 영속 엔티티로 복사 → UPDATE 실행
// 작동은 하지만, 의도와 다른 경로
```

**OSIV(Open Session In View) — Spring Boot 기본값이 `true`인 문제:**

```
OSIV = true (기본값):
HTTP 요청 시작 → EntityManager 열림
                      ↓
              Controller, Service, Repository 전부
              같은 영속성 컨텍스트 공유
                      ↓
HTTP 응답 완료 → EntityManager 닫힘

문제:
- Controller에서 LAZY 로딩 가능 → 편하지만
- DB 커넥션을 HTTP 요청 전체 동안 점유
- 트래픽 높으면 커넥션 풀 고갈
```

```yaml
# 권장 설정
spring:
  jpa:
    open-in-view: false
```

> 📎 **근거:** Spring Boot 공식 레퍼런스 — _"Open EntityManager in View"_, Vlad Mihalcea — _"The Open Session In View Anti-Pattern"_, vladmihalcea.com

---

### 2-6. Spring Data JPA — 쿼리 메서드 생성 원리

```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByCustomerIdAndStatusOrderByCreatedAtDesc(
        Long customerId, OrderStatus status
    );
}
```

**Spring이 이 메서드를 파싱하는 과정:**

```
메서드명 파싱:
find     By    CustomerId  And   Status   OrderBy  CreatedAt  Desc
 ↓        ↓        ↓        ↓      ↓         ↓         ↓        ↓
조회   조건시작  customer_id  AND  status   정렬기준  created_at  내림차순

→ 생성되는 JPQL:
SELECT o FROM Order o
WHERE o.customerId = :customerId
AND o.status = :status
ORDER BY o.createdAt DESC
```

**쿼리 메서드 생성의 한계:**

```java
// 이 정도 넘어가면 메서드명이 읽기 불가능해짐
List<Order> findByCustomerIdAndStatusInAndCreatedAtBetweenAndTotalPriceGreaterThan(
    Long customerId,
    List<OrderStatus> statuses,
    LocalDateTime from,
    LocalDateTime to,
    BigDecimal minPrice
);

// 이때는 @Query 사용
@Query("""
    SELECT o FROM Order o
    WHERE o.customerId = :customerId
    AND o.status IN :statuses
    AND o.createdAt BETWEEN :from AND :to
    AND o.totalPrice > :minPrice
    """)
List<Order> findByComplexCondition(...);
```

> 📎 **근거:** Spring Data JPA 공식 레퍼런스 — _"Query Creation"_, docs.spring.io/spring-data/jpa/docs/current/reference/html

---

## 3. 공식 근거 정리

|주장|출처|
|---|---|
|영속성 컨텍스트, 엔티티 상태|JPA 2.2 명세 (JSR-338) §3.1, §3.2|
|flush 동작 및 순서|Hibernate 공식 문서 §3.5|
|Dirty Checking 전체 컬럼 UPDATE 이유|Hibernate 공식 문서 §3.5.1|
|컬렉션 Fetch Join + 페이지네이션 경고|Hibernate HHH90003004 공식 설명|
|OSIV 안티패턴|Spring Boot 공식 레퍼런스, Vlad Mihalcea 블로그|
|쿼리 메서드 파싱 규칙|Spring Data JPA 공식 레퍼런스 §4.3|
|Object-Relational Impedance Mismatch|Martin Fowler — OrmHate (martinfowler.com)|

---

## 4. 이 설계를 이렇게 한 이유

### 영속성 컨텍스트가 1차 캐시를 두는 이유 — 그리고 대가

**이점:** 같은 트랜잭션 안에서 같은 엔티티를 여러 번 조회해도 SQL은 한 번만 나간다. 동일성 보장으로 `==` 비교가 의미를 가진다. 변경 감지로 `save()` 호출 없이 UPDATE가 가능하다.

**대가:** 영속성 컨텍스트는 **메모리다.** 대량 배치 처리 시 엔티티를 계속 1차 캐시에 쌓으면 OOM이 난다.

```java
// DANGEROUS: 10만 건 처리 시
for (Long id : hundredThousandIds) {
    Order order = orderRepository.findById(id).orElseThrow();
    order.setStatus(ARCHIVED);
    // 10만 개 엔티티가 영속성 컨텍스트에 쌓임 → OOM
}

// RIGHT: 주기적으로 flush + clear
@Transactional
public void batchProcess(List<Long> ids) {
    int count = 0;
    for (Long id : ids) {
        Order order = orderRepository.findById(id).orElseThrow();
        order.setStatus(ARCHIVED);
        
        if (++count % 1000 == 0) {
            em.flush();  // SQL 실행
            em.clear();  // 1차 캐시 비움
        }
    }
}
```

### LAZY 로딩이 기본값인 이유 — 그리고 잃는 것

**설계 의도:** 연관 엔티티를 항상 즉시 로딩하면 실제로 쓰지 않는 데이터까지 항상 DB에서 가져온다. LAZY는 필요할 때만 로딩해서 불필요한 쿼리를 제거한다.

**잃는 것:** 영속성 컨텍스트가 닫힌 후 LAZY 필드에 접근하면 `LazyInitializationException`이 터진다.

```java
// OSIV=false 환경에서 흔히 발생
@Transactional(readOnly = true)
public Order getOrder(Long id) {
    return orderRepository.findById(id).orElseThrow();
    // 트랜잭션 종료 → 영속성 컨텍스트 소멸
}

// Controller에서
Order order = orderService.getOrder(1L);
order.getItems().size(); // LazyInitializationException
// 영속성 컨텍스트가 없으니 프록시 초기화 불가
```

**해결 방향:** 트랜잭션 안에서 필요한 연관 데이터를 모두 로딩하거나, DTO로 변환해서 반환한다. OSIV에 의존하지 않는다.

---

## 5. 이어지는 개념

|순서|개념|이유|
|---|---|---|
|1|**Spring `@Transactional` 내부 동작 (AOP 프록시)**|JPA의 Dirty Checking과 flush가 트랜잭션 경계에 묶여있다 — `@Transactional`이 실제로 어떻게 트랜잭션을 만드는지 모르면 self-invocation 버그를 만난다|
|2|**QueryDSL**|`@Query` JPQL도 문자열이라 컴파일 타임 검증이 안 된다 — 복잡한 동적 쿼리를 타입 안전하게 작성하기 위한 실무 필수 도구|
|3|**DB 인덱스 & 실행 계획**|JPA가 생성하는 SQL이 인덱스를 타는지 안 타는지는 JPA가 알 수 없다 — N+1 해결 후 다음으로 만나는 성능 병목|
|4|**낙관적 락 / 비관적 락 (`@Version`)**|같은 엔티티를 동시에 수정하는 경우 — 영속성 컨텍스트의 동일성 보장이 분산 환경에서는 깨진다는 사실을 여기서 마주친다|