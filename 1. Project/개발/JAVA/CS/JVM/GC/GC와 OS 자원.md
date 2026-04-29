Connection은 왜 직접 연결해제를해야하지? GC 수거대상이아닌가?
## "수거 대상이 아닌 게 아니라, 수거돼도 문제가 남는다"

GC는 **메모리(힙 객체)** 를 관리하는 거야. 
근데 HTTP Connection 같은 건 메모리 말고 **OS 레벨 자원** 도 같이 들고 있거든.

---

## Connection 객체가 들고 있는 것들

```
HttpConnection 객체 (힙 메모리)
    ├── 필드들... (Java 힙, GC 대상 ✅)
    └── File Descriptor (OS 커널 자원, GC 모름 ❌)
                └── TCP 소켓 버퍼
                └── 포트 점유
                └── 상대 서버와의 연결 상태
```

GC는 **Java 힙만** 관리해. File Descriptor(FD) 같은 OS 자원은 GC가 아예 모르는 영역이야.

---

## 그럼 왜 수거가 "늦어지면" 문제야?

두 가지 문제가 겹쳐 있어.

**① OS 자원 고갈 (더 심각)**

`close()` 를 안 하면 FD가 계속 열려 있어. OS마다 프로세스당 FD 한도가 있는데 (Linux 기본 1024개), 이걸 다 쓰면 `Too many open files` 에러가 터져. GC가 나중에 객체를 수거해도, **그 전까지 FD는 계속 점유 중.**

**② GC가 언제 수거할지 보장이 없음**

```
connection = null;  // 참조 끊음
// GC Root에서 unreachable → GC 대상이 됨
// 근데 GC가 "언제" 실행될지는 JVM 마음
// → 그 사이에 FD는 계속 열려 있음
```

`finalize()` / `Cleaner` 같은 걸로 GC 시점에 FD를 닫을 수도 있는데, JVM이 이걸 **즉시 실행한다는 보장이 없어.** 심지어 프로그램 종료 전까지 안 부를 수도 있음.

---

## 정리하면

|                     | GC 대상?      | close() 필요?          |
| ------------------- | ----------- | -------------------- |
| 일반 Java 객체          | ✅ 알아서 수거    | ❌ 불필요                |
| Connection / Stream | ✅ 힙 객체는 수거됨 | ✅ **필수** (OS 자원 때문에) |

메모리 낭비도 있긴 한데, 진짜 문제는 **GC가 수거하기 전까지 OS 자원(FD, 포트)이 묶여있다는 것**이야.

그래서 Java에서 `try-with-resources` 가 있는 거고, `Closeable` / `AutoCloseable` 인터페이스가 따로 있는 거야. GC에 맡기지 말고 **명시적으로 OS에 자원 반납**하라는 의미.

---
## OS 자원과 웹 개발자가 알아야 할 핵심

---

### FD(File Descriptor)가 뭔지 정확히

OS는 모든 I/O를 **"파일"이라는 추상화로 통일**해. 진짜 파일이든, 소켓이든, 파이프든 전부 FD 하나로 표현해. Unix 철학인 "Everything is a file"이 여기서 나온 거야.

프로세스가 시작되면 OS는 FD 테이블을 만들어줘. FD 0/1/2는 stdin, stdout, stderr로 예약되어 있고, 이후 자원 할당마다 3, 4, 5... 순서로 부여돼. 이 테이블은 **커널 메모리**에 있어. JVM 힙이랑 완전히 분리된 영역이야.

---

### TCP Connection의 실제 비용

소켓 하나를 열면 단순히 FD 하나 소비로 끝나지 않아.

**커널이 실제로 잡는 것들:**

- 송신 버퍼 / 수신 버퍼 (기본 각 4KB~87KB, 튜닝 가능)
- TCP 상태 머신 (ESTABLISHED, TIME_WAIT 등)
- 로컬 포트 점유

여기서 웹 개발자한테 중요한 게 **TIME_WAIT** 상태야.

`close()`를 호출해도 소켓이 즉시 사라지지 않아. TCP 4-way handshake 이후 OS가 해당 포트를 **기본 60초~4분 동안 TIME_WAIT 상태로 붙잡아**둬. 이유는 "혹시 늦게 도착하는 패킷이 새 연결에 섞이면 안 되니까" 라는 안전장치야.

트래픽이 많은 서버에서 Connection을 계속 맺고 끊으면, 
TIME_WAIT 소켓이 수천 개 쌓여서 **사용 가능한 포트가 고갈**될 수 있어. (포트 범위는 0~65535로 유한함)

---

### 그래서 Connection Pool이 존재하는 이유

매 요청마다 Connection을 새로 맺으면:

1. TCP 3-way handshake 비용
2. TLS라면 추가 핸드셰이크 비용
3. TIME_WAIT 누적

Connection Pool은 이걸 해결하기 위해 **Connection을 미리 열어두고 재사용**해. close() 대신 풀에 반납하는 거야. FD는 계속 열려있지만, 의도적으로 열어두는 거지.

여기서 웹 개발자가 놓치는 포인트가 있어.

**Pool에서 꺼낸 Connection을 close() 안 하면?**

Pool 입장에서는 "아직 사용 중"으로 판단해서 반납이 안 돼. Pool 사이즈가 꽉 차면 새 요청들이 대기 큐에서 timeout날 때까지 기다려. 이게 실제 장애로 이어지는 패턴이야. FD 고갈이 아니라 **Pool 고갈**로 먼저 터지는 경우가 많아.

---

### Blocking I/O vs Non-Blocking I/O와 FD의 관계

전통적인 Blocking I/O 모델에서는 **Connection 하나당 Thread 하나**를 점유해. Connection이 100개면 Thread도 100개 필요하고, Thread는 각자 스택 메모리를 들고 있어 (기본 512KB~1MB).

즉, FD 하나가 결국 Thread 하나를 묶어두는 구조야.

Non-Blocking I/O (Java NIO, Netty, WebFlux)는 이걸 뒤집어서, **적은 Thread로 수많은 FD를 이벤트 방식으로 처리**해. OS의 `epoll` (Linux) / `kqueue` (macOS) 를 활용해서 "이 FD들 중에 읽을 준비된 게 생기면 알려줘" 방식으로 동작해.

Spring WebFlux나 Netty가 적은 Thread로 높은 동시성을 처리할 수 있는 근본 이유가 여기 있어.

---

### 핵심 연결고리 요약

```
FD 고갈         → Too many open files, 새 연결 불가
TIME_WAIT 누적  → 포트 고갈, 새 연결 불가  
Pool 고갈       → 요청 대기 → timeout → 장애
Thread 고갈     → Blocking 모델의 한계 → NIO 등장 배경
```

GC가 왜 이걸 처리 못 하는지도 이제 명확해. GC는 힙만 봐. FD 테이블은 커널 영역이고, TIME_WAIT는 OS TCP 스택이 관리하고, Pool 상태는 라이브러리 내부 상태야. **전부 JVM 외부**의 이야기들이야.