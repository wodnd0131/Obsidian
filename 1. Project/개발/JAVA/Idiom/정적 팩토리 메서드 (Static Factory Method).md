#Java/Idiom  #Status/완료 

---

정팩메 만드는 기준은
`특별한` 경우
기본 생성자로 표현이 어렵다고 느껴질 때 사용한다.

*상속 불가 문제는 실은 **장점으로 볼 수도 있다** — [[../OOP/상속|상속]]보다 컴포지션을 유도하기 때문. 
# 1. Problem First

생성자만 있을 때 어떤 고통이 생기는가.

```java
// 생성자만 있는 경우
LottoNumber number1 = new LottoNumber(7);
LottoNumber number2 = new LottoNumber(7);

// 이 둘이 같은가? 생성자 시그니처만 보면 알 수 없다
// 매번 새 객체를 만드는가? 알 수 없다
```

```java
// 시그니처 충돌 — 생성자는 이름을 가질 수 없다
public Lotto(List<Integer> numbers) { ... }
public Lotto(List<Integer> numbers, boolean isSorted) { ... } // 가능
// 근데 이건?
public Lotto(int bonusNumber) { ... }
public Lotto(int manualNumber) { ... } // 컴파일 에러 — 시그니처 동일
```

생성자의 구조적 한계 세 가지:

- 이름을 가질 수 없다 → 의도를 표현하지 못한다
- 매번 새 인스턴스를 반환해야 한다 → 캐싱 불가
- 반환 타입이 해당 클래스로 고정된다 → 하위 타입 반환 불가

---

# 2. Mechanics

정적 팩토리 메서드는 그냥 `static` 메서드다. 언어 차원의 특별한 지원은 없다. 관용구(idiom)다.

```java
public class LottoNumber {
    private static final int MIN = 1;
    private static final int MAX = 45;
    private static final Map<Integer, LottoNumber> CACHE = new HashMap<>();

    static {
        for (int i = MIN; i <= MAX; i++) {
            CACHE.put(i, new LottoNumber(i));
        }
    }

    private final int value;

    private LottoNumber(int value) { // 생성자를 private으로 막는다
        this.value = value;
    }

    public static LottoNumber of(int value) { // 팩토리가 유일한 진입점
        if (value < MIN || value > MAX) {
            throw new IllegalArgumentException("로또 번호는 1~45 사이여야 합니다.");
        }
        return CACHE.get(value); // 캐싱 — 동일 값이면 동일 인스턴스
    }
}
```

`LottoNumber.of(7) == LottoNumber.of(7)` → `true` 생성자로는 절대 달성할 수 없는 보장이다.

---

# 3. 공식 근거

**Effective Java 3rd Edition, Item 1:**

> "Consider static factory methods instead of constructors"

Bloch이 제시한 5가지 장점:

1. 이름을 가질 수 있다
2. 호출마다 새 인스턴스를 생성하지 않아도 된다 (캐싱 가능)
3. 반환 타입의 하위 타입 객체를 반환할 수 있다
4. 입력 매개변수에 따라 다른 클래스의 객체를 반환할 수 있다
5. 반환할 객체의 클래스가 작성 시점에 없어도 된다

단점도 명시함:

- 생성자 없이 팩토리만 있으면 **상속 불가** (private 생성자)
- **Javadoc에서 눈에 안 띈다** — 생성자는 별도 섹션, 팩토리는 일반 메서드에 묻힘

**JDK 표준 라이브러리 실제 사례:**

- `Integer.valueOf(int)` — `-128~127` 캐싱, `new Integer()`는 Java 9부터 deprecated
- `Collections.unmodifiableList()` — 하위 타입 반환
- `EnumSet.of()` — 원소 수에 따라 `RegularEnumSet` 또는 `JumboEnumSet` 반환 (구현체가 public이 아님)

`Integer.valueOf` 캐싱은 JLS §5.1.7에 명세되어 있다:

> "If the value p being boxed is true, false, a byte, or a char in the range \u0000 to \u007f, or an int or short number between -128 and 127, then let r1 and r2 be the results of any two boxing conversions of p. It is always the case that r1 == r2."

---

# 4. 트레이드오프

|            | 생성자      | 정적 팩토리          |
| ---------- | -------- | --------------- |
| 상속         | 가능       | private 생성자면 불가 |
| Javadoc 노출 | 자동 별도 섹션 | 일반 메서드에 묻힘      |
| 캐싱         | 불가       | 가능              |
| 의도 표현      | 불가       | 가능              |
| 하위 타입 반환   | 불가       | 가능              |

상속 불가 문제는 실은 **장점으로 볼 수도 있다** — 상속보다 컴포지션을 유도하기 때문. 
Effective Java Item 18("상속보다 컴포지션")과 맥락이 이어진다.

---

# 5. 실전 함정

**함정 1: 팩토리 만들고 생성자를 안 막는다**

```java
public class LottoNumber {
    public LottoNumber(int value) { ... }      // 열려 있음
    public static LottoNumber of(int value) { ... } // 팩토리도 있음
}

// 캐싱이 의미없어진다
LottoNumber a = LottoNumber.of(7);
LottoNumber b = new LottoNumber(7); // 다른 인스턴스
a == b // false
```

생성자를 `private`으로 막지 않으면 팩토리의 보장이 깨진다.

**함정 2: 이름이 의도를 표현하지 못한다**

```java
// 나쁜 예 — of()가 관용구라고 무조건 of()
LottoNumber.of(7)
LottoTickets.of(manualTickets, autoTickets) // 이게 뭘 하는 건지?

// 좋은 예 — 의도를 이름에 담는다
LottoTickets.combine(manualTickets, autoTickets)
WinningLotto.withBonusNumber(winningNumbers, bonusNumber)
```

**함정 3: 검증 로직을 팩토리가 아닌 생성자에 넣는다**

```java
// 생성자에 검증 — 팩토리를 우회하면 검증도 우회된다
private LottoNumber(int value) {
    if (value < 1 || value > 45) throw new IllegalArgumentException(...);
    this.value = value;
}
```

이건 사실 괜찮다. 생성자가 `private`이면 우회 경로가 없으니까. 검증을 어디에 두느냐는 설계 선택이지 정답이 없다. 단, 팩토리가 여러 개일 때 검증이 분산되면 문제가 생긴다.

---

# 6. 언어화

학생 코드에서 이런 게 보이면:

```java
// 학생 코드
public class Lotto {
    public Lotto(List<Integer> numbers) {
        this.numbers = numbers;
    }
}
```

던질 질문:

> "이 생성자를 호출하는 쪽에서 어떤 `List`를 넘겨야 하는지 어떻게 알 수 있나요? 정렬된 리스트여야 하나요, 중복이 없어야 하나요? 생성자 이름만 보고 알 수 있는 게 뭔가요?"

유도 방향:

```java
// Before
new Lotto(numbers)

// After
Lotto.from(numbers)          // 외부 입력을 받아 변환
Lotto.withNumbers(numbers)   // 명시적 의도
```

---

# 7. 이어지는 개념

```
정적 팩토리
    ├── 캐싱 → Flyweight 패턴
    ├── 하위 타입 반환 → 추상화 / 인터페이스 설계
    ├── private 생성자 → 상속 불가 → 컴포지션 유도 (Item 18)
    └── 인스턴스 통제 → Singleton, 불변 객체 (Item 17)
```

다음으로 파기 좋은 키워드: **원시값 포장** — 팩토리가 왜 VO에서 특히 중요한지로 자연스럽게 이어진다.

---
---
## 정적 팩토리 메서드 vs 정적 팩토리 클래스(생성자 Util 클래스)

---

### 먼저 용어 정리

|                | 위치           | 예시                             |
| -------------- | ------------ | ------------------------------ |
| **정적 팩토리 메서드** | 생성 대상 클래스 내부 | `LottoNumber.of(7)`            |
| **정적 팩토리 클래스** | 별도 유틸 클래스    | `LottoFactory.createNumber(7)` |

GoF _Design Patterns_의 **Factory Method 패턴**은 또 다르다 — 인스턴스 메서드, 상속 기반. Bloch의 Item 1과 GoF의 Factory Method는 이름만 비슷하고 다른 개념이다. 이 혼동이 논쟁의 출발점 중 하나다.

---

### 정적 팩토리 메서드 — 안티패턴인가?

**아니다. 단, 오남용 패턴은 존재한다.**

Bloch Item 1의 단점으로 명시된 것:

> "The main limitation of providing only static factory methods is that classes without public or protected constructors cannot be subclassed."

그리고:

> "A second shortcoming of static factory methods is that they are hard for programmers to find."

이 두 가지가 "안티패턴"으로 오해받는 근거인데, 둘 다 **설계 트레이드오프**지 안티패턴이 아니다.

**오남용 패턴은 이것:**

```java
// 안티패턴 — 팩토리가 검증도 하고, 변환도 하고, 로깅도 하고
public static LottoNumber of(int value) {
    log.info("Creating LottoNumber: {}", value);   // 부수효과
    validate(value);
    convert(value);
    persist(value);   // 생성 외의 책임
    return new LottoNumber(value);
}
```

팩토리 메서드에 생성 외의 책임이 붙기 시작하면 그때 냄새가 난다.

---

### 정적 팩토리 클래스 — 이건 다른 문제다

```java
// LottoFactory.java — 별도 클래스
public class LottoFactory {
    public static Lotto createAuto() { ... }
    public static Lotto createManual(List<Integer> numbers) { ... }
    public static LottoNumber createNumber(int value) { ... }
    public static LottoTickets createTickets(...) { ... }
}
```

이 패턴이 나쁜 이유는 구체적으로 세 가지다.

**1. 응집도 파괴**

`LottoNumber`의 생성 책임이 `LottoNumber` 밖에 있다. 생성 규칙이 바뀌면 `LottoFactory`를 열어야 한다. 객체와 그 객체를 만드는 규칙이 분리된 것.

**2. 팩토리가 모든 타입을 안다 — 의존성 역전**

```
LottoFactory → Lotto
LottoFactory → LottoNumber
LottoFactory → LottoTickets
```

도메인 전체에 의존하는 클래스가 생긴다. 변경 파급 범위가 넓어진다.

**3. 실제로 나쁜 냄새 — God Class로 가는 길**

시간이 지나면서 팩토리에 계속 메서드가 추가된다. 결국 도메인 생성 로직 전체를 한 클래스가 들고 있게 된다.

---

### 그럼 언제 별도 팩토리 클래스가 정당한가

두 가지 경우는 정당하다.

**1. 생성 로직이 진짜 복잡해서 대상 클래스에 넣기 부적절할 때**

```java
// LottoTicketFactory — 구매 금액, 수동/자동 분리 등
// 이 로직이 LottoTickets 클래스 안에 있으면 오히려 이상하다
public class LottoTicketFactory {
    public static LottoTickets create(Money amount, List<Lotto> manualLottos) {
        int autoCount = amount.divide(Lotto.PRICE) - manualLottos.size();
        List<Lotto> autoLottos = generate(autoCount);
        return LottoTickets.of(manualLottos, autoLottos);
    }
}
```

대상 클래스(`LottoTickets`)가 생성 방식을 알 이유가 없을 때.

**2. Abstract Factory — 구현체 교체가 설계 목적일 때**

```java
public interface LottoGeneratorFactory {
    LottoNumberGenerator createGenerator();
}

public class TestLottoGeneratorFactory implements LottoGeneratorFactory {
    public LottoNumberGenerator createGenerator() {
        return new FixedLottoNumberGenerator();
    }
}
```

테스트 더블 주입이 목적인 경우. 이건 GoF Abstract Factory의 정당한 사용.

---

### 커뮤니티 논쟁

Stack Overflow에서 반복적으로 올라오는 질문: 
_"Should factory methods be in a separate class or in the class itself?"_

높은 투표를 받은 답변들의 공통된 결론:

> 생성 로직이 단순하면 
> → 대상 클래스 내부 
> 정적 팩토리 메서드 생성 로직이 복잡하거나 외부 의존성이 필요하면 
> → 별도 팩토리 클래스

공식 근거로는 Bloch Item 1이 "대상 클래스 내부"를 기본으로 설명하고 있고, 별도 팩토리 클래스를 권장하는 공식 문서는 찾지 못했다. 커뮤니티 관행 수준.

---

### 한 줄 정리

- **정적 팩토리 메서드** — 안티패턴 아님. 오남용(생성 외 책임 추가)은 안티패턴.
- **정적 팩토리 클래스** — 안티패턴은 아니지만 기본값이 되면 안 됨. 복잡한 생성 로직이 있을 때만 정당화됨.

구분 기준은 하나다: **생성 책임이 대상 클래스에 자연스럽게 속하는가.**

---


@ 참고
> [[../Pattern/팩토리 패턴 (Factory Pattern)|팩토리 패턴 (Factory Pattern)]]