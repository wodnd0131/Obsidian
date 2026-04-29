
쿠키 전송: 도메인 매칭 + SameSite + credentials 세 조건 모두 필요
SOP: 응답을 JS가 못 읽게 막는 것 (응답을 브라우저가 막는다)
CORS: 서버가 특정 출처에게 SOP 해제를 선언하는 것
credentials: 교차 출처 fetch에 쿠키를 포함할지 (fetch API 기본값은 same-origin)
SameSite: 쿠키 자체가 교차 사이트에 전송될지
Origin 헤더: 브라우저가 자동으로 붙이고 JS가 조작 불가

## 1. 왜 교차 출처에 쿠키가 기본으로 안 붙는가

### 쿠키의 기본 동작 방식 먼저

쿠키는 **도메인 기준**으로 저장되고 전송된다.

```
브라우저가 api.example로 요청을 보낼 때
→ 브라우저 쿠키 저장소에서 api.example 쿠키를 꺼내서 자동으로 붙임
```

여기까지는 자연스럽다.

### 문제: 쿠키가 "어느 사이트에서 요청이 시작됐는지"와 무관하게 붙는다

```
사용자가 bank.com에 로그인 → 브라우저에 bank.com 쿠키 저장
사용자가 evil.com을 방문
evil.com의 JS: fetch('https://bank.com/transfer', { method: 'POST' })

→ 브라우저가 bank.com 쿠키를 자동으로 붙여서 전송
→ bank.com 서버 입장에서는 정상 인증된 요청처럼 보임
```

이게 CSRF다.

브라우저가 교차 출처 fetch에 기본적으로 쿠키를 안 붙이는 건 이 공격을 막기 위한 방어선이다.

---

## 2. `credentials: 'include'`가 뭘 여는가

```javascript
fetch('https://api.example/data', {
    credentials: 'include'
})
```

이 옵션은 **"교차 출처 요청에도 쿠키를 포함해서 보내겠다"** 는 클라이언트 측 명시다.

기본값은 `'same-origin'`이다. 같은 출처일 때만 쿠키 포함.

| credentials 값   | 동작                 |
| --------------- | ------------------ |
| `'omit'`        | 쿠키 절대 안 보냄         |
| `'same-origin'` | 같은 출처만 쿠키 포함 (기본값) |
| `'include'`     | 교차 출처에도 쿠키 포함      |

`'include'`로 설정했다고 쿠키가 바로 전송되는 게 아니다.  
**서버가 허가해줘야** 브라우저가 응답을 JavaScript에 전달한다.

---

## 3. 서버 측 두 가지 조건

`credentials: 'include'` 요청에 서버가 응답할 때 두 조건이 모두 충족돼야 한다.

```
조건 1: Access-Control-Allow-Credentials: true
조건 2: Access-Control-Allow-Origin: 명시적 출처 (* 안 됨)
```

왜 `*`가 안 되는가.

`*`는 "누구든 이 응답을 읽어도 된다"는 뜻이다.  
자격 증명(쿠키, 인증 헤더)이 포함된 응답을 `*`로 열면,  
**인증된 데이터를 모든 출처에서 읽을 수 있게 허용**하는 것이다.

```
bank.com이 Access-Control-Allow-Origin: * 로 설정했다면:
evil.com → credentials: 'include' → bank.com의 계좌 정보를 응답으로 받음
```

브라우저는 이걸 막기 위해 `*` + credentials 조합을 원천 차단한다.

```
서버 응답:
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true

→ 브라우저: 이 조합은 허용 안 함. 응답 차단.
   콘솔 에러: "The value of the 'Access-Control-Allow-Origin' header
               must not be the wildcard '*' when the request's credentials
               mode is 'include'."
```

---

## 4. SameSite 속성 — 쿠키 레벨의 교차 출처 방어

`credentials: 'include'`를 설정했다고 해도, 쿠키 자체에 SameSite 설정이 있으면 전송이 막힌다.

SameSite는 **쿠키를 교차 사이트 요청에 보낼지 말지**를 쿠키 단위로 제어한다.

### SameSite 세 가지 값

**`SameSite=Strict`**

```
같은 사이트에서 시작한 요청에만 쿠키를 보낸다.
```

```
bank.com에서 bank.com으로 요청 → 쿠키 포함
app.example에서 bank.com으로 요청 → 쿠키 절대 안 포함
```

가장 강력한 방어. 교차 출처 인증이 완전히 불가능하다.

**`SameSite=Lax` (Chrome 기본값)**

```
같은 사이트 요청 + 최상위 레벨 GET 네비게이션(링크 클릭 등)에만 쿠키 포함.
교차 출처 fetch/XHR에는 쿠키 안 붙음.
```

```
<a href="https://bank.com">클릭</a> → 쿠키 포함 (Lax 허용)
fetch('https://bank.com/api') → 쿠키 안 포함 (Lax 차단)
```

실무에서 가장 많이 마주치는 기본값이다.  
교차 출처 API 호출에 쿠키 인증을 쓰면 여기서 막힌다.

**`SameSite=None`**

```
교차 사이트 요청에도 쿠키를 보낸다.
반드시 Secure(HTTPS)와 함께 써야 한다.
```

```
Set-Cookie: session=abc123; SameSite=None; Secure
```

교차 출처에서 쿠키 인증을 하려면 이 설정이 필요하다.

---

## 5. React ↔ Spring Boot에서 쿠키 인증이 동작하려면

세 레이어가 전부 맞아야 한다.

```
[클라이언트] credentials: 'include'
[쿠키 속성] SameSite=None; Secure
[서버 CORS] allowCredentials(true) + 명시적 allowedOrigins
```

하나라도 빠지면 쿠키가 안 붙거나 응답이 차단된다.

### Spring Boot 쿠키 발급 시

```java
@PostMapping("/login")
public ResponseEntity<Void> login(@RequestBody LoginRequest request,
                                   HttpServletResponse response) {
    String sessionToken = authService.login(request);

    Cookie cookie = new Cookie("SESSION", sessionToken);
    cookie.setHttpOnly(true);        // JS 접근 차단
    cookie.setSecure(true);          // HTTPS만
    cookie.setPath("/");
    cookie.setMaxAge(3600);

    // SameSite는 Cookie 클래스가 직접 지원 안 함 (Servlet 5.0 이전)
    // 헤더로 직접 써야 함
    response.addHeader("Set-Cookie",
        "SESSION=" + sessionToken +
        "; Path=/; HttpOnly; Secure; SameSite=None; Max-Age=3600");

    return ResponseEntity.ok().build();
}
```

Spring Boot 3.x (Servlet 6.0)부터는 `Cookie.setAttribute("SameSite", "None")`이 가능하다.

### React fetch

```javascript
// 로그인
await fetch('https://api.example/login', {
    method: 'POST',
    credentials: 'include',   // 필수
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ username, password })
});
// → 서버가 Set-Cookie: SESSION=...; SameSite=None; Secure
// → 브라우저가 api.example 쿠키 저장

// 이후 API 호출
await fetch('https://api.example/profile', {
    credentials: 'include'    // 필수. 이게 없으면 쿠키 안 붙음
});
```

---

## 6. 개발 환경에서 자주 겪는 문제

### 로컬에서 HTTPS가 없으면 SameSite=None이 동작 안 한다

`SameSite=None`은 반드시 `Secure`가 필요하다.  
`Secure`는 HTTPS에서만 동작한다.  
`localhost`는 브라우저가 예외적으로 Secure 없이도 허용하지만,  
`http://192.168.x.x` 같은 로컬 IP는 안 된다.

### Chrome 기본값이 Lax로 바뀐 시점

Chrome 80 (2020년 2월)부터 SameSite 미지정 쿠키의 기본값이 Lax가 됐다.  
이 시점 이후 기존에 잘 되던 교차 출처 쿠키 전송이 갑자기 안 되는 문제가 많이 발생했다.

### 개발 환경 임시 해결법 — 프록시

교차 출처 자체를 없애버린다.

```javascript
// vite.config.js
export default {
    server: {
        proxy: {
            '/api': {
                target: 'http://localhost:8080',
                changeOrigin: true
            }
        }
    }
}
```

```
React(3000) → /api/data
→ Vite가 localhost:8080/api/data로 프록시
→ 브라우저 입장에서 같은 출처(3000)로 요청한 것
→ CORS + SameSite 문제 없음
```

프록시를 쓰면 쿠키의 SameSite나 CORS 설정 없이도 로컬에서 동작한다.  
프로덕션에서는 Nginx가 같은 역할을 한다.

```nginx
# 같은 도메인으로 앞단에서 분기
server {
    listen 443 ssl;
    server_name example.com;

    location /  { proxy_pass http://react-app:3000; }
    location /api { proxy_pass http://spring-app:8080; }
}
```

React와 Spring이 같은 도메인 아래 있게 되므로 교차 출처 문제가 사라진다.  
프로덕션에서 CORS와 SameSite를 신경 쓰지 않아도 되는 대신,  
Nginx 설정이 틀리면 라우팅이 깨진다.

---

## 7. 정리 — 교차 출처 쿠키 전송 체크리스트

```
클라이언트:
  ☐ fetch에 credentials: 'include' 설정

쿠키:
  ☐ SameSite=None
  ☐ Secure (HTTPS 필수)
  ☐ HttpOnly (XSS 방어, 선택이지만 권장)

서버 CORS:
  ☐ Access-Control-Allow-Credentials: true
  ☐ Access-Control-Allow-Origin: 명시적 출처 (* 금지)
```

---

## 8. 이어지는 개념

1. **CSRF 방어 — SameSite와 CSRF 토큰의 관계** — SameSite=Lax/Strict이 CSRF를 얼마나 막는지, 그래도 CSRF 토큰이 필요한 경우가 왜 있는지. `credentials: 'include'`를 열었을 때 CSRF 방어선이 어떻게 달라지는지.
    
2. **JWT vs 세션 쿠키 — 교차 출처 환경에서의 선택** — 쿠키 인증의 이 복잡함 때문에 교차 출처 환경에서는 Authorization 헤더 + JWT를 쓰는 경우가 많다. 두 방식의 트레이드오프, 각각 어떤 공격에 취약한지.
    
3. **Spring Security 필터 체인** — CORS 설정, 세션 관리, CSRF 토큰이 Security 필터에서 어떤 순서로 처리되는지. 이 순서를 모르면 설정이 왜 안 먹히는지 디버깅이 어렵다.