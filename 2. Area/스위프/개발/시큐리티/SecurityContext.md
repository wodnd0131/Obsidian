### 구조부터

```
SecurityContext
    └── Authentication
            ├── Principal        (누구인가)
            ├── Credentials      (인증 수단)
            ├── Authorities      (권한 목록)
            └── isAuthenticated  (인증 완료 여부)
```

실제 코드:

```java
// SecurityContext — 그냥 Authentication의 컨테이너
public interface SecurityContext {
    Authentication getAuthentication();
    void setAuthentication(Authentication authentication);
}

// Authentication — 실제 인증 정보
public interface Authentication {
    Object getPrincipal();          // 보통 UserDetails 객체
    Object getCredentials();        // 비밀번호 (인증 후엔 보통 null로 지움)
    Collection<? extends GrantedAuthority> getAuthorities(); // ["ROLE_USER", "ROLE_ADMIN"]
    boolean isAuthenticated();
}
```

세션에 저장되는 건 결국 이것이다:

```
{
  principal: UserDetails {
      username: "user@example.com",
      password: null,              // 인증 후 지워짐
      authorities: ["ROLE_USER"]
  },
  isAuthenticated: true
}
```

---

### "JWT 인증 시스템만 만들었는데 세션이 인증해준다"는 게 무슨 뜻인가

JWT 필터가 하는 일을 보면 이해된다.

```java
// JWT 필터 핵심 동작
public void doFilter(HttpServletRequest request, ...) {
    String token = extractToken(request);  // 헤더에서 JWT 꺼냄

    if (token != null && jwtProvider.validate(token)) {
        String username = jwtProvider.getUsername(token);
        UserDetails userDetails = userDetailsService.loadUser(username);

        // Authentication 객체 생성
        UsernamePasswordAuthenticationToken auth =
            new UsernamePasswordAuthenticationToken(
                userDetails, null, userDetails.getAuthorities()
            );

        // SecurityContext에 저장
        SecurityContextHolder.getContext().setAuthentication(auth);
    }

    chain.doFilter(request, response);
}
```

JWT 필터가 하는 일의 본질은 **"JWT를 검증해서 SecurityContext에 Authentication을 채우는 것"** 이다.

세션 방식은 이 과정을 JWT 없이 재현한다:

```
JWT 방식:
  JWT 파싱 → username 추출 → Authentication 생성 → SecurityContext에 저장

세션 방식:
  JSESSIONID 조회 → 세션에서 Authentication 꺼냄 → SecurityContext에 저장
```

**도착지가 같다.** SecurityContext에 Authentication이 채워지면, 이후 Spring Security는 그게 JWT에서 왔는지 세션에서 왔는지 구분하지 않는다.

```java
// FilterSecurityInterceptor — 최종 접근 제어
Authentication auth = SecurityContextHolder.getContext().getAuthentication();

if (auth != null && auth.isAuthenticated()) {
    // 통과 — 어디서 채워졌는지 상관없음
}
```

---

### 정리

JWT 인증 시스템을 만든다는 것은 "JWT → Authentication → SecurityContext" 경로를 구현한 것이다. 세션은 같은 SecurityContext를 다른 경로로 채우는 수단이다. `STATELESS` 설정이 없으면 두 경로가 공존하고, 세션 경로가 JWT 경로를 우회할 수 있다.