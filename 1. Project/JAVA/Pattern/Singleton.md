#Java/Pattern #Status/진행 

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

**3) Lazy Initialization — 올바른 버전 (Holder 패턴):**

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

### 3. 공식 근거

**Effective Java 3rd, Item 3:**

> "A single-element enum type is often the best way to implement a singleton."

Enum Singleton이 권장되는 이유 세 가지:

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

### 4. 트레이드오프

| |설명|
|---|---|
|전역 접근|어디서든 같은 인스턴스 접근 가능|
|테스트 어려움|Mock으로 교체 불가|
|전역 상태 오염|상태 있는 Singleton은 테스트 간 오염|
|숨겨진 의존성|Service Locator와 같은 문제 — 시그니처에 안 보임|
|멀티스레드|구현 방식에 따라 동시성 문제|

---

### 5. 안티패턴 검증

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

### 6. 커뮤니티 논쟁

Singleton은 디자인 패턴 중 가장 논쟁이 많은 것 중 하나다.

**안티패턴이라는 입장:**

Brian Button, "Why Singletons are Evil" (2004):

- 전역 상태를 도입한다
- 테스트를 어렵게 만든다
- 결합도를 높인다

**정당하다는 입장:**

GoF는 Singleton을 정식 패턴으로 분류했다. 진짜 하나여야 하는 자원(로거, 설정, 커넥션 풀)에서는 여전히 유효하다.

**실무 결론:**

Spring 같은 DI 프레임워크가 이 논쟁을 우회한다. 빈(Bean)의 기본 스코프가 싱글턴이지만, 컨테이너가 관리하기 때문에 테스트에서 Mock으로 교체 가능하다. Singleton의 이점은 취하면서 단점을 프레임워크가 흡수하는 구조.

```java
// Spring — Singleton이지만 테스트에서 교체 가능
@MockBean
LottoNumberGenerator generator; // 컨테이너가 Mock으로 교체해줌
```

---

### 7. 이어지는 개념

```
Singleton
    ├── 전역 상태 → 테스트 독립성 문제
    ├── 숨겨진 의존성 → Service Locator와 같은 문제
    ├── DI 프레임워크 → Singleton 문제를 프레임워크가 흡수
    └── 불변 객체 → 상태 있는 Singleton의 대안
```

다음 키워드 던져.