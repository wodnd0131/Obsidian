#SpringBoot/Tomcat 
```
소켓 (TCP 연결)
    정의 주체: POSIX 표준 / RFC 793 (TCP)
    Java 구현: java.net.Socket, java.net.ServerSocket (JDK)
    Tomcat:    이 JDK API를 사용해서 포트를 열고 연결을 수락

스레드
    정의 주체: POSIX Threads / OS 스케줄러
    Java 구현: java.lang.Thread (JDK)
    Tomcat:    ThreadPoolExecutor로 스레드 풀 구성 (직접 구현)

HTTP 파싱
    정의 주체: RFC 9110 (HTTP Semantics)
               RFC 9112 (HTTP/1.1 메시지 형식)
    Tomcat:    Coyote 모듈이 직접 구현
               "GET /users HTTP/1.1\r\nHost: ..." 를 파싱해서
               HttpServletRequest 객체로 만드는 것

HttpServletRequest / HttpServletResponse
    정의 주체: 서블릿 스펙 (Jakarta EE, 구 Java EE)
    Tomcat:    이 인터페이스를 구현한 객체를 생성해서 서블릿에 전달
```

---

### Tomcat의 내부 구조로 보면

Tomcat은 이 역할들을 모듈로 분리한다.

```
Tomcat
├── Coyote    (커넥터 모듈)
│   소켓 열기, TCP 연결 수락
│   HTTP 파싱 → HttpServletRequest 생성
│   스레드 풀 관리
│   → 서블릿 스펙과 무관한 영역
│
└── Catalina  (서블릿 컨테이너 모듈)
    서블릿 생명주기 관리 (init, service, destroy)
    Filter Chain 실행
    → 서블릿 스펙을 구현하는 영역
```

```
HTTP 요청
    │
    ▼
Coyote가 받음        ← RFC 793(TCP) + RFC 9112(HTTP) 구현
    │  소켓 → 파싱 → HttpServletRequest 생성
    ▼
Catalina로 넘김      ← 서블릿 스펙(Jakarta EE) 구현
    │  Filter → Servlet.service() 호출
    ▼
DispatcherServlet    ← Spring MVC 구현
```

---

### 한 줄 정리

```
소켓/스레드  → OS + JDK가 정의, Tomcat(Coyote)이 사용
HTTP 파싱   → RFC가 정의, Tomcat(Coyote)이 구현
서블릿 생명주기 → 서블릿 스펙(Jakarta EE)이 정의, Tomcat(Catalina)이 구현
Spring MVC  → 서블릿 스펙 위에서 Spring이 구현

각 레이어가 자신의 정의 주체를 가지고 있고
Tomcat이 이것들을 조합해서 하나의 런타임으로 만든다
```