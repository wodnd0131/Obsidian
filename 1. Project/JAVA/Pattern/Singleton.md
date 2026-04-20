#Java/Pattern/GoF #Java/Anti-Pattern  #Status/완료 

상태 불일치를 예방하기 위해 전역에 객체를 하나만 두자.

# 1. Problem First

```java
// 설정 객체가 여러 개 생기는 상황
public class LottoConfig {
    private int maxNumber;
    private int ticketPrice;

    public LottoConfig() {
        this.maxNumber = 45;
        this.ticketPrice = 1000;
    }
}

LottoConfig config1 = new LottoConfig();
LottoConfig config2 = new LottoConfig();

config1.setTicketPrice(2000); // config2엔 반영 안 됨
```

시스템 전체에서 하나만 존재해야 하는 객체가 여러 개 생기면 **상태 불일치**가 발생한다.

---

# 2. Mechanics

**구현 방식 세 가지:**

**1) Eager Initialization:**

```java
public class LottoConfig {
    // 클래스 로딩 시점에 생성
    private static final LottoConfig INSTANCE = new LottoConfig();

    private LottoConfig() {}

    public static LottoConfig getInstance() {
        return INSTANCE;
    }
}
```

JVM이 클래스를 로딩할 때 `static` 필드를 초기화한다. 멀티스레드에서 안전하다. JLS §12.4.2가 클래스 초기화의 스레드 안전성을 보장하기 때문이다.

단점: 실제로 쓰지 않아도 인스턴스가 생긴다.

**2) Lazy Initialization — 잘못된 버전:**

```java
public class LottoConfig {
    private static LottoConfig instance;

    public static LottoConfig getInstance() {
        if (instance == null) {           // 스레드 A, B 동시 통과 가능
            instance = new LottoConfig(); // 두 개 생성될 수 있음
        }
        return instance;
    }
}
```

멀티스레드 환경에서 인스턴스가 두 개 생길 수 있다.

3) Lazy Initialization — 올바른 버전 (**Holder 패턴**):

```java
public class LottoConfig {
    private LottoConfig() {}

    private static class Holder {
        private static final LottoConfig INSTANCE = new LottoConfig();
    }

    public static LottoConfig getInstance() {
        return Holder.INSTANCE;
    }
}
```

`Holder` 클래스는 `getInstance()`가 처음 호출될 때 로딩된다. 클래스 로딩은 JVM이 스레드 안전성을 보장한다. 실제로 쓸 때까지 인스턴스 생성을 미룰 수 있다.

**4) Enum Singleton — Bloch 최종 권장:**

```java
public enum LottoConfig {
    INSTANCE;

    private final int maxNumber = 45;
    private final int ticketPrice = 1000;

    public int getMaxNumber() { return maxNumber; }
    public int getTicketPrice() { return ticketPrice; }
}

// 사용
LottoConfig.INSTANCE.getMaxNumber();
```

---

# 3. 공식 근거

**Effective Java 3rd, Item 3:**

> "A single-element enum type is often the best way to implement a singleton."

[[../CS/자료구조/Enum Class|Enum]] Singleton이 권장되는 이유 세 가지:

첫째, JVM이 Enum 인스턴스를 단 한 번만 생성함을 보장한다. JLS §8.9:

> "An enum type has no instances other than those defined by its enum constants."

둘째, 직렬화가 자동 처리된다. 일반 Singleton은 직렬화/역직렬화 시 새 인스턴스가 생길 수 있다. `readResolve()` 메서드를 별도로 구현해야 한다. Enum은 이게 언어 레벨에서 보장된다.

셋째, 리플렉션 공격이 차단된다:

```java
// 일반 Singleton — 리플렉션으로 private 생성자 뚫을 수 있다
Constructor<LottoConfig> constructor =
    LottoConfig.class.getDeclaredConstructor();
constructor.setAccessible(true);
LottoConfig instance2 = constructor.newInstance(); // 두 번째 인스턴스 생성됨

// Enum — 리플렉션으로 뚫을 수 없다
// JVM이 Enum 인스턴스 추가 생성을 언어 레벨에서 차단
```

---

# 4. 트레이드오프

| |설명|
|---|---|
|전역 접근|어디서든 같은 인스턴스 접근 가능|
|테스트 어려움|Mock으로 교체 불가|
|전역 상태 오염|상태 있는 Singleton은 테스트 간 오염|
|숨겨진 의존성|Service Locator와 같은 문제 — 시그니처에 안 보임|
|멀티스레드|구현 방식에 따라 동시성 문제|

---

# 5. 안티패턴 검증

**개념 자체의 오남용 — 상태 있는 Singleton:**

```java
public enum LottoGameState {
    INSTANCE;

    private List<Lotto> purchasedTickets = new ArrayList<>(); // 가변 상태

    public void addTicket(Lotto lotto) {
        purchasedTickets.add(lotto);
    }
}
```

전역 가변 상태가 된다. 테스트 간 상태 오염, 멀티스레드 문제 전부 터진다.

**Singleton이 정당한 경우:**

```
상태가 없는 객체          → 정당
상태가 있어도 불변인 객체  → 정당
진짜 시스템 전체에서       → 정당
하나여야 하는 자원
(DB 커넥션 풀, 설정 등)
```

**잘못된 적용 — 테스트가 Singleton에 의존:**

```java
@Test
void 티켓_구매_테스트() {
    LottoGameState.INSTANCE.addTicket(new Lotto(...));
    // 다른 테스트에서 이 상태가 남아있음
    // 테스트 순서에 따라 결과가 달라짐
}
```

테스트가 전역 상태에 의존하면 테스트 독립성이 깨진다.

---

# 6. 커뮤니티 논쟁

Singleton은 디자인 패턴 중 가장 논쟁이 많은 것 중 하나다.

**안티패턴이라는 입장:**

Brian Button, "Why Singletons are Evil" (2004):

- 전역 상태를 도입한다
- 테스트를 어렵게 만든다
- 결합도를 높인다

**정당하다는 입장:**

GoF는 Singleton을 정식 패턴으로 분류했다. 
진짜 하나여야 하는 자원(로거, 설정, 커넥션 풀)에서는 여전히 유효하다.

**실무 결론:**

Spring 같은 DI 프레임워크가 이 논쟁을 우회한다. 빈(Bean)의 기본 스코프가 싱글턴이지만, 컨테이너가 관리하기 때문에 테스트에서 Mock으로 교체 가능하다. Singleton의 이점은 취하면서 단점을 프레임워크가 흡수하는 구조.

```java
// Spring — Singleton이지만 테스트에서 교체 가능
@MockBean
LottoNumberGenerator generator; // 컨테이너가 Mock으로 교체해줌
```

---

# 7. 이어지는 개념

```
Singleton
    ├── 전역 상태 → 테스트 독립성 문제
    ├── 숨겨진 의존성 → Service Locator와 같은 문제
    ├── DI 프레임워크 → Singleton 문제를 프레임워크가 흡수
    └── 불변 객체 → 상태 있는 Singleton의 대안
```


---

## Holder 패턴 추가 설명
내부 클래스의 static은 왜 호출 시에 생성되는가

---

JVM의 **클래스 로딩 규칙** 때문이다.

JVM은 클래스를 **처음 사용하는 시점**에 로딩한다. "처음 사용"의 정의가 JLS §12.4.1에 명시되어 있다:

> "A class or interface is initialized upon first active use."

active use에 해당하는 것들:

- 인스턴스 생성 (`new`)
- 정적 메서드 호출
- 정적 필드 접근

---

Holder 패턴에서 실제로 어떻게 작동하는지:

```java
public class LottoConfig {
    private LottoConfig() {}

    private static class Holder {
        // Holder 클래스가 로딩될 때 이 필드가 초기화된다
        private static final LottoConfig INSTANCE = new LottoConfig();
    }

    public static LottoConfig getInstance() {
        return Holder.INSTANCE; // 여기서 처음으로 Holder에 접근
    }
}
```

```
LottoConfig 클래스 로딩
    → Holder는 아직 로딩 안 됨 (내부에 있어도 별개의 클래스)
    → getInstance() 첫 호출
    → Holder.INSTANCE에 접근 — 이때 Holder 로딩
    → Holder 로딩 시점에 INSTANCE 초기화
```

`LottoConfig`가 로딩될 때 `Holder`는 같이 로딩되지 않는다. 중첩 클래스는 독립적인 클래스 파일(`.class`)로 컴파일된다. `LottoConfig$Holder.class`가 별도로 존재한다. 참조하기 전까지 JVM이 로딩할 이유가 없다.

프레임워크, 직렬화, 리플렉션 — 셋 다 **내가 직접 호출하지 않아도 클래스를 로딩하는 주체**가 존재한다는 공통점이 있다. 이 경우. Lazy 한 방식을 고려해볼 수 있음. ( **별로 차이 없다는,, )**

---

## Enum Singleton은 뭔가

---

네 질문이 정확하다. Enum을 Singleton으로 쓰는 건 **"이 타입의 인스턴스는 딱 하나만 존재한다"는 것을 언어 레벨에서 강제하는 것**이다.

Enum의 본질부터 짚으면:

```java
public enum Direction {
    NORTH, SOUTH, EAST, WEST
}
```

`Direction.NORTH`는 `Direction` 타입의 인스턴스다. JVM이 이 인스턴스들을 클래스 로딩 시점에 딱 한 번만 생성하고, 추가 생성을 언어 레벨에서 막는다.

Singleton Enum은 이 성질을 이용한다:

```java
public enum LottoConfig {
    INSTANCE; // 인스턴스가 하나뿐인 Enum

    private final int maxNumber = 45;
    private final int ticketPrice = 1000;

    public int getMaxNumber() { return maxNumber; }
    public int getTicketPrice() { return ticketPrice; }
}
```

`INSTANCE`가 유일한 인스턴스다. `private` 생성자도, `getInstance()`도 필요 없다. 언어가 강제하니까.

---

## 로직은 Singleton이 관리하지 않는다.

Singleton은 **인스턴스 통제**가 목적이다. 로직을 넣는 순간 두 가지 문제가 생긴다.

**문제 1 — 테스트에서 교체 불가:**

```java
public enum LottoService {
    INSTANCE;

    public Lotto generate() {
        // 랜덤 생성 로직
    }
}

// 테스트에서 고정 번호로 교체하고 싶은데
// Enum Singleton은 Mock으로 교체할 방법이 없다
```

**문제 2 — 책임이 섞인다:**

```java
public enum LottoConfig {
    INSTANCE;

    private final int ticketPrice = 1000;

    // 설정 객체가 생성 로직까지?
    public Lotto generateTicket() { ... } // 책임 과부하
}
```

---

### 그럼 Singleton에 뭘 넣어야 하는가

```
넣어도 되는 것                                               넣으면 안 되는 것
─────────────────────  ──────────────────────
불변 설정값                                       가변 상태
불변 상수                                          비즈니스 로직
                         교체 가능해야 하는 로직
```

실무에서 Singleton이 자연스러운 곳:

```java
// 불변 설정
public enum LottoConfig {
    INSTANCE;
    private final int maxNumber = 45;
    private final int ticketPrice = 1000;
}

// 상태 없는 유틸성 — 근데 이건 그냥 static 상수가 더 낫다
public enum LottoPriceTable {
    INSTANCE;
    public int getPrize(LottoRank rank) { return rank.getPrize(); }
}
```

사실 불변 설정값은 Singleton보다 **상수(static final 필드)나 Enum 자체**로 표현하는 게 더 자연스럽다:

```java
// 이게 더 단순하다
public class LottoConfig {
    public static final int MAX_NUMBER = 45;
    public static final int TICKET_PRICE = 1000;

    private LottoConfig() {}
}
```

Singleton이 진짜 필요한 순간은 **객체가 상태를 가지면서 시스템 전체에서 하나여야 할 때**다.

---
# Spring의 싱글톤?

---

### 1. Problem First

순수 Java Singleton의 문제를 다시 짚으면:

```java
public class LottoService {
    private static final LottoService INSTANCE = new LottoService();
    private final LottoNumberGenerator generator;

    private LottoService() {
        this.generator = new RandomLottoNumberGenerator(); // 내부에서 직접 생성
    }
}

// 테스트에서
@Test
void 테스트() {
    LottoService service = LottoService.getInstance();
    // generator를 FixedGenerator로 교체할 방법이 없다
    // 테스트 불가
}
```

두 가지 문제:

- 의존성이 내부에서 직접 생성된다 → 교체 불가
- JVM 전체에서 하나 → 테스트 간 상태 오염

---

### 2. Spring 배경 지식부터

Spring의 핵심은 [[../../Spring/IoC 컨테이너( ApplicationContext )|IoC 컨테이너( ApplicationContext )]] 다.

IoC가 뭔가. Inversion of Control — 제어의 역전.

원래는 객체가 자신의 의존성을 직접 만든다:

```java
// 제어가 객체 자신에게 있다
public class LottoMachine {
    private final LottoNumberGenerator generator;

    public LottoMachine() {
        this.generator = new RandomLottoNumberGenerator(); // 내가 만든다
    }
}
```

IoC는 이 제어를 컨테이너에게 넘긴다:

```java
// 제어가 컨테이너에게 있다
public class LottoMachine {
    private final LottoNumberGenerator generator;

    public LottoMachine(LottoNumberGenerator generator) { // 컨테이너가 넣어준다
        this.generator = generator;
    }
}
```

`LottoMachine`은 `LottoNumberGenerator`가 어떻게 만들어지는지 모른다. 
컨테이너가 알아서 만들어서 넣어준다.

---

### 3. Spring Bean이 뭔가

Spring 컨테이너가 관리하는 객체를 **Bean**이라고 한다.

```java
@Component // "나를 Bean으로 등록해줘"
public class LottoMachine {
    private final LottoNumberGenerator generator;

    @Autowired // "이 의존성을 컨테이너가 넣어줘"
    public LottoMachine(LottoNumberGenerator generator) {
        this.generator = generator;
    }
}

@Component
public class RandomLottoNumberGenerator implements LottoNumberGenerator {
    public List<LottoNumber> generate() { ... }
}
```

Spring이 시작될 때:

1. `@Component`가 붙은 클래스를 스캔
2. 인스턴스 생성
3. 의존성 주입
4. 컨테이너에 보관

---

### 4. Spring Singleton vs Java Singleton

**Java Singleton:**

```
JVM 전체에서 하나
→ ClassLoader 기준
→ 어떤 방법으로도 두 번째 인스턴스 생성 불가 (Enum 기준)
```

**Spring Singleton:**

```
ApplicationContext(컨테이너) 안에서 하나
→ 컨테이너 기준
→ 컨테이너가 다르면 다른 인스턴스
```

코드로 보면:

```java
// Spring Singleton — 컨테이너가 하나면 인스턴스도 하나
ApplicationContext context1 = new AnnotationConfigApplicationContext(AppConfig.class);
LottoMachine machine1 = context1.getBean(LottoMachine.class);
LottoMachine machine2 = context1.getBean(LottoMachine.class);
machine1 == machine2; // true — 같은 컨테이너, 같은 인스턴스

// 컨테이너가 다르면
ApplicationContext context2 = new AnnotationConfigApplicationContext(AppConfig.class);
LottoMachine machine3 = context2.getBean(LottoMachine.class);
machine1 == machine3; // false — 다른 컨테이너, 다른 인스턴스
```

Java Singleton은 `private` 생성자로 언어 레벨에서 막는다. 
Spring Singleton은 생성자가 열려 있다. **컨테이너가 관리하는 관행**이지 언어가 강제하는 게 아니다.

```java
// Spring Bean인데 직접 new로 만들 수 있다
LottoMachine machine = new LottoMachine(new RandomLottoNumberGenerator());
// 이건 컨테이너 밖의 인스턴스 — Spring이 모른다
```

이게 "엄밀히 싱글톤이 아니다"의 의미다.

---

### 5. 그럼 어떻게 테스트에서 교체가 가능한가

Spring은 테스트용 컨테이너를 따로 띄운다:

```java
@SpringBootTest
class LottoMachineTest {

    @MockBean // 테스트 컨테이너에서 이 Bean을 Mock으로 교체
    LottoNumberGenerator generator;

    @Autowired
    LottoMachine machine; // generator가 Mock으로 주입된 LottoMachine

    @Test
    void 테스트() {
        given(generator.generate())
            .willReturn(List.of(1,2,3,4,5,6)); // 고정값 지정

        Lotto lotto = machine.generate();
        // 결과 예측 가능
    }
}
```

**왜 가능한가:**

```
Java Singleton  → JVM 레벨에서 인스턴스 통제 → 교체 불가
Spring Singleton → 컨테이너 레벨에서 인스턴스 통제 → 컨테이너가 교체해줌
```

테스트 컨테이너에서 `LottoNumberGenerator` Bean을 Mock으로 바꿔치기한다. `LottoMachine`은 자신이 받은 게 Mock인지 모른다. 생성자로 받으니까.

---

### 6. 구조로 정리하면

```
Java Singleton
    생성자 막음 (private)
    JVM 전체에서 하나
    교체 불가 → 테스트 어려움
    의존성이 내부에서 생성됨 → 숨겨진 의존성

Spring Singleton (Bean)
    생성자 열려있음
    컨테이너 안에서 하나
    컨테이너가 교체 가능 → 테스트 용이
    의존성을 컨테이너가 주입 → 의존성 명시적
```

Spring이 Singleton 논쟁을 우회하는 방식:

```
Singleton의 이점  →  인스턴스 하나 → 메모리, 상태 일관성
Singleton의 단점  →  교체 불가, 숨겨진 의존성
                      ↓
                  컨테이너가 흡수
                  - 교체는 컨테이너가 담당
                  - 의존성은 생성자 주입으로 명시
```

---

### 7. 이어지는 개념

```
Spring Singleton
    ├── Bean Scope → Singleton / Prototype / Request / Session
    ├── DI → 생성자 주입 vs 필드 주입 vs setter 주입
    ├── IoC → 제어의 역전이 왜 테스트를 쉽게 만드는가
    └── @MockBean vs @Mock 차이 → 컨테이너 안/밖
```