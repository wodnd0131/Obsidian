#CS/Type/Primitive #CS/Type/Wrapper #Status/진행



## primitive가 값 그 자체인 이유

### JVM 타입 시스템의 설계 결정

JVM은 타입을 두 종류로 나눈다.

```
JVM 타입
├── primitive — byte, short, int, long, float, double, char, boolean
└── reference — 클래스, 인터페이스, 배열
```

이건 언어 설계 선택이다. Java 설계자들이 **성능** 때문에 의도적으로 넣었다.

---

### 스택 vs 힙

```java
int a = 5;
Integer b = new Integer(5);
```

```
Stack
┌──────────┐
│ a │  5   │  ← 값 자체가 여기 있음
├──────────┤
│ b │0x4A2F│  ← 주소만 여기 있음
└──────────┘

Heap
┌──────────────┐
│ Integer      │  ← 실제 데이터는 여기
│  value = 5   │
│  (헤더 등)   │
└──────────────┘
```

`int`는 스택에 값이 직접 들어간다. `Integer`는 힙에 객체가 생기고 스택엔 그 주소만 들어간다.

---

### 객체가 힙에 올라가면 생기는 비용

`Integer`가 힙에 올라가는 순간 값 `5` 외에 붙는 것들이 있다.

```
Integer 객체 메모리 레이아웃 (OpenJDK 64bit 기준)
┌─────────────────────┐
│ mark word (8 bytes) │ ← GC 정보, 락 상태, 해시코드
│ klass pointer(4~8b) │ ← 어떤 클래스인지
│ value (4 bytes)     │ ← 실제 int 값
│ padding             │ ← 8byte 정렬
└─────────────────────┘
총 16 bytes
```

`int` 하나는 **4 bytes**. `Integer` 객체는 **16 bytes**. 값 하나 저장하는 데 4배 크고, 힙 할당/GC 비용까지 붙는다.

루프 안에서 수백만 번 쓰이는 숫자가 전부 객체였다면 성능이 심각하게 떨어진다. primitive는 이 비용을 없애기 위한 설계다.

---

### 공식 근거

**JLS §4.2**

> "The primitive types are the boolean type and the numeric types. The numeric types are the integral types and the floating-point types." (primitive 타입은 boolean과 숫자 타입이다.)

**JVM Spec §2.3**

> "The Java Virtual Machine operates on two kinds of types: primitive types and reference types." "Primitive type variables hold primitive values." (JVM은 두 종류의 타입을 다룬다. primitive 타입 변수는 primitive 값을 직접 보유한다.)

즉 스펙 레벨에서 primitive는 "값을 직접 보유"하도록 정의되어 있다. 참조를 거치지 않는 게 언어 설계의 일부다.

---

### 결국 트레이드오프

primitive가 값 그 자체인 건 성능을 위한 선택이고, 그 대가로 **객체가 아니다** — `null` 불가, `equals()` 없음, 제네릭 불가(`List<int>` 안 됨).

그래서 박싱이 존재하는 거고, 박싱하는 순간 힙에 올라가면서 참조 문제가 다시 생기는 것.

# Primitive vs Wrapper

---

## 1. Problem First

```java
// 로또 번호를 Map으로 관리
Map<int, Integer> rankCount = new HashMap<>(); // 컴파일 에러
Map<Integer, Integer> rankCount = new HashMap<>(); // OK

// 당첨 번호가 없을 수 있는 경우
int winningNumber = getWinningNumber(); // null 표현 불가
Integer winningNumber = getWinningNumber(); // null 가능

// 수백만 건 통계 계산
List<Integer> allNumbers = new ArrayList<>(); // 객체 수백만 개 → GC 압박
int[] allNumbers = new int[1000000]; // 값만 배열에 직접
```

언제 뭘 써야 하는지 기준이 없으면 성능 문제, NPE, 컴파일 에러를 상황마다 다르게 만난다.

---

## 2. Mechanics

### Primitive

```
타입      크기        기본값
boolean   1bit(*)    false
byte      8bit       0
short     16bit      0
int       32bit      0
long      64bit      0L
float     32bit      0.0f
double    64bit      0.0d
char      16bit      '\u0000'
```

스택에 값이 직접 저장된다. 힙 할당 없음, GC 대상 아님.

```
Stack
┌──────────┐
│ a │  5   │ ← 값 자체
└──────────┘
```

### Wrapper

각 primitive에 대응하는 객체 타입이다.

```
primitive    Wrapper
boolean   →  Boolean
byte      →  Byte
short     →  Short
int       →  Integer
long      →  Long
float     →  Float
double    →  Double
char      →  Character
```

힙에 객체로 올라간다. 앞서 본 것처럼 `Integer` 하나가 16bytes.

```
Stack          Heap
┌───────────┐  ┌──────────────────┐
│ a │ 0x4A2F│─▶│ mark word (8b)   │
└───────────┘  │ klass ptr (4~8b) │
               │ value = 5  (4b)  │
               │ padding          │
               └──────────────────┘
```

### Autoboxing / Unboxing

컴파일러가 둘 사이를 자동 변환해준다.

```java
// 작성한 코드
Integer a = 5;
int b = a;

// 컴파일러가 변환한 코드
Integer a = Integer.valueOf(5);  // boxing
int b = a.intValue();            // unboxing
```

투명하게 동작하는 것처럼 보이지만 비용이 숨어 있다.

### 숨겨진 비용 — NPE

```java
Integer a = null;
int b = a; // NullPointerException — unboxing 시점에 터짐

// 더 위험한 케이스
Map<String, Integer> scores = new HashMap<>();
int score = scores.get("missing"); // null unboxing → NPE
```

unboxing은 내부적으로 `.intValue()`를 호출한다. null에 메서드 호출이니 NPE.

### 숨겨진 비용 — 성능

```java
// 안티패턴 — Long 대신 long
Long sum = 0L;
for (int i = 0; i < 1_000_000; i++) {
    sum += i; // 매 반복마다 unboxing → 덧셈 → boxing
}
```

루프 100만 번에 boxing/unboxing이 100만 번 발생한다. 힙 할당과 GC 압박이 따라온다.

---

## 3. 공식 근거

**JLS §5.1.7 — Boxing Conversion**

> "If the value p being boxed is an int between -128 and 127 inclusive, let r1 and r2 be the results of any two boxing conversions of p. It is always the case that r1 == r2." (박싱되는 int 값이 -128~127 사이면 동일한 참조를 반환하도록 보장된다.)

**JLS §5.1.8 — Unboxing Conversion**

> "If r is a reference of type Integer, then unboxing conversion converts r into r.intValue()" (Integer 타입 참조 r의 unboxing은 r.intValue()로 변환된다.)

→ null인 r에 `.intValue()` 호출 → NPE 발생 근거.

**Effective Java 3rd Edition, Item 61**

> "Prefer primitive types to boxed primitives." "When your program does mixed-type computations involving boxed and unboxed primitives, it does unboxing, and if a null object reference is unboxed, you get a NullPointerException." (primitive를 Wrapper보다 우선하라. 박싱/언박싱이 섞인 연산에서 null을 언박싱하면 NPE가 발생한다.)

---

## 4. 트레이드오프

||Primitive|Wrapper|
|---|---|---|
|메모리|스택, 값만|힙, 헤더 포함|
|성능|GC 없음|할당/수거 비용|
|null 표현|불가|가능|
|제네릭|불가|가능|
|equals/compareTo|없음(`==`로 값 비교)|있음|
|기본값|0, false 등|null|

---

## 5. 안티패턴 검증

### 개념 자체의 오남용

**Wrapper를 기본값처럼 쓰기**

```java
// 안티패턴
public class LottoResult {
    private Integer matchCount; // 왜 Integer인가?
    private Integer bonusMatch;
}
```

null이 유효한 상태가 아닌데 Wrapper를 쓰면 null 가능성을 열어두는 것이다. 호출하는 쪽에서 항상 null 체크를 강요받는다. 의도가 없는 Wrapper 사용은 그 자체가 설계 노이즈다.

**제네릭 때문에 어쩔 수 없다고 무조건 수용하기**

```java
// List는 어쩔 수 없이 Wrapper
List<Integer> numbers = new ArrayList<>();

// 성능이 중요하다면 대안이 있음
int[] numbers = new int[6];
// 또는 IntStream 활용
IntStream.of(1, 2, 3, 4, 5, 6);
```

### 잘못된 적용 방식

**Wrapper 간 `==` 비교**

앞서 다룬 것. -128~127 캐시 범위에서 우연히 통과하다가 범위 밖에서 터진다.

```java
// 안티패턴
if (a == b) { ... }         // Integer는 equals() 써야 함

// 올바른 방식
if (a.equals(b)) { ... }
if (a.intValue() == b.intValue()) { ... } // unboxing 후 비교
```

**불필요한 autoboxing을 루프 안에 방치**

```java
// 안티패턴
Long sum = 0L;
for (long i = 0; i < 1_000_000; i++) {
    sum += i; // 매 반복 boxing
}

// 올바른 방식
long sum = 0L;
for (long i = 0; i < 1_000_000; i++) {
    sum += i; // 순수 primitive 연산
}
```

---

## 6. 이어지는 개념

```
Primitive vs Wrapper
    ├── Optional<T>
    │       └── null 가능성을 타입으로 명시하는 올바른 방법
    │           Wrapper의 null을 Optional로 대체하는 패턴
    ├── 원시값 포장 (Value Object)
    │       └── int를 LottoNumber로 감싸는 이유
    │           Wrapper와 다르게 도메인 의미와 제약을 부여
    ├── 제네릭
    │       └── 왜 List<int>가 안 되는가
    │           타입 소거와 primitive의 관계
    └── Record (Java 16+)
            └── VO 보일러플레이트 제거
                primitive 필드를 가진 불변 객체를 간결하게
```

다음으로 팔 곳: **원시값 포장**. `int`를 그냥 쓰는 것과 `LottoNumber`로 감싸는 것의 차이 — Wrapper와 전혀 다른 목적이다.