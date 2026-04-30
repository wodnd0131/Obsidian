# Jackson ObjectMapper — 내부 동작과 설계 근거

---

## 1. Problem First

### 시나리오 A: Spring 기본 설정을 믿다가 프로덕션에서 터지는 경우

```java
// API 응답 모델
public class OrderResponse {
    private Long orderId;
    private OrderStatus status;
    private LocalDateTime createdAt;
    private BigDecimal totalPrice;
}
```

```json
// 클라이언트가 받은 응답
{
  "orderId": 1234,
  "status": "PENDING",
  "createdAt": [2024, 1, 15, 10, 30, 0, 0],  // ← 배열?
  "totalPrice": 99.99
}
```

`LocalDateTime`이 배열로 직렬화된다. 프론트엔드는 문자열 `"2024-01-15T10:30:00"`을 기대하고 있었다. **Spring Boot 기본 ObjectMapper에 `JavaTimeModule`이 등록되어 있지 않아서 생기는 문제다.**

---

### 시나리오 B: ObjectMapper를 Bean으로 등록했더니 오히려 망가진 경우

```java
@Configuration
public class JacksonConfig {
    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper(); // 순수 기본 ObjectMapper
    }
}
```

이 순간 **Spring Boot의 자동 설정(`JacksonAutoConfiguration`)이 무력화된다.** Spring MVC가 내부적으로 쓰던 ObjectMapper 설정이 전부 날아간다.

```
증상:
- Swagger UI JSON 파싱 오류
- Spring Security의 인증 객체 직렬화 실패
- Actuator 엔드포인트 응답 포맷 깨짐
```

---

### 시나리오 C: 같은 ObjectMapper를 멀티스레드에서 공유했을 때

```java
// 흔히 하는 실수
public class OrderService {
    private ObjectMapper objectMapper = new ObjectMapper(); // 인스턴스 필드

    public String serialize(Order order) {
        return objectMapper.writeValueAsString(order); // 스레드 안전한가?
    }
}
```

결론부터: **`ObjectMapper` 자체는 스레드 안전하다.** 단, `ObjectReader` / `ObjectWriter`를 만들기 전에 설정을 변경하면 안 된다.

```java
// DANGEROUS: 런타임에 설정 변경
objectMapper.configure(SerializationFeature.INDENT_OUTPUT, true); // 레이스 컨디션
```

> 📎 **근거:** Jackson 공식 문서 — _"Shared ObjectMapper instances are thread-safe after configuration is done"_

---

## 2. Mechanics

### 2-1. ObjectMapper 내부 구조

```
ObjectMapper
├── SerializationConfig      ← 직렬화 설정 (불변 스냅샷)
├── DeserializationConfig    ← 역직렬화 설정 (불변 스냅샷)
├── SerializerProvider       ← Serializer 캐시 & 조회
├── DeserializationContext   ← Deserializer 캐시 & 조회
├── TypeFactory              ← 제네릭 타입 처리
├── InjectableValues         ← 역직렬화 시 의존성 주입
└── Module 목록              ← 등록된 확장 모듈
```

**직렬화 요청이 들어왔을 때 내부 흐름:**

```
objectMapper.writeValueAsString(order)
    ↓
_serializerProvider().serializeValue(gen, value)
    ↓
SerializerProvider.findValueSerializer(Order.class)
    ↓
① 캐시 확인 (SerializerCache)
② 캐시 미스 → SerializerFactory로 새 Serializer 생성
③ BeanSerializerFactory → Order 클래스 분석
    ↓
BeanSerializer 생성 과정:
    - 리플렉션으로 프로퍼티 목록 수집
    - 각 프로퍼티별 Serializer 결정
    - 결과를 SerializerCache에 저장
    ↓
JsonGenerator로 출력
```

**Serializer는 한 번 생성되면 캐시된다.** 같은 타입을 반복 직렬화할 때 리플렉션 비용이 반복 발생하지 않는 이유다.

---

### 2-2. 프로퍼티 탐색 순서 — 정확한 규칙

Jackson이 직렬화할 필드를 결정하는 순서:

```java
public class Order {
    public Long id;                    // ① public 필드
    private String status;             // ③ private 필드 (단독으론 무시)
    
    public String getCustomerName() {  // ② public getter → "customerName" 키
        return customerName;
    }
    
    @JsonProperty("order_status")      // ④ @JsonProperty → 이름 오버라이드
    public String getStatus() {
        return status;
    }
}
```

**`MapperFeature.DEFAULT_VIEW_INCLUSION` 기본값에 따라 달라지는 동작:**

```java
// 기본 가시성 규칙 (VisibilityChecker)
objectMapper.setVisibility(PropertyAccessor.FIELD, Visibility.ANY);
// → private 필드도 직렬화 대상에 포함

objectMapper.setVisibility(PropertyAccessor.GETTER, Visibility.PUBLIC_ONLY);
// → public getter만 대상 (기본값)
```

> 📎 **근거:** Jackson Databind 소스코드 — `VisibilityChecker.Std`, `BeanSerializerFactory.findBeanProperties()`

---

### 2-3. 역직렬화 내부 — 타입 결정과 객체 생성

```
objectMapper.readValue(json, Order.class)
    ↓
JsonParser → 토큰 스트림 생성
    ↓
DeserializationContext.findRootValueDeserializer(Order.class)
    ↓
BeanDeserializer 생성:
    ① Order.class에서 생성자 탐색
        - 기본 생성자 (우선)
        - @JsonCreator 붙은 생성자/팩토리 메서드
    ② JSON 키 → 프로퍼티 매핑 테이블 구성
    ③ 각 프로퍼티 타입별 Deserializer 결정
    ↓
JSON 토큰 순회하며 객체에 값 주입
    ↓
(기본 생성자로 객체 생성 후 setter/필드 직접 주입)
```

**`@JsonCreator`가 필요한 경우:**

```java
// 불변 객체 — 기본 생성자 없음
public class Money {
    private final long amount;
    private final String currency;

    // Jackson이 이 생성자를 쓰도록 명시
    @JsonCreator
    public Money(
            @JsonProperty("amount") long amount,
            @JsonProperty("currency") String currency) {
        this.amount = amount;
        this.currency = currency;
    }
}
```

**`@JsonCreator`를 안 붙이면:**

```
InvalidDefinitionException:
Cannot construct instance of `Money`
(no Creators, like default constructor, exist)
```

---

### 2-4. Module 시스템 — 확장 동작 원리

Jackson이 기본적으로 모르는 타입(`LocalDateTime`, `Jdk8 Optional` 등)을 처리하려면 **Module을 등록해서 Serializer/Deserializer를 추가**해야 한다.

```java
// Module 등록 구조
objectMapper.registerModule(new JavaTimeModule());

// JavaTimeModule 내부가 하는 일:
public class JavaTimeModule extends SimpleModule {
    public JavaTimeModule() {
        // Serializer 등록
        addSerializer(LocalDateTime.class, new LocalDateTimeSerializer());
        addSerializer(LocalDate.class, new LocalDateSerializer());
        addSerializer(Instant.class, new InstantSerializer());
        
        // Deserializer 등록
        addDeserializer(LocalDateTime.class, new LocalDateTimeDeserializer());
        // ...
    }
}
```

**Module이 등록되면 SerializerFactory의 탐색 순서가 바뀐다:**

```
타입에 맞는 Serializer 탐색 순서:
① @JsonSerialize로 직접 지정된 Serializer
② 등록된 Module에서 해당 타입 Serializer 탐색  ← Module이 여기 끼어든다
③ 기본 내장 Serializer (String, Integer, ...)
④ BeanSerializer (일반 POJO)
```

> 📎 **근거:** Jackson Databind 소스코드 — `SimpleSerializers.findSerializer()`, jackson-modules-java8 GitHub 레포지토리

---

### 2-5. Spring Boot의 ObjectMapper 자동 설정

Spring Boot가 ObjectMapper를 어떻게 구성하는지 소스 레벨에서:

```java
// JacksonAutoConfiguration
@Bean
@ConditionalOnMissingBean(ObjectMapper.class) // ← Bean이 이미 있으면 건너뜀
public ObjectMapper jacksonObjectMapper(Jackson2ObjectMapperBuilder builder) {
    return builder.createXmlMapper(false).build();
}
```

**`Jackson2ObjectMapperBuilder`가 기본으로 적용하는 설정:**

```java
// Spring Boot 기본 설정 (application.yml로 제어 가능)
WRITE_DATES_AS_TIMESTAMPS = false        // LocalDateTime → 문자열
FAIL_ON_UNKNOWN_PROPERTIES = false       // 모르는 필드 무시
MapperFeature.DEFAULT_VIEW_INCLUSION = false

// 클래스패스에 있으면 자동 등록되는 모듈:
// - JavaTimeModule (jackson-datatype-jsr310)
// - Jdk8Module    (jackson-datatype-jdk8)
// - ParameterNamesModule
```

**올바른 커스터마이징 방법 — Bean을 교체하지 않는다:**

```java
// WRONG: Boot 자동 설정 전체를 날림
@Bean
public ObjectMapper objectMapper() {
    ObjectMapper mapper = new ObjectMapper();
    mapper.registerModule(new JavaTimeModule());
    return mapper;
}

// RIGHT 1: Builder를 커스터마이징
@Bean
public Jackson2ObjectMapperBuilderCustomizer customizer() {
    return builder -> builder
        .featuresToDisable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
        .modules(new JavaTimeModule());
}

// RIGHT 2: application.yml로 제어
// spring.jackson.serialization.write-dates-as-timestamps: false
// spring.jackson.default-property-inclusion: non_null
```

> 📎 **근거:** Spring Boot 공식 레퍼런스 — _"Auto-configured Jackson"_, docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.json.jackson

---

### 2-6. 자주 쓰는 설정의 실제 동작

```java
ObjectMapper mapper = new ObjectMapper();

// 1. 모르는 필드 무시
// API 응답에 새 필드가 추가돼도 역직렬화 실패 안 함
mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);

// 2. null 필드 제외
// {"name":"홍길동","email":null} → {"name":"홍길동"}
mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);

// 3. 빈 컬렉션 제외
// {"items":[]} → {}
mapper.setSerializationInclusion(JsonInclude.Include.NON_EMPTY);

// 4. snake_case 자동 변환
// Java: orderStatus → JSON: order_status
mapper.setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);
```

**`FAIL_ON_UNKNOWN_PROPERTIES`를 false로 두면 잃는 것:**

```java
// 클라이언트가 오타로 잘못된 필드명을 보낸 경우
{
  "order_id": 123,   // 실제 필드명: orderId
  "stauts": "DONE"   // 오타: status
}

// FAIL_ON_UNKNOWN_PROPERTIES = false이면
// 아무 오류 없이 역직렬화 성공
// orderId = null, status = null로 세팅됨
// 버그가 조용히 묻힌다
```

---

### 2-7. 다형성 역직렬화 — `@JsonTypeInfo`

```java
// 이전 gRPC 편에서 언급한 문제
@JsonTypeInfo(
    use = JsonTypeInfo.Id.NAME,      // 타입 식별자로 이름 사용
    include = As.PROPERTY,           // JSON 필드로 포함
    property = "type"                // 필드명
)
@JsonSubTypes({
    @JsonSubTypes.Type(value = EmailNotification.class, name = "EMAIL"),
    @JsonSubTypes.Type(value = SmsNotification.class,   name = "SMS")
})
public abstract class Notification { }
```

```json
// 직렬화 결과
{"type": "EMAIL", "to": "user@example.com", "subject": "주문 완료"}

// 역직렬화 시 Jackson 동작:
// 1. "type" 필드를 먼저 읽음
// 2. "EMAIL" → EmailNotification.class 매핑
// 3. EmailNotification으로 역직렬화
```

**`enableDefaultTyping()`을 절대 쓰면 안 되는 이유:**

```java
// DANGEROUS
mapper.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);

// 이 설정은 모든 non-final 타입에 대해
// "@class" 필드를 JSON에 포함하고
// 역직렬화 시 해당 클래스를 로드한다

// 공격자가 보내는 JSON:
{
  "@class": "com.zaxxer.hikari.HikariDataSource",
  "jdbcUrl": "jdbc:h2:mem:;TRACE_LEVEL_SYSTEM_OUT=3;INIT=..."
}
// → 임의 클래스 인스턴스화 → RCE
```

> 📎 **근거:** CVE-2017-7525, Jackson 공식 블로그 — _"Jackson 2.x의 Polymorphic Deserialization 취약점"_, `@JsonTypeInfo`를 명시적으로 쓰되 허용 타입을 제한할 것을 권고

---

## 3. 공식 근거 정리

|주장|출처|
|---|---|
|ObjectMapper 스레드 안전성|Jackson 공식 문서 — Threading|
|프로퍼티 가시성 규칙|Jackson Databind — `VisibilityChecker` Javadoc|
|Module 등록 및 Serializer 탐색 순서|Jackson Databind 소스 — `SimpleSerializers`|
|Spring Boot ObjectMapper 자동 설정|Spring Boot 공식 레퍼런스 §features.json.jackson|
|`Jackson2ObjectMapperBuilderCustomizer` 사용 권고|Spring Boot 공식 레퍼런스|
|`enableDefaultTyping` RCE|CVE-2017-7525, CVE-2019-14379|
|`@JsonTypeInfo` 안전한 다형성 처리|Jackson Databind 공식 문서 — Polymorphic Type Handling|

---

## 4. 이 설계를 이렇게 한 이유

### Serializer 캐싱 구조의 이점과 대가

**이점:** 동일 타입 반복 직렬화 시 리플렉션 비용 제거. 고트래픽 환경에서 결정적 성능 차이.

**대가:** 캐시는 타입 기준으로 저장된다. 제네릭 타입을 잘못 전달하면 캐시에서 잘못된 Serializer를 가져온다.

```java
// WRONG: 제네릭 타입 정보 소실 (Type Erasure)
List<Order> orders = getOrders();
mapper.writeValueAsString(orders);
// → List 안의 타입을 Object로 추론할 수 있음

// RIGHT: TypeReference로 제네릭 타입 명시
mapper.readValue(json, new TypeReference<List<Order>>() {});
// TypeReference가 익명 클래스 상속으로 제네릭 타입 정보를 런타임에 보존
```

### `FAIL_ON_UNKNOWN_PROPERTIES = false`가 Spring Boot 기본값인 이유

**설계 의도:** 마이크로서비스 환경에서 API 버전이 다른 서비스 간 통신 시 필드가 추가되더라도 구버전 클라이언트가 깨지지 않도록.

**잃는 것:** 오타나 잘못된 필드명이 조용히 무시된다. 내부 서비스 간 통신에서는 `true`로 설정해서 계약 위반을 즉시 감지하는 게 나을 수 있다.

---

## 5. 이어지는 개념

|순서|개념|이유|
|---|---|---|
|1|**Java Type Erasure & TypeReference**|제네릭 타입을 Jackson에 올바르게 전달하려면 Type Erasure가 왜 문제인지 알아야 한다 — `List<Order>` 역직렬화 버그의 근본 원인|
|2|**Spring MVC `HttpMessageConverter`**|ObjectMapper가 Spring MVC 안에서 어떻게 연결되는지 — `@RequestBody` / `@ResponseBody`가 ObjectMapper를 호출하는 실제 경로|
|3|**Redis 직렬화 전략 (`RedisTemplate`)**|`GenericJackson2JsonRedisSerializer` vs `Jackson2JsonRedisSerializer` 선택 기준 — 다형성 타입 정보 포함 여부가 핵심|
|4|**`@JsonView` & DTO 분리 전략**|같은 엔티티를 API마다 다르게 직렬화해야 할 때 — ObjectMapper 설정보다 더 정밀한 제어가 필요한 시점|