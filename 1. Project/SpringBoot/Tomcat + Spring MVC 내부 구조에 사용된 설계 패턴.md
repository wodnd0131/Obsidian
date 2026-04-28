#Java/Pattern   #SpringBoot 

## 전체 지도 먼저

```
요청 흐름                          적용 패턴
─────────────────────────────────────────────────────────
Tomcat 소켓 수신
    │
    ▼
Filter Chain              ←  Chain of Responsibility
    │
    ▼
DispatcherServlet         ←  Front Controller
    │
    ├─ HandlerMapping     ←  Registry (Map 기반 탐색)
    │
    ├─ HandlerAdapter     ←  Adapter
    │      │
    │      ├─ ArgumentResolver  ←  Strategy + Composite
    │      ├─ method.invoke()   ←  Template Method (추상화 레이어)
    │      └─ ReturnValueHandler ← Strategy + Composite
    │
    ├─ Interceptor Chain  ←  Chain of Responsibility
    │
    ├─ MessageConverter   ←  Strategy
    │
    └─ ViewResolver       ←  Strategy + Registry
```

---

## 1. Front Controller — DispatcherServlet

**문제:** 모든 서블릿이 각자 인증/로깅/예외처리를 중복 구현하고 있었다.

```java
// 패턴 이전 — 서블릿마다 중복
public class UserServlet extends HttpServlet {
    protected void doGet(...) {
        checkAuth();       // 중복
        logRequest();      // 중복
        // 실제 로직
    }
}
public class OrderServlet extends HttpServlet {
    protected void doGet(...) {
        checkAuth();       // 중복
        logRequest();      // 중복
    }
}
```

**해결:** 모든 요청을 단일 진입점이 받아서 공통 처리 후 위임한다.

```
모든 HTTP 요청 → DispatcherServlet (단일 진입점)
                      │
                      ├─ 공통 처리 (인터셉터, 예외처리)
                      └─ 적절한 핸들러로 위임
```

**Spring 구현:**

```java
// web.xml 또는 Spring Boot 자동 구성
// 모든 경로(/)를 DispatcherServlet 하나가 받는다
@Bean
public DispatcherServlet dispatcherServlet() {
    return new DispatcherServlet();
}
```

**잃는 것:** DispatcherServlet 자체가 단일 장애점이 된다. 이 서블릿이 뜨지 않으면 모든 요청이 실패한다.

---

## 2. [[Chain of Responsibility]] — Filter Chain / Interceptor Chain

**문제:** 요청 처리 단계를 하드코딩하면 순서 변경, 단계 추가/제거가 불가능하다.

```java
// 하드코딩된 처리 — 순서 바꾸려면 코드 수정
public void handleRequest(Request req) {
    checkEncoding(req);
    checkAuth(req);
    checkRateLimit(req);
    doActualWork(req);
}
```

**해결:** 각 처리 단계를 독립 객체로 만들고, 각자 다음 단계 호출 책임을 갖는다.

```java
// Filter — 자신이 chain.doFilter()를 호출해야 다음으로 넘어감
public class AuthFilter implements Filter {
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
        if (!isAuthenticated(req)) {
            ((HttpServletResponse) res).sendError(401);
            return;          // chain 호출 안 함 → 여기서 중단
        }
        chain.doFilter(req, res);   // 다음 필터로 넘김
        // 응답이 돌아온 후 처리할 것이 있으면 여기에
    }
}
```

**Filter vs Interceptor의 Chain 구현 차이:**

```
Filter Chain (Tomcat — ApplicationFilterChain)
  → pos 카운터로 배열 순차 순회
  → 필터가 직접 chain.doFilter() 호출
  → 중단 = chain 호출 안 하면 됨

Interceptor Chain (Spring — HandlerExecutionChain)
  → preHandle: 순방향, false 반환 시 중단 + 역순 afterCompletion 보장
  → postHandle / afterCompletion: 역방향
  → 중단 = false 반환하면 프레임워크가 나머지 처리
```

**잃는 것:** 체인이 길어질수록 디버깅이 어렵다. 어느 단계에서 요청이 막혔는지 추적하려면 각 필터/인터셉터를 전부 확인해야 한다.

---

## 3. Adapter — HandlerAdapter

**문제:** DispatcherServlet은 핸들러를 `Object`로 받는다. 타입마다 호출 방식이 다른데, DispatcherServlet이 모든 타입을 알면 OCP 위반이다.

```java
// 어댑터 없이 — DispatcherServlet이 모든 타입을 알아야 함
if (handler instanceof HandlerMethod) {
    // 리플렉션으로 메서드 호출
} else if (handler instanceof HttpRequestHandler) {
    // handleRequest() 호출
} else if (handler instanceof Servlet) {
    // service() 호출
} // 새 타입 추가할 때마다 여기 수정
```

**해결:** 타입별로 Adapter를 만들고, DispatcherServlet은 `handle()`만 호출한다.

```java
public interface HandlerAdapter {
    boolean supports(Object handler);           // 이 타입 처리 가능?
    ModelAndView handle(HttpServletRequest req,
                        HttpServletResponse res,
                        Object handler);        // 타입 무관하게 실행
}

// DispatcherServlet — 타입을 모른다
HandlerAdapter ha = getHandlerAdapter(handler); // supports()로 탐색
ha.handle(request, response, handler);          // 인터페이스만 사용
```

```
HandlerAdapter 구현체들
├─ RequestMappingHandlerAdapter   → HandlerMethod (@Controller)
├─ HttpRequestHandlerAdapter      → HttpRequestHandler
└─ SimpleServletHandlerAdapter    → Servlet
```

**잃는 것:** 새 핸들러 타입을 추가하면 Adapter 구현체도 함께 만들어야 한다. 간단한 경우엔 오버엔지니어링이 될 수 있다.

---

## 4. Strategy — ArgumentResolver / ReturnValueHandler / MessageConverter

세 컴포넌트가 모두 같은 패턴을 사용한다. `supports()` 로 자신이 처리 가능한지 확인하고, 처리한다.

```java
// 공통 구조
public interface HandlerMethodArgumentResolver {
    boolean supportsParameter(MethodParameter parameter); // 전략 선택 조건
    Object resolveArgument(...);                          // 실제 전략 실행
}
```

```
ArgumentResolver 구현체 탐색 흐름:

@PathVariable String id     → PathVariableMethodArgumentResolver.supports() = true  → 처리
@RequestBody UserDto dto    → RequestResponseBodyMethodProcessor.supports() = true  → 처리
@RequestParam String name   → RequestParamMethodArgumentResolver.supports() = true  → 처리
HttpServletRequest req      → ServletRequestMethodArgumentResolver.supports() = true → 처리
```

**Composite 패턴과 함께 사용:**

Strategy 목록을 하나의 컴포지트로 관리한다.

```java
// HandlerMethodArgumentResolverComposite
// 개별 Resolver들을 묶어 단일 인터페이스처럼 동작
public class HandlerMethodArgumentResolverComposite
        implements HandlerMethodArgumentResolver {

    private final List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>();

    public boolean supportsParameter(MethodParameter parameter) {
        return getArgumentResolver(parameter) != null; // 목록 순회
    }

    public Object resolveArgument(MethodParameter parameter, ...) {
        HandlerMethodArgumentResolver resolver = getArgumentResolver(parameter);
        return resolver.resolveArgument(parameter, ...); // 위임
    }
}
```

**MessageConverter의 Strategy:**

```java
public interface HttpMessageConverter<T> {
    boolean canRead(Class<?> clazz, MediaType mediaType);   // 역직렬화 가능?
    boolean canWrite(Class<?> clazz, MediaType mediaType);  // 직렬화 가능?
    T read(...);
    void write(...);
}

// Content-Type: application/json  → MappingJackson2HttpMessageConverter 선택
// Content-Type: text/xml          → Jaxb2RootElementHttpMessageConverter 선택
// Content-Type: text/plain        → StringHttpMessageConverter 선택
```

**잃는 것:** supports() 순회는 O(n)이다. Resolver/Converter가 많아질수록 매 요청마다 탐색 비용이 증가한다. Spring은 이를 `HandlerMethodArgumentResolverComposite` 내부 캐시로 완화한다.

---

## 5. Template Method — FrameworkServlet / HttpServlet 계층

**문제:** HTTP 메서드별(GET/POST/PUT...) 분기는 공통이지만, 실제 처리 로직은 서브클래스마다 다르다.

**해결:** 상위 클래스가 흐름을 정의하고, 가변 부분만 추상 메서드로 위임한다.

```java
// HttpServlet (Jakarta) — 템플릿 정의
public abstract class HttpServlet extends GenericServlet {
    // 흐름 고정 — 오버라이드 불가
    protected void service(HttpServletRequest req, HttpServletResponse res) {
        String method = req.getMethod();
        if ("GET".equals(method)) {
            doGet(req, res);      // 가변 지점
        } else if ("POST".equals(method)) {
            doPost(req, res);     // 가변 지점
        }
        // ...
    }

    // 가변 지점 — 서브클래스가 구현
    protected void doGet(HttpServletRequest req, HttpServletResponse res) {
        res.sendError(SC_METHOD_NOT_ALLOWED);
    }
}

// FrameworkServlet (Spring) — 추가 계층
public abstract class FrameworkServlet extends HttpServletBean {
    // doGet/doPost 등을 전부 processRequest()로 통일
    @Override
    protected final void doGet(HttpServletRequest req, HttpServletResponse res) {
        processRequest(req, res); // 공통 처리 후
    }

    protected final void processRequest(...) {
        // LocaleContext, RequestAttributes 설정 등 공통 처리
        doService(req, res);  // 가변 지점
    }

    // DispatcherServlet이 구현
    protected abstract void doService(...);
}

// DispatcherServlet — 실제 가변 지점 구현
public class DispatcherServlet extends FrameworkServlet {
    @Override
    protected void doService(...) {
        doDispatch(req, res); // 여기서 HandlerMapping, Adapter 등 실행
    }
}
```

```
HttpServlet          (Jakarta) — service() 템플릿 정의
    └─ HttpServletBean   (Spring) — 프로퍼티 바인딩 추가
           └─ FrameworkServlet  (Spring) — processRequest() 템플릿 추가
                  └─ DispatcherServlet  (Spring) — doService() 구현
```

**잃는 것:** 상속 계층이 깊어질수록 흐름 추적이 어렵다. `doGet()`이 최종적으로 어디서 처리되는지 알려면 4단계 클래스를 따라가야 한다.

---

## 6. Registry — HandlerMapping / ViewResolver

**문제:** URL → 핸들러 매핑 정보를 요청 시점에 동적으로 탐색하면 느리다. 매번 모든 클래스를 스캔할 수 없다.

**해결:** 애플리케이션 시작 시 매핑 정보를 Map에 미리 등록하고, 요청 시 조회한다.

```java
// RequestMappingHandlerMapping 초기화 시점
// ApplicationContext의 모든 빈을 순회하며 @RequestMapping 탐색
public void afterPropertiesSet() {
    // 스캔 — 시작 시 1회
    for (String beanName : applicationContext.getBeanNamesForType(Object.class)) {
        detectHandlerMethods(beanName);
    }
}

private void detectHandlerMethods(String beanName) {
    // @RequestMapping 붙은 메서드 찾기
    Map<Method, RequestMappingInfo> methods = MethodIntrospector.selectMethods(
        beanType, (Method method) -> getMappingForMethod(method, beanType)
    );
    // Registry에 등록
    methods.forEach((method, mapping) -> {
        registerMapping(mapping, beanName, method); // 내부 Map에 저장
    });
}

// 요청 시점 — O(1) 탐색 (URL 기반 해시)
public HandlerExecutionChain getHandler(HttpServletRequest request) {
    return lookupHandlerMethod(request); // 사전 구축된 Map 조회
}
```

**잃는 것:** 런타임에 동적으로 핸들러를 추가하기 어렵다. 시작 시점에 모든 매핑이 확정되어야 한다.

---

## 패턴 간 관계 요약

```
Front Controller (DispatcherServlet)
    │
    │  요청 수신 후 모든 패턴을 조율하는 중심
    │
    ├─ Chain of Responsibility
    │      Filter Chain (Tomcat), Interceptor Chain (Spring)
    │      "각 단계가 다음 단계 호출 여부를 결정"
    │
    ├─ Adapter
    │      Handler 타입을 추상화
    │      "DispatcherServlet이 핸들러 타입에 무관하게 동작"
    │      │
    │      └─ Strategy + Composite  (Adapter 내부)
    │             ArgumentResolver, ReturnValueHandler, MessageConverter
    │             "supports()로 적절한 구현체 선택"
    │
    ├─ Template Method
    │      HttpServlet → FrameworkServlet → DispatcherServlet 계층
    │      "공통 흐름은 상위 클래스, 가변 지점만 하위 클래스"
    │
    └─ Registry
           HandlerMapping, ViewResolver
           "시작 시 등록, 요청 시 O(1) 조회"
```