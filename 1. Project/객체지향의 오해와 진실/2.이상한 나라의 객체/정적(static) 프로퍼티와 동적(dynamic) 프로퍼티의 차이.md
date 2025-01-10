정적(static) 프로퍼티와 동적(dynamic) 프로퍼티의 차이를 자바를 기준으로 설명해드리겠습니다.

정적(static) 프로퍼티:
- 클래스 수준에서 관리되며, 모든 인스턴스가 공유하는 값입니다.
- 클래스가 메모리에 로드될 때 생성되어 프로그램이 종료될 때까지 유지됩니다.
- static 키워드로 선언됩니다.
- 객체 생성 없이도 클래스 이름으로 직접 접근이 가능합니다.

예시:
```java
public class User {
    public static int userCount = 0; // 정적 프로퍼티
    private String name; // 인스턴스 프로퍼티
    
    public User(String name) {
        this.name = name;
        userCount++; // 모든 User 인스턴스가 공유하는 값
    }
}
```

동적 프로퍼티:
- 각 객체의 인스턴스마다 별도로 가지는 값입니다.
- 객체가 생성될 때 메모리가 할당되고, 객체가 소멸될 때 메모리에서 제거됩니다.
- 인스턴스 변수라고도 부릅니다.
- 반드시 객체를 생성한 후에 접근할 수 있습니다.

추가로 참고할 만한 내용을 말씀드리면:

이펙티브 자바(Joshua Bloch 저)에서는 정적 프로퍼티의 사용에 대해 다음과 같은 조언을 합니다:
- 정적 프로퍼티는 상수나 싱글톤 인스턴스를 표현할 때 주로 사용하는 것이 좋습니다.
- 가변적인 정적 프로퍼티는 전역 변수와 같은 효과를 내므로 사용을 최소화해야 합니다.

------
---

자바에서 프로퍼티의 위치와 종류에 대해 좀 더 명확하게 설명드리겠습니다.

클래스 내 변수들의 정확한 용어는 다음과 같습니다:

1. 클래스 변수 (Class Variables)
```java
public class Example {
    static int count = 0;  // 클래스 변수 (정적 변수)
    static final String COMPANY = "ABC";  // 정적 상수
}
```
- static 키워드로 선언
- 정적 프로퍼티(static property)라고도 함
- 클래스 로딩 시점에 메모리 할당

2. 인스턴스 변수 (Instance Variables)
```java
public class Example {
    private String name;  // 인스턴스 변수
    private int age;     // 인스턴스 변수
}
```
- 클래스 내부에서 선언되지만 static이 아닌 변수
- 객체가 생성될 때마다 새로운 메모리 할당
- 동적 프로퍼티라고도 함

3. 지역 변수 (Local Variables)
```java
public void someMethod() {
    int localVar = 10;  // 지역 변수
    String temp = "temporary";  // 지역 변수
}
```
- 메소드 내부에서 선언되는 변수
- 메소드 실행 시에만 존재하고 종료되면 소멸
- 이는 엄밀히 말하면 프로퍼티가 아님

주의할 점은, 앞서 설명했던 "동적 프로퍼티"는 일반적으로 인스턴스 변수를 지칭하며, 지역 변수와는 다릅니다. 지역 변수는 객체의 상태를 구성하는 프로퍼티가 아니라 메소드의 실행 컨텍스트에서만 존재하는 임시 데이터입니다.