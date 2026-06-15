### 1. 프로젝트 생성

[console.cloud.google.com](https://console.cloud.google.com/) 접속 후

```
상단 프로젝트 선택 → 새 프로젝트 → 프로젝트 이름 입력 (예: swiipe-backend)
```

---

### 2. OAuth 동의 화면 설정

**경로:** `API 및 서비스 → OAuth 동의 화면`

|항목|설정값|설명|
|---|---|---|
|**User Type**|외부 (External)|일반 사용자 대상|
|**앱 이름**|Swiipe|사용자에게 보이는 이름|
|**사용자 지원 이메일**|팀 이메일|문의용|
|**개발자 연락처**|팀 이메일|구글이 연락할 이메일|

**Scope 추가 (중요)**

```
openid       → 필수 (OIDC 사용 선언)
email        → 이메일 주소
profile      → 이름, 프로필 사진
```

> 민감한 scope (예: Gmail 읽기 등) 추가 시 구글 검수 필요. `openid / email / profile` 은 검수 불필요.

**게시 상태**

- 개발 중 → **테스트 모드** (테스트 사용자만 로그인 가능, 최대 100명)
- 출시 시 → **프로덕션으로 게시** (검수 필요 없음, 위 3개 scope만 쓴다면)

---

### 3. OAuth 2.0 클라이언트 ID 생성

**경로:** `API 및 서비스 → 사용자 인증 정보 → 사용자 인증 정보 만들기 → OAuth 클라이언트 ID`

|항목|설정값|
|---|---|
|**애플리케이션 유형**|웹 애플리케이션|
|**이름**|swiipe-web (구분용)|

**승인된 JavaScript 원본** (CORS 허용 출처)

```
# 로컬 개발
http://localhost:3000
http://localhost:8080

# 프로덕션
https://swiipe.com
```

**승인된 리디렉션 URI** ← 가장 중요

```
# 로컬 개발
http://localhost:8080/oauth2/google/callback

# 프로덕션
https://api.swiipe.com/oauth2/google/callback
```

> 코드에서 `redirect_uri` 파라미터로 넘기는 값이 **여기 등록된 값과 1글자도 달라선 안 돼.** 슬래시 하나, http/https 차이도 에러남.

생성 후 **클라이언트 ID / 클라이언트 보안 비밀** 발급됨 → `application.yml`의 환경변수로 주입.

---

### 4. 환경별 관리 전략

클라이언트 ID를 **환경별로 분리**해서 만드는 게 좋아.

```
swiipe-web-local    → localhost redirect URI
swiipe-web-dev      → dev 서버 redirect URI  
swiipe-web-prod     → 프로덕션 redirect URI
```

각각 다른 `client-id / client-secret` 발급 → `.env` 또는 AWS Secrets Manager로 주입.

---

### 5. 체크리스트 요약

```
□ Google Cloud 프로젝트 생성
□ OAuth 동의 화면 설정 (앱 이름, 이메일, scope)
□ 테스트 사용자 등록 (테스트 모드일 경우)
□ OAuth 클라이언트 ID 생성 (웹 애플리케이션)
□ 리디렉션 URI 등록 (로컬/dev/prod 각각)
□ client-id / client-secret → 환경변수 주입
□ 출시 전 프로덕션으로 게시 전환
```

---

가장 흔한 실수가 **리디렉션 URI 불일치**야. 코드의 `redirect_uri`와 콘솔에 등록한 URI가 완전히 동일한지 꼭 확인해.