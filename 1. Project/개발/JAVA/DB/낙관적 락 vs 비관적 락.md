# 낙관적 락 vs 비관적 락

---

## 1. Problem First

### 시나리오: 재고 차감 레이스 컨디션

```java
// 재고 서비스 - 락 없음
@Transactional
public void decreaseStock(Long productId, int quantity) {
    Product product = productRepository.findById(productId).orElseThrow();
    // [1] SELECT: stock = 100
    
    if (product.getStock() < quantity) {
        throw new InsufficientStockException();
    }
    
    product.setStock(product.getStock() - quantity); // [2] 100 - 10 = 90
    // [3] UPDATE: stock = 90
}
```

```
Thread A: SELECT stock=100  →  UPDATE stock=90  (커밋)
Thread B:   SELECT stock=100          →  UPDATE stock=90  (커밋)

결과: 주문 2건 처리, 재고 90 → 실제 재고는 80이어야 함
Lost Update 발생
```

이게 단순한 "동시성 버그"가 아닌 이유:

- **DB 트랜잭션 격리 수준으로는 해결 안 됨.** `REPEATABLE READ`는 같은 트랜잭션 내 재조회 일관성을 보장하지, 다른 트랜잭션의 write를 막지 않는다
- **애플리케이션 레벨 `synchronized`로는 해결 안 됨.** 다중 서버 환경에서는 JVM 락이 서버 간 공유되지 않는다
- **결과적 일관성(Eventual Consistency)도 해결 안 됨.** 재고는 음수가 되어선 안 되는 하드 제약이다

> 이 문제를 해결하려면 "읽은 시점의 데이터가 커밋 시점까지 변하지 않았음"을 보장하는 메커니즘이 필요하다. 이것이 락 전략을 선택하는 근본 이유다.

---

## 2. Mechanics

### 2-1. 비관적 락 (Pessimistic Lock)

**전제 가정:** "어차피 충돌이 날 것이다. 미리 막자."

DB의 `SELECT ... FOR UPDATE` 또는 `SELECT ... FOR SHARE`를 사용한다.

```java
// Spring Data JPA
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdWithLock(@Param("id") Long id);
}
```

**DB 레벨에서 일어나는 일 (InnoDB 기준)**

```
Thread A: SELECT ... FOR UPDATE  →  X Lock 획득
Thread B: SELECT ... FOR UPDATE  →  Lock wait (innodb_lock_wait_timeout 기본 50초)

Thread A가 커밋/롤백 시 → Lock 해제 → Thread B 진행
```

InnoDB의 `SELECT ... FOR UPDATE`는 **넥스트 키 락(Next-Key Lock)** 을 사용한다. 레코드 락 + 갭 락의 조합으로, 인덱스 레코드와 그 앞의 갭을 함께 잠근다.

```
인덱스: [10] [20] [30]
SELECT ... FOR UPDATE WHERE id = 20

→ (10, 20] 범위에 Next-Key Lock
→ (20, 30] 갭에도 Lock (팬텀 리드 방지)
```

> **MySQL 공식 문서:** "A next-key lock is a combination of a record lock on the index record and a gap lock on the gap before the index record." — [InnoDB Locking, MySQL 8.0 Reference Manual](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)
> 
> _(번역: 넥스트 키 락은 인덱스 레코드에 대한 레코드 락과 인덱스 레코드 앞의 갭에 대한 갭 락의 조합이다.)_

**LockModeType 종류와 실제 SQL 매핑**

|JPA LockModeType|실제 SQL|허용|
|---|---|---|
|`PESSIMISTIC_WRITE`|`SELECT ... FOR UPDATE`|다른 읽기/쓰기 모두 대기|
|`PESSIMISTIC_READ`|`SELECT ... FOR SHARE`|다른 읽기는 허용, 쓰기만 대기|
|`PESSIMISTIC_FORCE_INCREMENT`|`FOR UPDATE` + version 증가|위와 동일 + 버전 카운터 증가|

> **JPA 2.2 Specification (JSR 338), Section 3.4.4:** "PESSIMISTIC_WRITE lock mode type specifies a database lock to prevent the entity from being modified or deleted by other transactions."
> 
> _(번역: PESSIMISTIC_WRITE 락 모드 타입은 다른 트랜잭션이 엔티티를 수정하거나 삭제하는 것을 방지하기 위한 데이터베이스 락을 명시한다.)_

---

### 2-2. 낙관적 락 (Optimistic Lock)

**전제 가정:** "충돌이 드물다. 일단 진행하고, 커밋 시점에 확인하자."

DB 락을 사용하지 않는다. **버전 컬럼**을 두어 CAS(Compare-And-Swap) 방식으로 충돌을 감지한다.

```java
@Entity
public class Product {
    @Id
    private Long id;
    
    private int stock;
    
    @Version  // JPA가 관리하는 버전 컬럼
    private Long version;
}
```

**커밋 시점에 일어나는 일**

```sql
-- JPA가 내부적으로 생성하는 UPDATE
UPDATE product
SET stock = 90, version = 2        -- version을 현재+1로
WHERE id = 1 AND version = 1       -- 읽은 시점의 version과 일치해야 커밋
```

```
Thread A: SELECT (stock=100, version=1)
Thread B: SELECT (stock=100, version=1)

Thread A: UPDATE WHERE version=1 → 성공 (version=2로 변경됨)
Thread B: UPDATE WHERE version=1 → affected rows = 0
          → JPA: OptimisticLockException 발생
```

**JPA 내부 처리 흐름 (Hibernate 기준)**

```
EntityManager.flush()
  └─ DefaultFlushEntityEventListener.onFlushEntity()
       └─ isVersionIncrementRequired() → true (dirty check)
            └─ Versioning.increment(version)
                 └─ UPDATE ... WHERE id=? AND version=?
                      └─ affectedRows == 0
                           └─ throw StaleObjectStateException
                                └─ (JPA wrapper) OptimisticLockException
```

> **Hibernate ORM 공식 문서:** "If the version check fails, the persistence provider must throw an OptimisticLockException and mark the transaction for rollback." — [Hibernate ORM 6.x User Guide, §20.2](https://docs.jboss.org/hibernate/orm/6.4/userguide/html_single/Hibernate_User_Guide.html#locking-optimistic)
> 
> _(번역: 버전 확인이 실패하면, 퍼시스턴스 프로바이더는 OptimisticLockException을 던지고 트랜잭션을 롤백으로 표시해야 한다.)_

**재시도 처리**

`OptimisticLockException`은 비즈니스 로직에서 잡아서 재시도해야 한다. Spring은 이를 자동화하지 않는다.

```java
@Service
public class StockService {

    // Spring Retry 활용
    @Retryable(
        retryFor = OptimisticLockingFailureException.class,
        maxAttempts = 3,
        backoff = @Backoff(delay = 100, multiplier = 2)
    )
    @Transactional
    public void decreaseStock(Long productId, int quantity) {
        Product product = productRepository.findById(productId).orElseThrow();
        product.decreaseStock(quantity);
        // flush 시점에 version 체크 → 실패 시 OptimisticLockingFailureException
    }

    @Recover
    public void recover(OptimisticLockingFailureException e, Long productId, int quantity) {
        throw new StockUpdateFailedException("재고 업데이트 실패: 동시 요청 과다", e);
    }
}
```

> **주의:** JPA의 `OptimisticLockException`은 Spring의 `OptimisticLockingFailureException`으로 변환된다 (`JpaOptimisticLockingFailureException`). `@Retryable`에는 Spring 예외 타입을 지정해야 한다.

---

### 2-3. 두 전략의 동작 비교

```
[비관적 락] - 충돌 시
─────────────────────────────────────────────
T=0ms  Thread A: LOCK 획득
T=0ms  Thread B: LOCK 대기 (blocked)
T=50ms Thread A: 커밋, LOCK 해제
T=50ms Thread B: LOCK 획득, 진행
T=100ms Thread B: 커밋

총 소요: 100ms, 실패 없음, 순차 처리


[낙관적 락] - 충돌 시
─────────────────────────────────────────────
T=0ms  Thread A: SELECT (version=1)
T=0ms  Thread B: SELECT (version=1)
T=50ms Thread A: UPDATE WHERE version=1 → 성공, version=2
T=50ms Thread B: UPDATE WHERE version=1 → 실패 (OptimisticLockException)
T=50ms Thread B: 재시도 1회차
T=100ms Thread B: SELECT (version=2), UPDATE WHERE version=2 → 성공

총 소요: 100ms + 재시도 오버헤드, 한 번의 실패 발생
```

---

## 3. 공식 근거

|주장|출처|
|---|---|
|`@Lock(PESSIMISTIC_WRITE)` → `SELECT ... FOR UPDATE`|JPA 2.2 Specification (JSR 338), §3.4.4|
|InnoDB Next-Key Lock 동작|MySQL 8.0 Reference Manual - InnoDB Locking|
|`@Version` 동작과 `OptimisticLockException` 발생 조건|Hibernate ORM 6.x User Guide, §20.2|
|`PESSIMISTIC_READ` → `FOR SHARE` 매핑|Jakarta Persistence 3.1 Spec, Table 3.9|

`@Retryable`과 `JpaOptimisticLockingFailureException` 변환 관계는 Spring 공식 문서에 명시적 설명이 없고, Spring Data JPA 소스코드(`SharedEntityManagerCreator`, `EntityManagerFactoryUtils`)에서 확인되는 구현 수준이다.

---

## 4. 이 설계를 이렇게 한 이유

### 비관적 락을 선택했을 때 잃는 것

**처리량(Throughput)**

락을 잡고 있는 동안 다른 모든 스레드가 대기한다. 트랜잭션이 길수록, 동시 요청이 많을수록 병목이 된다.

```
DB Connection Pool: 10개
평균 트랜잭션 시간: 200ms (락 대기 포함)
→ 최대 TPS = 10 / 0.2s = 50 TPS

락 없을 때 평균 트랜잭션 시간: 20ms
→ 최대 TPS = 10 / 0.02s = 500 TPS
```

**데드락 위험**

```java
// Thread A: Product(1) 락 → Order(100) 락 시도
// Thread B: Order(100) 락 → Product(1) 락 시도
// → Deadlock
```

InnoDB는 데드락을 감지하고 한 트랜잭션을 롤백시키지만(`innodb_deadlock_detect=ON`), 그 롤백 비용 자체가 시스템 부하가 된다.

**분산 환경 복잡도**

DB 락은 단일 DB 인스턴스 범위다. 샤딩이나 멀티 DB 환경에서는 적용할 수 없다.

---

### 낙관적 락을 선택했을 때 잃는 것

**충돌이 많을 때 재시도 폭풍**

```
동시 요청 100개 → 1개 성공, 99개 OptimisticLockException
99개 재시도 → 1개 성공, 98개 실패
...
→ 결국 처리는 되지만, DB에 불필요한 쿼리 폭증
→ 플래시 세일, 선착순 이벤트 같은 극단적 경쟁 상황에서 역효과
```

**재시도 로직 구현 책임이 애플리케이션으로**

재시도 횟수, 백오프 전략, 최종 실패 처리를 모두 직접 설계해야 한다. 잘못 구현하면 재시도 자체가 장애 원인이 된다.

**긴 트랜잭션에서 무의미해짐**

트랜잭션이 길면 충돌 확률이 올라가고, 재시도해도 또 충돌날 가능성이 높다. 낙관적 락의 전제("충돌이 드물다")가 깨진다.

---

### 선택 기준 정리

|상황|선택|이유|
|---|---|---|
|충돌 빈도 낮음, 읽기 多|낙관적 락|DB 락 없이 처리량 유지|
|충돌 빈도 높음 (경쟁 자원)|비관적 락|재시도 폭풍보다 순차 처리가 비용 낮음|
|트랜잭션이 긴 배치 작업|비관적 락|낙관적 락의 전제(드문 충돌)가 성립 안 됨|
|분산 DB / 샤딩 환경|낙관적 락 또는 Redis 분산 락|DB 락은 단일 인스턴스에만 적용됨|
|결제, 재고 차감 (정합성 최우선)|비관적 락 + 짧은 트랜잭션|재시도 실패 시 비즈니스 손실이 큼|

---

## 5. 이어지는 개념

### 1순위 — 선행 의존

**트랜잭션 격리 수준 (Isolation Level)**

낙관적/비관적 락은 격리 수준과 결합하여 동작한다. `REPEATABLE READ`에서 `SELECT ... FOR UPDATE`가 왜 팬텀 리드를 막는지, `READ COMMITTED`에서 낙관적 락이 왜 더 취약한지를 이해하지 못하면 락 전략의 동작 경계를 알 수 없다.

**MVCC (Multi-Version Concurrency Control)**

InnoDB가 비관적 락 대기 중에도 일반 SELECT를 블로킹하지 않는 이유다. 낙관적 락의 "읽기는 자유롭다"는 보장이 MVCC 위에서 성립한다. 이 구조를 모르면 락 전략의 성능 특성을 잘못 예측한다.

### 2순위 — 실무 임팩트

**분산 락 (Redis Redlock / Redisson)**

비관적 락은 단일 DB에 묶인다. 멀티 인스턴스, 샤딩, MSA 환경에서 동일한 보장을 얻으려면 분산 락이 필요하다. Redis `SETNX` 기반 구현과 Redlock 알고리즘의 한계(네트워크 파티션 시 안전성 논쟁)까지 알아야 실무에서 올바른 선택을 한다.

**데드락 탐지와 회피 전략**

비관적 락을 실무에 적용하는 순간 데드락은 필연적으로 마주친다. InnoDB의 데드락 탐지 메커니즘, `SHOW ENGINE INNODB STATUS`로 데드락 원인 분석, 락 획득 순서 정규화를 통한 회피 방법은 비관적 락 운영의 필수 지식이다.

**JPA 영속성 컨텍스트와 flush 타이밍**

낙관적 락의 버전 체크는 flush 시점에 발생한다. `OSIV(Open Session In View)` 패턴이 켜진 상태에서 트랜잭션 밖에서 flush가 일어나면 `OptimisticLockException`이 Controller 레이어에서 터진다. 이 연결고리를 모르면 예외 처리 전략을 잘못 설계한다.

---

# 면접 시나리오: 낙관적 락 vs 비관적 락

---

## 오프닝 질문 유형별 답변 전략

면접관이 이 주제를 꺼내는 방식은 보통 세 가지다.

---

### 유형 1. "낙관적 락과 비관적 락의 차이를 설명해보세요"

> 가장 흔한 오프닝. 여기서 개념 나열로 끝내면 꼬리질문 주도권을 면접관에게 넘긴다. **문제 → 메커니즘 → 트레이드오프** 순으로 답하면 꼬리질문을 내가 유도할 수 있다.

**모범 답변 흐름**

```
"두 전략 모두 동시성 환경에서 Lost Update를 막기 위한 방법인데,
전제 가정이 다릅니다.

비관적 락은 '어차피 충돌이 날 것'이라고 가정하고,
DB 레벨에서 SELECT ... FOR UPDATE로 X Lock을 걸어
다른 트랜잭션이 해당 행에 접근 자체를 막습니다.

낙관적 락은 '충돌이 드물 것'이라고 가정하고,
DB 락을 걸지 않습니다. 대신 @Version 컬럼을 두고
커밋 시점에 UPDATE WHERE version = {읽은 시점 버전}을 실행해서
affected rows가 0이면 충돌로 판단하고 OptimisticLockException을 던집니다.

선택 기준은 충돌 빈도입니다.
충돌이 잦으면 낙관적 락의 재시도 비용이 비관적 락의 대기 비용보다 커지기 때문에
경쟁이 심한 자원에는 비관적 락이 더 적합합니다."
```

**이 답변의 의도**

- "충돌 빈도"로 끝냄으로써 면접관이 자연스럽게 **"그럼 언제 어떤 걸 쓰나요?"** 로 넘어가게 유도
- 직접 "재시도 폭풍"이나 "DB Connection Pool"까지 언급하면 꼬리질문 없이 내가 주도 가능

---

### 유형 2. "실무에서 동시성 문제를 어떻게 해결했나요?" (경험 기반)

> 프로젝트 경험이 있으면 이 포맷으로 답하는 게 가장 강하다.

```
"재고 차감 API에서 Lost Update 이슈를 겪었습니다.

처음에는 @Transactional만 걸었는데,
REPEATABLE READ 격리 수준에서도 다른 트랜잭션의 write는 막지 않아서
재고가 음수가 되는 상황이 발생했습니다.

두 가지 옵션을 검토했습니다.
낙관적 락(@Version)을 먼저 시도했는데,
플래시 세일처럼 동시 요청이 몰리는 상황에서
OptimisticLockException 재시도가 폭증해서 DB 부하가 오히려 증가했습니다.

결국 비관적 락(SELECT ... FOR UPDATE)으로 전환했고,
트랜잭션 범위를 최소화해서 Lock 점유 시간을 줄이는 방향으로 처리했습니다."
```

**경험이 없을 경우 대체 포맷**

```
"직접 겪진 않았지만, 이 문제를 설계한다면 이렇게 접근하겠습니다.
[위 내용 동일하게]"

→ "경험은 없지만 설계 근거가 있다"는 포지션이
   "모르겠습니다"보다 훨씬 강하다.
```

---

## 꼬리질문 맵

```
낙관적 락 vs 비관적 락
│
├─ [메커니즘 방향]
│   ├─ @Version이 없으면 낙관적 락이 동작하나요?
│   ├─ PESSIMISTIC_READ와 PESSIMISTIC_WRITE 차이는?
│   └─ 낙관적 락에서 version 체크는 언제 일어나나요?
│
├─ [한계/장애 방향]
│   ├─ 비관적 락에서 데드락이 나면 어떻게 처리하나요?
│   ├─ 낙관적 락 재시도를 무한정 하면 어떻게 되나요?
│   └─ 트랜잭션이 길면 어떤 전략이 더 위험한가요?
│
├─ [확장 방향]
│   ├─ 서버가 2대면 비관적 락이 여전히 유효한가요?
│   ├─ Redis로 분산 락을 구현한다면?
│   └─ DB 락 없이 동시성을 보장하는 방법이 있나요?
│
└─ [격리 수준 방향]
    ├─ REPEATABLE READ에서도 Lost Update가 생기는 이유는?
    └─ READ COMMITTED에서 비관적 락의 동작이 달라지나요?
```

---

## 꼬리질문별 핵심 답변

### Q. `@Version`이 없으면 낙관적 락이 동작하나요?

```
"동작하지 않습니다.

@Version은 JPA가 낙관적 락을 구현하는 유일한 방법입니다.
이 컬럼이 없으면 JPA는 일반 UPDATE를 날리고
충돌을 감지할 방법이 없습니다.

@Lock(LockModeType.OPTIMISTIC)을 명시해도
@Version 필드가 없으면 Hibernate는 PersistenceException을 던집니다."
```

**근거:** Hibernate User Guide §20.2 — version 필드 없이 낙관적 락 모드를 사용하면 예외 발생

---

### Q. version 체크는 언제 일어나나요?

> 이 질문은 JPA 내부를 아는지 보는 것이다.

```
"EntityManager.flush() 시점에 일어납니다.
@Transactional 메서드가 끝나면서 flush가 트리거되고,
그때 UPDATE WHERE id=? AND version=? 쿼리가 나갑니다.

중요한 포인트는,
OSIV(Open Session In View)가 켜져 있으면
트랜잭션이 끝난 후 View 레이어에서 flush가 발생할 수도 있고,
그러면 OptimisticLockException이 Controller에서 터집니다.
예외 처리 위치를 잘못 설계하면 핸들링이 안 됩니다."
```

---

### Q. 서버가 2대면 비관적 락이 유효한가요?

> 확장 방향 꼬리질문 중 가장 자주 나온다.

```
"DB가 단일 인스턴스라면 유효합니다.
비관적 락은 DB 레벨의 X Lock이기 때문에
어느 서버에서 요청이 들어오든 DB에서 직렬화됩니다.

하지만 DB가 샤딩되거나 멀티 마스터 구조라면
락 범위가 단일 노드로 한정되어 보장이 깨집니다.
이 경우 Redis 분산 락(Redisson RLock)을 사용하거나
아키텍처 레벨에서 해당 자원의 요청을 단일 노드로 라우팅하는 방법을 씁니다."
```

---

### Q. 데드락이 발생하면 어떻게 대응하나요?

```
"InnoDB는 데드락을 자동 감지해서 한 트랜잭션을 롤백합니다.
애플리케이션에서는 DeadlockLoserDataAccessException으로 올라오고,
이건 재시도 가능한 예외입니다.

근본 예방은 락 획득 순서를 항상 동일하게 정규화하는 것입니다.
Product → Order 순서로 락을 잡는다고 전사 규칙을 정해두면
순환 대기 자체가 생기지 않습니다.

발생 원인 분석은 SHOW ENGINE INNODB STATUS의
LATEST DETECTED DEADLOCK 섹션에서
어떤 트랜잭션이 어떤 순서로 락을 잡았는지 확인합니다."
```

---

### Q. DB 락 없이 동시성을 보장하는 방법이 있나요?

> 깊이를 보는 질문. 여기까지 오면 상위 레벨이다.

```
"세 가지 방향이 있습니다.

첫째, 큐 직렬화입니다.
동일 자원에 대한 요청을 Kafka나 Redis Queue로 받아서
단일 컨슈머가 순차 처리하게 만들면
DB 락 없이 충돌 자체를 제거할 수 있습니다.
다만 처리 지연이 생기고 아키텍처 복잡도가 올라갑니다.

둘째, 원자적 UPDATE입니다.
SELECT 후 UPDATE하는 패턴 대신
UPDATE product SET stock = stock - ? WHERE id = ? AND stock >= ?
이렇게 조건부 원자적 UPDATE 한 방으로 처리하면
읽기-수정-쓰기 사이클 자체가 없어집니다.
affected rows로 성공 여부를 판단합니다.

셋째, Redis의 원자적 연산입니다.
DECR, Lua 스크립트 등으로 Redis에서 재고를 관리하고
DB는 최종 동기화에만 쓰는 방식입니다.
Redis가 단일 스레드로 명령을 처리하기 때문에
락 없이 원자성이 보장됩니다."
```

---

## 면접관이 원하는 신호

|수준|보이는 행동|
|---|---|
|**하**|개념 정의만 나열, "상황에 따라 다르다"로 마무리|
|**중**|메커니즘 설명 + 선택 기준 제시|
|**상**|트레이드오프를 구체적 수치/시나리오로 설명, 한계 인지|
|**최상**|"이 전략이 틀리는 상황"을 먼저 언급, 대안 설계까지 제시|

가장 강한 포지션은 **"비관적 락이 좋다/낙관적 락이 좋다"가 아니라 "이 상황에서 비관적 락을 선택하면 이것을 잃는다, 그래서 이 조건일 때만 쓴다"** 를 말하는 것이다.