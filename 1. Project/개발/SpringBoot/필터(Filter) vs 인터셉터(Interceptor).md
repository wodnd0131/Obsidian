# 필터(Filter) vs 인터셉터(Interceptor) — 왜 두 개인가

---

## 1. Problem First

### 필터와 인터셉터가 구분되지 않았다면 생기는 문제

**시나리오 A: 스프링 컨텍스트에 접근하지 못하는 인증**

```java
// 만약 필터만 존재한다면 — 스프링 빈에 접근 불가
public class AuthFilter implements Filter {
    // 이 시점에 ApplicationContext가 없다
    // JwtTokenValidator는 스프링 빈인데, 어떻게 가져올 것인가?
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
        // 방법 1: 직접 new JwtTokenValidator() → 빈 라이프사이클, AOP 프록시 전부 무시됨
        // 방법 2: WebApplicationContextUtils로 꺼내기 → 매 요청마다 컨텍스트 룩업 비용
        JwtTokenValidator validator = new JwtTokenValidator(); // 트랜잭션, 캐시 AOP 없음
    }
}
```

**시나리오 B: 인터셉터만 존재한다면 — 서블릿 도달 이전 처리 불가**

```
클라이언트 → [Tomcat이 처리해야 할 CORS/인코딩/보안헤더] → DispatcherServlet → 인터셉터
```

멀티파트 파싱, 문자 인코딩 설정은 **서블릿 컨텍스트 레벨**에서 처리되어야 한다.  
인터셉터는 이미 파싱이 끝난 `HttpServletRequest`를 받는다.  
인터셉터만 있었다면 이 레이어에서 처리할 방법이 없다.

**시나리오 C: HandlerMethod 정보가 없는 권한 처리**

```java
// 필터에서 @PreAuthorize("hasRole('ADMIN')") 어노테이션을 읽으려면?
public class RoleFilter implements Filter {
    public void doFilter(ServletRequest req, ...) {
        // HandlerMethod를 알 방법이 없다
        // 어떤 컨트롤러의 어떤 메서드가 호출될지 필터 시점에는 미결정
        // URL 패턴 파싱을 직접 구현해야 함 → 스프링 라우팅과 이중 관리
    }
}
```

---

## 2. Mechanics

### 핵심 전제: 두 개의 컨테이너

```
┌─────────────────────────────────────────────────┐
│              Tomcat (Servlet Container)          │
│  ┌──────────────────────────────────────────┐   │
│  │         Filter Chain (javax.servlet)      │   │
│  │  Filter1 → Filter2 → Filter3             │   │
│  │                  ↓                        │   │
│  │         DispatcherServlet                 │   │
│  │    ┌──────────────────────────────────┐  │   │
│  │    │   Spring ApplicationContext      │  │   │
│  │    │   HandlerMapping                 │  │   │
│  │    │   Interceptor Chain              │  │   │
│  │    │   Handler(Controller)            │  │   │
│  │    └──────────────────────────────────┘  │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

이것이 핵심이다. **두 개의 독립된 컨테이너가 각자의 체인을 가진다.**

---

### Filter의 내부 구조

**스펙 근거:** Jakarta Servlet Specification 6.0, Section 6.2

> "A filter is a reusable piece of code that can transform the content of HTTP requests, responses, and header information."

**Filter 인터페이스 (jakarta.servlet.Filter)**

```java
public interface Filter {
    // Servlet Container가 필터를 초기화할 때 1회 호출
    // FilterConfig를 통해 web.xml 또는 @WebFilter의 설정값에 접근
    default void init(FilterConfig filterConfig) throws ServletException {}

    // 매 요청마다 호출 — ServletRequest/Response (HTTP에 종속되지 않음)
    void doFilter(ServletRequest request, ServletResponse response,
                  FilterChain chain) throws IOException, ServletException;

    default void destroy() {}
}
```

**FilterChain의 실제 실행 흐름 — ApplicationFilterChain (Tomcat 구현체)**

```java
// org.apache.catalina.core.ApplicationFilterChain
public void doFilter(ServletRequest request, ServletResponse response) {
    // 내부적으로 pos (현재 위치) 카운터로 재귀 없이 순차 실행
    if (pos < n) {
        ApplicationFilterConfig filterConfig = filters[pos++];
        Filter filter = filterConfig.getFilter();
        filter.doFilter(request, response, this); // this를 chain으로 넘김
        // 필터가 chain.doFilter()를 호출하면 다시 이 메서드로 돌아와 pos++ 진행
    } else {
        // 마지막 필터 이후 → 서블릿 실행
        servlet.service(request, response);
    }
}
```

**핵심:** `chain.doFilter()`를 호출하지 않으면 체인이 중단된다.  
이것이 인증 실패 시 요청을 막는 방식이다.

**Filter가 등록되는 시점**

```
Tomcat 시작
  → StandardContext.startInternal()
  → FilterDef 로딩 (web.xml 또는 @WebFilter 또는 FilterRegistrationBean)
  → ApplicationFilterConfig 생성
  → Filter.init() 호출
  → FilterMap에 URL 패턴 매핑 저장
```

Spring Boot에서는 `FilterRegistrationBean`이 `ServletContextInitializer`를 구현하여  
Tomcat의 `ServletContext`에 필터를 등록한다.

```java
// org.springframework.boot.web.servlet.FilterRegistrationBean
// → AbstractFilterRegistrationBean.register() 내부
servletContext.addFilter(name, this.filter); // Tomcat ServletContext에 직접 등록
```

---

### Interceptor의 내부 구조

**스펙 근거:** Spring Framework Reference, Web MVC — Interception

> "HandlerInterceptor is the Spring-specific extension to the Filter."

**HandlerInterceptor 인터페이스**

```java
public interface HandlerInterceptor {
    // DispatcherServlet이 HandlerAdapter에 위임하기 전
    // handler = HandlerMethod (컨트롤러 메서드 정보 포함)
    default boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                               Object handler) throws Exception {
        return true; // false 반환 시 이후 처리 중단
    }

    // HandlerAdapter.handle() 이후, View 렌더링 이전
    default void postHandle(HttpServletRequest request, HttpServletResponse response,
                            Object handler, @Nullable ModelAndView modelAndView) {}

    // View 렌더링 완료 후 (예외 발생 여부와 무관하게 호출)
    default void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                 Object handler, @Nullable Exception ex) {}
}
```

**DispatcherServlet 내부에서 인터셉터가 실행되는 흐름**

```java
// org.springframework.web.servlet.DispatcherServlet.doDispatch() 핵심 발췌
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) {
    HandlerExecutionChain mappedHandler = getHandler(request);
    // HandlerExecutionChain = Handler(컨트롤러) + 해당 요청에 매핑된 Interceptor[]

    HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

    // 1. 모든 인터셉터의 preHandle() 순서대로 실행
    if (!mappedHandler.applyPreHandle(request, response)) {
        return; // 하나라도 false 반환 시 즉시 중단
    }

    // 2. 실제 핸들러(컨트롤러) 실행
    ModelAndView mv = ha.handle(request, response, mappedHandler.getHandler());

    // 3. postHandle() — 역순으로 실행
    mappedHandler.applyPostHandle(request, response, mv);

    // 4. afterCompletion() — 예외 발생해도 실행 (역순)
    mappedHandler.triggerAfterCompletion(request, response, null);
}
```

**`applyPreHandle` 내부 — 역순 rollback 보장**

```java
// org.springframework.web.servlet.HandlerExecutionChain
boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) {
    for (int i = 0; i < this.interceptorList.size(); i++) {
        HandlerInterceptor interceptor = this.interceptorList.get(i);
        if (!interceptor.preHandle(request, response, this.handler)) {
            // i번째에서 실패하면 0 ~ i-1 까지 afterCompletion() 역순 호출
            triggerAfterCompletion(request, response, null);
            return false;
        }
        this.interceptorIndex = i; // 성공한 마지막 인덱스 기록
    }
    return true;
}
```

**인터셉터 등록 흐름**

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new AuthInterceptor())
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/login");
    }
}
```

내부적으로:

```
WebMvcConfigurationSupport.requestMappingHandlerMapping()
  → RequestMappingHandlerMapping 생성
  → InterceptorRegistry의 인터셉터들을 HandlerMapping에 등록
  → 각 요청 시 getHandler()에서 URL 패턴 매칭하여 HandlerExecutionChain에 조립
```

---

### 실행 시점 전체 타임라인

```
HTTP 요청
    │
    ▼
[Tomcat] HTTP 파싱, 스레드 할당
    │
    ▼
[Filter 1] doFilter() ──── 스프링 컨텍스트 없음, Servlet API만
    │
    ▼
[Filter 2] doFilter()
    │
    ▼
[DispatcherServlet.service()]
    │
    ├─ HandlerMapping → HandlerExecutionChain 결정
    │
    ▼
[Interceptor 1] preHandle() ── HandlerMethod 알 수 있음, 스프링 빈 주입 가능
    │
    ▼
[Interceptor 2] preHandle()
    │
    ▼
[Controller Method 실행]
    │
    ▼
[Interceptor 2] postHandle() ── 역순
    │
    ▼
[Interceptor 1] postHandle()
    │
    ▼
[ViewResolver / MessageConverter]
    │
    ▼
[Interceptor 2] afterCompletion() ── 역순, 예외 무관
    │
    ▼
[Interceptor 1] afterCompletion()
    │
    ▼
[Filter 2] chain.doFilter() 이후 코드 실행
    │
    ▼
[Filter 1] chain.doFilter() 이후 코드 실행
    │
    ▼
HTTP 응답
```

---

## 3. 공식 근거

|주장|근거|
|---|---|
|Filter는 Servlet Container 소속|Jakarta Servlet Specification 6.0, §6.2|
|FilterChain은 ApplicationFilterChain으로 구현|Tomcat 10.x 소스, `org.apache.catalina.core.ApplicationFilterChain`|
|Interceptor는 DispatcherServlet 내부에서 실행|Spring Framework 6.x Reference — Web MVC / Interception|
|preHandle 실패 시 역순 afterCompletion 보장|`HandlerExecutionChain.applyPreHandle()` 소스|
|Spring Boot의 Filter 등록 메커니즘|`FilterRegistrationBean` → `ServletContextInitializer`|

---

## 4. 이 설계를 이렇게 한 이유

### 질문자의 직관이 맞다, 그런데 더 정확히는:

> "다형성 체제 유지"보다는 **"컨테이너 경계를 존중한 책임 분리"**가 더 정확한 표현이다.

**설계 의도:**

Servlet Specification은 스프링이 생기기 전부터 존재했다.  
스프링은 `DispatcherServlet` 하나를 Tomcat에 등록하는 방식으로 동작한다.  
즉, 스프링 입장에서 Tomcat은 **외부 인프라**다.

| |Filter|Interceptor|
|---|---|---|
|소속|Servlet Container (Tomcat)|Spring ApplicationContext|
|스프링 빈 주입|불가 (기본)|가능|
|Handler 정보|불가|가능 (HandlerMethod)|
|비 HTTP 요청|처리 가능|HTTP만|
|적용 범위|모든 서블릿|DispatcherServlet만|
|예외 처리|직접 처리해야 함|`@ControllerAdvice`가 처리|

**트레이드오프 — 무엇을 잃는가:**

**Filter를 인터셉터 역할까지 쓰려 할 때 잃는 것:**

- `@Transactional` 등 스프링 AOP가 적용된 빈을 자연스럽게 사용할 수 없다
- `HandlerMethod`의 어노테이션 정보를 읽으려면 URL 파싱을 직접 구현해야 한다
- 예외 발생 시 `@ControllerAdvice`로 위임할 수 없다 — 직접 JSON 에러 응답을 써야 한다

**Interceptor를 필터 역할까지 쓰려 할 때 잃는 것:**

- 문자 인코딩, GZIP 압축 해제 등 서블릿 레벨 처리를 할 수 없다
- Spring MVC를 거치지 않는 요청(정적 리소스 서블릿, 다른 서블릿)에 적용되지 않는다
- 멀티파트 파싱 이전 시점에 raw 요청을 조작할 수 없다

**실무 판단 기준:**

```
스프링 빈이 필요한가?           → 인터셉터
HandlerMethod 정보가 필요한가?  → 인터셉터
서블릿 전체에 적용해야 하는가?  → 필터
요청/응답 바이트를 직접 조작하는가? → 필터
스프링 MVC 이전에 막아야 하는가? → 필터 (예: IP 차단, Rate Limit)
인증/인가 (스프링 시큐리티)?    → 필터 체인 (SecurityFilterChain)
```

---

## 5. 이어지는 개념

|순서|개념|이유|
|---|---|---|
|1|**Spring Security FilterChain**|필터 구조 위에 인증/인가 체계 전체가 올라가 있다. 필터 동작을 모르면 Security 설정을 블랙박스로만 다루게 된다|
|2|**DispatcherServlet 전체 요청 처리 흐름**|인터셉터가 언제 개입하는지 정확히 알려면 `doDispatch()` 전체 흐름이 선행되어야 한다|
|3|**HandlerMethodArgumentResolver / ReturnValueHandler**|인터셉터 이후 컨트롤러 파라미터 바인딩이 어떻게 되는지 — 실무에서 커스텀 어노테이션 파라미터를 만들 때 바로 필요|
|4|**OncePerRequestFilter**|실무에서 필터를 직접 구현할 때 거의 항상 이것을 쓴다. 포워드/인클루드 시 중복 실행 방지 문제를 왜 신경 써야 하는지 이해해야 한다|
|5|**@ControllerAdvice / ExceptionHandlerExceptionResolver**|필터에서 예외가 발생하면 왜 `@ControllerAdvice`가 잡지 못하는지 — 인터셉터와 필터의 예외 처리 차이가 구체화된다|