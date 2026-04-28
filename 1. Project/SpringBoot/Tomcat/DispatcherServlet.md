#SpringBoot/Tomcat

## 1. Problem First

### 컨트롤러가 여러 개일 때 요청을 어떻게 분배하는가

서블릿 스펙만 쓰던 시절, URL마다 서블릿을 하나씩 등록해야 했다.

```xml
<!-- web.xml -->
<servlet>
    <servlet-name>userServlet</servlet-name>
    <servlet-class>com.example.UserServlet</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>userServlet</servlet-name>
    <url-pattern>/users</url-pattern>
</servlet-mapping>

<servlet>
    <servlet-name>orderServlet</servlet-name>
    <servlet-class>com.example.OrderServlet</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>orderServlet</servlet-name>
    <url-pattern>/orders</url-pattern>
</servlet-mapping>
```

URL이 100개면 서블릿도 100개다. 그리고 각 서블릿마다 아래 코드가 반복된다.

```java
public class UserServlet extends HttpServlet {
    protected void doGet(HttpServletRequest request,
                         HttpServletResponse response) {
        // 파라미터 파싱
        String id = request.getParameter("id");
        
        // 비즈니스 로직 호출
        User user = userService.findById(Long.parseLong(id));
        
        // JSON 직렬화 직접 구현
        String json = objectMapper.writeValueAsString(user);
        response.setContentType("application/json");
        response.getWriter().write(json);
    }
}
```

공통 처리(파라미터 바인딩, 직렬화, 예외 처리)가 서블릿마다 흩어진다. 한 곳에서 바꾸면 모든 서블릿을 다 바꿔야 한다.

이 문제를 해결하는 패턴이 **Front Controller**다.

---

## 2. Mechanics

### 2-1. Front Controller 패턴 — 설계 근거

모든 요청을 **단 하나의 진입점**으로 받고, 거기서 적절한 핸들러로 위임한다.

```
Before (서블릿마다):
요청 /users  → UserServlet  (파싱 + 로직 + 직렬화)
요청 /orders → OrderServlet (파싱 + 로직 + 직렬화)
요청 /items  → ItemServlet  (파싱 + 로직 + 직렬화)

After (Front Controller):
요청 /users  →
요청 /orders → DispatcherServlet → 공통처리 → 적절한 핸들러로 위임
요청 /items  →
```

> Spring 공식 레퍼런스: _"Spring MVC is designed around the front controller pattern where a central Servlet, the DispatcherServlet, provides a shared algorithm for request processing, while actual work is performed by configurable delegate components."_ (Spring MVC는 중앙 서블릿인 DispatcherServlet이 요청 처리를 위한 공유 알고리즘을 제공하고, 실제 작업은 설정 가능한 위임 컴포넌트들이 수행하는 Front Controller 패턴을 중심으로 설계되었다.)

---

### 2-2. DispatcherServlet의 정체

DispatcherServlet은 결국 **서블릿**이다. `HttpServlet`을 상속한다.

```
HttpServlet
    └── HttpServletBean          (Spring — 서블릿에 Spring 속성 주입)
            └── FrameworkServlet (Spring — doGet/doPost를 processRequest로 통일)
                    └── DispatcherServlet (Spring MVC — 실제 위임 로직)
```

Tomcat이 HTTP 요청을 받아 `service()` 메서드를 호출하면, 상속 체인을 타고 내려와 결국 `DispatcherServlet.doDispatch()`에 도달한다.

Spring Boot에서는 `DispatcherServletAutoConfiguration`이 자동으로 DispatcherServlet을 `/` 경로에 등록한다. 모든 요청이 DispatcherServlet으로 들어오는 이유다.

---

### 2-3. 초기화 — 기동 시 무슨 일이 일어나는가

DispatcherServlet은 기동 시 **전략 컴포넌트들을 초기화**한다.

```java
// DispatcherServlet.initStrategies() — 소스코드 기반
protected void initStrategies(ApplicationContext context) {
    initHandlerMappings(context);        // ① URL → 핸들러 매핑 로드
    initHandlerAdapters(context);        // ② 핸들러 실행 어댑터 로드
    initHandlerExceptionResolvers(context); // ③ 예외 처리기 로드
    initViewResolvers(context);          // ④ 뷰 리졸버 로드
    // ... 기타
}
```

이 시점에 `@RequestMapping`이 붙은 모든 메서드를 스캔해서 Map을 만들어둔다.

```
기동 시 스캔 결과 (HandlerMapping 내부):

GET  /users        → UserController.getUsers()
POST /users        → UserController.createUser()
GET  /users/{id}   → UserController.getUser(Long)
GET  /orders/{id}  → OrderController.getOrder(Long)
...
```

요청이 올 때마다 스캔하는 게 아니라, **기동 시 한 번만 스캔**하고 이후엔 Map 조회만 한다.

---

### 2-4. doDispatch() — 요청마다 실행되는 핵심 로직

```java
// DispatcherServlet.doDispatch() 핵심 흐름
protected void doDispatch(HttpServletRequest request,
                          HttpServletResponse response) throws Exception {
    Exception dispatchException = null;
    
    try {
        // ① 핸들러 조회
        HandlerExecutionChain mappedHandler = getHandler(request);
        if (mappedHandler == null) {
            noHandlerFound(request, response); // 404
            return;
        }

        // ② 핸들러 어댑터 조회
        HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

        // ③ Interceptor preHandle
        if (!mappedHandler.applyPreHandle(request, response)) {
            return; // preHandle이 false 반환하면 여기서 중단
        }

        // ④ 실제 핸들러(컨트롤러) 실행
        ModelAndView mv = ha.handle(request, response,
                                    mappedHandler.getHandler());

        // ⑤ Interceptor postHandle
        mappedHandler.applyPostHandle(request, response, mv);

    } catch (Exception ex) {
        dispatchException = ex;
    }

    // ⑥ 응답 처리 + 예외 처리 + Interceptor afterCompletion
    processDispatchResult(request, response,
                          mappedHandler, mv, dispatchException);
}
```

단계별로 내려간다.

---

### 2-5. ① HandlerMapping — URL에서 핸들러로

`getHandler(request)` 는 등록된 HandlerMapping 목록을 순서대로 순회한다.

```java
// 등록된 HandlerMapping 우선순위 순서
1. RequestMappingHandlerMapping   // @RequestMapping 처리 ← 대부분 여기서 해결
2. BeanNameUrlHandlerMapping      // Bean 이름이 URL인 경우
3. RouterFunctionMapping          // WebFlux 스타일 함수형 라우팅
```

`RequestMappingHandlerMapping`이 찾는 방법:

```
GET /users/42 요청

등록된 패턴 중 매칭 탐색:
"GET /users"      → 불일치
"GET /users/{id}" → 일치 ✓  PathVariable: {id=42}
"POST /users"     → HTTP 메서드 불일치

결과: HandlerExecutionChain {
    handler: UserController.getUser(Long),
    interceptors: [LoggingInterceptor, AuthInterceptor, ...]
}
```

반환값이 `HandlerExecutionChain`인 것에 주목. **핸들러 + 이 요청에 적용할 인터셉터 목록**을 묶어서 반환한다.

---

### 2-6. ② HandlerAdapter — 핸들러를 실행하는 방법

DispatcherServlet은 핸들러를 직접 호출하지 않는다. 왜인가.

```
Spring이 지원하는 핸들러 타입이 여러 개다:

@Controller 메서드          → RequestMappingHandlerAdapter
HttpRequestHandler 구현체   → HttpRequestHandlerAdapter
Controller 인터페이스 구현체 → SimpleControllerHandlerAdapter
```

DispatcherServlet은 핸들러 타입에 상관없이 `ha.handle()` 하나로 통일해서 호출한다. 어댑터 패턴이다.

실무에서 거의 항상 `RequestMappingHandlerAdapter`가 선택된다.

```java
// RequestMappingHandlerAdapter.handle() 내부
public ModelAndView handle(...) {
    return invokeHandlerMethod(request, response, handlerMethod);
}

private ModelAndView invokeHandlerMethod(...) {
    // ArgumentResolver로 파라미터 바인딩
    Object[] args = resolveArguments(handlerMethod, request);
    
    // 리플렉션으로 컨트롤러 메서드 호출
    Object returnValue = handlerMethod.invoke(args);
    
    // ReturnValueHandler로 반환값 처리
    handleReturnValue(returnValue, handlerMethod, response);
}
```

---

### 2-7. ArgumentResolver — 파라미터 바인딩의 실제 동작

컨트롤러 메서드 파라미터마다 담당 Resolver가 있다.

```java
@PostMapping("/users/{id}/orders")
public OrderResponse createOrder(
    @PathVariable Long id,              // PathVariableMethodArgumentResolver
    @RequestParam String coupon,        // RequestParamMethodArgumentResolver
    @RequestBody OrderRequest body,     // RequestResponseBodyMethodProcessor
    @RequestHeader String authorization // RequestHeaderMethodArgumentResolver
) { ... }
```

`@RequestBody`를 처리하는 `RequestResponseBodyMethodProcessor`의 동작:

```java
// 내부 동작
public Object resolveArgument(...) {
    // 1. Content-Type 확인
    MediaType contentType = request.getContentType(); // "application/json"
    
    // 2. 이 타입을 처리할 수 있는 MessageConverter 탐색
    for (HttpMessageConverter converter : messageConverters) {
        if (converter.canRead(OrderRequest.class, contentType)) {
            // MappingJackson2HttpMessageConverter 선택됨
            
            // 3. Request Body → Java 객체 역직렬화
            return converter.read(OrderRequest.class, request);
        }
    }
    throw new HttpMediaTypeNotSupportedException(...);
}
```

---

### 2-8. ReturnValueHandler + MessageConverter — 응답 생성

컨트롤러가 반환한 값을 HTTP 응답으로 변환한다.

`@ResponseBody` (또는 `@RestController`)가 있으면 `RequestResponseBodyMethodProcessor`가 처리한다.

```java
// Content Negotiation — 클라이언트가 원하는 형식으로
Accept: application/json → MappingJackson2HttpMessageConverter → JSON
Accept: application/xml  → MappingJackson2XmlHttpMessageConverter → XML
Accept: */*             → 등록된 첫 번째 converter → 보통 JSON
```

```java
// 내부 동작
public void handleReturnValue(Object returnValue, ...) {
    // 1. Accept 헤더와 서버 지원 MediaType 협상
    MediaType selectedContentType = contentNegotiationManager.resolveMediaTypes(request);
    
    // 2. 처리 가능한 converter 선택
    for (HttpMessageConverter converter : messageConverters) {
        if (converter.canWrite(returnValue.getClass(), selectedContentType)) {
            // 3. Java 객체 → HTTP 응답 바디 직렬화
            converter.write(returnValue, selectedContentType, response);
            return;
        }
    }
    throw new HttpMediaTypeNotAcceptableException(...);
}
```

---

### 2-9. 예외 처리 — processDispatchResult()

`doDispatch()` 에서 예외가 발생하면 `processDispatchResult()`가 처리한다.

```java
private void processDispatchResult(..., Exception exception) {
    if (exception != null) {
        // HandlerExceptionResolver 목록 순회
        mv = processHandlerException(request, response, handler, exception);
    }
    // 이후 뷰 렌더링 또는 응답 완료
}
```

```
예외 발생 시 HandlerExceptionResolver 순서:

1. ExceptionHandlerExceptionResolver
   → @ExceptionHandler가 있는가? (@ControllerAdvice 포함)
   → 있으면 해당 메서드 실행

2. ResponseStatusExceptionResolver
   → @ResponseStatus 애너테이션이 있는가?
   → 있으면 지정한 HTTP 상태코드로 응답

3. DefaultHandlerExceptionResolver
   → Spring MVC 표준 예외 처리
   → MethodArgumentNotValidException → 400
   → HttpRequestMethodNotSupportedException → 405
   → 등

4. 위 모두 해당 없음 → 예외를 서블릿 컨테이너(Tomcat)로 전파
```

**중요한 함정:**

```
Filter에서 던진 예외
    │
    │  DispatcherServlet에 도달하기 전에 발생
    │
    ▼
@ControllerAdvice로 잡히지 않는다
Tomcat의 기본 에러 페이지로 간다

→ Filter 예외는 Filter 안에서 직접 처리하거나
  별도 ErrorController를 구현해야 한다
```

---

### 2-10. 전체 흐름 한 번에

```
HTTP 요청
    │
    ▼
Tomcat (소켓 → HttpServletRequest/Response 생성)
    │
    ▼
Filter Chain (Security, Encoding 등)
    │
    ▼
DispatcherServlet.doDispatch()
    │
    ├─ HandlerMapping.getHandler()
    │      URL + HTTP Method → HandlerExecutionChain
    │
    ├─ HandlerAdapter 선택
    │      핸들러 타입에 맞는 어댑터
    │
    ├─ Interceptor.preHandle()
    │      false 반환 시 중단
    │
    ├─ ArgumentResolver
    │      파라미터 바인딩 (@PathVariable, @RequestBody 등)
    │
    ├─ Controller 메서드 실행    ← 개발자 코드
    │
    ├─ Interceptor.postHandle()
    │
    ├─ ReturnValueHandler
    │      반환값 처리 방식 결정
    │
    ├─ MessageConverter
    │      Java 객체 → JSON (직렬화)
    │
    └─ Interceptor.afterCompletion()
    │
    ▼
HTTP 응답
```

---

## 3. 공식 근거

|주장|출처|
|---|---|
|Front Controller 패턴 설명|Spring 공식 레퍼런스 — _Web on Servlet Stack, 1.1 DispatcherServlet_|
|`initStrategies()` 초기화 전략|Spring Framework 소스코드 — `DispatcherServlet.java`|
|HandlerMapping 우선순위|Spring 공식 레퍼런스 — _1.1.5 Special Bean Types_|
|Content Negotiation|Spring 공식 레퍼런스 — _1.11.2 Content Negotiation_|
|HandlerExceptionResolver 순서|Spring 공식 레퍼런스 — _1.1.7 Exception Handling_|

---

## 4. 이 설계를 이렇게 한 이유

### 얻는 것

- 공통 처리(파라미터 바인딩, 직렬화, 예외처리)를 한 곳에서 관리
- 컨트롤러는 HTTP를 몰라도 된다. Java 메서드만 작성하면 된다
- 전략 컴포넌트 교체가 쉽다 (MessageConverter, HandlerMapping 등을 Bean으로 교체 가능)

### 잃는 것

|손실|구체적 상황|
|---|---|
|**레이어가 많아 디버깅 어려움**|요청이 어느 단계에서 막혔는지 추적이 번거롭다. `logging.level.org.springframework.web=TRACE`로 DispatcherServlet 로그를 켜야 보인다|
|**Filter 예외가 @ControllerAdvice로 안 잡힘**|Security Filter에서 던진 예외가 의도한 에러 형식으로 응답되지 않는 문제. 실무에서 자주 만나는 함정|
|**리플렉션 비용**|컨트롤러 메서드 호출이 리플렉션으로 이루어진다. 일반 메서드 호출보다 느리지만 캐싱으로 완화|

---

## 5. 이어지는 개념

|순서|개념|이유|
|---|---|---|
|**1**|**Spring Security FilterChain**|DispatcherServlet 앞단인 Filter 레이어가 어떻게 인증/인가를 처리하는지. Filter 예외가 왜 @ControllerAdvice로 안 잡히는지 이해하려면 이 흐름이 필요|
|**2**|**Spring AOP + @Transactional**|컨트롤러에서 서비스를 호출할 때 `@Transactional`이 어떻게 끼어드는가. DispatcherServlet이 아닌 프록시 레이어에서 동작하는 또 다른 횡단 관심사|
|**3**|**WebFlux와 DispatcherHandler**|Spring MVC의 DispatcherServlet과 대응되는 WebFlux의 DispatcherHandler. 블로킹 모델의 한계와 논블로킹 모델의 차이|