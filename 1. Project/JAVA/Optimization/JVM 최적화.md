#Optimization/JVM #Status/보충
### JIT 컴파일과 인라이닝(Inlining)

[[../CS/JVM/JVM]] 은 처음엔 바이트코드를 인터프리터로 실행하다가, 자주 호출되는 코드(Hot Spot)를 감지하면 JIT(Just-In-Time) 컴파일러가 네이티브 코드로 변환한다.

**메서드 인라이닝**이란 메서드 호출을 호출 자체를 없애고 본문을 호출 지점에 직접 삽입하는 최적화다.

```java
// 원본 코드
int result = calculator.add(a, b);

// JIT가 인라이닝하면 실질적으로 이렇게 동작
int result = a + b;  // 메서드 호출 오버헤드 제거
```

호출 오버헤드(스택 프레임 생성, 파라미터 복사, 리턴 주소 저장)가 사라지므로 반복 호출 시 성능 차이가 유의미하다. **따라서 `calculator.add(a,b)` vs `a.add(b)` 자체보다, 해당 메서드가 인라이닝 가능한 구조인지가 성능에 더 직접적인 영향을 준다.**

### Escape Analysis & 스택 할당

```java
// 매번 Number 객체 생성 — 기본적으로 힙 할당
public Number add(Number other) {
    return new Number(this.value + other.value);
}
```

JVM은 이 `new Number(...)`가 메서드 밖으로 탈출(escape)하지 않는다고 판단하면, 힙 대신 **스택에 할당**한다. 스택 할당은 GC 대상이 아니므로 GC 압박이 줄어든다.

즉, 불변 객체라서 매번 `new`를 써도 "GC가 폭발한다"는 건 단순화된 얘기다. Escape Analysis가 잘 작동하면 오버헤드가 거의 없을 수 있다.

### Primitive vs Wrapper 성능 차이

|      | primitive `int` | `Integer` (wrapper) |
| ---- | --------------- | ------------------- |
| 메모리  | 4 bytes         | 16 bytes (헤더 포함)    |
| 연산   | CPU 직접 연산       | unboxing 후 연산       |
| 배열   | 연속 메모리          | 포인터 배열 (캐시 미스)      |
| null | 불가              | 가능                  |

`List<Integer>`는 내부적으로 오토박싱/언박싱이 발생한다. 성능이 중요한 루프에서는 `int[]`가 월등히 빠르다. 하지만 **일반 애플리케이션에서 이 차이가 병목이 되는 경우는 드물다.** 먼저 설계를 바르게 하고, 프로파일링 후 최적화하는 게 맞는 순서다.

### 정리: 실제로 성능에 영향 주는 요소

- 메서드 형태(`a.add(b)` vs `calculator.add(a,b)`)보다 **불변 객체 생성 빈도**, **GC 압박**, **캐시 지역성**이 훨씬 크다
- JIT와 Escape Analysis가 많은 부분을 자동 최적화해준다
- 성능 문제는 **추측하지 말고 측정(profiling)하라**

---
