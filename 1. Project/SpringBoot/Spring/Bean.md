#SpringBoot/Spring 
# Spring Bean — 프레임워크가 객체를 대신 관리하는 이유

---

## 1. Problem First

### Bean 없이, 즉 직접 객체를 관리하면 무슨 일이 벌어지는가

실무에서 가장 흔하게 터지는 시나리오 세 가지다.

---

**시나리오 1: 의존성 폭발 — 연결고리가 끊기는 순간 전체가 무너진다**

```java
// Bean 없이 직접 생성하는 세계
public class OrderController {
    private final OrderService orderService;

    public OrderController() {
        // OrderService가 뭘 필요로 하는지 OrderController가 알아야 한다
        DataSource dataSource = new HikariDataSource(hikariConfig); // 설정은 어디서?
        JdbcTemplate jdbcTemplate = new JdbcTemplate(dataSource);
        OrderRepository orderRepository = new OrderRepository(jdbcTemplate);
        PaymentClient paymentClient = new PaymentClient("https://payment.api", apiKey); // apiKey는?
        InventoryService inventoryService = new InventoryService(orderRepository);
        
        this.orderService = new OrderService(orderRepository, paymentClient, inventoryService);
    }
}
```

`OrderService` 생성자가 파라미터를 하나 추가하는 순간, `OrderService`를 직접 생성하는 **모든 곳**을 찾아서 수정해야 한다. 컴파일 오류가 나면 그나마 다행이고, 런타임에야 NPE로 터지는 경우도 생긴다.

이건 단순한 불편함이 아니다. **변경의 여파가 호출 그래프 전체로 전파된다**는 구조적 문제다. 의존성이 깊어질수록 지수적으로 커진다.

---

**시나리오 2: 인스턴스 난립 — 공유해야 할 자원이 공유되지 않는다**

```java
public class OrderService {
    private final OrderRepository orderRepository;
    
    public OrderService() {
        // 매번 새 Repository, 매번 새 DataSource, 매번 새 커넥션 풀
        this.orderRepository = new OrderRepository();
    }
}

public class PaymentService {
    private final OrderRepository orderRepository;
    
    public PaymentService() {
        // 또 다른 OrderRepository 인스턴스 → 또 다른 커넥션 풀
        this.orderRepository = new OrderRepository();
    }
}
```

`HikariCP` 커넥션 풀은 애플리케이션 전체에서 **하나**를 공유해야 한다. 위처럼 각자 생성하면 서비스 수만큼 커넥션 풀이 생기고, DB 커넥션 수가 `pool_size × service_count`로 폭증한다. RDS 기본 `max_connections`를 금방 초과한다.

---

**시나리오 3: 테스트 불가 — 구현체가 내부에 박혀 있다**

```java
public class OrderService {
    // new로 박아버리면 테스트 때 Mock으로 교체할 방법이 없다
    private final PaymentClient paymentClient = new PaymentClient();
    
    public void order(OrderRequest request) {
        paymentClient.charge(request.getAmount()); // 테스트할 때마다 실제 결제가 나간다
    }
}
```

인터페이스를 쓰더라도 생성 지점이 내부에 있으면 외부에서 교체할 수 없다. 테스트는 곧 실제 외부 시스템과의 통신을 의미하고, 이건 테스트가 아니라 통합 검증이다.

---

이 세 문제의 공통 원인은 하나다.

> **"객체가 자신의 의존성을 스스로 조달한다."**

이 책임을 외부로 역전시키는 것이 IoC(Inversion of Control)이고, Spring이 그 컨테이너 역할을 한다. Bean은 그 컨테이너가 관리하는 객체의 단위다.

---

## 2. Mechanics

### Bean이란 정확히 무엇인가

> "In Spring, the objects that form the backbone of your application and that are managed by the Spring IoC container are called beans. A bean is an object that is instantiated, assembled, and managed by a Spring IoC container."
> 
> — Spring Framework 공식 레퍼런스, [Core Technologies: The IoC Container](https://docs.spring.io/spring-framework/reference/core/beans/introduction.html)
> 
> _[번역] Spring에서 애플리케이션의 중추를 이루고 Spring IoC 컨테이너가 관리하는 객체를 Bean이라 한다. Bean은 Spring IoC 컨테이너가 인스턴스화하고, 조립하고, 관리하는 객체다._

Bean의 핵심은 **"내가 만드는 게 아니라 컨테이너가 만들어서 나에게 준다"** 는 것이다.

---

### ApplicationContext — 컨테이너의 실체

Spring IoC 컨테이너의 핵심 인터페이스는 `BeanFactory`이고, 실제 사용하는 것은 이를 확장한 `ApplicationContext`다.

```
BeanFactory
    └── ApplicationContext
            ├── ConfigurableApplicationContext
            │       └── AbstractApplicationContext
            │               ├── AnnotationConfigApplicationContext   ← Java Config
            │               └── AnnotationConfigServletWebServerApplicationContext  ← Spring Boot Web
```

Spring Boot의 `SpringApplication.run()`은 내부적으로 `AnnotationConfigServletWebServerApplicationContext`를 생성한다.

---

### Bean 등록에서 주입까지 — 내부 흐름

```
SpringApplication.run()
    │
    ├── 1. ApplicationContext 생성
    │
    ├── 2. BeanDefinition 스캔 및 등록
    │       ├── @ComponentScan → @Component, @Service, @Repository, @Controller 수집
    │       └── @Configuration + @Bean 메서드 파싱
    │           → BeanDefinitionRegistry에 BeanDefinition 등록
    │               (클래스 정보, 스코프, 생성자 파라미터 등 메타데이터)
    │
    ├── 3. BeanFactoryPostProcessor 실행
    │       └── @Value, @PropertySource 등 플레이스홀더 해석
    │
    ├── 4. Bean 인스턴스화 (기본: Eager initialization)
    │       ├── 생성자 분석 → 의존성 파악
    │       ├── 의존 Bean 먼저 생성 (재귀)
    │       └── 생성자/setter/필드 주입으로 의존성 연결
    │
    ├── 5. BeanPostProcessor 실행
    │       ├── @Autowired 처리 (AutowiredAnnotationBeanPostProcessor)
    │       ├── AOP 프록시 생성 (AbstractAutoProxyCreator)
    │       └── @PostConstruct 실행
    │
    └── 6. ApplicationContext 준비 완료
```

여기서 중요한 포인트: 4단계에서 생성되는 객체는 **싱글톤 스코프의 경우 딱 한 번만 생성되고 컨테이너가 그 레퍼런스를 보관**한다. 이후 `@Autowired`로 주입받는 것은 항상 그 동일한 인스턴스다.

---

### BeanDefinition — Bean의 설계도

Bean 자체보다 더 중요한 개념이다. 컨테이너는 클래스를 바로 인스턴스화하지 않는다. 먼저 `BeanDefinition`이라는 메타데이터 객체를 만들고, 이를 기반으로 인스턴스를 만든다.

```java
// BeanDefinition이 담고 있는 정보 (핵심만)
public interface BeanDefinition {
    String getBeanClassName();        // 어떤 클래스인가
    String getScope();                // singleton? prototype?
    boolean isLazyInit();             // 지연 초기화 여부
    ConstructorArgumentValues getConstructorArgumentValues(); // 생성자 인자
    MutablePropertyValues getPropertyValues();               // 프로퍼티
    String[] getDependsOn();          // 명시적 의존 순서
}
```

이 분리 덕분에 Spring은 **실제 인스턴스 없이도 의존 관계 그래프를 먼저 분석**할 수 있다. 순환 의존성 감지, 의존 순서 정렬이 여기서 이루어진다.

---

### Singleton 스코프의 실제 구현

기본 스코프인 싱글톤은 `DefaultSingletonBeanRegistry`가 관리한다. 내부에는 세 개의 Map이 있다. 이게 바로 Spring의 **순환 참조 문제를 다루는 3단계 캐시**다.

```java
// DefaultSingletonBeanRegistry 내부 (실제 Spring 소스 구조)
public class DefaultSingletonBeanRegistry {
    
    // 1차 캐시: 완전히 초기화된 Bean
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
    
    // 2차 캐시: 생성은 됐지만 아직 후처리 안 된 Bean (early reference)
    private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);
    
    // 3차 캐시: Bean을 만들 수 있는 팩토리 (AOP 프록시 생성에 사용)
    private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
}
```

A → B → A 순환참조 시 동작:

1. A 생성 시작 → `singletonFactories`에 A의 팩토리 등록
2. A의 의존성 B 생성 시작
3. B가 A를 필요로 함 → `singletonFactories`에서 A의 early reference 꺼냄
4. B 완성 → A에 주입 → A 완성

단, **생성자 주입 방식의 순환참조는 이 캐시로 해결 불가**하다. 생성자 단계에서는 팩토리에 등록할 타이밍 자체가 없기 때문이다. Spring Boot 2.6부터는 기본적으로 순환참조를 금지(`spring.main.allow-circular-references=false`)한다.

---

### Bean 스코프 종류

|스코프|인스턴스 생성 시점|소멸 시점|주요 사용처|
|---|---|---|---|
|`singleton`|컨테이너 시작 (기본)|컨테이너 종료|서비스, 레포지토리 등 대부분|
|`prototype`|`getBean()` 호출마다|**컨테이너가 관리 안 함** (호출자 책임)|매번 새 상태가 필요한 객체|
|`request`|HTTP 요청마다|요청 종료|Web 요청 범위 데이터|
|`session`|HTTP 세션마다|세션 만료|사용자 세션 데이터|

`prototype`은 **컨테이너가 소멸을 관리하지 않는다**는 점이 가장 큰 함정이다. `@PreDestroy`가 호출되지 않는다.

> "In contrast to the other scopes, Spring does not manage the complete lifecycle of a prototype bean: the container instantiates, configures, and otherwise assembles a prototype object, and hands it to the client, with no further record of that prototype instance."
> 
> — Spring Framework 공식 레퍼런스, [Bean Scopes: The Prototype Scope](https://docs.spring.io/spring-framework/reference/core/beans/factory-scopes.html#beans-factory-scopes-prototype)
> 
> _[번역] 다른 스코프와 달리, Spring은 prototype Bean의 전체 생명주기를 관리하지 않는다. 컨테이너는 prototype 객체를 인스턴스화하고 설정하고 조립하여 클라이언트에 넘긴 후, 해당 인스턴스에 대한 기록을 더 이상 보관하지 않는다._

---

### 의존성 주입 방식 세 가지와 실제 처리 방식

```java
// 1. 생성자 주입 — Spring 공식 권장
@Service
public class OrderService {
    private final OrderRepository orderRepository; // final 가능
    
    // @Autowired 생략 가능 (생성자가 하나일 때)
    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}

// 2. 수정자(Setter) 주입 — 선택적 의존성에 한정 사용
@Service
public class OrderService {
    private OrderRepository orderRepository;
    
    @Autowired(required = false)
    public void setOrderRepository(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }
}

// 3. 필드 주입 — Spring 팀이 권장하지 않음
@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository; // final 불가, 테스트 시 리플렉션 필요
}
```

생성자 주입을 권장하는 이유는 단순히 스타일 문제가 아니다:

- `final` 선언 → 컴파일러가 null 주입 불가를 보장
- 순환참조를 **런타임이 아닌 애플리케이션 시작 시점에** 감지
- Spring 없이 `new`로도 테스트 가능 → 테스트 코드에서 `@SpringBootTest` 불필요

> "The Spring team generally advocates constructor injection, as it lets you implement application components as immutable objects and ensures that required dependencies are not null."
> 
> — Spring Framework 공식 레퍼런스, [Dependencies: Dependency Injection](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html)
> 
> _[번역] Spring 팀은 일반적으로 생성자 주입을 권장한다. 이는 애플리케이션 컴포넌트를 불변 객체로 구현할 수 있게 하며, 필수 의존성이 null이 아님을 보장하기 때문이다._

---

### Bean 생명주기 콜백

```java
@Component
public class DatabaseConnectionPool {

    @PostConstruct          // 의존성 주입 완료 후, BeanPostProcessor에서 호출
    public void init() {
        // 커넥션 풀 초기화
        // 이 시점에 @Autowired 필드들은 모두 주입 완료된 상태
    }

    @PreDestroy             // 컨테이너 종료 시 호출 (singleton만)
    public void destroy() {
        // 커넥션 반납, 파일 핸들 닫기 등
    }
}
```

`@PostConstruct`는 JSR-250 표준 어노테이션이다 (Jakarta EE). Spring 전용이 아니므로 이식성이 있다.

> JSR-250 근거: `jakarta.annotation.PostConstruct` — [JSR 250: Common Annotations for the Java Platform](https://jcp.org/en/jsr/detail?id=250)

---

## 3. 공식 근거 요약

|주장|출처|
|---|---|
|Bean 정의 ("managed by IoC container")|Spring 공식 레퍼런스 — The IoC Container|
|생성자 주입 권장|Spring 공식 레퍼런스 — Dependency Injection|
|prototype 소멸 미관리|Spring 공식 레퍼런스 — The Prototype Scope|
|`@PostConstruct` 표준|JSR-250|
|순환참조 기본 금지 (2.6+)|[Spring Boot 2.6 Release Notes](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.6-Release-Notes#circular-references-between-beans-prohibited-by-default)|
|3단계 캐시 구조|OpenJDK 수준 대응: `DefaultSingletonBeanRegistry` Spring 소스코드 ([GitHub](https://github.com/spring-projects/spring-framework/blob/main/spring-beans/src/main/java/org/springframework/beans/factory/support/DefaultSingletonBeanRegistry.java))|

---

## 4. 이 설계를 이렇게 한 이유 — 트레이드오프

### 싱글톤을 기본으로 선택한 이유, 그리고 잃는 것

**얻는 것**: 객체 생성 비용 최소화, 메모리 공유, 상태 일관성

**잃는 것 — 이게 진짜 문제다**:

```java
// 이 코드는 멀티스레드 환경에서 데이터 오염이 발생한다
@Service  // 싱글톤
public class OrderService {
    private int orderCount = 0;  // ← 인스턴스 변수에 상태를 가지면 안 된다
    
    public void processOrder(Order order) {
        orderCount++;  // 여러 스레드가 동시에 접근 → race condition
    }
}
```

싱글톤 Bean은 **반드시 stateless(무상태)** 여야 한다. 이 제약을 모르면 멀티스레드 환경에서 데이터 오염이 발생한다. 상태가 필요하면 메서드 로컬 변수나 `ThreadLocal`, 또는 `prototype` 스코프를 사용해야 한다.

### Bean 컨테이너의 근본적인 비용

- **시작 시간**: 싱글톤 Bean은 모두 eager 초기화되므로 Bean이 많을수록 애플리케이션 시작이 느려진다. 대규모 Spring Boot 앱에서 시작 시간이 수십 초에 달하는 원인 중 하나다. `@Lazy`나 Spring Native(GraalVM)로 완화할 수 있지만 트레이드오프가 있다.
    
- **런타임 오버헤드**: AOP 프록시, `BeanPostProcessor` 체인 등 컨테이너 추상화 계층이 순수 Java 객체보다 무겁다. 대부분의 경우 무시할 수 있으나, 레이턴시가 극히 중요한 시스템에서는 고려 대상이다.
    
- **디버깅 난이도**: 실제로 주입된 객체가 원본인지 AOP 프록시인지 혼동이 생긴다. `@Transactional`이 걸린 Bean을 같은 클래스 내에서 직접 호출하면 트랜잭션이 적용되지 않는 이유가 이것이다 (Self-invocation 문제).
    

---

## 5. 이어지는 개념

|순서|개념|왜 이 순서인가|
|---|---|---|
|1|**@Component / @Service / @Repository / @Controller 계층 구조**|Bean을 등록하는 가장 기본 방법. 어노테이션 의미와 차이를 모르면 이후 개념 전부가 흔들린다|
|2|**@Configuration과 @Bean 메서드**|직접 Bean을 정의하는 방법. 외부 라이브러리 객체를 Bean으로 등록할 때 필수|
|3|**ApplicationContext와 BeanFactory 차이**|컨테이너 자체를 이해해야 Bean 생명주기 전체를 다룰 수 있다|
|4|**@Transactional과 AOP**|Bean에 AOP 프록시가 씌워지는 원리. 4단계에서 언급한 Self-invocation 함정의 실제 원인이 여기 있다|
|5|**@Scope("prototype")와 싱글톤 Bean 안에서의 prototype 주입 문제**|실무에서 잘못 쓰면 조용히 버그가 발생하는 영역. Lookup method injection, ObjectProvider 패턴까지 이어진다|