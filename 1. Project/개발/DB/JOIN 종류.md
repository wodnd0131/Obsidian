

---

## JOIN의 두 가지 분류 축

먼저 혼동되는 이유를 짚는다.

```
축 1: 결과 범위  → INNER / OUTER / CROSS
축 2: 실행 방식  → NLJ / Hash Join / Semi-join

이 둘은 독립적이다.
INNER JOIN을 NLJ로 실행할 수도 있고 Hash Join으로 실행할 수도 있다.
```

---

## 축 1 — 결과 범위 기준 JOIN 종류

### INNER JOIN

```sql
SELECT o.id, u.name
FROM orders o
INNER JOIN users u ON o.user_id = u.id;
```

```
orders:          users:
user_id=1   →   id=1 ✅ 매칭
user_id=2   →   id=2 ✅ 매칭
user_id=999 →   id=999 ❌ 없음 → 결과에서 제외

결과: 양쪽 모두 매칭되는 row만 반환
```

```
orders  ∩  users
```

---

### LEFT OUTER JOIN

```sql
SELECT o.id, u.name
FROM orders o
LEFT JOIN users u ON o.user_id = u.id;
```

```
orders:          users:
user_id=1   →   id=1 ✅ 매칭
user_id=2   →   id=2 ✅ 매칭
user_id=999 →   id=999 ❌ 없음 → orders row는 유지, users 컬럼은 NULL

결과: 왼쪽(orders) 전부 + 매칭되는 오른쪽
      매칭 없으면 오른쪽은 NULL로 채움
```

```
orders  ∪  (orders ∩ users)
= orders 전체
```

실무에서 자주 쓰는 패턴:

```sql
-- 주문은 있는데 유저가 탈퇴한 경우 포함해서 조회
SELECT o.id, COALESCE(u.name, '탈퇴한 유저') as name
FROM orders o
LEFT JOIN users u ON o.user_id = u.id;
```

---

### RIGHT OUTER JOIN

```sql
SELECT o.id, u.name
FROM orders o
RIGHT JOIN users u ON o.user_id = u.id;
```

```
LEFT JOIN의 반대
오른쪽(users) 전부 + 매칭되는 왼쪽
매칭 없으면 왼쪽은 NULL

= users 중 주문이 없는 유저도 포함
```

실무에서는 LEFT JOIN으로 테이블 순서를 바꿔 대체하는 경우가 많다.

```sql
-- 아래 둘은 결과 동일
SELECT * FROM orders o RIGHT JOIN users u ON o.user_id = u.id;
SELECT * FROM users u LEFT JOIN orders o ON o.user_id = u.id;
```

---

### CROSS JOIN

```sql
SELECT * FROM colors CROSS JOIN sizes;
```

```
colors: RED, BLUE
sizes:  S, M, L

결과:
RED  + S
RED  + M
RED  + L
BLUE + S
BLUE + M
BLUE + L

= 모든 조합 (카테시안 곱)
결과 행 수 = colors 행 수 × sizes 행 수
```

ON 조건 없음. 조건을 추가하면 INNER JOIN과 동일해진다.

---

### Semi-join / Anti-join

앞에서 다뤘지만 이 축에서 위치를 잡으면:

```sql
-- Semi-join: 오른쪽에 존재하는지만 확인, 오른쪽 데이터는 결과에 없음
SELECT * FROM orders
WHERE user_id IN (SELECT id FROM users WHERE status = 'ACTIVE');

-- Anti-join: 오른쪽에 존재하지 않는 것만 반환
SELECT * FROM orders
WHERE user_id NOT IN (SELECT id FROM users WHERE status = 'ACTIVE');
-- 또는
SELECT * FROM orders o
WHERE NOT EXISTS (SELECT 1 FROM users u WHERE u.id = o.user_id);
```

```
INNER JOIN:  양쪽 매칭된 것
Semi-join:   왼쪽 중 오른쪽에 존재하는 것 (오른쪽 데이터 미포함)
Anti-join:   왼쪽 중 오른쪽에 없는 것
```

---

## 전체 그림

```
          왼쪽만   양쪽 교집합   오른쪽만
          ┌────┐   ┌────────┐   ┌────┐
LEFT  →   │ ██ │   │   ██   │   │    │
INNER →   │    │   │   ██   │   │    │
RIGHT →   │    │   │   ██   │   │ ██ │
CROSS →   모든 조합
Semi  →   왼쪽 중 오른쪽에 존재하는 것 (교집합 여부만 확인)
Anti  →   왼쪽 중 오른쪽에 없는 것
```

---

## 축 2와의 관계 — 실행 방식은 독립적

```
INNER JOIN → NLJ, Hash Join 중 옵티마이저가 선택
LEFT JOIN  → NLJ로만 실행 가능 (Hash Join 제한적)
Semi-join  → Duplicate Weedout / FirstMatch / LooseScan / Materialization 중 선택
```

LEFT JOIN이 INNER JOIN보다 느린 경향이 있는 이유:

```
INNER JOIN: 매칭 없으면 버림 → 드라이버 선택 자유로움
LEFT JOIN:  왼쪽 전부 유지해야 함 → 왼쪽이 무조건 드라이버
            → 옵티마이저 최적화 자유도 감소
```

---

## 한 줄 정리

```
결과 범위 기준:
  INNER  → 양쪽 매칭된 것만
  LEFT   → 왼쪽 전부 + 매칭된 오른쪽
  RIGHT  → 오른쪽 전부 + 매칭된 왼쪽
  CROSS  → 모든 조합
  Semi   → 왼쪽 중 오른쪽에 존재하는 것 (오른쪽 데이터 미포함)
  Anti   → 왼쪽 중 오른쪽에 없는 것

실행 방식(NLJ/Hash Join/Semi-join 전략)은 별개로 옵티마이저가 결정
```