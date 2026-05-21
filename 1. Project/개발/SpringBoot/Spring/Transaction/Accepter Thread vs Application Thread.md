- **JVM/OS 관점**: 셋 다 동일한 `java.lang.Thread`, 동일한 native thread(리눅스 LWP). 차이 없음
- **역할 관점**: Acceptor(연결 수락) / Poller(I/O 이벤트 감지) / Worker(요청 처리)로 분리
- **"애플리케이션 스레드"**: 네가 코드(`@Controller`, `@Service`)에서 실제로 마주치는 스레드 = **Worker 스레드** = 개발자가 `server.tomcat.threads.max`로 양 조절하는 풀

---

| 컴포넌트       | 역할                                   |
| ---------- | ------------------------------------ |
| Acceptor   | TCP 연결 수락                            |
| **Poller** | **idle 연결 중 실제 요청 도착한 것을 감지**(읽진 않음) |
| Worker     | 골라진 요청 실제 처리                         |

## 1. Problem First

만약 Acceptor와 요청 처리를 같은 스레드가 한다면 어떤 일이 벌어지는가.

```java
// 가상의 단일 스레드 서버
ServerSocket server = new ServerSocket(8080);
while (true) {
    Socket client = server.accept();  // ① 연결 수락
    handleRequest(client);            // ② 요청 처리 (DB 조회 200ms)
    // ③ 처리 끝날 때까지 다음 accept() 호출 불가
}
```

문제는 명확하다.

- ②가 200ms 걸리는 동안 OS의 **accept queue(SYN 처리 끝난 ESTABLISHED 소켓 큐)** 에 연결이 쌓인다
- 큐가 가득 차면(`net.core.somaxconn` 초과) 신규 연결은 거부되거나 RST를 받는다
- TPS가 `1 / 평균_처리_시간` 으로 제한된다. DB가 느려지면 connection accept 자체가 막힌다

즉, **"연결을 받아들이는 일"과 "받은 연결을 처리하는 일"의 시간 척도가 다르기 때문에 분리가 필요하다.** Accept는 마이크로초, 처리는 밀리초~초 단위다.

---

## 2. Mechanics

### Tomcat의 스레드 분리 구조 (NIO 기준, Spring Boot 기본값)

```
┌─────────────────────────────────────────────────────────┐
│ Connector (Coyote)                                      │
│                                                         │
│  ┌──────────────┐   ┌──────────────┐   ┌─────────────┐  │
│  │ Acceptor     │──▶│ Poller       │──▶│ Worker      │  │
│  │ (1 thread)   │   │ (1 thread)   │   │ (N threads) │  │
│  │ accept()     │   │ Selector     │   │ Servlet 실행 │  │
│  └──────────────┘   └──────────────┘   └─────────────┘  │
└─────────────────────────────────────────────────────────┘
```

**Acceptor Thread** (`org.apache.tomcat.util.net.Acceptor`)

- 하는 일: `serverSocket.accept()` 호출만 한다. blocking I/O다
- 갯수: 기본 1개 (`acceptorThreadCount`, Tomcat 10부터는 deprecated되고 항상 1개)
- accept된 소켓을 Poller에 등록만 하고 즉시 다음 accept로 돌아간다

**Poller Thread** (`org.apache.tomcat.util.net.NioEndpoint$Poller`)

- 하는 일: Java NIO `Selector`로 등록된 소켓들의 `OP_READ` 이벤트를 감시
- 데이터가 도착한 소켓만 골라서 Worker 스레드 풀에 작업(`SocketProcessor`)을 submit

**Worker Thread** (Executor의 스레드 풀)

- 하는 일: 실제 HTTP 파싱 → Servlet 호출 → `DispatcherServlet` → `@Controller` 메서드 실행
- 이게 **네가 "애플리케이션 단에서 말하는 스레드"** 다
- 기본 풀 크기: `server.tomcat.threads.max=200` (Spring Boot 기본값)

### 그래서 같은 스레드인가?

JVM 관점에서는 **셋 다 똑같은 `java.lang.Thread` 인스턴스**다. OS 관점에서도 셋 다 동일한 native thread(리눅스에서는 LWP)다. 차이는 **어떤 풀에 속하고, 어떤 일을 처리하도록 분리되어 있느냐**일 뿐이다.

`Thread.currentThread().getName()`을 찍어보면 구분된다.

```java
@GetMapping("/hello")
public String hello() {
    System.out.println(Thread.currentThread().getName());
    // 출력: http-nio-8080-exec-1
    // exec-N이 Worker. Acceptor는 "http-nio-8080-Acceptor", Poller는 "http-nio-8080-Poller"
    return "ok";
}
```

`@Controller` 메서드는 항상 **Worker(exec-N)** 위에서 실행된다. Acceptor나 Poller 위에서는 절대 실행되지 않는다.

---

## 3. 공식 근거

**Tomcat 공식 문서 — HTTP Connector 설정**

> "The maximum number of request processing threads to be created by this Connector, which therefore determines the maximum number of simultaneous requests that can be handled. If not specified, this attribute is set to 200."
> 
> (이 Connector가 생성하는 요청 처리 스레드의 최대 개수이며, 동시에 처리 가능한 요청 수의 상한이 된다. 지정하지 않으면 200으로 설정된다.)

→ 출처: [Apache Tomcat 10 Configuration Reference — HTTP Connector, `maxThreads` 속성](https://tomcat.apache.org/tomcat-10.1-doc/config/http.html)

**Tomcat NIO Acceptor 소스 (Tomcat 10.1)**

```java
// org.apache.tomcat.util.net.Acceptor#run (요약)
while (endpoint.isRunning()) {
    // ...
    socket = endpoint.serverSocketAccept();   // blocking accept
    // ...
    if (!endpoint.setSocketOptions(socket)) { // Poller에 등록
        endpoint.closeSocket(socket);
    }
}
```

→ 출처: [Apache Tomcat GitHub — Acceptor.java](https://github.com/apache/tomcat/blob/main/java/org/apache/tomcat/util/net/Acceptor.java)

**Spring Boot 공식 문서 — Server Properties**

> `server.tomcat.threads.max` — Maximum amount of worker threads. Doesn't have an effect if virtual threads are enabled. Default: 200
> 
> (Worker 스레드의 최대 수. 가상 스레드가 활성화되어 있으면 효과가 없다. 기본값 200.)

→ 출처: [Spring Boot Application Properties — Server Properties](https://docs.spring.io/spring-boot/appendix/application-properties/index.html)

**리눅스 커널 — accept queue 관련**

`accept()` 시스템 콜이 가져오는 것은 이미 3-way handshake가 완료된 ESTABLISHED 소켓이다. 이 큐는 `net.core.somaxconn`과 `listen(backlog)`의 작은 값으로 제한된다.

→ 출처: [Linux man-pages — listen(2)](https://man7.org/linux/man-pages/man2/listen.2.html)

---

## 4. 이 설계를 이렇게 한 이유

**왜 Acceptor를 따로 두는가**

- accept()는 blocking 호출이고 OS 큐를 빠르게 비워야 한다. 처리 로직과 섞이면 큐가 막힌다
- 분리하면 Acceptor는 "받아두기"만 하고 즉시 다음 연결을 받을 수 있다

**왜 Poller가 또 있는가 (NIO에서만)**

- Worker 스레드를 연결 1개당 1개씩 묶어두면(BIO 방식) idle 커넥션이 많을 때 스레드가 낭비된다
- Poller가 Selector로 "실제로 읽을 데이터가 있는 소켓"만 골라 Worker에 전달하면, **C10K 문제**(스레드 수 ≠ 동시 연결 수) 해결이 가능하다
- Tomcat 9부터 기본 Connector가 BIO에서 NIO로 바뀐 이유다

**잃는 것**

- 컨텍스트 스위칭 비용: 하나의 요청이 Acceptor → Poller → Worker로 넘어가며 최소 2번의 스레드 간 전환이 일어난다. 짧은 요청에서는 BIO보다 오히려 느릴 수 있다
- 디버깅 복잡도: 스택 트레이스를 따라가도 어느 스레드에서 왜 멈췄는지 추적이 어렵다. 스레드 덤프 분석 시 이름 prefix(`Acceptor`/`Poller`/`exec`)로 구분해야 한다
- Worker 풀 고갈 문제는 여전히 존재한다. DB가 느려서 200개 Worker가 전부 점유되면, Poller가 아무리 빠르게 이벤트를 감지해도 처리할 스레드가 없다. Acceptor가 받아둔 연결은 큐(`acceptCount=100` 기본값)에 쌓이다 넘치면 거부된다

---

## 5. 이어지는 개념

질문이 "Worker 스레드 = 애플리케이션 단 스레드"의 정체를 묻는 거라 그 주변부터 짚자.

1. **Servlet 스레드 모델과 ThreadLocal 안전성** — Worker 스레드는 풀에서 재사용된다. `ThreadLocal`에 값을 넣고 안 비우면 다음 요청에서 누출된다. Spring Security의 `SecurityContextHolder`, Spring MVC의 `RequestContextHolder`가 정확히 이 메커니즘이다. 너의 인증 토큰 학습과 직접 이어진다
    
2. **`server.tomcat.threads.max` / `acceptCount` / `maxConnections` 튜닝** — Worker 풀 크기를 정하는 기준. "DB 커넥션 풀 크기와 어떻게 맞춰야 하는가"가 실무 핵심 질문이다. 1번 다음에 와야 풀 동작 원리가 이해된다
    
3. **Virtual Threads (Project Loom, JEP 444)** — Spring Boot 3.2+에서 `spring.threads.virtual.enabled=true`로 켜면 Worker가 가상 스레드로 바뀐다. 위 200개 제한이 사실상 무의미해진다. 2번을 이해해야 "왜 이게 게임 체인저인가"가 보인다
    
4. **`DispatcherServlet`과 요청 처리 파이프라인** — Worker 스레드가 실제로 무엇을 하는지. Filter → DispatcherServlet → HandlerMapping → HandlerAdapter → Controller. Spring MVC 내부 구조 학습의 출발점이고, 이전에 미뤄둔 "resolver" 용어가 여기서 `HandlerMethodArgumentResolver`, `ViewResolver` 등으로 등장한다
    

순서 이유: 1번은 지금 질문(Worker 스레드의 정체)과 가장 직결되고, 2번은 1번 위에서만 의미가 있으며, 3번은 2번의 한계 인식이 선행되어야 한다. 4번은 별개 트랙(Spring MVC 내부)으로 가지치는 지점이라 마지막에 뒀다.

# Poller가 하는 일

```java
// org.apache.tomcat.util.net.NioEndpoint$Poller#run (요약)
while (true) {
    int keyCount = selector.select(1000);  // OS에 "읽을 거 있는 소켓 알려줘"
    Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
    while (iterator.hasNext()) {
        SelectionKey sk = iterator.next();
        // 이 소켓은 실제로 데이터가 도착했음
        processKey(sk, ...);  // → Worker 풀에 SocketProcessor submit
    }
}
```

핵심은 `Selector.select()`다. 이건 리눅스에서 **`epoll_wait` 시스템 콜**로 내려간다. 커널이 "등록된 수만 개 소켓 중 지금 데이터가 도착한 것들"만 알려준다.

```
10,000개 연결 등록 → epoll_wait → "지금 읽을 게 있는 건 이 50개"
Poller가 이 50개만 Worker에 넘김 → Worker 50개만 일함 (200개 풀에서)
나머지 9,950개 idle 연결은 어떤 스레드도 점유하지 않음
```

즉 Poller의 역할은:

1. **다중 연결 감시** — 수만 개 소켓을 1개 스레드로 감시 (`Selector` 1개에 등록)
2. **이벤트 디스패치** — 실제 I/O 이벤트(`OP_READ`) 발생한 것만 골라냄
3. **Worker 풀에 작업 제출** — 골라낸 소켓 처리를 `SocketProcessor`로 감싸 Executor에 submit