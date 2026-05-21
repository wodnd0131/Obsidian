좋다. **각 풀별 튜닝 기준**을 실무 관점으로 정리하자. 이전 토픽들에서 yml 설정 키는 봤으니, 여기서는 **"숫자를 어떻게 결정하는가"**에 집중한다.

## 1. 튜닝의 본질 — 풀 사이즈는 답이 하나가 아니다

먼저 핵심 원칙. **튜닝의 목표는 "최대 사이즈"가 아니라 "병목 지점을 의도적으로 선택하는 것"이다.**

```
모든 풀은 어딘가에서 막혀야 한다.
어디서 막을지 네가 정해라. 안 정하면 가장 약한 곳이 먼저 무너진다.
```

예시:

- DB Connection을 작게 잡으면 → DB에서 막힘 (DB 보호)
- Worker를 작게 잡으면 → Tomcat에서 막힘 (앱 보호)
- 둘 다 크게 잡으면 → 메모리 부족이나 OS에서 막힘 (시스템 붕괴)

**의도된 병목**을 만드는 게 튜닝이다.

## 2. Worker 스레드 풀 튜닝

### 2.1 결정 요소

Worker 풀 사이즈를 결정하는 변수 4개:

```
1. 평균 응답 시간 (T_avg)        : 한 요청 처리에 걸리는 시간
2. 목표 TPS                      : 초당 처리하고 싶은 요청 수  
3. CPU 코어 수                   : 물리적 동시 실행 한계
4. I/O 대기 비율                 : 요청 중 I/O로 인한 대기 시간 비율
```

### 2.2 Little's Law 기반 산정

가장 기본적인 공식:

```
필요 Worker 수 = 목표 TPS × 평균 응답 시간(초)
```

예시:

- 목표: 1,000 TPS
- 평균 응답 시간: 200ms
- 필요 Worker = 1,000 × 0.2 = **200개**

이게 Spring Boot 기본값이 200인 합리적 이유 중 하나.

### 2.3 Brian Goetz 공식 (Java Concurrency in Practice)

I/O 비율을 반영한 정교한 공식:

```
N_threads = N_cpu × U_cpu × (1 + W/C)

N_cpu : CPU 코어 수
U_cpu : 목표 CPU 사용률 (0~1)
W/C   : 대기 시간 / 컴퓨팅 시간 비율
```

웹 애플리케이션 시나리오:

- CPU 8코어
- 목표 CPU 사용률 80% (0.8)
- 요청 시간 중 90%가 DB/외부 API 대기, 10%만 CPU 계산
- W/C = 90/10 = 9

```
N = 8 × 0.8 × (1 + 9) = 8 × 0.8 × 10 = 64개
```

**I/O 대기가 많을수록 Worker를 많이 둬도 된다.** CPU가 놀고 있으니까.

### 2.4 실무 결정 절차

```
1. 부하 테스트 (JMeter, k6, Gatling)
   ├─ 점진적으로 부하 증가
   └─ 응답 시간, TPS, CPU/메모리 측정
   
2. 병목 지점 식별
   ├─ Worker 풀 가득 차는지
   ├─ DB Connection 가득 차는지
   ├─ CPU 100% 치는지
   └─ 메모리 부족한지
   
3. 의도된 병목으로 조정
   └─ 보통: DB가 먼저 막히도록 (DB 보호)
   
4. 운영 모니터링으로 검증
   └─ Actuator /metrics에서 tomcat.threads.busy 추적
```

### 2.5 일반적 시작값

```yaml
server:
  tomcat:
    threads:
      max: 200          # 시작은 기본값
      min-spare: 20     # 트래픽 급증 대비
    max-connections: 8192
    accept-count: 100
    connection-timeout: 5s   # 기본 20s는 너무 김. Slow attack 방어
```

`max: 200`을 무작정 늘리지 않는다. 이유:

- 스레드 1개당 스택 메모리 ~1MB → 1,000개면 1GB
- 컨텍스트 스위칭 비용 증가
- DB Connection 풀이 못 따라가면 의미 없음

## 3. Spring `@Async` 풀 튜닝

### 3.1 작업 성격에 따라 다르다

`@Async`에서 실행하는 작업의 종류를 분류:

|작업 종류|예시|CPU/I/O|
|---|---|---|
|I/O bound|메일 전송, 외부 API 호출, 파일 업로드|I/O 위주|
|CPU bound|이미지 처리, 암호화, 계산|CPU 위주|
|Mixed|DB 작업 후 외부 API 호출|혼합|

### 3.2 결정 공식

**I/O bound 작업**:

```
core-size = CPU 코어 수 × 2 ~ 코어 수 × 10
max-size = core-size × 2 정도
```

**CPU bound 작업**:

```
core-size = CPU 코어 수
max-size = CPU 코어 수 + 1
```

이유: CPU bound 작업을 코어 수보다 많이 돌리면 컨텍스트 스위칭으로 오히려 느려진다.

### 3.3 실무 설정 예시

메일/알림 같은 I/O bound (8코어 기준):

```yaml
spring:
  task:
    execution:
      pool:
        core-size: 16
        max-size: 32
        queue-capacity: 100      # 큐 무한대는 절대 금지
        keep-alive: 60s
      thread-name-prefix: async-
```

### 3.4 ❗ 절대 빠뜨리지 말 것: queue-capacity

기본값이 `Integer.MAX_VALUE`(21억)인 점이 함정이다.

```
queue-capacity 무한대 시나리오:
  ↓
@Async 작업 폭주 → core-size 16개 가득 참
  ↓
max-size까지 늘리려면 큐가 먼저 가득 차야 함
  ↓
큐 용량이 21억이라 절대 가득 안 참
  ↓
max-size로 늘어나지 않음. 큐만 무한정 적재
  ↓
힙 메모리 부족 → OOM → JVM 죽음
```

`ThreadPoolExecutor`의 동작:

```
새 작업 도착
  ↓
core-size 미달? → 새 스레드 만들어 처리
  ↓
core-size 도달? → 큐에 적재
  ↓
큐도 가득? → max-size까지 새 스레드 생성
  ↓
max-size도 도달? → RejectedExecutionException
```

**큐 용량이 작아야 max-size로 확장된다.** 무한대 큐는 max-size를 무의미하게 만든다.

권장: `queue-capacity`를 명시적으로 작게 (보통 100~500).

### 3.5 RejectedExecutionHandler

큐도 가득 차고 max-size도 도달했을 때의 정책:

```java
@Bean
public TaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(16);
    executor.setMaxPoolSize(32);
    executor.setQueueCapacity(100);
    
    // 거부 정책 (기본은 AbortPolicy = 예외 throw)
    executor.setRejectedExecutionHandler(
        new ThreadPoolExecutor.CallerRunsPolicy()
    );
    executor.initialize();
    return executor;
}
```

정책 종류:

|정책|동작|언제 쓰나|
|---|---|---|
|`AbortPolicy` (기본)|RejectedExecutionException throw|작업 손실 감지 필요|
|`CallerRunsPolicy`|호출한 스레드(Worker)가 직접 실행|백프레셔 효과, 가장 안전|
|`DiscardPolicy`|조용히 버림|로그성 작업|
|`DiscardOldestPolicy`|큐의 가장 오래된 작업 버리고 새로 추가|최신성 중요할 때|

`CallerRunsPolicy`가 실무에서 가장 자주 쓰인다. 이유는:

- 작업 손실 없음
- Worker 스레드가 비동기 작업을 직접 실행하므로 그동안 HTTP 처리 못 함 → 자연스러운 부하 조절(백프레셔)

## 4. `@Scheduled` 풀 튜닝

### 4.1 결정 기준

```
필요 스레드 = 동시에 실행될 수 있는 스케줄 작업 수
```

스케줄 작업이 5개 있어도 시간이 겹치지 않으면 1개로 충분. 겹치면 그만큼 필요.

### 4.2 일반적 설정

```yaml
spring:
  task:
    scheduling:
      pool:
        size: 4              # 기본 1개는 너무 적음. 직렬화 위험
      thread-name-prefix: scheduling-
```

### 4.3 ❗ 기본값 1개의 함정

```java
@Scheduled(fixedRate = 5000)
public void taskA() { 
    Thread.sleep(10_000);  // 10초 걸림
}

@Scheduled(fixedRate = 5000)  
public void taskB() { 
    // 빨리 끝나야 함
}
```

`pool.size: 1`이면:

- taskA가 10초 동안 점유
- taskB는 5초 간격으로 실행되어야 하는데 못 함
- 큐에 쌓임 → taskA 끝난 후 한꺼번에 실행

`size`를 충분히 잡거나, 작업을 `@Async`로 위임해야 한다.

## 5. HikariCP 풀 튜닝

이전 토픽에서 다뤘지만, **Worker 풀과의 관계**가 핵심이라 한 번 더.

### 5.1 HikariCP 공식 공식

```
connections = (core_count × 2) + effective_spindle_count
```

8코어 SSD → 약 17개. **무조건 크게 잡지 않는다.**

### 5.2 Worker 풀과의 균형

```
Worker 200개 vs HikariCP 20개

상황: HTTP 요청 200개 동시 처리
  ↓
모든 Worker가 @Transactional 진입 시도
  ↓
20개만 Connection 획득. 180개는 connection-timeout 대기
  ↓
대부분 timeout → SQLException → 500
```

해결:

- **Worker를 줄여서 균형 맞추기** (보수적, DB 보호)
- **HikariCP를 늘리기** (DB 부하 증가, DB가 견디는 한도 내에서)
- **`@Transactional` 범위 줄이기** (Connection 점유 시간 단축)

### 5.3 실무 설정

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 20             # max와 같게 (HikariCP 권장)
      connection-timeout: 3000     # 3초. 기본 30초는 너무 김
      max-lifetime: 1800000        # 30분
      leak-detection-threshold: 60000  # 60초. 누수 감지 활성화
```

`leak-detection-threshold`는 운영 환경 필수. Connection을 빌렸는데 60초 안에 안 돌려주면 경고 로그.

## 6. 풀 간 균형 — 한 번에 보기

전체 풀이 어떻게 조화를 이뤄야 하는지 그림:

```
┌─ HTTP 트래픽 ────────────────────────────────────┐
│  목표 TPS: 1,000                                 │
└──────────────────────────────────────────────────┘
              ↓
┌─ Tomcat Worker 풀 ───────────────────────────────┐
│  threads.max: 200                                │
│  평균 응답 시간 0.2초 가정 → Little's Law로 충분  │
│  ❗ 이걸 통과한 요청만 다음 단계로                 │
└──────────────────────────────────────────────────┘
              ↓
┌─ Spring @Async 풀 (해당 시) ─────────────────────┐
│  core: 16, max: 32, queue: 100                  │
│  HTTP와 격리되어 폭주해도 영향 없음               │
└──────────────────────────────────────────────────┘
              ↓
┌─ HikariCP ───────────────────────────────────────┐
│  maximum-pool-size: 20                          │
│  ❗ 의도된 병목. DB 부하 차단                     │
└──────────────────────────────────────────────────┘
              ↓
┌─ DB (PostgreSQL/MySQL) ──────────────────────────┐
│  max_connections: 100                            │
│  앱 인스턴스 N대 × 20 ≤ 80 이어야 안전           │
└──────────────────────────────────────────────────┘
```

**위에서 아래로 갈수록 작아지는 게 정상.** 거꾸로면 DB가 못 받아준다.

## 7. 측정 — 튜닝의 전제

값을 정하는 것보다 측정이 먼저다. Spring Boot Actuator 활성화:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, metrics, prometheus
  metrics:
    enable:
      all: true
```

주요 메트릭:

```
Tomcat:
  tomcat.threads.busy          # 현재 사용 중 Worker 수
  tomcat.threads.config.max    # 설정값
  tomcat.connections.current   # 현재 연결 수
  
Executor (@Async):
  executor.active             # 활성 스레드
  executor.queued             # 큐 대기 작업
  executor.pool.size          # 현재 풀 크기
  
HikariCP:
  hikaricp.connections.active # 사용 중 Connection
  hikaricp.connections.idle   # idle Connection
  hikaricp.connections.pending # 대기 중인 요청
  hikaricp.connections.timeout # timeout 누적
```

**`hikaricp.connections.pending`이 0이 아니면 풀 부족.** 늘리거나 Worker 줄이는 신호.

**`tomcat.threads.busy`가 max에 자주 가까우면 Worker 부족.** 늘리거나 응답 시간 단축.

## 8. 공식 근거

**HikariCP 공식 — Pool Sizing**

> "A formula which has held up pretty well across a lot of benchmarks for years is that for maximizing throughput the number of active connections should be close to ((core_count * 2) + effective_spindle_count)."
> 
> (수년간 많은 벤치마크에서 검증된 공식: 처리량 최대화를 위한 활성 커넥션 수는 (코어 수 × 2 + 디스크 스핀들 수)에 가까워야 한다.)

→ 출처: [HikariCP GitHub Wiki — About Pool Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)

**Brian Goetz — Java Concurrency in Practice (8.2 Sizing Thread Pools)**

> "For compute-intensive tasks, an Ncpu-processor system usually achieves optimum utilization with a thread pool of Ncpu+1 threads. ... For tasks that also include I/O or other blocking operations, you want a larger pool."
> 
> (계산 집약적 작업은 N_cpu 프로세서 시스템에서 N_cpu+1 스레드의 풀이 최적. I/O나 블로킹 작업이 포함되면 더 큰 풀이 필요하다.)

→ 출처: _Java Concurrency in Practice_, Brian Goetz, Section 8.2

**Spring Boot 공식 — Task Execution**

> "If a ThreadPoolTaskExecutor is not configured by the user, Spring Boot auto-configures one with sensible defaults: 8 core threads that can grow and shrink according to the load."
> 
> (사용자가 ThreadPoolTaskExecutor를 구성하지 않으면 Spring Boot가 합리적 기본값(코어 8, 부하에 따라 가변)으로 자동 구성한다.)

→ 출처: [Spring Boot Reference — Task Execution](https://docs.spring.io/spring-boot/reference/features/task-execution-and-scheduling.html)

## 9. 잃는 것 — 풀을 크게 잡을 때의 비용

큰 풀의 비용을 늘 의식해야 한다.

|자원|비용|
|---|---|
|메모리|스레드당 스택 ~1MB (`-Xss` 기본). 1,000개 = 1GB|
|CPU|컨텍스트 스위칭 오버헤드. 코어 수 대비 너무 많으면 thrashing|
|DB|Connection 증가 시 DB 메모리/CPU 소비. PostgreSQL은 프로세스 기반이라 특히 비쌈|
|디버깅|스레드 덤프 분석 복잡도 증가|

**"풀이 크다고 빠른 게 아니다."** 시스템 전체에서 가장 약한 자원이 한계다.

## 10. 다음 단계

이 흐름의 자연스러운 다음 토픽들:

1. **부하 테스트 실전** — JMeter/k6로 위 값들을 어떻게 검증하는가
2. **Virtual Threads (JEP 444)** — 스레드 풀 사이즈 개념 자체를 바꾸는 변화. Spring Boot 3.2+의 `spring.threads.virtual.enabled=true`
3. **`CompletableFuture`와 `@Async`의 조합** — 비동기 작업 체이닝과 ThreadLocal 처리
4. **Actuator + Prometheus + Grafana** — 위 메트릭을 실시간 모니터링

지금 흐름에서는 **"풀을 어떻게 튜닝하는지의 원칙(의도된 병목 만들기, 측정 우선, 균형)"**까지 잡은 게 핵심이다. 구체 값은 부하 테스트로만 확정된다.