#SpringBoot/Spring
```
[TCP 연결]
Connector가 포트 Listen
→ 3-Way Handshake 수립
→ Worker 스레드 할당 (ThreadPool)

[Tomcat — 서블릿 컨테이너 레벨]
HTTP 요청 파싱
→ HttpServletRequest / HttpServletResponse 객체 생성
→ Filter Chain 진입

[Filter Chain — 서블릿 컨테이너 레벨]
Filter1 → Filter2 → Filter3 → ...
(Spring Security 등)
chain.doFilter() 호출로 다음으로 넘김

[DispatcherServlet — Spring MVC 레벨]
doDispatch() 진입
→ HandlerMapping: URL+Method로 HandlerMethod 조회 (내부 캐싱된 Map)
→ HandlerAdapter 선택: HandlerMethod 타입에 맞는 어댑터
→ Interceptor.preHandle()
→ ArgumentResolver: 파라미터 바인딩
→ HandlerMethod 실행 (리플렉션: method.invoke(bean, args))
→ Interceptor.postHandle()
→ ReturnValueHandler + MessageConverter: 반환값 → HTTP 응답 바디
→ Interceptor.afterCompletion()

[HTTP 응답]
```
## 1. Problem First

### 서블릿이 없던 시절 — 직접 소켓을 다뤄야 했다

```java
// 서블릿 이전, 순수 Java로 HTTP 서버를 만든다면
ServerSocket serverSocket = new ServerSocket(8080);
Socket clientSocket = serverSocket.accept();

// HTTP 요청을 직접 파싱해야 함
BufferedReader in = new BufferedReader(
    new InputStreamReader(clientSocket.getInputStream()));

String requestLine = in.readLine(); // "GET /users HTTP/1.1"
// 헤더 파싱, 바디 파싱, 쿼리스트링 파싱...
// 모두 직접 구현

// HTTP 응답도 직접 만들어야 함
PrintWriter out = new PrintWriter(clientSocket.getOutputStream());
out.println("HTTP/1.1 200 OK");
out.println("Content-Type: application/json");
out.println();
out.println("{\"users\": [...]}");
```

모든 컨트롤러마다 이 파싱 로직이 반복된다. 스레드 관리, 커넥션 관리도 직접 해야 한다.

이 반복을 없애기 위해 **서블릿 스펙**이 나왔고, 서블릿 컨테이너(Tomcat)가 이 저수준 작업을 대신 처리한다. Spring은 그 위에 올라탄다.

---

## 2. Mechanics — 전체 흐름

요청이 들어와서 응답이 나가기까지 레이어를 순서대로 내려간다.

```
[클라이언트 HTTP 요청]
         │
         ▼
  ① Tomcat (서블릿 컨테이너)
         │
         ▼
  ② Filter Chain
         │
         ▼
  ③ DispatcherServlet
         │
         ▼
  ④ HandlerMapping
         │
         ▼
  ⑤ HandlerAdapter
         │
    ┌────┴────┐
    ▼         ▼
  ⑥ Interceptor  →  ⑦ ArgumentResolver
                          │
                          ▼
                    ⑧ Controller 메서드
                          │
                          ▼
                    ⑨ ReturnValueHandler
                          │
                          ▼
                   ⑩ MessageConverter
         │
         ▼
  [클라이언트 HTTP 응답]
```

각 레이어를 내부 동작까지 파고든다.

---

### ① Tomcat — 소켓부터 서블릿까지

클라이언트의 TCP 연결 요청을 Tomcat이 받는다.

```
Tomcat 내부:

Connector (포트 8080 Listen)
    │  TCP 연결 수립 (3-Way Handshake)
    ▼
Acceptor Thread
    │  소켓을 Worker Thread에 넘김
    ▼
Worker Thread (ThreadPool에서 꺼냄)
    │  HTTP 요청 파싱
    │  "GET /users HTTP/1.1\r\nHost: ..."
    │         ↓
    │  HttpServletRequest 객체 생성
    │  HttpServletResponse 객체 생성
    ▼
Servlet으로 위임
```

Tomcat의 기본 스레드 풀 크기는 `max-threads=200`이다. 동시에 200개의 요청을 처리할 수 있고, 초과하면 큐에서 대기한다.

```yaml
# application.yml
server:
  tomcat:
    threads:
      max: 200        # 최대 Worker Thread 수
      min-spare: 10   # 최소 유지 Thread 수
    accept-count: 100 # 큐 대기 최대 수
```

Spring Boot는 Tomcat을 내장(Embedded)하므로 별도 설치 없이 `jar` 실행만으로 동작한다. 내부적으로 `TomcatServletWebServerFactory`가 Tomcat을 프로그래밍 방식으로 구성한다.

---

### ② Filter Chain — 서블릿 진입 전 처리

Tomcat이 `HttpServletRequest`를 만들면, **서블릿 스펙(JSR-340)** 에 정의된 Filter Chain을 통과한다.

```java
// Filter 인터페이스 (서블릿 스펙)
public interface Filter {
    void doFilter(ServletRequest request,
                  ServletResponse response,
                  FilterChain chain) throws IOException, ServletException;
}
```

```
요청 →  [Filter1] → [Filter2] → [Filter3] → DispatcherServlet
응답 ←  [Filter1] ← [Filter2] ← [Filter3] ←
```

`chain.doFilter()`를 호출하면 다음 Filter로, 호출하지 않으면 요청이 거기서 막힌다.

**Spring Security가 Filter 레이어에 위치하는 이유:**

Filter는 DispatcherServlet **바깥**이다. Spring의 컨텍스트(Bean 등)가 아직 개입하기 전이다. 인증/인가를 여기서 처리하면 인증되지 않은 요청이 Spring 내부로 아예 진입하지 못한다.

```java
// Spring Security의 핵심 Filter
public class DelegatingFilterProxy implements Filter {
    // Spring Security의 SecurityFilterChain을 감싸는 브릿지
    // 서블릿 Filter지만 Spring Bean을 사용할 수 있게 연결
}

// 내부적으로 이런 순서로 동작
UsernamePasswordAuthenticationFilter
    → JwtAuthenticationFilter  // 커스텀 필터
    → ExceptionTranslationFilter
    → FilterSecurityInterceptor
```

---

### ③ DispatcherServlet — Spring MVC의 핵심

Filter를 통과한 요청은 **DispatcherServlet** 에 도달한다. Spring MVC의 **Front Controller 패턴** 구현체다.

> Spring 공식 레퍼런스: _"Spring MVC, like many other web frameworks, is designed around the front controller pattern where a central Servlet, the DispatcherServlet, provides a shared algorithm for request processing."_ 
> (Spring MVC는 다른 많은 웹 프레임워크처럼, 중앙 서블릿인 DispatcherServlet이 요청 처리를 위한 공유 알고리즘을 제공하는 Front Controller 패턴을 중심으로 설계되었다.)

모든 요청이 DispatcherServlet 하나로 집중되고, DispatcherServlet이 적절한 컨트롤러로 위임한다.

```java
// DispatcherServlet.doDispatch() 핵심 흐름 (소스코드 기반)
protected void doDispatch(HttpServletRequest request,
                          HttpServletResponse response) throws Exception {

    // 1. HandlerMapping으로 핸들러 조회
    HandlerExecutionChain mappedHandler = getHandler(request);

    // 2. HandlerAdapter 조회
    HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

    // 3. Interceptor의 preHandle 실행
    mappedHandler.applyPreHandle(request, response);

    // 4. 실제 핸들러(컨트롤러) 실행
    ModelAndView mv = ha.handle(request, response, mappedHandler.getHandler());

    // 5. Interceptor의 postHandle 실행
    mappedHandler.applyPostHandle(request, response, mv);

    // 6. 응답 처리 (ViewResolver 또는 MessageConverter)
    processDispatchResult(request, response, mappedHandler, mv, ...);
}
```

---

### ④ HandlerMapping — "이 요청은 어느 컨트롤러가 처리하는가"

DispatcherServlet이 가장 먼저 하는 일은 **요청 URL에 맞는 핸들러를 찾는 것**이다.

```java
// Spring Boot 기동 시 HandlerMapping이 하는 일
// @RequestMapping이 붙은 모든 메서드를 스캔해서 Map으로 만들어둠

Map<RequestMappingInfo, HandlerMethod> registry = {
    "GET /users"         → UserController.getUsers(),
    "POST /users"        → UserController.createUser(),
    "GET /users/{id}"    → UserController.getUser(Long id),
    "DELETE /users/{id}" → UserController.deleteUser(Long id),
    ...
}
```

요청이 오면 이 Map에서 조회한다. `RequestMappingHandlerMapping`이 기본 구현체다.

```
GET /users/42 HTTP/1.1

HandlerMapping 조회:
"GET /users/{id}" 패턴 매칭 → UserController.getUser()
PathVariable: id = 42
```

---

### ⑤ HandlerAdapter — 다양한 핸들러 타입을 통일된 방식으로 실행

HandlerMapping이 핸들러를 찾았어도, DispatcherServlet이 핸들러를 직접 호출하지 않는다. **HandlerAdapter를 통해 간접 호출**한다.

이유: Spring은 `@Controller` 외에도 `HttpRequestHandler`, `SimpleControllerHandlerAdapter` 등 다양한 핸들러 타입을 지원한다. DispatcherServlet은 어떤 타입이든 `handle()` 메서드 하나로 동일하게 처리한다.

```java
// HandlerAdapter 인터페이스
public interface HandlerAdapter {
    boolean supports(Object handler);  // 이 핸들러를 처리할 수 있는가
    ModelAndView handle(...);          // 실제 실행
}

// @RequestMapping 메서드용 구현체
RequestMappingHandlerAdapter.handle()
    │
    ├─ ArgumentResolver로 파라미터 바인딩
    ├─ 컨트롤러 메서드 리플렉션으로 호출
    └─ ReturnValueHandler로 반환값 처리
```

---

### ⑥ Interceptor — Spring 컨텍스트 안에서의 전/후 처리

Filter가 서블릿 컨테이너 레벨이라면, Interceptor는 **Spring MVC 레벨**이다.

```java
public interface HandlerInterceptor {
    // 컨트롤러 실행 전
    boolean preHandle(HttpServletRequest request,
                      HttpServletResponse response,
                      Object handler);

    // 컨트롤러 실행 후, 응답 전송 전
    void postHandle(HttpServletRequest request,
                    HttpServletResponse response,
                    Object handler,
                    ModelAndView modelAndView);

    // 응답 완료 후
    void afterCompletion(HttpServletRequest request,
                         HttpServletResponse response,
                         Object handler,
                         Exception ex);
}
```

```
Filter vs Interceptor 비교:

                Filter              Interceptor
레이어          서블릿 컨테이너      Spring MVC
Spring Bean     직접 사용 어려움    자유롭게 사용 가능
실행 시점       DispatcherServlet 전  컨트롤러 전/후
접근 가능 정보  Request/Response만   Handler 정보도 접근 가능
주요 용도       인증, 인코딩, CORS   로깅, 권한 세부 체크, 실행시간 측정
```

---

### ⑦ ArgumentResolver — 메서드 파라미터를 어떻게 만드는가

컨트롤러 메서드의 파라미터를 자동으로 바인딩하는 것이 `HandlerMethodArgumentResolver`다.

```java
@GetMapping("/users/{id}")
public UserResponse getUser(
    @PathVariable Long id,               // PathVariableMethodArgumentResolver
    @RequestParam String name,           // RequestParamMethodArgumentResolver
    @RequestBody CreateUserRequest body, // RequestResponseBodyMethodProcessor
    @AuthenticationPrincipal User user,  // AuthenticationPrincipalArgumentResolver
    HttpServletRequest request           // ServletRequestMethodArgumentResolver
) { ... }
```

각 파라미터 타입/애너테이션마다 전담 Resolver가 있다.

```java
// RequestResponseBodyMethodProcessor가 @RequestBody를 처리하는 방식
public Object resolveArgument(...) {
    // 1. Content-Type 헤더 확인: "application/json"
    // 2. 적절한 MessageConverter 선택: MappingJackson2HttpMessageConverter
    // 3. Request Body를 읽어서 Java 객체로 역직렬화
    return messageConverter.read(CreateUserRequest.class, request);
}
```

---

### ⑧ Controller 메서드 실행

드디어 개발자가 작성한 코드가 실행된다.

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @GetMapping("/{id}")
    public UserResponse getUser(@PathVariable Long id) {
        // 여기까지 오는 데
        // TCP → Tomcat → Filter → DispatcherServlet
        // → HandlerMapping → HandlerAdapter
        // → Interceptor.preHandle → ArgumentResolver
        // 의 과정이 있었다
        return userService.findById(id);
    }
}
```

---

### ⑨⑩ ReturnValueHandler + MessageConverter — 반환값을 HTTP 응답으로

컨트롤러가 값을 반환하면 `HandlerMethodReturnValueHandler`가 처리한다.

`@RestController`는 `@Controller + @ResponseBody`다. `@ResponseBody`가 있으면 `RequestResponseBodyMethodProcessor`가 반환값을 처리한다.

```java
// MessageConverter가 Java 객체 → HTTP 응답 바디로 변환
public void handleReturnValue(Object returnValue, ...) {
    // 1. 클라이언트의 Accept 헤더 확인: "application/json"
    // 2. 반환 타입 확인: UserResponse.class
    // 3. 처리 가능한 MessageConverter 선택
    //    MappingJackson2HttpMessageConverter → JSON 변환 가능
    // 4. 직렬화
    messageConverter.write(returnValue, MediaType.APPLICATION_JSON, response);
}
```

Content Negotiation — 클라이언트가 원하는 형식으로:

```
클라이언트: Accept: application/json  → JSON으로 응답
클라이언트: Accept: application/xml   → XML로 응답 (converter 있으면)
클라이언트: Accept: */*              → 서버 기본값(보통 JSON)으로 응답
```

---

### 전체 흐름 — 예외 발생 시

```java
@GetMapping("/users/{id}")
public UserResponse getUser(@PathVariable Long id) {
    throw new UserNotFoundException(id);  // 예외 발생
}
```

```
예외 발생
    │
    ▼
HandlerExceptionResolver 목록 순회
    │
    ├─ @ExceptionHandler 있는가?
    │      ↓ 있으면
    │  ExceptionHandlerExceptionResolver 처리
    │  (@ControllerAdvice의 @ExceptionHandler 메서드 실행)
    │
    └─ 없으면 → ResponseStatusExceptionResolver
               → DefaultHandlerExceptionResolver
               → 최종적으로 Tomcat의 기본 에러 페이지
```

```java
// @ControllerAdvice는 DispatcherServlet 레벨에서 동작
// Filter에서 던진 예외는 여기서 잡히지 않음 — 중요한 함정
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound(
            UserNotFoundException e) {
        return ResponseEntity.status(404)
            .body(new ErrorResponse(e.getMessage()));
    }
}
```

---

## 3. 공식 근거

|주장|출처|
|---|---|
|DispatcherServlet Front Controller 패턴|Spring 공식 레퍼런스 — _Web on Servlet Stack, 1.1_|
|Filter 인터페이스 정의|Jakarta Servlet Specification 6.0 (구 JSR-340), Section 6|
|HandlerInterceptor 정의|Spring Framework 공식 Javadoc|
|HandlerMethodArgumentResolver|Spring 공식 레퍼런스 — _Annotated Controllers, Method Arguments_|
|MessageConverter (Content Negotiation)|Spring 공식 레퍼런스 — _HTTP Message Conversion_|
|Tomcat 스레드 풀 설정|Apache Tomcat 공식 문서 — _Connector Attributes_|

---

## 4. 이 설계를 이렇게 한 이유

### Front Controller 패턴이 얻는 것

- 모든 요청이 DispatcherServlet을 거치므로 공통 처리(로깅, 인증, 직렬화)를 한 곳에서
- 컨트롤러는 HTTP 파싱/직렬화를 모르고 비즈니스 로직에만 집중

### 레이어별 트레이드오프

|레이어|얻는 것|잃는 것|
|---|---|---|
|Filter|Spring 독립적, 모든 요청 처리|Spring Bean 사용 어려움, 예외가 @ControllerAdvice로 안 잡힘|
|Interceptor|Spring Bean 자유롭게 사용, Handler 정보 접근|DispatcherServlet 통과 후에만 동작, 정적 리소스 요청엔 동작 안 할 수 있음|
|@ControllerAdvice|전역 예외 처리 통일|Filter/Interceptor에서 던진 예외는 처리 못 함|

---

## 5. 이어지는 개념

|순서|개념|이유|
|---|---|---|
|**1**|**Spring Security Filter Chain 상세**|인증/인가가 Filter 레이어에서 어떻게 동작하는지. `SecurityFilterChain`, `OncePerRequestFilter` 가 이 흐름의 구체적 구현|
|**2**|**Spring AOP와 트랜잭션**|컨트롤러 → 서비스 호출 시 `@Transactional`이 어떻게 끼어드는가. 프록시 패턴으로 동작하며 Filter/Interceptor와 다른 레이어|
|**3**|**비동기 처리 — WebFlux와 Virtual Thread**|Tomcat 스레드 풀 모델의 한계(blocking I/O)를 어떻게 극복하는가. Spring MVC의 구조적 한계와 대안|

# 서블릿 컨테이너 레벨 vs Spring MVC 레벨
### 서블릿 컨테이너 레벨

Tomcat이 관리하는 영역이다. Spring이 개입하기 전이다.

```
Tomcat이 아는 것:
- 소켓, 스레드, HTTP 파싱
- 서블릿 스펙 (javax.servlet / jakarta.servlet)
- Filter, Listener

Tomcat이 모르는 것:
- Spring Bean
- @Controller, @Service
- ApplicationContext
```

Filter가 이 레벨에 있다는 것의 실질적 의미:
```java
// Filter에서 Spring Bean을 쓰려면 직접 꺼내야 했음 (과거)
WebApplicationContext ctx = WebApplicationContextUtils
    .getWebApplicationContext(filterConfig.getServletContext());
UserService userService = ctx.getBean(UserService.class);

// Spring Boot는 DelegatingFilterProxy로 이 문제를 해결
// Filter를 Spring Bean으로 등록 가능하게 브릿지 역할
```

**Filter에서 던진 예외가 @ControllerAdvice로 안 잡히는 이유:**

```
@ControllerAdvice는 DispatcherServlet 내부에서 동작
Filter는 DispatcherServlet 바깥

Filter에서 예외 발생
    │
    │ DispatcherServlet에 도달 못 함
    ▼
Tomcat의 기본 에러 처리
    │
    ▼
/error 엔드포인트로 포워딩 (Spring Boot의 BasicErrorController)
```

Spring Security Filter에서 인증 실패 시 응답 형식이 의도한 대로 안 나오는 문제가 여기서 비롯된다. 해결하려면 Filter 안에서 직접 response에 써야 한다.

```java
// Security Filter에서 직접 응답을 써야 하는 이유
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    protected void doFilterInternal(...) {
        try {
            // JWT 검증
        } catch (JwtException e) {
            // @ControllerAdvice로 못 보내므로 여기서 직접 처리
            response.setStatus(401);
            response.setContentType("application/json");
            response.getWriter().write("{\"error\": \"Unauthorized\"}");
            return;
        }
        filterChain.doFilter(request, response);
    }
}
```

---

### Spring MVC 레벨

DispatcherServlet부터 그 안의 컴포넌트들이 이 레벨이다. Spring ApplicationContext가 관리하는 영역이다.

```
Spring MVC 레벨이 아는 것:
- Spring Bean (ApplicationContext)
- @Controller, @Service, @Repository
- HandlerMapping, HandlerAdapter
- Interceptor, ArgumentResolver, MessageConverter
- @ControllerAdvice, @ExceptionHandler
```

Interceptor가 이 레벨에 있다는 것의 실질적 의미:

```java
// Interceptor는 Spring Bean을 자유롭게 사용 가능
@Component
public class AuthInterceptor implements HandlerInterceptor {

    private final JwtProvider jwtProvider; // Spring Bean 주입 가능

    public boolean preHandle(HttpServletRequest request, ...,
                             Object handler) {
        // handler가 HandlerMethod임을 알 수 있음
        HandlerMethod handlerMethod = (HandlerMethod) handler;

        // @LoginRequired 애너테이션 유무 확인 가능
        if (handlerMethod.hasMethodAnnotation(LoginRequired.class)) {
            // 인증 체크
        }
        return true;
    }
}
```

Filter는 `handler` 정보에 접근 불가능하다. 어떤 컨트롤러 메서드가 실행될지 모른다. Interceptor는 `HandlerMethod`가 무엇인지 알기 때문에 메서드 단위 세밀한 제어가 가능하다.

---

## 두 레벨 비교 — 한 번에

```
                서블릿 컨테이너 레벨      Spring MVC 레벨
관리 주체        Tomcat                  Spring
진입 시점        DispatcherServlet 전     DispatcherServlet 내부
Spring Bean     직접 사용 어려움          자유롭게 사용 가능
Handler 정보    접근 불가                 접근 가능 (HandlerMethod)
예외 처리        @ControllerAdvice 불가   @ControllerAdvice 가능
대표 컴포넌트    Filter                   Interceptor, ArgumentResolver
주요 용도        인증, 인코딩, CORS        로깅, 세밀한 권한, 실행시간 측정
```

---

## 판단 기준 — 어디에 구현할 것인가

```
이 로직이 Spring Bean이 필요한가?
    YES → Interceptor (또는 AOP)
    NO  → Filter도 가능

이 로직이 모든 요청에 적용돼야 하는가? (정적 리소스 포함)
    YES → Filter
    NO  → Interceptor

예외를 @ControllerAdvice로 통일 처리하고 싶은가?
    YES → Interceptor (또는 컨트롤러 내부)
    NO  → Filter도 가능

컨트롤러 메서드의 애너테이션을 읽어야 하는가?
    YES → Interceptor (HandlerMethod 접근 가능)
    NO  → Filter도 가능
```