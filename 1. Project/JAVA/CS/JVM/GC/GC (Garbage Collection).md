#CS/JVM/GC #Status/완료  
## 1. Problem First

GC가 없던 C/C++ 환경이다.

```c
// C에서 메모리 관리
char* buffer = malloc(1024);  // 직접 할당
processData(buffer);
free(buffer);                 // 직접 해제 — 개발자 책임

// 해제를 깜빡하면?
char* buffer = malloc(1024);
processData(buffer);
// free 없음 → 메모리 누수
// 프로그램이 오래 실행될수록 메모리를 계속 잡아먹음
// 서버는 결국 OOM으로 죽음

// 해제한 메모리를 또 쓰면?
free(buffer);
buffer[0] = 'A';  // Use-After-Free → 보안 취약점, 크래시
```

C 프로그램의 버그 중 상당수가 메모리 관리 실수에서 나온다. 
GC는 이 책임을 개발자에서 런타임으로 옮긴 것이다.

**그런데 GC도 공짜가 아니다.**

```
GC가 동작하는 동안
→ 애플리케이션 스레드가 멈춘다 (Stop-The-World)
→ 이 시간이 길면 서비스 응답 지연으로 직결
```

GC를 이해해야 하는 이유가 여기 있다.
"알아서 해준다"가 아니라 **"어떻게 해주는지 알아야 튜닝할 수 있다."**

---

## 2. Mechanics — 기본

### GC의 전제: 무엇을 수거할 것인가

**도달 가능성(Reachability)** 으로 판단한다.

```
GC Root에서 시작해서 참조를 따라갔을 때
닿을 수 있는 객체 → 살아있음 (Live)
닿을 수 없는 객체 → 죽음 (Garbage)
```

GC Root가 되는 것들:

```
- 각 스레드의 Stack에 있는 지역변수
- static 변수
- JNI 참조
```

```java
void method() {
    Object a = new Object();  // a: GC Root(Stack)에서 참조 → Live
    Object b = new Object();  // b: GC Root에서 참조 → Live
    a = null;                 // a가 참조 놓음 → 해당 객체 도달 불가 → Garbage
}
// method 종료 → b도 Stack에서 사라짐 → Garbage
```

참조 카운팅 방식(Python 등)과 다르다.

```
참조 카운팅 방식의 문제:
Object a = new Object();  // count = 1
Object b = new Object();  // count = 1
a.ref = b;  // b.count = 2
b.ref = a;  // a.count = 2
a = null;   // a.count = 1 (아직 b가 참조)
b = null;   // b.count = 1 (아직 a가 참조)
// 둘 다 도달 불가능한데 count가 0이 안 됨 → 누수
```

Java GC는 도달 가능성 기반이라 순환 참조를 자동으로 처리한다.

---

### GC 기본 알고리즘 세 가지

모든 GC 알고리즘은 이 세 가지의 조합이다.

**Mark-Sweep**

```
Mark  : GC Root에서 도달 가능한 객체에 표시
Sweep : 표시 안 된 객체 제거

Before: [A(live)][B(dead)][C(live)][D(dead)][E(live)]
After:  [A      ][      ][C      ][      ][E      ]
                   ↑ 빈 공간이 흩어짐 → 단편화 발생
```

단편화가 생기면 큰 객체를 넣을 연속 공간을 찾기 어렵다.

**Mark-Sweep-Compact**

```
Mark    : 도달 가능한 객체 표시
Sweep   : 죽은 객체 제거
Compact : 살아있는 객체를 앞쪽으로 당김

Before: [A(live)][B(dead)][C(live)][D(dead)][E(live)]
After:  [A][C][E][          빈 공간          ]
              ↑ 연속된 빈 공간 확보 → 단편화 없음
```

Compact 단계에서 객체를 이동시키면 포인터를 전부 재조정해야 한다. 비싸다. Old Gen처럼 드물게 실행될 때 쓴다.

**Copying**

```
힙을 두 영역으로 나눔 (From, To)
From에서 살아있는 객체만 To로 복사
From 전체를 한 번에 버림

Before From: [A(live)][B(dead)][C(live)][D(dead)]
Before To:   [                           ]

After From:  [                           ] ← 통째로 초기화
After To:    [A][C][        빈 공간       ]
```

복사 후 From을 통째로 버리니 단편화가 없다. 살아있는 객체가 적을수록 효율적이다. Young Gen에 쓰는 이유가 여기 있다. (Weak Generational Hypothesis)

---

### Stop-The-World (STW)

GC가 동작하는 동안 애플리케이션 스레드가 전부 멈추는 현상이다.

```
왜 멈춰야 하는가?

GC가 객체 그래프를 탐색하는 동안
애플리케이션이 객체를 계속 만들고 참조를 바꾸면
→ 탐색 결과가 일관성 없음
→ 살아있는 객체를 죽은 것으로 잘못 판단할 수 있음
→ 데이터 손상
```

```
타임라인:
앱 실행 ──────────┐ STW ┌──────────────── 앱 실행
                  └─────┘
                   GC 동작
                   이 구간 동안 모든 요청 처리 중단
```

STW 시간이 길면 사용자가 느끼는 응답 지연으로 직결된다. GC 튜닝의 핵심 목표는 **STW 시간을 줄이는 것**이다.

---

## 3. Mechanics — 심화

### Serial / Parallel GC

**Serial GC**

```
단일 스레드로 GC 수행
Young: Copying
Old:   Mark-Sweep-Compact
STW 동안 GC 스레드 1개만 동작
→ 단순하지만 멀티코어 활용 못함
→ 클라이언트 앱, 소규모 환경용
```

**Parallel GC (Java 8 기본값)**

```
멀티 스레드로 GC 수행 (STW는 동일하게 발생)
Young: Parallel Copying
Old:   Parallel Mark-Sweep-Compact
→ STW 시간은 동일하지만 GC 처리량(Throughput) 향상
→ 배치 처리처럼 응답 시간보다 처리량이 중요한 환경
```

```
Serial:   STW [──GC──────────────────] 재개
Parallel: STW [──GC①GC②GC③GC④──────] 재개
              멀티스레드로 더 빨리 끝냄
```

---

### G1GC (Java 9+ 기본값)

기존 GC의 근본적인 문제를 다르게 접근한다.

**기존 방식의 문제**

```
힙이 클수록 Old Gen도 크다
Old Gen Full GC → 전체를 스캔
→ 힙이 클수록 STW 시간이 선형으로 증가
→ 수 GB 힙에서 수 초 STW 발생 가능
```

**G1GC의 접근**

힙을 고정된 Young/Old 영역으로 나누지 않는다. 전체 힙을 **동일한 크기의 Region으로 분할**한다.

```
힙 전체
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ E │ E │ S │ O │ O │ H │ E │ O │
└───┴───┴───┴───┴───┴───┴───┴───┘
  E: Eden  S: Survivor  O: Old  H: Humongous(대형 객체)

각 Region은 동적으로 역할이 바뀜
→ 지금 Eden인 Region이 다음엔 Old가 될 수 있음
```

**Humongous Region**

Region 크기의 50% 이상인 객체는 Humongous로 직접 할당된다.

```java
// 이런 객체들
byte[] largeArray = new byte[1024 * 1024 * 10]; // 10MB
```

Humongous 객체가 많으면 G1GC 성능이 떨어진다. 연속된 Region을 차지하고 이동이 어렵기 때문이다.

**G1GC가 STW를 줄이는 방법**

```
Pause Target 설정:
-XX:MaxGCPauseMillis=200  (기본값 200ms)

G1GC가 이 목표 안에서 수거할 Region을 선택
→ 가장 쓰레기가 많은 Region부터 수거 (Garbage First의 어원)
→ 목표 시간 내에 끝날 만큼만 수거
→ 전체를 한 번에 수거하지 않음
```

**G1GC 동작 사이클**

```
Young GC (Minor)
→ Eden Region들을 수거
→ STW, 짧음

Concurrent Marking
→ 앱 실행과 동시에 Old Region의 살아있는 객체 마킹
→ STW 없음 (대부분)

Mixed GC
→ Young Region + 쓰레기 많은 Old Region 함께 수거
→ STW, Young GC보다 조금 김

Full GC (최후 수단)
→ 전체 힙 수거
→ STW, 오래 걸림
→ 이게 발생하면 튜닝이 필요하다는 신호
```

---

### ZGC (Java 15+ 정식, Java 21 Generational ZGC)

G1GC도 Mixed GC에서 STW가 발생한다. ZGC의 목표는 다르다.

```
목표: 힙 크기와 무관하게 STW를 수 ms 이하로 유지
     (수십 GB ~ 수 TB 힙에서도)
```

**어떻게 STW 없이 수거하는가 — Colored Pointers**

ZGC는 객체 포인터 자체에 메타데이터를 심는다.

```
일반 포인터 (64bit):
[000000000000000000000000000000000000000000000000] 주소

ZGC 포인터 (64bit):
[메타비트 4bit][주소 44bit][여유]
  │
  └── Marked0, Marked1, Remapped, Finalizable 상태 표시
```

이 비트를 보고 객체가 이동됐는지, 마킹됐는지 즉시 판단한다.

**Load Barrier**

```java
Object obj = someArray[i]; // 객체 참조를 읽는 순간
```

ZGC는 이 읽기 연산에 **Load Barrier**를 삽입한다.

```
객체 참조 읽기
    │
    ▼
포인터의 메타비트 확인
    │
    ├── 최신 상태 → 그대로 반환
    │
    └── 이동됐거나 마킹 필요 → 처리 후 반환
```

GC와 앱이 동시에 실행되면서도 항상 올바른 참조를 보장한다. STW 없이 Concurrent하게 수거할 수 있는 근거다.

**Generational ZGC (Java 21)**

초기 ZGC는 세대 구분이 없었다. 모든 객체를 동일하게 취급 → Weak Generational Hypothesis 활용 못함 → 처리량 낮음

Java 21에서 세대형 ZGC 도입:

```
Young Generation + Old Generation 구분 복원
+ ZGC의 Concurrent 수거 유지
→ 처리량과 저지연 동시 달성
```

---

### GC 선택 기준

```
Serial GC
→ 단일 코어, 소규모 힙 (< 100MB)
→ 현업에서 거의 안 씀

Parallel GC
→ 처리량 최우선, 응답 시간 덜 중요
→ 배치 처리, 데이터 파이프라인

G1GC (현업 기본값)
→ 범용, 응답 시간과 처리량 균형
→ 힙 4GB ~ 수십 GB
→ 웹 서비스 일반적인 선택

ZGC
→ 저지연 최우선 (p99 응답이 중요)
→ 힙이 매우 크거나 (수십 GB 이상)
→ 금융, 실시간 시스템
→ Java 21 Generational ZGC부터 처리량 문제 개선
```

---

## 4. 공식 근거

**GC Roots와 도달 가능성**

> _"An object is considered garbage when no live thread can access it."_ — [Oracle, HotSpot GC Tuning Guide](https://docs.oracle.com/en/java/se/21/gctuning/)
> 
> _어떤 살아있는 스레드도 접근할 수 없는 객체는 가비지로 간주된다._

**G1GC Pause Target**

> _"G1 GC is a generational, incremental, parallel, mostly concurrent, stop-the-world, and evacuating garbage collector which monitors pause-time goals in each of the stop-the-world pauses."_ — [Oracle, G1GC Tuning Guide](https://docs.oracle.com/en/java/se/21/gctuning/garbage-first-garbage-collector.html)
> 
> _G1GC는 각 STW 중단에서 일시 중지 시간 목표를 모니터링하는 세대형, 증분형, 병렬, 대부분 동시, STW, 이동형 가비지 컬렉터다._

**ZGC 목표**

> _"ZGC is a scalable low-latency garbage collector. ZGC performs all expensive work concurrently, without stopping the execution of application threads for more than a few milliseconds."_ — [OpenJDK, ZGC](https://wiki.openjdk.org/display/zgc)
> 
> _ZGC는 확장 가능한 저지연 가비지 컬렉터다. 애플리케이션 스레드 실행을 수 밀리초 이상 중단하지 않고 모든 비용이 큰 작업을 동시에 수행한다._

**Generational ZGC — JEP 439**

> _"Improve application performance by extending the Z Garbage Collector to maintain separate generations for young and old objects."_ — [JEP 439](https://openjdk.org/jeps/439)
> 
> _젊은 객체와 오래된 객체에 대해 별도의 세대를 유지하도록 Z 가비지 컬렉터를 확장하여 애플리케이션 성능을 개선한다._

---

## 5. 트레이드오프

GC 튜닝의 세 축은 동시에 만족시킬 수 없다.

```
      Throughput (처리량)
           △
           │
           │
           │
Footprint ◄─────────► Latency (응답 시간)
(메모리 사용량)
```

**Throughput을 높이면** → GC를 덜 자주, 한 번에 많이 수거 → STW 시간이 길어짐 → Latency 악화

**Latency를 낮추면** → GC를 자주, 조금씩 수거 → GC 오버헤드 증가 → Throughput 감소

**Footprint를 줄이면** → 힙이 작아서 GC가 자주 발생 → Throughput, Latency 둘 다 악화

GC 선택과 튜닝은 이 세 축에서 어디에 우선순위를 두느냐의 결정이다.

---

## 6. 실무 진단 포인트

**GC 로그 활성화**

```bash
# Java 11+
-Xlog:gc*:file=gc.log:time,uptime:filecount=5,filesize=20m
```

**로그에서 봐야 할 것**

```
[2.345s] GC(5) Pause Young (Normal) 512M→128M(1024M) 12.345ms
          │              │           │                  │
          시간           GC 종류      힙 변화            STW 시간

STW 시간이 목표값 초과 → 튜닝 필요
Full GC 발생 → 즉시 원인 파악 필요
Old Gen이 계속 증가 → 메모리 누수 의심
```

**OOM 종류별 원인**

```
java.lang.OutOfMemoryError: Java heap space
→ 힙에 객체가 너무 많음
→ 메모리 누수 또는 힙 크기 부족

java.lang.OutOfMemoryError: Metaspace
→ 클래스가 너무 많이 로딩됨
→ 동적 프록시/리플렉션 남용, ClassLoader 누수

java.lang.OutOfMemoryError: GC overhead limit exceeded
→ GC가 전체 시간의 98%를 쓰는데 2% 미만만 수거
→ GC thrashing 상태, 사실상 서비스 불능
```

---

## 한 줄 정리

> GC는 도달 가능성 기반으로 죽은 객체를 수거한다. STW를 줄이는 방향으로 Serial → Parallel → G1GC → ZGC로 발전했다. 처리량, 지연시간, 메모리는 동시에 최적화할 수 없고, 워크로드에 맞는 GC를 선택하고 로그로 실측해야 한다.

---

## 이어지는 개념

```
GC 완료
    │
    ├── 1순위: GC 튜닝 실전
    │         GC 로그 분석, 힙 덤프 분석
    │         컨테이너 환경에서의 메모리 설정
    │         (-XX:+UseContainerSupport)
    │
    ├── 2순위: 메모리 누수 패턴
    │         static 컬렉션, 리스너 미해제
    │         WeakReference / SoftReference
    │         GC가 수거 못하는 상황을 코드로 만드는 패턴
    │
    └── 3순위: 스레드와 동시성
              Heap 공유에서 생기는 race condition
              synchronized, volatile, happens-before
```