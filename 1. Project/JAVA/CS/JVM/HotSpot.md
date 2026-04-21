#CS/JVM #Status/완료 
## 1. Problem First

[[JVM/JVM|JVM]]은 명세(specification)다. Java 코드가 어떻게 동작해야 하는지를 정의한 문서다. 명세만 있으면 실제로 실행이 안 된다. 구현체가 필요하다.

JVM 명세를 구현하는 방법은 여러 가지다. 바이트코드를 한 줄씩 읽어서 해석만 해도 되고, 기계어로 컴파일해서 실행해도 되고, 둘을 섞어도 된다.

어떻게 구현하느냐에 따라 성능이 수십 배 차이가 난다. HotSpot은 이 중 가장 오래되고 가장 많이 쓰이는 구현체다.

---

## 2. Mechanics

### HotSpot의 위치

```
Java 소스코드 (.java)
        │  javac 컴파일
        ▼
바이트코드 (.class)        ← JVM 명세가 정의하는 영역
        │
        ▼
   HotSpot JVM            ← 여기서부터 구현체
   ┌─────────────────────────────────┐
   │  Class Loader                   │
   │  Runtime Data Areas             │
   │    - Heap (Young/Old/Metaspace) │
   │    - Stack                      │
   │    - Method Area                │
   │  Execution Engine               │
   │    - Interpreter                │
   │    - JIT Compiler (C1, C2)      │
   │  GC (G1GC, ZGC, ...)           │
   └─────────────────────────────────┘
        │
        ▼
   기계어 실행
```

### HotSpot이라는 이름의 유래

프로그램 실행 시간의 대부분은 **코드의 일부분(hot spot)**에서 소비된다. 80/20 법칙과 같은 맥락이다.

```
전체 코드의 20%가 실행 시간의 80%를 차지한다
→ 그 20%를 찾아서 집중적으로 최적화하면 된다
```

이 아이디어를 구현한 게 **JIT(Just-In-Time) 컴파일러**다.

### JIT 컴파일 — [[../CPU/Interpreter|Interpreter]]와의 조합

HotSpot은 처음부터 기계어로 컴파일하지 않는다.

```
메서드 최초 호출
    │
    ▼
Interpreter로 실행 (느리지만 즉시 시작 가능)
    │
    │ 호출 횟수 카운팅
    ▼
임계값 초과? (기본 10,000회)
    │
    ▼
JIT 컴파일러가 기계어로 컴파일
    │
    ▼
이후 호출은 기계어 직접 실행 (빠름)
```

JIT 컴파일러는 두 단계가 있다.

```
C1 컴파일러 (Client Compiler)
→ 빠르게 컴파일, 기본적인 최적화
→ 초기 응답 속도가 중요한 경우

C2 컴파일러 (Server Compiler)
→ 느리게 컴파일, 공격적인 최적화
→ 장시간 실행되는 서버 애플리케이션에 적합

Tiered Compilation (Java 8~, 기본값)
→ C1으로 먼저 컴파일 → 더 hot해지면 C2로 재컴파일
```

### HotSpot이 하는 대표적인 최적화

**인라이닝 (Inlining)**

```java
// 소스코드
int result = add(a, b);

int add(int x, int y) { return x + y; }

// JIT가 실제로 실행하는 코드 (메서드 호출 제거)
int result = a + b;
```

**Escape Analysis**

```java
public int calculate() {
    Point p = new Point(1, 2); // p가 이 메서드 밖으로 나가지 않음
    return p.x + p.y;
}

// JIT가 판단: p는 밖으로 탈출하지 않음
// → 힙 할당 대신 스택에 올리거나 아예 변수로 대체
// → GC 부담 감소
```

**Null Check 제거, 범위 검사 제거** 등 런타임 정보를 활용한 최적화도 수행한다. 
정적 컴파일러(javac)는 할 수 없는 최적화들이다.

### HotSpot의 구현체 계보

```
Sun JDK (원조)
    │
    ▼
Oracle JDK (Oracle이 Sun 인수 후)
    │
    ├── OpenJDK (오픈소스 버전, 사실상 동일한 HotSpot)
    │
    └── GraalVM (HotSpot + GraalJIT, AOT 컴파일 지원)
```

현업에서 쓰는 JDK 배포판들(Amazon Corretto, Azul Zulu, Eclipse Temurin)은 전부 OpenJDK 기반이고, HotSpot을 그대로 쓴다.

---

## 3. 공식 근거

**JVM 명세와 구현체의 분리**

> _"The Java Virtual Machine is an abstract computing machine."_ — [JVM Specification, Chapter 1](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-1.html)
> 
> _Java Virtual Machine은 추상적인 컴퓨팅 기계다._

명세는 "무엇을"만 정의하고, "어떻게"는 구현체에 위임한다.

**Tiered Compilation 공식 문서**

JEP 197에서 Tiered Compilation이 Java 8 기본값으로 확정됐다.

> — [JEP 197: Segmented Code Cache](https://openjdk.org/jeps/197) 관련 맥락

**Escape Analysis**

> _"Escape analysis is a technique by which the Java HotSpot Server Compiler can analyze the scope of a new object's uses and decide whether to allocate it on the Java heap."_ — [Oracle, Escape Analysis](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/performance-enhancements-7.html)
> 
> _Escape Analysis는 HotSpot Server Compiler가 새 객체의 사용 범위를 분석하여 Java 힙에 할당할지 여부를 결정하는 기법이다._

---

## 4. 트레이드오프

**JIT 컴파일의 워밍업 비용**

```
애플리케이션 시작 직후
→ 모든 코드가 Interpreter로 실행
→ 느린 구간 (Warm-up period)
→ 수십 초 ~ 수 분 후에야 최대 성능 도달
```

람다나 서버리스처럼 **콜드 스타트가 중요한 환경**에서 문제가 된다. 이걸 해결하려고 나온 게 GraalVM의 AOT(Ahead-Of-Time) 컴파일이다. 대신 AOT는 JIT의 런타임 최적화를 포기한다.

**HotSpot의 메모리 오버헤드**

JIT 컴파일된 코드는 **Code Cache**에 저장된다. 이 공간도 메모리를 차지한다.

```
실제 프로세스 메모리 = Heap + Metaspace + Code Cache + Stack + ...
```

컨테이너 환경에서 `-Xmx`만 보고 메모리 제한을 잡으면 Code Cache 때문에 OOM Killed가 발생할 수 있다.

---
## 한 줄 정리

> JVM 명세의 구현체. 
> Interpreter + JIT 조합으로 런타임 정보를 활용한 최적화를 수행한다. 
> 우리가 `java` 명령어를 실행할 때 실제로 돌아가는 프로그램이다.


