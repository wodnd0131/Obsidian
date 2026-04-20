#Java/Pattern #Status/완료 
### 뭔가

1996년 Sun Microsystems가 정의한 Java 컴포넌트 명세(JSR-beans)다. 
GUI 빌더 툴에서 컴포넌트를 조립하기 위한 규약으로 만들어졌다. **지금은 그 맥락은 사라졌고 관행만 남았다.**

규약:

- 기본 생성자(no-arg constructor) 필수
- 모든 필드는 `private`
- `getter`/`setter`로 접근

```java
public class LottoTicketOrder {
    private int totalCount;
    private int manualCount;
    private boolean superLotto;

    public LottoTicketOrder() {} // 기본 생성자

    public void setTotalCount(int totalCount) { this.totalCount = totalCount; }
    public void setManualCount(int manualCount) { this.manualCount = manualCount; }
    public void setSuperLotto(boolean superLotto) { this.superLotto = superLotto; }

    public int getTotalCount() { return totalCount; }
    // ...
}

// 사용
LottoTicketOrder order = new LottoTicketOrder();
order.setTotalCount(10);
order.setManualCount(3);
order.setSuperLotto(false);
```

Telescoping Constructor 대안으로 쓰이기도 했다. 
파라미터 순서 외울 필요 없고, 필요한 것만 set하면 되니까.

---

### Bloch이 왜 안티패턴으로 봤는가

**문제 1 — 불완전한 상태가 존재한다:**

```java
LottoTicketOrder order = new LottoTicketOrder();
// 여기서 이미 객체가 존재한다
// totalCount = 0, manualCount = 0 — 유효하지 않은 상태

order.setTotalCount(10);
// 여기서도 존재한다 — manualCount가 아직 없음

order.setManualCount(3);
// 여기서야 완전한 상태
```

생성과 초기화 사이에 갭이 생긴다. 이 사이에 객체를 누군가 사용하면 터진다.

**문제 2 — 불변 객체를 만들 수 없다:**

```java
order.setTotalCount(10);
// 나중에 언제든
order.setTotalCount(999); // 외부에서 변경 가능
```

setter가 있는 한 불변성 보장이 불가능하다.

**Effective Java 3rd, Item 2:**

> "Unfortunately, the JavaBeans pattern has serious disadvantages of its own. Because construction is split across multiple calls, a JavaBean may be in an inconsistent state partway through its construction."

---

### Bloch의 최종 권장 — Builder

```java
LottoTicketOrder order = new LottoTicketOrder.Builder(10) // 필수값
    .manualCount(3)       // 선택값
    .superLotto(false)    // 선택값
    .build();             // 여기서 검증 + 생성 — 불완전한 상태 없음
```

`build()` 호출 전까지 `LottoTicketOrder` 인스턴스가 존재하지 않는다. 불완전한 상태가 원천 차단된다.

---

### 세 패턴 비교

```
Telescoping Constructor  → 파라미터 순서를 외워야 함. 가독성 최악
JavaBeans               → 불완전한 상태 존재. 불변 불가
Builder                 → 이름 있는 파라미터. 불완전한 상태 없음. 불변 가능
```