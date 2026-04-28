#Springboot
## 한 줄 핵심

> **Spring은 프레임워크, Spring Boot는 Spring을 편하게 쓰기 위한 도구다.**

Spring Boot가 Spring을 대체하는 게 아니다. Spring Boot 안에 Spring이 있다.

---

## Spring이 해결한 것

Java EE(당시 J2EE) 시대의 문제는 **객체 간 의존성을 개발자가 직접 관리**해야 했다는 것이다.

```java
// Spring 이전
public class OrderService {
    // 직접 생성 — 강한 결합
    private UserRepository userRepository = new UserRepositoryImpl();
    private PaymentService paymentService = new PaymentServiceImpl();
}

// Spring의 DI 컨테이너
public class OrderService {
    private final UserRepository userRepository;
    private final PaymentService paymentService;

    // 생성자 주입 — Spring이 알아서 넣어줌
    public OrderService(UserRepository userRepository,
                        PaymentService paymentService) {
        this.userRepository = userRepository;
        this.paymentService = paymentService;
    }
}
```

Spring의 핵심 가치는 **IoC(제어의 역전) / DI(의존성 주입)** 다. 객체 생성과 의존성 연결을 개발자가 아닌 프레임워크(IoC 컨테이너)가 담당한다.

여기에 AOP, 트랜잭션 관리, Spring MVC 등이 더해진 것이 Spring Framework다.

---

## 그런데 Spring 자체는 설정이 복잡했다

Spring Framework만 쓰던 시절:

```xml
<!-- web.xml — DispatcherServlet 등록 -->
<servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>
        org.springframework.web.servlet.DispatcherServlet
    </servlet-class>
</servlet>

<!-- applicationContext.xml — Bean 설정 -->
<bean id="dataSource"
      class="org.apache.commons.dbcp.BasicDataSource">
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql://localhost/mydb"/>
    ...
</bean>

<bean id="transactionManager"
      class="org.springframework.orm.jpa.JpaTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>
```

프로젝트 시작할 때마다 이 설정들을 반복해서 작성해야 했다. 라이브러리 버전 충돌도 개발자가 직접 해결해야 했다.

---

## Spring Boot가 해결한 것 — 세 가지

### 1. Auto Configuration — 설정 자동화

```java
// Spring Boot에서는 이것으로 끝
@SpringBootApplication  // 이 애너테이션 하나가 전부
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

`spring-boot-autoconfigure` 모듈이 클래스패스에 어떤 라이브러리가 있는지 감지하고, 적절한 Bean을 자동으로 등록한다.

```
classpath에 spring-webmvc가 있다
    → DispatcherServlet 자동 등록

classpath에 spring-data-jpa + DB 드라이버가 있다
    → DataSource, EntityManagerFactory, TransactionManager 자동 설정

classpath에 spring-security가 있다
    → 기본 Security 설정 자동 적용
```

내부적으로 `@Conditional` 애너테이션 기반으로 동작한다.

```java
// Spring Boot 내부 AutoConfiguration 예시
@Configuration
@ConditionalOnClass(DataSource.class)      // DataSource 클래스가 있을 때만
@ConditionalOnMissingBean(DataSource.class) // 개발자가 직접 등록 안 했을 때만
public class DataSourceAutoConfiguration {
    @Bean
    public DataSource dataSource() {
        // HikariCP DataSource 자동 생성
    }
}
```

개발자가 직접 Bean을 등록하면 Auto Configuration은 물러난다. **"규약이 있으면 따르고, 없으면 자동"** 이 원칙이다.

### 2. Starter 의존성 — 버전 충돌 해결

```gradle
// Spring Boot 이전 — 모든 버전을 직접 맞춰야 함
implementation 'org.springframework:spring-webmvc:5.3.20'
implementation 'com.fasterxml.jackson.core:jackson-databind:2.13.3'
implementation 'org.hibernate:hibernate-core:5.6.9.Final'
// 이 셋이 서로 호환되는 버전인지 개발자가 확인해야 함

// Spring Boot — Starter 하나로 호환 보장된 버전 묶음
implementation 'org.springframework.boot:spring-boot-starter-web'
// spring-webmvc + jackson + tomcat + 기타 의존성을
// 검증된 버전 조합으로 한번에 가져옴
```

`spring-boot-dependencies` BOM(Bill of Materials)이 모든 라이브러리 버전을 중앙 관리한다.

### 3. 내장 서버 — 별도 WAS 배포 불필요

```
Spring 시절:
코드 작성 → war 빌드 → Tomcat 설치 → webapps에 war 배포 → Tomcat 시작

Spring Boot:
코드 작성 → jar 빌드 → java -jar app.jar
            Tomcat이 jar 안에 내장되어 있음
```

---

## 관계 정리

```
Spring Boot
    │
    ├── Spring Framework (IoC/DI, AOP, MVC...)  ← 핵심 엔진
    ├── Auto Configuration                       ← Boot가 추가한 것
    ├── Starter 의존성 관리                       ← Boot가 추가한 것
    └── 내장 Tomcat                              ← Boot가 추가한 것
```

Spring Boot는 Spring Framework를 **포함**하고, 그 위에 편의 기능을 얹은 것이다.

---

## 한 줄 정리

|     | Spring Framework      | Spring Boot                |
| --- | --------------------- | -------------------------- |
| 역할  | IoC/DI/MVC 등 핵심 기능 제공 | Spring을 쉽게 쓰기 위한 설정 자동화 도구 |
| 설정  | 개발자가 직접               | Auto Configuration으로 자동    |
| 배포  | 외부 WAS 필요             | 내장 Tomcat으로 jar 실행         |
| 관계  | 엔진                    | 엔진 + 자동화 래퍼                |
