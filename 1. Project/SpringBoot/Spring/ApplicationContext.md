# ApplicationContext — IoC 컨테이너의 실체

---

## 1. Problem First

### BeanFactory만 있었다면

```java
// BeanFactory — 스프링의 가장 기본적인 컨테이너
BeanFactory factory = new XmlBeanFactory(new ClassPathResource("beans.xml"));

// 문제 1: Lazy Loading만 지원
// getBean() 호출 시점에 처음 생성 → 시작 시점에 설정 오류를 모름
OrderService service = (OrderService) factory.getBean("orderService");
// 여기서 처음 생성 시도 → XML 설정 오류가 있으면 런타임에 터짐
// 서버 시작 후 첫 요청에서 장애 발생

// 문제 2: 국제화(i18n) 없음
// 문제 3: 이벤트 발행/구독 없음
// 문제 4: 환경별 설정(dev/prod) 분리 없음
// 문제 5: 리소스 로딩 추상화 없음
```

**BeanFactory는 빈을 만들고 관리하는 것만 한다.** 실제 애플리케이션에 필요한 나머지는 전부 직접 구현해야 했다.

---

## 2. Mechanics

### 인터페이스 계층 구조

```
BeanFactory                         // 빈 조회/생성만
    └─ ApplicationContext           // BeanFactory + 부가기능
            ├─ MessageSource        // 국제화 (i18n)
            ├─ ApplicationEventPublisher  // 이벤트 발행
            ├─ EnvironmentCapable   // 환경변수/프로파일
            └─ ResourcePatternResolver   // 리소스 로딩 추상화
```

```java
// ApplicationContext가 구현하는 인터페이스 전체
public interface ApplicationContext extends
    EnvironmentCapable,         // getEnvironment()
    ListableBeanFactory,        // getBeanNamesForType() 등
    HierarchicalBeanFactory,    // getParentBeanFactory()
    MessageSource,              // getMessage() — i18n
    ApplicationEventPublisher,  // publishEvent()
    ResourcePatternResolver {   // getResources()
}
```

---

### 실제 구현체 계층

```
ApplicationContext (interface)
    │
    ├─ ConfigurableApplicationContext (interface)
    │       refresh() / close() 추가
    │
    └─ AbstractApplicationContext (abstract)
            refresh() 구현 — 핵심 템플릿 메서드
            │
            ├─ GenericApplicationContext
            │       └─ AnnotationConfigApplicationContext
            │              (순수 자바 환경, 테스트에서 자주 사용)
            │
            └─ AbstractRefreshableApplicationContext
                    └─ AbstractXmlApplicationContext
                            └─ ClassPathXmlApplicationContext (레거시)
```

**Spring Boot 환경:**

```
SpringApplication.run()
    │
    ├─ 서블릿 환경 감지
    │       AnnotationConfigServletWebServerApplicationContext
    │               └─ 내장 Tomcat 생성 + 시작까지 담당
    │
    └─ 리액티브 환경
            AnnotationConfigReactiveWebServerApplicationContext
```

---

### refresh() — 컨테이너가 살아나는 과정

`ApplicationContext`의 핵심은 `refresh()`다. 컨테이너 초기화의 모든 것이 이 메서드 하나에 담겨있다.

```java
// AbstractApplicationContext.refresh() — 실제 소스 구조
public void refresh() throws BeansException {
    synchronized (this.startupShutdownMonitor) {

        // 1. 준비 단계
        prepareRefresh();
        // → Environment 초기화
        // → 프로퍼티 소스 등록
        // → 필수 프로퍼티 검증

        // 2. BeanFactory 준비
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        // → BeanDefinition 수집 시작
        // → @Component 스캔, @Bean 메서드 파싱

        prepareBeanFactory(beanFactory);
        // → 기본 BeanPostProcessor 등록
        // → ApplicationContext 자체를 빈으로 등록

        // 3. BeanFactoryPostProcessor 실행
        invokeBeanFactoryPostProcessors(beanFactory);
        // → @Configuration 클래스 처리
        // → @PropertySource 로딩
        // → ${...} 플레이스홀더 치환

        // 4. BeanPostProcessor 등록
        registerBeanPostProcessors(beanFactory);
        // → AOP 프록시 생성기 등록
        // → @Autowired 처리기 등록
        // → @PostConstruct 처리기 등록

        // 5. 부가기능 초기화
        initMessageSource();        // i18n
        initApplicationEventMulticaster(); // 이벤트 시스템

        // 6. 서브클래스 확장 지점
        onRefresh();
        // → Spring Boot: 여기서 내장 Tomcat 시작

        // 7. 이벤트 리스너 등록
        registerListeners();

        // 8. 싱글톤 빈 전부 생성
        finishBeanFactoryInitialization(beanFactory);
        // → Lazy가 아닌 모든 싱글톤 빈 인스턴스화
        // → 의존성 주입
        // → @PostConstruct 실행

        // 9. 완료
        finishRefresh();
        // → LifecycleProcessor 시작
        // → ContextRefreshedEvent 발행
    }
}
```

**`finishBeanFactoryInitialization`이 핵심:**

```
여기서 앞서 공부한 Bean Lifecycle 전체가 실행된다

BeanDefinition 목록 순회
    → 싱글톤이고 LazyInit 아닌 것들
    → 의존관계 위상 정렬
    → 순서대로 인스턴스 생성
    → BeanPostProcessor 적용 (AOP 프록시 포함)
    → 완성된 빈을 singletonObjects Map에 등록
```

---

### Environment와 PropertySource

`ApplicationContext`는 `EnvironmentCapable`을 구현한다. 설정값이 어디서 오는지 추상화하는 레이어다.

```
Environment
    │
    └─ PropertySource 목록 (우선순위 순)
            1. ServletConfig 파라미터
            2. ServletContext 파라미터
            3. JNDI
            4. JVM 시스템 프로퍼티 (-Dserver.port=8080)
            5. OS 환경변수 (SERVER_PORT=8080)
            6. application-{profile}.properties
            7. application.properties
```

```java
// 우선순위가 높은 것이 낮은 것을 덮어씀
// application.properties: server.port=8080
// JVM 옵션: -Dserver.port=9090
// → 실제 사용값: 9090

// @Value 동작 원리
@Value("${server.port}")
private int port;
// BeanFactoryPostProcessor 단계에서
// Environment.getProperty("server.port") 조회
// → PropertySource 목록을 우선순위 순으로 탐색
// → 찾은 값으로 BeanDefinition의 "${server.port}" 치환
```

**Profile:**

```java
// 환경별 빈 분리
@Configuration
@Profile("prod")
public class ProdDataSourceConfig {
    @Bean
    public DataSource dataSource() {
        // RDS 연결
    }
}

@Configuration
@Profile("test")
public class TestDataSourceConfig {
    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(H2).build();
    }
}

// 활성화
// -Dspring.profiles.active=prod
// 또는 application.properties: spring.profiles.active=prod
```

---

### ApplicationEvent — 이벤트 시스템

컨텍스트 내부에서 빈 간 직접 의존 없이 통신하는 방법이다.

```java
// 이벤트 정의
public class OrderCreatedEvent {
    private final Order order;
    public OrderCreatedEvent(Order order) { this.order = order; }
    public Order getOrder() { return order; }
}

// 이벤트 발행
@Service
public class OrderService {
    private final ApplicationEventPublisher publisher;

    public OrderService(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    public void createOrder(OrderRequest request) {
        Order order = orderRepository.save(new Order(request));
        publisher.publishEvent(new OrderCreatedEvent(order));
        // OrderService는 이후에 무슨 일이 일어나는지 모름
    }
}

// 이벤트 수신
@Component
public class NotificationHandler {
    @EventListener
    public void handle(OrderCreatedEvent event) {
        // 주문 생성 후 알림 전송
        sendNotification(event.getOrder());
    }
}

@Component
public class InventoryHandler {
    @EventListener
    public void handle(OrderCreatedEvent event) {
        // 주문 생성 후 재고 차감
        decreaseStock(event.getOrder());
    }
}
```

**기본 동작은 동기/같은 스레드:**

```java
// publishEvent() 호출 스레드에서 리스너가 순차 실행됨
// OrderService.createOrder() 스레드
//   → NotificationHandler.handle() 실행 (완료 대기)
//   → InventoryHandler.handle() 실행 (완료 대기)
//   → createOrder() 계속 진행

// 비동기로 바꾸려면
@EventListener
@Async  // 별도 스레드에서 실행
public void handle(OrderCreatedEvent event) { ... }
```

**트랜잭션 연동:**

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handle(OrderCreatedEvent event) {
    // 트랜잭션이 커밋된 후에만 실행
    // 주문 저장 롤백되면 이 리스너는 실행 안 됨
    sendNotification(event.getOrder());
}
```

---

### Parent-Child Context 계층구조

Spring MVC 전통 환경에서 컨텍스트가 두 개 존재했다.

```
Root ApplicationContext (부모)
    │  Service, Repository, 인프라 빈
    │  ContextLoaderListener가 생성
    │
    └─ WebApplicationContext (자식) — DispatcherServlet당 1개
            Controller, HandlerMapping, ViewResolver
            자식은 부모 빈을 볼 수 있음
            부모는 자식 빈을 볼 수 없음
```

```java
// 자식에서 부모 빈 접근 — 가능
@Controller
public class OrderController {
    @Autowired
    private OrderService orderService; // Root Context의 빈 — 접근 가능
}

// 부모에서 자식 빈 접근 — 불가
@Service
public class OrderService {
    @Autowired
    private OrderController controller; // WebApplicationContext의 빈 — 실패
}
```

**Spring Boot에서는 이 계층구조가 기본적으로 없다:**

```
SpringApplication.run()
    → AnnotationConfigServletWebServerApplicationContext 단일 컨텍스트
    → Controller, Service, Repository 전부 한 컨텍스트에

계층구조가 필요한 경우만 명시적으로 구성
(멀티 DispatcherServlet, 모듈 분리 등)
```

---

### Spring Boot의 자동 구성과 ApplicationContext

`@SpringBootApplication` 내부:

```java
@SpringBootApplication
= @SpringBootConfiguration    // @Configuration 포함
+ @EnableAutoConfiguration    // 자동 구성 활성화
+ @ComponentScan              // 현재 패키지부터 스캔
```

**자동 구성 동작:**

```
spring-boot-autoconfigure.jar
    └─ META-INF/spring/
            org.springframework.boot.autoconfigure.AutoConfiguration.imports
            → 자동 구성 클래스 목록 (수백 개)

SpringApplication.run()
    → AutoConfigurationImportSelector
    → imports 파일에서 자동 구성 클래스 목록 로딩
    → @Conditional 평가
            DataSource 자동 구성:
                @ConditionalOnClass(DataSource.class)  // 클래스패스에 있는가
                @ConditionalOnMissingBean(DataSource.class) // 이미 등록된 빈 없는가
            → 조건 충족 시 BeanDefinition 등록
            → 조건 미충족 시 스킵
```

```java
// 커스텀 빈이 자동 구성보다 우선되는 원리
@Configuration
public class MyDataSourceConfig {
    @Bean
    public DataSource dataSource() { // 직접 등록
        return new HikariDataSource(...);
    }
}

// 자동 구성의 DataSourceAutoConfiguration:
// @ConditionalOnMissingBean(DataSource.class)
// → 이미 DataSource 빈이 있으므로 자동 구성 스킵
// → 내가 만든 빈이 사용됨
```

---

## 3. 공식 근거

|주장|근거|
|---|---|
|ApplicationContext 인터페이스 계층|Spring Framework Reference 6.x — _The ApplicationContext_|
|refresh() 단계별 동작|Spring Framework 소스 `AbstractApplicationContext.refresh()`|
|PropertySource 우선순위|Spring Boot Reference — _Externalized Configuration_ 우선순위 목록|
|이벤트 동기 실행 기본값|Spring Framework Reference — _Standard and Custom Events_|
|@TransactionalEventListener|Spring Framework Reference — _Transaction-bound Events_|
|Parent-Child Context|Spring Framework Reference — _Additional Capabilities of the ApplicationContext_|
|자동 구성 @Conditional|Spring Boot Reference — _Understanding Auto-configured Beans_|

---

## 4. 이 설계를 이렇게 한 이유

**BeanFactory를 상속하지 않고 감싼 이유:**

ApplicationContext는 BeanFactory를 직접 들고 있다 (컴포지션). 상속했다면 BeanFactory의 내부 구현에 종속된다. 컴포지션으로 BeanFactory 구현체를 교체할 수 있는 유연성을 확보했다.

**refresh()를 별도 메서드로 분리한 이유:**

생성자에서 모든 초기화를 하면 서브클래스가 개입할 수 없다. `refresh()`를 분리함으로써 Spring Boot가 `onRefresh()`에서 내장 Tomcat을 시작하는 확장이 가능해졌다.

**잃는 것:**

|비용|내용|
|---|---|
|시작 시간|`finishBeanFactoryInitialization`에서 모든 싱글톤 빈을 즉시 생성. 빈이 많을수록 시작 느림|
|메모리|모든 싱글톤 빈이 컨테이너 종료까지 메모리 점유|
|이벤트 동기 실행|기본이 동기라 리스너가 느리면 발행자도 블로킹됨|
|자동 구성 디버깅|어떤 자동 구성이 적용됐는지 파악하려면 `--debug` 플래그 또는 Actuator 필요|

---

## 5. 이어지는 개념

```
현재: ApplicationContext
        │
        ▼
다음: AOP                   ← 바로 이어야 하는 이유:
                              BeanPostProcessor가 ApplicationContext의
                              refresh() 안에서 등록되고 실행된다는 걸 알았다.
                              AOP 프록시가 정확히 그 시점에 만들어진다.
                              ApplicationContext 없이는
                              "왜 @Transactional이 같은 클래스 내부 호출에서 안 되는가"를
                              설명할 수 없다.
```