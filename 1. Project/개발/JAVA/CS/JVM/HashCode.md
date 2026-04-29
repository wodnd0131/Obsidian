좋은 키워드야. 들어간다.

---

## 1. Problem First

```java
class LottoNumber {
    private final int value;
    
    LottoNumber(int value) { this.value = value; }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof LottoNumber)) return false;
        return this.value == ((LottoNumber) o).value;
    }
    // hashCode 미정의
}

LottoNumber a = new LottoNumber(7);
LottoNumber b = new LottoNumber(7);

System.out.println(a.equals(b));          // true  — 기대한 대로
System.out.println(a.hashCode() == b.hashCode()); // false — Object 기본값, 주소 기반

Set<LottoNumber> set = new HashSet<>();
set.add(a);
set.contains(b);  // false ← 버그
```

`equals`가 `true`인데 `HashSet`에서 못 찾는다. 이게 핵심 고통이야.

---

## 2. Mechanics

### hashCode가 뭔가

JVM에서 모든 객체는 생성 시 **identity hash code**를 가져. `Object.hashCode()`의 기본 구현이 이걸 반환해.

OpenJDK 기준으로 기본 `hashCode`는 객체의 메모리 주소를 기반으로 계산돼 (정확히는 GC 이후 주소가 바뀔 수 있어서 최초 계산 값을 object header에 캐싱함).

```
Object header (64bit JVM)
┌─────────────────────────────┐
│  mark word (64bit)          │
│  ├─ identity hash code      │ ← hashCode() 최초 호출 시 여기 저장
│  ├─ GC age                  │
│  └─ lock state              │
└─────────────────────────────┘
```

### HashMap/HashSet이 어떻게 작동하는가

```
put(key, value) 흐름:
1. key.hashCode() 호출
2. hash값으로 버킷(배열 인덱스) 결정
   index = (n-1) & hash
3. 해당 버킷에서 key.equals()로 동일 키 탐색
4. 없으면 삽입, 있으면 교체
```

**hashCode → 어느 버킷인가 (검색 공간 축소)** **equals → 같은 버킷 안에서 진짜 같은 객체인가 (정확한 비교)**

두 단계가 모두 통과해야 "같은 키"로 취급해. `hashCode`가 다르면 버킷 자체가 달라서 `equals`를 아예 호출 안 해.

### equals만 재정의하면 무슨 일이 벌어지나

```
a = LottoNumber(7), hashCode = 0x1a2b (주소 기반)
b = LottoNumber(7), hashCode = 0x3c4d (다른 주소)

HashSet 내부:
버킷[0x1a2b % n] → [a]
버킷[0x3c4d % n] → (비어있음)

set.contains(b):
  1. b.hashCode() = 0x3c4d → 버킷[0x3c4d % n] 탐색
  2. 비어있음 → false 반환
  (a.equals(b)를 호출조차 안 함)
```

---

## 3. 공식 근거

**JLS / Java API 계약:**

> "If two objects are equal according to the `equals(Object)` method, then calling the `hashCode` method on each of the two objects must produce the same integer result."
> 
> _— `Object.hashCode()` Javadoc, Java SE 21_
> 
> (번역: `equals`로 동등한 두 객체는 반드시 같은 `hashCode`를 반환해야 한다.)

이건 **명세(contract)** 야. 위반하면 JDK 컬렉션의 동작 보장이 깨져.

**Effective Java:**

> "You must override `hashCode` in every class that overrides `equals`. If you fail to do so, your class will violate the general contract for `hashCode`, and your class will not function properly in conjunction with all hash-based collections."
> 
> _— Effective Java 3rd, Item 11_
> 
> (번역: `equals`를 재정의한 모든 클래스에서 반드시 `hashCode`도 재정의해야 한다. 그렇지 않으면 해시 기반 컬렉션에서 정상 동작을 보장할 수 없다.)

---

## 4. 트레이드오프

**hashCode 구현 시 충돌(collision) 문제**

```java
// 극단적 예시 — 계약은 만족하지만 성능 최악
@Override
public int hashCode() { return 42; }
```

이렇게 하면 모든 객체가 같은 버킷에 몰려 `HashMap`이 O(1)이 아니라 O(n) 링크드 리스트 탐색이 돼. 계약은 지켰지만 해시 컬렉션의 의미가 없어짐.

**좋은 hashCode의 조건:**

- 같은 객체는 항상 같은 값 (결정론적)
- equals가 true인 객체는 반드시 같은 값
- 분포가 고를 것 (충돌 최소화)

**Objects.hash() vs 직접 구현:**

```java
// 편의 — 내부적으로 Arrays.hashCode 위임, 오토박싱 비용 있음
@Override
public int hashCode() {
    return Objects.hash(value);
}

// 성능 중요할 때
@Override
public int hashCode() {
    return Integer.hashCode(value);
}
```

primitive 필드 하나짜리 VO라면 직접 구현이 낫고, 필드 많으면 `Objects.hash()`가 실수 줄여줘.