#Java/Pattern/GoF #Status/완료 

캐싱된 상태를 관리하는 객체와
캐싱된 객체를 사용하는 객체를 분리하라
# 1. Problem First

`LottoNumber`는 1~45 사이의 값만 존재한다. 근데 이런 코드가 있으면:

```java
// 티켓 1000장 구매
for (int i = 0; i < 1000; i++) {
    new LottoNumber(7); // 매번 새 객체
}
```

힙에 `LottoNumber(7)` 인스턴스가 1000개 생긴다. 값은 모두 동일한데. 
로또 번호는 **상태가 변하지 않고**, **가짓수가 45개로 고정**되어 있다. 이런 객체를 매번 새로 만드는 건 낭비다.

---

# 2. Mechanics — 캐싱 구현

```java
public class LottoNumber {
    private static final Map<Integer, LottoNumber> CACHE = new HashMap<>();

    static {
        for (int i = 1; i <= 45; i++) {
            CACHE.put(i, new LottoNumber(i));
        }
    }

    private final int value;

    private LottoNumber(int value) {
        this.value = value;
    }

    public static LottoNumber of(int value) {
        if (value < 1 || value > 45) {
            throw new IllegalArgumentException();
        }
        return CACHE.get(value); // 항상 같은 인스턴스 반환
    }
}
```

`LottoNumber.of(7) == LottoNumber.of(7)` → `true`

이게 가능한 이유: 팩토리가 **인스턴스 생성을 통제**하기 때문. 생성자가 열려 있으면 이 보장은 즉시 깨진다.

---

# 3. Flyweight 패턴으로 이어지는 지점

캐싱을 구조화한 게 Flyweight다. GoF _Design Patterns_에서 정의한 의도:

> "Use sharing to support large numbers of fine-grained objects efficiently."

Flyweight가 캐싱과 다른 점은 **공유 가능한 상태(intrinsic)** 와
**공유 불가한 상태(extrinsic)** 를 명시적으로 분리한다는 것이다.

```
Intrinsic state  — 객체 내부에 저장, 모든 컨텍스트에서 동일
Extrinsic state  — 클라이언트가 전달, 컨텍스트마다 다름
```

로또 예시로 보면:

```java
// LottoNumber의 value(1~45) → intrinsic, 캐싱 대상
// 이 번호가 몇 번째 티켓의 몇 번째 자리인지 → extrinsic, 외부에서 관리

public class LottoTicket {
    private final List<LottoNumber> numbers; // 캐싱된 인스턴스를 참조

    // LottoNumber 객체는 공유됨
    // 티켓 내 위치(extrinsic)는 List의 index로 외부에서 관리
}
```

---

# 4. 공식 근거

**JDK 표준 구현 — `Integer.valueOf()`**

```java
// OpenJDK 소스 (java.lang.Integer)
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

`-128 ~ 127` 범위는 캐싱. 이 범위 명세는 JLS §5.1.7에 있다.

```java
Integer a = Integer.valueOf(127);
Integer b = Integer.valueOf(127);
a == b; // true — 같은 인스턴스

Integer c = Integer.valueOf(128);
Integer d = Integer.valueOf(128);
c == d; // false — 다른 인스턴스
```

이게 1학년이 `==` vs `equals()`에서 자주 틀리는 이유다. 캐싱 범위를 모르면 127은 `==`이 되고 128은 안 된다는 게 마법처럼 보인다.

**Effective Java 3rd, Item 1:**

> "The ability of static factory methods to return the same object from repeated invocations allows classes to maintain strict control over what instances exist at any time."

**GoF _Design Patterns_, Flyweight 챕터:**

> "A flyweight is a shared object that can be used in multiple contexts simultaneously."

---

# 5. 트레이드오프

|        | 설명                       |
| ------ | ------------------------ |
| 메모리 절약 | 동일 값 객체 재사용              |
| 동일성 보장 | `==` 비교 가능해짐             |
| 초기화 비용 | `static` 블록에서 선불로 45개 생성 |
| 설계 제약  | 반드시 불변이어야 함. 가변이면 공유 불가  |

마지막 제약이 핵심이다. **캐싱은 불변 객체에서만 안전하다.**

```java
// 만약 LottoNumber가 가변이라면
LottoNumber a = LottoNumber.of(7);
LottoNumber b = LottoNumber.of(7);
a.setValue(10); // b도 10이 된다 — 같은 인스턴스니까
```

캐싱 → Flyweight로 가는 길목에 **불변 객체**가 필수 조건으로 놓여 있다.

---
# 7. 이어지는 개념

```
캐싱 / Flyweight
    ├── 불변 객체가 전제조건 → Item 17
    ├── equals/hashCode 계약 → 동등성 vs 동일성
    └── Integer 캐싱 함정 → == vs equals 오해의 근원
```

구체적인 예시로 가자.

---

### 로또 번호 하나를 생각해보자

`LottoNumber(7)` 이 객체에 대해 두 가지 질문을 던진다.

**질문 1: "7이라는 값" — 이게 어느 티켓에 있든 바뀌는가?**

안 바뀐다. 1번 티켓에 있든 100번 티켓에 있든 7은 7이다.

**질문 2: "이 7이 몇 번 티켓의 몇 번째 자리에 있는가" — 이게 컨텍스트마다 다른가?**

다르다. 1번 티켓 3번째 자리일 수도 있고, 50번 티켓 1번째 자리일 수도 있다.

---

여기서 갈린다.

```
intrinsic  =  "7이라는 값"          → LottoNumber 객체 안에 저장
extrinsic  =  "몇 번 티켓 몇 번째"  → LottoTicket이 관리
```

---

### 코드로 보면

```java
// LottoNumber — intrinsic state만 가진다
public class LottoNumber {
    private final int value; // 이것만 저장. 항상 동일하니까 공유 가능

    private static final Map<Integer, LottoNumber> CACHE = new HashMap<>();

    static {
        for (int i = 1; i <= 45; i++) {
            CACHE.put(i, new LottoNumber(i));
        }
    }

    public static LottoNumber of(int value) {
        return CACHE.get(value); // 항상 같은 인스턴스
    }
}
```

```java
// LottoTicket — extrinsic state를 관리한다
public class LottoTicket {
    private final List<LottoNumber> numbers; // 위치 정보는 여기서 관리

    // LottoNumber(7) 인스턴스는 모든 티켓이 공유
    // "7이 이 티켓의 몇 번째인가"는 List의 index가 관리
}
```

---

### 메모리에서 실제로 어떻게 생기는가

Flyweight 없이:

```
티켓1 → [LottoNumber(7), LottoNumber(13), LottoNumber(25) ...]  // 새 객체
티켓2 → [LottoNumber(7), LottoNumber(13), LottoNumber(30) ...]  // 또 새 객체
티켓3 → [LottoNumber(7), LottoNumber(22), LottoNumber(30) ...]  // 또 새 객체
// LottoNumber(7) 인스턴스가 3개
```

Flyweight 적용:

```
캐시 → { 7: LottoNumber(7), 13: LottoNumber(13), ... } // 45개만 존재

티켓1 → [→LottoNumber(7), →LottoNumber(13), →LottoNumber(25) ...]  // 참조
티켓2 → [→LottoNumber(7), →LottoNumber(13), →LottoNumber(30) ...]  // 같은 인스턴스 참조
티켓3 → [→LottoNumber(7), →LottoNumber(22), →LottoNumber(30) ...]  // 같은 인스턴스 참조
// LottoNumber(7) 인스턴스는 딱 1개. 티켓들이 같은 걸 가리킨다
```

---

### 한 줄로 정리하면

> intrinsic = 객체가 "무엇인가" → 공유해도 되는 것 extrinsic = 객체가 "어디에 있는가" → 컨텍스트마다 달라서 공유 불가

`LottoNumber(7)`이 **무엇인지**는 항상 같다. **어디에 있는지**는 티켓마다 다르다. 그래서 "무엇인지"만 객체에 저장하고, "어디에 있는지"는 바깥에서 관리한다.