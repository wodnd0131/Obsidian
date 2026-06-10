
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


## Spring Security 내부 코드로 보는 전략 패턴

---

### 전략 패턴 구조

```
SecurityContextRepository (인터페이스 = 전략)
    ├── HttpSessionSecurityContextRepository  (STATEFUL 전략)
    ├── NullSecurityContextRepository         (STATELESS 전략)
    └── DelegatingSecurityContextRepository  (위임 래퍼)
```

**인터페이스 정의** ([Spring Security GitHub](https://github.com/spring-projects/spring-framework))

```java
// spring-security-web/.../context/SecurityContextRepository.java
public interface SecurityContextRepository {

    @Deprecated
    SecurityContext loadContext(HttpRequestResponseHolder requestResponseHolder);

    default DeferredSecurityContext loadDeferredContext(HttpServletRequest request) { ... }

    void saveContext(SecurityContext context,
                     HttpServletRequest request,
                     HttpServletResponse response);

    boolean containsContext(HttpServletRequest request);
}
```

---

### 세션이 있었다면 — 요청 흐름 비교

**STATELESS (현재 PR)**

```
요청 1: POST /login
  → JwtFilter: 토큰 없음 → 패스
  → UsernamePasswordAuthenticationFilter: 인증 처리
  → NullSecurityContextRepository.saveContext() → 아무것도 안 함
  → 응답: { accessToken: "eyJ...", refreshToken: "eyJ..." }
  → Set-Cookie 헤더 없음

요청 2: GET /api/mypage
  → JwtFilter: "Authorization: Bearer eyJ..." 헤더 파싱
  → 서명 검증 → 성공
  → SecurityContextHolder에 Authentication 직접 세팅
  → NullSecurityContextRepository.loadContext() → 빈 컨텍스트 (무시)
  → Controller 진입
  → 요청 끝나면 SecurityContextHolder.clearContext() → 완전 소멸
```

**STATEFUL (설정 없을 때 기본값)**

```
요청 1: POST /login
  → UsernamePasswordAuthenticationFilter: 인증 처리
  → HttpSessionSecurityContextRepository.saveContext()
      → HttpSession 신규 생성 (서버 메모리에 저장)
      → session.setAttribute("SPRING_SECURITY_CONTEXT", context)
  → 응답: { accessToken: "eyJ...", refreshToken: "eyJ..." }
  → Set-Cookie: JSESSIONID=3F2A9B...   ← 이 헤더가 자동으로 붙음

요청 2: GET /api/mypage (쿠키 없이, JWT만 보냄)
  → JwtFilter: 토큰 검증 성공 → SecurityContext 세팅
  → 동작은 함

요청 2': GET /api/mypage (JSESSIONID 쿠키와 함께)
  → HttpSessionSecurityContextRepository.loadContext()
      → session에서 SecurityContext 꺼냄
  → JwtFilter 도달 전에 이미 인증 완료
  → JWT 검증 필터가 사실상 bypass됨   ← 문제
```

---

### 핵심 문제: 두 인증 경로가 공존

세션이 있을 때 실제로 발생하는 위험한 시나리오:

```
시나리오: Access Token 만료 후 세션으로 계속 인증되는 경우

1. 사용자 로그인
   → Access Token 발급 (만료 1시간)
   → JSESSIONID 세션도 생성 (기본 만료 30분)

2. 45분 후: Access Token 만료
   → 서버 의도: "토큰 만료 → 재발급 필요"

3. 실제: 브라우저가 JSESSIONID 쿠키 자동 전송
   → 세션이 살아있으면 그냥 인증 통과
   → 토큰 만료가 의미 없어짐

4. 세션 기본 만료는 30분이지만
   요청이 올 때마다 세션이 갱신됨(sliding expiration)
   → 활성 사용자는 사실상 영구 로그인 상태
```

JWT를 도입해서 얻으려는 "만료 제어"가 세션 때문에 무력화되는 것이다. `STATELESS` 설정은 이 경로 자체를 차단한다.


### 1. JSESSIONID — 누가 만들고 어디에 저장하는가

Spring Security가 만드는 게 아니라, **서블릿 컨테이너(Tomcat)가 만든다.**

```
Spring Security → "인증됐으니 세션에 저장해"
      ↓
HttpSessionSecurityContextRepository.saveContext()
      ↓
request.getSession(true)  ← 여기서 Tomcat한테 세션 요청
      ↓
Tomcat → 새 HttpSession 생성 + JSESSIONID 발급
      ↓
응답 헤더에 Set-Cookie: JSESSIONID=ABC123 자동 추가
```

Spring Security는 "세션에 저장해달라"고 요청하는 것이고, 실제 세션 생성과 JSESSIONID 발급은 Tomcat이 한다.

**저장 위치:**

```
Tomcat 인메모리 (기본값)
  └── ConcurrentHashMap<String, Session>
        ├── "ABC123" → { SecurityContext, 생성시각, 마지막접근시각 }
        ├── "DEF456" → { SecurityContext, ... }
        └── ...
```

별도 설정 없으면 Tomcat JVM 힙 메모리 안에 저장된다. 서버 재시작하면 전부 소멸.

---

## 2. JWT 인증 시스템을 만들어도 세션으로 인증되는가

그렇다. Spring Security 필터 체인 구조 때문이다.

```
HTTP 요청
    ↓
SecurityContextPersistenceFilter   ← 여기서 세션 조회
    ↓                                 SecurityContext를 미리 로드해둠
JwtAuthenticationFilter            ← JWT 검증
    ↓
FilterSecurityInterceptor          ← 최종 접근 제어
```

`SecurityContextPersistenceFilter`가 JWT 필터보다 **앞에** 있다. JSESSIONID 쿠키가 오면 이미 이 시점에 SecurityContext가 채워진다.

```java
// SecurityContextPersistenceFilter 핵심 동작
public void doFilter(ServletRequest req, ...) {
    
    // JWT 필터 실행 전에 먼저 세션에서 꺼냄
    SecurityContext contextBeforeChainExecution =
        repo.loadContext(holder);  // ← JSESSIONID로 세션 조회 성공하면 여기서 끝

    SecurityContextHolder.setContext(contextBeforeChainExecution);

    chain.doFilter(request, response);  // 이후 JWT 필터 실행
}
```

JWT 필터에서 SecurityContext를 채우려 해도, **이미 세션이 채워놓은 상태**라면 덮어쓰는 구현인지 아닌지에 따라 결과가 달라진다. 그리고 JWT가 없는 요청이 와도 세션만 있으면 그냥 통과한다.

**실제 시나리오:**

```
1. 로그인 → JWT 발급 + JSESSIONID도 발급됨

2. 공격자가 JSESSIONID 탈취 (XSS 등)

3. 공격자 요청: JSESSIONID 쿠키만 첨부, JWT는 없음
   → SecurityContextPersistenceFilter가 세션에서 인증 정보 로드
   → JWT 필터는 토큰 없으니 그냥 패스
   → 인증 완료

4. Access Token 만료돼도 세션이 살아있으면 계속 접근 가능
   → JWT 만료 제어가 완전히 무력화
```

---

## 정리

| |STATELESS|기본값 (IF_REQUIRED)|
|---|---|---|
|세션 생성 주체|없음|Tomcat|
|저장 위치|없음|Tomcat 인메모리|
|인증 경로|JWT 하나|JWT + 세션 두 개 공존|
|JWT 만료 제어|정상 동작|세션이 살아있으면 무력화|

`STATELESS` 설정 한 줄이 하는 일은 결국 **"인증 경로를 JWT 하나로 강제하는 것"** 이다.