 — 횡단 관심사를 분리한다

---
## 1. Problem First

### AOP가 없던 시절

```java
// 핵심 비즈니스 로직에 부가 기능이 뒤섞인다
@Service
public class OrderService {
    private static final Logger log = LoggerFactory.getLogger(OrderService.class);
    private final TransactionManager tm;

    public void createOrder(OrderRequest request) {
        // 트랜잭션 시작 — 핵심 로직 아님
        Transaction tx = tm.begin();
        long start = System.currentTimeMillis();  // 성능 측정 — 핵심 로직 아님
        log.info("createOrder 시작: {}", request); // 로깅 — 핵심 로직 아님

        try {
            // 핵심 로직 — 단 3줄
            Order order = new Order(request);
            orderRepository.save(order);
            paymentService.pay(order);

            tx.commit();
        } catch (Exception e) {
            tx.rollback();
            log.error("createOrder 실패", e);
            throw e;
        } finally {
            long elapsed = System.currentTimeMillis() - start;
            log.info("createOrder 종료: {}ms", elapsed); // 핵심 로직 아님
        }
    }

    // 모든 메서드가 이 구조를 반복
    public void cancelOrder(Long orderId) {
        Transaction tx = tm.begin();
        long start = System.currentTimeMillis();
        log.info("cancelOrder 시작: {}", orderId);
        try {
            // 핵심 로직 2줄
            Order order = orderRepository.findById(orderId);
            order.cancel();

            tx.commit();
        } catch (Exception e) {
            tx.rollback();
            throw e;
        } finally {
            log.info("cancelOrder 종료: {}ms", System.currentTimeMillis() - start);
        }
    }
}
```

**근본적인 문제:**

트랜잭션 관리 코드를 바꾸려면 이 패턴이 박힌 모든 메서드를 찾아서 수정해야 한다. `OrderService`, `UserService`, `PaymentService` 전부. 핵심 로직이 뭔지 코드만 보고 파악하기 어렵다.

---

## 2. Mechanics

### AOP의 핵심 개념 — 용어 정리부터

```
Aspect       = 횡단 관심사 모듈 (트랜잭션, 로깅 등을 담은 클래스)
Advice       = 실제 부가 기능 코드 (언제 + 무엇을)
Pointcut     = Advice를 어디에 적용할지 (표현식으로 정의)
JoinPoint    = Advice가 끼어들 수 있는 지점 (메서드 실행, 예외 발생 등)
Target       = Advice가 적용될 실제 객체 (OrderService 원본)
Proxy        = Target을 감싸는 대리 객체 (스프링이 만들어 줌)
Weaving      = Aspect를 Target에 적용하는 과정
```

```
호출자
  │  orderService.createOrder() 호출
  ▼
Proxy (스프링이 생성)
  │  Pointcut 조건 확인 → 이 메서드에 Advice 적용 대상인가?
  │  Before Advice 실행
  │  → Target.createOrder() 실행 위임
  │  After/Around Advice 실행
  ▼
호출자에게 결과 반환
```

---

### 프록시 생성 방식 — JDK Dynamic Proxy vs CGLIB

스프링은 두 가지 프록시 방식을 상황에 따라 선택한다.

**JDK Dynamic Proxy:**

```java
// 인터페이스가 있을 때
public interface OrderService {
    void createOrder(OrderRequest request);
}

@Service
public class OrderServiceImpl implements OrderService {
    public void createOrder(OrderRequest request) { ... }
}

// 스프링이 생성하는 프록시 — java.lang.reflect.Proxy 사용
OrderService proxy = (OrderService) Proxy.newProxyInstance(
    OrderServiceImpl.class.getClassLoader(),
    new Class[]{OrderService.class},  // 인터페이스 기반
    new InvocationHandler() {
        public Object invoke(Object proxy, Method method, Object[] args) {
            // Before Advice
            Object result = method.invoke(target, args); // 원본 호출
            // After Advice
            return result;
        }
    }
);

// 한계: 인터페이스로만 캐스팅 가능
// OrderServiceImpl impl = (OrderServiceImpl) proxy; // ClassCastException
```

**CGLIB (Code Generation Library):**

```java
// 인터페이스 없을 때 (또는 Spring Boot 기본값)
@Service
public class OrderService {  // 인터페이스 없음
    public void createOrder(OrderRequest request) { ... }
}

// 스프링이 생성하는 프록시 — 서브클래스 바이트코드를 런타임에 생성
// 실제로는 이런 클래스가 동적으로 만들어짐
public class OrderService$$SpringCGLIB$$0 extends OrderService {
    @Override
    public void createOrder(OrderRequest request) {
        // Advice 실행
        super.createOrder(request); // 원본 호출
        // Advice 실행
    }
}

// 한계: final 클래스, final 메서드에 적용 불가
// → 서브클래스를 만들 수 없기 때문
@Service
public final class OrderService { ... } // CGLIB 프록시 생성 불가
```

**Spring Boot의 기본값:**

```
Spring Boot 2.0 이후 → CGLIB이 기본
이유: 인터페이스 없는 클래스에도 일관되게 동작
     @Autowired 시 구체 클래스 타입으로도 주입 가능

변경하려면:
spring.aop.proxy-target-class=false → JDK Dynamic Proxy 사용
```

---

### Advice 종류와 실행 시점

```java
@Aspect
@Component
public class LoggingAspect {

    // 메서드 실행 전
    @Before("execution(* com.example.service.*.*(..))")
    public void before(JoinPoint jp) {
        log.info("호출: {}", jp.getSignature().getName());
    }

    // 메서드 정상 반환 후
    @AfterReturning(
        pointcut = "execution(* com.example.service.*.*(..))",
        returning = "result"
    )
    public void afterReturning(JoinPoint jp, Object result) {
        log.info("반환: {}", result);
    }

    // 예외 발생 후
    @AfterThrowing(
        pointcut = "execution(* com.example.service.*.*(..))",
        throwing = "ex"
    )
    public void afterThrowing(JoinPoint jp, Exception ex) {
        log.error("예외: {}", ex.getMessage());
    }

    // 정상/예외 무관하게 실행 후
    @After("execution(* com.example.service.*.*(..))")
    public void after(JoinPoint jp) {
        log.info("종료: {}", jp.getSignature().getName());
    }

    // 메서드 실행 전체를 감쌈 — 가장 강력
    @Around("execution(* com.example.service.*.*(..))")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            Object result = pjp.proceed(); // 실제 메서드 호출
            return result;
        } finally {
            log.info("{}ms", System.currentTimeMillis() - start);
        }
        // pjp.proceed() 호출 안 하면 원본 메서드 실행 안 됨
        // 반환값 교체, 예외 변환 모두 가능
    }
}
```

**실행 순서:**

```
Around (진입)
    └─ Before
         └─ 실제 메서드 실행
              └─ AfterReturning 또는 AfterThrowing
    └─ After
Around (종료)
```

---

### Pointcut 표현식

```java
// 기본 문법
execution([접근제한자] 반환타입 [패키지.클래스.]메서드명(파라미터))

// 예시
"execution(* com.example.service.*.*(..))"
//          ↑ 반환타입(*)  ↑ 패키지     ↑클래스(*) ↑메서드(*) ↑파라미터(..)

// 자주 쓰는 패턴
"execution(* com.example..*(..))"             // 하위 패키지 전체
"execution(public * *(..))"                   // public 메서드 전체
"execution(* *..*Service.*(..))"              // Service로 끝나는 클래스
"@annotation(org.springframework.transaction.annotation.Transactional)"
                                               // @Transactional 붙은 메서드
"@within(org.springframework.stereotype.Service)"
                                               // @Service 붙은 클래스 전체

// 조합
@Pointcut("execution(* com.example.service.*.*(..))")
private void serviceLayer() {}

@Pointcut("@annotation(com.example.annotation.Loggable)")
private void loggable() {}

@Before("serviceLayer() && loggable()") // AND 조합
public void before() { ... }
```

---

### @Transactional의 실제 동작 — AOP 적용 사례

`@Transactional`은 AOP로 구현된 대표적인 사례다.

```java
// 개발자가 작성하는 코드
@Service
public class OrderService {
    @Transactional
    public void createOrder(OrderRequest request) {
        orderRepository.save(new Order(request));
        paymentService.pay(request);
    }
}
```

**내부에서 실제로 일어나는 일:**

```java
// 스프링이 생성하는 CGLIB 프록시 (개념적 표현)
public class OrderService$$SpringCGLIB$$0 extends OrderService {
    private final TransactionInterceptor txInterceptor;

    @Override
    public void createOrder(OrderRequest request) {
        // 1. 트랜잭션 속성 조회
        TransactionAttribute txAttr = txInterceptor.getTransactionAttribute(method);

        // 2. 트랜잭션 시작
        TransactionStatus status = txManager.getTransaction(txAttr);

        try {
            // 3. 원본 메서드 실행
            super.createOrder(request);

            // 4. 커밋
            txManager.commit(status);
        } catch (Exception e) {
            // 5. 롤백
            txManager.rollback(status);
            throw e;
        }
    }
}
```

**Self Invocation이 왜 안 되는가 — 구조로 이해:**

```java
@Service
public class OrderService {
    @Transactional
    public void createOrder(OrderRequest request) {
        validate(request);      // this.validate() 호출
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void validate(OrderRequest request) {
        // 새 트랜잭션 기대
    }
}
```

```
외부 호출:
  orderService.createOrder()
       │
       ▼ (orderService는 프록시)
  Proxy.createOrder()
       │ 트랜잭션 시작
       ▼
  원본 OrderService.createOrder()
       │ this.validate() 호출
       │ this = 원본 OrderService (프록시 아님)
       ▼
  원본 OrderService.validate()
       │ 프록시를 거치지 않음
       │ → REQUIRES_NEW 트랜잭션 생성 안 됨
       ▼
  같은 트랜잭션에서 실행됨
```

---

### 여러 Aspect의 실행 순서 — @Order

```java
@Aspect
@Component
@Order(1)           // 숫자가 낮을수록 바깥쪽 (먼저 진입, 나중에 종료)
public class LoggingAspect { ... }

@Aspect
@Component
@Order(2)
public class TransactionAspect { ... }

@Aspect
@Component
@Order(3)           // 가장 안쪽 (나중에 진입, 먼저 종료)
public class SecurityAspect { ... }
```

```
요청 흐름:

LoggingAspect.around 진입       (@Order 1 — 바깥)
    TransactionAspect.around 진입  (@Order 2)
        SecurityAspect.around 진입    (@Order 3 — 안쪽)
            실제 메서드 실행
        SecurityAspect.around 종료
    TransactionAspect.around 종료
LoggingAspect.around 종료
```

**@Transactional과 @Async 순서가 중요한 이유:**

```java
@Service
public class OrderService {
    @Async           // 별도 스레드에서 실행
    @Transactional   // 트랜잭션 필요
    public void processAsync(OrderRequest request) { ... }
}

// @Async가 바깥(@Order 낮음)이면:
// 새 스레드 생성 → 그 스레드에서 @Transactional 적용 → 정상

// @Transactional이 바깥이면:
// 현재 스레드에서 트랜잭션 시작 → 새 스레드 생성
// → 새 스레드는 다른 트랜잭션 컨텍스트 → 원래 트랜잭션과 무관
```

---

## 3. 공식 근거

|주장|근거|
|---|---|
|JDK Dynamic Proxy vs CGLIB 선택 기준|Spring Framework Reference 6.x — _Proxying Mechanisms_|
|Spring Boot CGLIB 기본값|Spring Boot Reference — `spring.aop.proxy-target-class=true` 기본값|
|Self Invocation 미동작|Spring Framework Reference — _Understanding AOP Proxies_|
|Advice 실행 순서|Spring Framework Reference — _Advice Ordering_|
|@Transactional 내부 구현|Spring Framework 소스 `TransactionInterceptor`|
|Pointcut 표현식 문법|Spring Framework Reference — _Declaring a Pointcut_|

---

## 4. 이 설계를 이렇게 한 이유

**컴파일 타임 위빙(AspectJ) 대신 런타임 프록시를 선택한 이유:**

||AspectJ 컴파일 타임 위빙|스프링 런타임 프록시|
|---|---|---|
|성능|오버헤드 없음|프록시 호출 비용|
|적용 범위|모든 코드 (private 포함)|public 메서드만|
|Self Invocation|동작함|동작 안 함|
|설정 복잡도|높음 (AspectJ 컴파일러)|낮음|
|스프링 빈 의존|없음|스프링 컨테이너 필요|

대부분의 실무 요구사항은 런타임 프록시로 충분하고, 설정 복잡도가 낮아 도입 비용이 적다. 대신 Self Invocation, private 메서드 미적용이라는 제약을 안는다.

**잃는 것 — 구체적으로:**

|제약|원인|실무 영향|
|---|---|---|
|private 메서드 미적용|프록시 서브클래스가 오버라이드 불가|@Transactional을 private에 붙여도 동작 안 함|
|final 클래스/메서드 미적용|CGLIB 서브클래스 생성 불가|Kotlin data class에 @Transactional 주의|
|Self Invocation 미동작|this 참조가 프록시를 우회|같은 클래스 내 @Transactional 메서드 간 호출|
|프록시 호출 오버헤드|리플렉션 + 인터셉터 체인 순회|메서드 호출당 수 마이크로초 — 대부분 무시 가능|

---

## 5. 이어지는 개념

```
현재: AOP
        │
        ▼
다음: PSA (Portable Service Abstraction)  ← 바로 이어야 하는 이유:
                                            @Transactional은 AOP로 구현되지만
                                            PSA의 대표 사례이기도 하다.
                                            PlatformTransactionManager라는 추상화가
                                            JPA/JDBC/JTA를 교체 가능하게 만드는 원리,
                                            @Cacheable이 Redis/EhCache를 추상화하는 원리가
                                            AOP + PSA의 조합으로 완성된다.
```