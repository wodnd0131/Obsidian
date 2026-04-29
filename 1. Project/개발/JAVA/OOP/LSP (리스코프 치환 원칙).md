**한 줄 정의**

> 상위 타입 자리에 하위 타입을 넣어도 프로그램이 동일하게 동작해야 한다.

**코드로**

java

```java
class LottoNumber {
    protected final int value;
    
    public boolean isValid() {
        return value >= 1 && value <= 45;
    }
}

class BonusNumber extends LottoNumber {
    @Override
    public boolean isValid() {
        return value >= 1 && value <= 45 && someExtraCondition();
    }
}

// 이 메서드는 LottoNumber를 받는다
void validate(LottoNumber number) {
    if (!number.isValid()) throw new IllegalArgumentException();
}
```

`validate()`는 `LottoNumber`를 기대하고 짰다. `BonusNumber`를 넣었을 때 `someExtraCondition()`이 추가 제약을 걸어서 `LottoNumber`로는 valid했던 게 invalid가 된다면 — LSP 위반이다. 호출자가 `LottoNumber`인지 `BonusNumber`인지 신경 써야 하는 순간 이미 깨진 것.

**Barbara Liskov 원문 (1987)**

> "If for each object o1 of type S there is an object o2 of type T such that for all programs P defined in terms of T, the behavior of P is unchanged when o1 is substituted for o2, then S is a subtype of T." (타입 T로 정의된 모든 프로그램 P에서, T의 객체 o2를 S의 객체 o1으로 대체해도 P의 동작이 바뀌지 않는다면, S는 T의 서브타입이다.)