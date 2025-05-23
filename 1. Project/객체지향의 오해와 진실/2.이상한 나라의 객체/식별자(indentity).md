객체가 식별 가능하다는 것은 객체를 서로 구별할 수 있느 특정한 프로퍼티가 객체 안에 존재함을 의미한다. 이 프로퍼티를 ==식별자==라 한다.

모든 객체가 식별자를 가진다는 것은, 반대로 객체가 아닌 단순한 값은 식별자를 가지지 않음을 의미한다. 이는 값과 객체의 가장 큰 차이점이다.

==값(value)==은 숫자, 날짜 등과 같은 변하지 않는 양, 수치를 모델링한다.
값의 상태는 변하지 않기 때문에, 불변 상태(immutable state)를 가진다고 말한다. 
때문에, 값의 상태가 같으면 두 인스턴스가 동일한 것으로 판단하고, 이처럼 상태를 이용해 두 값이 같은지 판단할 수 있는 성질을 ==동등성(equality)==라 한다.
- 값의 상태가 변하지 않기에, 서로 동등함을 판단할 수 있다.

객체는 시간에 따라 변경되는 상태를 포함하며, 행동을 통해 상태를 변경한다.
따라서 객체는 가변 상태(mutable state)를 가진다고 말한다.
타입이 같은 두 객체의 상태가 완전히 똑같더라도 두 객체는 독립적인 별개의 객체로 다뤄야 한다.

두 객체의 상태가 다르더라도 식별자가 같다면, 두 객체를 같은 객체로 판단할 수 있다. 이처럼 식별자를 기반으로 객체가 같은지 판단할 수 있는 성질을 ==동일성(identical)==이라고 한다.

상태가 가변적인 두 객체의 동일성을 판단하기 위해서는 상태 변경에 독립적인 별도의 식별자를 이용할 수 밖에 없다.

```
식별자란 어떤 객체를 다른 객체와 구분하는데 사용하는 객체의 프로퍼티이다. 값은 식별자를 가지지 않기 때문에 상태를 이용한 동등성 검사를 통해 두 인스턴스를 비교해야한다. 객체는 상태가 변경될 수 있기 때문에 식별자를 이용한 동일성 검사를 통해 두 인스턴스를 비교할 수 있다.
```