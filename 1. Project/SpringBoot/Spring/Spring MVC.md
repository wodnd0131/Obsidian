#SpringBoot/Spring 
## 1. Problem First

### MVC 패턴이 없던 시절 — 관심사가 뒤섞인 코드

```java
// 서블릿 하나에 모든 것이 섞인 구조
public class UserServlet extends HttpServlet {
    protected void doGet(HttpServletRequest request,
                         HttpServletResponse response) {

        // HTTP 파싱 (입력 처리)
        String idStr = request.getParameter("id");
        Long id = Long.parseLong(idStr);

        // 비즈니스 로직 (핵심 관심사)
        Connection conn = DriverManager.getConnection("jdbc:mysql://...");
        PreparedStatement ps = conn.prepareStatement(
            "SELECT * FROM users WHERE id = ?");
        ps.setLong(1, id);
        ResultSet rs = ps.executeQuery();
        User user = new User(rs.getLong("id"), rs.getString("name"));

        // 응답 생성 (출력 처리)
        response.setContentType("text/html");
        PrintWriter out = response.getWriter();
        out.println("<html><body>");
        out.println("<h1>" + user.getName() + "</h1>");
        out.println("</body></html>");
    }
}
```

한 클래스 안에 세 가지가 섞여있다.

- HTTP 요청 파싱
- DB 조회 비즈니스 로직
- HTML 생성

기능 하나를 바꾸려면 이 덩어리 전체를 건드려야 한다. 테스트도 불가능하다. HTTP 요청 없이는 비즈니스 로직만 단독으로 실행할 수 없다.

---
## 2. Mechanics

### 2-1. MVC 패턴 — 관심사 분리

MVC는 Spring의 개념이 아니다. **1979년 Trygve Reenskaug가 Smalltalk에서 제안한 소프트웨어 설계 패턴**이다. Spring MVC는 이 패턴을 웹에 적용한 것이다.

```
M (Model):      데이터와 비즈니스 로직
V (View):       사용자에게 보여지는 표현
C (Controller): 입력을 받아 Model과 View를 연결
```

역할을 분리하면:

```java
// Controller — HTTP 입력 처리만
@Controller
public class UserController {
    public String getUser(Long id, Model model) {
        User user = userService.findById(id); // Model에 위임
        model.addAttribute("user", user);     // View에 전달
        return "user/detail";                 // View 이름 반환
    }
}

// Model — 비즈니스 로직만 (HTTP를 모름)
@Service
public class UserService {
    public User findById(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
}

// View — 표현만 (비즈니스 로직을 모름)
// user/detail.html
// <h1>{{user.name}}</h1>
```

각 레이어가 자신의 책임만 가진다. `UserService`는 HTTP 없이도 단독으로 테스트 가능하다.

---

### 2-2. Spring MVC의 전체 구조

Spring MVC는 이 MVC 패턴을 **DispatcherServlet 중심으로 구현**한 웹 프레임워크다.

```
HTTP 요청
    │
    ▼
DispatcherServlet (Front Controller)
    │
    ├─ HandlerMapping    "어느 Controller가 처리할지 결정"
    │
    ├─ HandlerAdapter    "Controller를 어떻게 실행할지"
    │       │
    │   ArgumentResolver  "파라미터 바인딩"
    │       │
    │   [Controller 실행]  ← M, V, C 중 C
    │       │
    │   ReturnValueHandler "반환값 처리"
    │
    ├─ ViewResolver      "View 이름 → 실제 View 객체"  ← V
    │
    └─ MessageConverter  "객체 → JSON/XML"
```

DispatcherServlet이 이 컴포넌트들을 조율한다. 각 컴포넌트는 독립적으로 교체 가능하다.

---

### 2-3. Model — 데이터를 View로 전달하는 방법

`Model`은 Controller가 View에 데이터를 넘기는 **운반 객체**다.

```java
@Controller
public class UserController {

    @GetMapping("/users/{id}")
    public String getUser(@PathVariable Long id, Model model) {
        User user = userService.findById(id);

        // Model에 데이터를 담으면
        model.addAttribute("user", user);
        model.addAttribute("title", "사용자 정보");

        // View에서 꺼내 쓸 수 있음
        return "user/detail"; // View 이름
    }
}
```

```html
<!-- user/detail.html (Thymeleaf) -->
<h1 th:text="${title}">제목</h1>
<p th:text="${user.name}">이름</p>
```

**ModelAndView** 는 Model과 View 이름을 하나로 묶은 객체다. DispatcherServlet이 내부적으로 사용한다.

```java
// 명시적으로 ModelAndView를 반환하는 것도 가능
@GetMapping("/users/{id}")
public ModelAndView getUser(@PathVariable Long id) {
    ModelAndView mav = new ModelAndView("user/detail"); // View 이름
    mav.addObject("user", userService.findById(id));    // Model 데이터
    return mav;
}
```

---

### 2-4. View와 ViewResolver

Controller가 `"user/detail"` 이라는 문자열을 반환하면, ViewResolver가 이를 실제 View 객체로 변환한다.

```
Controller: return "user/detail"
                │
                ▼
        ViewResolver
        "prefix + 이름 + suffix"
        "/templates/" + "user/detail" + ".html"
                │
                ▼
        실제 파일: /templates/user/detail.html
                │
                ▼
        Thymeleaf가 HTML 렌더링
```

```yaml
# application.yml
spring:
  thymeleaf:
    prefix: classpath:/templates/
    suffix: .html
```

**View 기술은 교체 가능하다.**

|View 기술|특징|
|---|---|
|Thymeleaf|Spring Boot 기본, HTML5 친화적|
|JSP|과거 표준, Spring Boot에서 권장하지 않음|
|Freemarker|템플릿 엔진|
|JSON (MessageConverter)|@ResponseBody 사용 시 View 없음|

---

### 2-5. @RestController — View 없는 MVC

REST API는 HTML을 반환하지 않는다. View 단계를 건너뛰고 **객체를 바로 JSON으로 직렬화**한다.

```java
// @Controller + @ResponseBody = @RestController
@RestController
public class UserApiController {

    @GetMapping("/api/users/{id}")
    public UserResponse getUser(@PathVariable Long id) {
        // View 이름 문자열 대신 객체를 바로 반환
        return userService.findById(id);
        // MessageConverter가 UserResponse → JSON 변환
        // ViewResolver는 호출되지 않음
    }
}
```

```
@Controller 흐름:
Controller → Model → ViewResolver → View(HTML) → 응답

@RestController 흐름:
Controller → MessageConverter → JSON → 응답
             (ViewResolver 건너뜀)
```

---

### 2-6. Spring MVC의 레이어 구조 — 실무 관점

Spring MVC는 웹 레이어만이다. 그 아래 레이어는 Spring MVC가 아닌 Spring Framework의 영역이다.

```
[Web Layer]         — Spring MVC
Controller
    │
    ▼
[Service Layer]     — Spring Framework (DI, AOP, Transaction)
Service
    │
    ▼
[Repository Layer]  — Spring Data JPA / JDBC
Repository
    │
    ▼
[DB]
```

```java
// Web Layer — HTTP 요청/응답 처리
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @RequestBody @Valid CreateOrderRequest request) {
        OrderResponse response = orderService.create(request);
        return ResponseEntity.status(201).body(response);
    }
}

// Service Layer — 비즈니스 로직, HTTP를 모름
@Service
@Transactional
public class OrderService {

    public OrderResponse create(CreateOrderRequest request) {
        User user = userRepository.findById(request.getUserId())
            .orElseThrow(UserNotFoundException::new);
        Order order = Order.create(user, request.getItems());
        return OrderResponse.from(orderRepository.save(order));
    }
}

// Repository Layer — DB 접근만
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByUserId(Long userId);
}
```

각 레이어는 **아래 방향으로만 의존**한다. Controller는 Service를 알고, Service는 Repository를 알지만, 역방향은 없다.

---

### 2-7. Spring MVC 핵심 컴포넌트 정리

DispatcherServlet이 초기화 시 세팅하는 전략 컴포넌트들.

```
컴포넌트                역할                        기본 구현체
─────────────────────────────────────────────────────────────────
HandlerMapping         URL → 핸들러 매핑            RequestMappingHandlerMapping
HandlerAdapter         핸들러 실행 방식             RequestMappingHandlerAdapter
ArgumentResolver       파라미터 바인딩              (다수, 타입별로 존재)
MessageConverter       객체 ↔ HTTP 바디 변환        MappingJackson2HttpMessageConverter
ViewResolver           View 이름 → View 객체        ThymeleafViewResolver
ExceptionResolver      예외 처리                    ExceptionHandlerExceptionResolver
```

이 컴포넌트들은 모두 **Spring Bean으로 등록**된다. 교체하려면 같은 타입의 Bean을 직접 등록하면 Auto Configuration이 물러난다.

```java
// MessageConverter 커스터마이징 예시
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Override
    public void configureMessageConverters(
            List<HttpMessageConverter<?>> converters) {
        // Jackson 설정 커스터마이징
        MappingJackson2HttpMessageConverter converter =
            new MappingJackson2HttpMessageConverter();
        converter.setObjectMapper(customObjectMapper());
        converters.add(converter);
    }
}
```

---

## 3. 공식 근거

|주장|출처|
|---|---|
|MVC 패턴 기원|Trygve Reenskaug, _"Models-Views-Controllers"_, 1979|
|Spring MVC Front Controller 설명|Spring 공식 레퍼런스 — _Web on Servlet Stack, 1.1_|
|DispatcherServlet 전략 컴포넌트 목록|Spring 공식 레퍼런스 — _1.1.5 Special Bean Types_|
|@ResponseBody + MessageConverter|Spring 공식 레퍼런스 — _1.3.3 Handler Methods, @ResponseBody_|
|ViewResolver 동작|Spring 공식 레퍼런스 — _1.1.6 View Resolution_|

---

## 4. 이 설계를 이렇게 한 이유

### 얻는 것

- 관심사 분리 → 각 레이어를 독립적으로 테스트 가능
- 컴포넌트 교체 가능 → View 기술을 바꿔도 Controller 코드 불변
- 공통 처리 중앙화 → 예외 처리, 직렬화를 한 곳에서 관리

### 잃는 것

|손실|구체적 상황|
|---|---|
|**레이어 간 변환 비용**|Controller ↔ Service ↔ Repository 사이에 DTO 변환이 필요. 레이어가 많을수록 변환 코드가 늘어난다|
|**단순한 CRUD에 과한 구조**|간단한 조회 하나에도 Controller → Service → Repository 3단계를 거친다. 작은 프로젝트에서 오버엔지니어링이 될 수 있다|
|**블로킹 I/O 모델**|Tomcat 스레드 하나가 요청 하나를 처음부터 끝까지 담당한다. DB 조회 대기 중에도 스레드가 점유된다. 높은 동시성이 필요하면 WebFlux(비동기)를 고려해야 한다|

---

## 5. 이어지는 개념

|순서|개념|이유|
|---|---|---|
|**1**|**Spring AOP + @Transactional**|Service 레이어에서 트랜잭션이 어떻게 동작하는가. Spring MVC 흐름을 알면 "Controller에서 @Transactional이 왜 권장되지 않는가"가 바로 연결된다|
|**2**|**Spring Security FilterChain**|Spring MVC 앞단 Filter 레이어의 실제 구현. 인증/인가가 MVC 흐름의 어느 지점에서 끼어드는지|
|**3**|**WebFlux**|Spring MVC의 블로킹 모델 한계를 해결하는 비동기 웹 프레임워크. MVC 구조를 알아야 WebFlux가 무엇을 바꿨는지 보인다|