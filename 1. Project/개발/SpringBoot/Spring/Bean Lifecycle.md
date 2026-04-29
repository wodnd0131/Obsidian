
— 스프링이 객체를 언제 만들고, 어떻게 관리하고, 언제 죽이는가
## 1. Problem First
### 라이프사이클 관리가 없다면
**시나리오 A: 초기화 순서 보장 불가**

```java
// 라이프사이클 없이 — 생성자에서 모든 걸 하려 할 때
@Service
public class DatabaseConnectionPool {
    private final DataSource dataSource;
    private Connection[] pool;

    public DatabaseConnectionPool(DataSource dataSource) {
        this.dataSource = dataSource;
        // 문제: 생성자 실행 시점에 DataSource가 완전히 초기화됐다는 보장이 없다
        // 다른 빈의 @PostConstruct가 아직 실행 안 됐을 수 있음
        this.pool = initPool(); // 여기서 터질 수 있음
    }
}
```

**시나리오 B: 리소스 누수**

```java
// 라이프사이클 종료 훅이 없다면
@Service
public class KafkaConsumerService {
    private final KafkaConsumer<String, String> consumer;

    public KafkaConsumerService() {
        this.consumer = new KafkaConsumer<>(props);
        consumer.subscribe(List.of("orders"));
    }
    // 애플리케이션 종료 시 consumer.close() 호출할 방법이 없음
    // → 브로커 측에서 세션 타임아웃까지 커넥션 점유
    // → 재시작 시 리밸런싱 지연
}
```

**시나리오 C: 의존성 주입 완료 전 사용**

```java
@Service
public class CacheService {
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    // 생성자에서 redisTemplate 사용 시도
    public CacheService() {
        redisTemplate.opsForValue().set("init", "value"); // NPE
        // @Autowired 필드 주입은 생성자 실행 이후에 일어남
    }
}
```

---

## 2. Mechanics

### 전체 라이프 사이클 지도

```
스프링 컨테이너 시작
        │
        ▼
① BeanDefinition 수집 및 등록
        │  @Component 스캔, @Bean 메서드, XML 파싱
        │  → BeanDefinitionRegistry에 메타데이터만 저장
        │    (아직 인스턴스 없음)
        ▼
② BeanFactoryPostProcessor 실행
        │  BeanDefinition 자체를 수정할 수 있는 시점
        │  ex) PropertySourcesPlaceholderConfigurer
        │      → @Value("${db.url}") 의 "${db.url}"을 실제 값으로 치환
        ▼
③ 인스턴스 생성 (Instantiation)
        │  생성자 호출 — new OrderService(...)
        ▼
④ 의존성 주입 (Populate Properties)
        │  생성자 주입: ③과 동시에 발생
        │  필드/수정자 주입: ③ 이후 리플렉션으로 주입
        ▼
⑤ BeanPostProcessor — postProcessBeforeInitialization()
        │  @PostConstruct 처리 (CommonAnnotationBeanPostProcessor)
        │  초기화 메서드 호출 이전 개입 가능
        ▼
⑥ 초기화 (Initialization)
        │  @PostConstruct 메서드 실행
        │  InitializingBean.afterPropertiesSet()
        │  @Bean(initMethod = "init")
        ▼
⑦ BeanPostProcessor — postProcessAfterInitialization()
        │  AOP 프록시 생성이 여기서 일어남  ← 중요
        │  실제로 컨테이너에 등록되는 것은
        │  원본 빈이 아니라 프록시일 수 있음
        ▼
⑧ 빈 사용 (사용 가능 상태)
        │
        ▼
스프링 컨테이너 종료
        │
        ▼
⑨ 소멸 전 처리 (Destruction)
        │  @PreDestroy 메서드 실행
        │  DisposableBean.destroy()
        │  @Bean(destroyMethod = "close")
        ▼
⑩ 인스턴스 소멸
```

---
```
③④  원본 객체에 재료를 채운다   (생성 + 의존성 주입)
⑤⑥  원본 객체를 완성한다        (초기화)
⑦   완성된 원본을 포장한다       (프록시)
```
### ② BeanFactoryPostProcessor

BeanDefinition이 등록된 후, 실제 빈 생성 전에 개입한다. **빈 인스턴스가 아니라 빈 설계도를 수정하는 시점.**

```java
// PropertySourcesPlaceholderConfigurer 동작 원리
// @Value("${server.port}") 가 어떻게 8080으로 바뀌는가

// 1. BeanDefinition에는 아직 리터럴 "${server.port}" 저장됨
// 2. PropertySourcesPlaceholderConfigurer.postProcessBeanFactory() 실행
// 3. 모든 BeanDefinition 순회하며 ${...} 패턴 탐색
// 4. Environment의 PropertySource에서 실제 값 조회
// 5. BeanDefinition의 값을 "8080"으로 교체
// 6. 이후 빈 생성 시 이미 치환된 값으로 초기화

@FunctionalInterface
public interface BeanFactoryPostProcessor {
    void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory);
    // beanFactory.getBeanDefinition("orderService") 로 설계도에 직접 접근 가능
}
```

---

### ⑤⑦ BeanPostProcessor — AOP가 여기서 만들어진다

모든 빈의 생성 전후에 개입할 수 있는 훅. 스프링 내부에서 가장 많이 활용하는 확장 포인트다.

```java
public interface BeanPostProcessor {
    // 초기화(⑥) 이전 — @PostConstruct 처리가 여기서 일어남
    default Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean; // 원본 반환 또는 다른 객체로 교체 가능
    }

    // 초기화(⑥) 이후 — AOP 프록시 생성이 여기서 일어남
    default Object postProcessAfterInitialization(Object bean, String beanName) {
        return bean; // 프록시 객체를 반환하면 컨테이너는 프록시를 등록함
    }
}
```

**AOP 프록시 생성 흐름:**

```java
// AbstractAutoProxyCreator (BeanPostProcessor 구현체)
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) {
    // 이 빈에 적용할 Advisor(AOP 설정)가 있는가?
    if (이 빈에_적용할_AOP가_있다()) {
        // 원본 빈을 감싸는 프록시 생성
        return createProxy(bean); // 컨테이너에는 프록시가 등록됨
    }
    return bean; // AOP 대상 아니면 원본 그대로
}
```

```
@Transactional OrderService 빈 생성 시:

실제 등록 과정:
OrderService 인스턴스 생성 (③)
        │
        ▼
의존성 주입 (④)
        │
        ▼
@PostConstruct 실행 (⑥)
        │
        ▼
AbstractAutoProxyCreator.postProcessAfterInitialization() (⑦)
        │  "@Transactional 있음 → 프록시 필요"
        ▼
ProxyOrderService (원본 래핑) 생성
        │
        ▼
컨테이너에 ProxyOrderService 등록
        │
        ▼
@Autowired OrderService orderService
        → 주입되는 것은 ProxyOrderService
```

**이 구조가 만드는 유명한 문제 — Self Invocation:**

```java
@Service
public class OrderService {
    @Transactional
    public void order(OrderRequest req) {
        validate(req);
        save(req);
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void validate(OrderRequest req) {
        // order()에서 this.validate()로 호출 시
        // this = 원본 OrderService (프록시 아님)
        // → @Transactional 동작 안 함
    }
}

// 외부에서 호출 시:
// orderService.order()
//   → 프록시가 가로채서 트랜잭션 시작
//   → 원본 order() 실행
//   → this.validate() → 프록시 거치지 않음 → 새 트랜잭션 없음
```

---

### ⑥ 초기화 메서드 — 세 가지 방식과 실행 순서

```java
@Service
public class DataSourcePool implements InitializingBean {

    @PostConstruct                          // 실행 순서 1
    public void postConstruct() {
        System.out.println("@PostConstruct");
        // 의존성 주입 완료 후 첫 번째 실행
        // JSR-250 표준 → 스프링 종속 없음
    }

    @Override
    public void afterPropertiesSet() {      // 실행 순서 2
        System.out.println("afterPropertiesSet");
        // InitializingBean 인터페이스 구현
        // 스프링 인터페이스에 종속됨 → 비권장
    }
}

@Configuration
public class AppConfig {
    @Bean(initMethod = "init")              // 실행 순서 3
    public DataSourcePool dataSourcePool() {
        return new DataSourcePool();
    }
}
```

**권장 우선순위:**

|방식|권장도|이유|
|---|---|---|
|`@PostConstruct`|✅ 권장|JSR-250 표준, 스프링 종속 없음|
|`@Bean(initMethod)`|✅ 외부 라이브러리|소스 수정 불가한 클래스에 적용|
|`InitializingBean`|⚠️ 비권장|스프링 인터페이스 직접 구현|

---

### Scope — 라이프사이클이 달라진다

```java
// Singleton (기본) — 컨테이너와 생사를 같이함
@Service // = @Scope("singleton")
public class OrderService { }
// 컨테이너 시작 시 생성 → 컨테이너 종료 시 소멸
// 모든 요청이 동일 인스턴스 공유

// Prototype — 요청마다 새 인스턴스
@Service
@Scope("prototype")
public class ReportGenerator { }
// getBean() 호출마다 새 인스턴스 생성
// 소멸은 스프링이 관리하지 않음 (@PreDestroy 호출 안 됨)
// 호출한 쪽이 직접 정리해야 함

// Request — HTTP 요청 단위
@Service
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext { }
// 하나의 HTTP 요청 시작 ~ 응답 완료까지 생존

// Session — HTTP 세션 단위
@Service
@Scope(value = "session", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class UserCart { }
```

**Scope 불일치 문제 — 실무에서 자주 만나는 버그:**

```java
// Singleton이 Prototype을 주입받으면?
@Service // singleton
public class OrderService {
    @Autowired
    private ReportGenerator generator; // prototype인데...
    
    public void process() {
        // OrderService는 singleton이라 한 번만 생성됨
        // generator도 한 번만 주입됨
        // → 매번 새 인스턴스를 기대했지만 항상 같은 인스턴스
        generator.generate(); // prototype 의도가 무너짐
    }
}

// 해결: Provider 사용
@Service
public class OrderService {
    @Autowired
    private ObjectProvider<ReportGenerator> generatorProvider;

    public void process() {
        ReportGenerator generator = generatorProvider.getObject(); // 매번 새 인스턴스
        generator.generate();
    }
}
```

---

### ⑨ 소멸 — JVM 종료와 스프링 종료의 연결

```java
// SpringApplication.run() 내부
// JVM ShutdownHook에 컨텍스트 종료를 등록함
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    applicationContext.close(); // → 모든 빈의 @PreDestroy 호출
}));
```

```java
@Service
public class KafkaConsumerService {
    private final KafkaConsumer<String, String> consumer;

    @PreDestroy
    public void shutdown() {
        consumer.close(); // JVM 종료 전 정상 종료 보장
        // 브로커에 정상 탈퇴 신호 전송 → 리밸런싱 최소화
    }
}
```

**주의: SIGKILL은 ShutdownHook이 실행되지 않는다.**

```
SIGTERM → JVM ShutdownHook 실행 → @PreDestroy 호출 → 정상 종료
SIGKILL → JVM 즉시 종료 → ShutdownHook 없음 → @PreDestroy 미실행
```

쿠버네티스 환경에서 `preStop` 훅과 `terminationGracePeriodSeconds`를 설정하는 이유가 여기에 있다.

---

## 3. 공식 근거

|주장|근거|
|---|---|
|BeanPostProcessor로 AOP 프록시 생성|Spring Framework Reference 6.x — _Customizing beans using BeanPostProcessor_|
|초기화 메서드 실행 순서|Spring Framework Reference — _Combining lifecycle mechanisms_|
|@PostConstruct JSR-250 표준|JSR-250 Common Annotations for the Java Platform|
|Prototype 스코프 소멸 미관리|Spring Framework Reference — _The prototype scope_ : "the client code must clean up prototype-scoped objects"|
|ShutdownHook 등록|Spring Framework 소스 `AbstractApplicationContext.registerShutdownHook()`|
|Self-invocation AOP 미동작|Spring Framework Reference — _Understanding AOP proxies_|

---

## 4. 이 설계를 이렇게 한 이유

**BeanPostProcessor로 AOP를 구현한 이유:**

컴파일 타임이나 클래스 로딩 시점이 아닌, 빈 생성 이후 런타임에 프록시를 끼워 넣는 방식을 선택했다.

|                 | 컴파일 타임 위빙         | 로드 타임 위빙     | 런타임 프록시 (현재) |
| --------------- | ----------------- | ------------ | ------------ |
| 적용 시점           | 컴파일               | 클래스 로딩       | 빈 생성 후       |
| 성능              | 가장 빠름             | 빠름           | 프록시 호출 오버헤드  |
| 설정 복잡도          | 높음 (AspectJ 컴파일러) | 높음 (에이전트 필요) | 낮음           |
| Self invocation | 해결됨               | 해결됨          | **문제 있음**    |

런타임 프록시를 선택한 대가로 Self Invocation 문제를 얻었다. 편의성과 트레이드오프한 결과다.

**Prototype 스코프 소멸을 스프링이 관리하지 않는 이유:**

스프링이 Prototype 빈의 참조를 계속 들고 있으면 GC가 수거하지 못한다 → 메모리 누수. 스프링은 의도적으로 참조를 버린다. 소멸 책임은 사용한 쪽으로 넘어간다.

---

## 5. 이어지는 개념

```
현재: Bean Lifecycle
        │
        ▼
다음: ApplicationContext    ← 바로 이어야 하는 이유:
                              Bean Lifecycle이 일어나는 무대가 ApplicationContext
                              BeanFactory와 ApplicationContext의 차이
                              계층 구조 (Parent/Child Context)
                              이벤트 시스템 (ApplicationEvent)
                              Environment와 PropertySource 구성
                              이 맥락에서 이해된다
```