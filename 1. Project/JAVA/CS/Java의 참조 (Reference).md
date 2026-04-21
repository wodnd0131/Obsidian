#CS/JVM #Status/완료 
# 1. Problem First

```java
LottoNumber a = new LottoNumber(5);
LottoNumber b = a;
a = new LottoNumber(10);

System.out.println(b.getValue()); // 5? 10?
```

"b = a로 복사했으니 a가 바뀌면 b도 바뀌는 거 아닌가?" 
— 참조가 무엇인지 모르면 이 질문에 답할 수 없다. 
GC, `==`, 메모리 누수 전부 참조 개념 위에 서 있다.

---

# 2. Mechanics

### 변수는 객체가 아니다

```java
LottoNumber a = new LottoNumber(5);
```

이 한 줄에서 실제로 일어나는 일:

```
Stack                    Heap
┌─────────────┐                     ┌──────────────────┐
│ a      │ 0x4A2F   │────────▶│ LottoNumber      │
└─────────────┘                      │   value = 5      │
                        └──────────────────┘
```

- `new LottoNumber(5)` → 힙에 객체 할당, 주소(`0x4A2F`) 반환
- `a` → 그 주소를 담는 **스택의 변수**

`a`는 객체가 아니다. **객체의 위치를 가리키는 값**이다.

### 대입은 참조값 복사다

```java
LottoNumber a = new LottoNumber(5);
LottoNumber b = a;
```

```
Stack                    Heap
┌─────────────┐         ┌──────────────────┐
│ a │ 0x4A2F  │────────▶│ LottoNumber      │
├─────────────┤    ┌───▶│   value = 5      │
│ b │ 0x4A2F  │───┘     └──────────────────┘
└─────────────┘
```

`b = a`는 "주소값 `0x4A2F`"를 복사한 것이다. 객체가 두 개 생긴 게 아니라, **같은 객체를 두 변수가 가리키는 것**.

```java
a = new LottoNumber(10); // a가 새 객체를 가리킴
```

```
Stack                    Heap
┌─────────────┐         ┌──────────────────┐
│ a │ 0x7C11  │────────▶│ LottoNumber      │
├─────────────┤         │   value = 10     │
│ b │ 0x4A2F  │───┐     └──────────────────┘
└─────────────┘   │     ┌──────────────────┐
                  └────▶│ LottoNumber      │
                        │   value = 5      │
                        └──────────────────┘
```

`b`는 여전히 `0x4A2F`를 가리킨다. `b.getValue()`는 5. **a의 재할당이 b에 영향을 주지 않는다.**

반면 가변 객체라면:

```java
// LottoNumber가 가변 객체일 때
a.setValue(10); // a가 가리키는 객체의 내부 상태 변경
b.getValue();   // 10 ← 같은 객체를 가리키므로 영향받음
```

대입(재할당)과 내부 상태 변경을 구분하는 것이 핵심이다.

### `==`가 비교하는 것

```java
LottoNumber a = new LottoNumber(5);
LottoNumber b = new LottoNumber(5);
LottoNumber c = a;

a == b; // false — 주소값 다름 (0x4A2F vs 0x7C11)
a == c; // true  — 주소값 같음 (0x4A2F == 0x4A2F)
```

`==`는 스택의 변수가 들고 있는 **주소값 자체**를 비교한다. 객체의 내용을 보지 않는다.

### 메서드 호출과 참조 전달

Java는 **항상 값에 의한 전달(pass-by-value)** 이다. 단, 참조 타입의 경우 그 값이 **참조값**이다.

```java
void addNumber(List<Integer> numbers) {
    numbers.add(45); // 원본 리스트 변경됨
    numbers = new ArrayList<>(); // 호출자의 변수엔 영향 없음
}

List<Integer> original = new ArrayList<>(List.of(1,2,3));
addNumber(original);
// original은 여전히 같은 리스트 객체를 가리킴
// 단 내부에 45가 추가된 상태
```

- `numbers.add(45)` → 참조가 가리키는 **객체의 상태 변경** → 원본에 반영
- `numbers = new ArrayList<>()` → **로컬 변수의 참조값 교체** → 원본 변수에 무영향

### null의 의미

```java
LottoNumber a = null;
```

`null`은 "어떤 객체도 가리키지 않는 참조값"이다. 힙에 아무것도 없는 게 아니라, 스택의 변수가 유효한 주소를 갖고 있지 않은 상태.

```
Stack
┌──────────────┐
│ a │  null    │  ──▶  (없음)
└──────────────┘
```

`a.getValue()` → `NullPointerException`: null 참조를 통해 힙 접근 시도.

### GC와 참조의 관계

```java
LottoNumber a = new LottoNumber(5); // 힙에 객체 생성
a = new LottoNumber(10);            // a가 새 객체를 가리킴
// LottoNumber(5)를 가리키는 참조가 0개
// → GC 수거 대상
```

GC는 **어떤 참조도 가리키지 않는 객체**를 수거한다. 참조 개수가 0이 되는 순간 수거 대상이 된다. 
이것이 **도달 가능성(reachability)** 기반 GC의 핵심 원리다.

---

## 3. 공식 근거

**JLS §4.3.1 — Objects**

> "The reference values (often just references) are pointers to these objects, and a special null reference, which refers to no object." (참조값은 객체를 가리키는 포인터이며, null 참조는 어떤 객체도 가리키지 않는다.)

**JLS §15.21.3 — Reference Equality Operators**

> "The result of == is true if and only if the operands refer to the same object or both are null." (두 피연산자가 동일한 객체를 참조하거나 둘 다 null일 때만 == 결과가 true다.)

**JLS §8.4.1 — Formal Parameters**

> "When the method or constructor is invoked, the values of the actual argument expressions initialize newly created parameter variables." (메서드 호출 시 인수의 값이 새로 생성된 파라미터 변수를 초기화한다.)

→ Java는 항상 값을 복사한다. 참조 타입도 참조값(주소)이 복사될 뿐이다. "pass-by-reference"가 아니다.

**JVM Spec §2.2 — Data Types**

> "The reference type is the type of reference values — references to objects." (참조 타입은 참조값, 즉 객체에 대한 참조의 타입이다.)

**Java SE Docs — Garbage Collection**

공식 문서는 GC의 기본 원리를 도달 가능성(reachability)으로 설명한다. 어떤 살아있는 스레드에서도 도달할 수 없는 객체는 수거 대상이다. `java.lang.ref` 패키지의 `WeakReference`, `SoftReference` 등은 이 도달 가능성 등급을 세분화한다.

---

## 4. 트레이드오프

**참조 공유로 인한 의도치 않은 변이(Aliasing)**

```java
public class LottoTickets {
    private List<LottoNumber> tickets;

    public List<LottoNumber> getTickets() {
        return tickets; // 참조 그대로 반환
    }
}

List<LottoNumber> t = lottoTickets.getTickets();
t.clear(); // LottoTickets 내부 상태가 파괴됨
```

두 변수가 같은 객체를 가리키는 **aliasing** 문제. 
방어적 복사(`Collections.unmodifiableList`, `new ArrayList<>(tickets)`)로 막아야 한다. 
비용은 복사 오버헤드.

**GC 압박 — 참조를 오래 들고 있으면**

```java
// 캐시처럼 동작하는 필드
private static LottoResult lastResult; // 오래된 참조를 계속 보유

// GC가 수거 못함 → 메모리 누수
```

정적 필드나 long-lived 객체가 참조를 들고 있으면 GC 수거가 안 된다. 
`WeakReference`로 완화하거나, 명시적으로 null 처리가 필요.

---

## 5. 안티패턴 검증

### 개념 자체의 오남용

**"Java는 pass-by-reference다"라는 오해**

참조 타입을 넘길 때 객체 상태가 변하는 것을 보고 pass-by-reference로 착각한다. 하지만 메서드 내에서 파라미터 변수를 재할당해도 호출자의 변수가 바뀌지 않는다는 점이 이를 반증한다. Java는 **참조값을 복사해서 전달**하는 pass-by-value다.

### 잘못된 적용 방식

**방어적 복사 누락**

```java
// 안티패턴
public Lotto(List<Integer> numbers) {
    this.numbers = numbers; // 외부 참조 그대로 저장
}

List<Integer> raw = new ArrayList<>(List.of(1,2,3,4,5,6));
Lotto lotto = new Lotto(raw);
raw.add(7); // lotto 내부 상태가 오염됨
```

불변 객체를 의도했더라도 참조를 그대로 저장하면 외부에서 내부 상태를 변경할 수 있다.

```java
// 올바른 방식
public Lotto(List<Integer> numbers) {
    this.numbers = List.copyOf(numbers); // 방어적 복사 + 불변
}
```

**null 참조 미처리**

null을 "값 없음"의 표현으로 쓰다가 NPE를 만나는 패턴. `Optional<T>`로 null 가능성을 타입 수준에서 명시하거나, 도메인 객체 생성 시점에 null 검사로 차단하는 것이 낫다.

---

## 6. 이어지는 개념

```
JVM 참조
    ├── 방어적 복사 / 불변 객체
    │       └── 참조 공유 문제를 설계로 차단하는 방법
    ├── GC — 도달 가능성(Reachability)
    │       └── Strong / Soft / Weak / Phantom Reference 등급
    ├── 얕은 복사 vs 깊은 복사
    │       └── 참조 공유가 어느 깊이까지 전파되는가
    └── Optional<T>
            └── null 참조의 타입 안전한 대안
```

다음으로 팔 곳: **방어적 복사와 불변 객체**. 참조 공유가 왜 문제인지 알았으니, 그걸 설계 수준에서 차단하는 방법으로 자연스럽게 이어진다.