#CS/CPU #CS/JVM #Status/보충 
## 1. Problem First

CPU는 기계어만 실행할 수 있다.

```
CPU가 이해하는 것:
10110000 01100001  (x86 기계어: AL 레지스터에 0x61 로드)

CPU가 이해 못하는 것:
print("hello")     (Python)
System.out.println (Java 바이트코드)
```

사람이 작성한 고수준 코드를 CPU가 실행하려면 번역이 필요하다. 번역 방식이 두 가지로 갈린다.

```
방식 1: 미리 전부 번역해놓고 실행  → 컴파일러
방식 2: 실행하면서 한 줄씩 번역    → 인터프리터
```

---

## 2. Mechanics

### 컴파일러 방식

```
소스코드 전체
    │
    │ (빌드 타임에 전부 번역)
    ▼
기계어 바이너리
    │
    │ (실행 타임)
    ▼
CPU가 직접 실행
```

```
C 코드:  int a = 1 + 2;
    ↓ gcc 컴파일
기계어: mov eax, 1
        add eax, 2
        mov [a], eax
```

실행 전에 번역이 끝나있다. 실행할 때는 번역 비용이 없다.

---

### 인터프리터 방식

```
소스코드
    │
    │ (실행 타임, 한 줄씩)
    ▼
인터프리터가 읽고 → 해석하고 → 실행
    │
    ▼
다음 줄 읽고 → 해석하고 → 실행
    │
    ▼
반복
```

번역과 실행이 동시에 일어난다. 인터프리터 자체는 기계어로 만들어진 프로그램이고, 소스코드를 입력으로 받아서 그 의미에 맞는 동작을 수행한다.

---

### 인터프리터 내부 구조

인터프리터는 결국 **거대한 switch문**이다.

```c
// 인터프리터 내부를 의사코드로 표현하면
while (hasNextInstruction()) {
    instruction = fetch();  // 다음 명령어 읽기
    
    switch (instruction.opcode) {
        case ADD:
            result = stack.pop() + stack.pop();
            stack.push(result);
            break;
            
        case LOAD:
            stack.push(variables[instruction.operand]);
            break;
            
        case PRINT:
            System.out.println(stack.pop());
            break;
        
        // ... 수백 개의 case
    }
}
```

명령어를 읽고, 그 명령어에 해당하는 동작을 수행하고, 반복한다.

---

### JVM에서의 인터프리터

Java 바이트코드는 기계어가 아니다. JVM이 정의한 **가상 명령어 셋**이다.

```
Java 소스:      int a = 1 + 2;

바이트코드:     iconst_1    // 정수 1을 스택에 push
                iconst_2    // 정수 2를 스택에 push
                iadd        // 스택 top 두 개를 pop해서 더하고 push
                istore_1    // 스택 top을 변수 slot 1에 저장
```

HotSpot Interpreter는 이 바이트코드를 읽으면서 위의 switch 구조로 실행한다. CPU는 바이트코드를 모른다. 인터프리터가 중간에서 통역한다.

---

### 왜 인터프리터가 느린가

같은 연산을 컴파일러 방식과 비교하면:

```
컴파일러 방식 (C):
→ add eax, ebx        (기계어 1줄, CPU 직접 실행)

인터프리터 방식 (JVM):
→ iadd 명령어 읽기
→ switch에서 iadd case 찾기
→ 스택에서 두 값 pop
→ 더하기
→ 결과 push
→ 다음 명령어로 이동
(기계어 수십 줄 실행)
```

같은 덧셈 하나에 인터프리터는 훨씬 많은 기계어를 실행한다. 이 오버헤드가 누적되면 성능 차이가 크게 벌어진다.

---

### 인터프리터가 존재하는 이유

느린데 왜 쓰는가. 컴파일 방식이 줄 수 없는 것들을 준다.

**이식성**

```
C로 컴파일한 바이너리:
→ x86 Windows에서 컴파일하면 ARM Linux에서 안 돌아감
→ CPU 아키텍처마다 다른 기계어가 필요

Java 바이트코드:
→ 어디서 컴파일해도 동일한 바이트코드
→ JVM(인터프리터)만 있으면 어디서든 실행
→ "Write once, run anywhere"
```

**동적 실행**

```python
# Python
code = "1 + 2"
eval(code)  # 문자열을 런타임에 코드로 실행 가능
```

컴파일 방식은 빌드 타임에 코드가 확정되어야 한다. 인터프리터는 런타임에 코드를 받아서 실행할 수 있다.

**즉시 실행**

```
컴파일: 소스 → (컴파일 대기) → 실행
인터프리터: 소스 → 즉시 실행
```

스크립트, REPL(Read-Eval-Print Loop), 쉘 명령어가 인터프리터 방식인 이유다.

---

### 실행 모델 전체 지형

```
순수 컴파일
(C, C++, Rust)
소스 → 기계어 → 실행
장점: 빠름
단점: 이식성 없음, 빌드 필요

─────────────────────────────

순수 인터프리터
(초기 Python, Ruby, Shell)
소스 → 인터프리터 → 실행
장점: 이식성, 즉시 실행
단점: 느림

─────────────────────────────

중간 방식 (Bytecode + VM)
(Java, Python 3, C#)
소스 → 바이트코드 → VM(인터프리터+JIT) → 실행
장점: 이식성 + 어느정도 성능
단점: 워밍업, VM 메모리 오버헤드

─────────────────────────────

JIT 특화
(JavaScript V8, LuaJIT)
런타임에 핫 코드를 기계어로 즉시 컴파일
장점: 인터프리터 이식성 + 컴파일 성능에 근접
단점: 컴파일 오버헤드, 메모리
```

Java는 세 번째다. 바이트코드로 이식성을 확보하고, 인터프리터로 시작하다가 JIT으로 성능을 끌어올린다.

---

## 3. 공식 근거

**JVM 명세 — 바이트코드 실행 모델**

> _"The Java Virtual Machine knows nothing of the Java programming language, only of a particular binary format, the class file format."_ — [JVM Specification §1.2](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-1.html)
> 
> _JVM은 Java 프로그래밍 언어에 대해 아무것도 모른다. 오직 특정 바이너리 형식인 class 파일 형식만 안다._

JVM은 언어가 아닌 바이트코드 형식에 묶여있다는 뜻이다. 
Kotlin, Scala, Groovy가 JVM에서 돌아가는 이유가 여기 있다. 전부 같은 바이트코드로 컴파일된다.

---

## 한 줄 정리

> 인터프리터는 "코드를 실행하면서 한 줄씩 번역하는 프로그램"이다. CPU가 모르는 언어를 CPU가 아는 기계어 동작으로 통역해준다. 느리지만 이식성과 즉시 실행을 준다. JVM은 이 단점을 JIT으로 보완한다.