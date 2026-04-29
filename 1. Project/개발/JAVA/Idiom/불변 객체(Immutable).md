#Java/Idiom  #Status/진행
## 1. Problem First

```java
public class Lotto {
    private List<Integer> numbers;

    public Lotto(List<Integer> numbers) {
        this.numbers = numbers; // 외부 참조 그대로 저장
    }

    public List<Integer> getNumbers() {
        return numbers; // 내부 참조 그대로 반환
    }
}

List<Integer> raw = new ArrayList<>(List.of(1, 2, 3, 4, 5, 6));
Lotto lotto = new Lotto(raw);

// 공격 1 — 생성 후 외부에서 원본 변경
raw.add(7);
lotto.getNumbers(); // [1, 2, 3, 4, 5, 6, 7] ← 오염

// 공격 2 — getter로 꺼내서 변경
lotto.getNumbers().clear();
lotto.getNumbers(); // [] ← 파괴
```

`Lotto`는 생성 이후 상태가 바뀌면 안 되는 객체다. 그런데 외부에서 얼마든지 내부를 건드릴 수 있다. 이 문제를 설계 수준에서 차단하는 것이 불변 객체다.

---

## 2. Mechanics

### 불변 객체의 4가지 조건

**1 — 클래스를 `final`로 선언**

```java
public final class Lotto { ... }
```

하위 클래스가 가변 상태를 추가하는 것을 막는다.

```java
// final 없으면 이게 가능
public class MutableLotto extends Lotto {
    public void addNumber(int n) { ... } // 불변성 파괴
}
```

**2 — 모든 필드를 `private final`로**

```java
public final class Lotto {
    private final List<Integer> numbers;
}
```

`final` 필드는 생성자에서 한 번만 할당된다. 재할당 자체가 컴파일 에러.

```java
this.numbers = List.of(1,2,3); // OK
this.numbers = List.of(4,5,6); // 컴파일 에러
```

단, `final`은 **참조의 불변**이지 **객체 내부의 불변이 아니다.**

```java
private final List<Integer> numbers = new ArrayList<>();
numbers = new ArrayList<>(); // 컴파일 에러 — 참조 재할당 불가
numbers.add(7);              // 컴파일 통과 — 객체 내부 변경은 가능
```

이게 핵심 함정이다. `final`만으로는 부족하다.

**3 — 생성자에서 방어적 복사**

```java
public final class Lotto {
    private final List<Integer> numbers;

    public Lotto(List<Integer> numbers) {
        this.numbers = List.copyOf(numbers); // 방어적 복사
    }
}

List<Integer> raw = new ArrayList<>(List.of(1,2,3,4,5,6));
Lotto lotto = new Lotto(raw);
raw.add(7); // lotto 내부에 영향 없음
```

`List.copyOf()`는 복사 + 불변 보장을 동시에 한다. 외부 참조와 내부 참조가 끊긴다.

**4 — getter에서 방어적 복사**

```java
public List<Integer> getNumbers() {
    return Collections.unmodifiableList(numbers);
    // 또는 return List.copyOf(numbers);
}
```

내부 참조를 그대로 반환하면 외부에서 변경 가능하다. 복사본 또는 unmodifiable view를 반환해야 한다.

### `List.copyOf` vs `Collections.unmodifiableList`

```java
List<Integer> original = new ArrayList<>(List.of(1,2,3));

// unmodifiableList — 원본의 view. 원본이 바뀌면 같이 바뀜
List<Integer> view = Collections.unmodifiableList(original);
original.add(4);
view.get(3); // 4 ← 원본 변경이 반영됨

// copyOf — 완전한 복사본. 원본과 독립
List<Integer> copy = List.copyOf(original);
original.add(5);
copy.size(); // 4 ← 영향 없음
```

생성자 인자로 받을 때는 `List.copyOf`가 안전하다. 외부 참조와 완전히 단절되니까.

### 완성된 불변 Lotto

```java
public final class Lotto {
    private final List<Integer> numbers;

    public Lotto(List<Integer> numbers) {
        validate(numbers);
        this.numbers = List.copyOf(numbers); // 방어적 복사
    }

    public List<Integer> getNumbers() {
        return numbers; // List.copyOf 결과는 이미 불변
    }

    private void validate(List<Integer> numbers) {
        if (numbers.size() != 6) {
            throw new IllegalArgumentException("로또 번호는 6개여야 한다");
        }
    }
}
```

---

## 3. 공식 근거

**JLS §17.5 — final Field Semantics**

> "The usage model for final fields is a simple one: Set the final fields for an object in that object's constructor; and do not write a reference to the object being constructed in a place where another thread could see it before the object's constructor is complete." (final 필드는 생성자에서 설정하고, 생성자가 완료되기 전에 다른 스레드가 참조할 수 있는 곳에 객체를 노출하지 말라.)

**Effective Java 3rd Edition, Item 17 — Minimize Mutability**

> "Classes should be immutable unless there's a very good reason to make them mutable." (가변으로 만들 충분한 이유가 없다면 불변으로 만들어라.)

> "Make defensive copies in constructors, and don't provide mutators." (생성자에서 방어적 복사를 하고, 변경자를 제공하지 말라.)

**Effective Java 3rd Edition, Item 50 — Make Defensive Copies When Needed**

> "You must program defensively, with the assumption that clients of your class will do their best to destroy its invariants." (클래스의 클라이언트가 불변식을 파괴하려 최선을 다할 것이라 가정하고 방어적으로 프로그래밍해야 한다.)

---

## 4. 트레이드오프

**복사 비용**

방어적 복사는 객체를 새로 만드는 비용이 따른다. 원소가 많은 컬렉션을 자주 복사하면 GC 압박이 생긴다. 로또 번호 6개 수준에서는 무시할 수 있지만, 대용량 데이터에선 설계를 다시 고민해야 한다.

**유연성 감소**

상태 변경이 필요한 경우 새 객체를 만들어야 한다.

```java
// 가변이었다면
lotto.addNumber(7);

// 불변이라면
Lotto newLotto = new Lotto(append(lotto.getNumbers(), 7));
```

객체 생성이 늘어난다. Java의 `String`이 이 방식이다. `String.concat()`은 새 `String`을 반환한다.

**상속 불가**

`final` 클래스는 확장이 막힌다. 다형성이 필요한 경우 인터페이스로 분리해야 한다.

---

## 5. 안티패턴 검증

### 개념 자체의 오남용

**가변 상태가 핵심인 객체를 억지로 불변으로**

```java
// 로또 구매 진행 상태를 불변으로 강제하면
public final class LottoPurchaseState {
    private final int currentStep;
    
    public LottoPurchaseState nextStep() {
        return new LottoPurchaseState(currentStep + 1); // 매 상태마다 새 객체
    }
}
```

상태 전이가 잦은 객체를 불변으로 만들면 객체 생성이 폭증한다. 도메인 특성을 먼저 보고 결정해야 한다.

### 잘못된 적용 방식

**`final` 필드만 믿고 방어적 복사 생략**

```java
// 안티패턴 — final인데 가변
public final class Lotto {
    private final List<Integer> numbers;

    public Lotto(List<Integer> numbers) {
        this.numbers = numbers; // 복사 없음 — 외부에서 변경 가능
    }
}
```

`final`은 참조의 재할당만 막는다. 객체 내부 상태는 막지 못한다.

**얕은 복사로 착각**

```java
public final class LottoBundle {
    private final List<Lotto> lottos;

    public LottoBundle(List<Lotto> lottos) {
        this.lottos = List.copyOf(lottos); // 리스트는 복사됨
    }
}
```

`List.copyOf`는 리스트 자체는 복사하지만 **원소 객체는 같은 참조**다. `Lotto` 자체가 가변이라면 원소를 통한 변경이 가능하다. 원소까지 불변이어야 진짜 불변이다.

---

## 6. 이어지는 개념

```
불변 객체
    ├── 값 객체(Value Object)
    │       └── 불변이 전제되어야 equals/hashCode가 안전하게 작동
    ├── 스레드 안전성(Thread Safety)
    │       └── 불변 객체는 동기화 없이 공유 가능
    │           가변 객체 공유가 왜 위험한가
    ├── 방어적 복사 vs 불변 컬렉션
    │       └── List.copyOf / List.of / Collections.unmodifiableList
    │           각각의 보장 범위 차이
    └── Record (Java 16+)
            └── 불변 객체 보일러플레이트를 컴파일러가 처리
                final 클래스, private final 필드, 생성자 자동 생성
```

다음으로 팔 곳: **값 객체(Value Object)**. 불변성이 왜 VO의 전제 조건인지, 그리고 `LottoNumber`를 단순 `int` 대신 객체로 감싸는 것이 어떤 의미인지로 자연스럽게 이어진다.