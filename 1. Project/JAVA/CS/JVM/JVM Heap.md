#CS/JVM/Memory 
## 1. Problem First
```
java.lang.OutOfMemoryError: Java heap space
```

또는 GC가 멈추지 않고 CPU를 100% 잡아먹는 **GC thrashing** 현상이 발생한다.

원인을 진단하려면 `jvisualvm`이나 `jmap`을 열어야 하는데, 
거기서 보이는 게 바로 Heap 영역 구조다. 
이걸 모르면 숫자가 보여도 해석이 안 된다.

```
Heap usage:
  Eden Space:    256 MB (98% used)  ← 계속 꽉 차있음
  Survivor Space: 32 MB
  Old Gen:       512 MB (99% used)  ← 여기가 차면 Full GC
  Metaspace:      64 MB
```

이 숫자가 뭘 의미하는지 알아야 OOM을 고칠 수 있다.
### **jvisualvm / jmap 관련**

둘 다 JVM 모니터링/진단 도구다.

`jvisualvm`은 GUI 기반으로 힙 사용량, GC 활동, 스레드 상태를 실시간으로 시각화해준다. JDK 8까지는 번들로 포함됐고, 이후엔 별도 설치다.

`jmap`은 CLI 도구로, 실행 중인 JVM 프로세스의 힙 스냅샷(heap dump)을 파일로 뽑아낸다.

bash

```bash
jmap -dump:format=b,file=heap.hprof <pid>
```

이렇게 뽑은 `.hprof` 파일을 Eclipse MAT이나 IntelliJ에서 열면 어떤 객체가 메모리를 얼마나 점유하는지 분석할 수 있다. OOM이 발생했을 때 원인을 찾는 표준 루틴이다.

---

## 2. Mechanics

### 전체 구조

![[../../../../../Repository/JVM Heap-1.png]]

---

### Young Generation

**"대부분의 객체는 일찍 죽는다"** 는 [[GC/Weak Generational Hypothesis||Weak Generational Hypothesis]] 가 설계 근거다.

```java
public LottoResult calculate(LottoTickets tickets, WinningLotto winning) {
    // 이 메서드 안에서 생성되는 모든 중간 객체는
    // 메서드 종료 후 참조가 끊긴다 → Young Gen에서 수거
    Map<Rank, Long> rankCounts = tickets.stream()
        .map(ticket -> winning.match(ticket))   // 임시 Rank 객체들
        .collect(groupingBy(identity(), counting()));
    return new LottoResult(rankCounts);
}
```

Young Gen은 세 구역으로 나뉜다.

```
Eden → Minor GC 발생 → 살아남은 객체 → S0 (age=1)
                                    ↓ 다음 Minor GC
                               S1 (age=2)
                                    ↓ age가 임계값 초과
                               Old Gen 승격 (Promotion)
```

**Eden Space**

- 새 객체가 최초 할당되는 곳
- `new` 키워드 실행 시 여기서 시작
- [[Thread/TLAB (Thread-Local Allocation Buffer)|Thread-Local Allocation Buffer (TLAB)]] 덕분에 스레드 간 동기화 없이 bump-pointer 방식으로 할당됨 → 매우 빠름

**Survivor Space (S0, S1)**

- 항상 둘 중 하나는 비어 있음
- Minor GC마다 살아있는 객체를 S0 ↔ S1 사이에서 복사(Copy)
- 객체마다 **age counter** 가 증가
- `-XX:MaxTenuringThreshold` (기본 15) 초과 시 Old Gen으로 이동

---

### Minor GC 동작 순서

```
1. Eden이 꽉 참
2. GC Roots에서 도달 가능한 객체 마킹 (Stop-The-World)
3. 살아있는 Eden + 현재 Survivor 객체 → 빈 Survivor로 복사
4. age++ 처리, 임계값 초과 객체는 Old Gen으로 Promote
5. Eden + 방금 비워진 Survivor를 통째로 초기화 (매우 빠름)
```

"통째로 초기화"가 핵심이다. 
Mark-Sweep처럼 살아있는 것만 남기는 게 아니라, (마킹 - 마킹 안된 객체 제거) 
**살아있는 것만 다른 곳으로 복사하고 전체를 버린다** (Copying GC). 
그래서 단편화가 없다.

---

### Old Generation (Tenured)

승격된 객체, 또는 **Eden에서 바로 할당하기 너무 큰 객체**가 들어온다.

```java
// 아래처럼 초기부터 큰 배열을 만들면
// Eden을 건너뛰고 바로 Old Gen에 할당될 수 있음
byte[] largeBuffer = new byte[1024 * 1024 * 50]; // 50MB
```

Old Gen이 꽉 차면 **Major GC (Full GC)** 가 발생한다.

| 구분     | Young Gen (Minor GC) | Old Gen (Major/Full GC) |
| ------ | -------------------- | ----------------------- |
| 빈도     | 잦음 (초 단위)            | 드뭄 (분 단위)               |
| STW 시간 | 수 ms                 | 수백 ms ~ 수 초             |
| 알고리즘   | Copying              | Mark-Sweep-Compact      |
| 단편화    | 없음                   | 발생 가능                   |

Old Gen GC는 **Mark-Sweep-Compact** 방식이다.

```
Mark:    GC Root에서 도달 가능한 객체에 마킹
Sweep:   마킹 안 된 객체 제거
Compact: 살아있는 객체를 앞쪽으로 당겨 단편화 제거
```

Compact 단계가 비싸다. 전체 힙을 스캔하면서 포인터를 재조정해야 하기 때문이다.

---

### Metaspace (Java 8+)

Java 7까지는 **PermGen(Permanent Generation)** 이 Heap 안에 있었다.

```
Java 7: [ Eden | S0 | S1 | Old Gen | PermGen ]  ← 전부 Heap
Java 8: [ Eden | S0 | S1 | Old Gen ] + Metaspace (Native Memory)
```

PermGen의 문제:

- 크기가 고정 (`-XX:MaxPermSize`)
- 동적 클래스 생성이 많은 환경(리플렉션, 프록시, JSP 컴파일 등)에서
`OutOfMemoryError: PermGen space` 가 빈번하게 발생
- 튜닝하기 어렵고, 크게 잡으면 낭비, 작게 잡으면 OOM

Metaspace는 **Native Memory** (OS가 관리하는 힙 바깥)를 사용한다.

```java
// 이런 코드가 클래스를 동적으로 계속 만들면
// Metaspace가 계속 늘어남
Proxy.newProxyInstance(
    classLoader,
    new Class[]{SomeInterface.class},
    handler
);
```

Metaspace가 저장하는 것:

- 클래스 메타데이터 (이름, 필드, 메서드 목록)
- 상수 풀 (Constant Pool) — _단, 문자열 리터럴은 Java 7부터 Heap의 String Pool로 이동됨_
- JIT 컴파일된 코드 정보 (Code Cache는 별도)

기본적으로 크기 제한이 없어서 프로세스 메모리를 다 잡아먹을 수 있다.
**`-XX:MaxMetaspaceSize`를 반드시 설정해야 한다.**

---

### GC 알고리즘과 Heap 구조의 관계

GC 알고리즘에 따라 이 세대 구분을 다르게 사용한다.

```
Serial GC:     위 구조 그대로, 단일 스레드
Parallel GC:   위 구조 그대로, 멀티 스레드
G1GC (Java 9+ default):
  힙을 Region으로 나누고, 각 Region이 동적으로
  Eden/Survivor/Old 역할을 맡음
  → 전통적인 연속 공간 구분이 사라짐
ZGC / Shenandoah:
  세대 구분을 약화시키거나 없앰 (ZGC는 Java 21에서 세대형 ZGC 도입)
```

---

## 3. 공식 근거

**Weak Generational Hypothesis**

> "Most objects die young."

JVM GC 설계의 핵심 전제. Oracle 공식 GC 튜닝 가이드에 명시되어 있다.

> _"The hypothesis is that most objects die young. In other words, objects tend to be allocated in large quantities and become unreachable soon after creation."_ — Oracle, [Java SE HotSpot Virtual Machine Garbage Collection Tuning Guide](https://docs.oracle.com/en/java/se/21/gctuning/garbage-collector-implementation.html)
> 
> _대부분의 객체는 일찍 죽는다. 즉, 객체는 대량으로 할당된 후 생성 직후 곧 도달 불가능해지는 경향이 있다._

---

**PermGen → Metaspace 전환 근거 (JEP 122)**

> _"This is part of the JRockit and Hotspot convergence effort. JRockit customers do not need to configure the permanent generation (since JRockit does not have a permanent generation) and are accustomed to not configuring the permanent generation."_ — [JEP 122: Remove the Permanent Generation](https://openjdk.org/jeps/122)
> 
> _이것은 JRockit과 HotSpot 통합 작업의 일환이다. JRockit 사용자들은 permanent generation을 설정할 필요가 없었으며(JRockit에는 permanent generation이 없으므로), 이를 설정하지 않는 것에 익숙하다._

---

**TLAB (Thread-Local Allocation Buffer)**

> _"Each Java application thread has a thread-local allocation buffer (TLAB) in the young generation that it uses to allocate objects without synchronization."_ — Oracle GC Tuning Guide
> 
> _각 Java 애플리케이션 스레드는 Young Generation 내에 스레드 로컬 할당 버퍼(TLAB)를 가지며, 
> 동기화 없이 객체를 할당하는 데 사용한다._

---

**Effective Java, Item 6** (Bloch)

불필요한 객체 생성이 Young Gen 압박을 높인다는 맥락에서 자주 인용된다.

> _"Avoid creating unnecessary objects"_

직접적인 GC 언급은 없지만, 이 항목의 성능 근거가 Young Gen GC 비용이다.

---

## 4. 트레이드오프

**세대형 GC의 전제가 깨지는 경우**

```java
// LRU 캐시처럼 오래된 객체를 계속 참조 유지하면
// Old Gen이 빠르게 차고 Full GC 빈도가 높아짐
private final Map<String, LottoResult> cache =
    Collections.synchronizedMap(
        new LinkedHashMap<>(MAX_SIZE, 0.75f, true) {
            protected boolean removeEldestEntry(Map.Entry e) {
                return size() > MAX_SIZE;
            }
        }
    );
```

오래 사는 객체가 많을수록 Young Gen의 Copying 비용만 쌓이고 결국 Old Gen으로 다 올라간다.

**G1GC로 넘어가면 튜닝 방식이 달라진다**

전통적인 `-XX:NewRatio`, `-XX:SurvivorRatio` 튜닝이 G1GC에서는 덜 유효하다. 
G1GC는 Region 크기와 Pause Target (`-XX:MaxGCPauseMillis`)으로 제어하는 모델이기 때문이다.

**Metaspace는 Native Memory를 먹는다**

컨테이너(Docker/k8s)에서 `-Xmx`만 보고 메모리를 잡으면 실제 프로세스 메모리가 이를 초과할 수 있다. Metaspace + Code Cache + Stack이 추가로 필요하다.

```
실제 프로세스 메모리 ≈ -Xmx + Metaspace + Code Cache + 스레드 스택 + OS overhead
```

---

## 5. 안티패턴 검증

### 개념 자체의 오남용

**Young Gen을 너무 크게 잡는 것**

```bash
# 안 좋은 예: Young Gen을 전체 힙의 절반 이상으로 설정
-Xmx4g -XX:NewSize=3g
```

Young Gen이 크면 Minor GC의 STW 시간이 길어진다. Copying 대상이 늘기 때문이다. 반드시 실측 기반으로 잡아야 한다.

**`-XX:MaxMetaspaceSize`를 안 잡는 것**

개발 환경에서는 문제없다가 동적 프록시나 리플렉션 기반 프레임워크(Spring AOP, Hibernate 등)가 클래스를 계속 만드는 환경에서 프로세스가 OS 메모리를 다 잡아먹을 수 있다.

---

### 잘못된 적용 방식

**finalize() 에 의존한 자원 해제**

```java
// 안티패턴
@Override
protected void finalize() throws Throwable {
    // GC가 이 객체를 수거할 때 호출되길 기대하는 코드
    connection.close();
}
```

`finalize()`는 GC가 Old Gen을 수거할 때 호출되는데, 호출 시점이 전혀 보장되지 않는다. 
Java 9에서 `@Deprecated`, Java 18에서 `@Deprecated(forRemoval=true)` 처리됐다. 
`try-with-resources` 또는 `Cleaner` (Java 9+)를 써야 한다.

_Effective Java, Item 8: "Avoid finalizers and cleaners"_

**`System.gc()` 호출**

```java
// 안티패턴
System.gc(); // Full GC를 "요청"하지만 보장은 없음
```

GC는 JVM이 판단해서 실행하는 것이다. `System.gc()`는 힌트일 뿐이고, 대부분의 상황에서 불필요한 Full GC를 유도해 오히려 성능을 떨어뜨린다.

---

### 커뮤니티 논쟁: G1GC vs ZGC 전환 시점

G1GC가 Java 9부터 기본값이 됐지만, 저지연이 중요한 환경에서 ZGC로 전환할 시점에 대한 논쟁이 있다.

- **G1GC 유지 입장**: 튜닝 레퍼런스가 풍부하고, 대부분의 워크로드에서 충분하다. ZGC는 처리량(throughput)이 상대적으로 낮다.
- **ZGC 전환 입장**: 힙이 수십 GB 이상이거나 p99 지연이 중요한 서비스에서는 ZGC의 concurrent 수거가 훨씬 유리하다. Java 21의 Generational ZGC는 처리량 문제도 개선됐다.

공식 입장은 없고 워크로드에 따라 실측해야 한다.

---

## 6. 이어지는 개념

이 개념을 제대로 소화했다면 자연스럽게 다음으로 이어진다.

```
JVM Heap 구조
    │
    ├── GC 알고리즘 심화
    │     ├── G1GC Region 구조 및 Mixed GC
    │     ├── ZGC / Shenandoah의 Concurrent Marking
    │     └── Generational ZGC (Java 21)
    │
    ├── GC 튜닝 실전
    │     ├── GC 로그 분석 (-Xlog:gc*)
    │     ├── GC 지표 해석 (Throughput, Latency, Footprint)
    │     └── 컨테이너 환경 메모리 설정 (-XX:+UseContainerSupport)
    │
    ├── 객체 생명주기와 코드 설계
    │     ├── 불필요한 객체 생성 패턴 (Effective Java Item 6)
    │     ├── 메모리 누수 패턴 (static 컬렉션, 리스너 미해제)
    │     └── WeakReference / SoftReference 활용
    │
    └── JVM 메모리 전체 모델
          ├── Stack vs Heap (지역변수 vs 객체)
          ├── String Pool (Heap 내 위치 변화, Java 7+)
          └── Code Cache / JIT 컴파일
```