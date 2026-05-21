# AuthenticationEntryPoint와 AccessDeniedHandler

## 1. Problem First — Spring Security의 기본 응답이 왜 문제가 되는가

wddy가 Spring Security를 설정만 켜고 아무 커스텀도 안 한 상태를 상상해보자. 인증/인가 실패 시 어떤 응답이 나가는지:

### 시나리오 A: 토큰 없이 보호된 API 호출

```http
GET /api/users/me
(Authorization 헤더 없음)
```

Spring Security 기본 응답:

```http
HTTP/1.1 403 Forbidden
Content-Type: text/html;charset=utf-8

<!DOCTYPE html>
<html>
<head><title>...</title></head>
<body>
  <h1>Whitelabel Error Page</h1>
  <p>This application has no explicit mapping for /error, ...</p>
</body>
</html>
```

**문제 두 가지**:

1. **상태 코드가 403** (Forbidden). 의미상 401(Unauthorized)이 맞다. 인증되지 않은 사용자는 401, 인증은 됐는데 권한이 없으면 403
2. **HTML 응답**. REST API 클라이언트는 JSON을 기대하는데 HTML이 옴

### 시나리오 B: 일반 사용자가 관리자 API 호출

```http
GET /api/admin/users
Authorization: Bearer eyJ... (정상 토큰, 일반 사용자)
```

`@PreAuthorize("hasRole('ADMIN')")`가 막음. 응답:

```http
HTTP/1.1 403 Forbidden
Content-Type: text/html;charset=utf-8

<!DOCTYPE html>
... HTML 에러 페이지 ...
```

여기는 403이 맞지만, 여전히 HTML.

### 시나리오 C: 만료된 토큰

```http
GET /api/users/me
Authorization: Bearer eyJ... (만료된 토큰)
```

JWT Filter가 `ExpiredJwtException` 던짐 → ... 흠, 이건 wddy가 만든 Filter가 어떻게 짰느냐에 따라 다름. 잘못 짜면 500이 나갈 수도 있다.

### 근본 문제

REST API 환경에서:

- 인증 안 됨과 권한 부족이 **구분되지 않는다** (둘 다 403)
- 응답이 **HTML**이라 클라이언트가 파싱 못함
- **메시지가 일관되지 않다** (만든 사람마다 다름)

그래서 프론트엔드 개발자가 슬랙에 와서 "401인지 403인지 안 보여서 토큰 갱신 시점을 못 잡겠어요"라고 한다.

이 두 가지 문제를 분리해서 해결하는 게 `AuthenticationEntryPoint`와 `AccessDeniedHandler`다. **이름이 추상적이지만 역할은 단순하다**.

---

## 2. Mechanics — 누가 언제 호출되는가

### 핵심 구분: 두 가지 실패는 다른 일이다

| |AuthenticationEntryPoint|AccessDeniedHandler|
|---|---|---|
|언제|**인증 자체가 안 됨**|**인증은 됐는데 권한이 없음**|
|영어로|"Who are you?" 답을 못함|"I know you, but you can't do this"|
|HTTP 상태|401 Unauthorized|403 Forbidden|
|원인 예시|토큰 없음, 만료된 토큰, 잘못된 토큰|일반 사용자가 ADMIN 자원 접근|
|Spring 예외|`AuthenticationException` 계열|`AccessDeniedException`|

이름이 헷갈리는 부분 짚고 가자:

- **EntryPoint** = "보호된 자원에 진입하려는 시점" → 인증되지 않은 사용자를 막고 **인증을 요구하는 진입점**. 폼 로그인 시절에는 여기서 로그인 페이지로 리다이렉트했음
- **AccessDeniedHandler** = 접근이 거부된 상황을 다루는 핸들러

### 어디서 호출되는가

`ExceptionTranslationFilter`가 호출한다. 이 Filter는 Spring Security 필터 체인의 끝부분에 위치한다.

```
[Spring Security Filter Chain]
   │
   ├─ SecurityContextHolderFilter
   ├─ LogoutFilter
   ├─ JwtAuthenticationFilter (wddy 작성)
   ├─ UsernamePasswordAuthenticationFilter
   ├─ ...
   ├─ ★ ExceptionTranslationFilter ★ ←─── 여기가 핵심
   └─ AuthorizationFilter
```

`ExceptionTranslationFilter`의 동작 (단순화):

```java
public class ExceptionTranslationFilter extends GenericFilterBean {
    private AuthenticationEntryPoint authenticationEntryPoint;
    private AccessDeniedHandler accessDeniedHandler;

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
        try {
            chain.doFilter(req, res);
            // ★ 자기 뒤에 있는 Filter들 실행 (AuthorizationFilter 포함)
        }
        catch (Exception ex) {
            // ★ 자기 뒤의 Filter에서 던진 예외를 잡는다
            
            // 1. 예외 체인을 따라가서 우리가 아는 예외인지 확인
            RuntimeException securityException = findCause(ex,
                AuthenticationException.class, AccessDeniedException.class);
            
            if (securityException instanceof AuthenticationException) {
                // 인증 실패 → EntryPoint 호출
                handleAuthenticationException(req, res, (AuthenticationException) securityException);
            }
            else if (securityException instanceof AccessDeniedException) {
                // 인가 실패 → AccessDeniedHandler 호출
                handleAccessDeniedException(req, res, (AccessDeniedException) securityException);
            }
            else {
                // 우리가 모르는 예외 → 다시 던진다 (Tomcat이 /error로 보낼 것)
                throw ex;
            }
        }
    }

    private void handleAuthenticationException(req, res, AuthenticationException ex) {
        // SecurityContext 비우기
        SecurityContextHolder.clearContext();
        // ★ EntryPoint 호출
        authenticationEntryPoint.commence(req, res, ex);
    }

    private void handleAccessDeniedException(req, res, AccessDeniedException ex) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null || auth instanceof AnonymousAuthenticationToken) {
            // 익명 사용자의 인가 실패 → 사실상 인증 안 된 것 → EntryPoint로
            authenticationEntryPoint.commence(req, res, new InsufficientAuthenticationException(...));
        } else {
            // 인증된 사용자의 인가 실패 → AccessDeniedHandler
            accessDeniedHandler.handle(req, res, ex);
        }
    }
}
```

**여기 미묘한 부분**: 익명 사용자가 권한이 부족하면 `AccessDeniedException`이 던져지지만, `ExceptionTranslationFilter`는 그걸 **EntryPoint로 보낸다**. 왜냐하면 사실 그 사용자에게 필요한 건 "권한"이 아니라 "인증"이니까. 이 분기 로직 덕분에 클라이언트가 받는 응답이 의미적으로 정확해진다.

### 인터페이스 자체

```java
// AuthenticationEntryPoint
public interface AuthenticationEntryPoint {
    void commence(HttpServletRequest request,
                  HttpServletResponse response,
                  AuthenticationException authException) throws IOException, ServletException;
}

// AccessDeniedHandler
public interface AccessDeniedHandler {
    void handle(HttpServletRequest request,
                HttpServletResponse response,
                AccessDeniedException accessDeniedException) throws IOException, ServletException;
}
```

둘 다 **응답 객체를 직접 받는다**. `ResponseEntity`를 반환하는 게 아니라, `response.getWriter()`에 직접 쓰는 방식. Filter 영역의 컴포넌트라서 Spring MVC의 응답 모델을 쓸 수 없다.

### 구현 예시

wddy 환경에 맞춰 짜보면:

```java
@Component
public class CustomAuthenticationEntryPoint implements AuthenticationEntryPoint {
    private final ObjectMapper objectMapper;

    @Override
    public void commence(HttpServletRequest req, HttpServletResponse res,
                         AuthenticationException ex) throws IOException {
        res.setStatus(HttpStatus.UNAUTHORIZED.value());  // ★ 401
        res.setContentType(MediaType.APPLICATION_JSON_VALUE);
        res.setCharacterEncoding("UTF-8");

        ErrorResponse body = new ErrorResponse(
            "AUTHENTICATION_REQUIRED",
            "인증이 필요합니다",
            Instant.now(),
            req.getRequestURI()
        );
        objectMapper.writeValue(res.getWriter(), body);
    }
}

@Component
public class CustomAccessDeniedHandler implements AccessDeniedHandler {
    private final ObjectMapper objectMapper;

    @Override
    public void handle(HttpServletRequest req, HttpServletResponse res,
                       AccessDeniedException ex) throws IOException {
        res.setStatus(HttpStatus.FORBIDDEN.value());  // ★ 403
        res.setContentType(MediaType.APPLICATION_JSON_VALUE);
        res.setCharacterEncoding("UTF-8");

        ErrorResponse body = new ErrorResponse(
            "ACCESS_DENIED",
            "접근 권한이 없습니다",
            Instant.now(),
            req.getRequestURI()
        );
        objectMapper.writeValue(res.getWriter(), body);
    }
}
```

이걸 SecurityConfig에 등록:

```java
@Configuration
public class SecurityConfig {
    @Bean
    public SecurityFilterChain apiSecurityChain(HttpSecurity http,
            CustomAuthenticationEntryPoint entryPoint,
            CustomAccessDeniedHandler deniedHandler) throws Exception {
        http
            .securityMatcher("/api/**")
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(entryPoint)       // ← 등록
                .accessDeniedHandler(deniedHandler)         // ← 등록
            );
        return http.build();
    }
}
```

### CSRF 실패는 어디로 가나

wddy의 환경에서 중요한 디테일: **CSRF 토큰 불일치 = `AccessDeniedException`**.

`CsrfFilter`가 CSRF 토큰 검증에 실패하면 `InvalidCsrfTokenException`(이건 `AccessDeniedException`의 하위 클래스)을 던진다. 이걸 `ExceptionTranslationFilter`가 받아서 `AccessDeniedHandler`로 위임한다.

즉 wddy의 `/auth/refresh`에서 CSRF 토큰이 안 맞으면:

- 응답: 403
- 처리: `AccessDeniedHandler`

만약 wddy가 "CSRF 실패는 좀 더 명확한 메시지로 응답하고 싶다"면 `AccessDeniedHandler` 안에서 예외 타입을 분기할 수 있다:

```java
@Override
public void handle(HttpServletRequest req, HttpServletResponse res,
                   AccessDeniedException ex) throws IOException {
    res.setStatus(HttpStatus.FORBIDDEN.value());
    res.setContentType(MediaType.APPLICATION_JSON_VALUE);

    String code = ex instanceof InvalidCsrfTokenException 
        ? "CSRF_TOKEN_INVALID" 
        : "ACCESS_DENIED";
    String message = ex instanceof InvalidCsrfTokenException
        ? "CSRF 토큰이 유효하지 않습니다"
        : "접근 권한이 없습니다";

    ErrorResponse body = new ErrorResponse(code, message, Instant.now(), req.getRequestURI());
    objectMapper.writeValue(res.getWriter(), body);
}
```

---

## 3. 공식 근거

**Spring Security 공식 레퍼런스 (2순위) — ExceptionTranslationFilter**:

> "ExceptionTranslationFilter allows translation of AccessDeniedException and AuthenticationException into HTTP responses.
> 
> - If the user is not authenticated or it is an AuthenticationException, then Start Authentication.
>     - The SecurityContextHolder is cleared out.
>     - The HttpServletRequest is saved so that it can be used to replay the original request once authentication is successful.
>     - AuthenticationEntryPoint is used to request credentials from the client. For example, it might redirect to a log in page or send a WWW-Authenticate header.
> - Otherwise, if it is an AccessDeniedException, then Access Denied. AccessDeniedHandler is invoked to handle access denied."
> 
> (ExceptionTranslationFilter는 AccessDeniedException과 AuthenticationException을 HTTP 응답으로 변환할 수 있게 한다.
> 
> - 사용자가 인증되지 않았거나 AuthenticationException인 경우, 인증을 시작한다.
>     - SecurityContextHolder가 비워진다.
>     - HttpServletRequest가 저장되어, 인증이 성공한 후 원본 요청을 재생하는 데 사용될 수 있다.
>     - AuthenticationEntryPoint가 클라이언트에게 자격 증명을 요청하는 데 사용된다. 예를 들어 로그인 페이지로 리다이렉트하거나 WWW-Authenticate 헤더를 보낼 수 있다.
> - 그렇지 않고 AccessDeniedException인 경우, 접근 거부. AccessDeniedHandler가 접근 거부를 처리하기 위해 호출된다.) — Spring Security Reference, "Servlet Architecture — ExceptionTranslationFilter"

**AuthenticationEntryPoint Javadoc (2순위)**:

> "Used by ExceptionTranslationFilter to commence an authentication scheme. ... The implementation should modify the headers on the ServletResponse as necessary to commence the authentication process." (인증 절차를 시작하기 위해 ExceptionTranslationFilter에 의해 사용된다. ... 구현은 인증 프로세스를 시작하기 위해 필요에 따라 ServletResponse의 헤더를 수정해야 한다.) — Spring Security Javadoc, `org.springframework.security.web.AuthenticationEntryPoint`

**HTTP 상태 코드 의미 (1순위, RFC 9110)**:

> "The 401 (Unauthorized) status code indicates that the request has not been applied because it lacks valid authentication credentials for the target resource. ... The 403 (Forbidden) status code indicates that the server understood the request but refuses to fulfill it. ... If authentication credentials were provided in the request, the server considers them insufficient to grant access." (401 (Unauthorized) 상태 코드는 요청이 대상 리소스에 대한 유효한 인증 자격 증명을 가지지 않아 적용되지 않았음을 나타낸다. ... 403 (Forbidden) 상태 코드는 서버가 요청을 이해했지만 이행을 거부함을 나타낸다. ... 요청에 인증 자격 증명이 제공되었다면, 서버는 그것이 접근을 허가하기에 불충분하다고 본다.) — RFC 9110, §15.5.2 "401 Unauthorized", §15.5.4 "403 Forbidden"

이 RFC가 EntryPoint/AccessDeniedHandler의 분리를 정당화한다. **401과 403은 의미가 다르다**. 그래서 둘을 처리하는 컴포넌트도 다르다.

---

## 4. 이 설계를 이렇게 한 이유

### 왜 둘로 나누었나

한 인터페이스에서 분기로 처리할 수도 있었다. 그런데 분리한 이유:

- **의미적 차이가 크다**. 인증 실패와 인가 실패는 클라이언트가 다르게 대응한다. 인증 실패면 토큰 갱신을 시도하고, 인가 실패면 사용자에게 "권한 없음" 메시지를 보임. 응답이 명확히 구분되어야 클라이언트 로직이 깨끗해짐
- **확장 지점이 분리됨**. 인증 응답만 바꾸고 인가 응답은 그대로 두는 식의 부분 커스텀이 가능. 한 핸들러였다면 일부만 바꾸기 어려움
- **다른 인증 방식과의 호환성**. 폼 로그인은 EntryPoint에서 로그인 페이지로 리다이렉트, REST API는 EntryPoint에서 401 응답, OAuth2는 EntryPoint에서 인증 서버로 리다이렉트 — 모두 같은 인터페이스로 처리되지만 구현이 다름. 이 다형성이 가치 있음

### 잃는 것

- **익명 사용자 분기가 처음엔 헷갈림**. "권한 부족 = AccessDeniedHandler 호출"이라고 단순히 외우면 안 됨. 익명 사용자의 권한 부족은 EntryPoint로 가는 게 직관에 반함
- **구현 책임**. `ResponseEntity` 같은 추상화 못 쓰고 `response.getWriter()` 직접 다뤄야 함. Filter 영역의 제약
- **두 개를 다 짜야 함**. 비슷한 코드가 두 클래스에 생기기 쉬움. 공통 로직은 별도 컴포넌트로 추출하는 편이 좋음

### 안 짜도 되는 경우

wddy의 직관과 연결되는 지점:

- **백엔드가 프론트와 응답 포맷 계약을 강하게 갖지 않으면**: 그냥 두자. Spring Security 기본 응답으로도 401/403 상태 코드는 정확히 나간다. 클라이언트가 status 코드만 보면 충분
- **단순한 내부 API**: 굳이 안 만들어도 됨
- **MVP 단계**: 일단 동작이 우선. 응답 포맷은 나중에 정리

**짜야 하는 경우**:

- 프론트엔드와 응답 스키마 계약이 명확히 있음 (`{ code, message, ... }`)
- 401과 403을 명확히 구분해서 응답해야 함 (Spring Security가 가끔 잘못 보냄)
- CSRF 실패 같은 특정 케이스를 구분해야 함
- 응답에 추가 정보(요청 ID, 타임스탬프 등)를 일관되게 넣어야 함

---

## wddy의 환경에 매핑

wddy의 인증 구조에서 두 핸들러가 처리하는 케이스:

**`AuthenticationEntryPoint`가 호출되는 케이스**:

- Authorization 헤더 없는데 보호된 API 호출
- 만료된 Access Token (JWT Filter가 `AuthenticationException`을 던지도록 짰을 때)
- 위조된 Access Token
- DB에서 사용자를 못 찾음 (`UsernameNotFoundException`)

**`AccessDeniedHandler`가 호출되는 케이스**:

- 인증된 일반 사용자가 `/api/admin/**` 접근
- `@PreAuthorize("hasRole('ADMIN')")` 실패
- **`/auth/refresh` 요청 시 CSRF 토큰 불일치**
- CORS preflight 관련 이슈 (간접적)

wddy 환경에서 특히 중요한 건 마지막 케이스 — CSRF 실패. 만약 wddy가 CSRF 실패 응답을 일관되게 만들고 싶다면 `AccessDeniedHandler`가 그 자리다. `@ControllerAdvice`로는 못 잡는다 (`CsrfFilter`에서 발생하므로 영역 ②).

---

## 두 핸들러를 짤 때의 실무 팁

### JWT Filter와 EntryPoint의 협력

wddy가 JWT Filter를 짤 때, 토큰 검증 실패를 어떻게 처리할지 두 가지 선택지:

**선택지 A**: Filter에서 직접 응답 작성

```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    protected void doFilterInternal(...) {
        try {
            // 토큰 검증
            Authentication auth = jwtService.validate(token);
            SecurityContextHolder.getContext().setAuthentication(auth);
        } catch (ExpiredJwtException e) {
            // ★ 여기서 직접 응답
            response.setStatus(401);
            response.getWriter().write("...");
            return;
        }
        chain.doFilter(req, res);
    }
}
```

**선택지 B**: Filter는 SecurityContext 안 채우기만, EntryPoint가 응답 담당

```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    protected void doFilterInternal(...) {
        try {
            Authentication auth = jwtService.validate(token);
            SecurityContextHolder.getContext().setAuthentication(auth);
        } catch (JwtException e) {
            // SecurityContext에 아무것도 안 채움. 그냥 통과
            // ★ AuthorizationFilter가 "인증 안 됨" 처리 → EntryPoint 호출됨
        }
        chain.doFilter(req, res);
    }
}
```

**선택지 B가 권장**된다. 책임이 분리되기 때문. JWT Filter는 "검증하고 컨텍스트 채우는 일", EntryPoint는 "응답 작성하는 일". 응답 포맷 바꾸려면 EntryPoint만 손대면 됨.

이게 wddy가 앞서 다룬 **"각 Filter는 단일 책임"** 의 구체적 적용.

### Filter에서 던진 예외 vs SecurityContext 안 채우기

다만 미묘한 케이스가 있다. JWT 검증 중 만료/위조 같은 케이스는 위처럼 SecurityContext를 안 채우는 게 자연스럽지만, 어떤 경우(예: 사용자 DB 조회 중 일시적 장애)는 명확히 에러를 알리고 싶을 수 있다. 그럴 때는 `AuthenticationException`을 던지면 `ExceptionTranslationFilter`가 EntryPoint로 보낸다.

```java
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    protected void doFilterInternal(...) {
        try {
            Authentication auth = jwtService.validate(token);
            SecurityContextHolder.getContext().setAuthentication(auth);
        } catch (ExpiredJwtException e) {
            // 만료는 그냥 통과 (SecurityContext 안 채움)
            // → AuthorizationFilter에서 InsufficientAuthenticationException → EntryPoint
        } catch (SignatureException e) {
            // 위조는 명시적으로 알림
            throw new BadCredentialsException("Invalid JWT signature", e);
            // → ExceptionTranslationFilter가 잡음 → EntryPoint
        }
        chain.doFilter(req, res);
    }
}
```

이런 식으로 **예외를 던지든, SecurityContext를 비워두든, 결국 EntryPoint에 도달**한다. EntryPoint는 두 경로의 종착점이다.

---

## 5. 이어지는 개념

여기서 자연스럽게 이어지는 것:

1. **`AuthenticationFailureHandler` / `AuthenticationSuccessHandler`** — EntryPoint는 "보호 자원에 진입 차단"이지만, 명시적인 로그인 요청 자체의 성공/실패는 다른 핸들러가 다룬다. 폼 로그인이나 OAuth2 callback 처리 시 만남.
    
2. **`SecurityContextRepository`와 stateless 인증** — JWT 같은 stateless 인증에서 SecurityContext가 어떻게 요청마다 새로 만들어지는지. `HttpSessionSecurityContextRepository` vs `RequestAttributeSecurityContextRepository`. wddy의 Access Token 구조와 직결.
    
3. **`AuthorizationManager`와 인가 정책의 추상화** — `@PreAuthorize`가 내부적으로 어떻게 동작하는지. 권한 검사를 코드로 표현하는 방식. wddy의 비즈니스 룰이 복잡해지면 만남.
    
4. **Method Security와 AOP 프록시 — 자기 호출 문제** — wddy가 이전에 학습한 Spring Bean의 self-invocation 문제와 직접 연결. `@PreAuthorize`가 같은 클래스 내부 메서드 호출에서 동작 안 하는 이유.
    

순서: 2 → 1 → 3 → 4. 2번이 wddy의 현재 학습(JWT 인증 구조)과 가장 직접 연결되고, 그 뒤로는 인증/인가 모델의 깊이를 더하는 순서. 4번은 Spring 학습 트랙의 Bean/AOP와 인가의 교차점이라 마지막에 정리하기 좋다.