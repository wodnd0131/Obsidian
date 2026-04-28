— 기술이 바뀌어도 코드가 안 바뀐다

---

## 1. Problem First

### 추상화 없이 특정 기술에 직접 의존하면

**시나리오 A: 트랜잭션 관리**

```java
// JDBC 직접 사용 시
public class OrderService {
    private final Connection connection;

    public void createOrder(OrderRequest request) {
        try {
            connection.setAutoCommit(false);          // JDBC API
            // 비즈니스 로직
            connection.commit();                      // JDBC API
        } catch (SQLException e) {
            connection.rollback();                    // JDBC API
        }
    }
}

// JPA로 바꾸기로 결정
// → EntityTransaction으로 전부 교체
// → SQLException → 다른 예외 처리
// → OrderService 코드 전면 수정
// → OrderService를 사용하는 테스트 전부 수정
```

**시나리오 B: 캐시**

```java
// Redis 직접 사용
@Service
public class ProductService {
    private final RedisTemplate<String, Product> redisTemplate;

    public Product getProduct(Long id) {
        String key = "product:" + id;
        Product cached = redisTemplate.opsForValue().get(key);  // Redis API
        if (cached != null) return cached;

        Product product = productRepository.findById(id);
        redisTemplate.opsForValue().set(key, product, 1, TimeUnit.HOURS); // Redis API
        return product;
    }
}
// Caffeine(로컬 캐시)으로 교체 결정
// → CaffeineCache API로 전면 교체
// → ProductService 비즈니스 로직이 캐시 기술에 종속
```

**근본적인 문제:**

비즈니스 로직과 인프라 기술이 뒤섞인다. 기술 교체 비용이 비즈니스 로직 수정 비용이 된다. 단위 테스트 시 실제 인프라(Redis, DB)가 필요해진다.

---

## 2. Mechanics

### PSA의 구조

```
개발자 코드 (@Transactional, @Cacheable)
        │  어노테이션만 사용 — 기술 모름
        ▼
Spring Abstraction Layer (인터페이스)
        │  PlatformTransactionManager
        │  CacheManager
        │  PlatformMessageListener
        ▼
구체 구현체 (기술별)
        │  DataSourceTransactionManager (JDBC)
        │  JpaTransactionManager (JPA)
        │  RedisCacheManager (Redis)
        │  CaffeineCacheManager (Caffeine)
```

AOP가 "언제 끼어들 것인가"를 담당하고, PSA가 "끼어든 다음 어떤 추상화된 인터페이스로 실행할 것인가"를 담당한다.

---

### PlatformTransactionManager — 트랜잭션 추상화

**인터페이스 정의:**

```java
// spring-tx 모듈
public interface PlatformTransactionManager extends TransactionManager {
    TransactionStatus getTransaction(TransactionDefinition definition)
            throws TransactionException;
    void commit(TransactionStatus status) throws TransactionException;
    void rollback(TransactionStatus status) throws TransactionException;
}
```

`SQLException`, `PersistenceException` 같은 기술 종속 예외가 없다. `TransactionException` — 스프링의 unchecked 예외로 통일된다.

**구현체별 실제 동작:**

```java
// JDBC 환경
public class DataSourceTransactionManager implements PlatformTransactionManager {
    public TransactionStatus getTransaction(TransactionDefinition def) {
        Connection conn = dataSource.getConnection();
        conn.setAutoCommit(false);          // JDBC
        return new DataSourceTransactionObject(conn);
    }
    public void commit(TransactionStatus status) {
        ((DataSourceTransactionObject) status).getConnection().commit(); // JDBC
    }
}

// JPA 환경
public class JpaTransactionManager implements PlatformTransactionManager {
    public TransactionStatus getTransaction(TransactionDefinition def) {
        EntityManager em = emFactory.createEntityManager();
        em.getTransaction().begin();        // JPA
        return new JpaTransactionObject(em);
    }
    public void commit(TransactionStatus status) {
        ((JpaTransactionObject) status).getEntityManager()
            .getTransaction().commit();     // JPA
    }
}
```

**@Transactional은 이 인터페이스만 안다:**

```java
// TransactionInterceptor (AOP Advice) 내부 — 실제 소스 구조
public class TransactionInterceptor extends TransactionAspectSupport {
    public Object invoke(MethodInvocation invocation) throws Throwable {
        // PlatformTransactionManager 인터페이스만 사용
        // JDBC인지 JPA인지 모른다
        TransactionStatus status =
            transactionManager.getTransaction(txAttribute);  // 추상화
        try {
            Object result = invocation.proceed();
            transactionManager.commit(status);               // 추상화
            return result;
        } catch (Throwable ex) {
            transactionManager.rollback(status);             // 추상화
            throw ex;
        }
    }
}
```

**기술 교체 시 변경되는 것:**

```java
// JDBC → JPA 전환 시

// 변경 전
@Bean
public PlatformTransactionManager transactionManager(DataSource dataSource) {
    return new DataSourceTransactionManager(dataSource);
}

// 변경 후 — 이것만 바꾸면 됨
@Bean
public PlatformTransactionManager transactionManager(EntityManagerFactory emf) {
    return new JpaTransactionManager(emf);
}

// OrderService는 한 글자도 안 바뀐다
@Service
public class OrderService {
    @Transactional  // 그대로
    public void createOrder(OrderRequest request) { ... }
}
```

---

### TransactionSynchronizationManager — 트랜잭션 컨텍스트 전파

같은 트랜잭션 안에서 여러 빈이 같은 커넥션을 공유하는 원리.

```java
// TransactionSynchronizationManager 내부
public abstract class TransactionSynchronizationManager {
    // ThreadLocal로 현재 스레드의 트랜잭션 자원 관리
    private static final ThreadLocal<Map<Object, Object>> resources =
        new NamedThreadLocal<>("Transactional resources");

    // 현재 스레드에 커넥션 바인딩
    public static void bindResource(Object key, Object value) {
        resources.get().put(key, value);
    }

    // 현재 스레드에서 커넥션 조회
    public static Object getResource(Object key) {
        return resources.get().get(key);
    }
}
```

```
@Transactional createOrder() 시작
    │
    ▼
JpaTransactionManager.getTransaction()
    → Connection 획득
    → TransactionSynchronizationManager.bindResource(dataSource, conn)
    │  현재 스레드에 커넥션 등록
    ▼
orderRepository.save()   ← SimpleJpaRepository 내부
    → TransactionSynchronizationManager.getResource(dataSource)
    → 새 커넥션 안 만들고 기존 커넥션 재사용
    ▼
paymentRepository.save() ← 같은 과정
    → 같은 커넥션 재사용
    ▼
트랜잭션 커밋 → 커넥션 반환 → ThreadLocal 정리
```

---

### CacheManager — 캐시 추상화

```java
public interface CacheManager {
    Cache getCache(String name);
    Collection<String> getCacheNames();
}

public interface Cache {
    String getName();
    Object getNativeCache();                    // 실제 캐시 객체 (Redis, Caffeine 등)
    ValueWrapper get(Object key);
    <T> T get(Object key, Class<T> type);
    void put(Object key, Object value);
    void evict(Object key);
}
```

**@Cacheable 내부 동작:**

```java
// CacheInterceptor (AOP Advice) 개념적 동작
public Object invoke(MethodInvocation invocation) throws Throwable {
    // 1. 캐시 조회
    Cache cache = cacheManager.getCache("products"); // 추상화
    Cache.ValueWrapper cached = cache.get(key);      // 추상화

    if (cached != null) {
        return cached.get(); // 캐시 히트 → 메서드 실행 안 함
    }

    // 2. 캐시 미스 → 실제 메서드 실행
    Object result = invocation.proceed();

    // 3. 결과 캐시 저장
    cache.put(key, result);  // 추상화
    return result;
}
```

**구현체 교체:**

```java
// Redis 캐시
@Bean
public CacheManager cacheManager(RedisConnectionFactory factory) {
    return RedisCacheManager.builder(factory)
        .cacheDefaults(RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofHours(1)))
        .build();
}

// Caffeine 캐시로 교체 — ProductService 코드 변경 없음
@Bean
public CacheManager cacheManager() {
    CaffeineCacheManager manager = new CaffeineCacheManager();
    manager.setCaffeine(Caffeine.newBuilder().expireAfterWrite(1, TimeUnit.HOURS));
    return manager;
}

// 서비스 코드 — 그대로
@Service
public class ProductService {
    @Cacheable(value = "products", key = "#id")
    public Product getProduct(Long id) {
        return productRepository.findById(id).orElseThrow();
    }

    @CacheEvict(value = "products", key = "#product.id")
    public void updateProduct(Product product) {
        productRepository.save(product);
    }
}
```

---

### MessageListener — 메시지 추상화

```java
// Kafka, RabbitMQ, ActiveMQ 모두 같은 어노테이션
@Component
public class OrderEventListener {

    // Kafka
    @KafkaListener(topics = "orders")
    public void handleKafka(OrderEvent event) {
        process(event);
    }

    // RabbitMQ — 어노테이션만 다름, 메서드 내부는 동일
    @RabbitListener(queues = "orders")
    public void handleRabbit(OrderEvent event) {
        process(event);
    }

    private void process(OrderEvent event) {
        // 메시징 기술에 무관한 비즈니스 로직
    }
}
```

---

### Spring MVC — 서블릿 추상화

PSA는 트랜잭션/캐시에만 해당되는 이야기가 아니다. `DispatcherServlet`이 `HttpServlet`을 추상화하는 것도 PSA다.

```java
// HttpServlet 직접 사용 시
public class OrderServlet extends HttpServlet {
    protected void doGet(HttpServletRequest req, HttpServletResponse res) {
        String id = req.getParameter("id");       // 서블릿 API
        // 응답 직접 조립
        res.setContentType("application/json");   // 서블릿 API
        res.getWriter().write("{\"id\":" + id + "}");
    }
}

// Spring MVC — 서블릿 API 모름
@RestController
public class OrderController {
    @GetMapping("/orders/{id}")
    public OrderResponse getOrder(@PathVariable Long id) {
        return orderService.getOrder(id); // 순수 Java 객체 반환
        // HttpServletRequest/Response 없음
        // JSON 변환 — MessageConverter가 처리
        // 서블릿 API 종속 없음 → 단위 테스트 가능
    }
}
```

---

### 예외 추상화 — DataAccessException

JDBC, JPA, MongoDB 등 각 기술의 예외를 스프링 예외 계층으로 통일한다.

```java
// 기술별 예외가 다르다
// JDBC:   throws SQLException (checked)
// JPA:    throws PersistenceException (unchecked)
// MongoDB: throws MongoException (unchecked)

// 스프링 추상화 — DataAccessException 계층으로 통일
DataAccessException
    ├─ NonTransientDataAccessException
    │       ├─ DataIntegrityViolationException  // 중복키, FK 위반
    │       ├─ BadSqlGrammarException           // SQL 문법 오류
    │       └─ PermissionDeniedDataAccessException
    └─ TransientDataAccessException
            ├─ QueryTimeoutException            // 타임아웃
            └─ DeadlockLoserDataAccessException // 데드락

// 서비스 코드 — 기술 무관
@Service
public class OrderService {
    public void createOrder(OrderRequest request) {
        try {
            orderRepository.save(new Order(request));
        } catch (DataIntegrityViolationException e) {
            // JDBC든 JPA든 MongoDB든 동일하게 처리
            throw new DuplicateOrderException(e);
        }
    }
}
```

---

## 3. 공식 근거

|주장|근거|
|---|---|
|PlatformTransactionManager 설계 의도|Spring Framework Reference 6.x — _Spring Framework's consistent abstraction for transaction management_|
|TransactionSynchronizationManager ThreadLocal 사용|Spring Framework 소스 `TransactionSynchronizationManager`|
|@Cacheable 동작 원리|Spring Framework Reference — _Declarative Annotation-based Caching_|
|DataAccessException 계층|Spring Framework Reference — _Consistent Exception Hierarchy_|
|PSA 개념 명명|Toby Lee, _토비의 스프링 3.1_ — PSA 용어 정립|

---

## 4. 이 설계를 이렇게 한 이유

**DIP(의존성 역전 원칙)의 프레임워크 레벨 적용이다.**

```
기존:  비즈니스 로직 → 구체 기술 (JDBC, Redis)
PSA:  비즈니스 로직 → 추상화 인터페이스 ← 구체 기술
```

얻는 것은 명확하다. 기술 교체 시 비즈니스 코드를 건드리지 않아도 된다. 테스트 시 실제 인프라 없이 Mock 구현체로 대체 가능하다.

**잃는 것 — 구체적으로:**

|비용|내용|
|---|---|
|추상화 누수|추상화가 완전하지 않을 때 기술 종속 코드가 결국 새어 나옴. JPA의 `flush()`, Redis의 `TTL` 설정은 추상화 밖|
|디버깅 난이도|@Transactional이 안 될 때 AOP 프록시 → TransactionInterceptor → PlatformTransactionManager 계층을 전부 추적해야 함|
|성능 최적화 제약|추상화 레이어를 거치면 기술 고유의 최적화 기능 사용이 어려움. Redis Pipeline, Kafka batch commit 등은 직접 구현체 사용 필요|
|학습 비용|실제로 어떤 구현체가 동작하는지 모르면 장애 원인 파악이 어려움|

---

## 5. 전체 스프링 핵심 원리 완성

여기까지 오면 다섯 개의 축이 어떻게 연결되는지 보인다.

```
IoC / DI
  "객체 생성과 의존성 주입을 컨테이너가 담당"
        │
        │ 빈을 만드는 무대
        ▼
ApplicationContext
  "BeanDefinition 수집 → refresh() → 빈 생성 → BeanPostProcessor"
        │
        │ BeanPostProcessor가 프록시를 만든다
        ▼
AOP
  "프록시로 횡단 관심사를 분리"
        │
        │ 프록시 안에서 추상화된 인터페이스를 호출한다
        ▼
PSA
  "@Transactional → PlatformTransactionManager"
  "@Cacheable → CacheManager"
        │
        │ 빈 생성과 소멸, 의존성 주입 시점
        ▼
Bean Lifecycle
  "언제 만들어지고, 언제 초기화되고, 언제 소멸하는가"
```

```
실제 요청 흐름에서 다섯 개가 동시에 작동하는 순간:

HTTP 요청 → @Transactional createOrder() 호출
                │
                │  IoC/DI: OrderService 빈이 컨테이너에서 관리됨
                │  ApplicationContext: refresh()로 빈과 프록시가 준비됨
                │  AOP: 프록시가 요청을 가로챔
                │  PSA: PlatformTransactionManager로 트랜잭션 시작
                │  Bean Lifecycle: @PostConstruct로 초기화된 커넥션풀 사용
                ▼
             비즈니스 로직 실행
```