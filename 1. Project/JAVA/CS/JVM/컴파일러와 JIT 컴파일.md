#CS/CPU #CS/JVM #Status/완료
## 1. Problem First

두 개의 문제가 있다.

**문제 1 — 컴파일러의 한계: 이식성**

```
C 코드를 x86 Windows에서 컴파일
→ x86 Windows 전용 기계어 생성
→ ARM Mac에서 실행 불가
→ Linux 서버에서 실행 불가
```

CPU 아키텍처마다, OS마다 기계어가 다르다. 코드를 배포할 환경이 여러 개면 환경별로 각각 컴파일해야 한다.

**문제 2 — 인터프리터의 한계: 성능**

```
Python 인터프리터로 1억 번 루프:
→ 매 iteration마다 바이트코드 읽기 → 해석 → 실행
→ 같은 연산을 1억 번 반복해서 해석
→ C 컴파일 코드 대비 수십~수백 배 느림
```

이미 해석한 코드를 또 해석한다. 낭비다.

**Java가 풀려는 문제**

```
이식성 (인터프리터의 장점) + 성능 (컴파일러의 장점)
```

이 두 개를 동시에 잡으려고 나온 게 **바이트코드 + JIT** 조합이다.

---

## 2. Mechanics

### 컴파일러 — 기본 동작

컴파일러는 소스코드 전체를 받아서 다른 언어로 번역하는 프로그램이다.

```
입력: 소스코드 (고수준)
출력: 번역된 코드 (저수준 또는 다른 형태)
시점: 실행 전 (빌드 타임)
```

C 컴파일러(gcc)가 하는 일을 단계별로 보면:

```
소스코드 (.c)
    │
    ▼
전처리 (Preprocessor)
→ #include, #define 처리
→ 매크로 치환
    │
    ▼
컴파일 (Compiler)
→ 소스코드 → 어셈블리어
→ 문법 검사, 타입 검사
→ 최적화 수행
    │
    ▼
어셈블 (Assembler)
→ 어셈블리어 → 기계어 오브젝트 파일 (.o)
    │
    ▼
링크 (Linker)
→ 여러 오브젝트 파일 + 라이브러리 합치기
→ 실행 가능한 바이너리 생성
```

결과물은 특정 CPU 아키텍처와 OS에 종속된 기계어다.

### 컴파일러가 할 수 있는 최적화

빌드 타임에 코드 전체를 볼 수 있기 때문에 정적 분석이 가능하다.

**상수 폴딩 (Constant Folding)**

```c
// 소스코드
int x = 2 * 3 * 4;

// 컴파일 후 (런타임에 계산 안 함)
int x = 24;
```

**데드 코드 제거**

```c
// 소스코드
if (false) {
    doSomething(); // 절대 실행 안 됨
}

// 컴파일 후: 해당 블록 자체가 사라짐
```

단, 컴파일러는 **런타임 정보가 없다.** 실제로 어떤 값이 들어올지, 어떤 분기가 자주 실행될지 모른다.

---

### Java 컴파일러 (javac) — 특이점

Java의 `javac`는 기계어를 만들지 않는다.

```
소스코드 (.java)
    │  javac
    ▼
바이트코드 (.class)   ← 기계어가 아님
                         JVM이 정의한 가상 명령어
```

바이트코드는 어떤 CPU도 직접 실행할 수 없다. JVM이 있는 곳이라면 어디서든 실행 가능하다는 게 목적이다.

```
같은 .class 파일
    ├── x86 Windows JVM → x86 Windows 기계어로 실행
    ├── ARM Mac JVM     → ARM 기계어로 실행
    └── Linux JVM       → Linux 기계어로 실행
```

이식성 문제가 해결된다. 성능 문제가 남는다.

---

### JIT 컴파일러 — 핵심 아이디어

**Just-In-Time**: 필요한 순간에 컴파일한다.

```
인터프리터로 실행하면서 관찰
    │
    │ 이 메서드, 자주 호출되네 (hot)
    ▼
그 시점에 기계어로 컴파일
    │
    ▼
이후 호출은 기계어 직접 실행
```

인터프리터의 이식성을 유지하면서, 자주 실행되는 코드만 골라서 컴파일한다. 전체를 미리 컴파일하지 않으니 시작이 빠르고, 자주 쓰는 코드는 결국 기계어로 돌아가니 빠르다.

### HotSpot JIT — 구체적인 동작

```
메서드 호출 카운터 증가
    │
    ├── 카운터 < 임계값 → Interpreter 실행 계속
    │
    └── 카운터 >= 임계값 (기본 10,000)
            │
            ▼
        C1 컴파일러로 컴파일 (빠른 컴파일, 기본 최적화)
            │
            ▼
        C1 코드로 실행하면서 프로파일링 계속
            │
            ├── 더 hot해짐
            ▼
        C2 컴파일러로 재컴파일 (느린 컴파일, 공격적 최적화)
            │
            ▼
        C2 코드로 실행
```

이게 **Tiered Compilation**이다. Java 8부터 기본값이다.

```
Tier 0: Interpreter
Tier 1: C1 (프로파일링 없음, 단순 메서드)
Tier 2: C1 (제한적 프로파일링)
Tier 3: C1 (전체 프로파일링)
Tier 4: C2 (프로파일 데이터 기반 최적화)
```

### JIT이 컴파일러보다 잘하는 것 — 런타임 정보 활용

정적 컴파일러는 런타임 정보가 없다. JIT은 있다.

**가상 메서드 인라이닝**

```java
// 소스코드
interface Printer {
    void print(String s);
}

void process(Printer p, String s) {
    p.print(s); // 어떤 구현체가 올지 컴파일 타임엔 모름
}
```

정적 컴파일러는 `p.print()`가 어떤 구현체를 호출할지 모른다. 가상 디스패치(virtual dispatch) 비용이 매번 발생한다.

```java
// JIT이 런타임에 관찰
// "이 코드에서 p는 항상 ConsolePrinter였다"
// → ConsolePrinter.print()를 직접 인라이닝
void process(Printer p, String s) {
    // 가상 디스패치 제거, 직접 코드 삽입
    System.out.println(s); // ConsolePrinter 구현 내용
}
```

이걸 **Speculative Optimization(추측 최적화)** 이라 한다.
 "지금까지 항상 이랬으니 앞으로도 그럴 것"이라고 추측하고 최적화한다.

추측이 틀리면(다른 구현체가 들어오면)?

```
Deoptimization 발생
→ JIT 코드 폐기
→ Interpreter로 fallback
→ 다시 프로파일링 시작
```

**분기 예측 최적화**

```java
for (int i = 0; i < 1_000_000; i++) {
    if (i % 2 == 0) {  // 50만 번 true, 50만 번 false
        doEven();
    } else {
        doOdd();
    }
}
```

JIT은 실행 중에 이 분기가 어느 쪽으로 얼마나 가는지 카운팅한다. 자주 가는 쪽을 빠른 경로(fast path)로 배치한다.

---

### 컴파일러 vs 인터프리터 vs JIT 비교

```
              컴파일러        인터프리터      JIT
─────────────────────────────────────────────────
시작 속도                      느림            빠름           중간
              (빌드 필요)     (즉시 실행)    (워밍업 필요)

실행 속도                        빠름            느림           빠름
              (기계어)        (해석 오버헤드) (워밍업 후)

이식성                            없음            있음           있음
              (환경별 빌드)   (VM만 있으면)  (VM만 있으면)

런타임 최적화                 없음            없음           있음
              (정적 분석만)                  (프로파일 기반)

메모리                           적음            적음           Code Cache 필요
```

---

## 3. 공식 근거

**Tiered Compilation — JEP 197 맥락**

Tiered Compilation은 Java 7에서 실험적으로 도입되고 Java 8에서 기본값이 됐다.

> _"Tiered compilation, introduced in Java SE 7, brings client startup speeds to the server VM."_ — [Oracle, Java SE 8 Performance White Paper](https://www.oracle.com/java/technologies/javase/8-whitepapers.html)
> 
> _Java SE 7에서 도입된 Tiered Compilation은 클라이언트 수준의 시작 속도를 서버 VM에 제공한다._

**Speculative Optimization과 Deoptimization**

> _"The server compiler also performs a number of classic loop optimizations, such as induction variable simplification and loop unrolling."_ — Oracle HotSpot Performance Documentation
> 
> _서버 컴파일러는 귀납 변수 단순화 및 루프 언롤링과 같은 여러 고전적인 루프 최적화도 수행한다._

공식 근거를 찾지 못한 부분: Deoptimization의 구체적인 트리거 조건은 HotSpot 소스코드 수준의 내용으로, 공식 문서에 상세히 기술되지 않는다. 커뮤니티 수준의 관행이다.

---

## 4. 트레이드오프

**JIT의 워밍업 비용**

```
배포 직후 트래픽
→ 전부 Interpreter 또는 C1 수준
→ 수십 초 ~ 수 분간 응답 느림
→ 이후 C2 최적화 완료되면 정상 성능
```

Lambda, 서버리스처럼 인스턴스가 자주 뜨고 내려가는 환경에서 심각하다. 콜드 스타트마다 워밍업을 반복한다.

GraalVM Native Image(AOT)가 이 문제의 대안이다. 빌드 타임에 전부 기계어로 컴파일해서 워밍업이 없다. 대신 JIT의 런타임 최적화를 포기한다.

```
JIT:  시작 느림, 장기 실행 시 최고 성능
AOT:  시작 빠름, 장기 실행 시 JIT보다 낮은 성능
```

**C2의 컴파일 비용**

C2는 공격적인 최적화를 하기 때문에 컴파일 자체가 느리고 CPU를 쓴다. 컴파일하는 동안 그 CPU 자원이 실제 요청 처리에 못 쓰인다. 트래픽이 갑자기 몰리는 순간에 워밍업이 겹치면 최악이다.

---

## 한 줄 정리

> 컴파일러는 실행 전에 전체를 번역해서 빠르지만 이식성이 없다. JIT은 실행하면서 뜨거운 코드만 골라 번역한다. 이식성을 유지하면서 런타임 정보까지 활용한 최적화가 가능하다. 단, 워밍업 비용이 있고 장기 실행 서버에 유리한 모델이다.


좋은 질문이다. 개념을 정확하게 짚고 넘어가야 한다.

---



## 실제로 Java를 바로 기계어로 만드는 것들

이미 존재한다.

```
GraalVM Native Image
→ Java 소스를 AOT(Ahead-Of-Time) 컴파일
→ JVM 없이 실행 가능한 네이티브 바이너리 생성
→ C처럼 플랫폼별로 따로 빌드해야 함
→ 시작 빠름, JIT 최적화 없음
```

Spring Boot 3.x가 이걸 공식 지원한다. Lambda처럼 콜드 스타트가 중요한 환경에서 쓴다.

---

## 한 줄 정리

```
C:    소스 → 기계어          (이식성 없음, 빠름)
Java: 소스 → 바이트코드 → JVM → 기계어  (이식성 있음, 워밍업 있음)
GraalVM Native: 소스 → 기계어  (Java인데 C처럼, 이식성 없음, JIT 없음)
```

> 바이트코드는 기술적 한계가 아니라 이식성을 위한 설계 선택이다. 기계어로 바로 만드는 것도 가능하고, GraalVM이 실제로 그렇게 한다.