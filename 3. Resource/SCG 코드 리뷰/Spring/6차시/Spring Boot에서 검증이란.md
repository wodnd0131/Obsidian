

신입 때 흔히 드는 생각은 이거야: _"잘못된 값이 오면 에러 나면 되는 거 아냐?"_

틀리지 않았어. 하지만 **언제, 어디서, 어떤 에러가 나느냐**가 핵심이야.

DB에 `null`이 들어간 뒤 NPE가 터지는 것과, 요청 시점에 _"이 필드는 필수입니다"_ 라고 바로 알려주는 것 — 같은 에러가 아니야. 전자는 디버깅이고, 후자는 **계약(Contract)** 이야.

**검증이 요구하는 것은 결국 하나야: 시스템이 처리할 수 없는 데이터를 경계(boundary)에서 막는 것.** 내부로 들어올수록 고치기 어렵고, 잘못된 데이터가 DB까지 도달하면 이미 늦어.

---

### 검증에는 레이어가 있다

위 다이어그램에서 봤듯이, 검증은 한 곳에서 다 하는 게 아니야. 레이어마다 **책임이 다르고**, 그 경계를 지키는 게 좋은 설계야.

---

#### 1단계 — Controller 계층: 형식(Format) 검증

가장 먼저 맞닥뜨리는 관문이야. 여기서 물어볼 질문은 딱 하나: **"이 데이터가 우리 시스템이 이해할 수 있는 모양인가?"**

```java
// build.gradle — 이걸 먼저 추가
implementation 'org.springframework.boot:spring-boot-starter-validation'
```

```java
public class SignUpRequest {

    @NotBlank(message = "이메일은 필수입니다")
    @Email(message = "이메일 형식이 아닙니다")
    private String email;

    @NotBlank
    @Size(min = 8, max = 20, message = "비밀번호는 8~20자 사이여야 합니다")
    private String password;

    @NotNull
    @Min(value = 0, message = "나이는 0 이상이어야 합니다")
    private Integer age;
}
```

DTO에 어노테이션만 달았다고 검증이 되는 게 아니야. **`@Valid`를 Controller 파라미터에 붙여야 Spring이 검증을 실행해.**

```java
@RestController
@RequestMapping("/users")
public class UserController {

    @PostMapping
    public ResponseEntity<Void> signUp(@RequestBody @Valid SignUpRequest request) {
        // 여기 도달했다면 형식은 이미 통과
        userService.signUp(request);
        return ResponseEntity.created(...).build();
    }
}
```

검증 실패 시 Spring은 `MethodArgumentNotValidException`을 던져. 이걸 잡아서 클라이언트에게 명확한 메시지를 줘야 해:

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidation(
            MethodArgumentNotValidException e) {

        Map<String, String> errors = new HashMap<>();
        e.getBindingResult().getFieldErrors()
            .forEach(err -> errors.put(err.getField(), err.getDefaultMessage()));

        return ResponseEntity.badRequest().body(errors);
    }
}
```

**자주 쓰는 어노테이션 정리:**

|어노테이션|대상|설명|
|---|---|---|
|`@NotNull`|모든 타입|null 불가|
|`@NotEmpty`|String, Collection|null + empty("") 불가|
|`@NotBlank`|String|null + empty + 공백만 있는 것 불가|
|`@Email`|String|이메일 형식|
|`@Size(min, max)`|String, Collection|길이/크기 범위|
|`@Min` / `@Max`|숫자|최솟값 / 최댓값|
|`@Pattern(regexp)`|String|정규식|
|`@Positive`|숫자|양수만|

---

#### 2단계 — Service 계층: 비즈니스 규칙(Business Rule) 검증

Controller를 통과한 데이터는 형식은 맞아. 하지만 **"의미"가 맞는지는 다른 문제**야.

- 이메일이 이메일 형식이라도, **이미 가입된 이메일**이면 안 돼
- 나이가 숫자라도, **우리 서비스 이용 가능 연령**인지는 별도 규칙이야
- 주문 금액이 양수라도, **현재 재고가 있는지**는 DB를 봐야 알아

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;

    public void signUp(SignUpRequest request) {
        // 비즈니스 규칙 검증
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new DuplicateEmailException("이미 사용 중인 이메일입니다: " + request.getEmail());
        }
        
        // 통과했으면 실제 처리
        User user = User.of(request.getEmail(), request.getPassword(), request.getAge());
        userRepository.save(user);
    }
}
```

여기서 핵심은 **커스텀 예외**를 쓰는 거야. `IllegalArgumentException`을 그냥 던지면 나중에 어느 계층에서 어떤 이유로 실패했는지 알 수 없어.

```java
public class DuplicateEmailException extends RuntimeException {
    public DuplicateEmailException(String message) {
        super(message);
    }
}
```

그리고 이 예외도 `GlobalExceptionHandler`에서 잡아서 의미 있는 HTTP 응답으로 만들어줘야 해:

```java
@ExceptionHandler(DuplicateEmailException.class)
public ResponseEntity<String> handleDuplicate(DuplicateEmailException e) {
    return ResponseEntity.status(HttpStatus.CONFLICT).body(e.getMessage()); // 409
}
```

---

#### 3단계 — Domain 계층: 불변식(Invariant) 보호

이게 가장 많은 신입들이 놓치는 부분이야.

도메인 객체는 **자기 자신이 항상 유효한 상태여야 한다**는 보장을 스스로 해야 해. 외부에서 검증해줄 거라고 믿으면 안 돼 — 나중에 Service 코드가 바뀌거나, 다른 경로로 객체가 생성될 때 구멍이 생겨.

```java
public class User {

    private final String email;
    private final String password;
    private final int age;

    // 생성자에서 불변식을 스스로 지킨다
    private User(String email, String password, int age) {
        if (email == null || email.isBlank()) {
            throw new IllegalArgumentException("이메일은 필수입니다");
        }
        if (age < 0) {
            throw new IllegalArgumentException("나이는 0 이상이어야 합니다");
        }
        this.email = email;
        this.password = password;
        this.age = age;
    }

    public static User of(String email, String password, int age) {
        return new User(email, password, age);
    }
}
```

더 나아가면 **값 객체(Value Object)** 패턴으로 "이메일"이라는 개념 자체를 타입으로 만들 수 있어:

```java
public class Email {
    private final String value;
    
    public Email(String value) {
        if (value == null || !value.contains("@")) {
            throw new InvalidEmailException(value);
        }
        this.value = value;
    }
    
    public String getValue() { return value; }
}

// 사용하는 쪽
public class User {
    private final Email email; // String이 아니라 Email 타입
    // ...
}
```

이렇게 하면 `Email` 타입이 보이는 순간 "이건 이미 유효한 이메일"이라는 신뢰가 생겨. **타입 시스템이 검증을 대신해주는 거야.**

---

### 커스텀 검증 어노테이션 만들기

Bean Validation이 제공하는 어노테이션으로 부족할 때가 있어. 예를 들어 "한국 전화번호 형식"이나 "특정 enum 값 중 하나여야 함" 같은 경우. 이럴 때 커스텀 어노테이션을 만들어:

```java
// 1. 어노테이션 정의
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = KoreanPhoneValidator.class)
public @interface KoreanPhone {
    String message() default "올바른 한국 전화번호 형식이 아닙니다";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// 2. 실제 검증 로직
public class KoreanPhoneValidator implements ConstraintValidator<KoreanPhone, String> {
    
    private static final Pattern PHONE_PATTERN = 
        Pattern.compile("^01[016789]-?\\d{3,4}-?\\d{4}$");

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null) return true; // @NotNull과 조합해서 쓰도록
        return PHONE_PATTERN.matcher(value).matches();
    }
}

// 3. 사용
public class ProfileRequest {
    @KoreanPhone
    private String phoneNumber;
}
```

---

### DTO 단위 테스트 — 놓치기 쉬운 부분

Controller 통합 테스트 말고, DTO 자체를 단위 테스트로 검증하는 방법도 알아야 해. 이게 훨씬 빠르고 격리된 테스트야:

```java
class SignUpRequestTest {

    private static Validator validator;

    @BeforeAll
    static void init() {
        ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
        validator = factory.getValidator();
    }

    @Test
    void 이메일이_비어있으면_검증_실패() {
        SignUpRequest request = new SignUpRequest("", "password123", 25);
        
        Set<ConstraintViolation<SignUpRequest>> violations = validator.validate(request);
        
        assertThat(violations).isNotEmpty();
        assertThat(violations)
            .anyMatch(v -> v.getPropertyPath().toString().equals("email"));
    }

    @Test
    void 유효한_요청은_검증_통과() {
        SignUpRequest request = new SignUpRequest("user@example.com", "password123", 25);
        
        Set<ConstraintViolation<SignUpRequest>> violations = validator.validate(request);
        
        assertThat(violations).isEmpty();
    }
}
```

---

### 정리: 레이어별 검증 책임 한 줄 요약

**Controller**는 "이 데이터가 우리 API가 이해할 수 있는 모양인가?"를 물어. @Valid + Bean Validation 어노테이션으로 처리하고, 실패하면 400 Bad Request를 돌려줘.

**Service**는 "이 데이터가 현재 시스템 상태에서 허용되는가?"를 물어. DB 조회나 비즈니스 규칙이 필요한 검증이 여기 와야 해. 커스텀 예외와 함께.

**Domain**은 "이 객체가 항상 유효한 상태를 보장하는가?"를 스스로 책임져. 생성자와 값 객체로 불변식을 캡슐화해.

---

### 신입 때 자주 하는 실수

Service 계층에서 Controller 검증을 또 하거나, 반대로 Controller에서 DB까지 다녀오는 비즈니스 검증을 하는 거야. 각 레이어의 책임을 섞으면 테스트하기 어려워지고, 나중에 로직을 바꿀 때 어디를 건드려야 할지 몰라서 두려워져.

**"이 검증은 어느 계층의 책임인가?"** 라는 질문을 습관처럼 하면 돼. 그게 검증 설계의 핵심이야.