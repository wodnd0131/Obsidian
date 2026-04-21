#CS/JVM/Thread #Status/완료 

# 1. Problem First

TLAB이 없던 상황을 먼저 본다.

Eden Space는 모든 스레드가 공유하는 메모리 공간이다. 멀티스레드 환경에서 객체를 할당한다는 건 이렇게 된다.

```java
// 스레드 A, B, C가 동시에 new 를 호출하는 상황
// 내부적으로는 이런 경쟁이 발생

// Eden의 현재 끝 포인터 (bump pointer)
AtomicLong edenTop = new AtomicLong(EDEN_START);

// 스레드마다 이걸 해야 함
long myStart = edenTop.getAndAdd(objectSize); // CAS 연산
// → 모든 new 마다 CAS 경합 발생
```

`new` 키워드는 코드에서 굉장히 자주 호출된다. 루프 안, 스트림 중간 연산, 메서드 호출마다 생기는 임시 객체들 전부.

이게 매번 CAS(Compare-And-Swap) 경합을 유발하면 **객체 할당 자체가 병목**이 된다. 
할당이 느리면 처리량 전체가 떨어진다.

---

## 2. Mechanics

### 핵심 아이디어

Eden을 스레드마다 **미리 조각내서 나눠준다.**

```
Eden Space 전체
┌─────────────────────────────────────────────────┐
│  Thread-A TLAB  │  Thread-B TLAB  │  Thread-C TLAB  │  미할당  │
└─────────────────────────────────────────────────┘
```

각 스레드는 자기 TLAB 안에서만 할당한다. 자기 구역 안에서는 포인터를 앞으로 밀기만 하면 된다 
— **동기화 없음.**

### 할당 과정

```
스레드가 new 호출
        │
        ▼
   내 TLAB에 공간 있음?
   ┌────┴────┐
  YES        NO
   │          │
   ▼          ▼
bump pointer  JVM에 새 TLAB 요청
전진 (lock-free)  (이때만 동기화)
   │          │
   └────┬─────┘
        ▼
     객체 할당 완료
```

TLAB이 꽉 찼을 때 새 TLAB을 받는 작업만 동기화가 필요하다. 이 빈도는 객체 수 대비 훨씬 적다.

### 구조체 내부 (HotSpot 기준)

각 스레드의 TLAB은 세 포인터로 관리된다.

```
start ──────────────────── top ── end
  │                          │      │
  │◄──── 이미 할당된 영역 ────►│      │
                              │◄────►│
                           남은 공간
```

- `start` : 이 스레드의 TLAB 시작 주소
- `top` : 다음 객체가 할당될 위치 (bump pointer)
- `end` : TLAB 끝 주소

`new` 호출 시 JVM이 하는 일:

```
1. top + objectSize <= end  →  top을 objectSize만큼 전진, 완료
2. top + objectSize >  end  →  새 TLAB 요청 또는 Eden 직접 할당
```

### TLAB에 안 들어가는 경우

객체가 너무 크면 TLAB을 건너뛰고 Eden에 직접 할당한다. 그것도 안 되면 바로 Old Gen으로 간다.

```java
// 작은 객체 → TLAB 할당
String s = new String("hello");

// 큰 객체 → Eden 직접 또는 Old Gen 직행
byte[] buffer = new byte[1024 * 1024 * 50]; // 50MB
```

---

## 3. 공식 근거

**Oracle GC Tuning Guide**

> _"Each Java application thread has a thread-local allocation buffer (TLAB) in the young generation that it uses to allocate objects without synchronization."_ — [Oracle, HotSpot GC Tuning Guide](https://docs.oracle.com/en/java/se/21/gctuning/garbage-collector-implementation.html)
> 
> _각 Java 애플리케이션 스레드는 Young Generation 내에 스레드 로컬 할당 버퍼(TLAB)를 가지며, 동기화 없이 객체를 할당하는 데 사용한다._

**TLAB 크기 관련 JVM 옵션 (HotSpot)**

TLAB은 기본적으로 JVM이 자동 조정한다. 수동 제어도 가능하다.

```bash
-XX:TLABSize=<size>        # TLAB 초기 크기 직접 지정
-XX:+PrintTLAB             # TLAB 활동 로그 출력 (진단용)
-XX:-UseTLAB               # TLAB 비활성화 (테스트/진단 외엔 쓸 이유 없음)
```

이 옵션들의 존재 자체가 TLAB이 HotSpot의 공식 메커니즘임을 보여준다.

---

## 4. 트레이드오프

**TLAB 크기와 낭비의 관계**

TLAB이 크면 동기화 빈도가 줄지만, 각 스레드가 Eden을 많이 점유한다. 스레드 수가 많은 환경에서 TLAB이 크면 실제로 사용되지 않는 Eden 공간이 늘어난다.

```
스레드 100개 × TLAB 512KB = 50MB가 TLAB으로 예약
→ 그 중 실제 사용된 건 절반이라면 25MB 낭비
→ Eden이 작은 환경에서는 Minor GC가 더 자주 발생
```

그래서 JVM은 스레드 수, 할당 패턴을 보고 TLAB 크기를 동적으로 조정한다.

**TLAB은 Eden 안에 있다**

TLAB은 별도의 메모리 영역이 아니다. Eden의 일부를 스레드별로 예약한 것이다. Minor GC가 발생하면 TLAB도 Eden과 함께 수거 대상이 된다. TLAB 자체는 GC 알고리즘과 무관하고, 순수하게 **할당 성능**을 위한 최적화다.

---

## 한 줄 정리

> Eden을 스레드별로 미리 나눠줘서, `new` 호출 시 동기화 없이 포인터만 밀면 되게 만든 할당 최적화다. GC와는 무관하고, 할당 속도 문제를 해결하기 위한 구조다.