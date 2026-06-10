
### 1. Problem First

설정 안 하면 어떻게 되는가.

Spring Security의 기본값은 `IF_REQUIRED`다. JWT 기반으로 만들었는데 이 설정을 빠뜨리면:

```
1. 클라이언트가 JWT로 로그인 성공
2. Spring Security가 몰래 HttpSession도 생성
3. 응답 헤더에 Set-Cookie: JSESSIONID=ABC123 포함
4. 브라우저가 이후 요청에 JSESSIONID 쿠키 자동 첨부
5. 서버는 세션으로 인증 처리 → JWT 검증 필터가 사실상 무의미
```

"JWT 시스템을 만들었는데 세션이 같이 돌고 있는" 상태가 된다. 기능은 동작하지만 설계 의도와 전혀 다른 방식으로.

---

### 2. Mechanics

**`SessionCreationPolicy`가 제어하는 것은 두 가지다.**

| 동작             | 설명                                        |
| -------------- | ----------------------------------------- |
| Session **생성** | Spring Security가 새 `HttpSession`을 만드는가    |
| Session **참조** | 기존 `HttpSession`에서 `SecurityContext`를 읽는가 |
|                |                                           |

`STATELESS`는 이 둘을 모두 차단한다.

**내부 구현 — `SessionManagementFilter` + `HttpSessionSecurityContextRepository`**

Spring Security는 인증 정보(`SecurityContext`)를 요청 간에 유지하기 위해 기본적으로 `HttpSession`에 저장한다.

```java
// HttpSessionSecurityContextRepository (Spring Security 내부)
public SecurityContext loadContext(HttpRequestResponseHolder requestResponseHolder) {
    HttpSession httpSession = request.getSession(false);
    if (httpSession != null) {
        return readSecurityContextFromSession(httpSession); // 세션에서 꺼냄
    }
    return generateNewContext();
}
```

`STATELESS`로 설정하면 이 Repository 대신 `NullSecurityContextRepository`가 등록된다.

```java
// NullSecurityContextRepository
public SecurityContext loadContext(...) {
    return SecurityContextHolder.createEmptyContext(); // 항상 빈 컨텍스트 반환
}

public void saveContext(...) {
    // 아무것도 안 함 — 저장 자체를 안 함
}
```

결과적으로 **매 요청은 독립적**이다. 이전 요청의 인증 정보가 다음 요청으로 넘어가지 않는다.

**공식 근거**

> `STATELESS` — Spring Security will never create an `HttpSession` and it will never use it to obtain the `SecurityContext`
> 
> — [Spring Security 공식 레퍼런스, SessionCreationPolicy](https://docs.spring.io/spring-security/reference/servlet/authentication/session-management.html)
> 
> _번역: STATELESS — Spring Security는 HttpSession을 절대 생성하지 않으며, SecurityContext를 얻기 위해 HttpSession을 사용하지도 않는다._

---

### 3. 이 설계를 이렇게 한 이유

**얻는 것**

JWT 필터가 매 요청마다 토큰을 검증하므로, 세션 없이도 인증이 완결된다. 서버가 상태를 들고 있지 않으니 수평 확장(서버 추가)이 자유롭다.

**잃는 것**

세션 기반 보안 기능을 전부 수동으로 처리해야 한다. 예를 들어 세션 고정 공격(Session Fixation) 방어는 Spring Security가 기본 제공하지만, STATELESS에서는 의미 없다. 대신 JWT 탈취 시나리오를 직접 설계해야 한다(이 PR의 Refresh Token DB 저장이 그 역할).

---

### PR에서 이 설정이 의미하는 것

```java
.sessionManagement(session -> session
        .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
```

이 한 줄이 선언하는 것:

> "이 서버는 세션을 쓰지 않는다. 인증은 매 요청마다 JWT 필터가 처리한다."

`csrf().disable()`과 세트로 보면 된다. CSRF 공격은 브라우저가 쿠키(세션)를 자동으로 보내는 특성을 이용하는데, 세션이 없으면 CSRF 공격 자체가 성립하지 않기 때문에 같이 비활성화한다.