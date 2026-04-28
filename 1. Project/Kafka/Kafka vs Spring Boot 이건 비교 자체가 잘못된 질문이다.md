
## 먼저 짚어야 할 것

이 두 개는 **같은 레이어에 있지 않다.** 비교 대상이 아니다.

| |Apache Kafka|Spring Boot|
|---|---|---|
|**분류**|분산 메시지 스트리밍 플랫폼|JVM 애플리케이션 프레임워크|
|**만든 곳**|LinkedIn → Apache Foundation|Pivotal → VMware (Spring 팀)|
|**핵심 역할**|메시지를 저장하고 전달하는 **인프라**|애플리케이션을 빠르게 만드는 **프레임워크**|
|**관계**|Spring Boot로 Kafka를 사용한다|Kafka는 Spring Boot의 의존 대상 중 하나|

자동차와 도로를 비교하는 것과 같다. 도로 위에서 자동차가 달린다.

---

그럼 질문 의도를 두 가지로 해석하고 둘 다 답한다.

**해석 A.** "Kafka가 뭔지 모르겠다, Spring Boot랑 뭐가 다른 거냐" **해석 B.** "메시지 처리를 할 때 Kafka를 써야 하냐, Spring 내부 기능(이벤트 등)으로 해결하냐"

---

## 해석 A: Kafka 자체가 뭔가

### Problem First

Spring Boot 애플리케이션 하나가 주문을 받아서 처리한다고 하자.

```
[클라이언트] → POST /order → [OrderService]
                                    ↓ (동기 호출)
                              [재고 차감]
                                    ↓
                              [결제 처리]
                                    ↓
                              [알림 발송]
                                    ↓
                              [포인트 적립]
```

**문제 1: 결합도** 결제 서버가 다운되면 주문 자체가 실패한다. 알림 서버가 느리면 주문 응답이 느려진다.

**문제 2: 처리량** 초당 1만 건 주문이 들어오면, 모든 후속 처리가 이 요청 스레드 안에서 동기로 일어난다.

**문제 3: 이벤트 재처리 불가** 알림 발송이 실패했다. 어디서부터 다시 처리해야 하는가? 로그 뒤지고 수동으로 재처리해야 한다.

**이게 근본적으로 문제인 이유:** 시스템 간 강결합. 하나가 죽으면 전파된다. 이벤트의 내구성이 없다.

### Kafka가 해결하는 것

```
[클라이언트] → POST /order → [OrderService] → "order-created" 토픽에 메시지 발행
                                    ↓ (즉시 응답)
                              
[재고 서비스]  ←── "order-created" 토픽 구독
[결제 서비스]  ←── "order-created" 토픽 구독   (각자 독립적으로 소비)
[알림 서비스]  ←── "order-created" 토픽 구독
[포인트 서비스] ←── "order-created" 토픽 구독
```

Kafka는 **메시지를 디스크에 저장**한다. 알림 서비스가 죽어있어도 메시지는 사라지지 않는다. 살아나면 이어서 처리한다.

### Kafka 내부 동작 (Mechanics)

```
Topic: order-created
├── Partition 0: [msg1][msg2][msg5][msg8] ...
├── Partition 1: [msg3][msg6][msg9] ...
└── Partition 2: [msg4][msg7][msg10] ...
```

- **Partition**: 순서가 보장되는 단위. 파티션 내에서는 offset 순서가 보장된다
- **Offset**: 파티션 내 메시지 위치. Consumer가 어디까지 읽었는지 이 숫자로 추적한다
- **Consumer Group**: 같은 토픽을 여러 인스턴스가 나눠서 처리할 때 사용. 파티션 하나는 그룹 내 Consumer 하나에만 할당된다

메시지를 읽어도 **지우지 않는다.** `log.retention.ms` 설정 시간 동안 디스크에 유지된다. 이것이 재처리를 가능하게 한다.

> 공식 근거: [Apache Kafka Documentation — Core Concepts](https://kafka.apache.org/documentation/#intro_concepts_and_terms)

---

## 해석 B: Kafka vs Spring 내부 이벤트 기능

Spring에도 메시지/이벤트 처리 기능이 있다. 언제 뭘 쓰는가.

### Spring ApplicationEvent (프로세스 내 이벤트)

```java
// 발행
applicationEventPublisher.publishEvent(new OrderCreatedEvent(orderId));

// 구독
@EventListener
public void handleOrderCreated(OrderCreatedEvent event) { ... }

// 비동기로 만들려면
@Async
@EventListener
public void handleOrderCreated(OrderCreatedEvent event) { ... }
```

**한계:**

- 같은 JVM 프로세스 안에서만 동작한다
- 서버가 재시작되면 이벤트가 사라진다
- 다른 서비스(다른 프로세스)에 전달할 수 없다
- Consumer가 처리 실패하면 재처리 방법이 없다

### Spring + Kafka (프로세스 간 이벤트)

```java
// 발행
kafkaTemplate.send("order-created", orderId, orderEvent);

// 구독
@KafkaListener(topics = "order-created", groupId = "inventory-service")
public void handleOrderCreated(OrderCreatedEvent event) { ... }
```

**이점:**

- 서비스 간 통신. 완전히 다른 서버에 있는 서비스도 구독 가능
- 메시지 내구성. 서버 재시작해도 Kafka에 남아있음
- 실패 시 재처리. offset을 되돌려 다시 소비 가능

### 선택 기준

|상황|선택|
|---|---|
|같은 서비스 내 계층 간 이벤트|Spring ApplicationEvent|
|트랜잭션 내에서 처리 완결이 필요함|Spring ApplicationEvent (또는 동기 호출)|
|서비스 간 통신 (MSA)|Kafka|
|이벤트 유실이 절대 안 됨|Kafka|
|처리량이 매우 높음 (초당 수천 건 이상)|Kafka|
|이벤트 재처리, 감사 로그가 필요|Kafka|

---

## 이어지는 개념 (학습 순서)

1. **Kafka Producer/Consumer 내부 동작** — 지금 질문의 직접 후속. acks 설정, offset commit 전략, at-least-once vs exactly-once가 실무에서 바로 만나는 문제다
    
2. **Spring Kafka (`spring-kafka`)** — Spring Boot에서 Kafka를 어떻게 쓰는지. `@KafkaListener`, `KafkaTemplate`, `SeekToCurrentErrorHandler` 설정이 핵심
    
3. **트랜잭션 아웃박스 패턴 (Transactional Outbox)** — DB 저장과 Kafka 발행을 원자적으로 처리해야 할 때. 이걸 모르면 메시지 유실/중복 발행 버그를 반드시 만난다
    
4. **Consumer Group과 파티션 리밸런싱** — 스케일 아웃할 때 파티션 재할당이 어떻게 일어나는지. 운영 중 장애 원인이 되는 포인트