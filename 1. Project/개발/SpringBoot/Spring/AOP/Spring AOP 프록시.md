## 1. Problem First

Bean을 그냥 `new`로 만들면 끼어들 수 없다.

```java
// 이 코드에서 AOP가 작동하려면
@Service
public class OrderService {
    
    @Transactional
    public void placeOrder(OrderRequest req) {
        // DB 저장
    }
}

// 누군가가 placeOrder() 호출 전에 트랜잭션을 시작하고
// 호출 후에 커밋/롤백해야 한다
// 근데 OrderService 코드를 건드리지 않고 어떻게?
```

```
직접 해결하려면:
  1. OrderService 코드 수정 → 관심사 분리 깨짐
  2. 호출하는 쪽 코드마다 수정 → OrderService 쓰는 곳이 100군데면?
  3. 상속으로 오버라이드 → 모든 메서드에 해야 함, final이면 불가
```

**근본 문제:** 호출자와 피호출자 사이에, 양쪽 코드를 건드리지 않고 끼어들 방법이 없다.

---

## 2. Mechanics

### 2-1. 전체 흐름 — Bean 생성부터 프록시까지

Spring이 Bean을 만드는 과정을 먼저 봐야 한다.

```
ApplicationContext 시작
        ↓
1. BeanDefinition 읽기
   (클래스 스캔, @Bean 메서드, XML 등)
        ↓
2. BeanDefinition 등록
   (BeanDefinitionRegistry)
        ↓
3. BeanFactoryPostProcessor 실행
   (BeanDefinition 메타데이터 수정 가능 시점)
        ↓
4. Bean 인스턴스 생성 (new)
        ↓
5. 의존성 주입 (@Autowired)
        ↓
6. BeanPostProcessor 실행   ← 프록시가 여기서 만들어진다
        ↓
7. 완성된 Bean (실제론 프록시) → BeanFactory에 등록
```

**`BeanPostProcessor`가 핵심이다.**

```java
// BeanPostProcessor 인터페이스
public interface BeanPostProcessor {
    
    // Bean 초기화 전 호출
    default Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;  // 기본: 그대로 반환
    }
    
    // Bean 초기화 후 호출  ← AOP 프록시가 여기서 생성됨
    default Object postProcessAfterInitialization(Object bean, String beanName) {
        return bean;  // 기본: 그대로 반환
        // AOP가 적용돼야 하면: 원본 bean 대신 proxy 반환
    }
}
```

`postProcessAfterInitialization`이 원본 Bean 대신 **프록시를 반환**하면, 컨테이너는 그 프록시를 등록한다. 이후 `@Autowired`로 주입받으면 프록시가 주입된다.

---

### 2-2. `AbstractAutoProxyCreator` — 실제 프록시 생성자

Spring AOP의 실제 진입점이다.

```java
// 상속 구조
AbstractAutoProxyCreator
    └── AbstractAdvisorAutoProxyCreator
            └── AnnotationAwareAspectJAutoProxyCreator  ← @EnableAspectJAutoProxy가 등록하는 것
```

```java
// AbstractAutoProxyCreator의 핵심 메서드
public abstract class AbstractAutoProxyCreator implements BeanPostProcessor {
    
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        
        // 1. 이 Bean에 적용할 Advisor(Pointcut + Advice)가 있는가?
        Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(
            bean.getClass(), beanName, null
        );
        
        // 2. 적용할 것이 있으면 프록시 생성
        if (specificInterceptors != DO_NOT_PROXY) {
            return createProxy(
                bean.getClass(),
                beanName,
                specificInterceptors,
                new SingletonTargetSource(bean)  // 원본 Bean을 감쌈
            );
        }
        
        // 3. 없으면 원본 그대로 반환
        return bean;
    }
}
```

**`getAdvicesAndAdvisorsForBean`이 하는 일:**

```java
// 단순화한 흐름
protected Object[] getAdvicesAndAdvisorsForBean(Class<?> beanClass, ...) {
    
    List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
    // 등록된 모든 @Aspect를 순회하면서
    // 각 @Pointcut 표현식이 이 Bean 클래스/메서드에 매칭되는지 확인
    
    if (advisors.isEmpty()) {
        return DO_NOT_PROXY;  // 프록시 불필요
    }
    return advisors.toArray();
}
```

---

### 2-3. JDK Dynamic Proxy vs CGLIB — 내부 구현 차이

프록시를 만드는 두 가지 방법이 있다. 어떤 걸 쓸지는 `DefaultAopProxyFactory`가 결정한다.

```java
public class DefaultAopProxyFactory implements AopProxyFactory {
    
    @Override
    public AopProxy createAopProxy(AdvisedSupport config) {
        
        // CGLIB을 선택하는 조건
        if (config.isOptimize()           // 최적화 플래그
            || config.isProxyTargetClass() // @EnableAspectJAutoProxy(proxyTargetClass=true)
            || hasNoUserSuppliedProxyInterfaces(config)) {  // 구현 인터페이스 없음
            
            Class<?> targetClass = config.getTargetClass();
            
            // 인터페이스 자체거나 Proxy 클래스면 JDK로
            if (targetClass.isInterface() 
                || Proxy.isProxyClass(targetClass)) {
                return new JdkDynamicAopProxy(config);
            }
            return new ObjenesisCglibAopProxy(config);  // CGLIB
        }
        
        return new JdkDynamicAopProxy(config);  // JDK
    }
}
```

---

### JDK Dynamic Proxy 작동 원리

Java 표준 API(`java.lang.reflect.Proxy`)를 사용한다.

```java
// JDK Proxy는 반드시 인터페이스 기반
public interface OrderService {
    void placeOrder(OrderRequest req);
}

@Service
public class OrderServiceImpl implements OrderService {
    public void placeOrder(OrderRequest req) { ... }
}
```

```java
// JdkDynamicAopProxy 내부 (단순화)
public class JdkDynamicAopProxy implements AopProxy, InvocationHandler {
    
    private final AdvisedSupport advised;  // 원본 Bean + Advisor 목록
    
    @Override
    public Object getProxy(ClassLoader classLoader) {
        
        // 핵심: Proxy.newProxyInstance
        // 런타임에 인터페이스를 구현하는 클래스를 바이트코드로 생성
        return Proxy.newProxyInstance(
            classLoader,
            new Class<?>[]{ OrderService.class },  // 구현할 인터페이스
            this  // InvocationHandler — 모든 메서드 호출이 여기로 들어옴
        );
    }
    
    // 프록시의 모든 메서드 호출이 여기로 위임됨
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        
        // 이 메서드에 적용할 인터셉터 체인 구성
        List<Object> chain = advised.getInterceptorsAndDynamicInterceptionAdvice(
            method, targetClass
        );
        
        if (chain.isEmpty()) {
            // 적용할 Advice 없으면 직접 호출
            return method.invoke(target, args);
        }
        
        // 인터셉터 체인 실행
        MethodInvocation invocation = new ReflectiveMethodInvocation(
            proxy, target, method, args, chain
        );
        return invocation.proceed();
    }
}
```

**JDK Proxy가 만드는 것:**

```
Proxy.newProxyInstance() 호출 시 JVM이 런타임에 아래를 생성:

public final class $Proxy50 implements OrderService {
    
    private InvocationHandler h;  // JdkDynamicAopProxy
    
    public void placeOrder(OrderRequest req) {
        // 모든 메서드가 InvocationHandler.invoke()로 위임
        h.invoke(this, placeOrder_Method, new Object[]{req});
    }
}
```

바이트코드가 **런타임에 메모리에서** 만들어진다. 디스크에 `.class` 파일이 생기지 않는다.

---

### CGLIB 작동 원리

인터페이스 없이 **클래스를 상속**해서 서브클래스를 생성한다.

```java
// 인터페이스 없는 클래스도 프록시 가능
@Service
public class OrderService {  // 인터페이스 없음
    
    public void placeOrder(OrderRequest req) { ... }
}
```

```
CGLIB이 생성하는 것:

public class OrderService$$SpringCGLIB$$0 extends OrderService {
    
    // 모든 public 메서드를 오버라이드
    @Override
    public void placeOrder(OrderRequest req) {
        // MethodInterceptor 체인 실행
        // → Advice들 순서대로 실행
        // → 중간에 원본 super.placeOrder(req) 호출
    }
}
```

**CGLIB 내부 핵심 — `MethodInterceptor`:**

```java
// CGLIB의 콜백 인터페이스
public interface MethodInterceptor extends Callback {
    Object intercept(
        Object obj,           // 프록시 객체
        Method method,        // 호출된 메서드
        Object[] args,        // 인자
        MethodProxy proxy     // 원본 메서드 호출용
    ) throws Throwable;
}

// Spring이 구현한 CGLIB 어댑터
class CglibMethodInvocation extends ReflectiveMethodInvocation {
    
    private final MethodProxy methodProxy;
    
    @Override
    public Object proceed() throws Throwable {
        // 인터셉터 체인 모두 통과하면 원본 메서드 호출
        // methodProxy.invoke() — reflection 없이 직접 호출
        // JDK Proxy의 Method.invoke()보다 빠름
        return methodProxy.invoke(this.target, this.arguments);
    }
}
```

---

### Advice 체인 실행 — `ReflectiveMethodInvocation`

JDK든 CGLIB든 Advice 체인 실행은 동일한 구조를 쓴다.

```java
public class ReflectiveMethodInvocation implements MethodInvocation {
    
    private final List<Object> interceptorsAndDynamicMethodMatchers;
    private int currentInterceptorIndex = -1;
    
    @Override
    public Object proceed() throws Throwable {
        
        // 모든 인터셉터 소진 → 실제 메서드 호출
        if (currentInterceptorIndex == interceptorsAndDynamicMethodMatchers.size() - 1) {
            return invokeJoinpoint();  // 원본 메서드
        }
        
        // 다음 인터셉터 꺼내서 실행
        Object interceptorOrInterceptionAdvice =
            interceptorsAndDynamicMethodMatchers.get(++currentInterceptorIndex);
        
        MethodInterceptor interceptor = (MethodInterceptor) interceptorOrInterceptionAdvice;
        
        // 인터셉터에게 this를 넘김
        // 인터셉터가 invocation.proceed() 호출하면 다시 이 메서드로
        // → Filter Chain과 동일한 재귀 구조
        return interceptor.invoke(this);
    }
}
```

**호출 스택:**

```
proceed() index=0 → TransactionInterceptor.invoke()
    proceed() index=1 → TimingInterceptor.invoke()
        proceed() index=2 (소진) → OrderService.placeOrder() 실행
            ↓ 반환
        TimingInterceptor 나머지 실행 (시간 측정)
    TransactionInterceptor 나머지 실행 (커밋/롤백)
```

Filter Chain과 구조가 동일하다. `pos` 카운터 대신 `currentInterceptorIndex`를 쓸 뿐이다.

---

### Self-invocation이 왜 안 되는가 — 구조로 설명

```java
@Service
public class OrderService {
    
    @Transactional
    public void placeOrder() {
        validateAndSave();  // 내부 호출
    }
    
    @Transactional(propagation = REQUIRES_NEW)
    public void validateAndSave() { ... }
}
```

```
컨테이너가 등록한 것:
  "orderService" → OrderService$$CGLIB (프록시)

외부에서 호출 시:
  controller → 프록시.placeOrder()
                   ↓
               TransactionInterceptor (트랜잭션 시작)
                   ↓
               target.placeOrder()  ← 원본 객체의 메서드
                   ↓
               this.validateAndSave()
               ↑
               여기서 this = 원본 OrderService 객체
               프록시가 아님
               → TransactionInterceptor 거치지 않음
               → REQUIRES_NEW 무시
```

```
메모리 구조:

BeanFactory
  "orderService" → [프록시 객체]
                        ↓ target 참조
                   [원본 OrderService 객체]
                   
원본 객체의 this는 원본 객체 자신
프록시를 가리키지 않음
```

**해결 방법들:**

```java
// 방법 1: 자기 자신을 프록시로 주입
@Service
public class OrderService {
    
    @Autowired
    private OrderService self;  // 주입되는 건 프록시
    
    public void placeOrder() {
        self.validateAndSave();  // 프록시를 통한 호출 → AOP 작동
    }
}

// 방법 2: ApplicationContext에서 직접 꺼내기 (비추)
@Service
public class OrderService {
    
    @Autowired
    private ApplicationContext ctx;
    
    public void placeOrder() {
        ctx.getBean(OrderService.class).validateAndSave();
    }
}

// 방법 3: 메서드를 별도 Bean으로 분리 (가장 권장)
@Service
public class OrderValidator {
    
    @Transactional(propagation = REQUIRES_NEW)
    public void validateAndSave() { ... }
}
```

---

### `final`이 왜 안 되는가

```java
// CGLIB은 상속으로 프록시를 만든다

public final class OrderService {  // final 클래스
    // CGLIB: "상속할 수 없음" → 프록시 생성 실패
}

public class OrderService {
    
    public final void placeOrder() {  // final 메서드
        // CGLIB: "오버라이드할 수 없음" → 이 메서드는 AOP 적용 안 됨
        // 프록시 생성은 되지만, 이 메서드는 원본 직접 호출
    }
}
```

```
Spring Boot에서 실제 발생하는 에러:
  Cannot subclass final class com.example.OrderService

Kotlin 주의:
  Kotlin의 클래스/메서드는 기본이 final
  → Spring Bean에 open 키워드 필요
  → 또는 kotlin-spring 컴파일러 플러그인 사용 (자동으로 open 처리)
```

---

## 3. 공식 근거

**BeanPostProcessor 메커니즘:**

- Spring Framework Reference — [Container Extension Points / BeanPostProcessor](https://docs.spring.io/spring-framework/reference/core/beans/factory-extension.html#beans-factory-extension-bpp):
    
    > "BeanPostProcessor instances operate on bean instances. That is, the Spring IoC container instantiates a bean instance and then BeanPostProcessor instances do their work."  
    > _(BeanPostProcessor는 Bean 인스턴스에 대해 동작한다. Spring IoC 컨테이너가 Bean 인스턴스를 만든 후 BeanPostProcessor가 작업을 수행한다.)_
    

**JDK vs CGLIB 선택 기준:**

- Spring Framework Reference — [AOP / Proxying Mechanisms](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html):
    
    > "If the target object to be proxied implements at least one interface, a JDK dynamic proxy is used. [...] If the target object does not implement any interfaces, a CGLIB proxy is created."  
    > _(프록시 대상이 하나 이상의 인터페이스를 구현하면 JDK 동적 프록시가 사용된다. 인터페이스를 구현하지 않으면 CGLIB 프록시가 생성된다.)_
    

**Self-invocation 제약:**

- Spring Framework Reference — [AOP / Understanding AOP Proxies](https://docs.spring.io/spring-framework/reference/core/aop/understanding-aop-proxies.html):
    
    > "This is a key aspect of Spring AOP. It means that self-invocation is not going to result in the advice associated with a method invocation getting a chance to execute."  
    > _(이것이 Spring AOP의 핵심 측면이다. Self-invocation은 메서드 호출에 연관된 Advice가 실행될 기회를 얻지 못한다는 것을 의미한다.)_
    

---

## 4. 이 설계를 이렇게 한 이유

### 왜 바이트코드 직접 조작(AspectJ weaving) 대신 프록시인가

```
AspectJ — compile-time / load-time weaving
  장점: self-invocation 문제 없음, final 제약 없음
  단점: 별도 컴파일러 필요, 빌드 파이프라인 복잡, Spring 컨테이너와 통합 복잡

Spring AOP 프록시
  장점: 표준 Java만으로 동작, Spring 컨테이너와 자연스럽게 통합
  단점: self-invocation 함정, final 제약, 인터페이스 or 상속 강제
```

Spring은 **"설정 없이 바로 작동하는"** 것을 우선했다. AspectJ weaving이 더 강력하지만, 진입 비용이 높다.

### 잃는 것 — 구체적으로

**1. 같은 클래스 내 메서드 호출에서 AOP 무시** `@Transactional` self-invocation 버그는 실무에서 꾸준히 발생한다. 컴파일 타임에 잡히지 않는다.

**2. CGLIB 프록시의 생성자 제약**

```java
// CGLIB은 서브클래스 생성 시 부모 생성자 호출
// 기본 생성자(no-arg) 없으면 에러 발생 가능
// Spring Boot + Objenesis로 완화됐지만 완전히 해결은 아님
public class OrderService {
    private final OrderRepository repo;
    
    // 기본 생성자 없음
    public OrderService(OrderRepository repo) {
        this.repo = repo;
    }
    // Objenesis가 처리해주지만, 환경에 따라 문제 발생 가능
}
```

**3. 프록시 타입 혼동**

```java
// JDK Proxy: 인터페이스 타입으로만 주입 가능
@Autowired
private OrderServiceImpl orderService;  // 실패 — 프록시는 OrderService 인터페이스 타입
// → NoSuchBeanDefinitionException or 타입 불일치

@Autowired
private OrderService orderService;  // 성공
```

---

## 5. 이어지는 개념

**1순위 — `@Transactional` 전파 레벨과 프록시 동작**

- **이유:** 방금 배운 프록시 구조 위에서 `REQUIRED`, `REQUIRES_NEW`, `NESTED`가 어떻게 동작하는지가 바로 연결된다. Self-invocation + 전파 레벨 조합은 실무 트랜잭션 버그의 대부분을 설명한다.

**2순위 — `BeanPostProcessor` 전체 목록과 실행 순서**

- **이유:** AOP 프록시 외에 `@Autowired` 처리(`AutowiredAnnotationBeanPostProcessor`), `@Value` 처리도 모두 BeanPostProcessor다. 이 순서를 이해하면 "왜 이 Bean에 AOP가 안 걸리지?" 같은 문제를 직접 추적할 수 있다.

**3순위 — AspectJ Pointcut 표현식 매칭 원리**

- **이유:** `execution(* com.example.service.*.*(..))` 이 표현식이 어떻게 클래스/메서드를 매칭하는지, 성능에 어떤 영향을 주는지. AOP가 적용된 서비스가 많아질수록 Pointcut 매칭 비용이 누적된다.