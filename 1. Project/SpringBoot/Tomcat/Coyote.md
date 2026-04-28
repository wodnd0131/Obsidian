#SpringBoot/Tomcat 
# Coyote — Tomcat의 커넥터 모듈

## 1. Problem First

### HTTP 요청을 받는다는 것이 실제로 무슨 일인가

"클라이언트가 요청을 보낸다"는 것을 애플리케이션 개발자 관점에서 보면 `HttpServletRequest` 객체 하나가 뚝 생겨나는 것처럼 느껴진다. 하지만 그 사이에 실제로 일어나는 일은 이렇다.

```
클라이언트가 보내는 것 (실제 전기 신호 → 바이트):

47 45 54 20 2F 75 73 65 72 73 20 48 54 54 50 2F 31 2E 31 0D 0A
G  E  T  ' '/  u  s  e  r  s  ' ' H  T  T  P  /  1  .  1  \r \n
48 6F 73 74 3A 20 6C 6F 63 61 6C 68 6F 73 74 0D 0A
H  o  s  t  :  ' ' l  o  c  a  l  h  o  s  t  \r \n
...
```

이 바이트 스트림을 받아서 `HttpServletRequest` 객체로 만들기까지 누군가 해야 한다. 그것이 **Coyote**다.

---

## 2. Mechanics

### 2-1. Coyote의 정체

Coyote는 Tomcat의 **커넥터(Connector) 모듈**이다.

역할을 한 줄로 정의하면:

> **네트워크 바이트 스트림 → HttpServletRequest/Response 객체**

Catalina(서블릿 컨테이너)는 `HttpServletRequest`가 어떻게 만들어졌는지 모른다. 
Coyote가 만들어서 넘겨줄 뿐이다. 이 분리가 핵심이다.

```
[네트워크]                  [Coyote]                  [Catalina]
바이트 스트림    →    파싱 + 객체 생성    →    HttpServletRequest
                  (Coyote 책임)              (Catalina가 받아서 씀)
```

---

### 2-2. Coyote 내부 구조

```
Coyote
├── Endpoint      소켓 수준 처리 (TCP 연결 수락, 스레드 관리)
├── Processor     HTTP 프로토콜 파싱
└── Adapter       Catalina로 넘기는 연결 다리
```

각각 내려간다.

---

### 2-3. Endpoint — 소켓과 스레드 관리

Tomcat이 시작하면 Endpoint가 지정된 포트를 열고 연결을 기다린다.

```
Tomcat 시작
    │
    ▼
ServerSocket.bind(8080)    ← 포트 점유
    │
    ▼
Acceptor Thread (1개)      ← 연결 요청만 전담으로 수락
    │
    │  클라이언트 연결 요청 (TCP 3-Way Handshake)
    │  accept() → Socket 반환
    ▼
Worker Thread Pool         ← 실제 요청 처리 담당
(기본 max 200개)
```

**Acceptor Thread와 Worker Thread를 분리하는 이유:**

```
Acceptor가 요청 처리까지 하면:
연결 수락 → 처리(오래 걸림) → 그 동안 새 연결 수락 불가

분리하면:
Acceptor: 연결 수락만 → 즉시 Worker에게 넘김 → 다음 연결 수락
Worker:   실제 처리 담당 (시간이 걸려도 Acceptor에 영향 없음)
```

Tomcat이 지원하는 Endpoint 구현체:

```
NioEndpoint     Java NIO 기반 (기본값)
               논블로킹 I/O로 소켓 관리
               하나의 스레드가 여러 소켓 모니터링 가능

Nio2Endpoint    Java NIO2 (AIO) 기반
               비동기 I/O

AprEndpoint     Apache Portable Runtime 기반
               네이티브 라이브러리 사용, 고성능
```

Spring Boot 기본값은 `NioEndpoint`다.

---

### 2-4. Processor — HTTP 프로토콜 파싱

소켓에서 읽은 바이트 스트림을 HTTP 스펙(RFC 9112)에 따라 파싱한다.

```
읽어들인 바이트:
"GET /users/42 HTTP/1.1\r\n
Host: localhost:8080\r\n
Authorization: Bearer eyJ...\r\n
Content-Type: application/json\r\n
\r\n
{\"name\": \"kim\"}"

Processor가 파싱:

Request Line:  method=GET, uri=/users/42, protocol=HTTP/1.1
Headers:       Host=localhost:8080
               Authorization=Bearer eyJ...
               Content-Type=application/json
Body:          {"name": "kim"}  (Content-Length만큼 읽음)
```

파싱 결과로 Tomcat 내부 `Request` / `Response` 객체를 채운다. 아직 `HttpServletRequest`가 아니다. 
Tomcat 내부 표현이다.

**HTTP 버전별로 Processor 구현체가 다르다:**

```
Http11Processor   HTTP/1.1 파싱
Http2UpgradeHandler   HTTP/2 처리
```

HTTP/2는 바이너리 프레임 기반이라 파싱 방식이 완전히 다르다. Processor를 교체하는 것만으로 프로토콜 지원을 추가할 수 있다. Coyote 설계의 장점이다.

---

### 2-5. Adapter — Catalina로 넘기는 다리

Processor가 파싱을 마치면 Adapter가 Catalina로 넘긴다.

```java
// CoyoteAdapter.service() — 소스코드 기반
public void service(Request req, Response res) {

    // Tomcat 내부 Request → HttpServletRequest 래핑
    HttpServletRequest request = req.getNote(ADAPTER_NOTES);
    // (실제로는 Request 객체 자체가 HttpServletRequest를 구현)

    // Catalina로 위임
    connector.getService()
             .getContainer()  // Engine (Catalina)
             .getPipeline()
             .getFirst()
             .invoke(request, response);
}
```

이 시점에 비로소 **Catalina가 요청을 넘겨받는다.**

---

### 2-6. 전체 흐름

```
[클라이언트]
    │  TCP SYN
    ▼
[Endpoint — Acceptor Thread]
    │  accept() → Socket
    │  Worker Thread에 Socket 할당
    ▼
[Endpoint — Worker Thread]
    │  Socket에서 바이트 읽기
    ▼
[Processor]
    │  바이트 → HTTP 파싱 (RFC 9112)
    │  Tomcat 내부 Request/Response 객체 생성
    ▼
[Adapter]
    │  Catalina로 위임
    ▼
[Catalina]
    Filter → Servlet (DispatcherServlet)
```

---

## 3. 공식 근거

|주장|출처|
|---|---|
|TCP 소켓 연결 수립|RFC 793 (TCP)|
|HTTP/1.1 메시지 파싱 형식|RFC 9112 (HTTP/1.1 Message Syntax)|
|Tomcat Connector 구조|Apache Tomcat 공식 문서 — _Connectors_|
|NIO Endpoint 기본값|Apache Tomcat 공식 문서 — _The HTTP Connector_|

---

## 4. 이 설계를 이렇게 한 이유

### Coyote를 Catalina에서 분리한 이유

```
Coyote(네트워크) ↔ Catalina(서블릿) 분리

→ 프로토콜 교체가 가능해진다
  HTTP/1.1 → HTTP/2 → HTTP/3 (QUIC)
  Coyote의 Processor만 교체하면 됨
  Catalina(서블릿 컨테이너)는 변경 없음

→ Catalina 입장에서
  "어떻게 연결이 들어왔는지 모른다"
  "HttpServletRequest가 오면 처리할 뿐"
```

### 트레이드오프

|손실|구체적 상황|
|---|---|
|**Worker Thread 블로킹**|DB 조회 대기 중에도 Worker Thread가 점유된 상태. 스레드 200개가 모두 DB 응답 대기 중이면 새 요청을 못 받음. 이것이 WebFlux(논블로킹)가 나온 이유|
|**스레드당 메모리 비용**|각 Worker Thread는 기본 1MB 스택 메모리. 200개면 200MB가 스레드 스택에만 사용|
|**Keep-Alive 연결 점유**|HTTP Keep-Alive로 연결을 유지하면 해당 소켓을 Endpoint가 계속 관리해야 함. NIO로 완화되지만 완전한 해결은 아님|

---

## 5. 이어지는 개념

| 순서    | 개념                  | 이유                                                                   |
| ----- | ------------------- | -------------------------------------------------------------------- |
| **1** | **Catalina**        | Coyote가 만든 HttpServletRequest를 받아서 서블릿으로 전달하는 다음 단계. 서블릿 컨테이너의 실제 구현 |
| **2** | **NIO와 블로킹 I/O 차이** | Coyote가 왜 NIO를 쓰는지, 블로킹 방식과 무엇이 다른지. Worker Thread 문제의 근본 원인         |
| **3** | **WebFlux와 Netty**  | Tomcat+Coyote의 스레드 블로킹 한계를 해결하는 대안. Netty가 Coyote와 같은 역할을 논블로킹으로 처리  |
