#Java/Idiom  #Status/완료 
### 핵심 특성

값 객체는 **식별자가 아닌 값으로 동등성을 판단**하는 객체다.

```java
// Bad — int 그대로 사용
int price = 1000;
int count = 1000;
// 컴파일러가 잘못된 대입을 잡아주지 못함

// Good — 값 객체
public final class LottoPrice {
    private final int value;

    public LottoPrice(int value) {
        validate(value);
        this.value = value;
    }

    private void validate(int value) {
        if (value % 1000 != 0 || value <= 0) {
            throw new IllegalArgumentException("로또 금액은 1000원 단위여야 합니다.");
        }
    }

    public int ticketCount() {
        return value / 1000;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof LottoPrice)) return false;
        return value == ((LottoPrice) o).value;
    }

    @Override
    public int hashCode() {
        return Integer.hashCode(value);
    }
}
```

`LottoPrice price = new LottoCount(5)` — **컴파일 에러**. 타입 시스템이 실수를 막아준다.

### VO의 세 가지 조건

1. **불변(Immutable)** — setter 없음, 모든 필드 `final`
2. **동등성(Equality)** — `equals`/`hashCode`를 값 기반으로 재정의
3. **자가 검증(Self-validating)** — 생성 시점에 유효하지 않은 상태를 거부
