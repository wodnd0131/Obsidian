C2 컴파일 최적화
escape 되지 않는 변수에 대해 Heap이 아닌 레지스터에서 작업
## 1. Problem First

### EA가 없던 시절의 고통

```java
public long sumPoint(int count) {
    long sum = 0;
    for (int i = 0; i < count; i++) {
        Point p = new Point(i, i);  // 매 iteration마다 힙 할당
        sum += p.x + p.y;
    }
    return sum;
}
```

`count = 10_000_000` 이면:

- **힙 할당 1000만 번** → Eden 영역 즉시 포화
- **Minor GC 수십 번** → STW(Stop-The-World) 반복 발생
- `Point` 객체는 루프 바깥으로 절대 나가지 않음 → **힙에 있을 이유가 없음**

근본 문제: JVM은 "이 객체가 여기서만 쓰인다"는 걸 알지만, 힙 할당을 강제하는 구조였음.

---

## 2. Mechanics

### EA가 하는 일 — 객체의 "탈출 범위" 분류

HotSpot JIT(C2 컴파일러)가 메서드를 컴파일할 때, 각 객체에 대해 **3단계 escape state**를 분석합니다.

```
NoEscape → ArgEscape → GlobalEscape
   ↑ 최적화 가능        탈출 ↑ 최적화 불가
```

|Escape State|의미|최적화 가능 여부|
|---|---|---|
|`NoEscape`|메서드 내부에서만 사용, 외부 참조 없음|✅ Stack Allocation + Scalar Replacement|
|`ArgEscape`|인자로 넘겨지지만 callee가 저장하지 않음|✅ 부분적 (Lock Elision만)|
|`GlobalEscape`|힙 필드에 저장되거나 리턴됨|❌ 일반 힙 할당|

---

### EA가 적용하는 3가지 최적화

#### ① Stack Allocation (스택 할당)

`NoEscape` 객체를 힙 대신 **스택 프레임에 직접 할당**합니다.

```java
// Before JIT
Point p = new Point(i, i);  // 힙 할당

// After EA (개념적 변환 — 실제 바이트코드 아님)
// 스택 프레임에 Point의 필드만 직접 놓음
int p_x = i;
int p_y = i;
```

메서드 종료 시 스택 프레임과 함께 자동 소멸 → **GC 부담 0**

---

#### ② Scalar Replacement (스칼라 치환) ← 실제로 더 자주 쓰임

객체를 **아예 분해해서 개별 지역변수(scalar)로 치환**합니다. HotSpot은 Stack Allocation보다 이걸 더 적극적으로 사용합니다.

```java
// 원본
Point p = new Point(i, i);
sum += p.x + p.y;

// Scalar Replacement 후 (JIT가 내부적으로 변환)
int p_x = i;  // Point 객체 없음, 필드만 존재
int p_y = i;
sum += p_x + p_y;
```

`Point` 객체 자체가 **존재하지 않게 됩니다.** 힙도 스택도 아닌, 레지스터에 직접 올라갈 수 있습니다.

---

#### ③ Lock Elision (락 제거)

`ArgEscape` 이하인 객체의 `synchronized`를 **제거**합니다.

```java
public String concat(String a, String b) {
    StringBuffer sb = new StringBuffer(); // NoEscape
    sb.append(a);
    sb.append(b);
    return sb.toString();
}
```

`StringBuffer`는 `append()`마다 `synchronized`가 걸려있지만, `sb`가 이 메서드 밖으로 나가지 않으면 **다른 스레드가 접근할 수 없으므로** 락 자체를 제거합니다.

---

### 탈출 판단 — 구체적 기준

```java
public class EscapeDemo {

    static User globalUser;  // ← 케이스별 확인

    public void case1() {
        User u = new User("A");
        System.out.println(u.name);
        // ✅ NoEscape: 메서드 내부에서만 사용
    }

    public void case2(Order order) {
        User u = new User("B");
        order.setUser(u);       // ← 핵심
        // ❌ GlobalEscape: order는 외부에서 온 힙 객체
        //    order.user 필드에 저장 → u가 힙으로 탈출
    }

    public User case3() {
        User u = new User("C");
        return u;
        // ❌ GlobalEscape: 호출자에게 반환 → 탈출
    }

    public void case4() {
        User u = new User("D");
        globalUser = u;
        // ❌ GlobalEscape: static 필드에 저장 → 탈출
    }

    public void case5(List<User> list) {
        User u = new User("E");
        list.add(u);
        // ❌ GlobalEscape: list는 외부 힙 객체
    }
}
```

---

### 처음 질문의 `new User("guest")`가 EA 대상이 안 되는 이유

```java
order.setUser(new User("guest"));
```

```
User 생성
    ↓
setUser(user) 호출
    ↓
order.user = user  ← order는 외부 힙 객체의 필드
    ↓
GlobalEscape 판정 → EA 불가
```

`order`가 파라미터나 외부에서 온 힙 객체이기 때문에, `user`를 `order.user`에 저장하는 순간 **힙으로 탈출**합니다.

---

### JIT 컴파일 조건 — EA가 실제로 동작하는 시점

EA는 인터프리터 실행 중엔 작동하지 않습니다.

```
인터프리터 실행
    → 호출 횟수 카운터 증가
    → 임계값 도달 (기본: 10,000회 / CompileThreshold)
    → C2 컴파일러가 JIT 컴파일
    → 이 시점에 EA 수행
```

즉 **충분히 많이 호출된 hot method에서만** EA가 동작합니다. 일회성 코드는 EA 혜택 없음.

---

## 3. 공식 근거

> _"Escape analysis is a technique by which the Java HotSpot Server Compiler can analyze the scope of a new object's uses and decide whether to allocate it on the Java heap."_ — Oracle HotSpot JVM 공식 문서
> 
> (한국어) Escape Analysis는 HotSpot Server 컴파일러가 새 객체의 사용 범위를 분석하여 힙 할당 여부를 결정하는 기술이다.

> _"Scalar replacement: if an object does not escape a compiled method, the fields of the object can be replaced by scalar values."_ — JEP 미포함, HotSpot 내부 구현 문서 (OpenJDK wiki)
> 
> (한국어) 스칼라 치환: 객체가 컴파일된 메서드를 탈출하지 않으면, 객체의 필드를 스칼라 값으로 대체할 수 있다.

JVM Specification §2.5.3은 힙 할당을 규정하지만, EA는 **명세가 아닌 구현 최적화**입니다. 따라서 JLS/JVMS 레벨 근거는 없고, HotSpot 구현 문서와 OpenJDK 소스가 1차 근거입니다.

---

## 4. 설계 트레이드오프

### EA가 포기하는 것

**① 분석 비용** 메서드 전체의 객체 참조 흐름을 추적해야 하므로 **JIT 컴파일 시간이 증가**합니다. C2 컴파일러가 무거운 이유 중 하나입니다.

**② 분석 범위 한계 — inlining 의존**

```java
public void process() {
    Point p = new Point(1, 2);
    helper(p);              // helper가 inline되지 않으면?
}

private void helper(Point p) {
    System.out.println(p.x);  // 실제론 NoEscape지만...
}
```

`helper()`가 **인라인되지 않으면** JIT는 `p`가 `helper` 내부에서 어떻게 쓰이는지 알 수 없습니다. → 보수적으로 `GlobalEscape`로 판정 → EA 포기

EA의 효과는 **인라이닝 성공 여부에 크게 의존**합니다.

**③ 디버깅 난이도** Scalar Replacement가 적용된 객체는 **힙에 존재하지 않습니다.** 따라서 힙 덤프(`jmap`, `jcmd`)에서 보이지 않고, 디버거에서 객체 상태 추적이 어렵습니다.

**④ JDK 버전별 신뢰성 문제** EA/Scalar Replacement는 오랫동안 버그가 있었습니다. JDK 15 이전에는 특정 상황에서 **잘못된 최적화로 데이터 오염**이 발생한 사례가 보고되었습니다. (`-XX:-DoEscapeAnalysis`로 끄고 재현 확인하는 게 표준 디버깅 절차였습니다.)

---

## 5. 이어지는 개념

|순서|개념|이유|
|---|---|---|
|1|**JIT 컴파일 / C1·C2 컴파일러**|EA는 C2 단계에서만 동작 — 선행 구조 이해 필요|
|2|**GC — Eden/Minor GC**|EA의 존재 이유 자체가 Eden 압박 감소 — 효과를 수치로 이해하려면 필요|
|3|**Inlining 최적화**|EA 효과의 실질적 전제조건 — inlining 없이는 EA 범위가 극히 제한됨|
|4|**Value Types (JEP 401 — Valhalla)**|EA가 해결하려는 문제를 언어 레벨에서 근본 해결하는 시도 — EA의 한계를 알아야 Valhalla의 의미가 보임|

---
# Stack Allocation의 단점

## 사실, HotSpot은 Stack Allocation을 거의 안 씁니다

이게 핵심입니다. 단점을 논하기 전에 현실부터 짚어야 합니다.

---

## 왜 Stack Allocation 대신 Scalar Replacement를 쓰는가

### Stack Allocation의 구조적 문제

**① 객체 레이아웃 문제**

```
힙의 객체 레이아웃:
┌─────────────────────┐
│ Mark Word (8 bytes) │  ← GC 상태, 해시코드, 락 정보
│ Klass Pointer(4~8b) │  ← 클래스 메타데이터 포인터
│ field: x (4 bytes)  │
│ field: y (4 bytes)  │
└─────────────────────┘

스택 프레임:
┌─────────────────────┐
│ return address      │
│ local variables     │  ← 여기에 위 구조를 그대로 넣으면?
│ operand stack       │
└─────────────────────┘
```

힙 객체는 **Object Header(Mark Word + Klass Pointer)** 를 반드시 포함합니다. 스택에 올리려면 이 헤더까지 통째로 올라가야 하는데, 스택 프레임 크기는 **컴파일 타임에 고정**됩니다.

> _"The sizes of the local variable array and the operand stack are determined at compile time"_ — JVMS §2.6
> 
> (한국어) 지역 변수 배열과 피연산자 스택의 크기는 컴파일 타임에 결정된다.

객체 헤더를 포함한 가변적 구조를 컴파일 타임에 정확히 맞추는 게 구조적으로 까다롭습니다.

---

**② 참조 일관성 문제 — 가장 결정적**

```java
void process() {
    Point p = new Point(1, 2);  // 스택 할당 가정
    
    log(p);  // 여기서 p의 주소를 넘기면?
}

void log(Object obj) {
    // obj는 스택 주소를 가리키고 있음
    // log()의 스택 프레임이 새로 쌓이면
    // p가 있던 스택 주소는 log()의 프레임에 덮일 수 있음
}
```

```
호출 전:              log() 호출 후:
┌──────────┐          ┌──────────┐
│ log()    │          │ log()    │ ← 새 프레임
│  frame   │          │  frame   │
├──────────┤          ├──────────┤
│process() │          │process() │
│  p: x,y  │◄─ obj    │  p: x,y  │  obj가 가리키는 주소가
│          │          │          │  여전히 유효한가?
└──────────┘          └──────────┘
```

스택은 **프레임이 쌓이고 사라지는 구조**라 주소가 불안정합니다. 힙은 GC가 명시적으로 수거하기 전까지 주소가 보장되지만, 스택 주소는 그렇지 않습니다.

이 문제를 해결하려면 **"이 참조가 스택을 벗어나는가"를 런타임에 계속 추적**해야 합니다. → 그 비용이 최적화 이익을 상쇄합니다.

---

**③ 그래서 Scalar Replacement가 더 나은 이유**

```java
Point p = new Point(i, i);
sum += p.x + p.y;

// Scalar Replacement 후:
int p_x = i;
int p_y = i;
sum += p_x + p_y;
```

객체를 **아예 없애버리면** 위 문제들이 전부 사라집니다.

- Object Header 없음
- 참조 자체가 없으니 주소 일관성 문제 없음
- 필드가 레지스터에 직접 올라갈 수 있음

**Stack Allocation은 "객체를 스택에 놓는 것"이고, Scalar Replacement는 "객체를 없애는 것"입니다.** 후자가 훨씬 근본적인 해결입니다.

---

## 그럼에도 Stack Allocation을 쓴다면 — 실제 단점

Scalar Replacement가 불가능한 경우 (예: 객체 구조가 복잡하거나 배열을 포함하는 경우) Stack Allocation이 폴백으로 고려될 수 있습니다. 이때의 단점:

### ① 스택 크기 제한

```
기본 스택 크기:
- 64bit Linux:  512KB ~ 1MB  (-Xss 옵션)
- 64bit Windows: 256KB ~ 1MB
```

힙은 `-Xmx`로 수십 GB까지 잡을 수 있지만, 스택은 스레드당 고정 크기입니다.

객체를 스택에 올리면 **스택 공간을 잠식**합니다. 재귀 호출이 깊거나, 
스택 할당 객체가 크면 `StackOverflowError` 위험이 높아집니다.

---

### ② 스레드 간 공유 불가 — 설계 제약

```java
// 이런 패턴은 스택 할당 객체로 불가
ExecutorService executor = ...;

void process() {
    Point p = new Point(1, 2);  // 스택 할당 가정
    
    executor.submit(() -> {
        System.out.println(p.x);  // 다른 스레드에서 p 접근
        // → p가 스택에 있으면 이미 사라졌을 수 있음
    });
}
```

스택은 **스레드 전용 공간**입니다. 다른 스레드와 객체를 공유해야 하는 순간, 스택 할당은 불가능합니다. → GlobalEscape 판정으로 이어지는 이유이기도 합니다.

---

### ③ 디버깅/프로파일링 불투명

```
힙 덤프 (jmap -histo):
 num     #instances         #bytes  class name
   1:       1000000       16000000  com.example.Point  ← 보임

스택 할당된 객체:
 → 힙 덤프에 안 보임
 → VisualVM, YourKit에서 추적 불가
 → 메모리 사용량 분석이 왜곡됨
```

**성능이 좋아진 이유를 설명할 수가 없습니다.** 프로파일러가 보지 못하는 영역에서 일어나는 일이라 블랙박스가 됩니다.

---

## 정리

```
Stack Allocation
├── 이론적으로 가능
├── 구조적 문제가 많음
│    ├── Object Header 고정 크기 문제
│    ├── 참조 일관성 보장 비용
│    └── 스택 크기 제한
└── 그래서 HotSpot은 Scalar Replacement를 우선함
         └── 객체를 스택에 올리는 게 아니라 "없애버림"
```

HotSpot에서 `-XX:+DoEscapeAnalysis`를 켜도 실제로는 **Scalar Replacement가 대부분의 일을 합니다.** Stack Allocation은 공식 문서에 언급되지만, 실제 구현에서 얼마나 적극적으로 쓰이는지는 **공식 근거를 찾지 못했고, OpenJDK 커뮤니티에서도 "Scalar Replacement가 주력"이라는 게 일반적 관행 수준의 이해입니다.**

---
# Scalar Replacement 동작 원리

## 질문의 핵심부터 정리

"scope에 따라 지역변수라 판단되면 원시형으로 대체" — 방향은 맞지만 두 가지를 짚어야 합니다.

1. **판단 기준은 scope가 아니라 escape 여부**
2. **대체 대상은 스택이 아니라 레지스터가 우선**

---

## Scalar Replacement가 실제로 하는 것

### "원시형으로 대체"의 정확한 의미

```java
// 원본
Point p = new Point(3, 4);
int result = p.x + p.y;
```

```java
// Scalar Replacement 후 JIT 내부 표현 (개념적)
int p_x = 3;   // Point.x 필드 → 독립 변수
int p_y = 4;   // Point.y 필드 → 독립 변수
int result = p_x + p_y;
```

**`Point` 객체가 사라집니다.** 필드들이 각각 독립적인 값으로 분리됩니다. 이 분리된 값들은 스택 슬롯이 아닌 **CPU 레지스터에 직접 올라가는 게 목표**입니다.

```
힙 할당 경로:
new Point() → Eden 영역 → GC 대상

Scalar Replacement 경로:
p_x = 3 → rax 레지스터
p_y = 4 → rbx 레지스터
result  → rcx 레지스터
```

---

## 복잡한 케이스들 — 실제로 어디까지 되는가

### 케이스 1 — 중첩 객체

```java
class Line {
    Point start;
    Point end;
    
    Line(Point s, Point e) {
        this.start = s;
        this.end = e;
    }
}

void calc() {
    Line line = new Line(new Point(1,2), new Point(3,4));
    int dx = line.end.x - line.start.x;
}
```

```java
// Scalar Replacement 후
int line_start_x = 1;
int line_start_y = 2;
int line_end_x   = 3;
int line_end_y   = 4;
int dx = line_end_x - line_start_x;
```

**재귀적으로 분해합니다.** `Line` → `Point` → `int` 까지 내려갑니다. 단, 이게 가능하려면 `Line`, `Point` 둘 다 `NoEscape`여야 합니다.

---

### 케이스 2 — 반복문 (질문하신 케이스)

```java
long sum = 0;
for (int i = 0; i < 10_000_000; i++) {
    Point p = new Point(i, i);
    sum += p.x + p.y;
}
```

```java
// JIT 내부 변환 (개념적)
long sum = 0;
for (int i = 0; i < 10_000_000; i++) {
    int p_x = i;  // 매 iteration, 레지스터 재사용
    int p_y = i;
    sum += p_x + p_y;
}
```

**매 iteration마다 새 객체가 아니라, 레지스터 값이 갱신됩니다.** 힙 할당 1000만 번 → 힙 할당 0번.

여기서 **scope가 아니라 escape 여부가 기준인 이유**가 드러납니다:

```java
for (int i = 0; i < 10; i++) {
    Point p = new Point(i, i);
    list.add(p);              // list는 외부 힙 객체
    // → GlobalEscape → Scalar Replacement 불가
    // → scope가 루프 내부여도 EA 안 됨
}
```

루프 내부에 있어도 탈출하면 힙 할당입니다.

---

### 케이스 3 — 배열 (Scalar Replacement 한계)

```java
void process() {
    int[] arr = new int[3];  // NoEscape처럼 보이지만
    arr[0] = 1;
    arr[1] = 2;
    arr[2] = 3;
    int sum = arr[0] + arr[1] + arr[2];
}
```

**고정 크기 배열은 이론적으론 가능하지만, HotSpot에서 실제로 잘 안 됩니다.**

이유:

```
배열 접근: arr[i]
    ↓
i가 런타임에 결정되면
    ↓
JIT가 컴파일 타임에 어떤 슬롯인지 특정 불가
    ↓
Scalar Replacement 포기
```

```java
// 이건 됨 (컴파일 타임에 인덱스 확정)
arr[0] + arr[1]

// 이건 안 됨
arr[runtimeIndex]   // 어떤 필드인지 컴파일 타임에 모름
```

---

### 케이스 4 — 조건 분기

```java
void process(boolean flag) {
    Point p = new Point(1, 2);
    
    if (flag) {
        globalList.add(p);   // GlobalEscape 경로
    }
    // flag가 false면 NoEscape인데...
}
```

JIT는 **보수적으로** 판단합니다. `flag`가 런타임 값이라 `GlobalEscape` 경로가 하나라도 있으면 → **전체를 GlobalEscape로 판정.**

단, JIT가 **프로파일링 데이터**로 "이 분기는 항상 false였다"는 걸 알면 → Speculative Optimization으로 NoEscape 처리 후 deopt 가드를 답니다.

```
if (flag) → 항상 false였음 (프로파일링)
    ↓
JIT: "false로 가정하고 Scalar Replacement 적용"
    ↓
런타임에 flag=true 오면
    ↓
Deoptimization → 인터프리터로 폴백 → 힙 할당
```

---

## "대체 과정이 복잡하다" — 실제 복잡도

### JIT가 수행하는 단계

```
1. Inlining
   └── 호출된 메서드를 caller에 펼침
       (이게 없으면 EA 범위가 현재 메서드로 제한)

2. EA (Connection Graph 분석)
   └── 객체마다 노드, 참조마다 엣지
   └── 각 노드의 escape state 계산

3. Scalar Replacement
   └── NoEscape 객체의 필드를 독립 변수로 분리
   └── 객체 생성 코드 제거

4. Register Allocation
   └── 분리된 필드들을 레지스터에 배치

5. Dead Code Elimination
   └── 이제 쓰이지 않는 Object Header 관련 코드 제거
```

**각 단계가 이전 단계의 결과에 의존하고, 어느 단계가 실패해도 전체 최적화가 취소됩니다.**

---

### 실패하는 대표적 케이스 — 인라이닝 실패

```java
void process() {
    Point p = new Point(1, 2);
    helper(p);              // helper가 너무 크면 인라이닝 거부
}

void helper(Point p) {
    // 실제론 p를 외부로 안 넘기지만
    // JIT는 helper 내부를 분석할 수 없음 (인라이닝 안 했으니)
    // → p를 GlobalEscape로 보수적 판정
}
```

인라이닝 임계값 (`-XX:MaxInlineSize=35` bytes, 기본값):

```
helper() 바이트코드 크기 > 35 bytes
    → 인라이닝 거부
    → EA가 helper() 내부를 볼 수 없음
    → p → GlobalEscape
    → Scalar Replacement 포기
```

---

## 정리

```
Scalar Replacement 판단 기준
├── scope (X) ← 직접 기준 아님
└── escape 여부 (O)
         ├── NoEscape → 필드를 독립 변수로 분해
         │              → 레지스터 할당 (스택도 아님)
         └── GlobalEscape → 힙 할당 유지
                  ↑
         루프 내부여도 외부로 나가면 힙 할당
```

**"원시형으로 대체"는 맞는 표현이고, 복잡하다는 직관도 맞습니다.** 그래서 JIT는 인라이닝 실패, 배열 동적 인덱스, 조건 분기 하나에도 보수적으로 포기합니다. 최적화가 적용됐는지 확인하려면 `-XX:+PrintEliminateAllocations` 로그가 유일한 방법입니다.

---

## 실행 파이프라인 전체 흐름

```
소스코드 (.java)
    ↓ javac
바이트코드 (.class)       ← EA 없음, 인라이닝 없음
    ↓
인터프리터 실행           ← EA 없음, 단순 실행
    ↓ (호출 횟수 임계값 도달)
C1 컴파일 (Tier 3)        ← 가벼운 최적화, EA 없음
    ↓ (더 많이 호출됨)
C2 컴파일 (Tier 4)        ← 인라이닝 + EA + Scalar Replacement
```

**EA는 C2 단계에서만 작동합니다.**

---

## Tiered Compilation — 단계별 정확한 기준

JDK 7u4부터 기본값인 Tiered Compilation 기준:

|Tier|실행 주체|임계값|EA 여부|
|---|---|---|---|
|0|인터프리터|-|❌|
|1~2|C1 (프로파일링 없음)|빠른 진입|❌|
|3|C1 (프로파일링 포함)|-|❌|
|4|C2|**10,000회** (CompileThreshold)|✅|

```
// 확인 옵션
-XX:+PrintCompilation          // JIT 컴파일 시점 로그
-XX:CompileThreshold=10000     // C2 진입 기본값
-XX:+PrintEliminateAllocations // Scalar Replacement 적용 여부
```

---

## 중요한 추가 조건 — 호출 횟수만으로 안 됨

**C2에 진입했다고 바로 EA가 적용되는 게 아닙니다.**

```
C2 컴파일 시작
    ↓
inlining 시도
    ├── 성공 → EA 분석 범위 확장됨
    └── 실패 → EA가 현재 메서드 내부만 봄
         ↓
EA 수행
    ├── NoEscape 확정 → Scalar Replacement
    └── 불확실 → 보수적으로 힙 할당 유지
```

**인라이닝이 EA의 전제인 이유:**

```java
void process() {                    // C2 컴파일 대상
    Point p = new Point(1, 2);
    validate(p);                    // 인라이닝 실패시
    use(p.x);
}

void validate(Point p) {            // JIT가 내부를 못 봄
    // p를 저장하는지 안 하는지 모름
}
```

```
인라이닝 실패
    ↓
JIT 입장: validate()가 p를 힙 어딘가에 저장할 수도 있음
    ↓
보수적 판정 → GlobalEscape
    ↓
Scalar Replacement 포기
```

인라이닝이 성공해야 **JIT가 전체 코드 흐름을 한 번에 보고** escape 여부를 확정할 수 있습니다.

---

## 그래서 실무에서 의미하는 것

```java
// 이 루프가 EA 혜택을 받으려면:
for (int i = 0; i < 10_000_000; i++) {
    Point p = new Point(i, i);
    sum += p.x + p.y;
}
```

```
1. 이 메서드가 10,000번 이상 호출되어야 C2 진입
2. C2가 루프 내부를 분석
3. new Point() 생성자가 인라이닝되어야 EA 범위에 포함
4. EA가 p → NoEscape 확정
5. Scalar Replacement 적용
```

**처음 10,000번은 힙 할당으로 실행되다가, C2 컴파일 이후부터 Scalar Replacement로 전환됩니다.**

---

## 정리

```
EA 작동 조건
├── C2 컴파일 진입 (hot method, 10,000회 기본)
│       ← 인터프리터/C1 단계에서는 작동 안 함
└── 인라이닝 성공
        ← 실패하면 EA 분석 범위가 좁아져 보수적 판정
```

---
# 레지스터 vs 힙 — 용량 vs 속도 트레이드오프

## 먼저 실제 크기부터

```
CPU 레지스터 (x86-64 기준)
├── 범용 레지스터: rax, rbx, rcx, rdx, rsi, rdi, r8~r15
│    └── 16개 × 8 bytes = 128 bytes
├── XMM/YMM (SIMD): 16개 × 32 bytes = 512 bytes
└── 합계: ~1KB 미만

L1 캐시:  32KB ~ 64KB   (코어당)
L2 캐시:  256KB ~ 1MB   (코어당)
L3 캐시:  8MB ~ 64MB    (소켓 공유)
힙:       수 GB          (JVM 전체)
```

**레지스터는 극단적으로 작습니다.** 그런데도 JIT가 레지스터를 목표로 하는 이유는 속도 차이가 압도적이기 때문입니다.

---

## 속도 차이 — 접근 레이턴시

```
레지스터:        1 cycle    (~0.3ns)
L1 캐시:         4 cycle    (~1ns)
L2 캐시:        12 cycle    (~4ns)
L3 캐시:        40 cycle    (~12ns)
힙 (DRAM):     200 cycle    (~60ns)
```

**레지스터는 힙보다 200배 빠릅니다.** Scalar Replacement로 `int p_x`가 레지스터에 올라가면, 힙에서 `p.x`를 읽어오는 메모리 접근 자체가 사라집니다.

---

## "용량이 적은데 괜찮은가" — Register Spilling

레지스터가 부족하면 JIT는 자동으로 **spill** 합니다.

```
변수가 레지스터 수보다 많아지면:
    ↓
덜 쓰이는 변수를 스택으로 내림 (spill)
    ↓
다시 필요해지면 스택에서 레지스터로 reload
```

```java
void heavyMethod() {
    // Scalar Replacement로 분해된 필드들
    int a_x, a_y, b_x, b_y, c_x, c_y ...  // 레지스터 16개 초과
    
    // JIT가 내부적으로:
    // rax = a_x  (레지스터)
    // [rsp+8] = b_x  (스택으로 spill)
    // 필요할 때 mov rax, [rsp+8]  (reload)
}
```

**spill이 발생해도 힙보다는 빠릅니다.** 스택은 L1/L2 캐시에 거의 항상 올라와 있기 때문입니다.

```
spill 발생시 접근 경로:
스택 → L1 캐시 히트 확률 매우 높음 → ~1~4 cycle

힙 객체 필드 접근:
객체 주소 → 힙 → 캐시 미스 가능 → 최악 200 cycle
```

---

## 그래서 전체 그림

```
Scalar Replacement 결과물의 저장 위치 우선순위:

1순위: CPU 레지스터        (1 cycle)
    ↓ 레지스터 부족시 spill
2순위: 스택 (L1 캐시 상주) (4 cycle)
    ↓ (Scalar Replacement 포기시)
3순위: 힙                  (200 cycle)
```

**용량이 작은 게 문제가 아니라, 용량이 작기 때문에 빠른 겁니다.** 레지스터는 "자주 쓰는 값만 잠깐 올려두는 곳"이고, JIT의 Register Allocator가 어떤 값을 레지스터에 둘지 자동으로 결정합니다.

---

## 정리

```
레지스터 용량이 작다
    ↓
그래서 CPU와 물리적으로 가장 가까이 있을 수 있음
    ↓
접근 속도 200배 차이

넘치면? → spill → 스택
스택도? → L1 캐시 상주 → 힙보다 훨씬 빠름

결론:
Scalar Replacement는 최악의 경우(spill)에도
힙 할당보다 빠른 접근 경로를 유지합니다.
```