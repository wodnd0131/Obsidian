# 옵티마이저가 선택하는 JOIN 알고리즘

---

## 1. Problem First — 왜 JOIN 알고리즘이 여러 개인가

```sql
-- 이 쿼리 하나를 실행하는 방법이 몇 가지인가?
SELECT o.id, u.name
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.status = 'ACTIVE';
```

JOIN의 본질은 **두 집합의 교차 검사**다. `orders`의 모든 row와 `users`의 모든 row를 비교해서 조건이 맞는 쌍을 찾아야 한다.

문제는 테이블이 수백만 건일 때 **모든 쌍을 다 비교하면 O(N×M)** 이라는 것이다.

```
orders: 1,000만 건
users:  100만 건
모든 쌍: 10조 번 비교 → 불가능
```

그래서 "어떻게 하면 비교 횟수를 줄일 수 있는가"라는 문제에 대한 각기 다른 해법이 **JOIN 알고리즘**이다.

---

## 2. Mechanics — JOIN 알고리즘 3종

MySQL InnoDB 기준으로 옵티마이저가 선택할 수 있는 알고리즘은 세 가지다.

```
Nested Loop Join  (NLJ)   — MySQL 전통적 기본
Block Nested Loop (BNL)   — MySQL 8.0.18 이전 fallback
Hash Join         (HJ)    — MySQL 8.0.18부터 정식 도입
```

Semi-join은 알고리즘이 아니라 **최적화 전략**이다. 뒤에서 별도로 다룬다.

---

### ① Nested Loop Join (NLJ)
중첩 루프 조인

**구조:**

```
for each row r1 in 외부 테이블(드라이버):
    for each row r2 in 내부 테이블:
        if r1.join_key == r2.join_key:
            결과에 추가
```

**실제 동작:**

```
orders (드라이버, 외부)
  └─ row 1: user_id = 5
       └─ users 인덱스에서 id = 5 조회  ← 인덱스 랜덤 I/O
  └─ row 2: user_id = 23
       └─ users 인덱스에서 id = 23 조회
  └─ row 3: ...
```

**핵심 전제 조건: 내부 테이블(users)에 인덱스가 있어야 한다.**

```
인덱스 있음: 내부 테이블 조회가 O(log N) → 전체 O(M × log N)
인덱스 없음: 내부 테이블 풀스캔 → 전체 O(M × N) → 재앙
```

**EXPLAIN에서 보이는 모습:**

```sql
EXPLAIN SELECT o.id, u.name
FROM orders o JOIN users u ON o.user_id = u.id;

-- id=1, table=o, type=ALL  (드라이버, 풀스캔)
-- id=1, table=u, type=eq_ref  ← NLJ의 시그니처
--                  key=PRIMARY, rows=1
```

`eq_ref` = "드라이버 테이블의 각 row마다 내부 테이블에서 PK/유니크 인덱스로 정확히 1건 조회" 이게 NLJ가 가장 이상적으로 동작하는 상태다.

**강점 / 약점:**

```
✅ 인덱스만 있으면 매우 빠름
✅ 드라이버 테이블이 작을수록 유리
✅ 첫 결과가 빨리 나옴 (스트리밍 가능)
✅ 추가 메모리 거의 불필요

❌ 내부 테이블 인덱스 없으면 최악
❌ 드라이버가 클수록 랜덤 I/O가 선형 증가
❌ 내부 테이블이 크고 인덱스 선택도 낮으면 풀스캔보다 느릴 수 있음
```

---

### ② Block Nested Loop Join (BNL) — MySQL 8.0.18 이전

**NLJ의 문제:**

```
orders 1,000만 건 × users 랜덤 I/O 1,000만 번
→ 디스크 랜덤 I/O가 병목
→ 내부 테이블을 매번 처음부터 읽는 비효율
```

**BNL의 해법: 드라이버 테이블을 청크로 잘라서 배치 처리**

```
join_buffer (기본 256KB)에 드라이버 row를 최대한 채움
    └─ 예: orders 1,000건이 버퍼에 들어감

내부 테이블(users) 풀스캔 1번
    └─ 각 users row를 버퍼의 1,000건과 한 번에 비교

버퍼 비우고 orders 다음 1,000건 채움
    └─ users 풀스캔 또 1번
    ...
```

```
NLJ 인덱스 없음:  users 풀스캔 × orders 행 수  = 1,000만 번
BNL:              users 풀스캔 × (orders / 버퍼 크기) = 1만 번
```

**EXPLAIN에서 보이는 모습:**

```
Extra: Using join buffer (Block Nested Loop)
```

이게 보이면 **인덱스가 없어서 차선책을 쓰고 있다는 신호**다. 인덱스를 추가하거나 `join_buffer_size`를 키우거나, Hash Join으로 유도해야 한다.

**MySQL 8.0.18 이후:** BNL은 Hash Join으로 대체됐다. 동일한 상황에서 옵티마이저가 자동으로 Hash Join을 선택한다.

"어차피 users를 풀스캔해야 한다면, 한 번 읽을 때 드라이버 테이블의 여러 행을 동시에 비교해서 풀스캔 횟수 자체를 줄이자"

---

### ③ Hash Join (MySQL 8.0.18+)

**구조:**

```
Phase 1 - Build:
    작은 테이블(users)을 전부 읽어서
    join key(id)를 해시 함수에 통과시켜
    메모리 해시 테이블 구축

Phase 2 - Probe:
    큰 테이블(orders)을 스캔하면서
    각 row의 user_id를 해시 함수에 통과시켜
    해시 테이블에서 O(1)로 매칭 확인
```

**시각화:**

```
Build Phase:
users → [id=1, name="Alice"] → hash(1) = 버킷 A에 저장
        [id=5, name="Bob"]   → hash(5) = 버킷 C에 저장
        ...

해시 테이블 (메모리):
버킷 A: [{id:1, name:"Alice", status:"ACTIVE"}]
버킷 C: [{id:5, name:"Bob",   status:"ACTIVE"}]
...

Probe Phase:
orders row: user_id=5 → hash(5) → 버킷 C 조회 → "Bob" 매칭 ✅
orders row: user_id=1 → hash(1) → 버킷 A 조회 → "Alice" 매칭 ✅
orders row: user_id=9 → hash(9) → 버킷 F 조회 → 없음 ❌
```

**복잡도:**

```
Build:  O(S)       — 작은 테이블 크기
Probe:  O(L)       — 큰 테이블 크기
전체:   O(S + L)   ← NLJ O(M × log N) 보다 유리한 경우 많음
```

**인덱스가 없어도 된다**는 게 핵심이다.

**메모리 초과 시 (Grace Hash Join):**

해시 테이블이 `join_buffer_size`를 초과하면 디스크로 spill한다.

```
users를 여러 파티션으로 나눠 디스크에 씀
    파티션 1 → 메모리에 올려서 처리
    파티션 2 → 메모리에 올려서 처리
    ...
```

성능이 떨어지므로 `join_buffer_size` 조정이 필요할 수 있다.

**EXPLAIN에서 보이는 모습:**

```
Extra: Using join buffer (hash join)  -- 8.0.18+
```

**강점 / 약점:**

```
✅ 인덱스 없어도 효율적
✅ 등치 조건(=)에서 NLJ보다 빠른 경우 많음
✅ 큰 테이블 × 큰 테이블 조인에서 강함

❌ Build Phase에서 작은 테이블 전체를 메모리에 올려야 함
❌ 메모리 부족 시 디스크 spill → 성능 급락
❌ 첫 결과가 Build Phase 끝난 후에야 나옴 (스트리밍 불가)
❌ 범위 조건(>, <, BETWEEN)에서는 사용 불가 — 등치만 가능
```

---

### 세 알고리즘 비교

|               | NLJ        | BNL            | Hash Join          |
| ------------- | ---------- | -------------- | ------------------ |
| 인덱스 필요        | ✅ 필수       | ❌              | ❌                  |
| 메모리 사용        | 최소         | join_buffer    | join_buffer (더 많이) |
| 등치 조건         | ✅          | ✅              | ✅                  |
| 범위 조건         | ✅          | ✅              | ❌                  |
| 작은 드라이버       | 최강         | 보통             | 보통                 |
| 큰 테이블 × 큰 테이블 | 인덱스 없으면 최악 | 나쁨             | 강함                 |
| MySQL 버전      | 전통         | ~8.0.17 (더 안씀) | 8.0.18+            |

---

## 3. Semi-join — 알고리즘이 아닌 최적화 전략

앞에서 말했듯 Semi-join은 JOIN 알고리즘이 아니다. **"IN/EXISTS 서브쿼리를 어떻게 실행할 것인가"에 대한 최적화 전략**이다.

```sql
-- 이 패턴에 적용됨
SELECT * FROM orders
WHERE user_id IN (SELECT id FROM users WHERE status = 'ACTIVE');
```

**일반 JOIN과의 차이:**

```sql
-- 일반 INNER JOIN: users에 매칭되는 orders row를 모두 반환
-- users 1명에 orders 10건이면 10건 모두 반환
SELECT o.* FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.status = 'ACTIVE';

-- Semi-join이 원하는 것: orders row가 매칭 여부만 확인
-- users 1명에 orders 10건이어도 orders는 10건 모두 반환 (중복 없이)
-- 즉, "users에 존재하는가"만 확인하고 users 데이터는 결과에 포함 안 함
```

**Semi-join 실행 전략 (옵티마이저가 선택):**

```
① Duplicate Weedout
   → 일반 JOIN으로 실행 후 임시 테이블로 중복 제거
   → 가장 범용적, 메모리 사용

② FirstMatch
   → 내부 테이블에서 첫 번째 매칭만 찾으면 즉시 다음으로
   → 중복 제거 불필요, 빠름
   → orders row당 users에서 1건만 확인하면 되므로 NLJ와 유사

③ LooseScan
   → 인덱스를 활용해 중복된 join key를 건너뜀
   → 인덱스 있을 때 매우 효율적

④ Materialization
   → 서브쿼리 결과를 임시 테이블(해시 테이블)로 물질화
   → 이후 Hash Join처럼 O(1) 조회
   → 비상관 서브쿼리에서 효과적
```

**EXPLAIN에서 보이는 모습:**

```sql
EXPLAIN SELECT * FROM orders
WHERE user_id IN (SELECT id FROM users WHERE status = 'ACTIVE');

-- Semi-join 변환 성공 시:
-- select_type = SIMPLE  (SUBQUERY가 아님)
-- Extra = Start temporary / End temporary  → Duplicate Weedout
-- Extra = FirstMatch(users)               → FirstMatch
-- Extra = LooseScan                       → LooseScan
-- select_type = MATERIALIZED              → Materialization
```

---

## 4. 옵티마이저가 선택하는 기준 — 한 장 정리

```
JOIN 조건이 등치(=)인가?
├─ NO  (범위, >, <)
│   └─ NLJ만 가능 (인덱스 있을 때)
│      인덱스 없으면 BNL/풀스캔
│
└─ YES
    ├─ 내부 테이블에 인덱스 있는가?
    │   ├─ YES + 드라이버가 충분히 작음
    │   │   └─ NLJ 선택 (eq_ref / ref)
    │   │
    │   └─ 인덱스 있지만 선택도 낮음 / 드라이버가 큰 경우
    │       └─ Hash Join이 유리할 수 있음
    │
    └─ 내부 테이블에 인덱스 없음
        └─ Hash Join (8.0.18+)
           메모리 부족 시 → Grace Hash Join (디스크 spill)
```

**실무에서 판단 기준:**

```sql
-- NLJ가 맞는 상황
-- 드라이버 작고, 내부 테이블 PK/유니크 인덱스 조인
SELECT o.id FROM orders o          -- 10건짜리 최근 주문
JOIN users u ON o.user_id = u.id;  -- PK 조인 → eq_ref, NLJ 최적

-- Hash Join이 맞는 상황
-- 인덱스 없는 컬럼 조인, 또는 두 테이블 모두 대용량
SELECT a.id FROM table_a a
JOIN table_b b ON a.category = b.category;  -- category에 인덱스 없음
-- → Hash Join, join_buffer_size 충분히 확보 필요
```

---

## 5. 공식 근거

- **MySQL 8.0 — Hash Join Optimization**: [https://dev.mysql.com/doc/refman/8.0/en/hash-joins.html](https://dev.mysql.com/doc/refman/8.0/en/hash-joins.html)
    
    > "Hash joins are the default algorithm used when possible for joins that use only equi-join conditions." (번역: 등치 조인 조건만 사용하는 경우 가능하면 해시 조인이 기본 알고리즘으로 사용된다.)
    
- **MySQL 8.0 — Semi-join Optimization**: [https://dev.mysql.com/doc/refman/8.0/en/semijoins.html](https://dev.mysql.com/doc/refman/8.0/en/semijoins.html)
    
- **MySQL 8.0 — Nested Loop Join**: [https://dev.mysql.com/doc/refman/8.0/en/nested-loop-joins.html](https://dev.mysql.com/doc/refman/8.0/en/nested-loop-joins.html)
    
- _High Performance MySQL 4th Ed._ — "Join Execution Strategy" 챕터 (3순위)
    

---

## 이어지는 개념

|순서|개념|이유|
|---|---|---|
|1|**join_buffer_size 튜닝**|Hash Join 성능이 이 값에 직결됨. 지금 당장 실무 적용 가능|
|2|**Index Merge**|단일 테이블에서 여러 인덱스를 조합하는 최적화. EXPLAIN에서 자주 마주침|
|3|**Derived Table / CTE 물질화**|WITH절, 서브쿼리가 어떻게 임시 테이블로 바뀌는지 — Semi-join Materialization의 연장선|