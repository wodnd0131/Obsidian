#Java/Idiom #Status/완료 

## 1. Problem First

```java
// 문제 상황: 로또 번호 중복 검사
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 5);
Set<Integer> unique = new HashSet<>(numbers);
// unique.size() == 5 → 중복 감지됨. 잘 동작한다.

// 그런데 원시값을 포장하면?
public class LottoNumber {
    private final int value;
    public LottoNumber(int value) { this.value = value; }
}

LottoNumber a = new LottoNumber(5);
LottoNumber b = new LottoNumber(5);

Set<LottoNumber> unique = new HashSet<>();
unique.add(a);
unique.add(b);
unique.size(); // 2 ← 중복 감지 실패
```

`equals`/`hashCode`를 재정의하지 않으면 `LottoNumber(5) == LottoNumber(5)`가 **다른 객체**로 취급된다. 중복 검사, `Map` 키, `contains()` 전부 망가진다.

---

## 2. Mechanics

### 동일성(Identity): `==`

```java
LottoNumber a = new LottoNumber(5);
LottoNumber b = new LottoNumber(5);
a == b; // false — 힙의 서로 다른 주소
```

JVM은 `==`를 **참조값(reference) 비교**로 처리한다. `a`와 `b`는 각각 힙에 할당된 별개의 객체이고, `==`는 그 주소가 동일한지만 본다.

### 동등성(Equality): `equals()`

`Object.equals()`의 기본 구현은:

```java
// OpenJDK Object.java
public boolean equals(Object obj) {
    return (this == obj); // 기본값은 동일성과 동일
}
```

재정의하지 않으면 `equals()`도 `==`와 동일하게 동작한다.

### `hashCode`와의 연결 — 이게 핵심

`HashMap`/`HashSet`의 동작을 추적하면:

```
put(key, value) 호출 시:
  1. key.hashCode() → 버킷 인덱스 계산
  2. 해당 버킷에서 key.equals(existing) → 동등성 비교
```

**따라서 두 규칙이 반드시 같이 지켜져야 한다:**

> `a.equals(b) == true` 이면 반드시 `a.hashCode() == b.hashCode()`

역은 성립하지 않아도 됨(해시 충돌은 허용). 하지만 `equals`만 재정의하고 `hashCode`를 그대로 두면:

```java
LottoNumber a = new LottoNumber(5);
LottoNumber b = new LottoNumber(5);

a.equals(b); // true (재정의 후)
a.hashCode() != b.hashCode(); // 기본 구현은 객체 주소 기반

Map<LottoNumber, String> map = new HashMap<>();
map.put(a, "five");
map.get(b); // null ← 버킷을 못 찾음
```

`equals`는 같다고 하는데 `hashCode`가 다르면 `HashMap`이 아예 다른 버킷을 뒤진다.

### String interning — 흔한 착각의 원인

```java
String a = "hello";
String b = "hello";
a == b; // true ← String pool(interning) 때문
```

리터럴 문자열은 JVM이 String pool에 캐싱해서 같은 참조를 반환한다. `==`가 true여도 **우연**이다. `new String("hello") == new String("hello")`는 false.

### Integer 캐싱 — 또 다른 함정

```java
Integer a = 127;
Integer b = 127;
a == b; // true ← JVM이 -128~127 캐싱

Integer c = 128;
Integer d = 128;
c == d; // false
```

JLS §5.1.7에 명시된 autoboxing 캐시 범위. 경계값에서 `==` 결과가 달라진다.

---

## 3. 공식 근거

**JLS §15.21.3 — Reference Equality Operators**

> "The result of `==` is `true` if and only if the operands refer to the same object." (`==`의 결과는 두 피연산자가 동일한 객체를 참조할 때만 true다.)

**JLS §5.1.7 — Boxing Conversion**

> "If the value p being boxed is true, false, a byte, or a char in the range `\u0000` to `\u007f`, or an int or short number between -128 and 127 (inclusive), then let r1 and r2 be the results of any two boxing conversions of p. It is always the case that r1 == r2." (위 범위의 값은 박싱 시 항상 동일한 참조를 반환하도록 보장된다.)

**`Object.equals()` 계약 — Javadoc**

> "Note that it is generally necessary to override the `hashCode` method whenever this method is overridden, so as to maintain the general contract for the `hashCode` method, which states that equal objects must have equal hash codes." (equals를 재정의할 때는 hashCode도 반드시 함께 재정의해야 한다. 동등한 객체는 동일한 해시코드를 가져야 한다는 계약을 유지하기 위해.)

**Effective Java 3rd Edition, Item 11**

> "You must override `hashCode` in every class that overrides `equals`." (equals를 재정의하는 모든 클래스에서 hashCode도 재정의해야 한다.)

**Effective Java 3rd Edition, Item 10**

> "The `equals` method implements an equivalence relation." (equals 메서드는 동치 관계를 구현한다.)
> 
> — 반사성(reflexive), 대칭성(symmetric), 추이성(transitive), 일관성(consistent), null 비교 규칙 5가지를 명시.

---

## 4. 트레이드오프

**`equals`/`hashCode` 재정의 비용**

재정의한 순간 **불변식 유지 책임**이 생긴다. 5가지 계약(반사성·대칭성·추이성·일관성·null 처리) 중 하나라도 깨지면 `HashMap`, `TreeSet`, `Stream.distinct()` 등 컬렉션 전체가 예측 불가능하게 동작한다.

**상속과 equals — 해결 불가능한 긴장**

```java
class Point {
    int x, y;
    // equals: x, y 비교
}
class ColorPoint extends Point {
    Color color;
    // equals에 color까지 포함하면 대칭성 위반
    // color 빼면 ColorPoint의 의미가 없음
}
```

Bloch는 _Effective Java_ Item 10에서 이 문제를 "상속 계층에서 equals의 리스코프 치환 원칙 준수와 동등성 확장은 동시에 달성 불가능하다"고 명시한다. 권고 해법은 **상속 대신 컴포지션**.

**성능**

`hashCode`를 필드 전체로 계산하면 객체가 많을 때 비용이 붙는다. Guava의 `Objects.hashCode()` 또는 `@Override int hashCode() { return Objects.hash(field1, field2); }` 로 간결하게 쓰더라도 내부적으로 `Arrays.hashCode`를 호출한다. 극단적 성능이 필요한 경우 커스텀 해시 전략이 필요하다.

---

## 5. 안티패턴 검증

### 개념 자체의 오남용

**가변 객체에 equals/hashCode 정의하기**

```java
public class MutableLotto {
    private List<Integer> numbers; // mutable
    
    @Override
    public boolean equals(Object o) { ... } // numbers 기반
    @Override
    public int hashCode() { return numbers.hashCode(); }
}

Set<MutableLotto> set = new HashSet<>();
MutableLotto lotto = new MutableLotto(List.of(1,2,3,4,5,6));
set.add(lotto);

lotto.setNumber(0, 45); // 상태 변경
set.contains(lotto); // false ← 버킷이 달라져버림
```

Bloch Item 10:

> "Do not write an `equals` method that depends on unreliable resources." (신뢰할 수 없는 자원에 의존하는 equals를 작성하지 말라.)

`equals`/`hashCode`는 **불변 객체**에서만 안전하게 작동한다. 가변 객체에 정의하면 컬렉션 내부에서 객체가 "사라지는" 버그가 생긴다.

### 잘못된 적용 방식

**equals만 재정의하고 hashCode를 빠뜨리기**

앞서 본 것처럼 `HashMap`/`HashSet`이 망가진다. IDE(IntelliJ, Eclipse) 코드 생성을 쓰면 둘 다 자동 생성되므로 수동 작성 시 특히 주의.

**`instanceof` 대신 `getClass()` 비교**

```java
// 잘못된 패턴
@Override
public boolean equals(Object o) {
    if (o.getClass() != this.getClass()) return false;
    ...
}
```

`getClass()` 비교는 하위 클래스 인스턴스를 항상 다르다고 판단한다. 리스코프 치환 원칙을 깨뜨린다. `instanceof`를 써야 하되, 상속 문제가 있을 경우 앞서 말한 컴포지션으로 해결.

**`==`로 String 비교**

워낙 유명한 안티패턴이지만, String pool interning 때문에 테스트 환경에서는 우연히 통과하고 프로덕션에서 깨지는 경우가 실제로 발생한다.

---

## 6. 이어지는 개념

이 개념은 다음 주제들의 **전제 조건**이다.

```
동등성/동일성
    ├── 원시값 포장(Value Object)
    │       └── VO는 equals/hashCode로 동등성을 정의하는 것이 핵심
    ├── 불변 객체(Immutable Object)
    │       └── 가변 객체에서 equals/hashCode가 왜 위험한지 연결됨
    ├── HashSet/HashMap 내부 구조
    │       └── 버킷, 해시 충돌, load factor
    └── Record (Java 16+)
            └── equals/hashCode/toString을 컴파일러가 자동 생성
                VO 구현의 보일러플레이트를 제거하는 방향
```

다음으로 팔 곳: **불변 객체**. `equals`/`hashCode`를 안전하게 쓰기 위한 조건이 불변성이고, 로또 도메인에서 `Lotto` 클래스 설계의 핵심이기도 하다.