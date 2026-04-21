#CS/JVM #Status/보충 

## 전체 흐름

```
개발자가 코드 작성
        │
        ▼
MyApp.java  (소스코드)
        │
        │  javac (빌드 타임)
        ▼
MyApp.class (바이트코드)
        │
        │  java 명령어 → JVM 시작 (런타임)
        ▼
┌─────────────────────────────┐
│  ① Class Loader             │
│     .class → 메모리에 올림   │
├─────────────────────────────┤
│  ② Runtime Data Areas       │
│     Heap, Stack 등 초기화    │
├─────────────────────────────┤
│  ③ Execution Engine         │
│     Interpreter로 실행 시작  │
│     핫 코드 → JIT 컴파일     │
├─────────────────────────────┤
│  ④ GC                       │
│     죽은 객체 수거            │
└─────────────────────────────┘
        │
        ▼
   기계어 실행 (CPU)
```

---

## 단계별 상세

### 빌드 타임 — javac

```java
// MyApp.java
public class MyApp {
    public static void main(String[] args) {
        String msg = "hello";
        System.out.println(msg);
    }
}
```

```bash
javac MyApp.java
```

javac가 하는 일:

```
문법 검사      : 잘못된 문법 → 컴파일 에러
타입 검사      : 타입 불일치 → 컴파일 에러
바이트코드 생성 : MyApp.class 파일 생성
```

결과물인 바이트코드를 들여다보면:

```bash
javap -c MyApp.class  # 바이트코드 역어셈블

public static void main(java.lang.String[]);
  Code:
     0: ldc           #7   // String "hello"
     2: astore_1           // 변수 msg에 저장
     3: getstatic     #9   // System.out 가져오기
     6: aload_1            // msg 로드
     7: invokevirtual #15  // println 호출
    10: return
```

이 시점에서 CPU는 아직 아무것도 실행하지 않았다. 플랫폼 중립적인 바이트코드만 만들어진 상태다.

---

### 런타임 시작 — JVM 초기화

```bash
java MyApp
```

이 명령어가 실행되면:

```
1. JVM 프로세스 시작
2. Heap, Stack, Metaspace 등 메모리 영역 초기화
3. Bootstrap ClassLoader 준비
   → java.lang.Object, java.lang.String 등 핵심 클래스 로딩
4. main 스레드 생성
```

---

### Class Loading — 클래스를 메모리에 올리기

```
MyApp.class 로딩 요청
        │
        ▼
Parent Delegation
Bootstrap → Extension → Application 순으로 위임
        │
        ▼
Application ClassLoader가 MyApp.class 발견
        │
        ▼
Loading     : 바이트코드를 메모리에 읽어들임
Linking     : 바이트코드 유효성 검증
              심볼 참조 연결 (System.out이 실제로 뭔지 연결)
Initializing: static 블록 실행, static 변수 초기화
        │
        ▼
Metaspace에 클래스 메타데이터 저장
(클래스 이름, 메서드 목록, 필드 목록 등)
```

---

### 실행 — Interpreter와 JIT

```
main() 호출
        │
        ▼
Stack에 main() Frame 추가
        │
        ▼
Interpreter가 바이트코드 한 줄씩 실행

ldc "hello"       → Heap에 String 객체 생성
astore_1          → Stack Frame의 변수 슬롯에 참조 저장
getstatic         → Metaspace에서 System.out 참조 가져옴
invokevirtual     → println() 호출
                     Stack에 println() Frame 추가
                     실행
                     Frame 제거
return            → main() Frame 제거
```

호출이 쌓이면서 JIT이 개입한다:

```
처음엔 Interpreter로 실행
        │
        │ 호출 횟수 카운팅
        ▼
임계값 초과 (기본 10,000회)
        │
        ▼
C1 컴파일러 → 기본 최적화된 기계어
        │
        │ 더 hot해지면
        ▼
C2 컴파일러 → 공격적 최적화
              인라이닝, Escape Analysis 등
        │
        ▼
Code Cache에 저장
이후 호출은 기계어 직접 실행
```

---

### GC — 메모리 수거

실행 중 객체 생성과 수거가 계속 일어난다:

```
new String("hello")
        │
        ▼
TLAB에서 Eden에 할당 (동기화 없이 빠르게)
        │
        │ Eden이 꽉 참
        ▼
Minor GC 발생 (STW)
살아있는 객체 → Survivor로 복사
죽은 객체    → 수거
        │
        │ age 초과
        ▼
Old Gen으로 승격
        │
        │ Old Gen이 꽉 참
        ▼
Major GC 발생 (STW, 오래 걸림)
```

---

## 빌드 타임 vs 런타임 한 눈에

```
빌드 타임                    런타임
─────────────────────────────────────────────
javac 실행                   java 명령어 실행
.java → .class               JVM 위에서 .class 실행
문법/타입 오류 검출           NullPointerException 등 발생
플랫폼 중립 바이트코드 생성   플랫폼별 기계어로 변환 및 실행
한 번만 수행                 매 실행마다 수행
```

---

## 한 줄 정리

```
javac  : .java → .class        (빌드 타임, 한 번)
JVM    : .class → 기계어 실행  (런타임, 매번)
  ├── Class Loader : 클래스를 메모리에 올림
  ├── Interpreter  : 바이트코드를 즉시 실행
  ├── JIT          : 핫 코드를 기계어로 최적화
  └── GC           : 죽은 객체 수거
```

> Java 코드는 javac가 바이트코드로 번역하고, JVM이 그 바이트코드를 받아서 실행, 메모리 관리, 최적화를 담당한다. 개발자가 `java MyApp`을 치는 순간부터는 전부 JVM의 영역이다.