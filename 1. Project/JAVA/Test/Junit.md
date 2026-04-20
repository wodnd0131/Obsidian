[공식문서](https://docs.junit.org/5.5.0/user-guide/)


# 기타
## @DisplayNameGenerator

JUnit5에서 테스트 메서드 이름을 **자동으로 가공해서 표시**해주는 어노테이션입니다.

---

### 기본 사용

```java
@DisplayNameGeneration(DisplayNameGenerator.ReplaceUnderscores.class)
class CalculatorTest {

    @Test
    void add_two_numbers_returns_sum() { ... }
    // 표시: "add two numbers returns sum"
}
```

---

### 내장 Generator 종류

**Standard** (기본값)

```java
void testAdd()  →  "testAdd()"
```

**ReplaceUnderscores**

```java
void add_two_numbers()  →  "add two numbers"
// 언더스코어를 공백으로 치환
```

**IndicativeSentences**

```java
class CalculatorTest {
    void add_two_numbers()  →  "CalculatorTest, add two numbers"
    // 클래스명 + 메서드명 조합
}
```

**Simple**

```java
void testAdd()  →  "testAdd"  // 괄호 제거
```

---

### `@DisplayName` 과 차이

```java
// @DisplayName → 메서드마다 직접 지정
@Test
@DisplayName("두 수를 더하면 합이 반환된다")
void addTest() { ... }

// @DisplayNameGeneration → 클래스 전체에 규칙 적용
@DisplayNameGeneration(DisplayNameGenerator.ReplaceUnderscores.class)
class CalculatorTest {
    @Test
    void add_two_numbers_returns_sum() { ... }  // 자동 변환
}
```

||@DisplayName|@DisplayNameGeneration|
|---|---|---|
|적용 범위|메서드 개별|클래스 전체|
|방식|직접 작성|규칙 기반 자동변환|
|유연성|높음|낮음|
|편의성|낮음|높음|

---

### 커스텀 Generator

직접 규칙을 만들 수도 있습니다.

```java
class KoreanDisplayNameGenerator extends DisplayNameGenerator.Standard {
    @Override
    public String generateDisplayNameForMethod(Class<?> testClass, Method testMethod) {
        // 메서드명 앞에 "[테스트] " 붙이기
        return "[테스트] " + testMethod.getName();
    }
}

@DisplayNameGeneration(KoreanDisplayNameGenerator.class)
class CalculatorTest { ... }
```

---

### 전체 프로젝트에 적용

매 클래스마다 붙이기 싫으면 `junit-platform.properties` 에 설정 가능합니다.

```properties
# src/test/resources/junit-platform.properties
junit.jupiter.displayname.generator.default=\
    org.junit.jupiter.api.DisplayNameGenerator$ReplaceUnderscores
```

---

### 실무에서

`ReplaceUnderscores` + 메서드명을 **한글 또는 영어 문장**으로 짜는 조합을 많이 씁니다.

```java
@DisplayNameGeneration(DisplayNameGenerator.ReplaceUnderscores.class)
class CalculatorTest {

    @Test
    void 두_수를_더하면_합이_반환된다() { ... }
    // 표시: "두 수를 더하면 합이 반환된다"
}
```

---

