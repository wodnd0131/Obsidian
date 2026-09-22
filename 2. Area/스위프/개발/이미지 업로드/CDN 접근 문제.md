## 확인 완료 / 배제된 요인

| 항목                                                 | 결과                                                     |
| -------------------------------------------------- | ------------------------------------------------------ |
| Presigned URL 만료 시간                                | 정상 (Expires가 미래 시점)                                    |
| EC2 서버 시계 동기화                                      | 정상 (NTP synced)                                        |
| Private key ↔ Public key 쌍                         | 일치 확인 (modulus 동일)                                     |
| 컨테이너(`swyp-spring-blue`)에 주입된 키                    | 로컬 파일과 동일                                              |
| CloudFront ↔ 콘솔 등록 공개키(`KFIP744J8UJUD`)            | 일치 확인                                                  |
| DNS (`cdn.orbitss.xyz` → CloudFront)               | 정상                                                     |
| CloudFront Behavior (Default *, Trusted Key Group) | 정상 설정                                                  |
| S3 Block Public Access                             | 비활성화 (문제 없음)                                           |
| S3 Object Ownership                                | Bucket owner enforced (ACL 무시, 문제 없음)                  |
| S3 버킷 정책                                           | `PublicReadGetObject` 제거, OAC statement만 남김 (보안 개선 완료) |

## 진짜 원인 (확정)

**Spring Boot에서 CDN/presigned URL을 생성할 때 객체 key에 확장자(`.webp`)를 붙이지 않고 있었음.**

- Lambda(`sharp` 기반 썸네일 생성)는 원본 포맷과 무관하게 **항상 `{variant}/{basename}.webp`** 형태로 S3에 저장하도록 고정되어 있음 (`thumbnail/`, `main/` 두 variant 모두 webp 고정)
- 반면 URL 서명 로직은 확장자 없이 `thumbnail/{id}` 형태로 서명 → S3에 실제 존재하는 key(`.webp` 포함)와 불일치 → CloudFront edge 검증은 통과하지만 origin(S3)에서 `AccessDenied` (ListBucket 권한 없어 404 대신 403 반환)
- 실제로 `.webp`를 수동으로 붙인 URL은 edge 서명 검증 실패로 403 (당연함 — 서명 자체가 확장자 없는 경로 기준으로 생성됐기 때문)
- → **URL 문자열을 사후 편집해서 검증하는 방식으론 확인 불가하고, 코드 자체에서 `.webp` 포함해 새로 서명해야 검증 가능**

## 남은 작업

1. Spring Boot에서 이미지 key/URL을 만드는 지점(presigned URL 생성 로직, 혹은 DB에 저장된 이미지 참조값)에 `.webp` 확장자를 반영
    - 가장 깔끔한 방법: 업로드 완료 시점에 DB에 저장하는 값 자체를 `{basename}.webp`로 통일, 이후 URL 생성 로직은 그 값을 그대로 사용
    - 또는 URL 생성 헬퍼에서 항상 `.webp`를 하드코딩으로 append (variant가 thumbnail/main 두 개로 고정이고 항상 webp라면 이게 더 간단)
2. 수정 후 재배포 → 새로 발급한 URL로 재테스트 (`200 OK` + `content-type: image/webp` 확인)
3. QA 리포트에 있는 다른 이미지 관련 버그들도 동일 원인인지 함께 확인 (archive/보관함 이미지 로딩 이슈도 이거였을 가능성 있음)

코드에서 실제 URL 생성하는 부분(서비스/유틸 클래스) 보여주시면 수정 지점 바로 짚어드릴게요.