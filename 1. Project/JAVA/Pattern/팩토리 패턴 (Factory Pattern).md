#Java/Pattern/GoF #Status/완료  
# 1. Problem First

```java
public class LottoMachine {
    public Lotto generate() {
        RandomLottoNumberGenerator generator = new RandomLottoNumberGenerator();
        return new Lotto(generator.generate());
    }
}
```

테스트를 짜려고 하면:

```java
@Test
void 로또_생성_테스트() {
    LottoMachine machine = new LottoMachine();
    Lotto lotto = machine.generate(); // 항상 랜덤 — 결과 검증 불가
}
```

`RandomLottoNumberGenerator`가 내부에 박혀 있어서 교체할 수 없다. 테스트도 못 짜고, 나중에 생성 방식이 바뀌면 `LottoMachine`을 열어야 한다.

근본 문제: **생성 책임과 사용 책임이 같은 곳에 있다.**

---

# 2. GoF 팩토리 패턴 — 두 가지가 있다

GoF _Design Patterns_에 팩토리 관련 패턴이 두 개다. 자주 혼동한다.

**Factory Method 패턴:**

생성 책임을 서브클래스로 미룬다.

```java
// 추상 클래스가 "뭘 만들지"는 정의하고, "어떻게 만들지"는 서브클래스에 맡긴다
public abstract class LottoMachine {
    public Lotto generate() {
        LottoNumberGenerator generator = createGenerator(); // 팩토리 메서드
        return new Lotto(generator.generate());
    }

    protected abstract LottoNumberGenerator createGenerator(); // 서브클래스가 구현
}

public class RandomLottoMachine extends LottoMachine {
    @Override
    protected LottoNumberGenerator createGenerator() {
        return new RandomLottoNumberGenerator();
    }
}

public class FixedLottoMachine extends LottoMachine {
    private final List<LottoNumber> fixedNumbers;

    @Override
    protected LottoNumberGenerator createGenerator() {
        return new FixedLottoNumberGenerator(fixedNumbers);
    }
}
```

테스트:

```java
@Test
void 고정번호_로또_생성() {
    LottoMachine machine = new FixedLottoMachine(List.of(1,2,3,4,5,6));
    Lotto lotto = machine.generate(); // 결과 예측 가능
}
```

이 경우 Template Method 패턴과 구조가 동일하다는 점도 인지하자.

**Abstract Factory 패턴:**

연관된 객체 군(family)을 함께 교체한다.

```java
// 생성 책임을 인터페이스로 완전히 분리
public interface LottoFactory {
    LottoNumberGenerator createGenerator();
    LottoTicketPrinter createPrinter();
}

public class RandomLottoFactory implements LottoFactory {
    public LottoNumberGenerator createGenerator() {
        return new RandomLottoNumberGenerator();
    }
    public LottoTicketPrinter createPrinter() {
        return new ConsoleLottoTicketPrinter();
    }
}

public class TestLottoFactory implements LottoFactory {
    public LottoNumberGenerator createGenerator() {
        return new FixedLottoNumberGenerator();
    }
    public LottoTicketPrinter createPrinter() {
        return new NoOpLottoTicketPrinter(); // 테스트에서 출력 불필요
    }
}

// LottoMachine은 팩토리에만 의존
public class LottoMachine {
    private final LottoFactory factory;

    public LottoMachine(LottoFactory factory) { // 주입받음
        this.factory = factory;
    }

    public Lotto generate() {
        return new Lotto(factory.createGenerator().generate());
    }
}
```

---

# 3. 공식 근거

**GoF _Design Patterns_, Factory Method 의도:**

> "Define an interface for creating an object, but let subclasses decide which class to instantiate. Factory Method lets a class defer instantiation to subclasses."

**GoF _Design Patterns_, Abstract Factory 의도:**

> "Provide an interface for creating families of related or dependent objects without specifying their concrete classes."

**둘의 차이 — GoF 명시:**

|       | Factory Method | Abstract Factory |
| ----- | -------------- | ---------------- |
| 수단    | 상속             | 컴포지션             |
| 단위    | 객체 하나          | 객체 군(family)     |
| 교체 방법 | 서브클래스          | 팩토리 구현체 교체       |

---

# 4. 트레이드오프

**Factory Method:**

|        | 설명                  |
| ------ | ------------------- |
| 교체 유연성 | 서브클래스로 생성 방식 교체 가능  |
| 상속 강제  | 생성 방식 바꾸려면 서브클래스 필요 |
| 클래스 폭발 | 생성 방식마다 서브클래스 생김    |

**Abstract Factory:**

|          | 설명                      |
| -------- | ----------------------- |
| 객체 군 일관성 | 연관 객체를 항상 같은 팩토리에서 만듦   |
| 확장 비용    | 새 제품 추가 시 모든 팩토리 구현체 수정 |
| 복잡도      | 인터페이스, 구현체 수 증가         |

## Factory Method가 어울리는 상황

**생성 방식만 바뀌고, 나머지 흐름은 고정일 때.**

```java
public abstract class LottoMachine {
    public Lotto generate() {        // 흐름은 고정
        LottoNumberGenerator gen = createGenerator(); // 이것만 바뀜
        return new Lotto(gen.generate());
    }
    protected abstract LottoNumberGenerator createGenerator();
}
```

`generate()`의 흐름 자체는 건드리지 않고, 내부 부품 하나만 교체하는 구조. 
Template Method 패턴과 목적이 같다.

**어울리는 상황:**

- 알고리즘 흐름은 고정, 일부 단계만 다를 때
- 교체 대상이 단일 객체 하나일 때

---
## Abstract Factory가 어울리는 상황

**연관된 객체 여러 개를 함께 교체해야 할 때.**

```java
// 프로덕션
LottoMachine machine = new LottoMachine(new RandomLottoFactory());

// 테스트
LottoMachine machine = new LottoMachine(new TestLottoFactory());
```

`TestLottoFactory` 하나만 바꾸면 내부 객체 군이 전부 교체된다. 
일관성이 보장된다.

**어울리는 상황:**

- 교체 대상이 여러 객체가 세트로 묶일 때
- 테스트/프로덕션 환경 전환처럼 컨텍스트 단위 교체가 필요할 때
- 상속 계층을 늘리고 싶지 않을 때

---

## 실질적 판단 기준

```
교체 대상이 하나      → Factory Method
교체 대상이 여럿(세트) → Abstract Factory

상속을 써도 되는 구조  → Factory Method
상속 없이 가고 싶다    → Abstract Factory

단순한 구조 선호       → Factory Method
DI 프레임워크 쓴다     → Abstract Factory (Spring이 이 역할을 한다)
```

---

# 5. 안티 패턴 검증

**개념 자체의 오남용 — 단순한 걸 패턴으로 과설계:**

```java
// 생성자 하나면 충분한데
new RandomLottoNumberGenerator()

// 팩토리 인터페이스, 구현체, 주입까지 만드는 것
```

생성 방식이 바뀔 가능성이 없고, 테스트에서 교체할 필요도 없다면 팩토리 패턴은 오버엔지니어링이다.

**잘못된 적용 — Factory Method인데 상속 목적이 뒤바뀐다:**

```java
public class LottoMachine {
    protected LottoNumberGenerator createGenerator() {
        return new RandomLottoNumberGenerator();
    }
    // 나머지 로직 100줄
}

public class TestLottoMachine extends LottoMachine {
    @Override
    protected LottoNumberGenerator createGenerator() {
        return new FixedLottoNumberGenerator();
    }
}
```

테스트만을 위해 프로덕션 클래스를 상속하는 것. 테스트가 프로덕션 설계를 오염시키는 냄새다. 
이 경우엔 Abstract Factory + 의존성 주입이 맞다.

---

# 6. 이어지는 개념

```
팩토리 패턴
    ├── Factory Method  → 상속 기반 → Template Method 패턴과 구조 동일
    ├── Abstract Factory → 컴포지션 기반 → 의존성 주입(DI)으로 자연스럽게 이어짐
    └── 둘 다 → "누가 생성 책임을 갖는가" → IoC (제어의 역전)
```



## 추가로 Abstract Factory — 구체적인 예시

---

### 상황: 테스트 환경 vs 프로덕션 환경

로또 게임에서 필요한 객체들:

```
LottoNumberGenerator  — 번호 생성
LottoTicketPrinter    — 출력
LottoPriceCalculator  — 금액 계산
```

프로덕션에서는 랜덤 생성, 콘솔 출력이 필요하다. 테스트에서는 고정 번호, 출력 없음이 필요하다.

**세 개가 항상 세트로 바뀐다.**

---

### Abstract Factory 없이 짜면

```java
public class LottoMachine {
    public Lotto generate(boolean isTest) {
        LottoNumberGenerator generator;
        LottoTicketPrinter printer;

        if (isTest) {
            generator = new FixedLottoNumberGenerator();
            printer = new NoOpPrinter();
        } else {
            generator = new RandomLottoNumberGenerator();
            printer = new ConsolePrinter();
        }
        // 객체가 늘어날수록 if-else가 길어진다
    }
}
```

새로운 환경(스테이징 등)이 추가될 때마다 `LottoMachine`을 열어야 한다. OCP 위반.

---

### Abstract Factory 적용

```java
// 팩토리 인터페이스 — 연관 객체 군을 묶는다
public interface LottoComponentFactory {
    LottoNumberGenerator createGenerator();
    LottoTicketPrinter createPrinter();
    LottoPriceCalculator createCalculator();
}

// 프로덕션 구현체
public class ProductionLottoFactory implements LottoComponentFactory {
    public LottoNumberGenerator createGenerator() {
        return new RandomLottoNumberGenerator();
    }
    public LottoTicketPrinter createPrinter() {
        return new ConsolePrinter();
    }
    public LottoPriceCalculator createCalculator() {
        return new DefaultPriceCalculator();
    }
}

// 테스트 구현체
public class TestLottoFactory implements LottoComponentFactory {
    private final List<LottoNumber> fixedNumbers;

    public TestLottoFactory(List<LottoNumber> fixedNumbers) {
        this.fixedNumbers = fixedNumbers;
    }

    public LottoNumberGenerator createGenerator() {
        return new FixedLottoNumberGenerator(fixedNumbers);
    }
    public LottoTicketPrinter createPrinter() {
        return new NoOpPrinter(); // 테스트에서 출력 불필요
    }
    public LottoPriceCalculator createCalculator() {
        return new DefaultPriceCalculator();
    }
}
```

```java
// LottoMachine은 팩토리만 안다 — 구현체를 모름
public class LottoMachine {
    private final LottoNumberGenerator generator;
    private final LottoTicketPrinter printer;
    private final LottoPriceCalculator calculator;

    public LottoMachine(LottoComponentFactory factory) {
        this.generator = factory.createGenerator();
        this.printer = factory.createPrinter();
        this.calculator = factory.createCalculator();
    }
}
```

```java
// 프로덕션
LottoMachine machine = new LottoMachine(new ProductionLottoFactory());

// 테스트
LottoMachine machine = new LottoMachine(
    new TestLottoFactory(List.of(1, 2, 3, 4, 5, 6))
);
```

---

### 핵심이 뭔가

`LottoMachine`은 `if (isTest)`가 없다. 환경이 추가되어도 `LottoMachine`을 열지 않는다. 새 팩토리 구현체만 추가하면 된다.

**일관성 보장이 핵심이다.** `TestLottoFactory`를 주입하면 그 안의 객체들이 항상 테스트용 세트로 맞춰진다. 개발자가 실수로 `RandomGenerator`와 `NoOpPrinter`를 섞어서 주입하는 상황이 원천 차단된다.

```java
// Abstract Factory 없으면 이런 실수 가능
LottoMachine machine = new LottoMachine(
    new RandomLottoNumberGenerator(), // 프로덕션용
    new NoOpPrinter(),                // 테스트용 — 섞임
    new DefaultPriceCalculator()
);
```

---

### Factory Method와 비교하면

```java
// Factory Method — 하나만 교체
public class TestLottoMachine extends LottoMachine {
    @Override
    protected LottoNumberGenerator createGenerator() {
        return new FixedLottoNumberGenerator();
        // Printer, Calculator는 그대로 — 세트 교체 불가
    }
}
```

세트로 묶인 객체들을 일관되게 교체하는 게 목적이라면 Factory Method로는 한계가 있다.

---

### 한 줄 정리

> 연관 객체들이 항상 같이 바뀌어야 하고, 그 일관성을 컴파일 타임에 강제하고 싶을 때 Abstract Factory가 정당화된다.