JSON(JavaScript Object Notation)은 데이터를 표현하는 텍스트 형식이다.

---

## 한 줄 정의

```
키-값 쌍으로 데이터를 표현하는 텍스트 포맷
언어에 독립적 (Java, Python, JS 어디서나 읽고 씀)
```

---

## 생김새

```json
{
  "name": "홍길동",
  "age": 25,
  "isStudent": false,
  "scores": [90, 85, 92],
  "address": {
    "city": "서울",
    "zip": "04524"
  }
}
```

지원하는 타입은 6가지뿐이다.

```
문자열  "hello"
숫자    42, 3.14
불리언  true, false
null    null
배열    [1, 2, 3]
객체    {"key": "value"}
```

---

## Java 객체와 비교

```java
// Java 객체
public class User {
    String name = "홍길동";
    int age = 25;
}

// JSON 표현
{"name": "홍길동", "age": 25}
```

Java 객체는 JVM 메모리 위에만 존재한다. JSON은 텍스트라서 네트워크 전송, 파일 저장 어디서든 쓸 수 있다.

이전에 다룬 [[직렬화, 역직렬화]]가 바로 이 변환이다.

```
Java 객체 → JSON 문자열  (직렬화)
JSON 문자열 → Java 객체  (역직렬화)
```

---

