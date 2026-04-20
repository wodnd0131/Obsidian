

- 원래 `package-private`이었던 메서드를 리뷰 받고 `private`으로 바꿨는데
- 그러자 테스트에서 접근이 안 되는 상황이 생김
- "테스트를 위해 다시 `package-private`으로 되돌리는 건 아닌 것 같다"는 직관이 생기면서 이 고민이 시작된 것
## 글의 핵심 주제

**"private 메서드를 테스트해야 하나?"** 에 대한 두 입장을 정리하고, 본인 의견을 도출하는 글입니다.

---

## 두 입장 정리

### 입장 1 — 테스트하지 말아야 한다

> _"private 메서드는 구현 세부사항이다. 테스트하면 캡슐화가 깨진다. 꼭 테스트해야 할 만큼 복잡하다면, 그건 클래스를 쪼개야 한다는 신호다."_

- public 인터페이스만 잘 테스트되면 충분
- 테스트를 위해 접근자를 바꾸는 건 설계가 잘못된 것

### 입장 2 — 테스트해야 한다

> _"테스트의 목적이 미래의 코드 변경으로 인한 문제를 잡는 것이라면, private도 테스트하는 게 유리하다."_

- A(public)가 B(private)를 호출할 때, C(공통 의존)가 바뀌어서 A가 실패했다면 
  → B도 테스트되어 있어야 어디서 깨졌는지 알 수 있다
- public 커버리지가 100%가 안 될 때, private 테스트가 보완재가 될 수 있다

---
---
## 한 줄 요약

> private 테스트 논쟁의 본질은 **"테스트의 목적이 뭐냐"** 에 달려 있고, 
> 글쓴이는 _설계 원칙(캡슐화 유지)_ 쪽에 무게를 두되 아직 확신은 없는 상태입니다.

ㄴ 돔푸 : 필요하면 짠다

테스트의 본 목적인 회귀방지를 위해선 사실 테스트가 촘촘할수록 좋은거잖아요? 
1개 메서드에 대해 1억개의 테스트가 있다면 아마 그 코드는 매우 단단해질 것 같고요.  

그런 관점(회귀방지 우선)으로 보면 사실 private 메서드도 테스트하는게 너무 당연할 수 있고요. 
그런 관점이라면 private 접근제한자를 붙이는게 안티패턴일 수도 있어요. ㅎㅎ

---
## private 테스트하는 방법

### 1. 리플렉션 (Reflection)

```java
// private 메서드 강제 접근
Method method = MyClass.class.getDeclaredMethod("privateMethod", String.class);
method.setAccessible(true);
Object result = method.invoke(instance, "input");
```

- 가장 직접적이지만 코드가 지저분해짐
- 메서드 이름을 문자열로 쓰기 때문에 **리팩토링 시 컴파일 에러가 안 남** → 위험

### 2. package-private으로 올리기 + `@VisibleForTesting`

```java
// 프로덕션 코드
@VisibleForTesting
int parseInteger(String input) { ... }  // private → package-private

// 테스트 코드 (같은 패키지)
assertThat(instance.parseInteger("123")).isEqualTo(123);
```

- Google Guava, AOSP에서 흔히 쓰는 방식
- "이건 테스트용으로만 열어둔 거야"를 명시적으로 표현

### 3. 클래스 분리 (가장 권장)

```java
// private 메서드가 복잡하다 → 별도 클래스로 추출
public class IntegerParser {        // 원래 private 메서드였던 것
    public int parse(String input) { ... }
}

// 테스트
assertThat(new IntegerParser().parse("123")).isEqualTo(123);
```

---

## private 테스트가 필요한 경우

**1. 로직이 복잡한데 public으로 간접 검증이 어려울 때**

```java
// public 메서드가 너무 많은 걸 한꺼번에 해서
// 실패해도 어느 private에서 깨진지 모를 때
```

**2. 버그가 private 내부에서 반복적으로 발생할 때**

```
핫스팟이 명확하다 → 그 지점을 직접 겨냥하는 테스트가 효율적
```

**3. public 커버리지로 해당 분기를 만들기가 너무 복잡할 때**

```java
// private 내부의 엣지케이스를 유발하려면
// public 쪽에서 엄청 복잡한 셋업이 필요한 경우
```

---

```
public/protected  →  반드시 테스트
private           →  public으로 충분히 커버되면 생략
                      복잡하거나 버그 핫스팟이면 클래스 분리 후 테스트
테스트를 위해 접근자를 바꾸는 건  →  설계를 다시 보라는 신호
```