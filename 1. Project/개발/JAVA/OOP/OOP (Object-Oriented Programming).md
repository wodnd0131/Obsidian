#OOP #Status/정리


>  "데이터와 행동을 함께 캡슐화한 객체를 단위로, 
>  책임에 따라 객체 간 협력으로 도메인을 표현하는 패러다임""
## 1. Problem First
- **데이터와 그 데이터를 다루는 행동이 분리**되어 있어서, 로직이 어디에나 복제된다
- **"로또 번호란 무엇인가"** 라는 개념이 코드 어디에도 명시적으로 존재하지 않는다

---

## 2. Mechanics

**JVM 레벨에서 객체란 무엇인가**

객체는 힙(heap)에 할당된 메모리 덩어리야. 두 부분으로 구성돼:

```
[ Object Header ]  — 12~16 bytes
  - Mark Word: GC 정보, 락 상태, 해시코드
  - Klass Pointer: 이 객체가 어느 클래스인지 (메서드 영역 참조)

[ Instance Data ]
  - 필드 값들 (primitive는 값 자체, reference는 주소)
```

**다형성이 JVM에서 작동하는 방식 — vtable**

```java
interface LottoGenerator {
    LottoTicket generate();
}

class RandomLottoGenerator implements LottoGenerator {
    public LottoTicket generate() { ... }
}

class TestLottoGenerator implements LottoGenerator {
    public LottoTicket generate() { ... }
}
```

인터페이스 메서드 호출은 **invokeinterface** 바이트코드로 컴파일돼. 런타임에 실제 구현체의 vtable을 탐색해서 메서드를 찾는다. 이게 다형성의 실제 비용 — invokevirtual보다 느린 이유야.

**캡슐화가 강제되는 지점 — 접근 제어자**

`private`은 컴파일러가 강제하는 게 아니라 JVM이 강제해. 리플렉션으로 뚫을 수 있는 건 JVM이 허용하는 예외 경로가 있기 때문이야.

---

## 3. 공식 근거

**캡슐화 + 추상화**

> _"The principle of information hiding... consists of hiding design decisions in a particular module from all other modules."_ (정보 은닉 원칙은 특정 모듈의 설계 결정을 다른 모든 모듈로부터 숨기는 것으로 구성된다.)
> 
> — David Parnas, _On the Criteria To Be Used in Decomposing Systems into Modules_ (1972)

OOP의 이론적 뿌리. 캡슐화는 OOP가 발명한 게 아니라 모듈화 원칙에서 온 것.

**객체의 협력**

> _"Objects are not just data structures with procedures attached; they are entities that communicate by sending messages."_ (객체는 단순히 프로시저가 붙은 데이터 구조가 아니라, 메시지를 전송하여 소통하는 엔티티다.)
> 
> — Alan Kay, OOP 창시자

Alan Kay가 강조한 건 상속이 아니라 **메시지 패싱**이었어. Java의 메서드 호출이 그 메시지야.

**Effective Java**

> _"Classes and interfaces are the heart of the Java programming language, and one of the primary goals of encapsulation is to separate API from implementation."_ (클래스와 인터페이스는 Java의 핵심이며, 캡슐화의 주요 목표 중 하나는 API와 구현을 분리하는 것이다.)
> 
> — Bloch, _Effective Java_ 3rd ed., Item 15

---


## 5. 안티패턴 검증

**커뮤니티 논쟁**

함수형 진영의 비판:

> "OOP의 공유 가변 상태(shared mutable state)가 동시성 버그의 근원이다"

OOP 진영의 반론:

> "캡슐화로 상태를 격리하면 해결된다"

실제로 Java도 이 논쟁을 수용해서 `record`, `final`, `Collections.unmodifiableList`가 강화됐어.

---

## 6. 이어지는 개념

```
OOP
 ├── SOLID 원칙 — OOP를 잘 하기 위한 설계 원칙
 │    ├── SRP → 책임 귀속의 기준
 │    ├── OCP → 다형성 활용의 기준
 │    └── DIP → 의존성 방향의 기준
 │
 ├── 디자인 패턴 — 객체 관계의 검증된 형태
 │    ├── Strategy → 다형성 활용
 │    ├── Factory → 생성 책임 귀속
 │    └── Decorator → 상속 대신 컴포지션
 │
 ├── 도메인 모델 — OOP로 도메인을 표현하는 방법
 │    └── DDD로 이어짐
 │
 └── 클린코드 — OOP 구조를 읽기 좋게 만드는 실천
```

## OOP 핵심 4가지 개념

### 1. 캡슐화 (Encapsulation)

데이터와 그 데이터를 다루는 로직을 하나로 묶고, 내부를 숨기는 것

```java
// ❌ 캡슐화 X — 외부에서 데이터를 꺼내서 판단
if (lotto.getNumbers().contains(winningNumber)) { ... }

// ✅ 캡슐화 O — 객체 스스로 판단 (Tell, Don't Ask)
if (lotto.contains(winningNumber)) { ... }
```

데이터를 꺼내서 외부에서 판단하는 순간 캡슐화가 깨져요.

---

### 2. 상속 (Inheritance)

부모 클래스의 속성과 행동을 자식이 물려받는 것

```java
public abstract class Shape {
    public abstract double area();
}

public class Circle extends Shape {
    private final double radius;

    @Override
    public double area() {
        return Math.PI * radius * radius;
    }
}
```

다만 상속은 **강한 결합**을 만들어요. Java 진영에서는

> "상속보다 조합(Composition)을 선호하라" — Effective Java

는 원칙이 있을 만큼, 남용하면 안 돼요.

---

### 3. 추상화 (Abstraction)

복잡한 내부를 숨기고 **필요한 인터페이스만 노출**하는 것

```java
// 내부가 어떻게 생성되는지 몰라도 됨
public interface LottoGenerator {
    List<Integer> generate();
}

// 테스트용 구현체
public class FixedLottoGenerator implements LottoGenerator {
    public List<Integer> generate() {
        return List.of(1, 2, 3, 4, 5, 6);
    }
}

// 실제 구현체
public class RandomLottoGenerator implements LottoGenerator {
    public List<Integer> generate() {
        // 랜덤 생성 로직
    }
}
```

TDD에서 `Random` 로직을 인터페이스로 분리하는 게 바로 이 추상화예요.

---

### 4. 다형성 (Polymorphism)

같은 인터페이스로 **다양한 구현체를 동일하게 다루는** 것

```java
// LottoGenerator가 뭔지 몰라도 동일하게 사용 가능
public class LottoMachine {
    private final LottoGenerator generator;

    public LottoMachine(LottoGenerator generator) {
        this.generator = generator;  // 어떤 구현체든 주입 가능
    }
}

// 프로덕션
new LottoMachine(new RandomLottoGenerator());

// 테스트
new LottoMachine(new FixedLottoGenerator());
```

---

### 4가지의 관계

```
추상화 — "무엇을 할지" 인터페이스만 정의
    ↓
다형성 — 같은 인터페이스로 다양한 구현체 교체 가능
    ↓
캡슐화 — 각 구현체 내부는 숨김
    ↓
상속  — 공통 로직 재사용 (단, 조합으로 대체 가능한지 먼저 고민)
```

로또 미션 기준으로 보면 `Random` 생성 로직을 인터페이스로 뽑아낸 것 하나만으로도 추상화 + 다형성 + 테스트 가능한 설계가 동시에 해결돼요.

---

### 객체지향의 사실과 오해 (조영호)

**핵심 메타포: 역할, 책임, 협력**

4가지 개념(캡슐화/상속/추상화/다형성)을 나열하는 방식을 지양하고, OOP를 **객체들의 협력 시스템**으로 바라봐요.

> "객체는 고립된 존재가 아니라 협력하는 존재다"

```
커피 주문 예시
손님 → (주문) → 캐셔 → (제조 요청) → 바리스타

각자 역할이 있고, 메시지를 주고받으며 협력
→ 이게 OOP의 본질
```

이 책이 강조하는 것들은 다음과 같아요.

**메시지가 먼저, 객체가 나중** "어떤 객체가 필요한가"가 아니라 "어떤 메시지가 필요한가"를 먼저 고민하라고 해요. `lotto.getNumbers()`가 아니라 `lotto.contains(number)`처럼 메시지 중심으로 설계하는 게 여기서 나온 사고방식이에요.

**역할과 인터페이스** 역할은 대체 가능해야 한다 → 다형성의 본질을 역할 개념으로 설명해요.

---

### 오브젝트 (조영호)

**핵심 메타포: 코드로 보는 설계 원칙**

사실과 오해의 후속작 성격이에요. 전작이 "왜 OOP인가"를 설명했다면, 오브젝트는 **"어떻게 짜야 하는가"** 를 실제 코드로 보여줘요.

> "변경에 취약한 코드는 의존성이 잘못 설계된 코드다"

이 책이 강조하는 것들은 다음과 같아요.

**의존성 관리**

```java
// ❌ 변경에 취약 — 구체 클래스에 의존
public class LottoMachine {
    private RandomLottoGenerator generator = new RandomLottoGenerator();
}

// ✅ 변경에 유연 — 추상에 의존
public class LottoMachine {
    private final LottoGenerator generator; // 인터페이스
}
```

**책임 주도 설계(RDD)** 사실과 오해에서 나온 역할/책임/협력 개념을 실제 설계 프로세스로 발전시켜요.

**트레이드오프** 정답은 없고, 항상 설계는 트레이드오프라는 걸 반복해서 강조해요.

---
## 책임 주도 설계 (Responsibility-Driven Design)

### 기원

Rebecca Wirfs-Brock가 1990년 제안한 설계 방법론이에요. 오브젝트에서 조영호가 이를 깊이 다루면서 국내에 널리 알려졌어요.

---

### 핵심 사고 전환

**기존 데이터 중심 설계**

> "어떤 데이터가 필요한가?" 를 먼저 고민

```java
// 데이터 중심 — 데이터를 먼저 정의하고 로직을 나중에 붙임
class Lotto {
    private List<Integer> numbers; // 데이터 먼저
}

class LottoService {
    // 로직이 외부로 새어나옴
    public int countMatch(Lotto lotto, List<Integer> winning) {
        return lotto.getNumbers().stream()
            .filter(winning::contains)
            .count();
    }
}
```

**책임 주도 설계**

> "어떤 책임이 필요한가?" 를 먼저 고민

```java
// 책임 중심 — 메시지를 먼저 정의하고 객체를 나중에 결정
class Lotto {
    // "당첨 번호와 몇 개 일치하는지 알아야 해" → Lotto의 책임
    public int countMatch(List<Integer> winningNumbers) {
        return numbers.stream()
            .filter(winningNumbers::contains)
            .count();
    }
}
```

---

### 설계 프로세스

```
1. 시스템이 수행할 책임을 찾는다
        ↓
2. 그 책임을 수행할 객체를 찾는다
        ↓
3. 책임을 수행하기 위해 필요한 협력자를 찾는다
        ↓
4. 협력자에게 메시지를 보낸다
        ↓
5. 반복
```

로또 미션으로 예시를 들면 다음과 같아요.

```
"당첨 결과를 계산해야 한다" (책임 발견)
        ↓
WinningLotto가 적절하다 (객체 결정)
        ↓
비교하려면 Lotto가 필요하다 (협력자 발견)
        ↓
lotto.countMatch(winningNumbers) 메시지 전송
```

---

### CRC 카드

RDD에서 설계 도구로 쓰는 카드예요. 코드 짜기 전에 이걸 먼저 작성해요.

```
┌─────────────────────────────┐
│ 클래스명: Lotto              │
├──────────────┬──────────────┤
│ 책임          │ 협력자        │
│ - 번호 보유   │              │
│ - 일치 개수   │ WinningLotto │
│   계산        │              │
└──────────────┴──────────────┘
```

---

### 데이터 중심 vs 책임 중심 결과 비교

```java
// ❌ 데이터 중심 설계의 결말
class LottoService {
    public Rank getRank(Lotto lotto, WinningLotto winning) { ... }
    public double getProfitRate(List<Lotto> lottos, ...) { ... }
    public int countMatch(Lotto lotto, ...) { ... }
    // 모든 로직이 Service로 몰림 → 절차지향과 다를 바 없음
}

// ✅ 책임 주도 설계의 결말
lotto.countMatch(winningNumbers);       // Lotto가 직접 계산
winningLotto.rank(lotto);               // WinningLotto가 등수 판단
lottoTickets.profitRate(winningLotto);  // LottoTickets가 수익률 계산
```

---
## 데이터 중심 설계의 문제점

### 전형적인 패턴

데이터 중심 설계는 보통 이렇게 시작해요.

```java
// 1단계 — 데이터를 먼저 정의
class Lotto {
    private List<Integer> numbers;
    
    public List<Integer> getNumbers() { return numbers; }
    public void setNumbers(List<Integer> numbers) { this.numbers = numbers; }
}
```

데이터만 있고 로직이 없어요. 그러면 로직은 어디로 갈까요?

---

### 문제 1 — 로직이 외부로 새어나온다

```java
// 모든 로직이 Service로 몰림
class LottoService {

    public int countMatch(Lotto lotto, List<Integer> winningNumbers) {
        return lotto.getNumbers().stream()
            .filter(winningNumbers::contains)
            .count();
    }

    public boolean hasBonusNumber(Lotto lotto, int bonusNumber) {
        return lotto.getNumbers().contains(bonusNumber);
    }

    public Rank getRank(Lotto lotto, List<Integer> winningNumbers, int bonusNumber) {
        int matchCount = countMatch(lotto, winningNumbers);
        boolean hasBonus = hasBonusNumber(lotto, bonusNumber);
        // 등수 판단 로직...
    }

    public double getProfitRate(List<Lotto> lottos, int price,
                                List<Integer> winningNumbers, int bonusNumber) {
        // 수익률 계산 로직...
    }
}
```

`Lotto`는 데이터 보관함이 되고, `LottoService`가 모든 걸 다 해요. 이건 **절차지향 코드**예요. 클래스를 쓴다고 OOP가 아니에요.

---

### 문제 2 — 변경에 극도로 취약하다

```java
// Lotto 내부 구조를 바꾸면?
class Lotto {
    // List<Integer> → Set<Integer> 로 변경
    private Set<Integer> numbers;
}

// 이 코드가 전부 깨짐
lotto.getNumbers().get(0);        // Set은 get() 없음
lotto.getNumbers().stream()...    // 여기저기 흩어진 로직 전부 수정
```

내부 구조가 외부에 노출되어 있어서, **Lotto 하나를 바꾸면 Service 전체를 수정**해야 해요.

---

### 문제 3 — 같은 로직이 여러 곳에 중복된다

```java
class LottoService {
    public Rank getRank(...) {
        int match = lotto.getNumbers().stream()  // 중복
            .filter(winningNumbers::contains).count();
    }
}

class LottoResultView {
    public void print(...) {
        long match = lotto.getNumbers().stream()  // 또 중복
            .filter(winningNumbers::contains).count();
    }
}

class LottoStatistics {
    public void calculate(...) {
        int count = lotto.getNumbers().stream()  // 또또 중복
            .filter(winningNumbers::contains).count();
    }
}
```

`getNumbers()`를 호출하는 순간 로직이 복사되기 시작해요.

---

### 문제 4 — 테스트가 어려워진다

```java
@Test
void 당첨_등수_계산() {
    LottoService service = new LottoService();
    
    // 테스트하려면 Lotto, winningNumbers, bonusNumber
    // 전부 세팅해야 함
    Lotto lotto = new Lotto();
    lotto.setNumbers(List.of(1, 2, 3, 4, 5, 6));
    
    List<Integer> winningNumbers = List.of(1, 2, 3, 4, 5, 6);
    int bonusNumber = 7;
    
    // Service가 너무 많은 걸 알아야 테스트 가능
    Rank rank = service.getRank(lotto, winningNumbers, bonusNumber);
    assertThat(rank).isEqualTo(Rank.FIRST);
}
```

반면 책임 주도 설계라면요.

```java
@Test
void 당첨_등수_계산() {
    Lotto lotto = new Lotto(List.of(1, 2, 3, 4, 5, 6));
    WinningLotto winning = new WinningLotto(List.of(1, 2, 3, 4, 5, 6), 7);
    
    // 객체 스스로 판단
    assertThat(winning.rank(lotto)).isEqualTo(Rank.FIRST);
}
```