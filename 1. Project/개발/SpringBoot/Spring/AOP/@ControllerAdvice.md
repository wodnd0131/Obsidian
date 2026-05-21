# @ControllerAdvice의 적용 범위와 예외 처리의 경계선

## 1. Problem First — 일관된 에러 응답을 만들려는 시도가 왜 실패하는가

wddy가 REST API를 만들면서 가장 먼저 부딪히는 요구사항 중 하나: **"모든 에러 응답을 일관된 포맷으로 통일하자"**.

표준 에러 포맷을 정한다:

```json
{
  "code": "USER_NOT_FOUND",
  "message": "사용자를 찾을 수 없습니다",
  "timestamp": "2026-05-21T10:30:00Z",
  "path": "/api/users/123"
}
```

`@RestControllerAdvice`로 글로벌 예외 처리기를 만든다:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(UserNotFoundException e) {
        return ResponseEntity.status(404).body(new ErrorResponse(
            "USER_NOT_FOUND", e.getMessage(), Instant.now(), null
        ));
    }

    @ExceptionHandler(ValidationException.class)
    public ResponseEntity<ErrorResponse> handleValidation(ValidationException e) {
        return ResponseEntity.status(400).body(new ErrorResponse(
            "VALIDATION_FAILED", e.getMessage(), Instant.now(), null
        ));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleAll(Exception e) {
        return ResponseEntity.status(500).body(new ErrorResponse(
            "INTERNAL_ERROR", "서버 오류", Instant.now(), null
        ));
    }
}
```

"이제 모든 에러가 일관된 포맷이겠지" 하고 안심한다. 그런데 실제로 운영하다 보면:

**시나리오 1**: 정상 동작

```
GET /api/users/999  → UserService에서 UserNotFoundException 던짐
응답: { "code": "USER_NOT_FOUND", ... }  ✓ 잘 잡힘
```

**시나리오 2**: 인증 토큰이 잘못된 요청

```
GET /api/users/123  (잘못된 JWT)
→ JWT Filter에서 인증 실패
응답: HTML 페이지 또는 Spring Security 기본 응답  ✗ 포맷이 다름
```

**시나리오 3**: 존재하지 않는 경로

```
GET /api/nonexistent  
→ DispatcherServlet이 매핑 못 찾음
응답: HTML 에러 페이지 또는 Spring Boot 기본 JSON  ✗ 또 포맷이 다름
```

**시나리오 4**: Controller의 Interceptor에서 예외

```
GET /api/users/123  (Interceptor의 preHandle에서 RuntimeException)
응답: { "code": "INTERNAL_ERROR", ... }  ✓ 잡힘
```

**시나리오 5**: 본문 파싱 실패

```
POST /api/users  Content-Type: application/json
바디: { 잘못된 JSON }
→ @RequestBody 파싱 실패 (HttpMessageNotReadableException)
응답: 잡히긴 하는데 처리가 미묘함
```

**근본 문제**: `@ControllerAdvice`는 만능이 아니다. **DispatcherServlet 안에서 발생한 예외만 잡는다**. 그 밖에서 발생한 예외는 다른 메커니즘이 처리한다.

이 경계선을 정확히 모르면, "어떤 에러는 잡히고 어떤 에러는 안 잡히는데 왜 그러지?"의 미궁에 빠진다. 그리고 운영 중인 API에서 일부 에러만 HTML로 나오는 식의 일관성 깨짐이 발생한다.

---

## 2. Mechanics — 예외가 어디서 어떻게 흐르는가

### 전체 그림: 예외 처리의 세 영역

```
[클라이언트 요청]
    ↓
┌────────────────────────────────────────────────────────┐
│ Servlet Container (Tomcat)                             │
│                                                        │
│  영역 ③: Tomcat이 처리                                  │
│   ┌────────────────────────────────────────────┐       │
│   │ 여기서 던져진 예외는 Tomcat의              │       │
│   │ 에러 페이지 메커니즘이 처리                 │       │
│   │ → Spring Boot의 BasicErrorController로 포워딩 │       │
│   └────────────────────────────────────────────┘       │
│                                                        │
│  영역 ②: Servlet Filter 체인                           │
│   ┌────────────────────────────────────────────┐       │
│   │ JWT Filter, Security Filter 등              │       │
│   │ 여기서 예외 던지면 @ControllerAdvice 못 잡음 │       │
│   └────────────────────────────────────────────┘       │
│                ↓                                       │
│  영역 ①: DispatcherServlet 내부                        │
│   ┌────────────────────────────────────────────┐       │
│   │ Interceptor, Controller, ArgumentResolver   │       │
│   │ 여기서 던진 예외만 @ControllerAdvice가 잡음  │       │
│   │                                            │       │
│   │ 처리 메커니즘: HandlerExceptionResolver     │       │
│   └────────────────────────────────────────────┘       │
└────────────────────────────────────────────────────────┘
```

핵심 분기점: **예외가 DispatcherServlet 안에서 발생했는가, 밖에서 발생했는가**.

### 영역 ① — DispatcherServlet 내부에서의 예외 처리

이 영역에서의 예외 처리는 `HandlerExceptionResolver`가 담당한다. DispatcherServlet의 `doDispatch()` 메서드를 단순화하면:

```java
// DispatcherServlet.doDispatch() 단순화
protected void doDispatch(HttpServletRequest req, HttpServletResponse res) throws Exception {
    Exception dispatchException = null;
    ModelAndView mv = null;
    
    try {
        // 1. HandlerMapping으로 Handler 찾기
        HandlerExecutionChain mappedHandler = getHandler(req);
        
        // 2. Interceptor preHandle
        if (!mappedHandler.applyPreHandle(req, res)) return;
        
        // 3. Handler 실행 (Controller 메서드)
        HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
        mv = ha.handle(req, res, mappedHandler.getHandler());
        
        // 4. Interceptor postHandle
        mappedHandler.applyPostHandle(req, res, mv);
    }
    catch (Exception ex) {
        dispatchException = ex;  // ★ 예외를 모은다
    }
    
    // 5. 결과 처리 (예외 처리 포함)
    processDispatchResult(req, res, mappedHandler, mv, dispatchException);
}

private void processDispatchResult(..., Exception exception) throws Exception {
    if (exception != null) {
        // ★ HandlerExceptionResolver 체인을 돌린다
        mv = processHandlerException(req, res, handler, exception);
    }
    // 뷰 렌더링 등...
}

private ModelAndView processHandlerException(..., Exception ex) throws Exception {
    for (HandlerExceptionResolver resolver : this.handlerExceptionResolvers) {
        ModelAndView exMv = resolver.resolveException(req, res, handler, ex);
        if (exMv != null) {
            return exMv;  // 한 resolver가 처리하면 끝
        }
    }
    throw ex;  // 아무도 처리 못하면 다시 던짐 → 영역 ③으로
}
```

**핵심**: `try-catch`가 dispatch 전체를 감싸고 있고, 잡힌 예외를 `HandlerExceptionResolver`에 위임한다. Spring Boot의 기본 resolver 체인:

1. **`ExceptionHandlerExceptionResolver`** — `@ExceptionHandler` 메서드를 찾아 실행. 이게 `@ControllerAdvice`를 동작시키는 주체
2. **`ResponseStatusExceptionResolver`** — `@ResponseStatus`나 `ResponseStatusException` 처리
3. **`DefaultHandlerExceptionResolver`** — Spring MVC 표준 예외 처리 (`HttpRequestMethodNotSupportedException` 등)

이 셋 중 누구도 처리 못 하면 `processHandlerException`이 예외를 다시 던지고, 그게 DispatcherServlet 밖으로 빠져나간다 → **영역 ③으로 넘어감**.

### 왜 Interceptor 예외는 잡히는가

Interceptor는 DispatcherServlet **안에서** 실행된다 (앞서 다룬 구조 기억). Interceptor의 `preHandle()`이 예외를 던지면:

```java
// DispatcherServlet 안의 흐름
try {
    mappedHandler.applyPreHandle(req, res);  // ★ Interceptor.preHandle() 호출
    // 여기서 예외 → 아래 catch로 잡힘
}
catch (Exception ex) {
    dispatchException = ex;
}
processDispatchResult(req, res, ..., dispatchException);
// → ExceptionHandlerExceptionResolver가 @ControllerAdvice 찾아 실행
```

Interceptor 예외도 dispatch 전체의 try-catch에 잡히므로 자동으로 `@ControllerAdvice` 처리 대상이 된다.

### 왜 Filter 예외는 안 잡히는가

Filter는 DispatcherServlet **밖에서** 실행된다. Filter가 예외를 던지면:

```java
// 흐름 (단순화)
Tomcat → Filter1.doFilter() 
       → Filter2.doFilter()  (★ 여기서 예외 던짐)
       ✗ DispatcherServlet에 도달조차 못함
       
       예외가 Filter 체인을 거꾸로 빠져나옴
       → Tomcat이 받음
       → 영역 ③의 ErrorController로 포워딩
```

DispatcherServlet의 `try-catch`는 자기 안의 코드만 감싼다. Filter는 DispatcherServlet의 호출자(Tomcat)가 실행하므로 그 catch가 영향을 못 미친다. **공간적으로 도달 불가능**하다.

### 영역 ③ — Servlet Container의 에러 메커니즘과 ErrorController

DispatcherServlet 밖으로 예외가 빠져나오거나, 서블릿 응답이 4xx/5xx 상태 코드로 끝나면, Servlet 컨테이너의 **에러 페이지 메커니즘**이 발동한다.

Servlet 스펙(JSR-340) §10.9.2는 에러 페이지를 정의한다:

- `web.xml`이나 코드로 "이 상태 코드 또는 이 예외 타입일 때 이 경로로 포워딩하라"고 등록
- 컨테이너가 응답 상태 코드를 확인하거나 예외를 잡으면, 해당 경로로 **포워딩**

Spring Boot는 자동으로 이걸 등록한다. `ErrorPageRegistrar`가 모든 에러를 `/error` 경로로 포워딩하도록 설정. 그리고 `/error` 경로를 처리하는 컨트롤러가 `BasicErrorController`다.

```java
// Spring Boot의 BasicErrorController (단순화)
@Controller
@RequestMapping("${server.error.path:/error}")
public class BasicErrorController {

    @RequestMapping(produces = MediaType.TEXT_HTML_VALUE)
    public ModelAndView errorHtml(HttpServletRequest req, HttpServletResponse res) {
        // 브라우저용 HTML 에러 페이지 (Whitelabel Error Page)
        HttpStatus status = getStatus(req);
        Map<String, Object> model = getErrorAttributes(req, ...);
        return new ModelAndView("error", model);
    }

    @RequestMapping
    public ResponseEntity<Map<String, Object>> error(HttpServletRequest req) {
        // JSON 응답 (Accept: application/json)
        HttpStatus status = getStatus(req);
        Map<String, Object> body = getErrorAttributes(req, ...);
        return new ResponseEntity<>(body, status);
    }
}
```

**핵심 통찰**: `BasicErrorController`는 **그냥 평범한 Controller**다. `@Controller` 어노테이션이 붙어있고 `/error` 경로에 매핑되어 있다.

이게 의미하는 바:

1. Filter에서 예외 발생
2. Tomcat이 예외 잡음 → `/error`로 포워딩
3. **이제 다시 DispatcherServlet으로 들어옴** (포워딩이니까)
4. `/error`를 처리하는 `BasicErrorController`가 실행됨
5. `BasicErrorController`가 응답을 만들어 보냄

여기서 미묘한 함정이 있다 — Filter 예외가 ErrorController를 통해 처리될 때, 응답 포맷은 `BasicErrorController`의 것이지 `@ControllerAdvice`의 것이 아니다.

### 영역들의 응답 포맷이 다른 이유

다시 시나리오들을 보면:

**시나리오 2: JWT Filter 실패**

- 예외 발생 위치: 영역 ② (Filter)
- 처리 경로: Tomcat → /error 포워딩 → BasicErrorController
- 응답 포맷: BasicErrorController의 기본 포맷 (Spring Boot 기본 JSON)
- **`@ControllerAdvice` 절대 안 탐**

**시나리오 3: 404 (매핑 없음)**

- 발생 위치: DispatcherServlet 안 (`NoHandlerFoundException`)
- 기본 설정으로는 DispatcherServlet이 예외를 던지지 않고 그냥 404 응답을 보냄
- → Tomcat이 404 상태 코드 감지 → /error 포워딩 → BasicErrorController
- 설정 `spring.mvc.throw-exception-if-no-handler-found=true`를 켜면 DispatcherServlet이 예외를 던지고, 그때 `@ControllerAdvice`가 `NoHandlerFoundException`을 잡을 수 있음

**시나리오 4: Interceptor 예외**

- 발생 위치: 영역 ① (DispatcherServlet 안)
- 처리 경로: `HandlerExceptionResolver` → `ExceptionHandlerExceptionResolver` → `@ControllerAdvice`
- 응답 포맷: 일관된 커스텀 포맷 ✓

**시나리오 5: @RequestBody 파싱 실패**

- 발생 위치: 영역 ① (`HandlerMethodArgumentResolver`가 본문 파싱 시도, `HttpMessageNotReadableException` 던짐)
- 이 예외는 `HandlerExceptionResolver` 체인에서 `DefaultHandlerExceptionResolver`가 우선 잡으려 함
- 하지만 `@ControllerAdvice`에 `@ExceptionHandler(HttpMessageNotReadableException.class)`를 정의하면 그게 먼저 매칭됨
- 잡으려면 `@ControllerAdvice`에서 명시적으로 처리해야 함

### Spring Security가 영역 ②에서 인증 실패를 처리하는 방식

Spring Security는 영역 ②(Filter)에서 동작하므로 `@ControllerAdvice`로 못 잡는다. 그래서 자체적인 예외 처리 메커니즘을 가진다:

```
JWT Filter에서 인증 실패
  ↓ 예외 던짐 (AuthenticationException)
ExceptionTranslationFilter가 잡음 ★
  ↓
AuthenticationEntryPoint.commence() 호출
  ↓ (사용자가 정의한 entry point)
응답 작성 (401 + JSON)
```

`ExceptionTranslationFilter`는 Spring Security가 자기 영역(Filter 체인)에서 발생한 예외를 **HTTP 응답으로 변환**하는 전용 Filter다. 이게 `@ControllerAdvice`의 Filter 영역 대응물이다.

따라서 일관된 응답 포맷을 만들려면:

- 영역 ①의 예외: `@RestControllerAdvice`
- 영역 ②의 Spring Security 예외: `AuthenticationEntryPoint` + `AccessDeniedHandler` 커스터마이즈
- 영역 ③의 처리: `BasicErrorController`를 자체 구현으로 교체 (또는 비활성화 후 `@ControllerAdvice`가 모든 걸 처리하도록)

### 통합 전략 — 모든 에러를 같은 포맷으로

실무에서 일관성을 만드는 패턴:

```java
// 1. 공통 응답 포맷 정의
public record ErrorResponse(String code, String message, Instant timestamp, String path) {}

// 2. @RestControllerAdvice — 영역 ① 커버
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusiness(BusinessException e, HttpServletRequest req) {
        return ResponseEntity.status(e.getStatus())
            .body(new ErrorResponse(e.getCode(), e.getMessage(), Instant.now(), req.getRequestURI()));
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException e, HttpServletRequest req) {
        return ResponseEntity.badRequest()
            .body(new ErrorResponse("VALIDATION_FAILED", e.getMessage(), Instant.now(), req.getRequestURI()));
    }
}

// 3. Spring Security — 영역 ② 커버
@Component
public class CustomAuthenticationEntryPoint implements AuthenticationEntryPoint {
    private final ObjectMapper objectMapper;
    
    @Override
    public void commence(HttpServletRequest req, HttpServletResponse res, AuthenticationException e) 
            throws IOException {
        res.setStatus(401);
        res.setContentType("application/json");
        ErrorResponse body = new ErrorResponse("AUTH_REQUIRED", e.getMessage(), Instant.now(), req.getRequestURI());
        objectMapper.writeValue(res.getWriter(), body);
    }
}

// 4. BasicErrorController 대체 — 영역 ③ 커버
@RestController
public class CustomErrorController implements ErrorController {
    @RequestMapping("/error")
    public ResponseEntity<ErrorResponse> handleError(HttpServletRequest req) {
        Integer status = (Integer) req.getAttribute(RequestDispatcher.ERROR_STATUS_CODE);
        String message = (String) req.getAttribute(RequestDispatcher.ERROR_MESSAGE);
        return ResponseEntity.status(status != null ? status : 500)
            .body(new ErrorResponse("ERROR_" + status, message, Instant.now(), req.getRequestURI()));
    }
}
```

이 세 가지가 다 있어야 비로소 어떤 경로로 에러가 발생해도 일관된 포맷이 나온다. **`@ControllerAdvice` 하나로 끝낼 수 있다는 환상이 가장 큰 함정**.

---

## 3. 공식 근거

**Spring Framework 공식 레퍼런스 (2순위) — HandlerExceptionResolver**:

> "HandlerExceptionResolver implementations deal with unexpected exceptions that occur during controller execution. A HandlerExceptionResolver somewhat resembles the exception mappings you can define in the web.xml file. However, it provides a more flexible way to do so. For example, it can provide status information about which handler was executing when the exception was thrown." (HandlerExceptionResolver 구현은 컨트롤러 실행 중에 발생하는 예기치 못한 예외를 다룬다. HandlerExceptionResolver는 web.xml 파일에서 정의할 수 있는 예외 매핑과 다소 유사하다. 그러나 더 유연한 방식을 제공한다. 예를 들어, 예외가 던져졌을 때 어떤 핸들러가 실행 중이었는지에 대한 상태 정보를 제공할 수 있다.) — Spring Framework Reference, "Web on Servlet Stack — Exceptions"

"during controller execution" — 컨트롤러 실행 중에 발생하는 예외만 다룬다는 점이 명시되어 있다. Filter는 컨트롤러 실행이 아니다.

**Servlet 스펙 (1순위, JSR-340 §10.9.2) — 에러 페이지**:

> "If the location of the error handler is a servlet or a JSP page: ... The servlet or JSP page MAY use the following request attributes to do its processing: jakarta.servlet.error.status_code, jakarta.servlet.error.exception_type, jakarta.servlet.error.message, jakarta.servlet.error.exception, jakarta.servlet.error.request_uri, jakarta.servlet.error.servlet_name." (에러 핸들러의 위치가 서블릿이거나 JSP 페이지인 경우: ... 그 서블릿 또는 JSP 페이지는 처리를 위해 다음 요청 속성들을 사용할 수 있다: status_code, exception_type, message, exception, request_uri, servlet_name.) — JSR-340 Servlet 3.1 Specification, §10.9.2 "Error Pages"

이 attribute들이 `BasicErrorController`가 읽어서 에러 정보를 구성하는 출처다.

**Spring Boot 공식 레퍼런스 (2순위) — Error Handling**:

> "Spring Boot provides an /error mapping by default that handles all errors in a sensible way, and it is registered as a 'global' error page in the servlet container. For machine clients, it produces a JSON response with details of the error, the HTTP status, and the exception message. For browser clients, there is a 'whitelabel' error view that renders the same data in HTML format. You can customize it by adding a View that resolves to 'error' or by adding a @ControllerAdvice." (Spring Boot는 기본적으로 모든 에러를 합리적인 방식으로 처리하는 /error 매핑을 제공하며, 서블릿 컨테이너에 '글로벌' 에러 페이지로 등록된다. 머신 클라이언트에게는 에러의 세부사항, HTTP 상태, 예외 메시지를 담은 JSON 응답을 생성한다. 브라우저 클라이언트에게는 같은 데이터를 HTML 형식으로 렌더링하는 'whitelabel' 에러 뷰가 있다. 'error'로 해석되는 View를 추가하거나 @ControllerAdvice를 추가하여 커스터마이즈할 수 있다.) — Spring Boot Reference, "Error Handling"

"registered as a 'global' error page in the servlet container" — Spring Boot가 Servlet 스펙의 에러 페이지 메커니즘을 사용한다는 것이 명시되어 있다.

**Spring Security 공식 레퍼런스 (2순위) — ExceptionTranslationFilter**:

> "ExceptionTranslationFilter allows translation of AccessDeniedException and AuthenticationException into HTTP responses. ExceptionTranslationFilter is inserted into the FilterChainProxy as one of the Security Filters." (ExceptionTranslationFilter는 AccessDeniedException과 AuthenticationException을 HTTP 응답으로 변환할 수 있게 한다. ExceptionTranslationFilter는 보안 필터 중 하나로 FilterChainProxy에 삽입된다.) — Spring Security Reference, "Servlet Architecture — ExceptionTranslationFilter"

**찾지 못한 부분**: "왜 DispatcherServlet의 try-catch가 Filter 영역까지 확장되지 않도록 설계되었는가"에 대한 명시적 설계 근거는 찾지 못했다. 하지만 이건 설계 결정이라기보다 **물리적 한계**다. DispatcherServlet은 Filter의 호출 대상(callee)이지 호출자(caller)가 아니므로, Filter에서 발생한 예외를 잡을 위치에 있지 않다.

---

## 4. 이 설계를 이렇게 한 이유

### 왜 `@ControllerAdvice`의 범위가 좁은가

근본적으로 `@ControllerAdvice`는 **"Controller를 위한 횡단 관심사"** 다. 이름에 그대로 적혀있다 — Controller Advice. AOP의 advice 개념을 차용한 것이고, 적용 대상은 컨트롤러다.

만약 Spring이 Filter 예외까지 잡으려고 했다면:

- Filter는 Spring 외부 표준(Servlet 스펙) 컴포넌트인데, Spring이 임의로 가로채는 게 부적절
- Spring Security처럼 Filter 기반 프레임워크가 자체 예외 처리를 가지는 게 일반적인데, 그것과 충돌
- Filter의 응답 작성은 매우 자유로워서(원시 OutputStream 조작 등), 일관된 가로채기가 기술적으로 어려움

따라서 `@ControllerAdvice`의 범위를 DispatcherServlet 안으로 제한한 건 **합리적인 경계 설정**이다.

### 왜 BasicErrorController가 별도로 있는가

Servlet 스펙은 에러 페이지를 정적 리소스(HTML 파일) 또는 Servlet으로 등록할 수 있게 한다. Spring Boot는 동적으로 에러를 처리해야 하므로 Controller로 만들었다. 이 Controller가 `BasicErrorController`다.

`@ControllerAdvice`만으로는 부족한 이유:

- 404처럼 Controller 매핑이 없는 경우 → `@ControllerAdvice`가 동작할 컨텍스트 자체가 없음
- Filter 예외처럼 DispatcherServlet 밖 예외 → 마찬가지
- 정적 리소스 요청 실패 → DispatcherServlet 안 갈 수도 있음

이런 경우들을 모두 `/error`로 모아서 단일 진입점을 만든 게 Spring Boot의 선택. 일관성을 만들 **재시도 기회**를 제공하는 것.

### 잃는 것

이 다층 구조의 비용:

- **이해해야 할 메커니즘이 많다**. 영역마다 다른 메커니즘 (`@ControllerAdvice`, `AuthenticationEntryPoint`, `ErrorController`)
- **응답 포맷 통일이 자동이 아니다**. 세 군데를 다 손봐야 일관됨
- **에러 응답 디버깅이 어렵다**. "이 에러가 어디서 처리됐지?"를 추적하려면 영역을 알아야 함
- **포워딩 비용**. Filter 예외 → /error 포워딩은 추가 디스패치를 발생시킴 (성능에 미미한 영향)

### 잘못된 선택의 예

**선택 1**: "그냥 Filter에서도 예외를 잡는 글로벌 핸들러를 만들자"

- 가능한 방법: 가장 바깥쪽에 `OncePerRequestFilter`를 두고 `try-catch`로 모두 잡기
- 문제: 그 Filter가 Spring Security Filter보다 바깥에 있어야 하는데, 그러면 Spring Security 안에서 던진 예외도 잡힘 → Spring Security의 자체 처리(`ExceptionTranslationFilter`)와 충돌
- 결과: 표준 메커니즘과 싸우게 되어 유지보수 지옥

**선택 2**: "BasicErrorController를 비활성화하고 모든 걸 `@ControllerAdvice`로"

- 일부 시나리오(매핑 없는 404)는 여전히 잡히지 않음
- `spring.mvc.throw-exception-if-no-handler-found=true` + 정적 리소스 핸들러 비활성화 등 여러 설정 필요
- 가능하지만 복잡

**현실적 선택**: 세 영역 모두 커스터마이즈하되, **공통 응답 빌더**를 추출해서 코드 중복을 줄임:

```java
@Component
public class ErrorResponseBuilder {
    public ErrorResponse build(String code, String message, HttpServletRequest req) {
        return new ErrorResponse(code, message, Instant.now(), req.getRequestURI());
    }
}

// @ControllerAdvice, AuthenticationEntryPoint, ErrorController에서 모두 이 빌더를 주입받아 사용
```

---

## wddy의 환경에 적용

wddy의 인증 구조(Access Token + Refresh Token + CSRF)에서 발생할 수 있는 에러들:

|에러 시나리오|발생 위치|처리 주체|
|---|---|---|
|잘못된 JWT (만료, 위조)|JWT Filter (영역 ②)|`AuthenticationEntryPoint`|
|Refresh Token 쿠키 없음|`/auth/refresh` Controller (영역 ①)|`@ControllerAdvice`|
|CSRF 토큰 불일치|`CsrfFilter` (영역 ②)|Spring Security의 `AccessDeniedHandler`|
|권한 부족 (`@PreAuthorize` 실패)|Method Security AOP → DispatcherServlet 안|Spring Security 또는 `@ControllerAdvice` (어디서 잡히는지는 미묘)|
|비즈니스 예외 (`UserNotFoundException`)|Service → Controller (영역 ①)|`@ControllerAdvice`|
|잘못된 JSON 본문|`@RequestBody` 파싱 (영역 ①)|`@ControllerAdvice`로 `HttpMessageNotReadableException` 처리|
|존재하지 않는 경로|DispatcherServlet → 매핑 없음|`BasicErrorController` 또는 설정으로 `@ControllerAdvice`|
|Tomcat 레벨 에러 (요청 본문 크기 초과 등)|Tomcat 자체|`BasicErrorController`|

**핵심**: wddy의 응답 포맷이 통일되려면 최소 세 군데를 손봐야 한다:

1. `@RestControllerAdvice` — 비즈니스 예외, 검증 예외, JSON 파싱 예외
2. Spring Security 설정 — `AuthenticationEntryPoint`, `AccessDeniedHandler`
3. `ErrorController` 또는 BasicErrorController 커스터마이즈 — 404, 정적 리소스 에러, Tomcat 레벨 에러

이걸 모르고 `@RestControllerAdvice`만 만들어두면, 운영 중에 "왜 어떤 에러는 우리 포맷으로 나오고 어떤 건 Spring Boot 기본 포맷으로 나오지?"의 미스터리를 만나게 된다.

---

## 5. 이어지는 개념

여기까지 오면 wddy의 학습 트랙이 한 단계 정리된다. 이어질 만한 개념:

1. **`@ControllerAdvice`의 `basePackages` / `assignableTypes` / `annotations` 속성** — Advice의 적용 범위를 더 좁힐 수 있다. 마이크로서비스에서 여러 Advice를 두고 적용 범위를 분리할 때 사용. 실무에서 모듈 분리 시 만남.
    
2. **`@ResponseStatus` vs `@ExceptionHandler` vs `ResponseStatusException`** — 예외에 HTTP 상태를 부여하는 세 가지 방법. 어떤 걸 언제 쓰는지의 트레이드오프.
    
3. **Problem Details (RFC 9457) 표준 에러 응답 포맷** — `application/problem+json` Content-Type. Spring 6부터 `ProblemDetail` 클래스로 1급 지원. 자체 에러 포맷 만들지 말고 표준 따르자는 흐름.
    
4. **`AbstractErrorController` 확장과 `ErrorAttributes` 커스터마이즈** — `BasicErrorController`를 통째로 갈아엎지 않고도 일부만 손볼 수 있는 확장 지점.
    


---

처음 시작: 아무것도 커스텀하지 않는다
   │
   ↓
운영하면서 다음 중 하나라도 발생?
   │
   ├─ 프론트가 응답 포맷 통일을 요구한다
   │       → @RestControllerAdvice + 필요시 Spring Security
   │         AuthenticationEntryPoint 정도만 추가
   │
   ├─ 응답에 민감한 정보가 노출된다
   │       → application.yml의 server.error.* 설정만 점검
   │         (대부분 여기서 해결)
   │
   ├─ 특정 경로에서 응답이 부적절하다 (302, HTML 등)
   │       → 해당 영역만 핀포인트로 커스텀
   │
   └─ 다 괜찮다
           → 그대로 둔다. 건드리지 마라.