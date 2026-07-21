

> 배경: S3 프리사인드 URL 기반 이미지 제공 → CloudFront 캐싱 도입 검토 작성일: 2026-07-16

---

## 1. 문제: 캐싱이 안 되는 이유가 뭔가?

**증상**: CloudFront를 붙여도 캐시 히트율이 안 나올 것으로 예상됨

**원인 분석**:

- 프리사인드 URL은 쿼리스트링에 `X-Amz-Signature`, `X-Amz-Date` 등 **요청마다 달라지는 값**이 포함됨
- CloudFront 기본 Cache Policy는 쿼리스트링 전체를 캐시 키에 포함 → 같은 이미지여도 매번 다른 캐시 키로 인식 → 캐시 미스 반복

**확인 결과**:

- 현재 실제 사용 중인 URL은 `https://swyp-prod-media.s3.ap-northeast-2.amazonaws.com/main/{key}.webp` 형태로 **쿼리스트링이 없는 순수 객체 키 URL**
- → 이 문제는 **해당 없음**. 경로 자체가 캐시 키가 되므로 캐싱 정상 동작 가능

---

## 2. 문제: 그런데 지금 구조가 보안적으로 괜찮은가?

**증상**: 별도 인증 없이 URL만 알면 누구나 접근 가능한 상태 (버킷 퍼블릭 or 퍼블릭 ACL)

**리스크**:

- URL이 유출/로그 노출될 경우 만료 없이 영구 접근 가능
- UUID 기반 파일명에만 의존하는 "obscurity 보안" → 실질적 접근 통제 없음
- 소셜 프로필 교환 앱 특성상 개인정보 성격의 이미지 → 컴플라이언스 리스크 존재

**결정**: 캐싱 도입과 별개로, **보안 강화(Private 전환 + 인증 방식 도입)를 같이 진행**하기로 함

---

## 3. 문제: 보안 강화 시 아키텍처를 어떻게 바꿔야 하나?

**선택한 방향**: OAC(Origin Access Control) + CloudFront 인증 방식으로 전환

- S3 버킷 완전 Private 전환 (Block Public Access 재활성화, 퍼블릭 ACL 제거)
- OAC 설정으로 **CloudFront만 S3에 접근 가능**하도록 버킷 정책 제한
- 인증 파라미터(서명)는 CloudFront 레벨에서만 검증, S3는 원천 비공개

**남은 결정 사항**: 인증 방식을 Signed URL로 할지 Signed Cookie로 할지

---

## 4. 문제: Signed URL이면 조회 때마다 백엔드가 서명을 매번 만들어줘야 하나?

**질문 배경**: 기존엔 이미지 key만 반환하던 구조였는데, 조회마다 권한(서명)을 발급하는 형태로 바뀌는 게 맞는지 확인 필요했음

**답**: 맞음. 다만:

- 서명 생성은 AWS API 호출이 아니라 **개인키로 로컬에서 계산하는 RSA 연산** → 지연/비용 부담 거의 없음
- 실질적 이슈는 "이미지 여러 장을 한 번에 반환하는 API(보관함 리스트 등)"에서 **N개 이미지 = N번 서명 반복**이 발생한다는 점

---

## 5. 문제: Signed URL vs Signed Cookie, 뭘 써야 하나?

**비교 기준**: 협업 환경이 원활하지 않은 상황 → 프론트-백엔드 계약 변경 최소화가 우선순위

|항목|Signed URL|Signed Cookie|
|---|---|---|
|서명 단위|이미지 1개당 1회|세션/로그인당 1회|
|프론트 응답 필드 변경|`imageKey` → `imageUrl` (필드명만 교체)|`imageKey` 유지, but 조립 로직 + 쿠키 핸들링 신규 필요|
|RN 호환성 리스크|없음 (완성 URL이라 어떤 Image 라이브러리든 그대로 동작)|있음 (RN Image 컴포넌트가 쿠키 자동 전송 안 할 수 있음 → 401 리스크)|
|재인증 처리|불필요 (URL 자체에 만료 포함, 재요청 시 새로 발급)|쿠키 만료 감지 → 재발급 API 호출 → 재시도 로직 별도 구현 필요|
|협업 커뮤니케이션 비용|낮음 (필드명 변경 공지 한 줄)|높음 (쿠키 발급 시점/도메인/재발급 트리거 등 다수 논의 필요)|

**결정**: **Signed URL 방식 채택**

- 서버 부하(서명 반복 계산)는 로컬 RSA 연산이라 무시 가능한 수준으로 판단
- 대신 프론트 변경 범위를 최소화하는 게 현재 협업 상황에서 더 중요한 이득

---

## 6. 결정된 최종 방향 요약

1. **S3**: Block Public Access 활성화, 퍼블릭 ACL 전면 제거
2. **CloudFront**: OAC 기반 오리진 연결, S3는 CloudFront를 통해서만 접근 가능
3. **인증**: Trusted Key Group 기반 **Signed URL** 방식 채택 (Cookie 방식 대비 RN 호환성/협업 비용 측면에서 우위)
4. **응답 스펙 변경**: `imageKey` 필드 → `imageUrl` 필드로 변경, 완성된 서명 URL을 그대로 내려줌
5. **캐싱**: 쿼리스트링(Policy/Signature/Key-Pair-Id)은 캐시 키에서 제외되도록 Cache Policy 구성 → 경로 기준으로 정상 캐싱 동작

---

## 7. 다음에 결정해야 할 것 (Open Items)

- [ ] Spring Boot 백엔드에 CloudFront 서명 발급 로직 구현 (SDK v1 `CloudFrontUrlSigner` vs v2 `CloudFrontUtilities`, 현재 SDK 버전 확인 필요)
- [ ] 개인키(`private_key.pem`) 보관 위치 결정 (AWS Secrets Manager / Parameter Store)
- [ ] Signed URL 만료 시간(TTL) 정책 결정 (보관함 조회 특성상 몇 분~몇 시간이 적절한지)
- [ ] Lambda 썸네일 생성 파이프라인에서 S3 업로드 시 `Cache-Control` 헤더 명시적으로 설정할지 여부
- [ ] CloudFront 커스텀 도메인(`cdn.orbitss.xyz` 등) 및 인증서(ACM) 발급 여부