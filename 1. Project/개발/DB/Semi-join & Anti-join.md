

---

## 1. Problem First — 왜 이 개념이 필요한가

```sql
-- 요구사항: ACTIVE 유저가 주문한 orders만 조회
SELECT * FROM orders
JOIN users u ON orders.user_id = u.id
WHERE u.status = 'ACTIVE';
```

INNER JOIN으로 짜면 생기는 문제:

```
users 1명이 orders 10건을 가지고 있으면
→ 같은 orders row가 10번 중복 반환될 수 있음

user_id=1, orders 10건:
[order_id=1, user_id=1, name="Alice"]
[order_id=2, user_id=1, name="Alice"]
...
[order_id=10, user_id=1, name="Alice"]  ← users 데이터가 10번 복제
```

원하는 건 **"ACTIVE 유저의 orders"** 인데, INNER JOIN은 **"ACTIVE 유저와 매칭된 모든 조합"** 을 반환한다.

DISTINCT로 해결하려 해도:

```sql
SELECT DISTINCT o.* FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.status = 'ACTIVE';
-- 중복 제거를 위해 임시 테이블 + 정렬 비용 추가 발생
```

근본적으로 **"존재 여부만 확인하고 싶은데 JOIN을 쓰면 데이터가 곱해진다"** 는 문제다.

---

## 2. Mechanics

### Semi-join — 존재하면 반환

```sql
-- 작성하는 SQL (IN / EXISTS)
SELECT * FROM orders
WHERE user_id IN (SELECT id FROM users WHERE status = 'ACTIVE');

SELECT * FROM orders o
WHERE EXISTS (SELECT 1 FROM users u WHERE u.id = o.user_id AND u.status = 'ACTIVE');
```

**핵심 동작:**

```
orders의 각 row에 대해:
  "이 user_id가 ACTIVE 유저 집합에 존재하는가?"
  YES → 반환
  NO  → 버림

users 데이터는 결과에 포함되지 않음
매칭이 여러 개여도 orders row는 1번만 반환
```

```
결과:
orders row  →  ACTIVE 유저 존재 여부만 확인
               users 컬럼은 결과에 없음
               중복 없음
```

---

### Anti-join — 존재하지 않으면 반환

```sql
-- NOT IN
SELECT * FROM orders
WHERE user_id NOT IN (SELECT id FROM users WHERE status = 'ACTIVE');

-- NOT EXISTS
SELECT * FROM orders o
WHERE NOT EXISTS (SELECT 1 FROM users u WHERE u.id = o.user_id AND u.status = 'ACTIVE');
```

**핵심 동작:**

```
orders의 각 row에 대해:
  "이 user_id가 ACTIVE 유저 집합에 존재하는가?"
  YES → 버림
  NO  → 반환
```

실무 사용 예:

```sql
-- 한 번도 주문하지 않은 유저 조회
SELECT * FROM users
WHERE id NOT IN (SELECT user_id FROM orders);

-- 탈퇴했지만 처리 안 된 주문 조회
SELECT * FROM orders o
WHERE NOT EXISTS (
    SELECT 1 FROM users u
    WHERE u.id = o.user_id
      AND u.deleted_at IS NULL
);
```

---

### NOT IN의 치명적 함정 — NULL

```sql
SELECT * FROM orders
WHERE user_id NOT IN (SELECT id FROM users WHERE status = 'ACTIVE');
```

서브쿼리 결과에 NULL이 하나라도 있으면:

```
NOT IN (1, 2, 3, NULL)

user_id = 5 NOT IN (1, 2, 3, NULL)
= user_id != 1 AND user_id != 2 AND user_id != 3 AND user_id != NULL
= TRUE AND TRUE AND TRUE AND UNKNOWN
= UNKNOWN  ← TRUE가 아님

→ 결과: 아무 row도 반환 안 됨
```

```sql
-- users.id가 NULL 가능한 컬럼이라면
-- 또는 서브쿼리 결과에 NULL이 섞이면
-- NOT IN은 빈 결과를 반환한다

-- 안전한 대안: NOT EXISTS 사용
SELECT * FROM orders o
WHERE NOT EXISTS (
    SELECT 1 FROM users u
    WHERE u.id = o.user_id
      AND u.status = 'ACTIVE'
);
-- NULL에 안전, 동일한 결과 보장
```

---

### 옵티마이저의 실행 전략

Semi-join과 Anti-join은 SQL 문법이고, 실제 실행은 옵티마이저가 전략을 선택한다.

**Semi-join 전략:**

```
① Materialization
   서브쿼리 결과를 해시 테이블로 만들어두고
   orders 스캔하며 O(1) 존재 확인
   → 비상관 서브쿼리, 서브쿼리 결과가 작을 때 유리

② FirstMatch
   orders row마다 users에서 첫 매칭 발견 즉시 다음으로
   → 매칭이 빨리 되는 경우 유리

③ Duplicate Weedout
   일반 JOIN으로 실행 후 임시 테이블로 중복 제거
   → 범용적

④ LooseScan
   인덱스의 중복 키를 건너뛰며 스캔
   → 인덱스 있을 때 유리
```

**Anti-join 전략:**

```
MySQL 8.0.17부터 Anti-join 최적화 도입

기존:
  NOT EXISTS → Nested Loop로 매 row마다 서브쿼리 확인

8.0.17+:
  Anti-join 변환 → 매칭 없는 row만 효율적으로 필터
  EXPLAIN에서 "anti join" 표시
```

```sql
EXPLAIN SELECT * FROM orders o
WHERE NOT EXISTS (
    SELECT 1 FROM users u WHERE u.id = o.user_id
);

-- 8.0.17+:
-- -> Anti join (cost=...)
--    → 옵티마이저가 Anti-join으로 변환했다는 신호
```

---

## 3. IN vs EXISTS 선택 기준

```
IN (서브쿼리):
  비상관 서브쿼리 → 옵티마이저가 Materialization 선택 가능
  서브쿼리 결과가 작을 때 유리

EXISTS:
  상관 서브쿼리 구조
  외부 테이블이 작고 내부 테이블이 클 때 유리
  NULL 안전 (NOT EXISTS)
```

실무 판단:

```sql
-- 서브쿼리 결과가 작을 때 (ACTIVE 유저가 소수)
WHERE user_id IN (SELECT id FROM users WHERE status = 'BANNED')
-- → Materialization으로 해시 테이블 만들고 O(1) 조회

-- 외부 테이블이 작을 때
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id)
-- → 외부 row마다 내부 확인, 외부가 작으면 효율적

-- NOT은 항상 NOT EXISTS
WHERE NOT EXISTS (...)  -- NULL 함정 없음
```

---

## 4. 공식 근거

- **MySQL 8.0 — Semi-join Optimization**: [https://dev.mysql.com/doc/refman/8.0/en/semijoins.html](https://dev.mysql.com/doc/refman/8.0/en/semijoins.html)
- **MySQL 8.0 — Anti-join Optimization (8.0.17)**: [https://dev.mysql.com/doc/refman/8.0/en/subquery-optimization.html](https://dev.mysql.com/doc/refman/8.0/en/subquery-optimization.html)
    
    > "The optimizer converts a WHERE condition of the form NOT EXISTS (subquery) or NOT IN (subquery) to an antijoin where possible." (번역: 옵티마이저는 가능한 경우 NOT EXISTS 또는 NOT IN 형태의 WHERE 조건을 Anti-join으로 변환한다.)
    

---

## 5. 한 줄 정리

```
Semi-join:  왼쪽 중 오른쪽에 존재하는 것 반환
            → IN / EXISTS로 작성
            → 중복 없음, 오른쪽 데이터 미포함

Anti-join:  왼쪽 중 오른쪽에 없는 것 반환
            → NOT IN / NOT EXISTS로 작성
            → NOT IN은 NULL 함정 주의, 실무에서는 NOT EXISTS 권장

둘 다 "존재 여부 확인"이 목적
INNER JOIN처럼 데이터를 곱하지 않는다는 게 핵심
```