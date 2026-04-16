#TDD
JUnit 5는 자바 언어를 사용하는 소프트웨어 개발자들을 위한 테스트 프레임워크 중 하나다.
이는 자바 언어로 작성된 소프트웨어의 품질을 향상시키기 위해 사용되며, 주로 단위 테스트를 작성하고 실행하는 것에 목적을 둔다.

참고 : [JUnit5 User Guide](https://junit.org/junit5/docs/current/user-guide/)

# Annotation
1.  @Test
'@Test' 애노테이션은 해당 메서드가 테스트 메서드임을 나타낸다.
테스트 메서드는 반환 타입으로 'void'를 가진다.
테스트 메서드 명은 테스트의 으도를 나타내는 이름으로 작성하는 게 좋다.
```java
@Test  
void Test_애노테이션을_붙여_테스트_메서드로_만든다() {}
```

2. @DisplayName
'@DisplayName' 애노테이션은 해당 테스트의 이름을 나타낸다.
테스트명에 한글 작성에는 3가지 문제점이 있다.
- 도구 및 라이브러리의 호환성: 일부 개발 도구, 라이브러리 및 프레임 워크는 한글을 지원하지 않는다.
- 언어 혼합 : 언어 식별자가 혼합된 코드베이스에서는 일관성을 유지하기 어려울 수 있다. 전체 가독성을 위해 하나의 언어를 사용한다.
- 경고 발생 : 무의미한 경고가 많다면 중요한 경고를 놓칠 수 있다. `Non-ASCII characters`
따라서, 테스트 명을 한글로 작성하고 싶다면 '@DisplayName' 애노테이션을 활용한다.

3. @Nested
'@Nested' 애노테이션이 없다면 해당 메서드는 중첩 클래스가 아니다.
따라서 해당 메서드는 중첩 클래스 내부에 있는 것이 아니기 때문에, 테스트 메서드로서 역할을 수행하지 않는다.

4. @Disabled
'@Disabled' 애노테이션은 해당 테스트를 비활성화한다.
테스트를 비활성화한다는 것은 테스트 실행을 하지 않음을 뜻한다.
'@Test'와 함께 사용된다.

# Assertions
1. assertEquals
'assertEquals' 메서드는 두 값이 같은지 비교한다.
두 값이 같다면 테스트는 성공하며, 다를 경우 실패한다.

'assertEquals' 메서드는 'org.junit.jupiter.api.Assertions' 클래스에 정의되어 있다.
따라서, 'Assertion' 클래스를 static으로 import 하여 사용하는 것이 좋다.

'assertEquals(expected, actual)'의 형태로 오버로딩되어 있다.
'expected'는 예상되는 값이며, 'actual'은 실제 값이다.

또한, 'equals' 메소드를 사용하기에 'expected'와 'actual' 각각에 'equals'가 정의되어 있어야 사용 가능하다.

다음은, eqauls를 정의하는 예시다.
```java
class LocalObject {  
    private final int value;  
  
    public LocalObject(int value) {  
        this.value = value;  
    }  
  
    @Override  
    public boolean equals(Object o){  
        if(!(o instanceof LocalObject object)){  
            return false;  
        }  
        return this.value==object.value;  
    }  
}
```
`o instanceof LocalObject object` 이 코드에서는 o가 LocalObject 타입인지, 그리고 해당 타입으로 캐스팅한 object를 선언함을 의미한다.
'assertEquals(expected,actural,message)' 에서 message는 실패했을 경우 출력되는 메세지이다.

반대로 'assertNotEquals' 메서드는 두 값이 서로 다름을 테스트한다.

2. assertSame
'assertSame' 메서드는
