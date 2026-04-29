
정규표현식(Regex)을 Java에서 사용할 때 쓰는 두 핵심 클래스입니다. 둘 다 `java.util.regex` 패키지에 있어요.

---

### 기본 구조

```java
import java.util.regex.Pattern;
import java.util.regex.Matcher;

String text = "Hello, my number is 010-1234-5678";
String regex = "\\d{3}-\\d{4}-\\d{4}";

Pattern pattern = Pattern.compile(regex);   // 패턴 컴파일
Matcher matcher = pattern.matcher(text);    // 텍스트에 패턴 적용
```

> `Pattern.compile()`은 비용이 있으므로, **반복 사용 시 static으로 선언**하는 게 좋습니다.

---

### 주요 Matcher 메서드

|메서드|설명|
|---|---|
|`matches()`|**전체** 문자열이 패턴과 일치하는지|
|`find()`|패턴과 일치하는 부분을 **순서대로 탐색**|
|`group()`|가장 최근에 매칭된 문자열 반환|
|`group(n)`|n번 캡처 그룹 반환|
|`start()` / `end()`|매칭된 위치의 시작/끝 인덱스|
|`replaceAll(str)`|매칭된 모든 부분을 str로 교체|
|`replaceFirst(str)`|첫 번째 매칭만 교체|

---

### `matches()` vs `find()`

```java
String text = "010-1234-5678";

// matches() - 전체 문자열이 패턴과 완전히 일치해야 함
Pattern p = Pattern.compile("\\d{3}-\\d{4}-\\d{4}");
Matcher m = p.matcher(text);
System.out.println(m.matches()); // true

// find() - 문자열 내 어딘가에 패턴이 있으면 됨
String sentence = "전화번호는 010-1234-5678 입니다.";
Matcher m2 = p.matcher(sentence);
System.out.println(m2.matches()); // false ← 전체가 아니므로
System.out.println(m2.find());    // true  ← 부분 탐색이므로
```

---

### `find()` 반복으로 여러 개 찾기

```java
String text = "010-1234-5678 그리고 02-987-6543";
Pattern p = Pattern.compile("\\d{2,3}-\\d{3,4}-\\d{4}");
Matcher m = p.matcher(text);

while (m.find()) {
    System.out.println("발견: " + m.group());
    System.out.println("위치: " + m.start() + " ~ " + m.end());
}
// 발견: 010-1234-5678  위치: 0 ~ 13
// 발견: 02-987-6543    위치: 18 ~ 29
```

---

### 캡처 그룹 `group(n)`

괄호 `()`로 묶으면 그룹을 만들 수 있어요.

```java
String text = "2024-04-20";
Pattern p = Pattern.compile("(\\d{4})-(\\d{2})-(\\d{2})");
Matcher m = p.matcher(text);

if (m.matches()) {
    System.out.println("전체: "  + m.group(0)); // 2024-04-20
    System.out.println("연도: "  + m.group(1)); // 2024
    System.out.println("월: "    + m.group(2)); // 04
    System.out.println("일: "    + m.group(3)); // 20
}
```

---

### `replaceAll()` / `replaceFirst()`

```java
String text = "비밀번호는 abc123 이고 백업은 xyz789 입니다.";
Pattern p = Pattern.compile("[a-z]+\\d+");

// Matcher로 교체
String result1 = p.matcher(text).replaceAll("****");
System.out.println(result1); // 비밀번호는 **** 이고 백업은 **** 입니다.

String result2 = p.matcher(text).replaceFirst("****");
System.out.println(result2); // 비밀번호는 **** 이고 백업은 xyz789 입니다.
```

> 간단한 경우엔 `String.replaceAll(regex, str)` 로도 가능하지만,  
> 내부적으로 매번 `Pattern.compile()`을 호출하므로 **반복 호출 시엔 비효율적**입니다.

---

### Pattern 플래그

```java
// 대소문자 무시
Pattern p = Pattern.compile("hello", Pattern.CASE_INSENSITIVE);
System.out.println(p.matcher("HELLO World").find()); // true

// 여러 플래그 동시 사용
Pattern p2 = Pattern.compile("hello", Pattern.CASE_INSENSITIVE | Pattern.MULTILINE);
```

|플래그|설명|
|---|---|
|`CASE_INSENSITIVE`|대소문자 무시|
|`MULTILINE`|`^`, `$`가 각 줄의 시작/끝에 매칭|
|`DOTALL`|`.`이 줄바꿈 문자도 포함|

---

### 자주 쓰는 정규표현식 패턴

|패턴|의미|
|---|---|
|`\\d`|숫자 [0-9]|
|`\\w`|단어 문자 [a-zA-Z0-9_]|
|`\\s`|공백 문자|
|`.`|임의의 한 문자 (줄바꿈 제외)|
|`+`|1개 이상|
|`*`|0개 이상|
|`?`|0개 또는 1개|
|`{n,m}`|n개 이상 m개 이하|
|`^` / `$`|문자열 시작 / 끝|
|`[abc]`|a, b, c 중 하나|