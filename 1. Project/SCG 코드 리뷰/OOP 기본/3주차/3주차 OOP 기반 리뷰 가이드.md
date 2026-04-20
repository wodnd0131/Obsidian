#SCG/3주차 #OOP/Review 
## Priority 1 — 원시값 포장 → 일급 컬렉션 → 불변 (연결해서 보기)

### 원시값 포장이 없다

**로또 번호 하나 (`Integer`)**

`LottoTicket`이 `List<Integer>`를 들고 있는데, 번호 하나를 표현하는 클래스가 없다. 그 결과가 뭐냐면 — **1~45 범위 검증이 코드 어디에도 없다.** `validate()`는 중복 체크만 하고, 범위는 `RandomNumberListGenerator`가 알아서 1~45를 뽑는 것에 의존하고 있다. 사용자가 `999`를 입력해도 막을 방법이 없는 구조.

번호 하나를 포장하는 클래스가 있었다면 그 클래스 생성 시점에 범위 검증이 들어갔을 것이고, `LottoTicket`은 그걸 신뢰할 수 있었을 것.

**금액 (`Integer price`)**

`Cashier.validatePrice()`가 금액에 대한 비즈니스 규칙(양수인지, 1000원 이상인지, 1000원 단위인지)을 들고 있다. 금액 자체의 규칙인데 금액을 다루는 `Cashier`에 있다 보니, 금액이 다른 곳에서 쓰일 때도 `Cashier`를 경유하거나 검증 로직이 복붙되는 구조가 된다.

---

### 일급 컬렉션이 내부를 다 열어준다

`LottoTicket`은 `List<Integer>`를 포장한 일급 컬렉션의 형태를 갖추고 있는데, `getTicket()`으로 내부 리스트를 그대로 반환하고 있다. 그래서 `OutputView`가 이렇게 된다:

```java
// OutputView.java:46

List<Integer> ticket = lottoTicket.getTicket();
System.out.println(ticket);
```

포장의 의미가 없어지는 지점이다. 외부에서 리스트를 직접 다루는 코드가 생기면, 이후 출력 포맷이 바뀌거나 티켓 표현 방식이 바뀔 때 `OutputView`를 고쳐야 한다.

`Lotto`도 마찬가지다. `getTickets()`로 티켓 목록을 통째로 반환하고 있어서 `OutputView`가 직접 순회한다 (`OutputView.java:12`).

---

### 불변이라고 했는데 불변이 아니다

`LottoTicket`의 가장 중요한 포인트.

```java
// LottoTicket.java:13-16
public LottoTicket(List<Integer> ticket) {
    validate(ticket);
    Collections.sort(ticket);   // ← 외부에서 넘긴 리스트를 직접 변경
    this.ticket = ticket;       // ← 외부 리스트 참조를 그대로 저장
}
```

두 가지 문제가 동시에 있다:

1. `Collections.sort(ticket)` — 생성자에 넘긴 리스트가 호출자 입장에서도 정렬되어버린다. 의도하지 않은 사이드 이펙트.
2. `this.ticket = ticket` — 외부에서 같은 리스트 참조를 들고 있으면, 나중에 그 리스트를 수정하면 `LottoTicket` 내부 상태도 바뀐다. `final`을 써도 참조가 final인 것이지 내용이 final인 게 아니다.

`field`에 `final`을 붙이는 것만으로는 불변이 완성되지 않는다. **방어적 복사**가 빠져 있다.

`getTicket()`도 같은 이유로 반환할 때 `Collections.unmodifiableList()` 또는 새 복사본을 반환해야 진짜 불변이다.

---

## Priority 2 — Tell, Don't Ask + Enum

> P1 작업 후 자연스럽게 이 질문이 생길 것 → 그때 보는 것 권장

**Cashier가 너무 많이 꺼내서 계산한다**

```java
// Cashier.java:30-36
List<Integer> results = lotto.getResults(winnerTicket);
return new LottoResult(
    Collections.frequency(results, THREE_CORRECT),
    ...
);
```

`Lotto`에게 결과를 물어봐서(Ask), `Cashier`가 직접 카운팅한다. 결과를 집계하는 책임이 `Cashier`에 있는데, 이 로직은 어떤 매칭이 몇 번 일치인지에 대한 도메인 지식이다.

**당첨 조건과 상금이 분리되어 두 곳에 흩어져 있다**

```java
// Cashier.java:40-43  — 상금 하드코딩
Integer totalProfit = 5000 * result.getThreeCorrectCount()
    + 50000 * result.getFourCorrectCount()
    ...

// OutputView.java:24-27  — 출력에도 같은 상금 정보
System.out.println("3개 일치 (5000원)- " + ...);
```

"3개 일치 = 5000원"이라는 사실이 `Cashier`와 `OutputView` 양쪽에 나뉘어 있다. `Rank` Enum 하나로 묶으면 "몇 개 일치 / 상금 / 출력 문구"를 한 곳에서 관리할 수 있다. 나중에 5개+보너스 조건이 추가될 때 `Cashier`와 `OutputView`를 동시에 고쳐야 하는 상황이 바로 이 구조에서 나온다.

---

## 작업 순서 제안

1. **`LottoNumber` 클래스 만들기** — `Integer` 하나를 포장, 생성 시 1~45 범위 검증
2. **`LottoTicket` 방어적 복사 추가** — 생성자에서 새 리스트로 복사 후 정렬, `getTicket()`도 불변 반환
3. **`getTicket()` 노출 최소화** — 외부에서 리스트를 직접 다루지 않아도 되도록
4. P1이 잡히면 `Rank` Enum, `Cashier` 책임 정리 순서로

`Rank Enum` 이부분 말야. 좋긴한데, 그렇게되었을때, view와 도메인의 의존관계는 깨지지 않겠지?

좋은 포인트야. 결론부터 말하면 **의존 방향은 깨지지 않아.** 근데 거기서 파생되는 설계 선택이 하나 있어.

---

## 왜 방향이 안 깨지냐

지금도 `OutputView`는 도메인 클래스를 이미 참조하고 있어.

```
view.OutputView  →  domain.Lotto
view.OutputView  →  domain.LottoResult
view.OutputView  →  domain.LottoTicket
```

`Rank` Enum도 domain 패키지에 들어갈 거니까, `OutputView`가 `Rank`를 쓰면:

```
view.OutputView  →  domain.Rank   ← 기존과 같은 방향
```

**view → domain 방향**이고, domain이 view를 모르는 한 의존성 규칙은 유지돼.

---

## 근데 여기서 선택이 하나 생겨

`Rank` Enum에 **출력 문자열을 넣느냐 마느냐**야.

**A안 — Rank가 출력 문자열까지 들고 있는 경우**

```java
FIVE(5, 1_500_000, "5개 일치 (1500000원)- ")
```

- 방향은 안 깨지지만, 도메인 클래스가 UI 관심사(한국어 출력 포맷)를 품게 됨
- 도메인이 View를 import하는 건 아니지만, "출력하면 어떻게 보일지"를 도메인이 알고 있는 모양새

**B안 — Rank는 순수 도메인 개념만 들고, 출력은 View가 조합하는 경우**

```java
FIVE(5, 1_500_000)   // 일치 개수 + 상금만
```

- `OutputView`가 `rank.getMatchCount()`, `rank.getPrize()`를 읽어서 문자열을 직접 조합
- 도메인은 도메인 언어만, View는 표현만 담당 → 역할이 깔끔하게 분리

---

## 어떻게 판단하면 되냐

> **"이 정보가 UI가 없어도 의미있는 정보냐?"**

- 일치 개수 → YES, 도메인 개념
- 상금액 → YES, 도메인 개념
- `"5개 일치 (1500000원)- "` 한국어 문자열 → NO, View의 관심사

그래서 B안이 더 원칙에 맞아. 의존 방향이 깨지는 게 아니라, **도메인 순수성**의 문제야.