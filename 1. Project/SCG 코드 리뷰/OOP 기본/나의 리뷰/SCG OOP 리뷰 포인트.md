#SCG #SCG/OOP

## 🏗️ OOP 관점

### 도메인 모델링

- **원시값 포장** — `LottoNumber`, `LottoPrice`, `MatchCount` 등 int/String을 그대로 쓰지 않았는지
- **일급 컬렉션** — `List<Integer>` 대신 `Lotto`, `LottoTickets` 같은 전용 클래스로 감쌌는지
- **불변 객체(Immutable)** — `Lotto`의 번호 리스트가 외부에서 변경 가능한 상태인지 (`Collections.unmodifiableList`)
- **값 객체(VO)** — `equals`/`hashCode` 재정의 여부, 동등성 vs 동일성 구분
- **Tell, Don't Ask** — 로또 당첨 여부를 외부에서 꺼내서 판단하는지, 객체 스스로 판단하는지
- **도메인 책임 위치** — 당첨 번호 비교 로직이 어느 클래스에 있는지 (`Lotto`? `WinningLotto`? `LottoService`?)
- **열거형(Enum) 활용** — `Rank`/`Prize`를 Enum으로 모델링했는지, 상금 로직을 Enum 내부로 응집했는지
- **정적 팩토리 메서드** — 생성자 직접 노출 vs `Lotto.of(...)` 패턴

### 설계 원칙

- **SRP** — `LottoMachine`이 생성 + 검증 + 출력까지 다 하고 있지 않은지
- **의존성 방향** — `domain`이 `view`나 `util`에 의존하고 있지 않은지
- **패키지 구조** — `domain` / `view` / `controller` 계층 분리가 되어 있는지

---

## 🧹 클린코드 관점

### 네이밍

- **축약 금지** — `cnt`, `num`, `amt` 같은 변수명이 없는지
- **의도 드러내는 이름** — `list1`, `numbers` vs `purchasedTickets`, `winningNumbers`
- **컬렉션 네이밍** — 단수/복수 구분, `lottoList` vs `lottos`

### 메서드

- **10라인 제한** — 단순 카운팅이 아닌 "이 메서드가 한 가지 일을 하는가"
- **파라미터 수** — 3개 이상이면 객체로 묶을 것을 고려했는지
- **Boolean 반환 메서드 네이밍** — `isWinning()`, `hasNumber()` 등 동사 형태

### 구조

- **indent depth 1 준수** — for + if 중첩이 있는지
- **else 제거** — early return 패턴으로 전환했는지
- **3항 연산자 금지** 준수 여부
- **매직 넘버** — `1000`, `6`, `45` 등이 상수로 추출되어 있는지 (`LOTTO_PRICE`, `LOTTO_NUMBER_MAX`)
- **스트림 가독성** — stream이 과하게 체이닝되어 오히려 읽기 어렵지 않은지

### 입출력 분리

- **View 분리** — `System.out.println`이 도메인 객체 안에 있지 않은지
- **InputView / OutputView** 명확한 역할 분리
- **예외 처리** — 잘못된 금액 입력 시 처리 여부, `IllegalArgumentException` vs `IllegalStateException` 구분

---

## 🧪 TDD 관점

### 테스트 설계

- **단위 테스트 범위** — `Lotto`, `WinningLotto`, `Rank` 각각 독립적으로 테스트 가능한지
- **테스트 가능한 설계** — `Random` 생성 로직이 도메인 내부에 박혀 있어서 테스트가 불가능한 구조인지 (→ **의존성 주입**으로 분리)
- **픽스처 재사용** — `@BeforeEach`, 헬퍼 메서드로 중복 setup 제거했는지
- **DisplayName 작성** — `@DisplayName("1000원 미만 입력 시 예외가 발생한다")` 형태로 의도가 드러나는지
- **경계값 테스트** — 번호 범위(1~45), 구매 금액(0원, 999원, 1000원), 번호 중복 등

### 테스트 품질

- **테스트 독립성** — 테스트 간 공유 상태가 없는지
- **하나의 테스트, 하나의 검증** — `assertAll` 과용 vs 관심사 분리
- **예외 테스트** — `assertThatThrownBy` 활용, 예외 메시지까지 검증하는지
- **수익률 계산 테스트** — 부동소수점 비교 시 `offset` 사용 여부 (`assertThat(result).isCloseTo(0.35, offset(0.01))`)
- **Enum Rank 테스트** — 각 등수별 조건 분기 테스트가 있는지

---

## ⚡ 자주 놓치는 포인트 (보너스)

|포인트|키워드|
|---|---|
|`Collections.shuffle()` 후 정렬 안 함|출력 순서 보장|
|수익률 소수점 처리|`BigDecimal` vs `double`|
|번호 유효성 검증 위치|생성자 vs 정적 팩토리|
|`Rank.MISS` 처리|통계 출력에서 제외|
|`WinningLotto`와 `Lotto` 분리|보너스 번호 대비 확장성|
