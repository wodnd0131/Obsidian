## 1. Problem First

### `java -jar app.jar` 한 줄로 서버가 뜬다 — 실제로 무슨 일이 일어나는가

Spring Boot 이전에는 이 과정이 수동이었다.

```
1. Tomcat 별도 설치
2. war 파일 빌드
3. Tomcat의 webapps/ 디렉토리에 war 복사
4. web.xml에 DispatcherServlet 등록
5. Tomcat 시작 스크립트 실행
```

Spring Boot는 이것을 `java -jar app.jar` 한 줄로 압축했다. 어떻게 가능한가.

핵심은 두 가지다.

```
1. Tomcat이 jar 안에 내장되어 있다
2. 시작 순서가 코드로 자동화되어 있다
```

이 자동화의 내부를 순서대로 따라간다.

---

## 2. Mechanics

### 전체 초기화 순서 개요

```
java -jar app.jar
    │
    ▼
① main() → SpringApplication.run()
    │
    ▼
② SpringApplication 준비
   (ApplicationContext 타입 결정)
    │
    ▼
③ ApplicationContext 생성
   (AnnotationConfigServletWebServerApplicationContext)
    │
    ▼
④ Bean 등록 및 초기화
   (ComponentScan, AutoConfiguration)
    │
    ▼
⑤ TomcatServletWebServerFactory → Tomcat 인스턴스 생성
    │
    ▼
⑥ Tomcat 내부 구성
   (Connector, Engine, Host, Context)
    │
    ▼
⑦ DispatcherServlet 등록
    │
    ▼
⑧ Tomcat.start() → 포트 Listen 시작
    │
    ▼
⑨ ApplicationContext refresh 완료
   → 요청 받을 준비 완료
```

각 단계를 내려간다.

---

### ① main() → SpringApplication.run()

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

`@SpringBootApplication`은 세 가지 애너테이션의 합성이다.

```java
@SpringBootConfiguration   // @Configuration과 동일
@EnableAutoConfiguration   // AutoConfiguration 활성화 ← 핵심
@ComponentScan             // 현재 패키지부터 Bean 스캔
```

`SpringApplication.run()`이 호출되면 `SpringApplication` 인스턴스를 만들고 `run()` 메서드를 실행한다.

---

### ② SpringApplication 준비 — 어떤 ApplicationContext를 만들지 결정

```java
// SpringApplication 생성자 내부
public SpringApplication(Class<?>... primarySources) {
    // 클래스패스를 보고 웹 환경 타입 결정
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
}
```

```
클래스패스에 무엇이 있는가?

spring-webmvc + Servlet API 있음
    → WebApplicationType.SERVLET
    → 생성할 ApplicationContext: AnnotationConfigServletWebServerApplicationContext

spring-webflux 있음
    → WebApplicationType.REACTIVE
    → 생성할 ApplicationContext: AnnotationConfigReactiveWebServerApplicationContext

둘 다 없음
    → WebApplicationType.NONE
    → 생성할 ApplicationContext: AnnotationConfigApplicationContext
```

우리는 Spring MVC 경로를 따라간다.

---

### ③ ApplicationContext 생성

```java
// SpringApplication.run() 내부
ConfigurableApplicationContext context = createApplicationContext();
// → AnnotationConfigServletWebServerApplicationContext 생성
```

`AnnotationConfigServletWebServerApplicationContext`는 일반 `ApplicationContext`에서 두 가지가 추가된 것이다.

```
ApplicationContext (기본)
    Bean 등록, DI, AOP
        │
        ▼
ServletWebServerApplicationContext (추가)
    + 내장 웹 서버(Tomcat) 생성/시작 능력
    + DispatcherServlet 등록 능력
        │
        ▼
AnnotationConfigServletWebServerApplicationContext (추가)
    + @Configuration, @ComponentScan 처리 능력
```

---

### ④ Bean 등록 — ComponentScan + AutoConfiguration

`context.refresh()` 가 호출되면서 Bean 등록이 시작된다.

```
refresh() 내부에서 일어나는 일:

1. ComponentScan
   @SpringBootApplication이 있는 패키지부터 스캔
   @Controller, @Service, @Repository, @Component 등록

2. AutoConfiguration 처리
   @EnableAutoConfiguration이 트리거
   META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
   파일에서 자동 설정 클래스 목록 로드

   주요 AutoConfiguration:
   DispatcherServletAutoConfiguration  → DispatcherServlet Bean 등록
   TomcatServletWebServerFactory      → Tomcat 설정
   WebMvcAutoConfiguration            → HandlerMapping 등 MVC 컴포넌트 등록
   SecurityAutoConfiguration          → Spring Security 설정 (의존성 있을 때)
```

이 중 `TomcatServletWebServerFactory`가 핵심이다.

```java
// TomcatServletWebServerFactory AutoConfiguration
@Bean
@ConditionalOnMissingBean
public TomcatServletWebServerFactory tomcatServletWebServerFactory() {
    return new TomcatServletWebServerFactory();
    // 개발자가 직접 등록하지 않았을 때만 생성
}
```

---

### ⑤⑥ TomcatServletWebServerFactory → Tomcat 구성

`ServletWebServerApplicationContext.onRefresh()` 에서 웹 서버 생성이 시작된다.

```java
// ServletWebServerApplicationContext.onRefresh() 내부
protected void onRefresh() {
    super.onRefresh();
    createWebServer(); // 여기서 Tomcat 생성
}

private void createWebServer() {
    ServletWebServerFactory factory = getWebServerFactory();
    // → TomcatServletWebServerFactory

    this.webServer = factory.getWebServer(getSelfInitializer());
    // → Tomcat 인스턴스 생성 + 구성
}
```

`TomcatServletWebServerFactory.getWebServer()` 내부:

```java
public WebServer getWebServer(ServletContextInitializer... initializers) {

    // 1. Tomcat 인스턴스 생성
    Tomcat tomcat = new Tomcat();

    // 2. 임시 작업 디렉토리 설정
    File baseDir = createTempDir("tomcat");
    tomcat.setBaseDir(baseDir.getAbsolutePath());

    // 3. Connector(Coyote) 구성
    Connector connector = new Connector(this.protocol);
    // protocol 기본값: "org.apache.coyote.http11.Http11NioProtocol"
    connector.setPort(8080);
    tomcat.getService().addConnector(connector);

    // 4. Engine, Host 구성
    tomcat.getHost().setAutoDeploy(false);

    // 5. Context 구성
    prepareContext(tomcat.getHost(), initializers);

    // 6. TomcatWebServer 래핑해서 반환 (아직 start() 안 함)
    return getTomcatWebServer(tomcat);
}
```

`prepareContext()` 에서 Context를 구성한다.

```java
protected void prepareContext(Host host,
                              ServletContextInitializer[] initializers) {
    // TomcatEmbeddedWebappClassLoader 생성 (앱 전용 클래스로더)
    TomcatEmbeddedWebappClassLoader classLoader = ...;

    // Context 생성
    TomcatEmbeddedWebappContext context = new TomcatEmbeddedWebappContext();
    context.setPath("/");              // 컨텍스트 경로
    context.setClassLoader(classLoader);

    // ServletContextInitializer 등록
    // → 이후 Tomcat 시작 시 DispatcherServlet 등록에 사용됨
    context.addServletContainerInitializer(
        new ServletContainerInitializerAdapter(initializers), ...);

    host.addChild(context);
}
```

---

### ⑦ DispatcherServlet 등록

`ServletContextInitializer`가 실행되면서 DispatcherServlet이 Context에 등록된다.

```java
// DispatcherServletAutoConfiguration 내부
@Bean
public DispatcherServlet dispatcherServlet() {
    return new DispatcherServlet();
}

@Bean
public DispatcherServletRegistrationBean dispatcherServletRegistration(
        DispatcherServlet dispatcherServlet) {

    DispatcherServletRegistrationBean registration =
        new DispatcherServletRegistrationBean(dispatcherServlet, "/");
    // "/" 경로 → 모든 요청을 DispatcherServlet으로

    registration.setLoadOnStartup(1);
    // Tomcat 시작 시 즉시 init() 호출
    // (요청이 올 때까지 미루지 않음)

    return registration;
}
```

`setLoadOnStartup(1)` 이 중요하다.

```
loadOnStartup = -1 (기본):
    첫 번째 요청이 올 때 서블릿 init() 호출
    → 첫 요청이 느려짐 (Cold Start)

loadOnStartup = 1:
    Tomcat 시작 시 즉시 init() 호출
    → ApplicationContext 초기화, HandlerMapping 스캔이 시작 시 완료
    → 첫 요청도 빠름
    → Spring Boot 기본값
```

---

### ⑧ Tomcat.start() — 포트 Listen 시작

```java
// TomcatWebServer.start() 내부
public void start() throws WebServerException {
    this.tomcat.start();
    // → Connector(Coyote) 시작
    // → 포트 8080 바인딩
    // → Acceptor Thread 시작
    // → Worker Thread Pool 초기화

    // 이 시점부터 실제로 요청을 받을 수 있음
}
```

이때 DispatcherServlet의 `init()`이 호출된다. (`loadOnStartup=1` 이므로)

```java
// DispatcherServlet.init() → FrameworkServlet.initWebApplicationContext()
protected WebApplicationContext initWebApplicationContext() {

    // 이미 생성된 ApplicationContext를 찾아서 연결
    WebApplicationContext rootContext =
        WebApplicationContextUtils.getWebApplicationContext(getServletContext());

    // DispatcherServlet 전용 자식 ApplicationContext 생성 (경우에 따라)
    // Spring Boot에서는 보통 rootContext를 그대로 씀

    // MVC 컴포넌트 초기화
    onRefresh(wac);
    // → initHandlerMappings()
    // → initHandlerAdapters()
    // → initViewResolvers()
    // → ...

    return wac;
}
```

---

### ⑨ 완료 — 로그로 확인

```
2024-04-27 10:00:00 INFO  o.s.b.w.e.tomcat.TomcatWebServer  - Tomcat initialized with port(s): 8080 (http)
2024-04-27 10:00:00 INFO  o.a.catalina.core.StandardService  - Starting service [Tomcat]
2024-04-27 10:00:00 INFO  o.a.catalina.core.StandardEngine   - Starting Servlet engine: [Apache Tomcat/10.1.x]
2024-04-27 10:00:00 INFO  o.s.b.w.e.tomcat.TomcatWebServer  - Tomcat started on port(s): 8080 (http) with context path ''
2024-04-27 10:00:00 INFO  c.e.Application                   - Started Application in 2.345 seconds (JVM running for 2.678)
```

각 로그가 위 단계 중 어디에 해당하는지 보인다.

---

### 전체 흐름 한 번에

```
main()
    │
    ▼
SpringApplication.run()
    │
    ├─ WebApplicationType 결정 (SERVLET)
    │
    ├─ AnnotationConfigServletWebServerApplicationContext 생성
    │
    ├─ context.refresh()
    │       │
    │       ├─ ComponentScan → @Controller, @Service 등 Bean 등록
    │       │
    │       ├─ AutoConfiguration
    │       │       ├─ TomcatServletWebServerFactory Bean 등록
    │       │       ├─ DispatcherServlet Bean 등록
    │       │       └─ HandlerMapping 등 MVC 컴포넌트 Bean 등록
    │       │
    │       └─ onRefresh() → createWebServer()
    │               │
    │               ├─ new Tomcat()
    │               ├─ Connector(Coyote) 구성 (포트 8080)
    │               ├─ Engine, Host 구성
    │               ├─ Context("/") 구성
    │               └─ DispatcherServlet → Context에 Wrapper로 등록
    │
    ├─ TomcatWebServer.start()
    │       ├─ 포트 8080 바인딩
    │       ├─ Acceptor Thread 시작
    │       ├─ Worker Thread Pool 초기화
    │       └─ DispatcherServlet.init() 호출
    │               └─ HandlerMapping 스캔, MVC 컴포넌트 초기화
    │
    └─ "Started Application in 2.345 seconds"
       요청 받을 준비 완료
```

---

## 3. 공식 근거

|주장|출처|
|---|---|
|`@SpringBootApplication` 합성 애너테이션 구성|Spring Boot 공식 레퍼런스 — _Using the @SpringBootApplication Annotation_|
|AutoConfiguration 로딩 메커니즘|Spring Boot 공식 레퍼런스 — _Auto-configuration, META-INF/spring/...AutoConfiguration.imports_|
|`ServletWebServerApplicationContext.onRefresh()`|Spring Boot 소스코드 — `ServletWebServerApplicationContext.java`|
|`loadOnStartup` 동작|Jakarta Servlet Specification 6.0, Section 2.3.1|
|`TomcatServletWebServerFactory`|Spring Boot 공식 레퍼런스 — _Embedded Servlet Container Support_|

---

## 4. 이 설계를 이렇게 한 이유

### Spring Boot가 이 자동화로 얻는 것

- 개발자가 Tomcat 설치, web.xml 작성, war 배포 과정을 몰라도 된다
- `application.yml` 설정만으로 Tomcat 동작을 제어할 수 있다
- 웹 서버를 교체할 수 있다 (Tomcat → Jetty → Undertow, 의존성 교체만으로)

### 트레이드오프

|손실|구체적 상황|
|---|---|
|**시작 시간**|ComponentScan, AutoConfiguration, HandlerMapping 스캔이 모두 시작 시 발생. 빈이 많을수록 시작이 느려진다. Lambda 같은 환경에서 Cold Start 문제의 원인|
|**자동화의 블랙박스**|문제가 생겼을 때 어느 AutoConfiguration이 어떤 Bean을 등록했는지 추적하기 어렵다. `--debug` 플래그로 AutoConfiguration 리포트를 출력해 확인해야 한다|
|**외부 Tomcat 배포 시 충돌**|내장 Tomcat과 외부 Tomcat이 충돌한다. 외부 Tomcat에 배포하려면 `spring-boot-starter-tomcat`을 `provided`로 변경하고 `SpringBootServletInitializer`를 상속해야 한다|

---

## 5. 이어지는 개념

|순서|개념|이유|
|---|---|---|
|**1**|**Spring Bean 생명주기**|`context.refresh()` 안에서 Bean이 어떻게 생성되고 의존성이 주입되는지. 초기화 순서를 알면 `@PostConstruct`, `ApplicationListener` 등의 실행 시점이 보인다|
|**2**|**GraalVM Native Image + Spring AOT**|시작 시간 문제의 해결책. 빌드 시점에 Bean 스캔/AutoConfiguration을 미리 처리해서 런타임 시작 시간을 수십 ms로 줄이는 접근|
|**3**|**WebFlux + Netty 초기화 순서**|같은 `SpringApplication.run()`이지만 `WebApplicationType.REACTIVE`로 분기되어 Tomcat 대신 Netty가 뜨는 경로. 비교하면 각 단계의 역할이 더 명확히 보인다|