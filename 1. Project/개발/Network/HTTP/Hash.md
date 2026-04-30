# Hash — 내부 동작과 설계 근거

---

## 1. Problem First

### 시나리오 A: 비밀번호를 평문으로 저장하던 시절

```java
// DB에 그대로 저장
INSERT INTO users (password) VALUES ('myPassword123');

// 인증
SELECT * FROM users WHERE password = 'myPassword123';
```

**DB 덤프 한 번이면 전체 사용자 비밀번호가 노출된다.** 2012년 LinkedIn 유출 사고가 정확히 이 구조였다 — 650만 건.

---

### 시나리오 B: 암호화(Encryption)로 저장하던 시절

```java
String encrypted = AES.encrypt(password, secretKey);
// DB에 encrypted 저장
```

**근본적으로 틀린 접근이다.** 암호화는 복호화가 가능하다는 전제 위에 있다. 
즉, `secretKey`가 노출되는 순간 전체 복호화가 가능하다.

**진짜 필요한 것은 "검증은 가능하지만, 원본을 되돌릴 수 없는" 연산이다.** → 이것이 Hash가 등장한 근본 이유다.

---

## 2. Mechanics

### 2-1. Hash란 무엇인가 — 수학적 정의

Hash 함수 `H`는 다음 성질을 만족하는 함수다.

```
H : {0,1}* → {0,1}^n

임의 길이 입력  →  고정 길이 출력
```

**암호학적 해시 함수(Cryptographic Hash Function)가 추가로 요구하는 세 가지 성질:**

| 성질                             | 정의                                               | 깨지면 생기는 문제           |
| ------------------------------ | ------------------------------------------------ | -------------------- |
| **Preimage resistance**        | `H(x) = y`일 때, `y`만 보고 `x`를 찾는 것이 계산상 불가능        | 해시값으로 원본 복원 가능       |
| **Second preimage resistance** | `x`가 주어졌을 때, `H(x') = H(x)`인 `x' ≠ x`를 찾는 것이 불가능 | 다른 입력으로 같은 해시 위조 가능  |
| **Collision resistance**       | `H(x) = H(x')`인 임의의 `x ≠ x'` 쌍을 찾는 것이 불가능        | 같은 해시를 가진 두 문서 생성 가능 |

> 📎 **근거:** NIST FIPS 180-4 _"Secure Hash Standard"_, Section 3 — Security Properties

---

### 2-2. SHA-256 내부 동작 — 비트 단위까지

JWT 서명에서 `HS256`은 SHA-256 기반이다. 실제로 어떻게 작동하는지 따라간다.

**구조: Merkle–Damgård Construction**

```
입력 메시지 M
    ↓
[패딩] → 512비트 블록 단위로 정렬
    ↓
 블록1 → 블록2 → 블록3 → ... → 블록n
   ↓        ↓
  압축함수  압축함수
   ↓        ↓
  IV → [state] → [state] → ... → 최종 256비트 해시
```

**핵심: 압축 함수 한 라운드**

```
입력: 256비트 내부 상태 (a, b, c, d, e, f, g, h) + 512비트 메시지 블록
      ↓
      64라운드 반복 (각 라운드에서 비트 혼합 연산)
      ↓
      - Ch(e,f,g)    = (e AND f) XOR (NOT e AND g)
      - Maj(a,b,c)   = (a AND b) XOR (a AND c) XOR (b AND c)
      - Σ0(a)        = ROTR²(a) XOR ROTR¹³(a) XOR ROTR²²(a)
      - Σ1(e)        = ROTR⁶(e) XOR ROTR¹¹(e) XOR ROTR²⁵(e)
      ↓
출력: 새로운 256비트 내부 상태
```

> ROTR = 비트 오른쪽 회전(Rotate Right)

**왜 이 연산들인가?** 각 출력 비트가 입력 전체에 의존하도록 만들기 위해서다. 입력 1비트가 바뀌면 출력의 절반(~128비트)이 바뀐다 — **Avalanche Effect**.

```java
// Avalanche Effect 직접 확인
MessageDigest sha256 = MessageDigest.getInstance("SHA-256");

byte[] hash1 = sha256.digest("hello".getBytes());
byte[] hash2 = sha256.digest("hellp".getBytes()); // 마지막 글자 1개만 변경

// hash1: 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
// hash2: b2b6f4c8e6b0e2f4c8e6b0e2f4c8e6... (완전히 다른 값)
```

> 📎 **근거:** NIST FIPS 180-4, Section 6.2 — SHA-256 알고리즘 상세

---

### 2-3. JWT 서명에서 Hash가 쓰이는 방식 — HMAC

JWT의 `HS256`은 Hash를 그대로 쓰는 게 아니라 **HMAC(Hash-based Message Authentication Code)** 을 쓴다.

**왜 Hash를 그냥 못 쓰는가?**

```
// 나쁜 예 — Hash만 사용
signature = SHA256(header + "." + payload)

// 공격자도 똑같이 계산할 수 있다
// payload를 바꾸고 SHA256을 다시 돌리면 유효한 서명이 만들어진다
```

Hash는 **키가 없는 공개 함수**다. 누구나 계산할 수 있기 때문에 서명으로 쓸 수 없다.

**HMAC 구조:**

```
HMAC(K, M) = H( (K' XOR opad) || H( (K' XOR ipad) || M ) )

K  = 비밀키
M  = 메시지 (header.payload)
opad = 0x5c 반복
ipad = 0x36 반복
```

코드로 풀면:

```java
// Java에서 HMAC-SHA256 직접 구현 흐름 (실제 JWT 라이브러리 내부와 동일)
Mac mac = Mac.getInstance("HmacSHA256");
SecretKeySpec keySpec = new SecretKeySpec(secret.getBytes(), "HmacSHA256");
mac.init(keySpec);

// JWT에서 실제로 서명하는 대상
String signingInput = base64UrlEncode(header) + "." + base64UrlEncode(payload);
byte[] signature = mac.doFinal(signingInput.getBytes(StandardCharsets.UTF_8));
```

**HMAC이 단순 `H(K || M)`보다 안전한 이유:**

```
단순 H(K || M) 방식의 취약점 — Length Extension Attack

SHA-256은 Merkle-Damgård 구조이기 때문에
H(K || M)을 알면, K를 모르더라도
H(K || M || 추가데이터)를 계산할 수 있다.

HMAC은 두 번 해싱하는 구조로 이 공격을 차단한다.
```

> 📎 **근거:** RFC 2104 — _"HMAC: Keyed-Hashing for Message Authentication"_, Section 2

---

### 2-4. RS256과의 차이 — Hash + 비대칭 암호

```
HS256: HMAC-SHA256  → 대칭키, 서명/검증에 같은 키 사용
RS256: RSA + SHA256 → 비대칭키, 서명=개인키, 검증=공개키

RS256 내부:
1. SHA256(header.payload) → 32바이트 다이제스트
2. RSA_PKCS1_v1.5_Sign(privateKey, digest) → 서명

// Hash가 필요한 이유:
// RSA는 큰 수의 거듭제곱 연산이라 입력이 작아야 한다
// 원문을 직접 RSA에 넣지 않고, 먼저 Hash로 압축한 뒤 RSA에 넣는다
```

---

## 3. 공식 근거

|주장|출처|
|---|---|
|SHA-256 알고리즘 상세 (라운드 연산, 패딩)|NIST FIPS 180-4, §6.2|
|Preimage / Collision resistance 정의|NIST FIPS 180-4, §3|
|HMAC 구조 및 ipad/opad 설계|RFC 2104, §2|
|Length Extension Attack 및 HMAC이 막는 방식|RFC 2104, §6 Security|
|JWT 서명 알고리즘 (HS256, RS256 명세)|RFC 7518 — JSON Web Algorithms, §3|

---

## 4. 이 설계를 이렇게 한 이유

### SHA-256이 Merkle–Damgård를 쓰는 이유와 대가

**이점:** 스트리밍 처리 가능. 메시지 전체를 메모리에 올리지 않고 블록 단위로 처리한다.

**대가:** Length Extension Attack에 구조적으로 취약하다. → SHA-3(Keccak)은 이 구조를 버리고 Sponge Construction으로 설계되어 이 문제를 해결했다. → 그래서 SHA-256을 쓸 때는 반드시 HMAC으로 감싸야 한다.

### 256비트 고정 출력 길이의 트레이드오프

**이점:** 어떤 크기의 입력이든 동일한 크기로 비교 가능. 인덱싱, 서명에 예측 가능한 크기 보장.

**대가:** 비둘기집 원리상 충돌은 반드시 존재한다. 현재 SHA-256의 충돌 탐색 난이도는 `2^128` 연산으로, 현재 컴퓨팅 파워로는 불가능하다. 양자 컴퓨터 환경에서는 Grover 알고리즘으로 `2^64`로 줄어든다 — 이 때문에 SHA-3/SHA-512로의 전환 논의가 이미 진행 중이다.

---

## 5. 이어지는 개념

|순서|개념|이유|
|---|---|---|
|1|**Salt & Rainbow Table**|해시를 비밀번호 저장에 쓸 때 HMAC만으로 충분하지 않은 이유 — 이걸 모르면 `bcrypt`를 왜 쓰는지 이해 못 한다|
|2|**bcrypt / Argon2 (Password Hashing)**|SHA-256은 빠른 게 장점이지만, 비밀번호 해싱에서는 빠른 게 단점이 된다는 역설 — 선행으로 Salt 개념 필요|
|3|**PKI & 인증서 체계 (X.509)**|RS256을 쓸 때 공개키를 어떻게 신뢰하는가 — JWT의 실무 배포에서 반드시 만나는 문제|
|4|**SHA-3 / Keccak**|Merkle–Damgård의 구조적 한계를 어떻게 해결했는가 — 현재 표준이 왜 바뀌고 있는지 이해하는 데 필요|