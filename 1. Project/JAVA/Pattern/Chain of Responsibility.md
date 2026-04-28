## 이 구조가 해결하려 한 건 "누가 무엇을 알아야 하는가"다

Filter/Interceptor Chain이 생긴 근본 이유는 재사용성 자체가 목표가 아니었다.

**"비즈니스 로직이 인프라를 알아서는 안 된다"** 는 압력이 먼저였고, 그 결과로 재사용성이 따라온 것이다.

```java
// 이게 진짜 문제였던 것
public class OrderService {
    public void placeOrder(HttpServletRequest req) {
        // 비즈니스 로직인데 왜 HTTP를 알아야 하지?
        String token = req.getHeader("Authorization");
        if (!validate(token)) throw new UnauthorizedException();
        
        // 비즈니스 로직인데 왜 인코딩을 알아야 하지?
        req.setCharacterEncoding("UTF-8");
        
        // 진짜 하고 싶은 것
        createOrder(...);
    }
}
```

OrderService가 JWT 검증 방식을 알아야 할 이유가 없다.  
JWT → OAuth2로 바뀌면 OrderService를 수정해야 하는 상황 자체가 설계 실패다.

---

## 세 가지 설계 원칙이 동시에 작용했다

### 1. 관심사 분리 (네가 말한 것)

각 필터는 **하나의 인프라 관심사만** 담당한다.

```
AuthFilter        → "이 요청이 인증됐는가"만
LoggingFilter     → "이 요청을 기록하라"만
EncodingFilter    → "인코딩을 맞춰라"만
RateLimitFilter   → "한도를 초과했는가"만
```

이게 분리되면 자연스럽게 재사용 가능해지는 것이고, 재사용이 목적이 아니다.

### 2. 순서 의존성의 명시적 관리

처리 단계 사이에 **암묵적 의존**이 있다.

```
RateLimit → Auth 순서라면:
  미인증 사용자도 rate limit 카운터 소모
  → 공격자가 의도적으로 정상 사용자 차단 가능 (Rate Limit 소진 공격)

Auth → RateLimit 순서라면:
  인증된 사용자에게만 rate limit 적용
  → 의도한 동작
```

이 순서 의존성을 코드 안에 숨기지 않고, `setOrder()`라는 **명시적 계약**으로 끌어낸 것이다.

### 3. 횡단 관심사(Cross-cutting Concern)의 격리

인증, 로깅, 트랜잭션은 **모든 요청에 걸쳐있다**. OOP로는 상속이나 유틸 클래스로 해결하려다 결국 산으로 가게 된다.

```java
// OOP로 해결하려 했던 시도 — 실패 사례
public abstract class AuthenticatedServlet extends HttpServlet {
    protected void doPost(HttpServletRequest req, ...) {
        checkAuth(req);      // 상위 클래스에서 강제
        doPostInternal(req); // 하위 클래스가 구현
    }
}

// 문제: 인증 + 로깅 + 인코딩 전부 필요하면?
// Java는 다중 상속 없음 → 상속 체인이 깊어지거나 불가능
public class OrderServlet extends LoggingServlet 
    extends AuthServlet  // 컴파일 에러
```

체인 구조는 이걸 **수직 상속 대신 수평 조합**으로 해결한다.

---

## 재사용성은 결과이지 목적이 아니다

정확하게 말하면 이 구조가 챙긴 건 세 가지다.

|설계 목표|실제 의미|
|---|---|
|관심사 분리|비즈니스 로직이 인프라를 모르게 한다|
|순서 제어|처리 단계 간 암묵적 의존을 명시적으로 만든다|
|조합 가능성|상속 없이 기능을 붙이고 뗄 수 있게 한다|

재사용성은 **관심사가 제대로 분리된 코드에서 나오는 부산물**이다.  
"재사용하려고 이렇게 만들었다"가 아니라, "제대로 나눴더니 재사용이 됐다"가 더 정확한 표현이다.

---

다음으로 자연스럽게 이어지는 질문이 있다면:

- **"그럼 AOP는 Filter/Interceptor와 뭐가 다른가"** — 같은 횡단 관심사 문제를 다른 레이어에서 다른 방식으로 해결한다. Spring에서 이 둘이 공존하는 이유가 있다
- **"조합 가능성을 극대화한 게 Strategy + Chain 조합인가"** — Spring Security가 정확히 이렇게 설계되어 있다
# Chain of Responsibility — Spring Filter Chain / Interceptor Chain 심층 분석

---

## 1. Problem First

### Filter 없던 시절 — 서블릿 직접 처리

```java
// 2000년대 초 — 모든 서블릿마다 공통 로직 복붙
public class OrderServlet extends HttpServlet {
    protected void doPost(HttpServletRequest req, HttpServletResponse res) {
        // 모든 서블릿에 이 블록이 복붙됨
        String encoding = req.getHeader("Content-Type");
        if (encoding == null || !encoding.contains("UTF-8")) {
            res.sendError(400);
            return;
        }
        
        String token = req.getHeader("Authorization");
        if (token == null || !tokenService.validate(token)) {
            res.sendError(401);
            return;
        }
        
        // 실제 비즈니스 로직
        processOrder(req, res);
    }
}

// PaymentServlet, UserServlet, ProductServlet... 전부 동일한 블록 복붙
// 인증 로직 바꾸려면 → 전체 서블릿 수정
```

**근본 문제는 단순 중복이 아니다.**

```
장애 시나리오:
- 인증 로직에 버그 발견 → 서블릿 50개 전부 수정
- 수정 중 OrderServlet은 빠뜨림 → 프로덕션에서 인증 우회 가능
- "어느 서블릿에 rate limit 걸었더라?" → 전수조사 필요
- 새 요구사항: 모든 요청에 요청 ID 붙여서 로깅 → 서블릿 50개 또 수정
```

**코드 중복이 아니라, 관심사 분리 불가능이 문제다.**  
비즈니스 로직 담당 객체가 인프라 관심사(인코딩, 인증, 로깅)를 알아야 하는 구조적 결함.

---

## 2. Mechanics

### 2-1. Servlet Filter — Tomcat `ApplicationFilterChain` 내부

```
HTTP 요청
    ↓
Connector (Coyote) → Request 파싱
    ↓
StandardEngineValve → StandardHostValve → StandardContextValve
    ↓
ApplicationFilterChain.doFilter()   ← 여기서부터 Filter Chain
    ↓
    [Filter 0] → [Filter 1] → [Filter 2] → Servlet.service()
                                                   ↓
    [Filter 0] ← [Filter 1] ← [Filter 2] ← 응답 반환
```

**`ApplicationFilterChain` 핵심 구조 (Tomcat 소스):**

```java
// org.apache.catalina.core.ApplicationFilterChain
public final class ApplicationFilterChain implements FilterChain {
    
    // 핵심: 배열 + 정수 카운터로 구현
    private ApplicationFilterConfig[] filters = new ApplicationFilterConfig[0];
    private int pos = 0;        // 현재 실행 위치
    private int n = 0;          // 총 필터 수
    private Servlet servlet = null;
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response)
            throws IOException, ServletException {
        
        if (pos < n) {
            // 현재 위치 필터 꺼내고 pos 증가
            ApplicationFilterConfig filterConfig = filters[pos++];
            Filter filter = filterConfig.getFilter();
            
            // 필터에게 this(체인 자신)를 넘김
            // 필터가 chain.doFilter() 호출하면 → 다시 이 메서드로 돌아와서 pos++ 반복
            filter.doFilter(request, response, this);
            
        } else {
            // 모든 필터 통과 → 실제 서블릿 실행
            servlet.service(request, response);
        }
    }
}
```

**호출 스택이 어떻게 쌓이는지:**

```
doFilter() pos=0 → AuthFilter.doFilter() 진입
    doFilter() pos=1 → LoggingFilter.doFilter() 진입
        doFilter() pos=2 → RateLimitFilter.doFilter() 진입
            doFilter() pos=3 (pos >= n) → servlet.service() 실행
                ↓ 응답 생성
            RateLimitFilter.doFilter() chain 호출 이후 코드 실행  ← 여기가 핵심
        LoggingFilter.doFilter() chain 호출 이후 코드 실행
    AuthFilter.doFilter() chain 호출 이후 코드 실행
```

**중단 메커니즘:**

```java
public class AuthFilter implements Filter {
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
        if (!isAuthenticated(req)) {
            ((HttpServletResponse) res).sendError(401);
            return;
            // chain.doFilter() 미호출 → pos가 더 이상 증가 안 함
            // 스택이 여기서 언와인드 → 이전 필터들의 chain 이후 코드만 실행됨
        }
        chain.doFilter(req, res);  // 이 줄이 재귀적으로 다음 doFilter() 호출
    }
}
```

**Filter 등록 순서 결정 위치:**

```java
// Spring Boot — FilterRegistrationBean 사용 시
@Bean
public FilterRegistrationBean<AuthFilter> authFilter() {
    FilterRegistrationBean<AuthFilter> bean = new FilterRegistrationBean<>();
    bean.setFilter(new AuthFilter());
    bean.setOrder(1);           // ApplicationFilterChain 배열에서의 위치
    bean.addUrlPatterns("/*");
    return bean;
}

// 내부적으로 StandardContext가 이 order 기준으로
// ApplicationFilterChain.filters[] 배열을 정렬해서 채움
```

---

### 2-2. Spring Interceptor — `HandlerExecutionChain` 내부

Filter는 Servlet 컨테이너(Tomcat) 레이어.  
Interceptor는 Spring MVC 레이어. **DispatcherServlet 안에서** 동작한다.

```
ApplicationFilterChain (Tomcat)
    ↓ 모든 Filter 통과
DispatcherServlet.service()
    ↓
HandlerExecutionChain (Spring MVC)
    ↓
    preHandle() — 순방향
    HandlerAdapter.handle() — 실제 Controller 실행
    postHandle() — 역방향
    afterCompletion() — 역방향 (예외 발생해도 반드시 실행)
```

**`HandlerExecutionChain` 핵심 구조 (Spring 소스):**

```java
// org.springframework.web.servlet.HandlerExecutionChain
public class HandlerExecutionChain {
    
    private final Object handler;  // 실제 Controller 메서드
    private final List<HandlerInterceptor> interceptorList = new ArrayList<>();
    private int interceptorIndex = -1;  // preHandle 성공한 마지막 인덱스
    
    // DispatcherServlet이 호출
    boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) {
        for (int i = 0; i < this.interceptorList.size(); i++) {
            HandlerInterceptor interceptor = this.interceptorList.get(i);
            
            if (!interceptor.preHandle(request, response, this.handler)) {
                // false 반환 → 중단
                // 이미 통과한 인터셉터들의 afterCompletion은 보장해야 함
                triggerAfterCompletion(request, response, null);
                return false;
            }
            this.interceptorIndex = i;  // 성공한 마지막 인덱스 기록
        }
        return true;
    }
    
    void applyPostHandle(HttpServletRequest request, HttpServletResponse response, 
                         ModelAndView mv) {
        // 역순 실행
        for (int i = this.interceptorList.size() - 1; i >= 0; i--) {
            this.interceptorList.get(i).postHandle(request, response, this.handler, mv);
        }
    }
    
    void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response, 
                                Exception ex) {
        // interceptorIndex까지만 역순 실행 (preHandle 통과한 것들만)
        for (int i = this.interceptorIndex; i >= 0; i--) {
            try {
                this.interceptorList.get(i).afterCompletion(request, response, this.handler, ex);
            } catch (Throwable ex2) {
                // afterCompletion 예외는 삼킴 — 원래 예외 전파 보장
                logger.error("...", ex2);
            }
        }
    }
}
```

**`interceptorIndex`가 왜 중요한가:**

```
인터셉터: [A, B, C]

시나리오: A.preHandle=true, B.preHandle=true, C.preHandle=false

     A.preHandle() → true  (interceptorIndex = 0)
     B.preHandle() → true  (interceptorIndex = 1)
     C.preHandle() → false (interceptorIndex = 1에서 멈춤)
         ↓
     triggerAfterCompletion() — i = interceptorIndex(1) 부터 역순
         B.afterCompletion()
         A.afterCompletion()
         (C는 preHandle 통과 못했으므로 afterCompletion 호출 안 됨)
```

자원 획득(DB 커넥션, 락 등)을 preHandle에서 했다면, afterCompletion에서 반드시 해제해야 한다.  
이 설계 덕분에 C가 실패해도 A, B의 afterCompletion은 **반드시** 실행된다.

---

### 2-3. Filter vs Interceptor — 실행 가능 범위 차이

```
┌─────────────────────────────────────────────────────────────┐
│  Tomcat                                                       │
│  ┌─────────────────────────────────────────────────────┐     │
│  │  ApplicationFilterChain                              │     │
│  │  Filter1 → Filter2 → Filter3                        │     │
│  │  ┌───────────────────────────────────────────────┐  │     │
│  │  │  DispatcherServlet                             │  │     │
│  │  │  HandlerExecutionChain                        │  │     │
│  │  │  preHandle → Controller → postHandle          │  │     │
│  │  │                                               │  │     │
│  │  │  Spring Context (Bean, DI) 접근 가능          │  │     │
│  │  └───────────────────────────────────────────────┘  │     │
│  │                                                      │     │
│  │  Filter = Spring Context 밖 (원칙상)                 │     │
│  └─────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

| 항목             | Filter                                 | Interceptor                       |
| -------------- | -------------------------------------- | --------------------------------- |
| 레이어            | Servlet Container (Tomcat)             | Spring MVC (DispatcherServlet 내부) |
| Spring Bean 주입 | `DelegatingFilterProxy`로 우회 가능         | 직접 가능                             |
| 적용 대상          | 모든 요청 (정적 리소스 포함)                      | DispatcherServlet이 처리하는 요청만       |
| 응답 바디 조작       | 가능 (`ContentCachingResponseWrapper` 등) | 불가 (이미 커밋된 경우 많음)                 |
| 예외 처리          | `@ControllerAdvice` 도달 전 예외 → 직접 처리 필요 | `@ControllerAdvice`로 전파 가능        |

**`DelegatingFilterProxy` 동작 원리:**

```java
// Spring Security의 실제 구현 패턴
// Filter는 Tomcat이 생성하지만, 실제 로직은 Spring Bean에 위임

public class DelegatingFilterProxy extends GenericFilterBean {
    
    private String targetBeanName;  // Spring Bean 이름
    private volatile Filter delegate;  // 실제 처리할 Spring Bean
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
        // 최초 1회 Spring ApplicationContext에서 Bean 조회 (lazy init)
        Filter delegateToUse = this.delegate;
        if (delegateToUse == null) {
            delegateToUse = initDelegate(getWebApplicationContext());
            this.delegate = delegateToUse;
        }
        // Spring Bean에게 실제 처리 위임
        delegateToUse.doFilter(request, response, chain);
    }
}

// Spring Security는 이걸 이용해
// "springSecurityFilterChain" 이름의 Bean(FilterChainProxy)에 위임
```

---

### 2-4. Spring Security의 FilterChainProxy — Chain of Chain

Spring Security는 Chain of Responsibility를 한 단계 더 중첩한다.

```
ApplicationFilterChain (Tomcat)
    ↓
DelegatingFilterProxy ("springSecurityFilterChain")
    ↓
FilterChainProxy (Spring Security)
    ↓ URL 패턴 매칭으로 SecurityFilterChain 선택
SecurityFilterChain[0] — "/api/**"
    [UsernamePasswordAuthenticationFilter]
    [JwtAuthenticationFilter]
    [ExceptionTranslationFilter]
    [AuthorizationFilter]

SecurityFilterChain[1] — "/admin/**"
    [다른 필터 조합]
```

```java
// FilterChainProxy 핵심 로직
public class FilterChainProxy extends GenericFilterBean {
    
    private List<SecurityFilterChain> filterChains;
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
        // 요청 URL에 매칭되는 첫 번째 SecurityFilterChain 선택
        List<Filter> filters = getFilters(request);
        
        if (filters == null || filters.isEmpty()) {
            chain.doFilter(request, response);  // 매칭 없으면 스킵
            return;
        }
        
        // 선택된 SecurityFilterChain으로 새 VirtualFilterChain 구성
        VirtualFilterChain vfc = new VirtualFilterChain(chain, filters);
        vfc.doFilter(request, response);
    }
}
```

---

## 3. 공식 근거

**Filter Chain:**

- Jakarta Servlet Specification 6.0, Section 6.2.4 "Filters and the RequestDispatcher":
    
    > "Each filter has access to a FilterConfig object from which it can obtain initialization parameters and a reference to the ServletContext."  
    > _(각 필터는 초기화 파라미터와 ServletContext 참조를 얻을 수 있는 FilterConfig 객체에 접근한다.)_
    
- Tomcat 소스: `org.apache.catalina.core.ApplicationFilterChain` (Apache Tomcat GitHub)

**HandlerExecutionChain:**

- Spring Framework Reference — [Web MVC / DispatcherServlet / Interception](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-servlet/handlermapping-interceptor.html):
    
    > "All HandlerInterceptor methods return a boolean value. [...] the preHandle(..) method returns false to indicate to DispatcherServlet that it should skip further execution and skip the processing of the interceptor chain."  
    > _(preHandle이 false를 반환하면 DispatcherServlet은 이후 실행을 중단한다.)_
    
- Spring Framework 소스: `org.springframework.web.servlet.HandlerExecutionChain`

**DelegatingFilterProxy:**

- Spring Security Reference — [Architecture / DelegatingFilterProxy](https://docs.spring.io/spring-security/reference/servlet/architecture.html#servlet-delegatingfilterproxy):
    
    > "Spring provides a Filter implementation named DelegatingFilterProxy that allows bridging between the Servlet container's lifecycle and Spring's ApplicationContext."  
    > _(DelegatingFilterProxy는 서블릿 컨테이너 생명주기와 Spring ApplicationContext 사이를 연결한다.)_
    

---

## 4. 이 설계를 이렇게 한 이유

### 얻는 것

- **개방/폐쇄 원칙** — 새 필터/인터셉터 추가 시 기존 코드 무수정
- **단일 책임** — AuthFilter는 인증만, LoggingFilter는 로깅만
- **순서 제어 가능** — `@Order`, `setOrder()`로 런타임 조합

### 잃는 것 — 구체적으로

**1. 디버깅 비용**

```
"왜 이 요청이 401을 반환하는가?"
→ SecurityFilterChain 내 15개 필터를 순서대로 확인해야 함
→ 어느 필터에서 막혔는지 로그 없으면 추적 불가
→ 해결책: TRACE 로그 활성화 (logging.level.org.springframework.security=TRACE)
   → 그러면 요청마다 로그 수백 줄 — 프로덕션에서 켤 수 없음
```

**2. 예외 처리 불일치**

```
Filter에서 던진 예외
→ DispatcherServlet 밖 → @ControllerAdvice 도달 안 함
→ Tomcat의 기본 에러 페이지 or 빈 응답 반환
→ API 서버에서 Filter 예외 = 클라이언트에게 HTML 에러 페이지가 내려갈 수 있음

Interceptor에서 던진 예외
→ DispatcherServlet 안 → @ControllerAdvice 도달 가능
```

**3. 순서 오류는 런타임에 발견됨**

```java
// 컴파일 타임에 잡히지 않는 버그
@Bean
public FilterRegistrationBean<JwtFilter> jwtFilter() {
    bean.setOrder(10);  // 인증 필터
    return bean;
}

@Bean  
public FilterRegistrationBean<RateLimitFilter> rateLimitFilter() {
    bean.setOrder(5);   // Rate limit이 인증보다 먼저 실행됨
    // 의도: 인증된 사용자에게만 rate limit 적용
    // 결과: 미인증 요청도 rate limit 카운터에 포함됨
    return bean;
}
// 이 버그는 부하 테스트 or 프로덕션에서 발견됨
```

**4. 상태 공유의 위험**

```java
// Filter/Interceptor는 보통 싱글턴 Bean
// 인스턴스 변수에 상태 저장 → 멀티스레드 환경에서 레이스 컨디션
public class BadFilter implements Filter {
    private int requestCount = 0;  // 위험: 모든 요청이 공유
    
    public void doFilter(...) {
        requestCount++;  // 원자적이지 않음
    }
}
```

---

## 5. 이어지는 개념

**1순위 — `ThreadLocal` / `RequestContextHolder`**

- **이유:** Filter와 Interceptor 사이, 또는 Interceptor에서 Controller까지 요청 범위 데이터를 어떻게 전달하는가. `SecurityContextHolder`, MDC 로깅 등이 모두 이 위에 구현됨. Chain 이해 후 "그래서 Security Context는 어디 저장되나"가 자연스러운 다음 질문.

**2순위 — Spring Security Filter Chain 전체 구조**

- **이유:** 실무에서 인증/인가 문제의 70%는 "어느 필터가 어떤 순서로 실행되는가"를 모르기 때문에 발생. `UsernamePasswordAuthenticationFilter`, `JwtAuthenticationFilter`, `ExceptionTranslationFilter`의 책임 분리를 이해하면 Security 관련 버그 추적 속도가 완전히 달라짐.

**3순위 — `OncePerRequestFilter`**

- **이유:** 실무에서 커스텀 Filter를 만들 때 `Filter` 대신 이것을 써야 하는 이유가 있음 (forward/include 시 필터 중복 실행 문제). Chain 동작을 이해한 후 "그럼 Filter가 한 요청에 두 번 실행될 수 있나?"가 자연스러운 의문.



---
# 1. AOP vs Filter/Interceptor

## Problem First

Filter/Interceptor로 못 막는 곳이 있다.

```java
// 이런 요구사항이 생겼다
// "모든 Service 메서드 실행 시간을 측정하라"

@Service
public class OrderService {
    
    public void placeOrder(...) { ... }
    public void cancelOrder(...) { ... }
    public void getOrderHistory(...) { ... }
}

@Service  
public class PaymentService {
    public void processPayment(...) { ... }
    public void refund(...) { ... }
}
```

```
Filter/Interceptor로 해결 시도:
  Filter   → HTTP 요청 단위. Service 메서드 단위 측정 불가
  Interceptor → Controller 진입 전/후. Service 내부 진입 불가

  OrderService.placeOrder() 안에서
  PaymentService.processPayment()를 호출한다면?
  → Interceptor는 이 내부 호출을 볼 수 없다
```

**근본 문제:** Filter/Interceptor는 **HTTP 요청 경계**에서만 작동한다. 객체 간 메서드 호출 경계에는 끼어들 수 없다.

---

## Mechanics

### AOP가 끼어드는 위치

```
HTTP 요청
    ↓
Filter Chain        ← Filter 작동 위치
    ↓
DispatcherServlet
    ↓
Interceptor         ← Interceptor 작동 위치
    ↓
Controller
    ↓
Service.method()    ← AOP 작동 위치
    ↓
Repository.method() ← AOP 작동 위치
```

HTTP와 무관하게, **메서드 호출 자체**가 경계다.

---

### Spring AOP 내부 — 프록시 기반
Spring AOP는 바이트코드를 직접 조작하지 않는다.  
**프록시 객체**로 원본 Bean을 감싼다.

```java
// 원본 클래스
@Service
public class OrderService {
    public void placeOrder(OrderRequest req) {
        // 실제 로직
    }
}

// @Transactional 붙이면 Spring이 런타임에 이런 프록시를 생성
public class OrderService$$SpringCGLIB$$0 extends OrderService {
    
    private OrderService target;  // 원본 객체
    
    @Override
    public void placeOrder(OrderRequest req) {
        // Advice 실행 (Before)
        TransactionStatus tx = transactionManager.getTransaction(...);
        
        try {
            target.placeOrder(req);  // 원본 메서드 호출
            transactionManager.commit(tx);
        } catch (Exception e) {
            transactionManager.rollback(tx);
            throw e;
        }
        // Advice 실행 (After)
    }
}
```

**Spring이 프록시를 생성하는 두 가지 방법:**

```
1. JDK Dynamic Proxy
   - 대상 클래스가 인터페이스를 구현할 때
   - java.lang.reflect.Proxy 사용
   - 인터페이스 타입으로만 주입 가능

2. CGLIB Proxy (Spring Boot 기본값)
   - 인터페이스 없어도 가능
   - 클래스를 상속해서 서브클래스 생성
   - final 클래스/메서드에는 적용 불가 (상속 불가이므로)
```

```java
// 이게 왜 중요한가 — 실수하기 쉬운 지점

@Service
public class OrderService {
    
    @Autowired
    private OrderService self;  // 자기 자신을 주입
    
    public void placeOrder() {
        self.validateAndPlace();  // 프록시를 통한 호출 → AOP 작동
    }
    
    @Transactional
    public void validateAndPlace() { ... }
    
    public void directCall() {
        validateAndPlace();  
        // this.validateAndPlace() — 프록시 우회
        // → @Transactional 무시됨
        // → 트랜잭션 없이 실행되는 버그
    }
}
```

**Self-invocation 문제:** 같은 클래스 내부에서 메서드를 직접 호출하면 프록시를 거치지 않는다. `@Transactional`이 붙어있어도 트랜잭션이 시작되지 않는다. 실무에서 매우 자주 만나는 버그다.

---

### AOP 용어와 실제 동작 매핑

```java
@Aspect
@Component
public class TimingAspect {
    
    // Pointcut — "어디에 끼어들 것인가"
    // execution(* com.example.service.*.*(..))
    // = com.example.service 패키지의 모든 클래스, 모든 메서드
    
    @Around("execution(* com.example.service.*.*(..))")
    public Object measureTime(ProceedingJoinPoint joinPoint) throws Throwable {
        
        // Before 시점
        long start = System.currentTimeMillis();
        
        // 원본 메서드 실행
        Object result = joinPoint.proceed();
        
        // After 시점 (정상 반환)
        long elapsed = System.currentTimeMillis() - start;
        log.info("{} took {}ms", joinPoint.getSignature(), elapsed);
        
        return result;
    }
}
```

```
Aspect     = 횡단 관심사 모듈 전체 (TimingAspect 클래스)
Pointcut   = 어디에 적용할지 표현식
JoinPoint  = 실제로 끼어든 그 메서드 호출 시점
Advice     = 끼어들어서 실행할 코드 (@Around, @Before, @After)
Weaving    = 프록시 생성해서 Advice를 끼워넣는 과정
```

---

### Filter / Interceptor / AOP 비교 정리

```
┌─────────────────┬──────────────┬─────────────────┬──────────────────┐
│                 │    Filter    │  Interceptor    │      AOP         │
├─────────────────┼──────────────┼─────────────────┼──────────────────┤
│ 작동 레이어     │ Servlet      │ Spring MVC      │ Spring Bean      │
│ 작동 경계       │ HTTP 요청    │ Controller 진입 │ 메서드 호출      │
│ HTTP 객체 접근  │ O            │ O               │ 기본 X           │
│ Spring Bean     │ 우회 필요    │ O               │ O                │
│ 적용 대상       │ 모든 요청    │ MVC 요청만      │ 모든 Bean 메서드 │
│ 주 용도         │ 인코딩, 보안 │ 인증, 로깅      │ 트랜잭션, 측정   │
└─────────────────┴──────────────┴─────────────────┴──────────────────┘
```

**잃는 것:**

- 프록시 기반이라 `final` 클래스/메서드 적용 불가
- Self-invocation 함정 — 같은 클래스 내부 호출에서 AOP 무시됨
- Pointcut 표현식이 복잡해질수록 어느 메서드에 AOP가 걸려있는지 파악 어려움
- 성능 — 모든 메서드에 `@Around` 걸면 프록시 호출 오버헤드가 누적됨

---
---

# 2. Strategy + Chain 조합 — Spring Security

## Problem First

인증 방식이 하나가 아니다.

```
같은 서비스에서:
  /api/**        → JWT Bearer Token
  /oauth/**      → OAuth2
  /admin/**      → Session 기반
  /actuator/**   → Basic Auth (내부망 전용)
  /public/**     → 인증 없음
```

```java
// 하나의 Filter에서 전부 처리하려 하면
public class AuthFilter implements Filter {
    public void doFilter(ServletRequest req, ...) {
        String path = ((HttpServletRequest) req).getRequestURI();
        
        if (path.startsWith("/api/")) {
            // JWT 검증 로직 100줄
        } else if (path.startsWith("/oauth/")) {
            // OAuth2 로직 200줄
        } else if (path.startsWith("/admin/")) {
            // Session 검증 로직 80줄
        }
        // ... 계속 늘어남
    }
}
```

**근본 문제:** 인증 전략이 추가될 때마다 이 클래스를 수정해야 한다. 인증 방식 하나의 버그가 전체에 영향을 준다.

---

## Mechanics

### Spring Security의 구조 — Chain of Chain

```
DelegatingFilterProxy
    ↓
FilterChainProxy
    ↓ URL 패턴으로 SecurityFilterChain 선택
    ├── SecurityFilterChain[0]  "/api/**"
    │       [SecurityContextPersistenceFilter]
    │       [JwtAuthenticationFilter]        ← 커스텀
    │       [ExceptionTranslationFilter]
    │       [AuthorizationFilter]
    │
    ├── SecurityFilterChain[1]  "/admin/**"
    │       [SecurityContextPersistenceFilter]
    │       [UsernamePasswordAuthenticationFilter]
    │       [ExceptionTranslationFilter]
    │       [AuthorizationFilter]
    │
    └── SecurityFilterChain[2]  "/**"
            [permitAll]
```

각 SecurityFilterChain은 **독립적인 필터 조합**이다.  
`/api/**`의 JWT 필터 버그가 `/admin/**` 체인에 영향을 주지 않는다.

---

### Strategy 패턴이 끼어드는 지점

Chain이 "언제 실행하는가"를 결정한다면,  
Strategy는 "어떻게 인증하는가"를 결정한다.

```java
// AuthenticationManager — Strategy의 인터페이스
public interface AuthenticationManager {
    Authentication authenticate(Authentication authentication)
        throws AuthenticationException;
}

// ProviderManager — 여러 Strategy를 순서대로 시도하는 구현체
public class ProviderManager implements AuthenticationManager {
    
    private List<AuthenticationProvider> providers;  // Strategy 목록
    
    @Override
    public Authentication authenticate(Authentication authentication) {
        
        for (AuthenticationProvider provider : providers) {
            
            // 이 provider가 이 인증 방식을 처리할 수 있는가?
            if (!provider.supports(authentication.getClass())) {
                continue;  // 못 하면 다음 provider로
            }
            
            try {
                // 처리할 수 있는 provider가 실제 인증 수행
                Authentication result = provider.authenticate(authentication);
                if (result != null) {
                    return result;  // 성공하면 즉시 반환
                }
            } catch (AuthenticationException e) {
                lastException = e;
            }
        }
        
        throw lastException;  // 모든 provider 실패
    }
}
```

```java
// 각 AuthenticationProvider = 하나의 인증 전략
public class JwtAuthenticationProvider implements AuthenticationProvider {
    
    @Override
    public Authentication authenticate(Authentication auth) {
        String token = (String) auth.getCredentials();
        // JWT 검증 로직
        return new UsernamePasswordAuthenticationToken(user, null, authorities);
    }
    
    @Override
    public boolean supports(Class<?> authClass) {
        // 이 Provider는 JwtAuthentication 타입만 처리
        return JwtAuthenticationToken.class.isAssignableFrom(authClass);
    }
}

public class OAuth2AuthenticationProvider implements AuthenticationProvider {
    
    @Override
    public boolean supports(Class<?> authClass) {
        return OAuth2AuthenticationToken.class.isAssignableFrom(authClass);
    }
    // ...
}
```

**두 패턴의 역할 분리:**

```
Chain of Responsibility
  → "이 요청을 처리할 단계들을 순서대로 통과시킨다"
  → 필터 순서, 실행 여부 결정

Strategy
  → "이 인증 요청을 처리하는 방법이 여러 개 있다. 적합한 것을 선택한다"
  → 실제 인증 알고리즘 교체 가능하게 함
```

---

### 전체 흐름 한 번에

```java
// JWT 인증 요청이 들어왔을 때 실제 흐름

// 1. FilterChainProxy — Chain 선택
//    "/api/orders" → SecurityFilterChain[0] 선택

// 2. JwtAuthenticationFilter (Chain의 한 단계)
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    
    private AuthenticationManager authenticationManager;  // Strategy 진입점
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, ...) {
        String token = extractToken(request);
        if (token == null) {
            chain.doFilter(request, response);
            return;
        }
        
        // Chain → Strategy로 위임
        // "이 토큰을 어떻게 검증할지는 내가 모른다. Manager에게 맡긴다"
        JwtAuthenticationToken authRequest = new JwtAuthenticationToken(token);
        Authentication result = authenticationManager.authenticate(authRequest);
        
        SecurityContextHolder.getContext().setAuthentication(result);
        chain.doFilter(request, response);
    }
}

// 3. ProviderManager — supports() 체크 후 JwtAuthenticationProvider 선택
// 4. JwtAuthenticationProvider — 실제 JWT 검증
// 5. 성공 → SecurityContext에 Authentication 저장
// 6. AuthorizationFilter — 권한 확인
```

---

## 공식 근거

**AOP 프록시 메커니즘:**

- Spring Framework Reference — [AOP / Proxying Mechanisms](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html):
    
    > "Spring AOP uses either JDK dynamic proxies or CGLIB to create the proxy for a given target object."  
    > _(Spring AOP는 대상 객체에 대한 프록시를 생성하기 위해 JDK 동적 프록시 또는 CGLIB을 사용한다.)_
    

**Self-invocation 제약:**

- Spring Framework Reference — [AOP / Understanding AOP Proxies](https://docs.spring.io/spring-framework/reference/core/aop/understanding-aop-proxies.html):
    
    > "The key thing to understand here is that the client code inside the main(..) method has a reference to the proxy. This means that method calls on that object reference are calls on the proxy."  
    > _(핵심은 클라이언트 코드가 프록시에 대한 참조를 갖는다는 것이다. 즉, 그 객체 참조에 대한 메서드 호출은 프록시에 대한 호출이다.)_
    

**ProviderManager:**

- Spring Security Reference — [Authentication / ProviderManager](https://docs.spring.io/spring-security/reference/servlet/authentication/architecture.html#servlet-authentication-providermanager):
    
    > "ProviderManager delegates to a List of AuthenticationProviders. Each AuthenticationProvider has an opportunity to indicate that authentication should be successful, fail, or indicate it cannot make a decision."  
    > _(ProviderManager는 AuthenticationProvider 목록에 위임한다. 각 Provider는 인증 성공, 실패, 또는 판단 불가를 나타낼 수 있다.)_
    

---

## 이 설계를 이렇게 한 이유

### 얻는 것

- 인증 방식 추가 = `AuthenticationProvider` 하나 추가, 기존 코드 무수정
- URL 패턴별 완전히 다른 보안 정책 독립 유지
- 테스트 용이 — 각 Provider를 독립적으로 단위 테스트 가능

### 잃는 것 — 구체적으로

**1. 진입 장벽**

```
Spring Security 처음 접하는 개발자:
"403이 왜 나오지?" → 어느 Filter인지, 어느 Provider인지, 
SecurityContext가 비어있는지, 권한이 없는지 — 
확인해야 할 레이어가 5개 이상
```

**2. 과도한 추상화의 위험**

```java
// 간단한 요구사항: "JWT 검증하면 됨"
// 실제로 건드려야 하는 것:
SecurityFilterChain         // 필터 체인 구성
JwtAuthenticationFilter     // 필터 구현
JwtAuthenticationToken      // Authentication 구현체
JwtAuthenticationProvider   // Provider 구현
UserDetailsService          // 사용자 조회

// 단순한 인증 하나에 클래스 5개
// 팀원이 Security 모르면 → 아무도 못 건드리는 코드
```

**3. 필터 순서 실수 — 런타임에 발견**

```java
// ExceptionTranslationFilter가 JwtFilter보다 앞에 있으면
// JWT 예외가 ExceptionTranslationFilter를 통과 못 하고 밖으로 튀어나옴
// → 클라이언트에게 의도치 않은 에러 형식 반환
// 컴파일 타임에 잡히지 않음
```

---

## 이어지는 개념

**1순위 — `@Transactional` 내부 동작**

- **이유:** 실무에서 AOP를 가장 많이 만나는 곳이 트랜잭션이다. Self-invocation 함정, 전파 레벨, 롤백 조건 — 이 모든 것이 AOP 프록시 메커니즘 위에서 작동한다. AOP를 이해했으면 `@Transactional`이 왜 그렇게 동작하는지가 바로 연결된다.

**2순위 — `SecurityContext` / `ThreadLocal`**

- **이유:** `SecurityContextHolder.getContext().getAuthentication()` — 이게 어떻게 요청 전체에서 같은 인증 객체를 반환하는가. Filter에서 저장한 것을 Controller에서 꺼낼 수 있는 이유가 ThreadLocal이다. Security Chain 이해 후 자연스러운 다음 질문이다.