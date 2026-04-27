#Network/Protocol/HTTP
# TCP (Transmission Control Protocol)
https://developer.mozilla.org/ko/docs/Glossary/TCP

TCP (전송 제어 프로토콜)은 두 개의 호스트를 연결하고 데이터 스트림을 교환하게 해주는 중요한 네트워크 [프로토콜](https://developer.mozilla.org/ko/docs/Glossary/Protocol)입니다. TCP는 데이터와 [[패킷]]이 보내진 순서대로 전달하는 것을 보장해줍니다. Vint CERF와 Bob Kahn (당시 DARPA 과학자)는 TCP를 1970년 대에 설계하였습니다.

TCP의 역할은 에러가 없이 패킷이 신뢰할 수 있게 전달 되었는지 보증해 주는 것입니다. TCP는 [혼잡 제어](https://en.wikipedia.org/wiki/TCP_congestion_control)를 구현합니다. 이는 초기 요청은 작게 시작하여 컴퓨터들과 서버 및 네트워크가 지원할 수 있는 대역폭 수준까지 크기
가 증가한다는 것을 의미합니다.



## 1. Problem First

### 인터넷 위에서 데이터를 보낸다는 게 왜 어려운가

인터넷은 근본적으로 **패킷 교환망(packet-switched network)** 이다. 데이터를 한 덩어리로 보내는 게 아니라, 잘게 쪼갠 패킷들이 각자 다른 경로로 목적지를 향해 떠난다.

이 구조에서 네트워크 레이어(IP)는 딱 하나의 약속만 한다.

> **"Best-effort delivery"** — 최선을 다해 보내긴 할게. 근데 도착 보장은 못 해.

즉, IP 계층은 아래 세 가지를 **보장하지 않는다.**

|문제|실제 발생 원인|
|---|---|
|패킷 유실|라우터 버퍼 초과, 물리 회선 장애|
|패킷 순서 뒤바뀜|패킷마다 다른 경로 경유|
|패킷 중복 수신|라우터가 ACK를 못 받고 재전송|

### TCP 없이 HTTP 통신을 구현한다면

```java
// TCP 없이 순수 IP 위에서 HTTP 응답을 받는다고 가정한 의사코드
DatagramSocket socket = new DatagramSocket();
socket.send(httpRequestPacket);

// 문제 1: 패킷이 유실되면 응답이 영원히 안 온다
// 문제 2: 응답이 3개 패킷으로 쪼개져 왔을 때 순서가 뒤바뀌면
//         HTTP 헤더가 중간에 끼어들어 파싱 자체가 불가능
// 문제 3: 내가 요청을 보냈는데 서버가 열려있는지조차 모른다
byte[] response = receive(); // 이게 완전한 데이터인지 알 수 없음
```

Spring의 `RestTemplate`이나 `HttpClient`가 HTTP 요청을 보낼 때 개발자가 패킷 유실을 걱정하지 않아도 되는 이유가 바로 아래에서 설명할 TCP가 이 모든 문제를 해결해주기 때문이다.

---

## 2. Mechanics

### TCP가 해결하는 세 가지 문제와 내부 동작

---

### 2-1. 연결 수립 — 3-Way Handshake

통신 전에 "우리 지금 연결됐어?" 를 먼저 확인한다.

```
Client                          Server
  |                               |
  |  ── SYN (seq=100) ──────────> |   "연결하고 싶어, 내 시퀀스 번호는 100"
  |                               |
  |  <── SYN-ACK (seq=300,      | 
  |        ack=101) ─────────── |   "좋아, 내 번호는 300. 너 101번 기다릴게"
  |                               |
  |  ── ACK (ack=301) ─────────> |   "확인. 너 301번 기다릴게"
  |                               |
  |         [데이터 전송 시작]      |
```

**왜 3번인가? 2번으로는 안 되는가?**

2-Way로 끝내면 서버는 클라이언트가 자신의 SYN-ACK를 받았는지 확인할 방법이 없다. 클라이언트의 ACK(3번째)가 있어야 비로소 **양방향 모두** 통신 가능함이 검증된다.

> RFC 793, Section 3.4: _"The reason for the three-way handshake is to prevent old duplicate connection initiations from causing confusion."_ (3-Way Handshake를 사용하는 이유는 오래된 중복 연결 시도가 혼란을 일으키는 것을 막기 위해서다.)

**시퀀스 번호(ISN)는 왜 0부터 시작하지 않는가?**

위 예시에서 `seq=100`, `seq=300` 처럼 임의의 값으로 시작한다. 이를 **ISN(Initial Sequence Number)** 라 한다. 0부터 시작하면 이전 연결의 지연된 패킷이 새 연결에 끼어드는 문제가 생긴다. ISN은 시간 기반 알고리즘으로 생성해 이를 방지한다.

> RFC 793, Section 3.3: _"To avoid confusion we must prevent segments from one incarnation of a connection from being used while the same pair of sockets are in an incarnation from a previous incarnation is still in the network."_

---

### 2-2. 신뢰성 — ACK & 재전송

모든 데이터에 **시퀀스 번호**를 붙이고, 수신 측은 **ACK(Acknowledgment)** 로 확인 응답한다.

```
Sender                          Receiver
  |                               |
  |  ── [seq=1, data="Hello"] ──> |
  |  ── [seq=6, data="World"] ──> |   (패킷 유실 가정)
  |  ── [seq=11, data="!"] ─────> |
  |                               |
  |                               |   seq=6 패킷이 없음 → 
  |  <── ACK=1  ─────────────── |   "1번 받았어. 6번 줘"
  |  <── ACK=1  ─────────────── |   (중복 ACK — 6번 아직 없음)
  |  <── ACK=1  ─────────────── |   (중복 ACK 3회)
  |                               |
  |  ── [seq=6 재전송] ──────────> |   3 Duplicate ACK → 즉시 재전송
  |                               |
  |  <── ACK=12 ────────────── |   "12번 기다릴게" (전부 받음)
```

재전송 트리거는 두 가지다.

|트리거|설명|
|---|---|
|**Retransmission Timeout (RTO)**|ACK가 일정 시간 내 안 오면 재전송|
|**3 Duplicate ACK**|같은 ACK가 3번 오면 즉시 재전송 (Fast Retransmit)|

> RFC 5681 (TCP Congestion Control), Section 2: _"TCP SHOULD use the "fast retransmit" algorithm to detect and repair loss, based on incoming duplicate ACKs."_

---

### 2-3. 순서 보장 — 시퀀스 번호 재조립

패킷이 `seq=11 → seq=1 → seq=6` 순서로 도착해도, 수신 측 버퍼에서 시퀀스 번호 순서대로 재조립한 뒤 애플리케이션에 올려보낸다.

Java 애플리케이션 입장에서는 `InputStream.read()`로 데이터를 읽을 때 항상 순서가 보장된 바이트 스트림을 받는 이유가 이것이다.

---

### 2-4. 흐름 제어 — Sliding Window

수신 측이 처리할 수 있는 양보다 더 많이 보내면 버퍼가 넘친다. 이를 막기 위해 수신 측이 **윈도우 크기(rwnd, receive window)** 를 광고(advertise)한다.

```
Receiver → Sender: "ACK=101, window=4096"
// "101번부터 받을게. 지금 내 버퍼 여유는 4096바이트야."

Sender: 4096바이트 이상은 ACK 받기 전까지 보내지 않음
```

수신 버퍼가 꽉 차면 `window=0`을 보내 송신을 일시 중단시킨다.

---

### 2-5. 혼잡 제어 — 네트워크 보호

흐름 제어가 "수신 측 보호"라면, 혼잡 제어는 **"네트워크 전체 보호"** 다.

핵심 알고리즘: **Slow Start → Congestion Avoidance → Fast Recovery**

```
cwnd (혼잡 윈도우)
  ^
  |        /
  |       /     ← Congestion Avoidance (선형 증가)
  |      /|
  |  /\/ |
  | /    |← 패킷 손실 감지 → cwnd 절반으로 줄임
  |/
  +-------------------------> time
  Slow Start (지수 증가)
```

실제로 보내는 양 = `min(rwnd, cwnd)` — 수신 측과 네트워크 양쪽 모두를 고려한다.

> RFC 5681, Section 1: _"This document specifies four intertwined TCP congestion control algorithms: slow start, congestion avoidance, fast retransmit, and fast recovery."_

---

### 2-6. 연결 종료 — 4-Way Handshake

```
Client                          Server
  |  ── FIN ──────────────────> |   "나는 다 보냈어"
  |  <── ACK ─────────────────  |   "알겠어"
  |                               |   (서버는 아직 보낼 게 있을 수 있음)
  |  <── FIN ─────────────────  |   "나도 다 보냈어"
  |  ── ACK ──────────────────> |   "확인"
  |                               |
  |  [TIME_WAIT 2MSL 대기]       |   연결 종료
```

**왜 3-Way가 아니라 4-Way인가?**

Half-close 때문이다. 클라이언트가 FIN을 보내도 서버는 아직 보낼 데이터가 남아 있을 수 있다. 그래서 ACK와 FIN이 분리된다.

**TIME_WAIT 상태란?**

클라이언트는 마지막 ACK를 보낸 후 바로 종료하지 않고 **2MSL(Maximum Segment Lifetime)** 동안 대기한다. 자신의 ACK가 유실됐을 경우 서버의 FIN 재전송을 받아 다시 ACK를 보내기 위해서다.

> 이것이 Spring 애플리케이션에서 대량 트래픽 시 `TIME_WAIT` 소켓이 쌓이는 원인이다. 
> OS가 포트를 재사용하기 전까지 해당 포트를 점유한다.

**MSL:** 패킷이 네트워크에서 살 수 있는 최대 수명.
연결을 끊으려고 할 때, 마지막으로 주고받는 과정에서 발생할 수 있는 **최악의 시나리오**를 대비하기 위해서입니다.
1. **마지막 ACK의 왕복 시간:** 내가 보낸 마지막 확인 응답(ACK)이 상대방에게 가는 시간 (**1 MSL**)
2. **유실 시 재전송 시간:** 만약 내가 보낸 ACK가 유실되어 상대방이 다시 보낸 종료 신호(FIN)가 나에게 오는 시간 (**1 MSL**)
=> 2MSL

---

## 3. 공식 근거

|주장|출처|
|---|---|
|TCP Best-effort 위에서 신뢰성 제공|RFC 793 (1981), IETF — _"TCP is able to recover from errors that occur in the internet system"_|
|3-Way Handshake 설계 이유|RFC 793, Section 3.4|
|Fast Retransmit (3 Duplicate ACK)|RFC 5681, Section 2|
|Slow Start / Congestion Avoidance|RFC 5681, Section 3|
|ISN 임의값 사용|RFC 793, Section 3.3|
|TIME_WAIT 2MSL 대기|RFC 793, Section 3.5|

---

## 4. 이 설계를 이렇게 한 이유

### TCP가 얻는 것

- 애플리케이션 개발자가 네트워크 불안정성을 신경 쓰지 않아도 된다
- HTTP, SMTP, JDBC 등 상위 프로토콜이 "전송은 믿어도 된다"는 전제로 설계될 수 있다

### TCP가 잃는 것 (트레이드오프)

| 손실               | 구체적 상황                                                                         |
| ---------------- | ------------------------------------------------------------------------------ |
| **레이턴시**         | 3-Way Handshake로 인해 첫 데이터 전송 전 최소 1.5 RTT 소요. 한국→미국 RTT ~150ms라면 연결 수립에만 225ms |
| **HOL Blocking** | 앞 패킷이 유실되면 뒤 패킷이 다 와도 애플리케이션에 전달 안 됨. HTTP/1.1의 고질적 성능 문제 원인                   |
| **연결 유지 오버헤드**   | TIME_WAIT 상태 누적. 대량 단기 연결 서버에서 포트 고갈 발생 가능                                     |
| **실시간 통신 부적합**   | 게임, 화상통화처럼 오래된 데이터는 필요 없는 상황에서 재전송 대기가 오히려 독                                   |
|                  |                                                                                |

→ 이 트레이드오프를 해결하려고 나온 것이 **UDP 기반의 QUIC 프로토콜**이고, HTTP/3의 기반이 된다.

---

## 5. 이어지는 개념

| 순서    | 개념                                               | 이유                                                                                             |
| ----- | ------------------------------------------------ | ---------------------------------------------------------------------------------------------- |
| **1** | **HTTP/1.1 — Keep-Alive와 Persistent Connection** | TCP 연결 비용이 크기 때문에 HTTP가 어떻게 연결을 재사용하는지가 바로 다음 질문이 된다. TCP를 모르면 왜 Keep-Alive가 존재하는지 이해 불가.      |
| **2** | **TLS/SSL Handshake**                            | HTTPS는 TCP 3-Way Handshake 위에 TLS Handshake가 추가된다. TCP 연결 흐름을 알아야 전체 HTTPS 연결 수립 비용을 계산할 수 있다. |
| **3** | **HTTP/2 멀티플렉싱**                                 | TCP의 HOL Blocking 문제를 HTTP 레이어에서 어떻게 우회했는지. TCP 한계를 알아야 HTTP/2 설계 이유가 보인다.                     |
| **4** | **QUIC / HTTP/3**                                | TCP의 근본적 한계(HOL Blocking, Handshake 비용)를 UDP 위에서 재설계한 프로토콜. TCP 트레이드오프를 이해한 뒤에 읽어야 의미가 있다.     |
| **5** | **Spring의 커넥션 풀 (HikariCP, HTTP Client Pool)**   | TCP 연결 수립 비용이 크기 때문에 왜 Spring이 DB/HTTP 연결을 풀링하는지 실무 맥락으로 연결된다.                                 |
# TLS/SSL Handshake
## 1. Problem First
### HTTP는 왜 도청에 무력한가

TCP는 데이터가 **순서대로, 빠짐없이** 도착함을 보장한다. 하지만 그 데이터가 **누구에게나 읽힌다는 사실은 보장하지 않는다.**

HTTP 요청이 TCP 위에서 평문으로 이동하는 구조를 보자.

```
[내 브라우저] ──패킷──▶ [공유기] ──패킷──▶ [ISP 라우터] ──▶ ... ──▶ [서버]
                            ↑
                     이 구간 어디서든
                     패킷 캡처 가능
```

실제 캡처된 HTTP 패킷의 내용:

```
POST /login HTTP/1.1
Host: mybank.com
Content-Type: application/x-www-form-urlencoded

username=kim&password=mypassword1234   ← 평문 그대로 노출
```

같은 Wi-Fi에 있는 누군가가 Wireshark를 켜면 위 내용을 그대로 볼 수 있다. 이것이 공공 Wi-Fi가 위험한 이유다.

### 암호화만 하면 해결되는가 — 더 근본적인 문제

단순히 데이터를 암호화하는 것만으로는 충분하지 않다. 
암호화에는 **키(key)** 가 필요하고, 그 키를 어떻게 공유하느냐는 별개의 문제다.

```
나: "앞으로 이 키로 암호화해서 통신하자" ──▶ 서버
                                    ↑
                             이 키 전달 구간도
                             도청 가능
```

키를 평문으로 보내면 공격자도 키를 얻는다. 그러면 암호화가 무의미해진다. 
이것이 **키 배송 문제(Key Distribution Problem)** 다.

### 서버 신원 확인 문제

설령 암호화에 성공했어도, **내가 진짜 서버와 통신하고 있다는 걸 어떻게 아는가.**

```
[내 브라우저] ──▶ [공격자 서버] ──▶ [진짜 서버]
                       ↑
               "나 mybank.com이야" 라고
               거짓말하는 중간자
               (MITM, Man-In-The-Middle)
```

공격자가 진짜 서버인 척 중간에 끼어들면, 암호화가 돼 있어도 공격자와 암호화된 채널을 맺는 꼴이 된다.

TLS는 이 세 가지 문제를 **한 번의 Handshake** 로 해결한다.

- 키 배송 문제 → **비대칭 암호화**
- 서버 신원 확인 → **인증서(Certificate) + CA**
- 이후 통신 암호화 → **대칭 암호화**

---

## 2. Mechanics
### 전제 지식 — 두 가지 암호화 방식
TLS Handshake를 이해하려면 두 암호화 방식의 특성 차이를 먼저 알아야 한다.

|         | 대칭 암호화               | 비대칭 암호화                    |
| ------- | -------------------- | -------------------------- |
| 키 구성    | 암호화 = 복호화 = **같은 키** | **공개키**로 암호화, **개인키**로 복호화 |
| 속도      | 빠름                   | 느림 (약 100~1000배)           |
| 문제점     | 키를 어떻게 안전하게 공유하는가    | 속도가 느려 대량 데이터에 부적합         |
| 대표 알고리즘 | AES                  | RSA, ECDH                  |
TLS의 설계 핵심은 이렇다.

> **비대칭 암호화로 키를 안전하게 공유하고, 이후 실제 데이터는 대칭 암호화로 빠르게 통신한다.**
---
### 2-1. TLS 1.2 Handshake 전체 흐름
```
Client                                      Server
  │                                            │
  │──── ① ClientHello ─────────────────────▶ │
  │                                            │
  │ ◀── ② ServerHello ──────────────────────  │
  │ ◀── ③ Certificate ──────────────────────  │
  │ ◀── ④ ServerKeyExchange ────────────────  │
  │ ◀── ⑤ ServerHelloDone ─────────────────  │
  │                                            │
  │──── ⑥ ClientKeyExchange ───────────────▶ │
  │──── ⑦ ChangeCipherSpec ────────────────▶ │
  │──── ⑧ Finished ────────────────────────▶ │
  │                                            │
  │ ◀── ⑨ ChangeCipherSpec ─────────────────  │
  │ ◀── ⑩ Finished ────────────────────────   │
  │                                            │
  │══════ 이후 모든 통신: 대칭 암호화 ══════════│
```

TCP 3-Way Handshake 완료 후, TLS Handshake가 시작된다. 
HTTPS 연결에는 **TCP RTT 1.5회 + TLS RTT 2회** 가 필요하다.

각 단계를 내부 동작까지 파고든다.

---

### ① ClientHello

클라이언트가 서버에게 자신이 지원하는 것들을 알린다.

```
ClientHello {
  client_version: TLS 1.2          // 지원하는 최고 TLS 버전
  client_random: [32 bytes 난수]    // 나중에 세션 키 생성에 사용
  session_id: []                    // 세션 재사용 시 이전 ID
  cipher_suites: [                  // 지원하는 암호화 알고리즘 목록
    TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
    TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
    ...
  ]
  compression_methods: [null]
}
```

`cipher_suite` 하나를 분해하면:

```
TLS _ ECDHE _ RSA _ WITH _ AES_256_GCM _ SHA384
 │      │      │                                              │               │
 │      │      │                                              │               └─ MAC 알고리즘 (무결성 검증)
 │      │      │                                              └─ 대칭 암호화 알고리즘 + 모드 (데이터 암호화)
 │      │      └─ 인증 알고리즘 (인증서 서명 방식)
 │      └─ 키 교환 알고리즘 (세션 키를 어떻게 공유할 것인가)
 └─ 프로토콜
```

---

### ② ServerHello

서버가 클라이언트의 목록 중 하나를 **선택**해 응답한다.

```
ServerHello {
  server_version: TLS 1.2
  server_random: [32 bytes 난수]    // 클라이언트 난수와 함께 세션 키 재료
  session_id: [새로 생성된 ID]
  cipher_suite: TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384  // 선택된 하나
}
```

---

### ③ Certificate — 서버 신원 증명

서버가 자신의 **인증서(X.509 Certificate)** 를 보낸다.

인증서 내부 구조:

```
Certificate {
  subject:    "CN=*.google.com, O=Google LLC"   // 이 인증서의 주인
  issuer:     "CN=GTS CA 1C3, O=Google Trust Services LLC"  // 발급한 CA
  validity: {
    not_before: 2024-01-01
    not_after:  2024-12-31
  }
  public_key: [서버의 공개키]       // 클라이언트가 암호화에 사용할 키
  signature:  [CA의 개인키로 서명]   // CA가 "이 인증서 내용을 보증한다"
}
```

**클라이언트는 이 인증서를 어떻게 신뢰하는가 — CA 체인**

```
[내 브라우저/OS]
      │
      │  신뢰 저장소에 미리 저장된
      ▼
[Root CA] ──개인키로 서명──▶ [Intermediate CA] ──개인키로 서명──▶ [서버 인증서]
 (DigiCert)                                                      (GTS CA 1C3)                        (*.google.com)
      ↑
이 Root CA는 OS/브라우저 설치 시
이미 신뢰 목록에 들어있다
```

검증 과정:

```
1. 서버 인증서의 서명을 Intermediate CA의 공개키로 검증
2. Intermediate CA 인증서의 서명을 Root CA의 공개키로 검증
3. Root CA가 내 신뢰 저장소에 있는가 확인
4. 인증서의 CN(Common Name)이 접속한 도메인과 일치하는가 확인
5. 인증서 유효기간 확인
6. 인증서 폐기 여부 확인 (CRL 또는 OCSP)
```

이 체인 검증이 실패하면 브라우저가 "이 연결은 안전하지 않습니다" 경고를 띄운다.

---

### ④ ServerKeyExchange — 키 교환 재료 전달

`ECDHE` (Elliptic Curve Diffie-Hellman Ephemeral) 키 교환을 쓰는 경우, 서버가 DH 파라미터를 보낸다.

**Diffie-Hellman 키 교환의 핵심 아이디어**

```
공개 파라미터: g=5, p=23 (실제로는 훨씬 큰 수)

서버: 개인값 b=6 선택
      B = g^b mod p = 5^6 mod 23 = 8  ──▶ 클라이언트에게 B=8 전송

클라이언트: 개인값 a=4 선택
            A = g^a mod p = 5^4 mod 23 = 4  ──▶ 서버에게 A=4 전송

서버 계산:   S = A^b mod p = 4^6 mod 23 = 2
클라이언트 계산: S = B^a mod p = 8^4 mod 23 = 2

둘 다 S=2 도출 ← 이것이 공유 비밀값(Pre-Master Secret)이 된다
도청자는 A=4, B=8만 알고 a, b를 모르므로 S를 계산할 수 없다
```

실제로는 이산 대수 문제의 어려움에 기반한다. `g^a mod p`에서 `a`를 역산하는 것은 계산적으로 불가능에 가깝다.

---
### ⑥ ClientKeyExchange + 세션 키 생성

양쪽이 Pre-Master Secret을 가졌으면, 아래 재료로 최종 세션 키를 생성한다.

```
Master Secret = PRF(
    Pre-Master Secret,   // DH로 공유한 비밀값
    "master secret",     // 고정 레이블
    client_random        // ① ClientHello의 난수
    + server_random      // ② ServerHello의 난수
)

// PRF = Pseudo-Random Function (SHA-256 기반)
```

이 Master Secret에서 실제 사용할 키들이 파생된다.

```
Key Material = PRF(Master Secret, "key expansion", ...)
                    │
          ┌─────────┴──────────┐
          ▼                    ▼
  client_write_key       server_write_key    ← 대칭 암호화 키 (AES)
  client_write_MAC_key   server_write_MAC_key ← MAC 키 (무결성)
  client_write_IV        server_write_IV      ← 초기화 벡터
```

난수 두 개를 섞는 이유: 서버나 클라이언트 중 한 쪽의 난수 생성이 취약하더라도 나머지 하나가 보완하게 하기 위해서다.

---

### ⑦⑧⑨⑩ ChangeCipherSpec + Finished

```
ChangeCipherSpec: "지금부터 우리가 협상한 암호화로 전환할게"

Finished: PRF(Master Secret, "client finished", handshake_hash)
          ↑
  지금까지 주고받은 모든 Handshake 메시지의 해시값을 포함
  → 중간자가 Handshake 메시지를 변조했다면 이 값이 맞지 않아 탐지됨
```

양쪽 모두 Finished 검증에 성공하면 Handshake 완료. 이후 모든 통신은 AES-256-GCM으로 암호화된다.

---

### 2-2. TLS 1.3 — 무엇이 달라졌나

TLS 1.2는 2회의 RTT가 필요했다. TLS 1.3(RFC 8446, 2018)은 이것을 **1-RTT** 로 줄였다.

```
TLS 1.2: ClientHello → ServerHello+Cert+ServerKeyExchange → ClientKeyExchange → Finished
         ←──── RTT 1 ────▶←──────────── RTT 2 ──────────▶

TLS 1.3: ClientHello (+ 키 교환 재료 동시 전송)
              → ServerHello (+ 키 교환 재료 + 인증서 + Finished 동시 전송)
                   → Client Finished
         ←──────────────── RTT 1 ──────────────────────▶
```

TLS 1.3이 1-RTT로 줄일 수 있었던 이유:

1. **cipher suite 협상 제거** — 취약한 알고리즘(RSA 키 교환, RC4, MD5 등)을 아예 스펙에서 제거. 협상할 필요 없이 남은 것들 중 쓰면 된다
2. **ClientHello에 키 교환 재료 동봉** — 어차피 ECDHE만 쓰니까 첫 메시지에 DH 파라미터를 바로 보낼 수 있다

**0-RTT (Early Data)**

재연결 시 이전 세션의 키를 재사용해 첫 번째 패킷부터 암호화된 데이터를 보낼 수 있다.

```
Client                    Server
  │── ClientHello ──────▶ │
  │── [암호화된 요청] ────▶ │  ← 0-RTT, Handshake 완료 전에 데이터 전송
  │                        │
  │ ◀── ServerHello ─────  │
  │ ◀── Finished ────────  │
```

단, 0-RTT는 **재전송 공격(Replay Attack)** 에 취약하다. 공격자가 캡처한 패킷을 그대로 재전송하면 서버가 동일한 요청을 두 번 처리할 수 있다. 멱등성이 보장되는 GET 요청에만 허용해야 한다.

> RFC 8446, Section 2.3: _"0-RTT data is not forward secret... and is subject to replay attack."_ (0-RTT 데이터는 전방 비밀성을 갖지 않으며 재전송 공격의 대상이 될 수 있다.)

---

### 2-3. Spring/Java 애플리케이션에서의 TLS

Java에서 TLS는 **JSSE(Java Secure Socket Extension)** 가 처리한다.

```java
// Spring Boot는 application.yml 설정만으로 TLS 활성화
// server:
//   ssl:
//     key-store: classpath:keystore.p12
//     key-store-password: secret
//     key-store-type: PKCS12

// 내부적으로 이 흐름으로 동작:
SSLContext sslContext = SSLContext.getInstance("TLS");
sslContext.init(keyManagers, trustManagers, secureRandom);

SSLServerSocketFactory factory = sslContext.getServerSocketFactory();
SSLServerSocket serverSocket = (SSLServerSocket) factory.createServerSocket(443);

// 클라이언트 연결 시 자동으로 TLS Handshake 수행
SSLSocket clientSocket = (SSLSocket) serverSocket.accept();
clientSocket.startHandshake();  // ← 여기서 위의 Handshake 전체가 일어남
```

Spring이 RestTemplate이나 WebClient로 외부 HTTPS를 호출할 때는 JVM의 **truststore** 에 해당 서버의 CA가 있어야 한다. 사내 자체 서명 인증서를 쓰는 서버에 연결할 때 `SSLHandshakeException`이 나는 이유가 이것이다.

```java
// 흔히 보는 에러
javax.net.ssl.SSLHandshakeException: 
  PKIX path building failed: 
  unable to find valid certification path to requested target
// → 이 서버의 CA가 JVM truststore에 없음
```

---

## 3. 공식 근거

|주장|출처|
|---|---|
|TLS 1.2 Handshake 전체 절차|RFC 5246, Section 7.3|
|TLS 1.3 1-RTT 및 0-RTT 정의|RFC 8446, Section 2, 2.3|
|X.509 인증서 구조|RFC 5280|
|Diffie-Hellman 키 교환|RFC 2631|
|ECDHE 알고리즘|RFC 8422|
|0-RTT Replay Attack 경고|RFC 8446, Section 2.3|
|Master Secret 도출 PRF|RFC 5246, Section 8.1|
|JSSE 구현|Oracle Java SE 공식 문서 — _JSSE Reference Guide_|
|TLS 1.2에서 취약 cipher suite 제거|RFC 8446, Appendix B.4 (TLS 1.3에서 제거된 목록 명시)|

---

## 4. 이 설계를 이렇게 한 이유

### 얻는 것

- **기밀성**: 대칭키를 비대칭 방식으로 공유해 도청 불가
- **무결성**: MAC으로 데이터 변조 탐지
- **인증**: CA 체인으로 서버 신원 보증
- **전방 비밀성(Forward Secrecy)**: ECDHE는 세션마다 새 키 생성 → 과거 트래픽 캡처본이 있어도 나중에 서버 개인키가 유출되더라도 복호화 불가

### 잃는 것 / 트레이드오프

| 손실               | 구체적 상황                                                                                                             |
| ---------------- | ------------------------------------------------------------------------------------------------------------------ |
| **레이턴시**         | TLS 1.2: TCP 1.5RTT + TLS 2RTT = 3.5RTT. 한국→미국(150ms RTT)이면 연결 수립에만 **525ms**. TLS 1.3은 2.5RTT로 개선                 |
| **CPU 비용**       | 비대칭 암호화(RSA/ECDH)는 무겁다. 대규모 트래픽에서 TLS termination을 별도 L7 로드밸런서(ALB, Nginx)에 위임하는 이유                                |
| **CA에 대한 신뢰 의존** | CA가 해킹되거나 신뢰를 남용하면 전체 체계가 흔들린다. 2011년 DigiNotar CA 해킹 사건이 실제 사례. 이를 보완하기 위해 **Certificate Transparency(CT)** 가 도입됨 |
| **인증서 관리 부담**    | 유효기간(보통 90일~1년) 만료 시 갱신 필요. 갱신 실패 시 서비스 전면 장애. AWS ACM, Let's Encrypt 자동 갱신으로 보완                                   |
| **0-RTT의 보안 약화** | 재전송 공격 가능. 비멱등 요청(POST, 결제 등)에 허용 시 심각한 보안 문제                                                                      |

---

## 5. 이어지는 개념

|순서|개념|이유|
|---|---|---|
|**1**|**HTTP/2와 ALPN**|TLS Handshake 중 `ClientHello`의 확장 필드에서 HTTP/2 사용 여부를 협상한다(ALPN 확장). TLS를 모르면 HTTP/2가 어떻게 활성화되는지 설명이 불가|
|**2**|**인증서 관리 — OCSP, CRL, Certificate Transparency**|CA 체인을 이해했으면 "인증서가 폐기됐는지 어떻게 확인하는가" 가 바로 다음 실무 문제. AWS ACM 자동 갱신과도 직결|
|**3**|**AWS ALB의 TLS Termination**|실무에서 Spring 애플리케이션은 직접 TLS를 처리하지 않고 ALB에 위임한다. "왜 위임하는가, 내부 트래픽은 어떻게 처리되는가" 가 인프라 설계의 핵심|
|**4**|**mTLS (Mutual TLS)**|서버만 인증서를 제시하는 것이 아니라 클라이언트도 제시한다. MSA 환경에서 서비스 간 인증에 사용. 단방향 TLS를 완전히 이해한 뒤에 봐야 한다|