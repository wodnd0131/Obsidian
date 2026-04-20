#OOP/Rich_Domain #Status/완료

### Anemic Domain Model (빈약한 도메인 모델)

객체가 **데이터만 들고 있고, 행위(로직)는 Service 계층에 있는** 구조.

```java
// Anemic: Lotto는 그냥 데이터 봉투
public class Lotto {
    private List<Integer> numbers;
    public List<Integer> getNumbers() { return numbers; }
    public void setNumbers(List<Integer> numbers) { this.numbers = numbers; }
}

// 로직이 Service에 흩어짐
public class LottoService {
    public int countMatch(Lotto lotto, List<Integer> winningNumbers) {
        return (int) lotto.getNumbers().stream()
            .filter(winningNumbers::contains)
            .count();
    }

    public boolean isValid(Lotto lotto) {
        return lotto.getNumbers().size() == 6
            && lotto.getNumbers().stream().distinct().count() == 6;
    }
}
```

**문제점:**

- `Lotto`가 유효하지 않은 상태로 존재 가능 (size != 6인 Lotto 생성 가능)
- `countMatch` 로직이 `LottoService`, `LottoResultService`, `LottoController` 등에 **중복될 위험**
- `Lotto`를 쓰는 곳마다 getter로 꺼내서 판단 → **Tell, Don't Ask 위반**

### Rich Domain Model (풍부한 도메인 모델)

객체가 **자신의 데이터와 그에 관련된 행위를 함께 가지는** 구조.

```java
public final class Lotto {
    private final List<LottoNumber> numbers;

    public Lotto(List<LottoNumber> numbers) {
        validate(numbers);
        this.numbers = List.copyOf(numbers); // 불변
    }

    private void validate(List<LottoNumber> numbers) {
        if (numbers.size() != 6) {
            throw new IllegalArgumentException("로또 번호는 6개여야 합니다.");
        }
        if (new HashSet<>(numbers).size() != 6) {
            throw new IllegalArgumentException("로또 번호는 중복될 수 없습니다.");
        }
    }

    public int countMatch(Lotto other) {
        return (int) numbers.stream()
            .filter(other.numbers::contains)
            .count();
    }

    public boolean contains(LottoNumber number) {
        return numbers.contains(number);
    }
}

// Service는 조율만
public class LottoResultService {
    public LottoRank getRank(Lotto ticket, WinningLotto winning) {
        int matchCount = winning.countMatch(ticket); // Lotto가 스스로 판단
        boolean bonusMatch = winning.matchBonus(ticket);
        return LottoRank.of(matchCount, bonusMatch);
    }
}
```

**이점:**

- `Lotto`는 생성 시점부터 항상 유효 — **불변식(invariant) 보장**
- 매칭 로직이 `Lotto` 안에 응집 — 중복 제거
- Service가 비대해지지 않음

### 핵심 비교

| |Anemic|Rich|
|---|---|---|
|객체 역할|데이터 컨테이너|행위 + 데이터|
|유효성 검사 위치|Service|생성자/정적 팩토리|
|Tell, Don't Ask|위반|준수|
|테스트 단위|Service 중심|도메인 객체 단위|
|적합한 상황|CRUD 단순 앱|복잡한 비즈니스 규칙|
