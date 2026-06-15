# SSO에서 OIDC가 필요한 이유

SSO의 핵심 목적은 **"한 번 로그인으로 여러 서비스 이용"** 이야.

이걸 구현하려면 반드시 **"이 사람이 누구인지"** 를 증명할 수 있어야 해.

---

### OAuth2만으로 SSO를 구현하면?

```
서비스 A 로그인 → Access Token 발급
서비스 B에서 "이 사람 누구야?" 물어봄
→ Access Token으로 /userinfo API 호출해야 함
→ 서비스마다 매번 API 호출 필요
→ 구글 API 의존도 높음, 느림, 표준화 안 됨
```

OAuth2는 **인가(Authorization)** 프로토콜이라 신원 정보를 표준화된 방식으로 전달하는 기능이 없어.

---

### OIDC를 쓰면?

```
구글 로그인 한 번
→ ID Token (JWT) 발급
→ 토큰 안에 sub, email, name 포함
→ 서비스 A, B, C 모두 ID Token으로 신원 확인
→ 추가 API 호출 없음
→ 토큰 서명 검증만으로 신뢰 가능
```

**ID Token이 "신원 증명서" 역할**을 하기 때문에 SSO가 가능해지는 거야.

---

### 한 줄 요약

```
OAuth2  →  "뭘 할 수 있냐" (인가)  →  SSO 불가
OIDC    →  "누구냐"       (인증)  →  SSO 가능
```

SSO = 신원 확인이 전제 → 신원 확인 = OIDC의 역할. 그래서 SSO에 OIDC가 필수야.

---

## OIDC Authorization Code Flow (서버 플로우)


![[../../../../../Repository/OIDC-1.png]]
---

## 각 단계 핵심 포인트

**1단계 — Auth URL 구성 시 포함할 파라미터**

```
client_id     → 구글 클라이언트 ID
redirect_uri  → 콜백 엔드포인트
response_type → code
scope         → openid email profile   ← openid 필수!
nonce         → 랜덤값 (재전송 공격 방지)
```

**4단계 — ID Token 검증 체크리스트**

```
iss   → https://accounts.google.com 인지
aud   → 내 client_id 와 일치하는지
exp   → 만료 안 됐는지
nonce → 1단계에서 보낸 값과 일치하는지  ← OIDC 추가 검증
```

**sub 클레임 — 핵심 식별자**

```
sub = 구글이 부여한 사용자 고유 ID
→ OAuthAccount 테이블의 provider_user_id에 저장
→ 이메일은 바뀔 수 있지만 sub는 절대 안 바뀜
```

---

nonce가 OAuth2에 없는 OIDC 전용 보안 장치야. 
1단계에서 생성한 nonce를 세션/Redis에 저장해뒀다가, 
ID Token 검증 시 클레임의 nonce값과 비교해서 **중간자 공격, 토큰 재사용 공격**을 막는 거야.