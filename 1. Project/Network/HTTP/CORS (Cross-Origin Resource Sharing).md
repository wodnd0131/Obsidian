## 1. Problem First — CORS가 없었을 때 어떤 일이 생기나

### 동일 출처 정책(Same-Origin Policy)이 먼저다

브라우저는 기본적으로 **다른 출처의 리소스를 스크립트로 읽는 것을 막는다.**  
이게 SOP(Same-Origin Policy)다.

출처(Origin)는 세 가지가 전부 같아야 같은 출처다.

```
https://foo.example:443/page

스킴:   https
호스트: foo.example
포트:   443
```

```
https://foo.example/api      → 같은 출처 (포트 443 기본값)
http://foo.example/api       → 다른 출처 (스킴 다름)
https://bar.example/api      → 다른 출처 (호스트 다름)
https://foo.example:8080/api → 다른 출처 (포트 다름)
```

### SOP가 없으면 생기는 공격

```
사용자가 bank.com에 로그인되어 있다 (쿠키 보유).
악성 사이트 evil.com을 방문한다.

evil.com의 JavaScript:
fetch('https://bank.com/transfer', {
  method: 'POST',
  body: '{ "to": "attacker", "amount": 1000000 }'
})
// 사용자의 쿠키가 자동으로 포함되어 전송됨
// → 사용자 몰래 이체 실행
```

이게 CSRF(Cross-Site Request Forgery)의 기본 구조다.  
SOP는 이걸 막기 위해 존재한다.

### 그런데 SOP가 너무 강하면 정상적인 요청도 막힌다

```
React 앱: https://app.example:3000
Spring 서버: https://api.example:8080
```

이건 완전히 정상적인 구성인데, 출처가 다르다.  
SOP만 있으면 React에서 Spring API를 아예 호출할 수 없다.

**CORS는 "특정 조건 하에 교차 출처를 허용한다"는 서버의 명시적 선언이다.**  
SOP를 우회하는 게 아니라, 서버가 허가한 범위에서 SOP의 제한을 푸는 것이다.

---

## 2. Mechanics — 브라우저 내부에서 어떻게 동작하는가

### 핵심: CORS는 브라우저가 강제한다

```
서버는 CORS 헤더를 응답에 붙인다.
브라우저는 그 헤더를 보고 응답을 JavaScript에 넘길지 말지 결정한다.
```

**서버는 막지 않는다. 브라우저가 막는다.**

```
React fetch('/api/data')
    ↓
브라우저가 요청 전송
    ↓
서버가 응답 (데이터 포함)
    ↓
브라우저가 CORS 헤더 확인
  → 허가된 출처? → JavaScript에 응답 전달
  → 허가 안 됨?  → 응답 차단 (JavaScript는 오류만 받음)
```

서버는 이미 요청을 처리했다.  
CORS 에러는 서버가 아니라 브라우저 레벨에서 발생한다.  
이 때문에 **서버 로그에는 요청이 찍히는데 클라이언트는 에러**인 상황이 생긴다.

### 요청 유형 분류 — 단순 요청 vs 사전 요청

MDN 문서에서 두 가지 시나리오를 다룬다.

**단순 요청(Simple Request) — Preflight 없이 바로 보낸다**

아래 조건을 모두 충족해야 단순 요청이다.

```
메서드: GET, HEAD, POST 중 하나
헤더: 브라우저 자동 설정 헤더 + 아래만 허용
  - Accept
  - Accept-Language
  - Content-Language
  - Content-Type (단, text/plain, multipart/form-data, application/x-www-form-urlencoded만)
```

React + Spring Boot 환경에서 `application/json`으로 POST를 보내면  
**Content-Type이 조건에 해당하지 않아 단순 요청이 아니다.**  
실무에서 단순 요청이 되는 경우는 드물다.

```
단순 요청 흐름:

React                           Spring
  |                               |
  |-- GET /api/data ------------->|
  |   Origin: https://app.example |
  |                               |
  |<-- 200 OK --------------------|
  |   Access-Control-Allow-Origin: https://app.example
  |                               |
브라우저: Origin이 허가됨 → 응답을 React에 전달
```

**사전 요청(Preflight) — OPTIONS를 먼저 보내고 허가받는다**

단순 요청 조건을 벗어나면 브라우저가 자동으로 OPTIONS 요청을 먼저 보낸다.  
"이 요청 보내도 돼?"를 미리 확인하는 것이다.

```
React (POST + Content-Type: application/json)

1단계: Preflight

브라우저 → Spring
OPTIONS /api/data HTTP/1.1
Origin: https://app.example
Access-Control-Request-Method: POST
Access-Control-Request-Headers: content-type

← Spring 응답
204 No Content
Access-Control-Allow-Origin: https://app.example
Access-Control-Allow-Methods: POST, GET, OPTIONS
Access-Control-Allow-Headers: Content-Type
Access-Control-Max-Age: 86400

2단계: 허가됨 → 실제 요청 전송

브라우저 → Spring
POST /api/data HTTP/1.1
Origin: https://app.example
Content-Type: application/json

← Spring 응답
200 OK
Access-Control-Allow-Origin: https://app.example
{ ... }
```

`Access-Control-Max-Age: 86400`은 Preflight 결과를 86400초 캐시한다.  
그 안에는 같은 URL+메서드 조합에 대해 Preflight를 다시 보내지 않는다.

---

## 3. 공식 근거

> Fetch Living Standard (WHATWG): "The CORS protocol exists to enable sharing responses cross-origin and to allow for more versatile fetches than possible with HTML's form element."
> 
> _→ CORS 프로토콜은 교차 출처 응답 공유를 가능하게 하고, HTML form 요소보다 유연한 요청을 허용하기 위해 존재한다._

> MDN (mozilla.org/ko/docs/Web/HTTP/Guides/CORS): "보안상의 이유로 브라우저는 스크립트에서 시작한 교차 출처 HTTP 요청을 제한합니다."

---

## 4. Spring Boot에서 CORS 설정

### 방법 1: `@CrossOrigin` (컨트롤러 단위)

```java
@RestController
@CrossOrigin(origins = "https://app.example")
public class ProductController {

    @GetMapping("/products")
    public List<Product> getProducts() {
        return productService.getAll();
    }
}
```

특정 컨트롤러나 메서드에만 적용할 때.  
origin을 `*`로 열면 모든 출처 허용 — 개발 시에만 쓴다.

### 방법 2: `WebMvcConfigurer` (전역 설정, 실무 표준)

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")         // 적용 경로
            .allowedOrigins(
                "https://app.example",
                "https://www.app.example"
            )
            .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
            .allowedHeaders("Content-Type", "Authorization")
            .allowCredentials(true)            // 쿠키/인증 허용 시
            .maxAge(86400);                    // Preflight 캐시 시간
    }
}
```

### 방법 3: Spring Security와 함께 쓸 때

Spring Security가 있으면 Security 필터가 먼저 동작한다.  
`WebMvcConfigurer`만 설정하면 Security가 Preflight를 막아버릴 수 있다.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .csrf(csrf -> csrf.disable())
            // ...
        return http.build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://app.example"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("Content-Type", "Authorization"));
        config.setAllowCredentials(true);
        config.setMaxAge(86400L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return source;
    }
}
```

Spring Security 환경에서는 반드시 이 방식으로 설정해야 한다.

---

## 5. 자격 증명(Credentials) — 쿠키를 교차 출처로 보낼 때

기본적으로 교차 출처 요청에는 쿠키가 포함되지 않는다.  
쿠키 기반 인증을 쓰는 경우 양쪽 모두 설정이 필요하다.

```javascript
// React: credentials 옵션 추가
fetch('https://api.example/data', {
    credentials: 'include'  // 쿠키 포함
})
```

```java
// Spring: allowCredentials 허용
config.setAllowCredentials(true);
// + allowedOrigins에 * 쓰면 안 됨. 반드시 명시적 출처
config.setAllowedOrigins(List.of("https://app.example"));
```

MDN이 명시하는 중요한 제약:

> MDN (mozilla.org/ko/docs/Web/HTTP/Guides/CORS): "자격 증명이 포함된 요청에 응답할 때, 서버는 Access-Control-Allow-Origin 헤더의 값으로 '*' 와일드카드를 지정하는 대신 특정 출처를 반드시 지정해야 합니다."

`credentials: 'include'` + `Access-Control-Allow-Origin: *` 조합은  
브라우저가 응답 자체를 차단한다.

---

## 6. 설계 트레이드오프

### `allowedOrigins("*")`를 쓰면 잃는 것

편하지만 모든 출처에서 접근 가능해진다.  
공개 API라면 괜찮다.  
인증이 필요한 API에 `*`를 쓰면 CSRF 방어선이 무너진다.

### Preflight 비용

단순 요청이 아닌 모든 요청에 Preflight가 붙는다.  
요청 1번에 HTTP 왕복이 2번 일어난다.

```
Preflight OPTIONS → 응답
실제 요청        → 응답
```

`Access-Control-Max-Age`로 Preflight를 캐시해야 한다.  
캐시하지 않으면 API 호출마다 2배의 레이턴시가 발생한다.

### CORS는 서버 보안이 아니다

CORS는 **브라우저에서만** 강제된다.  
curl, Postman, 서버 간 통신은 CORS의 영향을 받지 않는다.

```bash
# 이건 CORS에 걸리지 않는다
curl -X POST https://api.example/data \
  -H "Content-Type: application/json" \
  -d '{"data": "..."}'
```

CORS를 보안 수단으로 오해하면 안 된다.  
인증/인가는 서버 레벨에서 별도로 구현해야 한다.

---

## 7. 이어지는 개념

1. **CSRF(Cross-Site Request Forgery)** — SOP와 CORS를 배웠으면 CSRF가 왜 여전히 필요한지 이어진다. CORS를 열었을 때 CSRF 방어가 어떻게 달라지는지, Spring Security의 CSRF 토큰이 어디서 작동하는지.
    
2. **쿠키 SameSite 속성** — `credentials: include`로 쿠키를 교차 출처 전송할 때 `SameSite=None; Secure` 설정이 필요하다. CORS와 쿠키 정책이 맞물리는 지점이다.
    
3. **Spring Security 필터 체인** — CORS 설정이 Security 환경에서 왜 따로 설정해야 하는지, 필터 순서가 어떻게 동작하는지. Spring Security를 쓰는 환경에선 이 이해가 없으면 CORS 설정이 안 먹히는 이유를 못 찾는다.