#Java/자료구조/Collections #Status/진행

# 내부 함수

## 1. Collections.shuffle()

**특징**

```java
List<Integer> numbers = new ArrayList<>(List.of(1,2,3,4,5,6,7,8,9,10));
Collections.shuffle(numbers);
// [7, 3, 9, 1, 5, ...] 매번 달라짐
```

- 내부적으로 `Random`을 써서 Fisher-Yates 알고리즘으로 섞어
- **원본 리스트를 직접 변경**해 — 새 리스트를 반환하는 게 아님
- `List` 인터페이스면 다 되지만, **`LinkedList`는 느려** (인덱스 접근이 O(n)이라서)

**테스트 관점에서의 문제**

```java
public class LottoMachine {
    public Lotto create() {
        List<Integer> numbers = new ArrayList<>(/* 1~45 */);
        Collections.shuffle(numbers); // 랜덤 — 테스트에서 결과 예측 불가
        return new Lotto(numbers.subList(0, 6));
    }
}
```

`shuffle()`이 내부에 박혀있으면 **항상 다른 결과**가 나와서 단위 테스트를 쓸 수가 없어.

**대체제 — 의존성 주입으로 분리**

```java
// shuffle 전략을 외부에서 주입
public class LottoMachine {
    private final Supplier<List<LottoNumber>> numberGenerator;

    public LottoMachine(Supplier<List<LottoNumber>> numberGenerator) {
        this.numberGenerator = numberGenerator;
    }

    public Lotto create() {
        return new Lotto(numberGenerator.get());
    }
}

// 실제 사용
LottoMachine machine = new LottoMachine(() -> {
    List<LottoNumber> numbers = createAllNumbers(); // 1~45
    Collections.shuffle(numbers);
    return numbers.subList(0, 6);
});

// 테스트 — 고정값 주입 가능
LottoMachine machine = new LottoMachine(() ->
    List.of(1, 2, 3, 4, 5, 6).stream()
        .map(LottoNumber::from)
        .collect(toList())
);
```

---

## 2. Collections.sort()

**특징**

```java
List<LottoNumber> numbers = new ArrayList<>(...);
Collections.sort(numbers); // LottoNumber가 Comparable 구현해야 함
```

- 마찬가지로 **원본을 직접 변경**
- Tim Sort 알고리즘 — O(n log n), 안정 정렬
- 정렬 기준은 두 가지 방법으로 줄 수 있어

```java
// 방법 1 — Comparable 구현 (LottoNumber 자체에 기준 내장)
public class LottoNumber implements Comparable<LottoNumber> {
    @Override
    public int compareTo(LottoNumber other) {
        return Integer.compare(this.value, other.value);
    }
}

// 방법 2 — Comparator 외부 주입 (기준을 바꿀 수 있음)
Collections.sort(numbers, Comparator.comparingInt(LottoNumber::getValue));
```

**대체제**

```java
// List.sort() — 같은 동작, 더 현대적인 방식
numbers.sort(Comparator.comparingInt(LottoNumber::getValue));

// stream sorted() — 원본 안 건드리고 새 리스트 반환
List<LottoNumber> sorted = numbers.stream()
        .sorted(Comparator.comparingInt(LottoNumber::getValue))
        .collect(Collectors.toList());
```

**어느 걸 써야 하나**

|상황|선택|
|---|---|
|불변 객체 유지하고 싶을 때|`stream().sorted()`|
|원본을 바로 바꿔도 될 때|`List.sort()`|
|Java 7 이하 환경|`Collections.sort()`|

로또 미션에서 `Lotto`를 불변으로 만들었다면 **생성 시점에 정렬**하고 끝내는 게 깔끔해.

```java
public class Lotto {
    private final List<LottoNumber> numbers;

    public Lotto(List<LottoNumber> numbers) {
        validate(numbers);
        this.numbers = numbers.stream()
                .sorted()
                .collect(Collectors.toUnmodifiableList()); // 정렬 + 불변 한번에
    }
}
```

---

## 3. ArrayList.contains()

**특징**

```java
List<LottoNumber> winningNumbers = new ArrayList<>(List.of(...));
winningNumbers.contains(new LottoNumber(7)); // equals()로 비교
```

- 내부적으로 `equals()`를 써서 비교 — **4번(값 객체)이 여기서 직결돼**
- `ArrayList`는 처음부터 끝까지 순차 탐색 — **O(n)**
- 로또는 6개라 사실 성능 차이는 없지만, 개념은 알아야 해

**대체제 — HashSet**

```java
// ArrayList contains — O(n)
List<LottoNumber> list = new ArrayList<>(winningNumbers);
list.contains(number); // 6번 순회

// HashSet contains — O(1)
Set<LottoNumber> set = new HashSet<>(winningNumbers);
set.contains(number); // hashCode로 바로 찾음
```

`HashSet`을 쓰려면 **`hashCode`가 반드시 재정의**되어 있어야 해. 여기서 4번(값 객체)의 `equals`/`hashCode` 얘기가 다시 나와.

```java
// Lotto 안에서 쓴다면
public class Lotto {
    private final List<LottoNumber> numbers;

    public int countMatch(List<LottoNumber> winningNumbers) {
        Set<LottoNumber> winningSet = new HashSet<>(winningNumbers); // 변환 한 번
        return (int) numbers.stream()
                .filter(winningSet::contains) // O(1)씩
                .count();
    }
}
```

---

## 세 개를 연결해서 보면

```
shuffle()  →  랜덤 생성 (테스트하려면 외부 주입으로 분리)
   ↓
sort()     →  생성 시점에 정렬해서 불변으로 고정
   ↓
contains() →  equals/hashCode 없으면 당첨 비교가 안 됨
```

힌트 세 개가 사실 **P1 개념들(원시값 포장, 불변, 값 객체)과 전부 연결돼 있어.** 힌트를 그냥 따라 쓰면 동작은 하는데, 개념을 알고 쓰면 설계까지 달라지는 거야.