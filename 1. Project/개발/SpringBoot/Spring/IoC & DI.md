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


---
---
# DI/메타데이터가 빠진 IoC

좋은 포인트다. **IoC와 DI는 자주 같이 등장하지만 별개 개념**이다. 이걸 분리해서 보자.

## 1. Problem First — 두 개념을 헷갈리면 생기는 오해

흔한 오해: "IoC = DI = Spring 컨테이너"

이렇게 묶어 이해하면 다음 질문에 답할 수 없다.

```java
// 이건 IoC인가? DI인가?
Collections.sort(list, (a, b) -> a.compareTo(b));

// 이건?
button.addEventListener("click", () -> handleClick());

// 이건?
@Override
protected void doGet(HttpServletRequest req, HttpServletResponse resp) { ... }
```

셋 다 **IoC지만 DI는 아니다.** "프레임워크가 내 코드를 호출하는" 패턴이지만, 어떤 객체도 주입받지 않는다.

거꾸로:

```java
// 이건 DI지만 IoC 컨테이너 없이 가능
UserService service = new UserService(new UserRepository(dataSource));
```

순수 `new`로도 DI는 가능하다. **IoC 컨테이너는 DI를 자동화하는 도구일 뿐**이지, DI의 본질이 아니다.

## 2. Mechanics — IoC의 진짜 정의

### 2.1 IoC의 본질: "호출 방향의 역전"

전통적 제어:

```java
// 내 코드가 라이브러리를 호출 — Library-style
public static void main(String[] args) {
    String input = readInput();           // 내가 부름
    String result = parse(input);          // 내가 부름
    String output = transform(result);     // 내가 부름
    print(output);                         // 내가 부름
}
```

→ **흐름의 주인이 나(`main`)다.** 라이브러리는 부르면 응답할 뿐.

역전된 제어:

```java
// 프레임워크가 내 코드를 호출 — Framework-style
@GetMapping("/api/users")
public List<User> list() {                // ← 내가 정의만 함
    return userService.findAll();          // ← Tomcat이 언젠가 호출함
}
```

→ **흐름의 주인이 프레임워크다.** 나는 "이 시점에 이걸 해줘"라고 정의만 한다.

이 둘의 차이를 **Hollywood Principle**이라고 부른다: "Don't call us, we'll call you."

### 2.2 IoC가 발현되는 방식들 (DI는 그중 하나일 뿐)

Martin Fowler가 정리한 IoC의 형태:

|형태|방식|예시|
|---|---|---|
|**Template Method**|추상 클래스가 흐름 정의, 자식이 구멍 채움|`HttpServlet.service()` → `doGet()` 호출|
|**Callback**|함수를 등록해두면 프레임워크가 호출|`Comparator`, 이벤트 리스너|
|**Lifecycle hooks**|"초기화 시점", "종료 시점"에 호출|`@PostConstruct`, `InitializingBean`|
|**Scheduler-driven**|프레임워크가 시간에 맞춰 호출|`@Scheduled`|
|**Dependency Injection**|의존 객체를 외부에서 주입|`@Autowired`, 생성자 주입|

**DI는 5번째 형태일 뿐이다.** 나머지는 DI 없이도 IoC다.

### 2.3 Endpoint에서 작동하는 IoC (DI 없이)

```java
// org.apache.tomcat.util.net.NioEndpoint (요약)
public class NioEndpoint {
    private Acceptor acceptor;
    private Poller poller;
    private Executor executor;

    public void startInternal() {
        // 1. 자식 컴포넌트들을 직접 new로 생성 (DI 아님)
        executor = new ThreadPoolExecutor(...);
        poller = new Poller();
        acceptor = new Acceptor(this);

        // 2. 각각 start() — Lifecycle 주도권을 Endpoint가 쥠
        new Thread(poller).start();
        new Thread(acceptor).start();
    }

    public void stopInternal() {
        acceptor.stop();
        poller.destroy();
        executor.shutdown();
    }
}
```

여기서 작동하는 IoC:

1. **흐름 제어 역전** — Acceptor/Poller/Worker는 "내가 일하고 싶을 때" 일하지 않는다. Endpoint가 start하면 일하고, stop하면 멈춘다
2. **Lifecycle 위임** — Acceptor는 자기 생명주기를 모른다. Endpoint가 결정
3. **Hollywood Principle 적용** — `SocketProcessor`(Worker가 실행할 작업)는 자기가 언제 실행될지 모른다. Executor가 알아서 호출

근데 여기 **DI는 없다.** Endpoint가 `new Acceptor(this)`로 직접 만들고, 메타데이터(설정 파일)로 "Acceptor 대신 다른 구현체 써"라고 바꿀 수도 없다.

**→ 그래도 이건 IoC다.** 호출 방향이 역전되어 있으니까.

## 3. 공식 근거

**Martin Fowler — Inversion of Control**

> "The question is: what aspect of control are they inverting? ... The answer is: the control of when the framework is called. The framework calls your code rather than your code calls the framework."
> 
> (질문은: 무엇의 제어를 역전시키는가? 답은: 프레임워크가 언제 호출되는지에 대한 제어다. 네 코드가 프레임워크를 호출하는 게 아니라, 프레임워크가 네 코드를 호출한다.)

→ 출처: [Martin Fowler — InversionOfControl](https://martinfowler.com/bliki/InversionOfControl.html)

**Martin Fowler — IoC와 DI는 다르다 (결정적 논문)**

> "Inversion of Control is a common phenomenon that you come across in frameworks ... As a result I think we need a more specific name for this pattern. Inversion of Control is too generic a term, and thus people find it confusing. ... So I'm going to be Mr Picky and stick to the term Dependency Injection for this kind of inversion."
> 
> (IoC는 프레임워크에서 흔히 나타나는 일반 현상이다. 그래서 이 패턴에는 더 구체적인 이름이 필요하다. IoC는 너무 일반적인 용어라 혼란스럽다. 그래서 나는 까다롭게 굴자면 이런 종류의 역전에는 Dependency Injection이라는 용어를 쓰겠다.)

→ 출처: [Martin Fowler — Inversion of Control Containers and the Dependency Injection pattern](https://martinfowler.com/articles/injection.html)

**핵심**: Fowler가 명시적으로 **"IoC는 너무 큰 개념이고, DI는 그중 한 가지 구체 패턴"**이라고 못 박았다. 우리가 흔히 "Spring IoC 컨테이너"라고 부르는 건 사실 **"DI 컨테이너"**다.

**Spring Framework 공식 레퍼런스**

> "IoC is also known as dependency injection (DI). It is a process whereby objects define their dependencies ... only through constructor arguments, arguments to a factory method, or properties that are set on the object instance after it is constructed or returned from a factory method."
> 
> (IoC는 의존성 주입(DI)이라고도 알려져 있다. 객체가 자기 의존성을 생성자 인수, 팩토리 메서드 인수, 또는 객체 생성 후 설정되는 프로퍼티를 통해서만 정의하는 프로세스다.)

→ 출처: [Spring Framework Reference — The IoC Container](https://docs.spring.io/spring-framework/reference/core/beans/introduction.html)

**주의**: Spring 문서는 IoC와 DI를 동일시한다. 이건 **Spring 컨텍스트 한정**이다. Spring이 다루는 IoC는 거의 항상 DI 형태로 구현되기 때문에 동일시한 거지, **IoC 자체의 정의가 그렇다는 게 아니다.** Fowler의 정리가 더 일반론으로 정확하다.

## 4. 이 분리가 왜 중요한가

**잃는 것 vs 얻는 것**

DI 없이 IoC만 적용했을 때:

✓ **단순하다** — `new`로 직접 만들면 코드 흐름이 명확. Tomcat NioEndpoint 소스는 `new Acceptor(...)` 한 줄로 끝남 ✓ **빠르다** — 컨테이너 부트스트랩 비용 없음. 리플렉션, 프록시 생성, BeanDefinition 처리 다 없음 ✓ **추적 쉽다** — IDE에서 "이 객체 어디서 만들어지지?" → `new` 호출 지점 한 군데

✗ **결합도 높다** — Acceptor 구현을 바꾸려면 Endpoint 코드를 수정해야 함 ✗ **테스트 어렵다** — Mock Acceptor 주입 불가 ✗ **설정으로 동작 변경 불가** — 코드 재컴파일 필요

**그래서 Tomcat은 왜 DI를 안 쓰는가**

Tomcat 같은 **저수준 인프라**는:

- 컴포넌트 교체 가능성이 거의 없음 (Acceptor 다른 구현으로 바꿀 일 없음)
- 부트스트랩 속도가 중요 (서버 시작은 빨라야 함)
- 메모리 푸세 작음

반면 Spring 같은 **애플리케이션 프레임워크**는:

- 사용자가 컴포넌트를 갈아 끼울 수 있어야 함 (Repository 구현체, DataSource 종류 등)
- 테스트 가능성이 중요
- 부트스트랩 시간보다 유연성이 우선

**같은 IoC 철학을 따르되, 인프라 레이어는 hardcoded composition으로, 애플리케이션 레이어는 DI 컨테이너로 풀어낸다.** 이게 같은 시스템 안에서 둘이 공존하는 이유다.

## 5. 이어지는 개념

DI/메타데이터 없는 IoC를 이해했으니, 자연스러운 다음 질문들:

1. **Spring이 DI를 어떻게 자동화하는가** — `BeanDefinition` 메타데이터, `BeanFactory`, `ApplicationContext`의 역할. 네가 전에 공부한 3-level cache가 여기 연결된다. **선행 의존**: IoC 일반론 → Spring의 DI 구현. 지금 흐름의 직선 다음 단계
    
2. **Hollywood Principle의 다른 발현 — Template Method 패턴** — Tomcat의 `Valve` 체인, Servlet의 `service()` → `doGet()`이 이 패턴. DI 없는 IoC의 또 다른 전형. Spring MVC `DispatcherServlet`이 이걸로 흐름 제어한다. **실무 임팩트**: Spring MVC 내부 이해의 출발
    
3. **`@Configuration` + `@Bean` — 메타데이터 없이 코드로 DI 구성하기** — XML 메타데이터의 대안. "DI는 쓰되 메타데이터 파일은 안 쓰는" 중간 형태. 네가 미뤄둔 Spring 학습 트랙의 다음 토픽과 정확히 일치. **실무 임팩트 + 선행 의존**: 1번 이해 후 가장 자연스러움
    
4. **DI 없이 IoC만 쓰는 다른 예: JDK의 `ServiceLoader`** — 메타데이터(`META-INF/services`)로 구현체를 찾지만 컨테이너는 없음. Spring 이전 시대의 IoC 패턴. **참고 수준**: 시야 넓히기용
    

순서 이유: 1번이 지금 질문의 정확한 후속(IoC → DI 자동화 메커니즘)이고, 2번은 "DI 없는 IoC가 Spring 안에도 있다"는 시야 확장, 3번은 메타데이터 자체를 코드로 대체한 형태로 Spring 학습 트랙과 정확히 만난다. 4번은 흥미용.