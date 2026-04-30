## 1. 현재 구현 분석

### 1.1 OIDC 플로우와 구현 코드 매핑

### 전체 OIDC Authorization Code Flow

```
1. [프론트엔드] 카카오 Authorization URL로 리다이렉트
   ↓
2. [카카오] 사용자 인증 후 Authorization Code 발급
   ↓
3. [LoginController] Authorization Code 수신 및 처리
   ↓
4. [KakaoLoginClient] Authorization Code → ID Token 교환
   ↓
5. [JwtOIDProvider] ID Token JWT 서명 검증
   ↓
6. [OidcPublicKeyResolver] 카카오 공개키로 서명 검증
   ↓
7. [KakaoOidcPublicKeyResolver] JWKS에서 공개키 조회
   ↓
8. [KakaoJwksCache] 카카오 JWKS 캐싱 및 관리
   ↓
9. [LoginService] 사용자 로그인 처리
   ↓
10. [애플리케이션] JWT Access Token 발급

```

- **OIDC Authorization Code Flow가 필요한 이유**
    
    **✅ OIDC가 해결하는 방식:**
    
    ```
    사용자 ← 직접 통신 → 카카오 (비밀번호는 카카오만 알게 됨)
       ↓                    ↓
    우리 서버 ← 안전한 토큰 ← 카카오 (우리는 토큰만 받음)
    
    ```
    
    ## 1. 각 단계가 필요한 보안적 이유
    
    ### 1-2단계: 프론트엔드 리다이렉트 → Authorization Code
    
    **왜 이렇게 해야 하나?**
    
    ```jsx
    // 사용자를 카카오로 직접 보냄
    window.location.href = "<https://kauth.kakao.com/oauth/authorize?.>..";
    
    ```
    
    **보안 이유:**
    
    - 🔐 **비밀번호 격리**: 사용자 비밀번호가 우리 서버를 절대 거치지 않음
    - 🔐 **사용자 동의**: 카카오가 직접 "이 앱에 정보 제공할까요?" 물어봄
    - 🔐 **피싱 방지**: 진짜 카카오 도메인에서만 로그인 가능
    
    ### 3-4단계: Authorization Code → ID Token 교환
    
    **왜 Authorization Code를 한 번 더 교환해야 하나?**
    
    **보안 이유 1: 브라우저 노출 최소화**
    
    ```
    ❌ 만약 ID Token을 바로 브라우저로 보낸다면:
    <https://oursite.com/callback#access_token=eyJ0eXAiOiJKV1Q>...
    → 브라우저 히스토리, 로그, 레퍼러에 토큰 노출
    
    ✅ Authorization Code 방식:
    <https://oursite.com/callback?code=O2L6RVSC4X9B2T1A>...
    → 의미 없는 일회용 코드만 노출
    
    ```
    
    **보안 이유 2: 서버 간 직접 통신**
    
    ```java
    // 4단계: 우리 서버가 카카오 서버와 직접 통신
    POST <https://kauth.kakao.com/oauth/token>
    {
        "grant_type": "authorization_code",
        "client_id": "our_app_id",
        "client_secret": "our_secret", // 브라우저가 절대 모르는 비밀키
        "code": "O2L6RVSC4X9B2T1A..."
    }
    
    ```
    
    **왜 이게 안전한가?**
    
    - 🔐 **Client Secret 보호**: 브라우저는 절대 우리 앱의 비밀키를 모름
    - 🔐 **네트워크 도청 방지**: 서버 간 HTTPS 통신으로 안전
    - 🔐 **일회용 코드**: Authorization Code는 한 번만 사용 가능
    
    ### 5-8단계: JWT 서명 검증 과정
    
    **문제 상황: 만약 검증 없이 토큰을 믿는다면?**
    
    ```java
    ❌ 위험한 코드:
    String userInfo = idToken; // 그냥 믿고 사용
    // → 누구나 가짜 토큰을 만들어서 보낼 수 있음
    
    ```
    
    **JWT 서명 검증이 필요한 이유:**
    
    ```
    공격자가 가짜 토큰 생성:
    {
      "sub": "admin_user_id",
      "iss": "<https://kauth.kakao.com>",
      "aud": "our_app"
    }
    
    하지만 서명이 없거나 잘못된 서명
    → 우리가 카카오 공개키로 검증하면 즉시 탐지됨
    ```
    
    **5-6단계: 서명 검증**
    
    ```java
    // 카카오가 비밀키로 서명한 토큰인지 확인
    boolean isValid = JWT.require(algorithm)
        .build()
        .verify(idToken);
    ```
    
    **7-8단계: 공개키 관리**
    
    ```java
    // 카카오의 공개키를 안전하게 가져와서 캐싱
    // → 매번 네트워크 호출하지 않고 성능 최적화
    ```
    
    ## 3. 전체 과정의 신뢰 체인(Chain of Trust)
    
    ### 보안이 어떻게 연결되는가?
    
    ```
    1. 사용자 ←trust→ 카카오 도메인 (HTTPS 인증서로 검증)
       ↓
    2. 카카오 ←trust→ 우리 앱 (client_id/secret으로 검증)
       ↓
    3. 우리 앱 ←trust→ JWT 토큰 (카카오 공개키로 검증)
       ↓
    4. JWT 토큰 ←trust→ 사용자 정보 (암호학적 서명으로 검증)
    
    ```
    
    **각 단계에서 검증하는 것:**
    
    - 1단계: "진짜 카카오 사이트인가?" (SSL 인증서)
    - 2단계: "우리가 등록한 정당한 앱인가?" (Client Secret)
    - 3단계: "카카오가 실제로 서명한 토큰인가?" (공개키 검증)
    - 4단계: "토큰이 변조되지 않았는가?" (JWT 서명)
    
    ## 4. 왜 더 간단한 방법을 쓰면 안 되는가?
    
    ### 대안들과 그 문제점
    
    **🔴 대안 1: API 키 방식**
    
    ```
    사용자가 카카오 API 키를 우리에게 직접 제공
    ❌ 문제: API 키가 탈취되면 사용자 계정 완전 장악
    ❌ 문제: 권한 범위 제어 불가
    
    ```
    
    **🔴 대안 2: 단순 쿠키 기반 인증**
    
    ```
    사용자 정보를 암호화해서 쿠키에 저장
    ❌ 문제: XSS 공격으로 쿠키 탈취 가능
    ❌ 문제: 쿠키 크기 제한
    ❌ 문제: 모바일 앱에서 사용 어려움
    
    ```
    
    ## 5. 현실적인 장점들
    
    ### 개발자 관점
    
    ```java
    // OIDC 덕분에 우리가 하지 않아도 되는 것들:
    - 사용자 비밀번호 저장/관리 ❌
    - 비밀번호 암호화/해싱 ❌
    - 비밀번호 분실 처리 ❌
    - 2FA 구현 ❌
    - 계정 잠금 정책 ❌
    - 비밀번호 복잡도 정책 ❌
    ```
    
    ### 사용자 관점
    
    ```
    ✅ 새로운 비밀번호 만들 필요 없음
    ✅ 기존 카카오 계정으로 바로 로그인
    ✅ 비밀번호를 여러 사이트에 입력할 필요 없음
    ✅ 카카오의 강력한 보안 시스템 활용
    ```
    
    ### 비즈니스 관점
    
    ```
    ✅ 사용자 가입 장벽 낮춤 → 전환율 증가
    ✅ 보안 사고 책임 분산 → 리스크 감소
    ✅ 개발 비용 절약 → 핵심 기능에 집중
    ✅ 컴플라이언스 부담 감소
    ```
    

### 구현 코드별 상세 매핑

**1-2. OIDC Discovery & Authorization Request (프론트엔드)**

- **구현 위치**: 프론트엔드에서 처리
- **역할**: 카카오 Authorization Server로 리다이렉트
- **URL 형식**: `https://kauth.kakao.com/oauth/authorize?...`

**3. Authorization Code 수신 (LoginController)**

- **파일**: `LoginController.java`
- **메서드**: `processCode()`
- **역할**: 카카오에서 리다이렉트된 Authorization Code 처리
- **OIDC 표준**: Authorization Code Grant의 Callback 처리

**4. Token Exchange (KakaoLoginClient)**

- **파일**: `KakaoLoginClient.java`
- **메서드**: `getIdToken()`
- **역할**: Authorization Code를 ID Token으로 교환
- **OIDC 표준**: Token Endpoint 호출 (`/oauth/token`)

**5. ID Token 검증 (JwtOIDProvider)**

- **파일**: `JwtOIDProvider.java`
- **메서드**: `extractProviderIdFromIdToken()`
- **역할**: JWT 파싱, 서명 검증, Claims 추출
- **OIDC 표준**: ID Token Validation

**6-8. 공개키 관리 (JWKS)**

- **파일**: `KakaoOidcPublicKeyResolver.java`, `KakaoJwksCache.java`
- **역할**: 카카오 JWKS에서 공개키 조회 및 캐싱
- **OIDC 표준**: JWKS Endpoint를 통한 공개키 획득

**9-10. 사용자 인증 완료 (LoginService)**

- **파일**: `LoginService.java`
- **메서드**: `login()`
- **역할**: 검증된 사용자 정보로 애플리케이션 토큰 발급

## 2. 심각한 보안 취약점

단, 현재 보안은 중요 사항이 아니라 판단하여 간략화되었다는 배경을 염두할 것.

### 2.1 불완전한 ID Token 검증 ⚠️ **CRITICAL**

```java
// 오직 subject만 추출, 다른 중요 Claims 무시
Long providerId = Long.parseLong(signedJWT.getJWTClaimsSet().getSubject());
```

**실제 공격 시나리오:**

- **토큰 재사용 공격**: 만료된 토큰도 유효하게 처리 (`exp` 미검증)
- **Cross-Origin 토큰 탈취**: 다른 앱용 토큰으로 로그인 가능 (`aud` 미검증)
- **발급자 스푸핑**: 가짜 서버 토큰도 신뢰 (`iss` 미검증)
- **시간 조작 공격**: 미래 날짜 토큰도 허용 (`iat` 미검증)

**OIDC 표준 위반**: RFC 7519에서 **MUST**로 규정한 Claims 검증 완전 누락

### 2.2 CSRF 공격 취약성 ⚠️ **HIGH**

**현재 문제:**

```java
// State 매개변수 완전 누락
public TokenResponse processCode(AuthCodeRequest request) {
    // Authorization Code만 검증, CSRF 방어 없음
}

```

**실제 공격 시나리오:**

1. 공격자가 Authorization Code 탈취 또는 자신의 코드로 유도
2. 피해자가 공격자 계정으로 로그인됨
3. 피해자의 개인정보가 공격자 계정에 저장

**OAuth 2.1 위반**: RFC 9749에서 State를 **REQUIRED**로 규정

### 2.3 Rate Limiting 부재 ⚠️ **HIGH**

**현재 위험:**

- **Brute Force 공격**: 초당 수천번 로그인 시도 가능
- **DDoS 공격**: 카카오 API 호출 제한 초과로 서비스 차단
- **경제적 공격**: 클라우드 API 비용 폭증

**실제 피해 사례:**

- 정상 사용자 접근 차단
- 시스템 리소스 고갈
- 운영비 급증

### 2.4 웹 보안 취약점 ⚠️ **MEDIUM**

**누락된 보안 헤더들:**

```
X-Content-Type-Options: nosniff     → MIME 스니핑 공격 방지
X-Frame-Options: DENY               → Clickjacking 공격 방지
Strict-Transport-Security           → MITM 공격 방지
Content-Security-Policy             → XSS 공격 방지

```

- 어캐 구현해야 할까?
    
    ```java
    @Configuration
    @RequiredArgsConstructor
    public class WebConfig implements WebMvcConfigurer {
    
        @Override
        public void addCorsMappings(CorsRegistry registry) {
            registry.addMapping("/**")
                    .allowedOrigins("<http://localhost:3000>", "<https://pickeat.io.kr>", "<https://api.pickeat.io.kr>")
                    .allowedMethods("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS")
                    .allowedHeaders("*")
                    .allowCredentials(true)
                    .maxAge(3600);
        }
    
        @Override
        public void addInterceptors(InterceptorRegistry registry) {
            registry.addInterceptor(new SecurityHeaderInterceptor());
        }
    
        /**
         * 보안 헤더를 추가하는 인터셉터
         */
        public static class SecurityHeaderInterceptor implements HandlerInterceptor {
    
            @Override
            public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
                // MIME 스니핑 공격 방지
                response.setHeader("X-Content-Type-Options", "nosniff");
    
                // Clickjacking 공격 방지 (iframe 삽입 차단)
                response.setHeader("X-Frame-Options", "DENY");
    
                // HTTPS 강제 사용 (1년간 캐시)
                response.setHeader("Strict-Transport-Security", "max-age=31536000; includeSubDomains");
    
                // XSS 공격 방지 - 기본 CSP 설정
                response.setHeader("Content-Security-Policy",
                    "default-src 'self'; " +
                    "script-src 'self' 'unsafe-inline' <https://developers.kakao.com>; " +
                    "style-src 'self' 'unsafe-inline'; " +
                    "img-src 'self' data: https:; " +
                    "connect-src 'self' <https://kauth.kakao.com> <https://kapi.kakao.com>; " +
                    "frame-ancestors 'none'; " +
                    "base-uri 'self';"
                );
    
                // XSS 필터 활성화 (구형 브라우저 호환)
                response.setHeader("X-XSS-Protection", "1; mode=block");
    
                // 레퍼러 정보 제한 (개인정보 보호)
                response.setHeader("Referrer-Policy", "strict-origin-when-cross-origin");
    
                // 권한 정책 설정 (불필요한 브라우저 기능 차단)
                response.setHeader("Permissions-Policy",
                    "geolocation=(), " +
                    "microphone=(), " +
                    "camera=(), " +
                    "payment=(), " +
                    "usb=(), " +
                    "magnetometer=(), " +
                    "gyroscope=(), " +
                    "speaker=()"
                );
    
                return true;
            }
        }
    }
    
    ```
    
    ## 보안 헤더 설명
    
    ### 🔐 핵심 보안 헤더들
    
    **1. X-Content-Type-Options: nosniff**
    
    ```
    공격 방지: MIME 스니핑 공격
    - 브라우저가 Content-Type을 임의로 추측하는 것을 차단
    - 이미지 파일로 위장한 JavaScript 실행 방지
    
    ```
    
    **2. X-Frame-Options: DENY**
    
    ```
    공격 방지: Clickjacking 공격
    - 우리 사이트가 다른 사이트의 iframe에 삽입되는 것을 차단
    - 투명한 iframe으로 사용자 클릭을 가로채는 공격 방지
    
    ```
    
    **3. Strict-Transport-Security**
    
    ```
    공격 방지: MITM(중간자) 공격
    - 브라우저가 HTTPS만 사용하도록 강제
    - HTTP로 다운그레이드하는 공격 차단
    
    ```
    
    **4. Content-Security-Policy**
    
    ```
    공격 방지: XSS(Cross-Site Scripting) 공격
    - 허용된 리소스만 로드하도록 제한
    - 인라인 스크립트 실행 제한
    - 카카오 관련 도메인만 허용
    
    ```
    
    ### 🔧 추가 보안 헤더들
    
    **5. X-XSS-Protection**
    
    ```
    구형 브라우저의 XSS 필터 활성화
    최신 브라우저는 CSP를 사용하지만 호환성을 위해 추가
    
    ```
    
    **6. Referrer-Policy**
    
    ```
    다른 사이트로 이동 시 레퍼러 정보 제한
    개인정보가 포함된 URL 파라미터 노출 방지
    
    ```
    
    **7. Permissions-Policy**
    
    ```
    불필요한 브라우저 API 접근 차단
    - 지오로케이션, 카메라, 마이크 등 민감한 기능 차단
    - 공격 표면 최소화
    
    ```
    
    ## 더 나은 방법: Spring Security 사용
    
    `@EnableWebSecurity`만으로도 **부분적으로** 해결됩니다.
    
    ## 🟡 Spring Security 기본 제공 헤더들
    
    Spring Security는 기본적으로 다음 보안 헤더들을 자동으로 추가합니다:
    
    ```
    ✅ Cache-Control: no-cache, no-store, max-age=0, must-revalidate
    ✅ Pragma: no-cache
    ✅ Expires: 0
    ✅ X-Content-Type-Options: nosniff
    ✅ Strict-Transport-Security: max-age=31536000; includeSubDomains
    ✅ X-Frame-Options: DENY
    ✅ X-XSS-Protection: 1; mode=block
    
    ```
    
    ## 🔴 여전히 누락되는 중요한 헤더들
    
    ```java
    🟡 필수: Content-Security-Policy       → XSS 공격 방지의 핵심
    🟡 권장: Referrer-Policy               → 개인정보 보호
    🟢 선택: Permissions-Policy            → 브라우저 API 접근 제한
    🟢 선택: Cross-Origin-Embedder-Policy  → 격리 보안
    🟢 선택: Cross-Origin-Opener-Policy    → 윈도우 간 격리
    ```
    
    ## 완전한 보안을 위한 설정
    
    ```java
    @Configuration
    @EnableWebSecurity
    public class SecurityConfig {
    
       @Bean
       public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
           return http
               // 📌 핵심만 설정: CSP + 카카오 OAuth 허용
               .headers(headers -> headers
                   .contentSecurityPolicy(
                       "default-src 'self'; " +
                       "script-src 'self' <https://developers.kakao.com>; " +
                       "connect-src 'self' <https://kauth.kakao.com> <https://kapi.kakao.com>; " +
                       "img-src 'self' data: https:"
                   )
                   // 나머지는 Spring Security 기본값 사용
               )
               .authorizeHttpRequests(auth -> auth.anyRequest().permitAll())
               .csrf(csrf -> csrf.disable())
               .build();
       }
    }
    ```
    

## 3. 설정 관리의 경직성

**문제:**

```java
// 하드코딩된 설정값들
private final Duration ttlCacheDuration = Duration.ofMinutes(15);
private final String jwksUrl = "<https://kauth.kakao.com/.well-known/jwks.json>";

```

**운영상 문제:**

- 카카오 엔드포인트 변경 시 코드 수정 + 재배포 필요
- 환경별(dev/staging/prod) 설정 구분 불가
- 긴급 상황 시 실시간 설정 조정 불가

---

---

---

## ID Token 검증 누락으로 인한 구체적인 공격 플로우

### 1. 토큰 재사용 공격 (`exp` 미검증)

**공격 플로우:**

```
1. 공격자가 정상적으로 카카오 로그인
   ↓
2. ID Token 획득 (예: exp: 2024-01-01T00:00:00Z)
   ↓
3. 토큰을 안전한 곳에 저장
   ↓
4. 시간이 지나 토큰 만료 (현재: 2024-06-01)
   ↓
5. 공격자가 만료된 토큰을 우리 서버에 재전송
   ↓
6. 서버에서 exp 검증 없이 토큰 허용
   ↓
7. 공격자가 계속 로그인 성공 ✅

```

**실제 코드 동작:**

```java
// 현재 코드: exp 검증 없음
Long providerId = Long.parseLong(signedJWT.getJWTClaimsSet().getSubject());
// → 만료 여부 무관하게 subject만 추출하여 로그인 허용

```

**실제 위험:**

- 탈취된 토큰으로 **영구적 접근** 가능
- 직원 퇴사 후에도 토큰 재사용 가능
- 토큰 유출 시 피해 범위 무제한

---

### 2. Cross-Origin 토큰 탈취 (`aud` 미검증)

**공격 플로우:**

```
1. 공격자가 다른 카카오 연동 앱에서 로그인
   예: 카카오페이 앱 (aud: "kakao-pay-client-id")
   ↓
2. 해당 앱용 ID Token 획득
   ↓
3. 공격자가 탈취한 토큰을 우리 서버에 전송
   ↓
4. 서버에서 aud 검증 없이 토큰 허용
   ↓
5. 다른 앱용 토큰으로 우리 서비스 로그인 성공 ✅

```

**토큰 구조 예시:**

```json
// 카카오페이 앱용 토큰
{
  "iss": "<https://kauth.kakao.com>",
  "aud": "kakao-pay-client-id",
  "sub": "1234567890",
  "exp": 1640995200
}

// 우리 앱용이어야 할 토큰
{
  "iss": "<https://kauth.kakao.com>",
  "aud": "our-app-client-id",  ← 이것을 검증해야 함
  "sub": "1234567890",
  "exp": 1640995200
}

```

**현재 코드 문제:**

```java
// aud 검증 없이 subject만 사용
Long providerId = Long.parseLong(signedJWT.getJWTClaimsSet().getSubject());
// → 어떤 앱용 토큰이든 같은 사용자라면 로그인 허용

```

**실제 공격 시나리오:**

- 공격자가 카카오톡, 카카오페이 등에서 발급받은 토큰 재사용
- 취약한 다른 카카오 연동 앱에서 토큰 탈취 후 우리 서비스 침투

---

### 3. 발급자 스푸핑 (`iss` 미검증)

**공격 플로우:**

```
1. 공격자가 가짜 OAuth 서버 구축
   도메인: evil-kakao.com
   ↓
2. 가짜 서버에서 위조 토큰 발급
   {
     "iss": "<https://evil-kakao.com>",
     "sub": "admin-user-id",
     "aud": "our-app-client-id"
   }
   ↓
3. 공격자가 위조 토큰을 우리 서버에 전송
   ↓
4. 서버에서 iss 검증 없이 토큰 허용
   ↓
5. 관리자 권한으로 로그인 성공 ✅

```

**더 정교한 공격:**

```
1. 공격자가 DNS 스푸핑으로 kauth.kakao.com 가로채기
   ↓
2. 가짜 서버에서 정상적인 형태의 토큰 발급
   ↓
3. 실제 카카오인 것처럼 위장하여 토큰 전송
   ↓
4. 서버에서 발급자 검증 없이 신뢰

```

**현재 코드 문제:**

```java
// 발급자가 누구든 상관없이 처리
Long providerId = Long.parseLong(signedJWT.getJWTClaimsSet().getSubject());
// → 카카오가 아닌 서버에서 발급한 토큰도 신뢰

```

---

### 4. 시간 조작 공격 (`iat` 미검증)

**공격 플로우:**

```
1. 공격자가 미래 날짜로 시스템 시간 설정
   예: 2030-01-01
   ↓
2. 해당 시간으로 토큰 생성 또는 조작
   {
     "iat": 1893456000,  // 2030-01-01
     "exp": 1893459600,  // 2030-01-01 + 1시간
     "sub": "victim-id"
   }
   ↓
3. 시스템 시간을 현재로 되돌림
   ↓
4. 미래에서 온 토큰을 서버에 전송
   ↓
5. 서버에서 iat 검증 없이 토큰 허용 ✅

```

**클라이언트 시간 조작 공격:**

```
1. 사용자 브라우저/앱에서 시간 조작
   ↓
2. 조작된 시간 기반으로 토큰 요청
   ↓
3. 비정상적인 타임스탬프의 토큰 획득
   ↓
4. 서버에서 시간 검증 없이 허용

```

**현재 코드 문제:**

```java
// 토큰 발급 시간을 전혀 검증하지 않음
Long providerId = Long.parseLong(signedJWT.getJWTClaimsSet().getSubject());
// → 언제 발급된 토큰이든 허용

```

---

### 5. 종합적인 공격 시나리오

**실제 해커가 사용할 수 있는 복합 공격:**

```
1. 다른 카카오 앱에서 토큰 탈취 (aud 검증 없음 이용)
   ↓
2. 탈취한 토큰의 만료시간 연장 (exp 검증 없음 이용)
   ↓
3. 가짜 발급자로 토큰 재생성 (iss 검증 없음 이용)
   ↓
4. 시간 조작으로 토큰 유효성 연장 (iat 검증 없음 이용)
   ↓
5. 우리 서비스에 무제한 접근 가능

```

**피해 규모:**

- 전체 사용자 계정 접근 가능
- 개인정보 대량 유출
- 서비스 신뢰도 완전 붕괴
- 법적 책임 및 과징금

이러한 공격들이 가능한 이유는 **현재 코드가 JWT의 핵심 보안 Claims들을 전혀 검증하지 않기 때문**입니다. 단순히 `subject`만 추출하여 사용자를 식별하는 것은 보안상 매우 위험한 접근 방식입니다.