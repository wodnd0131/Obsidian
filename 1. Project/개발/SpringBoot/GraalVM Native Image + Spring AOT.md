#SpringBoot #CS/최적화
## 왜 이게 나왔는가 — 시작 시간 문제
방금 다룬 Spring Boot 초기화 순서를 보면:

```
시작 시 일어나는 일:
ComponentScan    → 클래스 파일 수백 개 읽기
AutoConfiguration → 조건 판단 (수백 개의 @Conditional 평가)
HandlerMapping   → @RequestMapping 스캔
Bean 생성        → 의존성 그래프 분석 + 객체 생성
리플렉션         → 메서드/필드 메타데이터 수집

결과: 일반적인 Spring Boot 앱 시작 시간 2~5초
```

일반 서버에서는 괜찮다. 한 번 뜨면 계속 떠있으니까.

그런데 **AWS Lambda, Google Cloud Functions** 같은 환경에서는:

```
요청이 올 때마다 새 인스턴스를 띄울 수 있음
                │
                ▼
요청 처리 시간: 50ms
Spring Boot 시작 시간: 3000ms

사용자가 느끼는 응답: 3050ms ← 대부분이 시작 시간
```

이게 **Cold Start 문제**다. 서버리스 환경에서 Spring Boot를 쓰기 어려운 근본 이유였다.

---

## JVM의 근본적인 동작 방식이 원인

```
Java 코드 → javac → 바이트코드(.class) → JVM이 런타임에 해석/실행

JVM이 런타임에 하는 일:
- 클래스 로딩
- 바이트코드 해석
- JIT 컴파일 (자주 쓰는 코드를 네이티브 코드로 변환)
- 가비지 컬렉션

이 모든 게 실행 중에 일어남 → 시작이 느리고 메모리를 많이 씀
```

Spring이 리플렉션을 많이 쓰는 것도 문제를 키운다.

```java
// 리플렉션 = 런타임에 클래스 구조를 동적으로 분석
Class<?> clazz = Class.forName("com.example.UserController");
Method method = clazz.getMethod("getUser", Long.class);
method.invoke(instance, 42L);

// JVM 입장에서 비싼 작업
// + JIT가 최적화하기 어려움 (동적이라서)
```

---

## GraalVM Native Image — 접근 방식 자체를 바꾼다

GraalVM은 Oracle이 만든 **고성능 JDK 구현체**다. 여기서 `native-image` 라는 도구가 핵심이다.

```
기존 Java:
소스코드 → [빌드] → 바이트코드 → [런타임: JVM] → 실행

Native Image:
소스코드 → [빌드: AOT 컴파일] → 네이티브 실행파일 → [런타임: JVM 없음] → 실행
```

**AOT = Ahead-Of-Time 컴파일**

JVM이 런타임에 하던 일을 **빌드 시점에 미리 다 해버린다.**

```
빌드 시 일어나는 일:
- 도달 가능한 모든 클래스 분석 (정적 분석)
- 바이트코드 → 네이티브 기계어 컴파일
- 필요한 클래스만 실행파일에 포함
- 힙 스냅샷 생성 (초기 Bean 상태를 이미지에 굽기)

결과물: 운영체제가 직접 실행 가능한 단일 실행파일
        JVM 설치 불필요
```

결과:

```
시작 시간: 3000ms → 50ms 이하
메모리:    500MB  → 50~100MB
실행파일:  jar    → OS 네이티브 바이너리 (Linux면 ELF)
```

---

## Spring AOT — Spring이 Native Image를 지원하기 위해 한 것

문제가 있었다. Spring은 리플렉션과 동적 프록시를 매우 많이 쓴다.

```
Native Image의 제약:
"빌드 시점에 알 수 없는 것은 런타임에도 할 수 없다"

Spring이 런타임에 동적으로 하던 것들:
- 리플렉션으로 @Controller 스캔
- CGLIB으로 @Transactional 프록시 동적 생성
- @Autowired 의존성 런타임 주입
→ 이것들이 Native Image에서 그냥은 동작 안 함
```

Spring 6 / Spring Boot 3부터 **Spring AOT**가 이를 해결한다.

```
빌드 단계에 AOT 처리 추가:

mvn spring-boot:build-image (또는 ./gradlew nativeCompile)
    │
    ▼
Spring AOT 처리기 실행
    │
    ├─ ApplicationContext를 빌드 시점에 미리 실행
    │  (Bean 스캔, 의존성 분석을 런타임이 아닌 빌드 시에)
    │
    ├─ 리플렉션 정보를 정적 코드로 생성
    │  (런타임 리플렉션 → 빌드 시 생성된 일반 코드)
    │
    ├─ CGLIB 동적 프록시 → 빌드 시 미리 생성된 프록시 클래스
    │
    └─ GraalVM에게 힌트 파일 생성
       (reflect-config.json, proxy-config.json 등)
           "이 클래스들은 리플렉션으로 접근할 거야"
    │
    ▼
GraalVM native-image 컴파일
    │
    ▼
네이티브 실행파일 완성
```

---

## 트레이드오프 — 공짜가 아니다

```
얻는 것:
빠른 시작 (50ms)
낮은 메모리
JVM 불필요 → 컨테이너 이미지 경량화

잃는 것:

1. 빌드 시간이 매우 길어진다
   일반 jar 빌드: 5~10초
   Native Image 빌드: 3~10분

2. 빌드 시 메모리를 많이 쓴다
   Native Image 빌드: 4~8GB RAM 필요

3. JIT 최적화가 없다
   JVM은 런타임에 자주 쓰는 코드를 점점 빠르게 최적화함
   Native Image는 처음부터 끝까지 동일한 속도
   → 장기 실행 서버에서는 JVM이 결국 더 빠를 수 있음

4. 동적 기능 제약
   런타임에 클래스를 동적으로 로드하는 코드는 별도 힌트 필요
   일부 라이브러리는 Native Image 미지원
```

```
결론적으로 적합한 상황:

Native Image가 유리:          JVM이 유리:
- 서버리스 (Lambda)           - 장기 실행 서버
- CLI 도구                   - 트래픽이 많고 오래 뜨는 서비스
- 컨테이너 빠른 스케일 아웃   - 동적 기능을 많이 쓰는 경우
- 메모리가 중요한 환경        - 빌드 시간이 중요한 개발환경
```

---

## 한 줄 정리

```
JVM:          "바이트코드를 받아서 런타임에 다 처리할게"
Native Image: "빌드할 때 다 처리해서 바로 실행 가능한 파일 줄게"
Spring AOT:   "Spring이 런타임에 동적으로 하던 것들을 빌드 시점으로 당겨줄게"

셋이 합쳐지면:
Spring Boot 앱이 JVM 없이 50ms 안에 뜨는 네이티브 실행파일이 된다
```

실무에서 당장 쓸 기술은 아닐 수 있다. 하지만 **"왜 Spring Boot 시작이 느린가"** 의 근본 원인과, **"그 한계를 어떻게 해결하는가"** 의 방향을 이해하는 데 의미가 있다.