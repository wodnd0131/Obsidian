#CS/CPU #CS/JVM/Thread #Status/완료 
## 1. Problem First
멀티스레드 환경에서 공유 변수를 수정하는 가장 단순한 방법은 락이다.

```java
// 락 기반 카운터
public class Counter {
    private long count = 0;
    
    public synchronized void increment() {
        count++; // 한 번에 하나의 스레드만 진입
    }
}
```

`count++`는 사실 세 단계다.

```
1. count 읽기  (load)
2. +1 계산     (add)
3. count 쓰기  (store)
```

이 세 단계 사이에 다른 스레드가 끼어들면 결과가 깨진다. `synchronized`는 이 구간을 통째로 잠가서 해결한다.

문제는 락의 비용이다.

```
락 획득 시도
    │
    ├── 락이 비어있음 → OS 개입 없이 진행 (빠름)
    │
    └── 락이 잡혀있음 → 스레드를 블로킹
                         OS가 컨텍스트 스위치
                         락 해제될 때까지 대기
                         다시 스케줄링
                         → 수 마이크로초 ~ 수백 마이크로초 비용
```

`new` 호출처럼 극도로 자주 일어나는 연산에 이 비용을 매번 지불하면 처리량이 떨어진다.

---

## 2. Mechanics

### CPU 레벨에서의 CAS

CAS는 소프트웨어가 아니라 **CPU가 제공하는 단일 원자적 명령어**다.

```
x86:  LOCK CMPXCHG
ARM:  LDXR / STXR (Load-Exclusive / Store-Exclusive)
```

```
CAS(메모리주소, 기대값, 새값)

의사코드로 표현하면:
if (*주소 == 기대값) {
    *주소 = 새값;
    return true;
} else {
    return false;
}

단, 이 전체가 CPU 레벨에서 분리 불가능한 하나의 연산이다.
```

"분리 불가능"하다는 게 핵심이다. 
읽기-비교-쓰기 사이에 다른 CPU 코어가 끼어들 수 없다. OS를 거치지 않으니 컨텍스트 스위치도 없다.

### Java에서의 CAS

Java는 `sun.misc.Unsafe`를 통해 CPU의 CAS 명령어에 직접 접근한다. 
`AtomicLong`, `AtomicInteger` 등이 내부적으로 이걸 쓴다.

```java
// AtomicLong 내부 (OpenJDK 소스 기준)
public class AtomicLong {
    private volatile long value;  // volatile: 캐시가 아닌 메인 메모리에서 읽기

    public final long getAndIncrement() {
        return U.getAndAddLong(this, VALUE, 1L);
        // U = Unsafe 인스턴스
        // 내부적으로 LOCK CMPXCHG 명령어로 컴파일됨
    }
}
```

### CAS의 동작 패턴 — CAS Loop

CAS 실패 시 재시도하는 패턴이 일반적이다.

```java
// CAS Loop 패턴
long oldValue, newValue;
do {
    oldValue = top.get();           // 현재값 읽기
    newValue = oldValue + objectSize; // 새값 계산
} while (!top.compareAndSet(oldValue, newValue)); // 실패하면 재시도
```

```
스레드 A: read(100) → 계산(132) → CAS(100→132) 성공
스레드 B: read(100) → 계산(132) → CAS(100→132) 실패 (이미 132)
          read(132) → 계산(164) → CAS(132→164) 성공
```

### CAS의 한계 — ABA 문제

```
스레드 A: 값을 읽음 → 100
스레드 B: 100 → 200 → 100으로 두 번 바꿈
스레드 A: CAS(100 → 132) 성공
          → 값이 100이니 아무도 안 건드린 것처럼 보이지만
            실제로는 중간에 변경이 있었음
```

값은 같아도 **중간에 변경이 있었다는 사실**을 CAS는 감지하지 못한다. 
Java에서는 `AtomicStampedReference`로 버전 번호를 함께 관리해서 해결한다.

### CAS vs Lock 선택 기준

| 상황                | 적합한 방식                      |
| ----------------- | --------------------------- |
| 경합이 낮음 (스레드 수 적음) | CAS 유리 — 재시도 비용 낮음          |
| 경합이 높음 (스레드 수 많음) | Lock 유리 — CAS 재시도가 폭발적으로 증가 |
| 단순 카운터/포인터 조작     | CAS                         |
| 복잡한 복합 연산 (여러 변수) | Lock                        |

TLAB이 CAS를 쓰지 않는 이유가 여기 있다.
 스레드 수가 늘수록 Eden top 포인터에 대한 CAS 경합이 선형으로 늘어난다. 
 TLAB은 이 경합 자체를 없애는 방식으로 해결한 것이다.

---

## 3. 공식 근거

**JEP 193 — Variable Handles (Java 9)**

> _"The existing mechanisms, such as `sun.misc.Unsafe`, are unsafe operations that bypass Java's type safety."_ — [JEP 193](https://openjdk.org/jeps/193)
> 
> _`sun.misc.Unsafe` 같은 기존 메커니즘은 Java의 타입 안전성을 우회하는 안전하지 않은 연산이다._

Java 9부터 `VarHandle`이 공식 대안으로 도입됐다. `AtomicLong` 등도 Java 9 이후 내부적으로 `VarHandle`로 전환됐다.

**Java Concurrency in Practice — Brian Goetz**

> _"CAS-based counters significantly outperform lock-based counters under low to moderate contention."_
> 
> _낮은~보통 경합 수준에서 CAS 기반 카운터는 락 기반 카운터보다 성능이 크게 우수하다._

---

## 한 줄 정리

>  OS를 거치지 않는 CPU 레벨의 원자적 연산. 락보다 가볍지만 경합이 높으면 오히려 불리하다.
