#SpringBoot/Tomcat 
# Catalina — Tomcat의 서블릿 컨테이너 모듈

## 1. Problem First

### Coyote가 넘겨준 요청, 그 다음은 누가 처리하는가

Coyote는 네트워크 바이트를 `HttpServletRequest` 객체로 만들어서 넘긴다. 
하지만 Coyote는 이 요청을 **어느 애플리케이션이, 어느 서블릿이 처리해야 하는지 모른다.**
```
Coyote: "요청 왔어. HttpServletRequest 만들었어. 받아."
           │
           │ 근데 Coyote는 모른다:
           │ - 이 요청이 어느 웹 애플리케이션 것인지
           │ - 어느 서블릿이 처리해야 하는지
           │ - Filter가 있는지
           │ - 서블릿 생명주기는 어떻게 관리하는지
           ▼
        Catalina가 이 모든 것을 안다
```

Tomcat 하나에 여러 웹 애플리케이션을 올릴 수 있다. `/app1`, `/app2` 처럼 컨텍스트가 다른 애플리케이션들이 공존한다. 어느 요청이 어느 애플리케이션으로 가야 하는지, 그 안에서 어느 서블릿이 처리해야 하는지 결정하는 것이 Catalina의 핵심 역할이다.

---

## 2. Mechanics

### 2-1. Catalina의 컴포넌트 계층 구조

Catalina는 **컨테이너 계층 구조**로 요청을 처리한다.

```
Server
└── Service
      ├── Connector (Coyote)     ← 네트워크 담당
      └── Engine                 ← Catalina 시작점
            └── Host             ← 가상 호스트 (도메인)
                  └── Context    ← 웹 애플리케이션 하나
                        └── Wrapper  ← 서블릿 하나
```

실제 Spring Boot 애플리케이션에 대입하면:

```
Server (Tomcat 인스턴스)
└── Service
      ├── Connector (포트 8080, Coyote)
      └── Engine ("Catalina")
            └── Host ("localhost")
                  └── Context ("/")          ← Spring Boot 앱 하나
                        └── Wrapper          ← DispatcherServlet
```

각 컴포넌트가 무엇을 결정하는지 내려간다.

---

### 2-2. Engine — 어느 Host로 보낼지 결정

Coyote의 Adapter로부터 요청을 받는 첫 번째 Catalina 컴포넌트다.

```
요청의 Host 헤더를 보고 어느 가상 호스트로 보낼지 결정

GET /users HTTP/1.1
Host: api.myservice.com    ← 이걸 보고

Engine: "api.myservice.com 담당 Host 컴포넌트로 보내"
```

Tomcat 하나로 여러 도메인을 서빙할 수 있는 **가상 호스팅(Virtual Hosting)** 이 여기서 구현된다.

---

### 2-3. Host — 어느 Context(웹 애플리케이션)로 보낼지 결정

```
요청 URL의 컨텍스트 경로를 보고 결정

GET /app1/users → Context("/app1") 으로
GET /app2/users → Context("/app2") 으로
GET /users      → Context("/") 으로     ← Spring Boot 기본
```

Spring Boot는 단일 애플리케이션이므로 Context가 하나(`/`)다. 하나만 있으니 무조건 그쪽으로 간다.

---

### 2-4. Context — 웹 애플리케이션 하나의 실행 환경

Context가 Catalina에서 가장 중요한 컴포넌트다.

**Context가 관리하는 것들:**

```
Context("/")
├── ServletContext          웹 애플리케이션 전역 저장소
├── ClassLoader             이 애플리케이션 전용 클래스로더
├── Filter 목록             등록된 Filter들
├── Wrapper 목록            등록된 서블릿들
├── ServletContextListener  앱 시작/종료 이벤트 리스너
└── Session Manager         HTTP 세션 관리
```

**ClassLoader를 Context마다 분리하는 이유:**

```
Tomcat에 두 앱이 올라간 경우:

Context("/app1") → ClassLoader A → jackson-2.13.jar
Context("/app2") → ClassLoader B → jackson-2.15.jar

같은 라이브러리 다른 버전을 동시에 사용 가능
클래스로더가 분리돼 있으므로 충돌 없음
```

Spring Boot 단일 앱에서는 이 장점이 잘 안 보이지만, 
전통적인 WAS 환경에서 하나의 Tomcat에 여러 war를 배포할 때 핵심 기능이다.
#### 스프링부트 단일 앱에선 눈에 띄지 않는다.
##### Host — 도메인 단위 분리

Tomcat 하나가 포트 하나(8080)를 열고 있다. 그런데 아래 두 도메인이 같은 Tomcat으로 들어온다고 가정하자.

```
api.myservice.com   → 같은 Tomcat 인스턴스
admin.myservice.com → 같은 Tomcat 인스턴스
```

어떻게 같은 Tomcat으로 들어오는가? DNS가 두 도메인을 같은 서버 IP로 연결하기 때문이다.

```
api.myservice.com   → DNS → 123.456.789.0:8080 → Tomcat
admin.myservice.com → DNS → 123.456.789.0:8080 → Tomcat
```

Tomcat 입장에서 두 요청은 동일한 포트로 들어온다. 구분하는 방법이 **HTTP Host 헤더**다.

```
요청 A:
GET /users HTTP/1.1
Host: api.myservice.com      ← "나는 api 도메인으로 온 요청"

요청 B:
GET /dashboard HTTP/1.1
Host: admin.myservice.com    ← "나는 admin 도메인으로 온 요청"
```

Engine이 이 Host 헤더를 보고 분기한다.

```
Engine
├── Host("api.myservice.com")    → 요청 A 처리
└── Host("admin.myservice.com")  → 요청 B 처리
```

각 Host는 **완전히 독립된 웹 애플리케이션 영역**이다. api와 admin이 서로를 모른다.

---

##### Context — 같은 도메인 안에서 애플리케이션 단위 분리

Host 안에서 URL 경로로 다시 분기한다.

같은 `api.myservice.com` 도메인이지만 두 개의 독립적인 앱이 올라가 있다고 가정하자.

```
api.myservice.com/v1/users  → 구버전 API 앱
api.myservice.com/v2/users  → 신버전 API 앱
```

```
Host("api.myservice.com")
├── Context("/v1")   → 구버전 앱 (v1.war)
└── Context("/v2")   → 신버전 앱 (v2.war)
```

URL의 첫 번째 경로 세그먼트가 Context 경로와 매칭된다.

```
GET /v1/users → Context("/v1") 으로
GET /v2/users → Context("/v2") 으로
```

각 Context는 **독립된 ClassLoader, 독립된 Spring ApplicationContext**를 가진다. 같은 라이브러리를 다른 버전으로 각각 쓸 수 있다.

---

##### Spring Boot에서는 왜 다 하나인가

Spring Boot는 "Tomcat 하나 = 앱 하나" 를 전제로 설계됐다.

```
Host가 하나:
→ 도메인 여러 개를 하나의 Tomcat으로 서빙할 일이 없음
→ 도메인 분기는 앞단 로드밸런서(ALB, Nginx)가 함

Context가 하나("/"):
→ 앱이 하나니까 경로 분기 필요 없음
→ 모든 URL이 DispatcherServlet으로 감
```

그래서 Spring Boot 앱에서 Host, Context 계층이 보이지 않는 것이다. 있긴 한데 각각 하나씩이라 분기가 일어나지 않는다.

---

##### 한 번에 정리

```
실제 서버 하나에 이런 구조가 가능하다:

Tomcat
└── Engine
      ├── Host("api.myservice.com")
      │     ├── Context("/v1")  → 구버전 API
      │     └── Context("/v2")  → 신버전 API
      │
      └── Host("admin.myservice.com")
            └── Context("/")    → 관리자 앱


요청: GET /v1/users HTTP/1.1
      Host: api.myservice.com

Engine:  Host 헤더 보고 → api.myservice.com Host 선택
Host:    URL 경로 보고  → /v1 Context 선택
Context: 이 앱의 Filter, Servlet으로 처리
```

```
Host    = 도메인 단위 분리  (누구에게 온 요청인가)
Context = 앱 단위 분리      (어느 앱이 처리할 것인가)
```

Spring Boot에서 안 보이는 이유는 둘 다 하나씩이라 분기가 발생하지 않아서다.

---


### 2-5. Pipeline과 Valve — Catalina의 처리 체인

각 컨테이너(Engine, Host, Context, Wrapper)는 **Pipeline**을 가진다. Pipeline 안에 **Valve**들이 체인으로 연결된다.

```
Context.Pipeline:
[AccessLogValve] → [AuthenticatorValve] → [StandardContextValve]
                                                    │
                                                    ▼
                                          Wrapper.Pipeline:
                                          [StandardWrapperValve]
                                                    │
                                                    ▼
                                          Filter Chain 실행
                                                    │
                                                    ▼
                                          Servlet.service() 호출
```

Valve는 서블릿 스펙의 Filter와 비슷해 보이지만 다르다.

```
Valve:   Tomcat 내부 개념 (Catalina 레벨)
         컨테이너 간 이동 경로에 끼어들 수 있음
         Tomcat 설정으로 추가

Filter:  서블릿 스펙 (Jakarta EE)
         서블릿 진입 직전에만 동작
         코드로 등록 (@WebFilter, FilterRegistrationBean)
```

실무에서 Valve를 직접 다루는 경우는 드물다. Tomcat 레벨 접근 로그(`AccessLogValve`)가 대표적 사용례다.

---

### 2-6. Wrapper — 서블릿 하나의 관리자

Wrapper는 서블릿 인스턴스 하나를 감싸서 생명주기를 관리한다.

```java
// Wrapper가 하는 일 (소스코드 기반)

// 1. 서블릿 인스턴스 생성 (최초 요청 시 또는 load-on-startup)
Servlet servlet = (Servlet) instanceManager.newInstance(servletClass);

// 2. 초기화
servlet.init(servletConfig);

// 3. Filter Chain 구성 후 서블릿 실행
ApplicationFilterChain filterChain = createFilterChain(request, servlet);
filterChain.doFilter(request, response);
// Filter Chain 끝에서 servlet.service() 호출

// 4. 종료 시
servlet.destroy();
```

**Filter Chain을 Wrapper가 만드는 것이 중요하다.**

```
Wrapper가 요청마다 하는 일:

1. 이 요청에 적용할 Filter 목록 결정
   (URL 패턴, 서블릿 이름 매핑 기준)

2. ApplicationFilterChain 생성
   [Filter1 → Filter2 → Filter3 → Servlet]

3. filterChain.doFilter() 실행
   → 체인 끝에서 DispatcherServlet.service() 호출
```

개발자가 `@Component`로 등록한 Filter가 어떻게 DispatcherServlet 앞에 끼어드는지, 이 구조에서 이루어진다.

---

### 2-7. 전체 요청 처리 흐름 — Coyote부터 서블릿까지

```
[Coyote]
바이트 파싱 → HttpServletRequest 생성
    │
    ▼ CoyoteAdapter.service()
[Engine.Pipeline]
Host 헤더 보고 → Host 결정
    │
    ▼
[Host.Pipeline]
URL 컨텍스트 경로 보고 → Context 결정
    │
    ▼
[Context.Pipeline]
AccessLogValve 등 통과
    │
    ▼
[Wrapper.Pipeline — StandardWrapperValve]
Filter Chain 구성
    │
    ▼
[Filter Chain]
Filter1.doFilter()
    → Filter2.doFilter()
        → Filter3.doFilter()  (Spring Security 등)
            → chain.doFilter() 마지막 호출
                │
                ▼
        [DispatcherServlet.service()]
        Spring MVC 처리 시작
```

---

### 2-8. ServletContext — Context가 관리하는 전역 저장소

`ServletContext`는 웹 애플리케이션 전체에서 공유되는 저장소다.

```java
// 애플리케이션 전역에서 접근 가능
@Autowired
ServletContext servletContext;

// 전역 속성 저장
servletContext.setAttribute("appVersion", "1.0.0");

// 실제 파일 경로 조회
String realPath = servletContext.getRealPath("/WEB-INF/config");
```

Spring의 `ApplicationContext`와 혼동하기 쉽지만 다르다.

```
ServletContext:    Tomcat(Catalina)이 만들고 관리
                   서블릿 스펙 표준
                   웹 애플리케이션 하나당 하나

ApplicationContext: Spring이 만들고 관리
                    Spring Bean 컨테이너
                    DispatcherServlet.init()에서 생성
```

Spring은 `ApplicationContext`를 `ServletContext`에 저장해서 연결한다.

```java
// Spring 내부 — FrameworkServlet.initWebApplicationContext()
// ApplicationContext를 ServletContext에 저장
servletContext.setAttribute(
    WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE,
    applicationContext
);
// 이로써 Tomcat과 Spring이 연결됨
```

---

## 3. 공식 근거

|주장|출처|
|---|---|
|서블릿 생명주기 (init, service, destroy)|Jakarta Servlet Specification 6.0, Section 2.3|
|Filter Chain 동작 방식|Jakarta Servlet Specification 6.0, Section 6.2|
|ServletContext 정의|Jakarta Servlet Specification 6.0, Section 4|
|Tomcat Pipeline/Valve 구조|Apache Tomcat 공식 문서 — _Valve Component_|
|ClassLoader 계층 구조|Apache Tomcat 공식 문서 — _Class Loader HOW-TO_|

---

## 4. 이 설계를 이렇게 한 이유

### 계층 구조(Engine → Host → Context → Wrapper)를 둔 이유

```
각 계층이 자신의 책임만 가진다

Engine:  "어느 도메인인가"           만 결정
Host:    "어느 애플리케이션인가"      만 결정
Context: "이 앱의 실행 환경 제공"     만 담당
Wrapper: "이 서블릿의 생명주기 관리"  만 담당

→ 도메인 추가 = Host 하나 추가
→ 앱 추가 = Context 하나 추가
→ 서블릿 추가 = Wrapper 하나 추가
계층 간 영향 없음
```

### 트레이드오프

|손실|구체적 상황|
|---|---|
|**단일 앱에서 과한 구조**|Spring Boot 단일 앱은 Context 하나, Wrapper 하나다. Engine→Host→Context→Wrapper 4계층을 거치는 건 단순 앱에선 오버헤드|
|**서블릿 싱글턴**|Wrapper는 서블릿 인스턴스를 하나만 유지한다. 모든 요청이 같은 DispatcherServlet 인스턴스를 공유한다. 인스턴스 변수에 상태를 저장하면 동시성 문제 발생|
|**Session Manager 메모리**|Context의 Session Manager가 HTTP 세션을 메모리에 관리한다. 세션이 많으면 메모리 압박. 외부 세션 저장소(Redis)로 대체하는 이유|

---

## 5. 이어지는 개념

| 순서    | 개념                               | 이유                                                                                                                    |
| ----- | -------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| **1** | **서블릿 싱글턴과 동시성**                 | Wrapper가 서블릿을 하나만 유지한다는 것의 실무 의미. DispatcherServlet, Controller가 왜 싱글턴이고 상태를 가지면 안 되는지                                |
| **2** | **Spring Boot 내장 Tomcat 초기화 순서** | `SpringApplication.run()` 이후 Tomcat이 어떻게 구성되는지. `EmbeddedWebApplicationContext`, `TomcatServletWebServerFactory` 의 역할 |
| **3** | **Netty와 Tomcat 비교**             | Catalina의 스레드 기반 처리 모델 한계를 Netty(WebFlux)가 어떻게 다르게 설계했는지                                                              |

---

아까 Catalina 설명에서 나온 것들이다. 다시 맥락을 잡고 들어간다.

---

## 왜 Pipeline/Valve가 필요한가

Catalina의 컨테이너 계층을 떠올려보자.

```
Engine → Host → Context → Wrapper → 서블릿
```

요청이 이 계층을 타고 내려간다. 그런데 각 계층을 **통과할 때 뭔가 처리를 끼워넣고 싶으면** 어떻게 하는가.

예를 들어:

- 모든 요청의 접근 로그를 남기고 싶다
- 특정 IP를 차단하고 싶다
- 요청 처리 시간을 측정하고 싶다

이걸 서블릿 코드나 Filter에 넣을 수도 있지만, **Tomcat 레벨에서 처리하고 싶은 경우**가 있다. 그때 쓰는 것이 Pipeline/Valve다.

---

## Pipeline과 Valve의 구조

**Pipeline** = 처리 단계들을 연결한 체인 **Valve** = 그 체인 안의 처리 단계 하나

```
Pipeline:
[Valve1] → [Valve2] → [Valve3] → [StandardValve]
                                        ↑
                                  마지막 Valve는 항상 존재
                                  다음 계층으로 넘기는 역할
```

서블릿 스펙의 Filter Chain과 구조가 거의 같다.

```
Filter Chain (서블릿 스펙):
[Filter1] → [Filter2] → [Servlet]

Pipeline/Valve (Tomcat 내부):
[Valve1] → [Valve2] → [StandardValve → 다음 계층]
```

---

## 계층마다 Pipeline이 있다

```
Engine.Pipeline
└── StandardEngineValve (마지막 — Host로 넘김)

Host.Pipeline
└── StandardHostValve (마지막 — Context로 넘김)

Context.Pipeline
├── AccessLogValve      ← 접근 로그 기록
└── StandardContextValve (마지막 — Wrapper로 넘김)

Wrapper.Pipeline
└── StandardWrapperValve (마지막 — Filter Chain → 서블릿 호출)
```

요청이 흐르는 전체 경로:

```
Engine.Pipeline 통과
    → Host.Pipeline 통과
        → Context.Pipeline 통과
            → Wrapper.Pipeline 통과
                → Filter Chain
                    → DispatcherServlet
```

---

## 실제로 어디서 보이는가

Spring Boot 기본 설정에 AccessLogValve가 있다.

```yaml
# application.yml
server:
  tomcat:
    accesslog:
      enabled: true
      pattern: "%h %s %{User-Agent}i %D ms"
```

이게 Context.Pipeline에 `AccessLogValve`를 추가하는 것이다.

```
모든 요청이 Context.Pipeline을 지나므로
AccessLogValve가 모든 요청/응답을 기록할 수 있음
```

---

## Filter와 Valve의 차이

```
                Valve                   Filter
위치            Tomcat 내부             서블릿 스펙
설정 방법       Tomcat 설정             코드 (@WebFilter 등)
Spring Bean     사용 불가               사용 가능 (DelegatingFilterProxy로)
접근 범위       Engine/Host까지 가능    Context(앱) 레벨만
주요 용도       Tomcat 레벨 로깅, IP 차단   인증, 인코딩, CORS
```

핵심 차이는 **범위**다.

```
Valve는 Host 레벨에 달 수 있다:
→ 이 도메인으로 오는 모든 요청 (앱 여러 개 포함)

Filter는 Context 레벨만:
→ 이 앱으로 오는 요청만
```

Spring Boot 단일 앱에서는 이 차이가 의미없다. Context가 하나뿐이라서. 그래서 실무에서 Valve를 직접 다룰 일이 거의 없고, 대부분 Filter로 해결한다.

---

## 한 줄 정리

```
Pipeline = 각 Catalina 계층에 있는 처리 체인
Valve    = 그 체인 안의 처리 단계 하나
           (서블릿의 Filter와 같은 구조, Tomcat 내부 레벨)

Spring Boot 단일 앱에서는:
→ 거의 신경 안 써도 됨
→ AccessLogValve 정도만 설정으로 쓰는 수준
→ 나머지는 Filter/Interceptor로 해결
```