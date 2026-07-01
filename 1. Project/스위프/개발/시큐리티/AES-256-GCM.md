
## GCM의 구조: CTR 모드 + 인증

AES-GCM은 두 부분의 조합이야.

1. **CTR(Counter) 모드**로 평문을 암호화 — AES를 블록 암호가 아니라 스트림 암호처럼 쓰는 방식. Nonce + 카운터를 AES로 암호화한 결과를 평문과 XOR
2. **GHASH**로 인증 태그 생성 — 암호화된 결과 전체에 대한 MAC을 계산

이 둘을 합쳐서 "암호화하면서 동시에 변조 여부까지 검증 가능한" AEAD(Authenticated Encryption with Associated Data)가 되는 거고, 그래서 별도로 HMAC을 덧붙일 필요가 없어.

## 저장/조회 시 실제 만들어지는 값

Cipher 호출하면 이렇게 나와:

```
IV (12 bytes, 96bit 권장) + Ciphertext + Auth Tag (16 bytes, 128bit)
```

AttributeConverter에서 이 셋을 concat해서 (보통 Base64로) 컬럼에 저장하는 구조일 텐데, 여기서 **IV는 매번 새로 랜덤 생성**해서 암호문 앞에 붙여야 해. GCM에서 가장 치명적인 실수가 **같은 키로 같은 IV를 두 번 쓰는 것**이야 — 이러면 XOR 특성상 keystream이 재사용되면서 평문 복구가 가능해질 수 있어. `SecureRandom`으로 매 암호화마다 새 IV를 뽑고 있는지가 핵심 체크포인트.

## javax.crypto 구현 시 흔한 함정

```java
Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
```

- `Cipher` 인스턴스는 **thread-safe하지 않음** — AttributeConverter가 싱글톤 빈이면 `Cipher.getInstance()`를 매 호출마다(또는 ThreadLocal로) 새로 만들어야지, 필드로 캐싱해서 재사용하면 동시성 문제 생김
- `GCMParameterSpec`의 tag length는 보통 128bit(16byte)로 고정하는 게 안전
- 복호화 시 태그 검증 실패하면 `AEADBadTagException`이 던져지는데, 이걸 잡아서 로그에 원본 ciphertext나 key 관련 정보가 새지 않도록 주의

## 진짜 중요한 건 "암호화 키를 어디에 저장하느냐"

여기가 이 설계에서 제일 취약해지기 쉬운 지점이야. AES 키 자체를:

- 코드/설정파일에 하드코딩 → Apple private key 평문 커밋했던 것과 똑같은 실수 반복
- application.yml에 평문 → git에 안 올려도 서버 파일시스템 접근 시 노출
- 환경변수 → 최소선은 되지만, EC2 인스턴스 자체가 털리면 같이 털림

이상적으로는 AWS KMS나 Secrets Manager로 키 자체를 분리 보관하고, 애플리케이션은 부팅 시점에만 가져와 메모리에 두는 구조가 맞아. 지금 이 AES 키는 어디서 가져오고 있어? (env var인지, KMS인지) 여기에 따라 "DB 유출돼도 안전"이라는 전제가 실제로 성립하는지가 갈려.

## 하나 더: 키 로테이션 고려했는지

지금 구조에서 나중에 AES 키를 교체해야 하는 상황(유출 의심, 정기 로테이션 정책 등)이 오면, 기존에 저장된 모든 refresh_token을 복호화 → 재암호화하는 마이그레이션이 필요해. 지금 당장 급한 건 아니지만, 컬럼에 "어떤 키 버전으로 암호화됐는지" 식별할 수 있는 필드(key version)를 미리 넣어두면 나중에 편해.