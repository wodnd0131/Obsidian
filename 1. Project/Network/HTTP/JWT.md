#Network/HTTP/Security #Network/HTTP #Network/JWT
# JWT (JSON Web Token)

## 1. Problem First

### 서버가 "이 요청이 인증된 사용자인가"를 어떻게 확인하는가

세션 방식의 문제를 다시 짚는다.

```
[클라이언트] ──요청 + session_id──▶ [서버 1]
                                        │
                                   Redis 조회
                                   "abc123 → userId:1, role:ADMIN"
                                        │
                                   응답 반환
```

이 구조는 **모든 요청마다 Redis 조회가 필수**다. 초당 10만 요청이 들어오면 Redis도 초당 10만 쿼리를 처리해야 한다. Redis가 죽으면 전체 인증이 불가능해진다.

근본 질문은 이것이다.

> **"매번 저장소를 조회하지 않고, 요청 자체만 보고 인증 여부를 판단할 수 있는가?"**

이 질문의 답이 JWT다.

### 핵심 아이디어

```
세션 방식: "이 열쇠(session_id)의 주인이 누구인지 금고(저장소)에서 찾아본다"

JWT 방식:  "이 신분증(JWT) 자체에 신원 정보가 있고,
            위조 여부는 서명만 검증하면 된다 — 저장소 조회 없음"
```

신분증이 위조되지 않았음을 확인하는 방법이 **서명(Signature)** 이다.

---

## 2. Mechanics

### 2-1. JWT의 물리적 구조

JWT는 `.`으로 구분된 세 부분의 문자열이다.

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
.
eyJ1c2VySWQiOjEsInJvbGUiOiJVU0VSIiwiaWF0IjoxNzE0MTc2MDAwLCJleHAiOjE3MTQxNzY5MDB9
.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

[        Header (Base64url)        ] . [              Payload (Base64url)               ] . [ Signature ]
```

각 부분을 디코딩하면:

```json
// Header
{
  "alg": "HS256",   // 서명 알고리즘
  "typ": "JWT"      // 토큰 타입
}

// Payload
{
  "userId": 1,
  "role": "USER",
  "iat": 1714176000,   // issued at (발급 시각)
  "exp": 1714176900    // expiration (만료 시각)
}

// Signature
HMACSHA256(
  base64url(Header) + "." + base64url(Payload),
  서버의_비밀키
)
```

**Base64url은 암호화가 아니다.** 누구나 Header와 Payload를 디코딩해 내용을 읽을 수 있다. JWT가 보장하는 것은 **내용의 기밀성이 아니라 무결성**이다.

> RFC 7519, Section 3: _"The claims in a JWT are encoded as a JSON object that is used as the payload of a JSON Web Signature (JWS) structure or as the plaintext of a JSON Web Encryption (JWE) structure."_ (JWT의 클레임은 JWS 구조의 페이로드 또는 JWE 구조의 평문으로 사용되는 JSON 객체로 인코딩된다. — 암호화는 JWE를 써야 하고, 기본 JWT는 JWS 즉 서명만)

---

### 2-2. 서명 — 위조를 막는 핵심 메커니즘

서명이 어떻게 위조를 막는지 구체적으로 본다.

**서명 생성 (발급 시):**

```
Signature = HMACSHA256(
    base64url({"alg":"HS256","typ":"JWT"})
    + "."
    + base64url({"userId":1,"role":"USER","exp":...}),
    "서버만_아는_비밀키_절대_노출_금지"
)
```

**서명 검증 (요청 수신 시):**

```
클라이언트가 보낸 JWT:
  Header.Payload.Signature_received

서버가 직접 계산:
  Signature_expected = HMACSHA256(Header + "." + Payload, 비밀키)

검증:
  Signature_received == Signature_expected ?
  → 같으면: 이 토큰은 서버가 발급했고 변조되지 않음 ✓
  → 다르면: 위조된 토큰 ✗
```

**공격자가 Payload를 변조하면 어떻게 되는가:**

```
원본 Payload: {"userId":1, "role":"USER"}
변조 시도:    {"userId":1, "role":"ADMIN"}  ← role을 바꾸려는 시도

변조된 Payload로 Signature를 다시 계산하려면 비밀키가 필요
→ 비밀키를 모르면 올바른 Signature 생성 불가
→ 서버 검증에서 Signature 불일치 → 거부
```

비밀키를 서버만 알고 있는 한, Payload 내용을 바꾸면 Signature가 맞지 않아 반드시 탐지된다.

---

### 2-3. 서명 알고리즘 두 종류

JWT 서명에는 두 가지 방식이 있다.

**HMAC 방식 (HS256, HS512) — 대칭키**

```
발급:  Signature = HMAC(Header.Payload, 비밀키)
검증:  HMAC(Header.Payload, 비밀키) == Signature ?

비밀키 하나로 서명과 검증 모두 수행
→ 서명 서버와 검증 서버가 같은 키를 공유해야 함
→ 마이크로서비스에서 여러 서버가 검증해야 하면 키를 모두 나눠가져야 함 = 키 노출 위험 증가
```

**RSA/ECDSA 방식 (RS256, ES256) — 비대칭키**

```
발급:  Signature = RSA(Header.Payload, 개인키)   ← 인증 서버만 보유
검증:  RSA_verify(Header.Payload, Signature, 공개키)  ← 누구나 가져갈 수 있음

개인키: 토큰 발급 서버만 보유 (유출되면 안 됨)
공개키: 검증이 필요한 모든 서버에 배포 가능 (유출돼도 무방)
```

```
MSA 환경에서의 차이:

HS256:
인증서버 ──비밀키 공유──▶ 서비스A
         ──비밀키 공유──▶ 서비스B   ← 비밀키가 여러 곳에 퍼짐
         ──비밀키 공유──▶ 서비스C

RS256:
인증서버 (개인키 보유)
         ──공개키 배포──▶ 서비스A   ← 공개키는 노출돼도 무방
         ──공개키 배포──▶ 서비스B
         ──공개키 배포──▶ 서비스C
```

MSA 환경에서 여러 서비스가 토큰을 검증해야 한다면 **RS256이 더 안전한 구조**다.

---

### 2-4. Payload의 클레임 — 무엇을 담는가

JWT Payload 안의 각 항목을 **클레임(Claim)** 이라 한다.

**RFC 7519가 정의한 표준 클레임:**

| 클레임   | 이름         | 용도                                |
| ----- | ---------- | --------------------------------- |
| `iss` | Issuer     | 발급자 (예: "auth.myservice.com")     |
| `sub` | Subject    | 토큰 주체 (예: userId)                 |
| `aud` | Audience   | 토큰 수신 대상 (예: "api.myservice.com") |
| `exp` | Expiration | 만료 시각 (Unix timestamp)            |
| `iat` | Issued At  | 발급 시각                             |
| `jti` | JWT ID     | 토큰 고유 ID (재사용 방지용)                |

**커스텀 클레임 — 실무에서 자주 쓰는 패턴:**

```json
{
  "sub": "1",
  "role": "ADMIN",
  "email": "kim@company.com",
  "iat": 1714176000,
  "exp": 1714176900
}
```

**클레임에 담으면 안 되는 것:**

```
JWT는 Base64url 디코딩으로 누구나 읽을 수 있음
→ 비밀번호, 카드번호, 개인정보 절대 금지
→ 인증/인가에 필요한 최소한의 정보만
```

---

### 2-5. 검증 시 서버가 확인하는 것들

서버가 JWT를 받으면 순서대로 검증한다.

```java
// Spring에서 JWT 검증 흐름 (jjwt 라이브러리 기준)
try {
    Claims claims = Jwts.parserBuilder()
        .setSigningKey(secretKey)
        .build()
        .parseClaimsJws(token)  // 1. 서명 검증
        .getBody();

    // 2. 만료 시각 확인 (exp < 현재시각이면 ExpiredJwtException)
    // 3. aud, iss 등 클레임 검증 (필요 시)
    
    Long userId = claims.get("userId", Long.class);
    String role = claims.get("role", String.class);

} catch (ExpiredJwtException e) {
    // 만료된 토큰
} catch (JwtException e) {
    // 서명 불일치, 형식 오류 등
}
```

---

### 2-6. JWT의 구조적 한계 — 무효화 문제

JWT의 가장 큰 약점은 **발급한 토큰을 만료 전에 무효화할 수 없다**는 것이다.

```
상황: 사용자가 로그아웃, 또는 계정이 탈취당해 강제 로그아웃 필요

세션 방식:
Redis에서 session_id 삭제 → 즉시 무효화

JWT 방식:
서버에 저장된 것이 없음
→ 토큰 자체의 exp가 지나기 전까지 유효
→ 탈취된 토큰이 만료 전까지 사용 가능
```

이를 해결하는 방법들과 그 트레이드오프:

```
방법 1: Access Token 만료를 짧게 (15분)
트레이드오프: 15분마다 재발급 필요 → Refresh Token 패턴 도입 필요

방법 2: 블랙리스트 (Redis에 무효화된 jti 저장)
트레이드오프: 모든 요청마다 Redis 조회 → JWT 도입 이유(저장소 조회 제거)가 희석됨

방법 3: 버전 관리 (DB에 사용자별 token_version 저장)
트레이드오프: 역시 요청마다 DB 조회 필요
```

완전한 해결책은 없다. **어느 정도의 무효화 지연을 감수할 것인가**가 설계 결정이다.

> _"There is no way to invalidate a JWT before its expiration time without maintaining some state on the server."_ (서버에 어떤 상태를 유지하지 않고는 JWT를 만료 전에 무효화할 방법이 없다.) — 이 부분은 공식 RFC에 명시되지 않으며, 커뮤니티 관행 수준의 정설로 인정되는 내용이다.

---

### 2-7. Refresh Token 패턴 — 짧은 만료의 UX 문제 해결

```
Access Token:  만료 짧게 (15분) — 탈취 시 피해 최소화
Refresh Token: 만료 길게 (7~30일) — 재발급용, 엄격하게 관리
```

```
정상 흐름:

[클라이언트] ──API 요청 + Access Token──▶ [서버]
                                              │ 서명 검증만
                                              │ (저장소 조회 없음)
                                           응답

Access Token 만료 후:

[클라이언트] ──API 요청 + 만료된 Access Token──▶ [서버]
                                                    │ 401 반환

[클라이언트] ──/auth/refresh + Refresh Token──▶ [인증 서버]
                                                    │ Refresh Token 검증
                                                    │ (DB/Redis 조회)
                                                    │ 새 Access Token 발급
[클라이언트] ◀──새 Access Token────────────────────

[클라이언트] ──API 요청 + 새 Access Token──▶ [서버]
                                               응답
```

Refresh Token은 저장소 조회가 필요하다. 하지만 API 요청마다가 아니라 **Access Token 만료 시에만** 조회하므로 빈도가 훨씬 낮다.

---

## 3. 공식 근거

|주장|출처|
|---|---|
|JWT 구조 (Header.Payload.Signature)|RFC 7519, Section 3|
|표준 클레임 (iss, sub, exp 등) 정의|RFC 7519, Section 4.1|
|JWS (서명) vs JWE (암호화) 구분|RFC 7515 (JWS), RFC 7516 (JWE)|
|HMAC 서명 알고리즘|RFC 7518, Section 3.2|
|RSA 서명 알고리즘|RFC 7518, Section 3.3|
|Base64url 인코딩 정의|RFC 4648, Section 5|

---

## 4. 이 설계를 이렇게 한 이유

### 얻는 것

- 저장소 조회 없이 서명 검증만으로 인증 완료 → 서버 확장이 자유로움
- 인증 서버와 리소스 서버를 분리 가능 (RS256 사용 시)
- 클레임에 필요한 정보를 담아 추가 조회 감소

### 잃는 것

|손실|구체적 상황|
|---|---|
|**즉각적 무효화 불가**|탈취된 토큰이 exp 전까지 유효. 블랙리스트로 해결하면 저장소 조회가 다시 생김|
|**토큰 크기**|세션 ID(수십 바이트) vs JWT(수백 바이트). 모든 요청 헤더에 포함되므로 트래픽 증가|
|**비밀키 유출 시 전체 위협**|HMAC 비밀키가 유출되면 공격자가 임의의 유효한 토큰을 무한 생성 가능|
|**Payload 기밀성 없음**|Base64url 디코딩으로 누구나 읽음. 민감 정보를 담으면 안 됨|

---

## 5. 이어지는 개념

|순서|개념|이유|
|---|---|---|
|**1**|**Spring Security JWT 필터 구현** (`OncePerRequestFilter`)|JWT 개념을 실제 Spring 코드로 연결하는 단계. Security Filter Chain에서 JWT가 어느 위치에서 검증되는지|
|**2**|**OAuth 2.0**|JWT는 토큰 포맷이고, OAuth 2.0은 토큰을 발급하는 프로토콜이다. "카카오 로그인", "구글 로그인"이 JWT와 어떻게 연결되는지 이해하려면 선행 필요|
|**3**|**JWKS (JSON Web Key Set)**|RS256을 쓸 때 공개키를 어떻게 배포하는가. 인증 서버가 공개키를 엔드포인트로 제공하고, 리소스 서버가 이를 주기적으로 가져오는 패턴|

---
# Refresh Token은 Access Token보다 훨씬 위험하다.

- 수명이 길다 (7~30일)
- 탈취되면 Access Token을 계속 재발급받을 수 있다
- 즉, Refresh Token 탈취 = 장기간 계정 장악

그러면서도 클라이언트에 저장해야 한다. 이 딜레마를 어떻게 다루는지 정리한다.

---

## 1. 저장 위치 — HttpOnly 쿠키가 사실상 표준

앞서 저장 방식 편에서 다뤘지만, Refresh Token 관점에서 다시 본다.

```
Access Token  → 메모리 (수명 짧으니 새로고침 시 소멸해도 감수)
Refresh Token → HttpOnly 쿠키 (JS 접근 자체를 차단)
```

HttpOnly 쿠키는 JS에서 읽을 수 없다.

```javascript
// XSS로 심어진 공격 코드
document.cookie;           // refresh_token이 보이지 않음
localStorage.getItem(...); // 여기도 없음

// 탈취 불가
```

브라우저가 HttpOnly 쿠키를 JS 엔진에 아예 노출하지 않기 때문이다. XSS 공격의 가장 현실적인 위협을 차단한다.

추가 속성을 모두 걸면:

```
Set-Cookie: refresh_token=eyJ...;
  HttpOnly;           // JS 접근 차단
  Secure;             // HTTPS에서만 전송
  SameSite=Strict;    // 다른 사이트 요청엔 자동 첨부 안 함
  Path=/auth/refresh; // 재발급 엔드포인트에만 전송
  Max-Age=604800;     // 7일
```

`Path=/auth/refresh` 가 중요하다. 쿠키는 기본적으로 같은 도메인의 모든 요청에 첨부된다. Path를 제한하면 `/auth/refresh` 로 가는 요청에만 Refresh Token이 실린다. 다른 API 요청에는 아예 포함되지 않아 노출 면적이 줄어든다.

---

## 2. 서버 측 관리 — Refresh Token을 저장소에 보관

여기서 중요한 전환이 있다.

Access Token은 서버에 저장하지 않는다. 서명 검증만으로 충분하기 때문이다.

**Refresh Token은 서버에도 저장한다.**

```
발급 시:
[인증 서버] → Refresh Token 생성
           → 클라이언트: HttpOnly 쿠키로 전달
           → DB/Redis: 저장 (userId와 매핑)

검증 시:
[클라이언트] → /auth/refresh 요청 + Refresh Token
[인증 서버]  → 쿠키의 토큰값을 DB/Redis에서 조회
             → 존재하고 유효하면 → Access Token 재발급
             → 없거나 폐기됐으면 → 401
```

서버에 저장하기 때문에 **즉각 무효화가 가능**하다. 이것이 Access Token과의 핵심 차이다.

```
로그아웃:     DB에서 해당 Refresh Token 삭제 → 즉시 무효화
계정 탈취 의심: DB에서 해당 userId의 모든 Refresh Token 삭제
               → 모든 기기에서 강제 로그아웃
```

---

## 3. Refresh Token Rotation — 탈취 감지

저장소에 보관하는 것만으로는 부족하다. 탈취된 Refresh Token이 사용되는 걸 어떻게 감지하는가.

**Refresh Token Rotation** 이 그 답이다.

> 재발급 요청이 올 때마다 Refresh Token도 새것으로 교체한다. 이전 토큰은 즉시 폐기.

```
정상 흐름:

클라이언트: refresh_token=A 로 재발급 요청
서버: A 검증 → Access Token 발급 + 새 refresh_token=B 발급
     DB에서 A 삭제, B 저장
클라이언트: B를 쿠키에 저장

다음 재발급:
클라이언트: refresh_token=B 로 재발급 요청
서버: B 검증 → Access Token 발급 + 새 refresh_token=C 발급
     ...
```

**탈취 감지 흐름:**

```
공격자가 refresh_token=A 탈취

정상 클라이언트: A로 재발급 → 서버가 A를 B로 교체
                              DB: A 삭제, B 저장

공격자: 탈취한 A로 재발급 시도
서버: A를 DB에서 조회 → 없음 (이미 폐기됨)
    → 이건 재사용 공격이다
    → 해당 userId의 모든 Refresh Token 폐기
    → 정상 사용자 포함 전체 로그아웃
    → 사용자가 재로그인 시 탈취 사실을 인지할 수 있음
```

정상 사용자와 공격자 중 누가 먼저 토큰을 쓰느냐의 싸움이 된다. 어느 쪽이 A를 먼저 쓰든, 나머지 한 쪽이 A를 다시 쓰려 할 때 서버가 감지한다.

> RFC 6749 (OAuth 2.0), Section 10.4: _"The authorization server MUST maintain the binding between a refresh token and the client to whom it was issued."_ (인가 서버는 Refresh Token과 그것이 발급된 클라이언트 간의 바인딩을 반드시 유지해야 한다.)

Rotation 자체는 RFC에 명시된 개념은 아니고, RFC 6819 (OAuth 2.0 위협 모델)에서 권장하는 보안 관행이다.

---

## 4. 전체 구조 요약

```
[브라우저 메모리]                [HttpOnly 쿠키]          [서버 DB/Redis]
Access Token (15분)      Refresh Token (7일)        Refresh Token 목록
      │                          │                          │
      │ API 요청마다 헤더에 포함   │ /auth/refresh 시에만     │ 재발급 시 검증
      │ 서명 검증만으로 처리       │ 자동 첨부                │ + Rotation
      │                          │                          │
      └── 만료 시 ───────────────▶ 재발급 요청 ─────────────▶ 검증 후 교체
```

---

## 결론

Refresh Token 보안은 한 가지로 해결되지 않는다. 레이어를 겹쳐서 방어한다.

|레이어|수단|막는 것|
|---|---|---|
|저장 위치|HttpOnly 쿠키|XSS로 인한 JS 탈취|
|전송 제한|Secure + Path 제한|불필요한 노출 최소화|
|서버 저장|DB/Redis 보관|즉각 무효화 가능|
|Rotation|재발급 시 토큰 교체|탈취 감지 + 재사용 차단|

그리고 이 모든 것을 해도 **물리적으로 브라우저에 저장되는 이상 완벽한 보안은 없다.** 기기 자체가 탈취되거나 브라우저 취약점이 있으면 HttpOnly 쿠키도 위험하다. 보안 설계는 "완벽하게 막는다"가 아니라 **"공격 비용을 높이고, 탈취 시 피해를 최소화하고, 감지 가능하게 만든다"** 는 방향으로 접근하는 것이다.