# OAuth 2.0 / OIDC 전체 흐름

---

## 1. Problem First

### 시나리오 A: OAuth 이전 — 서드파티 앱에 비밀번호를 주던 시절

```
"구글 연락처를 가져오려면 구글 아이디/비밀번호를 입력하세요"

사용자 → 서드파티 앱에 구글 비밀번호 입력
            ↓
         서드파티 앱이 구글에 직접 로그인
            ↓
         연락처 가져옴
```

**근본적인 문제:**

- 서드파티 앱이 비밀번호를 저장하거나 다른 용도로 쓸 수 있다
- 앱에 부여한 권한을 취소하려면 비밀번호를 바꿔야 한다 — 모든 앱에서 로그아웃
- 앱이 침해당하면 구글 계정 전체가 노출된다

**핵심 문제: 인증(Authentication)과 인가(Authorization)가 분리되지 않았다.**

---

### 시나리오 B: OAuth 2.0을 도입했는데 OIDC를 모르고 쓴 경우

```java
// OAuth 2.0 Access Token으로 사용자 정보를 가져오는 코드
String accessToken = getAccessTokenFromGoogle();

// Google UserInfo API 호출
ResponseEntity<Map> userInfo = restTemplate.exchange(
    "https://www.googleapis.com/oauth2/v3/userinfo",
    HttpMethod.GET,
    new HttpEntity<>(headers),
    Map.class
);

String userId = userInfo.getBody().get("sub").toString();
```

**이게 왜 문제인가:**

```
OAuth 2.0 Access Token의 원래 목적:
"이 토큰을 가진 사람은 리소스에 접근할 수 있다"
→ 리소스 접근 권한 부여 (Authorization)

사용자가 누구인지(Authentication)는 OAuth 2.0 명세에 없다.
UserInfo API 응답 형식이 Google/Kakao/Naver마다 다르다.
Access Token이 유효하다고 해서 사용자 신원이 보장되지 않는다.
```

**이것이 OIDC(OpenID Connect)가 등장한 이유다.** OAuth 2.0 위에 인증 레이어를 표준화한 것이 OIDC다.

---

### 시나리오 C: Authorization Code 없이 토큰을 URL에 담아 받은 경우

```
// Implicit Flow (현재 deprecated)
https://myapp.com/callback#access_token=eyJ...&token_type=bearer

문제:
- URL Fragment는 브라우저 히스토리에 남는다
- Referer 헤더로 토큰이 유출된다
- 중간자가 토큰을 가로채면 재사용 가능
```

---

## 2. Mechanics

### 2-1. OAuth 2.0과 OIDC의 관계 — 계층 구조

```
┌─────────────────────────────────────────┐
│            OIDC (OpenID Connect)         │
│  - 사용자 신원 인증 표준화               │
│  - ID Token (JWT) 발급                  │
│  - UserInfo 엔드포인트 표준화            │
├─────────────────────────────────────────┤
│              OAuth 2.0                   │
│  - 리소스 접근 권한 위임                 │
│  - Access Token 발급                    │
│  - 4가지 Grant Type                     │
└─────────────────────────────────────────┘

OAuth 2.0: "A가 B의 리소스에 접근하도록 허용"
OIDC:      "A가 누구인지 B에게 증명"
```

> 📎 **근거:** OpenID Connect Core 1.0 — _"OpenID Connect 1.0 is a simple identity layer on top of the OAuth 2.0 protocol."_ (OpenID Connect 1.0은 OAuth 2.0 프로토콜 위의 단순한 신원 확인 레이어다.)

---

### 2-2. Authorization Code Flow — 가장 중요한 흐름

소셜 로그인에서 실제로 쓰이는 흐름이다.

![[../../../Repository/OAuth 2.0, OIDC 전체 흐름-1.png]]
**왜 code → token 교환을 서버에서 하는가:**

```
code는 URL 파라미터로 전달된다 (브라우저에 노출)
→ code 자체는 1회용, 수명이 짧음 (보통 10분)
→ code만으로는 아무것도 못 함

token 교환은 서버 to 서버 (백채널)
→ client_secret이 필요 (서버만 알고 있음)
→ 브라우저에 절대 노출 안 됨

결론:
code 탈취 → 재사용 불가 (client_secret 없음)
token 탈취 → 실제 피해 (그래서 HTTPS 필수)
```

> 📎 **근거:** RFC 6749 — _"The OAuth 2.0 Authorization Framework"_ §4.1 Authorization Code Grant

---

### 2-3. state 파라미터 — CSRF 방어

```java
// 로그인 요청 시
String state = UUID.randomUUID().toString();
session.setAttribute("oauth_state", state);

String authUrl = "https://accounts.google.com/o/oauth2/auth"
    + "?client_id=" + clientId
    + "&state=" + state  // 세션에 저장한 값
    + "...";

// 콜백 수신 시
String returnedState = request.getParameter("state");
String savedState = (String) session.getAttribute("oauth_state");

if (!savedState.equals(returnedState)) {
    throw new SecurityException("CSRF 공격 의심");
}
```

**state가 없으면:**

```
공격자가 만든 악성 링크:
https://myapp.com/callback?code=공격자의_code

피해자가 이 링크를 클릭하면:
→ 공격자의 구글 계정으로 내 서비스에 로그인됨
→ 피해자의 세션이 공격자 계정과 연결됨 (Account Fixation)
```

> 📎 **근거:** RFC 6749 §10.12 — _"Cross-Site Request Forgery"_

---

### 2-4. OIDC의 ID Token — OAuth 2.0과 결정적 차이

**Access Token vs ID Token:**

```
Access Token:
- 리소스 서버에 제출하는 열쇠
- "이걸 가진 사람에게 리소스를 줘라"
- 내용이 불투명해도 됨 (리소스 서버만 해석하면 됨)
- 사용자 신원 정보 없어도 됨

ID Token (OIDC):
- 클라이언트(내 서버)가 읽는 사용자 신원 증명서
- JWT 형식으로 표준화
- 반드시 포함해야 하는 클레임 명세가 있음
- 발급자(Google)의 서명으로 무결성 보장
```

**ID Token의 표준 클레임:**

```json
{
  "iss": "https://accounts.google.com",  // 발급자
  "sub": "110169484474386276334",         // 사용자 고유 ID (변하지 않음)
  "aud": "my-client-id",                 // 수신자 (내 앱)
  "exp": 1311281970,                     // 만료시간
  "iat": 1311280970,                     // 발급시간
  "nonce": "랜덤값",                      // Replay Attack 방어
  "email": "user@gmail.com",             // 선택적
  "name": "홍길동"                        // 선택적
}
```

**ID Token 검증 순서 — 반드시 전부 해야 한다:**

```java
// Spring Security OAuth2 내부 검증 로직과 동일한 순서
public void validateIdToken(Jwt idToken) {

    // 1. 서명 검증 (Google 공개키로)
    jwtDecoder.decode(idToken.getTokenValue());

    // 2. issuer 확인
    assert idToken.getIssuer().equals("https://accounts.google.com");

    // 3. audience 확인 (내 client_id인지)
    assert idToken.getAudience().contains(myClientId);

    // 4. 만료 확인
    assert idToken.getExpiresAt().isAfter(Instant.now());

    // 5. nonce 확인 (Replay Attack 방어)
    String savedNonce = session.getAttribute("nonce");
    assert idToken.getClaim("nonce").equals(savedNonce);
}
```

**nonce가 없으면:**

```
공격자가 이전에 탈취한 ID Token을 재사용 가능
→ 만료 전이라면 다시 로그인 없이 인증 통과
nonce = 요청마다 다른 랜덤값 → 재사용 불가
```

> 📎 **근거:** OpenID Connect Core 1.0 §3.1.3.7 — _"ID Token Validation"_

---

### 2-5. PKCE — Authorization Code Flow의 보안 강화

**Authorization Code Flow의 남은 취약점:**

```
모바일 앱 환경:
- client_secret을 앱 바이너리에 넣으면 역공학으로 추출 가능
- redirect_uri가 커스텀 스킴 (myapp://callback)
  → 다른 앱이 같은 스킴을 등록하면 code를 가로챌 수 있음
```

**PKCE(Proof Key for Code Exchange) 동작:**

```
1. 로그인 요청 전:
   code_verifier = 랜덤 문자열 (43~128자)
   code_challenge = BASE64URL(SHA256(code_verifier))

2. Authorization 요청에 포함:
   ?code_challenge=abc123...
   &code_challenge_method=S256

3. Google이 code_challenge를 저장

4. Token 요청 시:
   code_verifier=원본값 전송

5. Google 검증:
   SHA256(code_verifier) == 저장된 code_challenge ?
   → 일치하면 token 발급
```

```
공격자가 code를 가로채도:
→ code_verifier를 모름
→ token 교환 불가
```

```java
// Spring Security 6.x에서 PKCE 활성화
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.oauth2Login(oauth2 -> oauth2
        .authorizationEndpoint(authorization -> authorization
            .authorizationRequestRepository(
                new HttpSessionOAuth2AuthorizationRequestRepository()
                // PKCE 파라미터 자동 생성 + 저장
            )
        )
    );
}
```

> 📎 **근거:** RFC 7636 — _"Proof Key for Code Exchange by OAuth Public Clients"_

---

### 2-6. Refresh Token — 동작과 보안

```
Access Token 만료 (보통 1시간)
    ↓
Refresh Token으로 재발급 요청 (서버 to 서버)
    ↓
새 Access Token + 새 Refresh Token 발급
    ↓
기존 Refresh Token 무효화 (Rotation)
```

**Refresh Token Rotation이 중요한 이유:**

```
Refresh Token이 탈취됐을 때:

Rotation 없음:
- 공격자가 Refresh Token으로 계속 Access Token 재발급
- 피해자도 같은 Refresh Token으로 재발급 → 둘 다 유효한 세션

Rotation 있음:
- 공격자가 먼저 사용 → 새 Refresh Token 발급
- 피해자가 나중에 사용 → 이미 사용된 토큰 → 서버가 탐지
- 해당 사용자의 모든 Refresh Token 무효화 가능
```

```java
// Spring Authorization Server Refresh Token Rotation 설정
@Bean
public RegisteredClientRepository registeredClientRepository() {
    RegisteredClient client = RegisteredClient.withId(UUID.randomUUID().toString())
        .tokenSettings(TokenSettings.builder()
            .refreshTokenTimeToLive(Duration.ofDays(30))
            .reuseRefreshTokens(false) // Rotation 활성화
            .build())
        .build();
}
```

> 📎 **근거:** RFC 6749 §10.4 — _"Refresh Tokens"_, OAuth 2.0 Security Best Current Practice (RFC 9700) §4.14

---

### 2-7. Spring Security OAuth2 내부 처리 흐름

```
/oauth2/authorization/google 요청
    ↓
OAuth2AuthorizationRequestRedirectFilter
    ↓
DefaultOAuth2AuthorizationRequestResolver
    - client_id, redirect_uri, scope 조합
    - state 생성 + 세션 저장
    - PKCE code_verifier/challenge 생성
    ↓
Google 로그인 화면으로 302 Redirect

─────────────────────────────────────────

/login/oauth2/code/google?code=...&state=... 콜백
    ↓
OAuth2LoginAuthenticationFilter
    ↓
state 검증
    ↓
OAuth2AuthorizationCodeAuthenticationProvider
    - code → token 교환 (Google Token 엔드포인트 호출)
    ↓
OidcAuthorizationCodeAuthenticationProvider (OIDC)
    - ID Token 서명 검증
    - 클레임 검증 (iss, aud, exp, nonce)
    ↓
OAuth2UserService.loadUser()
    - UserInfo 엔드포인트 호출 (필요 시)
    ↓
SecurityContext에 Authentication 저장
    ↓
successHandler → 최종 목적지로 Redirect
```

```java
// 실제 사용자 정보 접근
@GetMapping("/me")
public ResponseEntity<?> me(
        @AuthenticationPrincipal OidcUser oidcUser) {

    String sub = oidcUser.getSubject();       // Google 고유 ID
    String email = oidcUser.getEmail();
    String name = oidcUser.getFullName();

    // ID Token 원본도 접근 가능
    OidcIdToken idToken = oidcUser.getIdToken();
    Map<String, Object> claims = idToken.getClaims();
}
```

> 📎 **근거:** Spring Security 공식 레퍼런스 — _"OAuth 2.0 Login"_, docs.spring.io/spring-security/reference/servlet/oauth2/login

---

## 3. 공식 근거 정리

|주장|출처|
|---|---|
|OAuth 2.0 Authorization Code Flow|RFC 6749 §4.1|
|state CSRF 방어|RFC 6749 §10.12|
|PKCE 동작 방식|RFC 7636|
|ID Token 표준 클레임|OpenID Connect Core 1.0 §2|
|ID Token 검증 순서|OpenID Connect Core 1.0 §3.1.3.7|
|Refresh Token Rotation|RFC 9700 §4.14|
|JWKS 공개키 배포|RFC 7517, OpenID Connect Core §10.1|

---

## 4. 이 설계를 이렇게 한 이유

### Authorization Code를 한 번 더 교환하는 이유 — 그리고 대가

**이점:** code가 탈취되어도 client_secret 없이는 token 교환 불가. 브라우저에 token이 직접 노출되지 않음.

**대가:** 왕복이 한 번 더 생긴다. 브라우저 → 내 서버 → Google Token 엔드포인트 → 내 서버 → 브라우저. 지연(latency) 증가. Google Token 엔드포인트 장애 시 로그인 전체 불가.

### Access Token과 ID Token을 분리한 이유 — 그리고 잃는 것

**이점:** 관심사 분리. Access Token은 리소스 서버만 알면 된다. ID Token은 클라이언트(내 서버)가 사용자 신원을 파악하는 데만 쓴다. 두 토큰의 수명과 범위를 독립적으로 관리 가능.

**잃는 것:** 토큰이 두 종류가 되어 관리 복잡도 증가. ID Token을 API 호출에 쓰는 잘못된 구현이 실무에서 자주 등장한다.

```java
// 흔한 실수
// ID Token을 API Authorization 헤더에 넣는 경우
headers.set("Authorization", "Bearer " + idToken); // WRONG
// ID Token은 신원 증명용, 리소스 접근용이 아님

headers.set("Authorization", "Bearer " + accessToken); // RIGHT
```

---

## 5. 이어지는 개념

|순서|개념|이유|
|---|---|---|
|1|**JWT 키 로테이션 전략 (JWKS + kid)**|소셜 로그인에서 Google이 개인키를 교체할 때 내 서버는 어떻게 새 공개키를 받아야 하는가 — 캐싱 전략 잘못 짜면 서명 검증 실패로 전체 로그인이 깨진다|
|2|**Spring Authorization Server**|직접 OAuth 2.0 서버를 구축하는 경우 — Google이 하는 역할을 내 서버가 해야 할 때 (사내 SSO, B2B 서비스)|
|3|**세션 vs 토큰 기반 인증 트레이드오프**|소셜 로그인 후 내 서버가 발급하는 토큰을 어떻게 관리할 것인가 — Stateless JWT의 로그아웃 문제, Refresh Token 저장 위치(Redis vs DB)|
|4|**CORS와 쿠키 SameSite 정책**|프론트엔드가 분리된 환경(React + Spring)에서 OAuth 콜백과 토큰 전달 시 CORS/쿠키 설정이 맞지 않아 로그인이 실패하는 케이스 — 실무에서 가장 자주 만나는 문제|