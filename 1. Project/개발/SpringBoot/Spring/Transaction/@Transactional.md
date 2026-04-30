# Spring `@Transactional` 내부 동작 — AOP 프록시

---

## 1. Problem First

### 시나리오 A: self-invocation — 가장 흔한 함정

```java
@Service
public class OrderService {

    @Transactional
    public void createOrder(OrderRequest request) {
        // 트랜잭션 적용됨
        saveOrder(request);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveOrder(OrderRequest request) {
        // 새 트랜잭션으로 분리하려 했지만...
        // 실제로는 createOrder의 트랜잭션 그대로 사용
        orderRepository.save(request.toEntity());
    }
}
```

```
기대:  createOrder 트랜잭션 → saveOrder 새 트랜잭션 (분리)
실제:  createOrder 트랜잭션 → saveOrder 같은 트랜잭션 (REQUIRES_NEW 무시)
```

배포 후 `saveOrder` 실패 시 `createOrder`까지 롤백되어야 하는데 롤백이 안 되거나, 반대로 격리되어야 할 작업이 같이 롤백되는 장애가 난다.

---

### 시나리오 B: `@Transactional`이 붙어있는데 트랜잭션이 안 걸리는 경우

```java
@Service
public class OrderService {

    @Transactional
    private void processInternal(Order order) { // private 메서드
        order.setStatus(PROCESSING);
        // Dirty Checking으로 UPDATE 되길 기대
        // → 트랜잭션 없음 → UPDATE 안 됨
    }
}
```

```java
@Service
public class OrderService {

    @Transactional
    public void process(Order order) {
        order.setStatus(PROCESSING);
    }
}

// 직접 인스턴스 생성해서 호출
OrderService service = new OrderService(repository);
service.process(order); // 트랜잭션 없음
// Spring 컨테이너를 통하지 않았기 때문
```

---

### 시나리오 C: 예외가 났는데 롤백이 안 된 경우

```java
@Transactional
public void createOrder(OrderRequest request) {
    orderRepository.save(request.toEntity());
    
    try {
        externalApiClient.notify(request); // 실패
    } catch (IOException e) {
        log.error("알림 실패", e);
        // 예외를 잡아서 처리했으니 괜찮다고 생각
        // → 트랜잭션은 커밋됨. 의도대로 동작.
    }
    
    inventoryService.decrease(request); // 여기서 RuntimeException 발생
    // → 롤백됨. 그런데...
    // IOException은 Checked Exception
    // → @Transactional 기본값: Checked Exception은 롤백 안 함
}
```

**`@Transactional`은 기본적으로 `RuntimeException`과 `Error`만 롤백한다.** `IOException`, `SQLException` 같은 Checked Exception은 롤백하지 않는다.

---

## 2. Mechanics

### 2-1. AOP 프록시 생성 과정 — 스프링 컨테이너 레벨

```
Spring 컨테이너 구동
    ↓
BeanDefinition 로딩 (@Service, @Component 스캔)
    ↓
BeanPostProcessor 목록 실행
    ↓
AutoProxyCreator (AnnotationAwareAspectJAutoProxyCreator)
    ↓
각 Bean에 대해:
    @Transactional 있는 메서드 탐색
        ↓ 있음
    ProxyFactory로 프록시 생성
        ↓
    원본 Bean 대신 프록시 Bean을 컨테이너에 등록
```

**두 가지 프록시 방식 — JDK Dynamic Proxy vs CGLIB:**

```java
// Case 1: 인터페이스가 있는 경우 → JDK Dynamic Proxy
public interface OrderService { void createOrder(...); }
public class OrderServiceImpl implements OrderService { ... }

// 생성되는 프록시:
// java.lang.reflect.Proxy가 OrderService 인터페이스를 구현한 익명 클래스 생성
// InvocationHandler로 메서드 호출을 가로챔

// Case 2: 인터페이스가 없는 경우 → CGLIB
public class OrderService { ... } // 인터페이스 없음

// 생성되는 프록시:
// CGLIB이 OrderService를 상속한 서브클래스 바이트코드를 런타임에 생성
// 메서드를 오버라이드해서 호출을 가로챔
```

**Spring Boot 2.x 이후 기본값은 CGLIB:**

```
이유:
JDK Dynamic Proxy는 인터페이스를 통해서만 주입 가능
→ 구체 타입으로 주입 시 ClassCastException 발생 가능
CGLIB은 구체 클래스를 직접 프록시 → 더 범용적

spring.aop.proxy-target-class=true (기본값)
```

> 📎 **근거:** Spring Framework 공식 레퍼런스 — _"Understanding AOP Proxies"_, docs.spring.io/spring-framework/docs/current/reference/html/core.html#aop-understanding-aop-proxies

---

### 2-2. 트랜잭션 인터셉터 — 메서드 호출 흐름

```
외부에서 orderService.createOrder() 호출
    ↓
실제로는 CGLIB 프록시의 createOrder() 호출
    ↓
TransactionInterceptor.invoke()
    ↓
TransactionAttributeSource로 @Transactional 메타데이터 읽기
    - propagation, isolation, readOnly, rollbackFor ...
    ↓
PlatformTransactionManager.getTransaction()
    ↓
현재 스레드에 트랜잭션 있는지 확인 (TransactionSynchronizationManager)
    ↓
[없음] → 새 트랜잭션 시작 → Connection 획득 → autoCommit=false
[있음] → propagation에 따라 분기
    ↓
실제 OrderServiceImpl.createOrder() 호출 (원본 메서드)
    ↓
정상 종료 → commit()
예외 발생 → rollbackOn() 조건 확인 → rollback() or commit()
    ↓
Connection 반환 (커넥션 풀로)
```

**`TransactionSynchronizationManager`가 트랜잭션 상태를 저장하는 방식:**

```java
// Spring 소스코드 내부
public abstract class TransactionSynchronizationManager {

    // ThreadLocal로 현재 스레드의 트랜잭션 리소스 관리
    private static final ThreadLocal<Map<Object, Object>> resources =
        new NamedThreadLocal<>("Transactional resources");

    private static final ThreadLocal<Boolean> actualTransactionActive =
        new NamedThreadLocal<>("Actual transaction active");
}
```

**ThreadLocal을 쓰는 이유:** 트랜잭션은 하나의 스레드 안에서 시작하고 끝난다. 여러 스레드가 동시에 각자의 트랜잭션을 갖기 위해 **스레드마다 독립된 저장공간**이 필요하다.

```
스레드 1: Connection A, Transaction X
스레드 2: Connection B, Transaction Y
스레드 3: Connection C, Transaction Z
→ ThreadLocal로 각 스레드가 자신의 Connection/Transaction만 접근
```

> 📎 **근거:** Spring Framework 소스코드 — `TransactionSynchronizationManager`, `TransactionInterceptor`, Spring 공식 레퍼런스 §Transaction Management

---

### 2-3. self-invocation이 안 되는 이유 — 정확한 메커니즘

```
외부 호출:
[외부] → [프록시.createOrder()] → [TransactionInterceptor] → [원본.createOrder()]
                                         ↑ 트랜잭션 처리

self-invocation:
[원본.createOrder()] → [원본.saveOrder()]
                            ↑ 프록시를 거치지 않음
                            ↑ TransactionInterceptor 없음
                            ↑ @Transactional 무시
```

**CGLIB 프록시가 생성하는 코드를 의사코드로 표현하면:**

```java
// CGLIB이 생성하는 OrderService 프록시 (의사코드)
public class OrderService$$SpringCGLIB extends OrderService {

    @Override
    public void createOrder(OrderRequest request) {
        // TransactionInterceptor 실행
        TransactionInterceptor.before();
        try {
            super.createOrder(request); // 원본 메서드 호출
            // ↑ 여기서 this는 원본 OrderService 인스턴스
            // this.saveOrder() 호출 시 프록시가 아닌 원본의 saveOrder()가 호출됨
            TransactionInterceptor.commit();
        } catch (Exception e) {
            TransactionInterceptor.rollback();
        }
    }
}
```

**해결책 1 — ApplicationContext에서 프록시를 직접 꺼낸다 (비권장):**

```java
@Service
public class OrderService {
    @Autowired
    private ApplicationContext applicationContext;

    public void createOrder(OrderRequest request) {
        // 자기 자신의 프록시를 꺼냄
        OrderService proxy = applicationContext.getBean(OrderService.class);
        proxy.saveOrder(request); // 프록시를 통해 호출
    }
}
// 순환 의존성, 코드 가독성 문제 → 권장하지 않음
```

**해결책 2 — `@Self` 주입 (권장):**

```java
@Service
public class OrderService {
    @Autowired
    @Lazy
    private OrderService self; // 자기 자신의 프록시 주입

    public void createOrder(OrderRequest request) {
        self.saveOrder(request); // 프록시를 통한 호출
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveOrder(OrderRequest request) { ... }
}
```

**해결책 3 — 클래스 분리 (가장 권장):**

```java
@Service
public class OrderService {
    private final OrderSaveService orderSaveService;

    public void createOrder(OrderRequest request) {
        orderSaveService.saveOrder(request); // 다른 Bean → 프록시 통과
    }
}

@Service
public class OrderSaveService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveOrder(OrderRequest request) { ... }
}
```

> 📎 **근거:** Spring Framework 공식 레퍼런스 — _"Understanding AOP Proxies"_ §self-invocation 섹션

---

### 2-4. Propagation — 6가지 전파 속성의 실제 동작

```
현재 트랜잭션 상태에 따른 분기:

                    트랜잭션 있음        트랜잭션 없음
                  ┌─────────────────┬──────────────────┐
REQUIRED          │ 기존 참여        │ 새로 시작         │ (기본값)
REQUIRES_NEW      │ 기존 중단+새로시작│ 새로 시작         │
SUPPORTS          │ 기존 참여        │ 트랜잭션 없이 실행 │
NOT_SUPPORTED     │ 기존 중단+비트랜잭션│ 비트랜잭션 실행  │
MANDATORY         │ 기존 참여        │ 예외 발생         │
NEVER             │ 예외 발생        │ 비트랜잭션 실행   │
NESTED            │ 중첩 트랜잭션    │ 새로 시작         │
                  └─────────────────┴──────────────────┘
```

**`REQUIRES_NEW`의 실제 동작 — Connection이 2개 생긴다:**

```java
@Transactional
public void createOrder(OrderRequest request) {
    orderRepository.save(...);         // Connection A 사용
    auditService.logActivity(request); // REQUIRES_NEW
    // → Connection A 일시 중단 (스택에 보관)
    // → Connection B 새로 획득
    // → auditService 완료 → Connection B 커밋/반환
    // → Connection A 재개
    inventoryService.decrease(request); // Connection A 재사용
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void logActivity(OrderRequest request) {
    // 이 트랜잭션은 createOrder 트랜잭션과 완전히 독립
    // createOrder가 롤백돼도 logActivity는 커밋됨
    auditRepository.save(...);
}
```

**`NESTED`와 `REQUIRES_NEW`의 차이:**

```
REQUIRES_NEW:
  - 완전히 독립된 트랜잭션
  - 부모 롤백과 무관하게 독립 커밋/롤백

NESTED:
  - 부모 트랜잭션 안의 Savepoint
  - 자식 롤백 → Savepoint로 복구 (부모는 계속)
  - 부모 롤백 → 자식도 같이 롤백
  - JDBC Savepoint 지원 필요 (MySQL, PostgreSQL 지원)
```

```java
@Transactional
public void createOrder(OrderRequest request) {
    orderRepository.save(...);
    
    try {
        notificationService.send(request); // NESTED
    } catch (Exception e) {
        // 알림 실패 → Savepoint로 롤백 (주문 저장은 유지)
        log.warn("알림 실패, 주문은 계속 진행");
    }
    
    inventoryService.decrease(request);
    // 전체 성공 → commit
}
```

> 📎 **근거:** Spring Framework 공식 레퍼런스 §Transaction Propagation, JPA 2.2 명세 (JSR-338) §7.6.4 — _"Transaction Propagation"_

---

### 2-5. `private` 메서드에 `@Transactional`이 안 되는 이유

```java
@Service
public class OrderService {

    @Transactional
    private void process() { ... } // 트랜잭션 안 걸림
}
```

**CGLIB 기반으로 이유를 설명하면:**

```java
// CGLIB은 상속으로 프록시를 만든다
public class OrderService$$SpringCGLIB extends OrderService {

    @Override
    public void process() { // private는 오버라이드 불가
        // → CGLIB이 이 메서드를 가로챌 방법이 없음
    }
}
// Java에서 private 메서드는 상속 자체가 불가
// → 프록시가 개입할 지점이 없음
```

**같은 이유로 `final` 메서드도 안 된다:**

```java
@Transactional
public final void process() { ... }
// CGLIB은 final 메서드를 오버라이드 못함 → 트랜잭션 무시
// JDK Dynamic Proxy도 마찬가지
```

> 📎 **근거:** Spring Framework 공식 레퍼런스 — _"Method visibility and @Transactional"_ — _"In proxy mode, only external method calls coming in through the proxy are intercepted."_ (프록시 모드에서는 프록시를 통해 들어오는 외부 메서드 호출만 인터셉트된다.)

---

### 2-6. 롤백 규칙 — 정확한 조건

```java
// 기본 롤백 규칙
@Transactional
// RuntimeException, Error → 롤백
// Checked Exception       → 커밋 (!!)

// 커스텀 롤백 규칙
@Transactional(rollbackFor = Exception.class)
// 모든 예외 → 롤백

@Transactional(noRollbackFor = IllegalArgumentException.class)
// IllegalArgumentException은 RuntimeException이지만 롤백 안 함
```

**내부 판단 로직:**

```java
// Spring 소스코드 (RuleBasedTransactionAttribute)
public boolean rollbackOn(Throwable ex) {
    // 1. noRollbackFor 규칙 확인 → 매칭되면 커밋
    // 2. rollbackFor 규칙 확인   → 매칭되면 롤백
    // 3. 기본 규칙: RuntimeException || Error → 롤백
    //              그 외 → 커밋
}
```

**Checked Exception이 기본적으로 롤백 안 하는 설계 의도:**

```
EJB 시절부터 내려온 관례:
- RuntimeException = 예상치 못한 오류 → 트랜잭션 오염 → 롤백
- Checked Exception = 예상 가능한 비즈니스 예외
  (재고 없음, 잔액 부족 등) → 애플리케이션이 처리 가능 → 커밋
```

**실무에서 이 기본값 때문에 생기는 버그:**

```java
@Transactional
public void transfer(Long fromId, Long toId, BigDecimal amount)
        throws InsufficientBalanceException { // Checked Exception

    accountRepository.decrease(fromId, amount);
    // 여기서 InsufficientBalanceException 발생

    accountRepository.increase(toId, amount);
    // 실행 안 됨

    // InsufficientBalanceException은 Checked → 롤백 안 됨
    // decrease는 커밋됨 → 돈이 사라짐
}

// 반드시:
@Transactional(rollbackFor = InsufficientBalanceException.class)
```

> 📎 **근거:** Spring Framework 공식 레퍼런스 §Declarative transaction rolling back, _Effective Java_ 3판 Item 70 — Checked vs Unchecked Exception 사용 기준

---

### 2-7. `readOnly = true`가 실제로 하는 일

```java
@Transactional(readOnly = true)
public Order findOrder(Long id) { ... }
```

**`readOnly = true`가 하는 일을 레이어별로:**

```
1. Hibernate 레벨:
   - FlushMode = MANUAL (자동 flush 비활성화)
   - Dirty Checking 스냅샷 생성 안 함
   → 메모리, CPU 절약

2. JDBC 드라이버 레벨 (드라이버마다 다름):
   - Connection.setReadOnly(true) 전달
   → 일부 드라이버는 힌트로만 처리

3. DB 레벨 (MySQL InnoDB):
   - 읽기 전용 트랜잭션으로 최적화
   - undo log 생성 최소화
   - 읽기 전용 복제본(Replica)으로 라우팅 가능
     (AbstractRoutingDataSource와 함께 사용 시)
```

**`readOnly = true`가 하지 않는 일:**

```java
@Transactional(readOnly = true)
public void findAndModify(Long id) {
    Order order = orderRepository.findById(id).orElseThrow();
    order.setStatus(COMPLETED);
    // FlushMode = MANUAL이므로 자동 flush 안 됨
    // 명시적으로 em.flush() 호출하면 UPDATE 실행됨
    // readOnly가 물리적 쓰기를 막지는 않음 — 힌트일 뿐
}
```

> 📎 **근거:** Hibernate 공식 문서 — _"Read-only transactions"_, Spring Framework 소스코드 — `HibernateTransactionManager.doBegin()`

---

## 3. 공식 근거 정리

|주장|출처|
|---|---|
|AOP 프록시 생성 방식 (JDK vs CGLIB)|Spring Framework 공식 레퍼런스 §AOP Proxies|
|self-invocation 동작|Spring 공식 레퍼런스 §Understanding AOP Proxies|
|TransactionSynchronizationManager ThreadLocal|Spring Framework 소스코드|
|private/final 메서드 트랜잭션 미적용|Spring 공식 레퍼런스 §Method visibility and @Transactional|
|트랜잭션 전파 속성|Spring 공식 레퍼런스 §Transaction Propagation|
|Checked Exception 기본 커밋|Spring 공식 레퍼런스 §Rolling Back|
|readOnly 동작 범위|Hibernate 공식 문서, Spring `HibernateTransactionManager` 소스|

---

## 4. 이 설계를 이렇게 한 이유

### AOP 프록시 방식의 이점과 대가

**이점:** 비즈니스 코드에 트랜잭션 관리 코드가 전혀 없다. `@Transactional` 하나로 선언적 처리. 트랜잭션 관리 방식(JPA, JDBC, JTA)을 교체해도 비즈니스 코드는 변경 없음.

**대가:** 프록시 기반이기 때문에 아래 제약이 구조적으로 발생한다.

```
- self-invocation 미작동
- private/final 메서드 미작동
- Spring 컨테이너 밖에서 직접 생성한 객체 미작동
- 멀티스레드: @Transactional 메서드가 새 스레드를 만들면
  자식 스레드는 부모 트랜잭션을 이어받지 못함
  (ThreadLocal은 스레드 간 공유 안 됨)
```

```java
@Transactional
public void createOrder(OrderRequest request) {
    CompletableFuture.runAsync(() -> {
        // 새 스레드 → ThreadLocal에 트랜잭션 없음
        // @Transactional 없는 것과 동일
        orderRepository.save(request.toEntity());
    });
}
```

### CGLIB이 기본값이 된 이유 — 그리고 잃는 것

**이점:** 인터페이스 없이도 프록시 생성 가능. 구체 타입으로 주입 가능.

**대가:**

```
- 기본 생성자 필요 (없으면 CGLIB 프록시 생성 실패)
  → Spring 4.0 이후 Objenesis로 완화됨
- final 클래스는 상속 불가 → 프록시 생성 불가
- 바이트코드 조작 라이브러리 의존성 추가
```

---

## 5. 이어지는 개념

|순서|개념|이유|
|---|---|---|
|1|**Spring AOP 전체 (Pointcut, Advice, Aspect)**|`@Transactional`은 AOP의 한 적용 사례다 — 커스텀 AOP를 작성하거나 트랜잭션 인터셉터 동작을 더 깊이 이해하려면 AOP 구조 자체를 알아야 한다|
|2|**낙관적 락 / 비관적 락 (`@Version`)**|트랜잭션 경계를 정확히 이해한 다음 단계 — 동시 수정 문제는 트랜잭션만으로 해결 안 되는 지점에서 시작된다|
|3|**트랜잭션 격리 수준 (Isolation Level)**|`@Transactional(isolation = ...)` 옵션이 DB 레벨에서 실제로 무엇을 하는지 — Dirty Read, Phantom Read가 어떤 상황에서 발생하는지 트랜잭션 내부를 알아야 이해된다|
|4|**분산 트랜잭션 (2PC, Saga 패턴)**|단일 DB 트랜잭션의 한계 — MSA에서 여러 서비스에 걸친 데이터 정합성을 어떻게 보장하는가, `@Transactional`이 통하지 않는 환경에서의 대안|