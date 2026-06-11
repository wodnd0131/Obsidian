픽잇의 로깅은 기본적으로 `slf4j + logback` 조합으로 이루어져있습니다.

시각화 파이프라인은 `alloy → loki → grafana` 로 이루어져있습니다.

## 애플리케이션에서의 로깅
### 1. Client → LogFilter : HTTP 요청 진입!

![[../../../Repository/로깅 애플리케이션 1.png]]
클라이언트로부터 HTTP 요청이 들어오면, 모든 요청은 컨트롤러에 도달하기 이전에

`LogFilter(OncePerRequestFilter)`를 먼저 통과합니다.

`LogFilter`는 **요청 단위 로깅의 시작 지점**이며,

이 시점부터 요청 추적을 위한 준비가 진행됩니다.

---

### 2. LogFilter : Request / Response 래핑

요청 및 응답 바디를 이후 단계에서도 참조할 수 있도록

`ContentCachingRequestWrapper`와 `ContentCachingResponseWrapper`로 감쌉니다.

이를 통해 컨트롤러나 서비스 로직에서 바디를 이미 읽었더라도,

필터 단계에서 요청/응답 내용을 다시 확인할 수 있습니다.

---

### 3. LogFilter : MDC 초기화 (요청 추적 정보 세팅)

```java
MDC.put("request_id", ...)
MDC.put("request_uri", ...)
MDC.put("client_ip", ...)
```

각 요청마다 고유한 `request_id`를 생성하고,

요청 URI 및 클라이언트 IP와 함께 MDC(Mapped Diagnostic Context)에 저장합니다.

이 시점 이후로 기록되는 모든 로그에는 MDC에 저장된 정보가 자동으로 포함됩니다.

이 구조를 통해 다음 로그들이 모두 동일한 `request_id`로 연결됩니다.

- RequestLog
- BusinessLog
- ErrorLog
- ResponseLog

---

### 4, LogFilter → DispatcherServlet : 실제 애플리케이션 로직 진입

```java
filterChain.doFilter(...)
```

MDC 설정이 완료된 후, 요청은 `DispatcherServlet`으로 전달되며

Spring MVC의 일반적인 처리 흐름(Controller → Service → Domain)을 따르게 됩니다.

---

## 정상 흐름(Logical Flow)에서의 로그 처리

---

### 5. DispatcherServlet → BusinessLogAspect (optional)

비즈니스 로직 중 `@BusinessLogging` 어노테이션이 선언된 메서드가 호출될 경우,

`BusinessLogAspect`가 AOP 방식으로 개입합니다.

```java
joinPoint.proceed()
```

위 메서드를 통해 실제 비즈니스 로직이 수행됩니다.

---

### 6. BusinessLogAspect → Logback : BusinessLog 기록

비즈니스 로직이 정상적으로 수행된 이후,

비즈니스 의미 단위의 로그(`BusinessLog`)가 기록됩니다.

- 사용자 ID
- 수행된 액션(action)
- 기타 도메인 의미 정보

해당 로그는 marker 기반 구조화 로그이며,

MDC에 저장된 `request_id`가 함께 포함됩니다.

이 로그는 **HTTP 요청 로그와는 별도로**,

비즈니스 이벤트를 추적하기 위한 용도로 사용됩니다.

---

### 7. DispatcherServlet → LogFilter 복귀

컨트롤러 및 서비스 로직이 정상적으로 종료되면,

응답 객체가 생성된 상태로 다시 `LogFilter`로 흐름이 돌아옵니다.

---

### 8. LogFilter : RequestLog 기록

```java
RequestLog.of(...)
log.info(...)
```

요청 자체에 대한 요약 정보(RequestLog)를 기록합니다.

- 요청 URI
- HTTP Method
- 요청 파라미터 및 Body
- 요청 메타 정보

---

## 예외 흐름(Exception Flow)에서의 로그 처리

---

### 5. DispatcherServlet → Service : 예외 발생

컨트롤러 또는 서비스 로직 수행 중 예외가 발생할 경우,

해당 예외는 상위로 전파됩니다.

---

### 6. DispatcherServlet → GlobalExceptionHandler

`@RestControllerAdvice`로 선언된 `GlobalExceptionHandler`가

발생한 예외를 가로채어 처리합니다.

예외 유형에 따라 서로 다른 처리 로직이 적용됩니다.

---

### 7. GlobalExceptionHandler → Logback : ErrorLog 기록

예외 유형 및 HTTP 상태 코드에 따라 다음과 같이 로그 레벨이 결정됩니다.

- 클라이언트 오류(4xx): INFO 또는 WARN
- 외부 API 오류 / 서버 오류(5xx): ERROR

`ErrorLog` DTO를 사용하여 에러 로그 구조를 일관되게 유지하며,

MDC의 `request_id`가 자동으로 포함됩니다.

이 시점에서는 아직 응답 로그(ResponseLog)는 기록되지 않습니다.

---

## 공통 종료 처리 (정상 / 예외 공통)

---

### 9. LogFilter finally 블록 : ResponseLog 기록

```java
ResponseLog.of(...)
log.info(...)
```

`finally` 블록에서 실행되므로,

정상 처리 여부와 관계없이 **항상 ResponseLog가 기록됩니다**.

- HTTP 상태 코드
- 응답 시간(elapsed time)
- 요청 URI

---

### 10. LogFilter : 응답 바디 복원

```java
cacheResponse.copyBodyToResponse()
```

`ContentCachingResponseWrapper`에 저장된 응답 바디를

실제 클라이언트 응답으로 다시 전달합니다.

---

### 11. LogFilter : MDC 정리

```java
MDC.clear()
```

요청 처리가 종료된 이후 MDC를 명확히 정리하여,

스레드 재사용으로 인한 로그 정보 오염을 방지합니다.

---

### 12. LogFilter → Client : HTTP 응답 반환

모든 로그 기록이 완료된 후,

클라이언트로 HTTP 응답이 정상적으로 반환됩니다.

---

## 요약

- 모든 로그의 시작과 종료는 `LogFilter`에서 관리됩니다.
- `request_id`는 요청 단위 로그를 연결하는 핵심 식별자입니다.
- BusinessLog와 ErrorLog는 **중간 이벤트 로그**입니다.
- ResponseLog는 정상/예외와 관계없이 반드시 기록됩니다.
- MDC clear를 통해 로그 오염 문제를 방지합니다.

---

## 로그 파이프라인
![[../../../Repository/로깅 파이프.png]]
### ❗중요❗

본 로그 파이프라인은 **Loki 멀티 테넌시(Multi-Tenancy)** 구성을 사용하고 있으며,

이를 위해 **`X-Scope-OrgID` 헤더를 필수적으로 사용**합니다.

- **prod 환경**: `X-Scope-OrgID = pickeat-prod`
- **dev 환경**: `X-Scope-OrgID = pickeat-dev`

이 값이 없거나 잘못 설정되면 Grafana에서 데이터소스 등록이 정상적으로 되지않습니다.

**`X-Scope-OrgID` 설정은 config.alloy의 다음 부분에 명시되어있습니다**

```sql
loki.write "grafana_loki" {
  endpoint {
    url = "<https://monitoring.pickeat.io.kr/loki/loki/api/v1/push>"
    tenant_id = "pickeat-prod"
    batch_wait = "1s"
    batch_size = "1MB"
  }
}
```

여기서 **`tenant_id = "pickeat-prod"`**가 **`X-Scope-OrgID`** 값이 됩니다.

해당 값을 그라파나에서 데이터 소스를 등록할 때도 설정해주어야합니다.

![image.png](attachment:ff70514e-4273-463f-8c72-0eed08e2025b:image.png)

(이것이 같은 Loki인데 Loki-Pickeat-Dev와 Loki-Pickeat-Prod로 데이터소스가 나뉘어져있는 이유…)

### 설명글

## 1. Spring Boot → Alloy : 로그 파일 생성 및 감지

Spring Boot 애플리케이션은

Logback 설정에 따라 **JSON 형태의 로그 파일**을 서버 로컬 디스크에 기록합니다.

- 로그는 Rolling 정책에 따라 파일로 관리됩니다.
    
- 애플리케이션은 로그를 **파일에만 기록**하며,
    
    네트워크 전송이나 외부 시스템과 직접 통신하지 않습니다.
    

Grafana Alloy는 서버 내에서 도커 컨테이너 형태로 실행되며,

지정된 로그 파일 경로를 지속적으로 감시(tail)합니다.

이 구조를 통해 애플리케이션은 로그 수집 파이프라인과 완전히 분리됩니다.

---

## 2. Alloy : 로그 수집 및 전처리

Alloy는 다음 역할을 수행합니다.

### 2-1. 로그 파일 Tail

- `loki.source.file`을 통해 로그 파일 변경 사항을 실시간으로 읽어옵니다.

### 2-2. 라벨 부여

```hcl
stage.static_labels {
  values = {
    service = "pickeat"
    env = "prod"
  }
}
```

- 서비스명, 환경(prod/dev 등)을 라벨로 부여합니다.
- Grafana 및 Loki에서 로그 필터링의 기준이 됩니다.

### 2-3. 전처리(Multiline 등)

- 멀티라인 로그가 있을 경우 하나의 이벤트로 묶어 처리합니다.
- 로그 구조를 Loki에 적합한 형태로 정리합니다.

---

## 3. Alloy → Loki : 로그 전송 (Push)

전처리가 완료된 로그는

Alloy에서 Loki의 HTTP Ingestion API로 **Push 방식**으로 전송됩니다.

- 배치 단위로 전송되어 네트워크 효율을 확보합니다.
- 전송 실패 시 재시도가 자동으로 처리됩니다.

애플리케이션 서버는 Loki와 직접 통신하지 않습니다.

---

## 4. Loki : 로그 저장 및 인덱싱

Loki는 수신된 로그를 다음과 같이 처리합니다.

### 4-1. 로그 데이터 저장

- 실제 로그 본문은 S3(Object Storage)에 저장됩니다.
- 장기 보관 및 비용 효율성을 고려한 구조입니다.

### 4-2. 인덱스 관리

- 로그의 라벨 정보(service, env 등)를 기준으로 인덱스를 생성합니다.
- 인덱스는 로컬 캐시와 S3를 함께 사용하여 관리됩니다.

### 4-3. 보관 정책(Retention)

- 설정된 기간(예: 30일) 이후 로그는 자동으로 삭제됩니다.

---

## 5. Grafana : 로그 조회 및 시각화

Grafana는 Loki를 데이터 소스로 사용하여 다음 기능을 제공합니다.

- 로그 탐색(Explore)
- 대시보드 기반 로그 시각화
- request_id 기반 요청 단위 추적
- 에러 유형별 패널 구성

운영자는 Grafana를 통해

**실시간 로그 확인 및 과거 로그 분석**을 수행할 수 있습니다.

---

## 6. Grafana → Discord : 알림 전송

특정 시간 동안 ERROR 로그 발생 등의 조건이 충족되면,

- Grafana Alertmanager가 알림을 생성
- Discord Webhook을 통해 알림 메시지를 전송합니다.

---

추가 자료

[Alloy + Loki + Grafana 로깅 시스템 구축기](https://app.notion.com/p/Alloy-Loki-Grafana-2470c7355c2a806a8a1acd6c3e54ecbf?pvs=21)

[픽잇 JSON 로깅 개선기](https://app.notion.com/p/JSON-2780c7355c2a80d7befee8899a767c45?pvs=21)

[로그 대시보드 개선](https://app.notion.com/p/2780c7355c2a80c8b671c847e6d47dc9?pvs=21)

[로그 S3로 옮기기](https://app.notion.com/p/S3-2860c7355c2a809b9b59e0293dc97c0b?pvs=21)