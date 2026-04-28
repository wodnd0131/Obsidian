#
# IoC / DI — 스프링이 객체를 대신 만들고 주입한다

---

## 1. Problem First

### IoC/DI가 없던 시절

```java
// 전통적인 객체 생성 — 사용하는 쪽이 직접 만든다
public class OrderService {
    // 구체 클래스에 직접 의존
    private final PaymentService paymentService = new KakaopayService();
    private final OrderRepository orderRepository = new JdbcOrderRepository(
        new DatabaseConnection("jdbc:mysql://...", "user", "password")
    );
    
    public void order(OrderRequest request) {
        orderRepository.save(request);
        paymentService.pay(request.getAmount());
    }
}
```

**이게 왜 근본적으로 문제인가:**

**문제 1 — 변경이 전파된다**

```java
// KakaopayService → TossPayService로 교체 결정
// OrderService 수정, CartService 수정, SubscriptionService 수정...
// KakaopayService를 new로 생성한 모든 곳을 찾아서 수정해야 함
public class CartService {
    private final PaymentService paymentService = new KakaopayService(); // 여기도
}
public class SubscriptionService {
    private final PaymentService paymentService = new KakaopayService(); // 여기도
}
```

**문제 2 — 테스트가 불가능하다**

```java
// OrderService를 단위 테스트하려면
// JdbcOrderRepository가 실제 DB 연결을 시도함
// DatabaseConnection이 MySQL 서버를 찾음
// 테스트 환경에 MySQL이 없으면 → 테스트 자체가 실행 안 됨
@Test
void 주문_생성_테스트() {
    OrderService service = new OrderService(); // DB 연결 시도 발생
    // 여기서 이미 터짐
}
```

**문제 3 — 객체 생성 비용이 중복 발생한다**

```java
// 애플리케이션 전체에서 DatabaseConnection을 각자 new로 만들면
// 커넥션 풀 없이 매번 TCP 연결 → 수십 개의 중복 연결
public class UserRepository {
    private DatabaseConnection conn = new DatabaseConnection(...); // 새 연결
}
public class OrderRepository {
    private DatabaseConnection conn = new DatabaseConnection(...); // 또 새 연결
}
```

---

## 2. Mechanics

### IoC란 무엇인가 — 제어의 역전

**핵심:** "누가 객체를 만드는가"의 주체가 바뀐다.

```
기존:  내 코드가 의존 객체를 new로 생성         (내가 제어)
IoC:   컨테이너가 객체를 만들어서 내 코드에 넣어줌 (컨테이너가 제어)
```

IoC는 개념이다. DI는 IoC를 구현하는 방식 중 하나다.

```
IoC를 구현하는 방식
├─ Dependency Injection (DI)     ← 스프링이 선택한 방식
├─ Service Locator               ← 안티패턴으로 분류됨 (이유는 아래)
└─ Factory Pattern               ← DI 이전에 쓰던 방식
```

**Service Locator가 안티패턴인 이유:**

```java
// Service Locator — 여전히 내 코드가 의존성을 "꺼내온다"
public class OrderService {
    public void order(OrderRequest request) {
        // 컨테이너에서 직접 꺼냄 — 의존성이 숨겨짐
        PaymentService payment = ServiceLocator.get(PaymentService.class);
        // 이 클래스가 뭘 필요로 하는지 외부에서 알 수 없음
        // 테스트 시 ServiceLocator 자체를 mock해야 함
    }
}
```

---

### DI의 세 가지 방식과 내부 동작

**생성자 주입 (권장)**

```java
@Service
public class OrderService {
    private final PaymentService paymentService;
    private final OrderRepository orderRepository;

    // @Autowired 생략 가능 (생성자 1개일 때)
    public OrderService(PaymentService paymentService,
                        OrderRepository orderRepository) {
        this.paymentService = paymentService;
        this.orderRepository = orderRepository;
    }
}
```

**필드 주입 (비권장)**

```java
@Service
public class OrderService {
    @Autowired
    private PaymentService paymentService; // final 불가 — 불변성 보장 안 됨
    // 리플렉션으로 private 필드에 직접 값을 넣음
    // 테스트 시 스프링 컨테이너 없으면 주입 방법이 없음
}
```

**수정자 주입 (선택적 의존성에만)**

```java
@Service
public class OrderService {
    private NotificationService notificationService; // 없어도 동작해야 할 때

    @Autowired(required = false)
    public void setNotificationService(NotificationService service) {
        this.notificationService = service;
    }
}
```

---

### 스프링 컨테이너의 DI 실행 순서

```
1. BeanDefinition 수집
   @ComponentScan → 클래스 스캔
   @Bean 메서드 → 팩토리 메서드 등록
   XML → 레거시 정의
        │
        ▼
2. BeanDefinition 파싱
   "OrderService는 PaymentService와 OrderRepository가 필요하다"
   → 의존관계 메타데이터로 저장 (아직 객체 생성 안 함)
        │
        ▼
3. 의존관계 그래프 구성
   OrderService → PaymentService → (없음)
               └→ OrderRepository → DataSource
        │
        ▼
4. 위상 정렬로 생성 순서 결정
   DataSource → OrderRepository → PaymentService → OrderService
        │
        ▼
5. 순서대로 인스턴스 생성 + 주입
   DataSource 생성
   OrderRepository 생성 (DataSource 주입)
   PaymentService 생성
   OrderService 생성 (PaymentService, OrderRepository 주입)
```

**순환 의존성이 문제가 되는 이유:**

```java
// A → B → A 순환
@Service class A {
    public A(B b) {}
}
@Service class B {
    public B(A a) {}  // A를 만들려면 B가 필요, B를 만들려면 A가 필요
}
// 생성자 주입: 시작 시점에 UnsatisfiedDependencyException 발생 (즉시 감지)
// 필드 주입:  과거 스프링은 프록시로 우회 — 런타임에 터짐 (늦게 감지)
// → 생성자 주입이 순환 의존성을 조기에 잡아주는 이유
```

---

### BeanDefinition — 스프링이 객체를 만드는 설계도

스프링은 객체를 바로 만들지 않는다. 먼저 **설계도(BeanDefinition)**를 만들고, 그 다음 객체를 생성한다.

```java
// BeanDefinition이 담고 있는 정보
public interface BeanDefinition {
    String getBeanClassName();       // 어떤 클래스인가
    String getScope();               // singleton? prototype?
    boolean isLazyInit();            // 지연 초기화 여부
    String[] getDependsOn();         // 명시적 의존 순서
    ConstructorArgumentValues getConstructorArgumentValues(); // 생성자 인자
    MutablePropertyValues getPropertyValues();               // 프로퍼티 값
}
```

```java
// @Component → BeanDefinition 변환 과정
// ClassPathScanningCandidateComponentProvider가 클래스 스캔
// → ScannedGenericBeanDefinition 생성
// → BeanDefinitionRegistry에 등록
// → 나중에 getBean() 호출 시 실제 인스턴스 생성
```

---

### Singleton Registry — 왜 기본이 싱글톤인가

스프링 빈의 기본 스코프는 싱글톤이다. 단순히 "메모리 절약"이 이유가 아니다.

```java
// DefaultSingletonBeanRegistry 내부 — 세 개의 Map
private final Map<String, Object> singletonObjects      // 완성된 빈
        = new ConcurrentHashMap<>(256);
private final Map<String, Object> earlySingletonObjects // 생성 중인 빈 (순환참조 처리용)
        = new ConcurrentHashMap<>(16);
private final Map<String, ObjectFactory<?>> singletonFactories // 빈 팩토리
        = new HashMap<>(16);
```

**요청 시 흐름:**

```java
// AbstractBeanFactory.getBean() 내부 요약
Object bean = getSingleton(beanName);   // 1. 캐시 조회
if (bean == null) {
    bean = createBean(beanName, ...);   // 2. 없으면 생성
    addSingleton(beanName, bean);       // 3. 캐시에 등록
}
return bean;
```

**싱글톤이기 때문에 지켜야 하는 것:**

```java
// 상태를 가지면 안 된다 — 모든 요청이 같은 인스턴스를 공유
@Service
public class OrderService {
    // 위험: 인스턴스 변수에 요청별 데이터 저장
    private Order currentOrder; // 스레드 A의 주문이 스레드 B에게 보임

    // 안전: 메서드 로컬 변수 사용
    public void order(OrderRequest request) {
        Order order = new Order(request); // 스택에만 존재
    }
}
```

---

## 3. 공식 근거

|주장|근거|
|---|---|
|IoC 개념 정의|Martin Fowler, _"Inversion of Control Containers and the Dependency Injection pattern"_, martinfowler.com (2004)|
|Service Locator vs DI|Martin Fowler 동일 문서 — "Service Locator hides dependencies"|
|생성자 주입 권장|Spring Framework Reference 6.x — _"Constructor-based or setter-based DI?"_|
|BeanDefinition 구조|Spring Framework Javadoc — `org.springframework.beans.factory.config.BeanDefinition`|
|Singleton Registry 구현|Spring Framework 소스 — `DefaultSingletonBeanRegistry`|
|순환 의존성 생성자 주입 감지|Spring Framework Reference — _"Circular dependencies"_|

---

## 4. 이 설계를 이렇게 한 이유

**얻는 것:**

```
구체 클래스 의존 → 인터페이스 의존으로 전환 가능
    → 교체, 테스트, 확장이 쉬워짐

객체 생성 책임을 컨테이너로 분리
    → 개발자는 "무엇을 할 것인가"에만 집중
```

**잃는 것 — 구체적으로:**

|비용|내용|
|---|---|
|시작 시간 증가|컨텍스트 로딩 시 모든 빈 스캔 + 생성. 빈이 수백 개면 수 초|
|런타임 오버헤드|리플렉션 기반 주입 — 생성자 주입은 1회지만 필드 주입은 매번|
|추적 어려움|IDE 없이 "이 인터페이스의 실제 구현체가 뭔지" 파악이 어려움|
|과도한 추상화 위험|구현체가 하나뿐인 인터페이스 — 불필요한 간접 계층만 추가|
|컨테이너 종속|스프링 없이 테스트하려면 의존성 직접 조립 필요|

---

## 5. 이어지는 개념

이 키워드에서 자연스럽게 이어지는 학습 순서:

```
현재: IoC / DI
    │
    ▼
다음: Bean Lifecycle          ← 바로 이어야 하는 이유:
                                DI가 일어나는 시점이 라이프사이클의 일부
                                @PostConstruct가 왜 생성자 이후인지
                                Scope(singleton/prototype)가 DI에 어떤 영향을 주는지
                                이 맥락에서 이해된다
```


# 싱글톤 선택 이유 + 상태 관리 위험

---

## 싱글톤을 선택한 이유 — 메모리 절약이 아니다

진짜 이유는 **무상태 서비스 객체의 재사용**이다.

```
HTTP 요청이 초당 1000개 들어오는 서버

Prototype 방식이었다면:
  요청마다 OrderService 생성
  + OrderService가 의존하는 PaymentService 생성
  + PaymentService가 의존하는 HttpClient 생성
  = 초당 수천 개 객체 생성 → GC 압박

Singleton 방식:
  OrderService 1개 → 1000개 요청이 공유
  상태가 없으니 공유해도 안전
  객체 생성/소멸 비용 없음
```

핵심 전제는 **"서비스 객체는 상태가 없다"** 는 것이다. 상태가 없으니까 공유할 수 있고, 공유하니까 싱글톤이 의미 있다.

---

## 상태(State)가 뭔지부터 정확히 짚자

```java
@Service
public class OrderService {

    // ① 스프링 빈 — 상태 아님
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    // 이건 의존성이다. 불변이고, 모든 요청이 같은 객체를 써도 문제없다.
    // 빈 자체가 무상태이기 때문.

    // ② 상수 — 상태 아님
    private static final int MAX_RETRY = 3;

    // ③ 인스턴스 변수 — 상태다 ← 위험 구간
    private Order currentOrder;         // 요청별로 달라져야 하는 값
    private int retryCount;             // 요청별로 달라져야 하는 값
    private String currentUserId;       // 요청별로 달라져야 하는 값
}
```

질문의 핵심이 맞다. **빈으로 등록되지 않은 일반 인스턴스 필드**가 문제다. 스프링 빈을 주입받은 필드는 괜찮다 — 그 빈 자체도 무상태이기 때문이다.

---

## 실제로 어떻게 터지는가

```java
@Service
public class OrderService {
    private Order currentOrder; // 위험한 인스턴스 변수

    public void process(OrderRequest request) {
        currentOrder = new Order(request); // 스레드 A: A의 주문 저장
        validate();                         // 여기서 컨텍스트 스위칭 발생
        save();
    }

    private void validate() {
        // 스레드 B가 process()를 호출해서 currentOrder를 덮어씀
        // 스레드 A의 validate()는 B의 주문을 검증하게 됨
        if (currentOrder.getAmount() <= 0) { ... }
    }
}
```

```
타임라인:

스레드 A: currentOrder = A의 주문
스레드 B: currentOrder = B의 주문  ← 덮어씀
스레드 A: validate() → B의 주문을 검증 ← 데이터 오염
스레드 A: save() → B의 주문을 저장    ← 장애
```

재현이 어렵다. 부하가 낮을 때는 거의 발생 안 하고, 트래픽이 몰릴 때 간헐적으로 발생한다. 로그도 남기기 어렵다.

---

## 어디에 두면 안전하고, 어디에 두면 위험한가

```java
@Service
public class OrderService {

    // ✅ 안전 — 의존성 (무상태 빈)
    private final OrderRepository orderRepository;

    // ✅ 안전 — 불변 상수
    private static final String PREFIX = "ORD";

    // ✅ 안전 — 메서드 로컬 변수 (스레드 스택에 존재)
    public void process(OrderRequest request) {
        Order order = new Order(request);   // 이 스레드 스택에만 존재
        String orderId = PREFIX + UUID.randomUUID();
        orderRepository.save(order);
    }

    // ❌ 위험 — 요청별로 달라지는 값을 인스턴스 변수에
    private Order currentOrder;
    private String currentUserId;
    private int step;
}
```

---

## 그러면 요청별 데이터는 어디에 두는가

**방법 1: 메서드 파라미터로 전달**

```java
@Service
public class OrderService {

    public void process(OrderRequest request) {
        Order order = new Order(request);
        validate(order);        // 파라미터로 넘김
        save(order);            // 파라미터로 넘김
    }

    private void validate(Order order) { ... }  // 인스턴스 변수 없음
    private void save(Order order) { ... }
}
```

가장 단순하고 안전하다. 대부분의 경우 이걸로 해결된다.

**방법 2: ThreadLocal — 요청 컨텍스트 전달이 필요할 때**

```java
// 스프링의 SecurityContextHolder가 실제로 이 방식 사용
public class UserContext {
    private static final ThreadLocal<String> currentUserId = new ThreadLocal<>();

    public static void set(String userId) { currentUserId.set(userId); }
    public static String get() { return currentUserId.get(); }
    public static void clear() { currentUserId.remove(); } // 반드시 정리
}

// 필터에서 설정
public class AuthFilter implements Filter {
    public void doFilter(ServletRequest req, ...) {
        String userId = extractUserId(req);
        UserContext.set(userId);
        try {
            chain.doFilter(req, res);
        } finally {
            UserContext.clear(); // 스레드 풀 환경에서 필수 — 안 하면 다음 요청에 오염
        }
    }
}

// 서비스에서 사용
@Service
public class OrderService {
    public void process(OrderRequest request) {
        String userId = UserContext.get(); // 파라미터 없이 접근
    }
}
```

**ThreadLocal 주의점:**

```
스레드 풀 환경 (Tomcat)에서:
  요청 A → 스레드 1에서 처리 → ThreadLocal에 A의 userId 저장
  요청 A 완료 → 스레드 1이 풀로 반환
  요청 B → 스레드 1 재사용 → ThreadLocal에 A의 userId가 남아있음

→ clear()를 finally에서 반드시 호출해야 하는 이유
```

**방법 3: RequestScope 빈**

```java
// HTTP 요청 1개 = 빈 1개
@Component
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class OrderContext {
    private String userId;
    private String traceId;
    // getter/setter
}

@Service
public class OrderService {
    private final OrderContext orderContext; // 요청마다 다른 인스턴스가 주입됨

    public void process(OrderRequest request) {
        String userId = orderContext.getUserId(); // 이 요청의 데이터
    }
}
```

---

## 실무에서 자주 놓치는 패턴

```java
// 놓치기 쉬운 케이스 1 — 누적되는 컬렉션
@Service
public class ReportService {
    private List<String> errors = new ArrayList<>(); // ❌

    public ReportResult generate(ReportRequest req) {
        errors.clear(); // 초기화하려 했지만
        // 멀티스레드에서 clear()와 add()가 동시에 실행되면 의미없음
        validate(req);
        return new ReportResult(errors);
    }
}

// 수정
@Service
public class ReportService {
    public ReportResult generate(ReportRequest req) {
        List<String> errors = new ArrayList<>(); // ✅ 로컬 변수
        validate(req, errors);
        return new ReportResult(errors);
    }
}
```

```java
// 놓치기 쉬운 케이스 2 — SimpleDateFormat
@Service
public class DateService {
    // SimpleDateFormat은 thread-safe하지 않음
    private final SimpleDateFormat sdf
        = new SimpleDateFormat("yyyy-MM-dd"); // ❌

    public String format(Date date) {
        return sdf.format(date); // 내부 Calendar 상태를 공유 → 오염
    }
}

// 수정
@Service
public class DateService {
    public String format(Date date) {
        // 매번 새로 생성 (비용 낮음) 또는
        return new SimpleDateFormat("yyyy-MM-dd").format(date);
        // 또는 DateTimeFormatter 사용 (Java 8+, thread-safe)
    }
}
```

```java
// 놓치기 쉬운 케이스 3 — 외부 라이브러리 클라이언트
@Service
public class ExternalApiService {
    private RestTemplate restTemplate = new RestTemplate(); // ✅ 이건 괜찮음
    // RestTemplate은 thread-safe하게 설계됨

    private HttpClient httpClient = HttpClient.newBuilder().build(); // ✅ 괜찮음
    // 재사용 목적으로 설계됨

    // 단, 이런 건 위험
    private HttpResponse lastResponse; // ❌ 요청별 데이터를 인스턴스 변수에
}
```

---

## 한 줄 판단 기준

```
이 필드가 요청마다 다른 값을 가져야 하는가?
    YES → 인스턴스 변수로 두면 안 된다
           메서드 파라미터 / ThreadLocal / RequestScope 빈 중 선택

이 필드가 모든 요청에서 항상 같은 객체인가?
    YES → 인스턴스 변수로 둬도 안전
           단, 그 객체 자체가 thread-safe한지 확인 필요
```