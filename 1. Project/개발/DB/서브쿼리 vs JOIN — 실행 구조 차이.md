
---

## 서브쿼리가 "매 row마다 반복"되는 이유

```sql
-- 이 형태가 문제의 원인
SELECT * FROM orders o
WHERE o.user_id IN (
    SELECT id FROM users WHERE status = 'ACTIVE'  -- ← 이게 언제 실행되나?
);
```

SQL은 선언형이라 실행 순서를 명시하지 않는다. 옵티마이저가 변환하기 전, **나이브한 실행 엔진이 그대로 해석하면** 이렇게 된다.

```
orders 테이블을 한 row씩 읽는다
    └─ 이 row의 user_id가 ACTIVE 유저인지 확인하려면?
           └─ SELECT id FROM users WHERE status = 'ACTIVE' 를 실행해서 결과와 비교
    └─ 다음 row... 또 서브쿼리 실행
    └─ 다음 row... 또 서브쿼리 실행
    ...
```

orders가 1,000만 건이면 서브쿼리가 **1,000만 번** 실행된다. 이걸 **Correlated Subquery(상관 서브쿼리)** 라고 부른다.

---

## "JPA 1차 캐시처럼 기억 못 하나?" — 정확한 질문이다

결론: **기억할 수도 있고, 못 할 수도 있다. 서브쿼리 종류에 따라 다르다.**

### 서브쿼리 두 종류

```sql
-- ① 비상관 서브쿼리 (Non-Correlated)
SELECT * FROM orders
WHERE user_id IN (
    SELECT id FROM users WHERE status = 'ACTIVE'
    -- 바깥 테이블(orders)을 전혀 참조하지 않음
);

-- ② 상관 서브쿼리 (Correlated)
SELECT * FROM orders o
WHERE EXISTS (
    SELECT 1 FROM users u
    WHERE u.id = o.user_id        -- ← 바깥 테이블의 o.user_id를 참조
      AND u.status = 'ACTIVE'
);
```

|구분|바깥 테이블 참조|캐시 가능 여부|반복 실행 여부|
|---|---|---|---|
|비상관|❌|✅ 한 번만 실행 후 결과 재사용|❌|
|상관|✅|❌ 값이 row마다 바뀜|✅ 매 row마다|

**비상관 서브쿼리는 왜 한 번만 실행 가능한가:**

```
SELECT id FROM users WHERE status = 'ACTIVE'
→ 이 결과는 orders의 어떤 row를 보든 항상 같다
→ 한 번 실행해서 결과셋을 메모리에 올려두면 끝
→ (1, 5, 23, 78, ...) 같은 ID 목록을 해시 테이블로 만들어두고 비교만
```

**상관 서브쿼리는 왜 캐시 불가능한가:**

```
WHERE u.id = o.user_id
→ o.user_id 값이 row마다 다르다
→ orders의 현재 row를 보기 전까지 서브쿼리의 결과를 미리 알 수 없다
→ 매 row마다 다른 값을 들고 users를 조회해야 함
```

---

## 그러면 JOIN은 구조적으로 뭐가 다른가

```sql
-- JOIN으로 변환된 형태
SELECT DISTINCT o.*
FROM orders o
INNER JOIN users u ON o.user_id = u.id
WHERE u.status = 'ACTIVE';
```

JOIN은 **두 테이블을 동등한 입장으로 놓고, 옵티마이저가 실행 순서를 자유롭게 결정**할 수 있다.

```
옵티마이저 선택지:

[A] users를 먼저 읽고 (status='ACTIVE' 필터 후 소수만 남김)
    → 그 결과로 orders를 조회
    → users 결과가 작으면 매우 효율적

[B] orders를 먼저 읽고
    → 각 user_id로 users를 조회
    → 거의 Nested Loop와 동일하지만 Hash Join 선택 가능

[C] 두 테이블 동시에 Hash Join
    → users 전체를 해시 테이블로 메모리에 올림
    → orders를 스캔하며 해시 테이블에서 O(1) 조회
```

서브쿼리는 **바깥 쿼리가 드라이버** 라는 구조가 고정되어 있다. JOIN은 **어느 테이블이 드라이버가 될지 옵티마이저가 선택**한다.

---

## "조건 푸시다운처럼 별개 SQL로 분리시키는 건 아니잖아?" — 이것도 맞다

푸시다운은 **논리적 변환** (쿼리 구조를 바꿈) 이고, 서브쿼리 → JOIN 변환도 **논리적 변환** 이다.

둘 다 별도 SQL로 쪼개는 게 아니라, **옵티마이저가 내부적으로 실행 계획을 만들기 전에 쿼리 트리를 재작성**한다.

```
원본 쿼리 AST (추상 구문 트리)
        │
        ▼
   논리적 변환 단계
   ┌─────────────────────────────────┐
   │ ① 상수 폴딩                     │
   │ ② 조건 푸시다운                  │
   │ ③ 서브쿼리 → semi-join 변환     │  ← 여기
   │ ④ 불필요 조건 제거              │
   └─────────────────────────────────┘
        │
        ▼
   변환된 쿼리 트리 → 비용 계산 → 실행 계획
```

변환 결과로 나오는 건 `INNER JOIN`이 아니라 정확히는 **Semi-join** 이다.

```sql
-- Semi-join: JOIN이지만 중복 제거가 내장된 형태
-- users에 매칭되는 orders row를 찾되,
-- 같은 user_id가 여러 번 매칭돼도 orders row는 한 번만 반환

-- 옵티마이저 내부 표현 (실제 SQL로는 쓸 수 없음)
SELECT o.* FROM orders o
WHERE SEMI JOIN users u ON u.id = o.user_id AND u.status = 'ACTIVE';
```

`EXPLAIN`에서 `SELECT_TYPE = SUBQUERY` 대신 `SIMPLE`로 바뀌어 있으면 옵티마이저가 성공적으로 semi-join으로 변환한 것이다.

---

## 한 줄 정리

```
상관 서브쿼리  →  바깥 row 수만큼 반복 실행 (캐시 불가, 구조적으로 고정)
비상관 서브쿼리 →  한 번 실행 후 결과 재사용 가능 (하지만 옵티마이저 자유도 낮음)
JOIN           →  어느 테이블을 드라이버로 쓸지 옵티마이저가 결정 (최적화 자유도 최대)
```

옵티마이저가 서브쿼리를 semi-join으로 변환하는 이유가 바로 이 **"최적화 자유도"** 를 얻기 위해서다.