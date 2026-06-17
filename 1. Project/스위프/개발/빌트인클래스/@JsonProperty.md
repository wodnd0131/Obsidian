Java에서 `@JsonProperty`는 **Jackson(잭슨) 라이브러리**가 제공하는 어노테이션으로, **자바 객체의 필드와 JSON 데이터의 키(Key)를 매핑**해주는 역할을 합니다.

개발을 하다 보면 자바와 JSON의 명명 규칙(Naming Convention)이 달라서 문제가 생기곤 합니다. 자바는 보통 `camelCase`를 쓰고, JSON API들은 `snake_case`를 자주 쓰니까요.

작성하신 코드와 함께 구체적으로 설명해 드릴게요.

### 1. 왜 사용하나요? (예시)

만약 외부 API가 로그인 성공 후 다음과 같은 JSON 데이터를 보내준다고 가정해 보겠습니다.

JSON

```
{
  "access_token": "abc123xyz",
  "refresh_token": "def456uvw"
}
```

이 데이터를 자바 객체로 받아오고 싶을 때, 자바에서는 변수명을 `access_token`처럼 언더바(_)를 넣어 만드는 것을 지양합니다. 자바 스타일(`camelCase`)에 맞추려면 `accessToken`이라고 적어야 하죠.

이때 `@JsonProperty`가 중간에서 **통역사 역할**을 해줍니다.

Java

```
public record TokenResponse(
    @JsonProperty("access_token") String accessToken,
    @JsonProperty("refresh_token") String refreshToken
) {}
```

> **"JSON에 `access_token`이라고 적힌 값이 들어오면, 자바의 `accessToken` 변수에 쏙 넣어줘!"** 라고 Jackson에게 알려주는 것입니다.

### 2. 양방향으로 작동합니다

`@JsonProperty`는 데이터를 읽을 때(역직렬화)뿐만 아니라, 데이터를 내보낼 때(직렬화)도 작동합니다.

- **JSON ➡️ Java (역직렬화 / Deserialization):** `access_token`이라는 JSON 키를 자바의 `accessToken` 필드로 매핑해 줍니다.
    
- **Java ➡️ JSON (직렬화 / Serialization):** 자바의 `TokenResponse` 객체를 다시 JSON으로 변환할 때, `accessToken` 필드가 아니라 `@JsonProperty`에 지정된 `"access_token"`이라는 키로 변환되어 출력됩니다.
    

### 💡 한 줄 요약

> `@JsonProperty("이름")`은 **"자바 변수 이름이 뭐든 간에, JSON이랑 주고받을 때는 무조건 여기 적힌 `"이름"`으로 취급해라"**라는 뜻입니다.