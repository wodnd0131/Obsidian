#CS/JVM #Status/보충
바이트코드 실행 + 메모리 관리(GC) + 플랫폼 추상화 이 세 가지를 하나로 묶은 것이 JVM이다.
# 전체 구조 한 눈에

![[../../../../../Repository/JVM-1.png]]

## 각 부분이 왜 있는가

### ① Class Loader — "언제 올리나"

```
처음부터 모든 클래스를 메모리에 올리지 않는다.
실제로 사용될 때 올린다. (Lazy Loading)

Spring 앱 기준 수천 개의 클래스가 있어도
시작 시점에 전부 올리지 않고 필요할 때 올린다.
```

Parent Delegation의 실무 의미:

```java
// 이런 코드가 있어도
public class String {  // java.lang.String과 같은 이름
    // ...
}

// **Bootstrap ClassLoader**가 먼저 진짜 java.lang.String을 찾아서 씀
// 가짜 String이 끼어들 수 없음 → 보안
```

### ② Runtime Data Areas — "어디에 저장하나"

```
Heap   → new로 만든 객체
Stack  → 지역변수, 메서드 호출 순서
           메서드 들어가면 Frame 추가
           메서드 나오면 Frame 제거

Metaspace → 클래스 자체의 정보
             (필드 목록, 메서드 목록, 상수)
```

스레드 공유 여부가 핵심이다:

```
공유 (모든 스레드가 같이 씀)
→ Heap, Metaspace
→ 동시성 문제가 생기는 곳

독립 (스레드마다 따로)
→ Stack, PC Register
→ 동시성 문제 없음
```

### ③ Execution Engine — "어떻게 실행하나"

```
처음엔 Interpreter로 즉시 실행 (느리지만 바로 시작)
        │
        │ 호출 횟수 카운팅
        ▼
자주 호출되면 JIT이 기계어로 컴파일
        │
        ▼
이후엔 기계어 직접 실행 (빠름)
```

[[컴파일러와 JIT 컴파일|JIT]]이 할 수 있는 것:

```
인라이닝       : 메서드 호출을 코드로 대체
Escape Analysis: 힙 할당을 스택으로 대체 가능
추측 최적화    : "항상 이 타입이었으니 그럴 것" → 가상 디스패치 제거
                 틀리면 Deoptimization → Interpreter로 fallback
```

[[GC/GC (Garbage Collection)|GC]]는 Heap만 관리한다:

```
Young Gen (Eden + Survivor)
→ Minor GC, 자주, 짧게
→ Copying 방식, 단편화 없음

Old Gen
→ Major GC, 드물게, 길게
→ Mark-Sweep-Compact
→ STW 시간이 길어서 서비스 지연 유발`
```

### ④ JNI — "Java 밖과 어떻게 통신하나"

```
java.io, java.net 등 JDK 내부도
결국 JNI를 통해 OS 시스템 콜을 호출한다.

개발자가 직접 쓸 일은 드물지만
JVM 자체가 OS와 소통하는 통로다.
```

---

## 실행 흐름 전체를 한 번에

```
1. java MyApp 명령어 실행
        │
2. JVM 프로세스 시작
   Heap, Metaspace, Stack 등 메모리 초기화
        │
3. Bootstrap ClassLoader가 java.lang.* 로딩
        │
4. Application ClassLoader가 MyApp.class 로딩
   → Metaspace에 클래스 메타데이터 저장
        │
5. main() 메서드 찾아서 실행 시작
   → main 스레드의 Stack에 Frame 추가
        │
6. Interpreter가 바이트코드 한 줄씩 실행
   new 만나면 → Heap에 객체 할당 (TLAB 사용)
        │
7. 호출 횟수 누적 → JIT이 핫 메서드 컴파일
   → Code Cache에 기계어 저장
   → 이후 호출은 기계어 직접 실행
        │
8. Eden이 꽉 참 → Minor GC 발생
   살아있는 객체 → Survivor로 복사
   오래된 객체  → Old Gen으로 승격
        │
9. 프로그램 종료 → JVM 프로세스 종료
   모든 메모리 반환
```

---

## 면접에서 나오는 포인트 요약

| 질문                    | 핵심 답변                                            |
| --------------------- | ------------------------------------------------ |
| JVM이 뭔가요              | 바이트코드 실행 환경. 이식성 제공. 명세와 구현체(HotSpot) 구분         |
| new 하면 항상 힙인가요        | 아니다. Escape Analysis로 JIT이 스택 할당으로 바꿀 수 있다       |
| 배포 직후 왜 느린가요          | JIT 워밍업. 처음엔 Interpreter, 이후 JIT 컴파일 완료되면 빨라진다   |
| StackOverflowError 원인 | Stack Frame이 쌓여서 Stack 공간 초과. 주로 무한 재귀           |
| OOM 종류                | heap space(객체 과다), Metaspace(클래스 과다), 둘을 구분해야 한다 |

---

## 이어지는 개념 순서

```
JVM 이해 완료
    │
    ├── 1순위: GC 심화
    │         지금까지 Minor GC 구조를 다뤘다
    │         G1GC가 어떻게 다른지, 튜닝은 어떻게 하는지로 이어진다
    │
    ├── 2순위: 스레드와 동시성
    │         Stack이 스레드별 독립이고 Heap이 공유라는 걸 알았다
    │         그럼 Heap 공유에서 생기는 문제(race condition)로 이어진다
    │
    └── 3순위: Spring과의 연결
              Spring Bean은 Heap에 올라간다
              싱글톤 Bean이 공유 Heap에 있다는 게
              왜 상태를 갖으면 안 되는지의 근거다
```

---
## 바이트코드로 바꾸는 것은 javac

JVM이 아니다.

```
javac (Java Compiler)
→ .java 소스코드를 .class 바이트코드로 변환
→ JDK에 포함된 별도 도구
→ 빌드 타임에 실행, JVM과 무관

JVM
→ .class 바이트코드를 받아서 실행
→ 런타임에 동작
```

역할 분리를 명확하게 정리하면:

```
개발자가 작성     javac가 변환        JVM이 실행
.java        →    .class         →   기계어 실행
(소스코드)        (바이트코드)         + 메모리 관리
                                     + GC
```

---

## 혼동하기 쉬운 이유

"Java를 실행한다"는 말에 javac와 JVM이 둘 다 포함되어 있기 때문이다.

```bash
javac MyApp.java   # javac: 소스 → 바이트코드
java MyApp         # JVM:   바이트코드 → 실행
```

빌드 도구(Maven, Gradle)를 쓰면 javac 호출이 숨겨져서 JVM이 다 하는 것처럼 느껴지기도 한다.

---

## 한 줄 정리

```
javac : 소스코드 → 바이트코드  (번역)
JVM   : 바이트코드 → 실행      (실행 + 관리)
```