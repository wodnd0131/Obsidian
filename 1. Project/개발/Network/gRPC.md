# gRPC — 내부 동작과 설계 근거

---

## 1. Problem First

### 시나리오 A: REST로 MSA 내부 통신을 하던 시절

```java
// Order 서비스 → Inventory 서비스 호출
@Service
public class OrderService {
    public void createOrder(OrderRequest request) {
        // 1. 객체 → JSON 직렬화
        String json = objectMapper.writeValueAsString(request);
        
        // 2. HTTP/1.1 요청
        ResponseEntity<String> response = restTemplate.postForEntity(
            "http://inventory-service/api/v1/check-stock",
            json,
            String.class
        );
        
        // 3. JSON → 객체 역직렬화
        InventoryResponse result = objectMapper.readValue(
            response.getBody(),
            InventoryResponse.class
        );
    }
}
```

**이 구조의 문제를 구체적으로 짚는다.**

**문제 1 — 계약이 없다:**

```
Order 서비스 개발자: "stock 필드는 Integer야"
Inventory 서비스 개발자: "아, 나는 Long으로 바꿨는데"

→ 런타임에 역직렬화 실패. 컴파일 타임에 잡을 방법이 없다.
```

**문제 2 — HTTP/1.1의 구조적 한계:**

```
요청 1: [TCP 연결] → [요청] → [응답] → [연결 유지 or 종료]
요청 2: [TCP 연결 재사용 or 새 연결] → [요청] → [응답]

Head-of-Line Blocking:
요청 1이 느리면 같은 연결의 요청 2는 무조건 기다려야 한다.
```

**문제 3 — 스트리밍이 불가능하다:**

```
// 실시간 재고 변동을 주시해야 하는 경우
// REST로는 폴링밖에 방법이 없다
while (true) {
    Thread.sleep(1000);
    restTemplate.getForEntity("/inventory/status", ...); // 매초 HTTP 요청
}
```

---

### 시나리오 B: 그래서 gRPC 없이 바이너리 통신을 직접 구현하면

```java
// "우리끼리 약속한 바이너리 포맷"
byte[] payload = new byte[16];
ByteBuffer.wrap(payload)
    .putLong(orderId)
    .putInt(quantity)
    .putInt(warehouseId);

socket.write(payload);

// 6개월 후: warehouseId를 Long으로 바꿔야 해서
// 포맷 변경 → 모든 클라이언트 동시 배포 필요
// 하나라도 버전이 다르면 데이터 깨짐
```

**직접 만든 바이너리 프로토콜은 버전 관리가 없다.** 스키마 변경이 전체 시스템 동시 배포를 강제한다.

---

## 2. Mechanics

### 2-1. gRPC 전체 스택 구조

```
┌─────────────────────────────────────┐
│         애플리케이션 코드             │  ← 개발자가 작성하는 영역
├─────────────────────────────────────┤
│         Generated Stub              │  ← protoc가 생성
│   (BlockingStub / AsyncStub)        │
├─────────────────────────────────────┤
│         gRPC Core (Channel)         │  ← 로드밸런싱, 인터셉터, 재시도
├─────────────────────────────────────┤
│         Protocol Buffers            │  ← 직렬화/역직렬화
├─────────────────────────────────────┤
│            HTTP/2                   │  ← 전송 계층
├─────────────────────────────────────┤
│         TLS (선택)                  │
├─────────────────────────────────────┤
│              TCP                    │
└─────────────────────────────────────┘
```

각 계층이 어떤 문제를 해결하는지 순서대로 내려간다.

---

### 2-2. HTTP/2 — gRPC가 HTTP/1.1을 버린 이유

**HTTP/1.1과 HTTP/2의 핵심 차이: 멀티플렉싱**

```
HTTP/1.1:
TCP 연결 1개 = 요청 1개 처리 (동시에)

[연결]──[요청A]──[응답A]──[요청B]──[응답B]──▶

Head-of-Line Blocking:
응답A가 느리면 요청B는 무조건 대기

---

HTTP/2:
TCP 연결 1개 = 스트림 N개 동시 처리

[연결]──[스트림1: 요청A ──────────── 응답A]──▶
        [스트림2: 요청B ── 응답B          ]──▶
        [스트림3: 요청C ────── 응답C      ]──▶
```

**HTTP/2의 프레임 구조:**

```
┌─────────────────────────────────┐
│  Length (24bit)                 │  페이로드 크기
│  Type   (8bit)                  │  DATA / HEADERS / SETTINGS ...
│  Flags  (8bit)                  │  END_STREAM, END_HEADERS ...
│  Stream Identifier (31bit)      │  어느 스트림인지
├─────────────────────────────────┤
│  Payload                        │
└─────────────────────────────────┘
```

**gRPC 메시지가 HTTP/2 위에서 전송되는 방식:**

```
gRPC 메시지 하나 = HTTP/2 DATA 프레임 하나 이상

┌──────────────────────────────────────────┐
│ Compressed-Flag (1byte)                  │  0: 비압축, 1: 압축
│ Message-Length  (4byte)                  │  Protobuf 바이트 길이
│ Message         (N byte)                 │  실제 Protobuf 바이트
└──────────────────────────────────────────┘
```

> 📎 **근거:** RFC 7540 — _"Hypertext Transfer Protocol Version 2 (HTTP/2)"_, §5 (스트림), §6 (프레임) gRPC 공식 스펙 — _"gRPC over HTTP2"_, github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md

---

### 2-3. Protocol Buffers — 필드 번호 기반 인코딩

이전 직렬화 편에서 "필드 이름 대신 번호를 전송한다"고 했다. **왜 이게 하위 호환성을 보장하는가:**

```protobuf
// v1 스키마
message InventoryRequest {
    int64 product_id = 1;
    int32 quantity   = 2;
}

// v2 스키마 — 필드 추가
message InventoryRequest {
    int64  product_id   = 1;  // 번호 그대로
    int32  quantity     = 2;  // 번호 그대로
    string warehouse_id = 3;  // 새 필드 추가
}
```

```
v1 클라이언트가 보내는 바이트:
08 [product_id 값] 10 [quantity 값]
(필드 번호 1, 2만 포함)

v2 서버가 받았을 때:
→ 필드 3 (warehouse_id) 없음 → 기본값(빈 문자열)으로 처리
→ 정상 동작. 재배포 없이 호환.
```

**Tag 인코딩 방식 (Wire Type):**

```
Tag = (field_number << 3) | wire_type

wire_type:
0 = Varint        (int32, int64, bool, enum)
1 = 64-bit        (fixed64, double)
2 = Length-delimited (string, bytes, 내장 message)
5 = 32-bit        (fixed32, float)

예시: field_number=1, wire_type=0 (varint)
→ Tag = (1 << 3) | 0 = 0x08
```

> 📎 **근거:** Protocol Buffers 공식 문서 — _"Encoding"_, developers.google.com/protocol-buffers/docs/encoding

---

### 2-4. 4가지 통신 패턴 — 내부 스트림 동작

```protobuf
service InventoryService {
    // 1. Unary
    rpc CheckStock(StockRequest) returns (StockResponse);

    // 2. Server Streaming
    rpc WatchStock(StockRequest) returns (stream StockResponse);

    // 3. Client Streaming
    rpc BulkUpdate(stream UpdateRequest) returns (UpdateResponse);

    // 4. Bidirectional Streaming
    rpc SyncInventory(stream SyncRequest) returns (stream SyncResponse);
}
```

**각 패턴이 HTTP/2 스트림과 어떻게 대응되는가:**

```
1. Unary:
클라이언트: HEADERS(END_HEADERS) → DATA(END_STREAM)
서버:       HEADERS(END_HEADERS) → DATA(END_STREAM)

2. Server Streaming:
클라이언트: HEADERS → DATA(END_STREAM)
서버:       HEADERS → DATA → DATA → DATA → DATA(END_STREAM)
                      ↑ 같은 스트림 ID로 계속 전송

3. Bidirectional Streaming:
클라이언트: HEADERS → DATA → DATA → DATA(END_STREAM)
서버:       HEADERS → DATA → DATA → DATA(END_STREAM)
            ↑ 동시에, 순서 무관하게 전송 가능
```

**Java에서 Bidirectional Streaming 구현:**

```java
// 서버 구현
@Override
public StreamObserver<SyncRequest> syncInventory(
        StreamObserver<SyncResponse> responseObserver) {
    
    return new StreamObserver<SyncRequest>() {
        @Override
        public void onNext(SyncRequest request) {
            // 클라이언트로부터 메시지 수신
            SyncResponse response = process(request);
            responseObserver.onNext(response); // 즉시 응답 가능
        }

        @Override
        public void onError(Throwable t) {
            log.error("Stream error", t);
        }

        @Override
        public void onCompleted() {
            responseObserver.onCompleted();
        }
    };
}
```

---

### 2-5. Spring에서 gRPC 동작 흐름

```
클라이언트 요청
    ↓
ManagedChannel (연결 풀 관리)
    ↓
ClientInterceptor (인증, 로깅, 트레이싱)
    ↓
Generated Stub → Protobuf 직렬화
    ↓
HTTP/2 프레임 전송
    ↓
[네트워크]
    ↓
ServerInterceptor (인증 검증, 메타데이터 추출)
    ↓
Generated Service Base → Protobuf 역직렬화
    ↓
실제 서비스 구현체
```

**Channel과 Stub의 관계 — 실무에서 자주 틀리는 부분:**

```java
// WRONG: 매 요청마다 채널 생성
public InventoryResponse checkStock(StockRequest request) {
    ManagedChannel channel = ManagedChannelBuilder
        .forAddress("inventory-service", 9090)
        .build(); // 매번 TCP + TLS 핸드셰이크 비용 발생
    
    InventoryServiceGrpc.InventoryServiceBlockingStub stub =
        InventoryServiceGrpc.newBlockingStub(channel);
    
    return stub.checkStock(request);
}

// RIGHT: 채널은 애플리케이션 생명주기 동안 재사용
@Configuration
public class GrpcConfig {
    @Bean
    public ManagedChannel inventoryChannel() {
        return ManagedChannelBuilder
            .forAddress("inventory-service", 9090)
            .build();
    }
    
    @Bean
    public InventoryServiceGrpc.InventoryServiceBlockingStub inventoryStub(
            ManagedChannel channel) {
        return InventoryServiceGrpc.newBlockingStub(channel);
        // Stub은 채널을 공유, 스레드 안전
    }
}
```

> 📎 **근거:** gRPC Java 공식 문서 — _"Stub, Channel, and Server"_, grpc.io/docs/languages/java/basics

---

### 2-6. 에러 처리 — HTTP 상태코드 vs gRPC Status

```java
// HTTP REST: 상태코드가 의미를 담는다
// 200, 404, 500, 409 ...

// gRPC: Status Code + 구조화된 에러
try {
    InventoryResponse response = stub.checkStock(request);
} catch (StatusRuntimeException e) {
    Status status = e.getStatus();
    
    switch (status.getCode()) {
        case NOT_FOUND:      // 재고 항목 없음
        case UNAVAILABLE:    // 서버 일시 불가 → 재시도 가능
        case RESOURCE_EXHAUSTED: // 재고 부족
        case INVALID_ARGUMENT:   // 잘못된 요청
    }
    
    // 구조화된 에러 상세 정보
    ErrorInfo errorInfo = StatusProto.fromThrowable(e)
        .getDetailsList()
        .get(0)
        .unpack(ErrorInfo.class);
}
```

**gRPC Status Code는 16개로 정의되어 있다:**

|Code|의미|HTTP 대응|
|---|---|---|
|`OK`|성공|200|
|`NOT_FOUND`|리소스 없음|404|
|`UNAVAILABLE`|서버 일시 불가, **재시도 가능**|503|
|`DEADLINE_EXCEEDED`|타임아웃|504|
|`UNAUTHENTICATED`|인증 실패|401|
|`PERMISSION_DENIED`|권한 없음|403|
|`RESOURCE_EXHAUSTED`|할당량 초과|429|

> 📎 **근거:** gRPC 공식 스펙 — _"Status Codes and their use in gRPC"_, github.com/grpc/grpc/blob/master/doc/statuscodes.md

---

## 3. 공식 근거 정리

|주장|출처|
|---|---|
|HTTP/2 멀티플렉싱, 프레임 구조|RFC 7540 §5, §6|
|gRPC over HTTP/2 프레이밍|gRPC 공식 스펙 PROTOCOL-HTTP2.md|
|Protobuf Wire Type 인코딩|Protocol Buffers 공식 문서 — Encoding|
|Protobuf 하위 호환성 (필드 추가/삭제 규칙)|Protocol Buffers 공식 문서 — Language Guide §Updating|
|gRPC 4가지 통신 패턴|gRPC 공식 문서 — Core concepts|
|gRPC Status Code 정의|gRPC 공식 스펙 statuscodes.md|
|Channel/Stub 생명주기|gRPC Java 공식 문서|

---

## 4. 이 설계를 이렇게 한 이유

### HTTP/2를 선택한 이유 — 그리고 잃는 것

**이점:** 멀티플렉싱으로 Head-of-Line Blocking 제거, 헤더 압축(HPACK), 스트리밍.

**대가:** HTTP/2는 바이너리 프레이밍 프로토콜이다.

```
curl로 직접 호출 불가.
브라우저에서 직접 호출 불가 (gRPC-Web 별도 필요).
API Gateway, 일부 로드밸런서가 HTTP/2를 제대로 지원 안 하는 경우 있음.
```

**언제 REST를 유지해야 하는가:**

- 외부에 공개하는 Public API — 클라이언트를 통제할 수 없다
- 브라우저가 직접 호출하는 API — gRPC-Web 레이어 추가 비용 발생
- 팀이 Protobuf 스키마 관리 프로세스를 갖추지 못한 경우

### Protobuf 스키마 강제의 이점과 대가

**이점:** 계약이 코드다. 스키마가 맞지 않으면 컴파일이 안 된다.

**대가:**

```
스키마 변경 → protoc 재실행 → 코드 재생성 → 재배포
팀 간 스키마 파일 공유 프로세스 필요 (Proto 파일 저장소 관리)
JSON처럼 눈으로 바로 읽을 수 없어 디버깅 비용 증가
```

**필드 삭제 시 반드시 지켜야 하는 규칙:**

```protobuf
// WRONG: 필드 번호 2를 재사용
message Request {
    int64 product_id = 1;
    // int32 quantity = 2;  삭제
    string new_field = 2;  // 번호 재사용 → 구버전 클라이언트 데이터 오염
}

// RIGHT: 삭제된 번호는 reserved로 봉인
message Request {
    int64 product_id = 1;
    reserved 2;
    reserved "quantity";
    string new_field = 3;  // 새 번호 사용
}
```

> 📎 **근거:** Protocol Buffers 공식 문서 — _"Updating a Message Type"_

---

## 5. 이어지는 개념

|순서|개념|이유|
|---|---|---|
|1|**Service Mesh (Istio/Envoy)**|gRPC 채널 레벨 로드밸런싱은 L7이다 — 쿠버네티스 환경에서 gRPC를 제대로 쓰려면 Envoy 프록시가 필요한 이유를 여기서 만난다|
|2|**Circuit Breaker (Resilience4j)**|`UNAVAILABLE` 상태가 연속으로 오면 채널을 끊어야 한다 — gRPC 재시도 정책만으로 커버 안 되는 장애 전파 시나리오|
|3|**분산 트레이싱 (OpenTelemetry)**|gRPC 인터셉터로 트레이스를 전파하는 구조 — MSA에서 호출 체인 디버깅의 핵심|
|4|**HTTP/3 (QUIC)**|HTTP/2도 TCP 레벨 Head-of-Line Blocking은 해결 못 한다 — QUIC이 이 문제를 어떻게 해결하는지, gRPC의 다음 전송 계층 후보|