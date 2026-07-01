

## Context
MVP 앱 배포를 앞두고 있고, Apple은 "Sign in with Apple"을 지원하는 앱에 대해 **앱 내 계정 삭제 시 Apple 계정 연결도 함께 해제(revoke)**하도록 요구한다(App Store Review Guideline 5.1.1(v)). 현재 회원탈퇴(`UserService.deleteUser`)는 로컬 DB 삭제만 수행하고 Apple 쪽 연결 해제는 하지 않는다.

원인은 Apple의 `refresh_token`을 애초에 저장하지 않고 있기 때문이다. 로그인 흐름을 조사한 결과:
- **iOS 앱 로그인** (`POST /auth/apple/token`, [AppleAuthController.java:34-39](src/main/java/org/swyp/com/backend/auth/oauth/apple/controller/AppleAuthController.java)) — `identityToken`만 받아 서명 검증만 하고 끝. Apple `/auth/token` 호출 자체를 안 하므로 refresh_token을 받을 수가 없음.
- **웹 콜백** (`POST /auth/apple/callback`, [AppleAuthController.java:56-71](src/main/java/org/swyp/com/backend/auth/oauth/apple/controller/AppleAuthController.java)) — Apple이 `code`와 `id_token`을 동시에 보내주는데(`response_type=code id_token`), `id_token`이 있으면 무조건 `loginWithIdentityToken` 분기를 타서 `code`를 그냥 버림. 결국 여기서도 refresh_token을 받은 적이 없음.

목표: iOS/Android/Web(테스트용) 세 경로 모두에서 회원탈퇴 시 Apple revoke가 실제로 동작하는 것을 확인한다. 사용자 승인: iOS 앱도 `authorizationCode`를 함께 보내도록 API 계약을 변경해도 됨.

## 설계

### 1. Apple refresh_token 저장
- **User 엔티티** ([User.java](src/main/java/org/swyp/com/backend/user/domain/User.java)): `appleRefreshToken` 컬럼 추가 (nullable). `ddl-auto=update`라 별도 마이그레이션 스크립트 불필요.
- **암호화**: 신규 `AttributeConverter<String, String>` (`AppleRefreshTokenConverter`, `auth/oauth/apple` 패키지)로 AES/GCM 암호화 후 저장. 키는 `AppleProperties`에 `refreshTokenSecret` 필드 추가 → `apple.refresh-token-secret` 프로퍼티로 주입(로컬/테스트/운영 properties에 32바이트 base64 키 추가). 컨버터를 컬럼에 `@Convert`로 적용.
- **User에 업데이트 메서드 추가**: `updateAppleRefreshToken(String token)` (기존 코드가 setter 없이 정적 팩토리 메서드 패턴을 쓰므로 동일 스타일 유지).

### 2. authorizationCode 수신 및 refresh_token 획득
- **`AppleLoginRequest`** ([AppleLoginRequest.java](src/main/java/org/swyp/com/backend/auth/oauth/apple/dto/AppleLoginRequest.java)): `authorizationCode` 필드 추가 (nullable — Android 등 기존 클라이언트 하위호환, `@NotBlank` 걸지 않음).
- **`AppleAuthService`** ([AppleAuthService.java](src/main/java/org/swyp/com/backend/auth/oauth/apple/service/AppleAuthService.java)):
  - `loginWithIdentityToken(String identityToken, String authorizationCode)`로 시그니처 변경(기존 단일 파라미터 오버로드는 제거 — 호출부 2곳뿐이라 전부 갱신).
  - 흐름: 기존처럼 `identityToken`으로 서명 검증 + 유저 조회/생성 → `authorizationCode`가 존재하면 `tokenClient.exchangeCode(authorizationCode)`로 refresh_token 획득 시도 → 성공하면 `user.updateAppleRefreshToken(...)`. 이 exchange는 **best-effort**로 처리(실패해도 로그인 자체는 성공시켜야 함 — try/catch로 감싸고 실패 시 로그만 남김. 실패 시 회원탈퇴 시 revoke만 스킵됨).
  - `loginWithCode(String code)` 경로(콜백에서 id_token 없을 때)도 동일하게 refresh_token을 저장하도록 `processAppleUser`에 반영.
- **`AppleAuthController`** ([AppleAuthController.java:35-39,56-71](src/main/java/org/swyp/com/backend/auth/oauth/apple/controller/AppleAuthController.java)):
  - `appleAppLogin`: `appleAuthService.loginWithIdentityToken(request.identityToken(), request.authorizationCode())` 호출.
  - `callback`: `id_token`이 있어도 `code`를 함께 넘겨 `loginWithIdentityToken(id_token, code)` 호출 — 웹 테스트 경로에서도 refresh_token이 저장되도록.
- **`AppleAuthApiSpec`** ([AppleAuthApiSpec.java](src/main/java/org/swyp/com/backend/auth/oauth/apple/controller/api/AppleAuthApiSpec.java)): Swagger 설명에 `authorizationCode` 파라미터 안내 추가.
- **테스트**: `AppleAuthServiceTest`의 기존 3개 테스트가 `loginWithIdentityToken("dummy.token")` 단일 인자로 호출 중 — 시그니처 변경에 맞춰 `loginWithIdentityToken("dummy.token", null)`로 수정, `tokenClient` 관련 stubbing은 필요한 경우만 추가.

### 3. Revoke API 호출
- **`AppleTokenClient`** ([AppleTokenClient.java](src/main/java/org/swyp/com/backend/auth/oauth/apple/service/AppleTokenClient.java)): `revoke(String token)` 메서드 추가. `POST https://appleid.apple.com/auth/revoke`, form body `client_id`, `client_secret`(기존 `AppleClientSecretGenerator.generate()` 재사용), `token`, `token_type_hint=refresh_token`. 기존 `exchangeCode`와 동일한 에러 처리 패턴(`ExternalApiConnectionException`) 사용.
- **`AppleAuthService`**: `revoke(User user)` 메서드 추가 — `user.getAppleRefreshToken()`이 없으면 즉시 반환(로그만), 있으면 `tokenClient.revoke(token)` 호출. 실패해도 예외를 던지지 않고 로그만 남김(탈퇴 자체를 막으면 안 됨 — Apple이 이미 revoke된 토큰에 재요청하거나 네트워크 문제가 있어도 로컬 탈퇴는 항상 성공해야 함).

### 4. 회원탈퇴 플로우 연동
- **`UserService`** ([UserService.java:31-49](src/main/java/org/swyp/com/backend/user/service/UserService.java)): `AppleAuthService` 주입 → `deleteUser`에서 `user.getProvider() == OAuthProvider.APPLE`이면 로컬 데이터 삭제 전에 `appleAuthService.revoke(user)` 호출. (다른 provider는 현재 GOOGLE/KAKAO뿐이고 revoke 요건이 확인되지 않았으므로 이번 범위에서는 다루지 않음.)

## 검증 계획
- 단위 테스트: `AppleAuthServiceTest`에 refresh_token 저장/미저장 케이스, revoke 호출 케이스 추가. `AppleTokenClient` revoke에 대한 테스트(성공/실패) 추가.
- `./gradlew test`로 전체 테스트 통과 확인.
- 수동 검증(가능한 범위): 로컬 서버 기동 후 Swagger `/auth/apple` 웹 리다이렉트로 실제 Apple 로그인 → 로그인 성공 후 DB에 `apple_refresh_token` 값이 저장되는지 확인 → `DELETE /users/me` 호출 → 서버 로그에서 Apple revoke 호출 및 응답(200) 확인.
- iOS/Android 클라이언트 쪽 `authorizationCode` 전달은 이번 작업 범위 밖(별도로 모바일 팀 전달 필요) — 백엔드는 필드가 없어도(null) 동작하도록 하위호환 유지.

## 커밋 전략
작은 단위로 나눠 커밋:
1. User 엔티티 + 암호화 컨버터 + 프로퍼티 추가
2. AppleLoginRequest/AppleAuthService/AppleAuthController — authorizationCode 수신 및 refresh_token 저장
3. AppleTokenClient/AppleAuthService — revoke 메서드 추가
4. UserService — 탈퇴 플로우에 revoke 연동
5. 테스트 보강