# CBO 절차 — 실행 계획을 어떻게 고르는가

---

## 전체 흐름 먼저

```
변환된 쿼리 트리
      │
      ▼
┌─────────────────────────┐
│  1. 후보 계획 생성       │  가능한 실행 경로 열거
│  2. 각 계획 비용 추정    │  통계 → 카디널리티 → 비용
│  3. 최저 비용 계획 선택  │  
└─────────────────────────┘
      │
      ▼
   실행 계획 확정 → Executor
```

---

## 1. 후보 계획 생성 — 뭘 열거하는가

쿼리 하나에 대해 옵티마이저가 고려하는 변수는 세 가지다.

```
① 어떤 인덱스를 쓸 것인가  (접근 경로)
② 어느 테이블을 먼저 읽을 것인가  (JOIN 순서)
③ 어떤 JOIN 알고리즘을 쓸 것인가  (NLJ / Hash Join)
```

```sql
SELECT o.id, u.name
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.created_at >= '2024-01-01'
  AND u.status = 'ACTIVE';
```

이 쿼리에 대한 후보 계획들:

```
계획 A: orders를 created_at 인덱스로 range scan
        → 각 row마다 users를 PK로 NLJ
        → users.status 필터

계획 B: users를 status 인덱스로 range scan (또는 풀스캔)
        → 각 row마다 orders를 user_id 인덱스로 NLJ
        → orders.created_at 필터

계획 C: orders 풀스캔
        → users Hash Join (메모리에 users 올림)
        → 두 조건 동시 필터

계획 D: users 풀스캔
        → orders Hash Join
        ...
```

테이블 N개면 JOIN 순서만 N! 가지다. 여기에 인덱스 조합, JOIN 알고리즘 조합까지 곱하면 탐색 공간이 폭발한다.

```
테이블 2개, 인덱스 각 2개, 알고리즘 2개
→ 2! × 2 × 2 × 2 = 16가지

테이블 5개
→ 5! × ... = 120가지 이상
```

그래서 옵티마이저는 **전부 탐색하지 않는다.**

---

## 2. 탐색 전략 — 어떻게 공간을 줄이는가

### Dynamic Programming (소규모)

테이블이 적을 때 사용하는 정확한 탐색 방법이다.

```
테이블: A, B, C

1단계: 단일 테이블 접근 비용 계산
  cost(A) = ?
  cost(B) = ?
  cost(C) = ?

2단계: 2개 조합 비용 계산 (앞 단계 결과 재사용)
  cost(A→B) = cost(A) + A 결과로 B 조회 비용
  cost(B→A) = cost(B) + B 결과로 A 조회 비용
  cost(A→C), cost(C→A), cost(B→C), cost(C→B)
  → 각 조합에서 최솟값만 보관

3단계: 3개 조합 비용 계산
  cost(A→B→C) = cost(A→B) + AB 결과로 C 조회 비용
  cost(B→A→C) = ...
  → 최솟값 선택
```

중간 결과를 재사용하기 때문에 완전 탐색보다 훨씬 빠르다.

### Greedy Search (대규모)

`optimizer_search_depth` 초과 시 전환된다. (기본값 62, 실질적으로 8~10개 테이블부터 영향)

```
현재까지 선택된 순서: [A]
다음 후보: B, C, D, E, F

각 후보를 붙였을 때 비용을 depth 제한까지만 내다봄
→ 가장 저렴한 것 하나 선택: [A → C]

다음 후보: B, D, E, F
→ [A → C → B]

...반복
```

**탐욕 알고리즘이라 전역 최적을 보장하지 않는다.** 각 단계에서 지역 최적을 선택하다 보면 전체적으로 차선이 될 수 있다.

---

## 3. 비용 추정 — 각 계획의 비용을 어떻게 계산하는가

후보 계획이 정해지면 각각의 비용을 계산한다. 비용은 두 가지로 구성된다.

```
총 비용 = I/O 비용 + CPU 비용
```

### I/O 비용 계산

```
[풀스캔]
I/O 비용 = 전체 페이지 수 × io_block_read_cost(1.0)

[인덱스 range scan]
I/O 비용 = (읽을 리프 페이지 수 × 0.5)         -- 순차에 가까움
          + (매칭 row 수 × io_block_read_cost)  -- 클러스터드 인덱스 랜덤 접근
```

range scan에서 **랜덤 I/O 항목**이 중요하다.

```
예: created_at 인덱스로 100만 건 range scan
   → 인덱스 리프: 순차 I/O (저렴)
   → 실제 row 가져오기: 100만 번 랜덤 I/O (비쌈)

이 랜덤 I/O 비용이 풀스캔의 순차 I/O 비용을 역전하는 순간
옵티마이저는 인덱스를 버리고 풀스캔을 선택한다.
```

이게 바로 **"인덱스 있는데 풀스캔 하는" 현상의 원인**이다.

->개별 인덱스가 문제가아니라 인덱스에서 인덱스로 `접근` 할때 랜덤I/O가 발생
=> 커버링 인덱스로 해결

### CPU 비용 계산

```
CPU 비용 = 처리할 row 수 × row_evaluate_cost(0.1)
```

row 수 추정이 카디널리티에서 온다. 카디널리티 추정이 틀리면 CPU 비용 추정도 틀린다.

### 카디널리티 추정 → 비용으로 이어지는 연결

```
[히스토그램/통계에서 선택도 계산]
WHERE status = 'BANNED'
→ 히스토그램: BANNED = 1%
→ 선택도 = 0.01

[카디널리티 추정]
전체 100만 건 × 선택도 0.01 = 10,000건 반환 예상

[비용 계산]
CPU 비용 = 10,000 × 0.1 = 1,000
I/O 비용 = 인덱스 랜덤 I/O 10,000번 × 1.0 = 10,000
총 비용  = 11,000

[비교: 풀스캔]
I/O 비용 = 전체 페이지 5,000개 × 1.0 = 5,000
CPU 비용 = 100만 × 0.1 = 100,000
총 비용  = 105,000

→ 인덱스 선택 (11,000 < 105,000)
```

선택도가 0.01이 아니라 0.95로 잘못 추정되면:

```
카디널리티 추정 = 95만 건
인덱스 랜덤 I/O = 95만 × 1.0 = 950,000
총 비용 = 1,050,000  >  풀스캔 105,000

→ 풀스캔 선택
```

**카디널리티 추정 오차 하나가 실행 계획 전체를 뒤집는다.** 히스토그램이 이 추정을 정확하게 만드는 도구라는 게 여기서 연결된다.

---

## 4. 최종 선택 — EXPLAIN에서 이 과정이 보이는 곳

```sql
EXPLAIN FORMAT=JSON SELECT o.id, u.name
FROM orders o JOIN users u ON o.user_id = u.id
WHERE u.status = 'BANNED';
```

```json
{
  "cost_info": {
    "query_cost": "11234.50"   ← 옵티마이저가 선택한 계획의 총 비용
  },
  "table": "u",
  "access_type": "ref",
  "rows_examined_per_scan": 10000,   ← 카디널리티 추정치
  "rows_produced_per_join": 10000,
  "filtered": "100.00",
  "cost_info": {
    "read_cost": "1000.00",          ← I/O 비용
    "eval_cost": "1000.00",          ← CPU 비용
    "prefix_cost": "2000.00"
  }
}
```

`rows_examined_per_scan`이 실제 결과와 크게 다르면 → 통계가 오래됐거나 히스토그램이 없다는 신호다.

`EXPLAIN ANALYZE`를 쓰면 예측과 실제를 나란히 볼 수 있다.

```sql
EXPLAIN ANALYZE
SELECT o.id, u.name
FROM orders o JOIN users u ON o.user_id = u.id
WHERE u.status = 'BANNED';

-- → Nested loop inner join
--   (cost=11234 rows=10000)        ← 옵티마이저 예측
--   (actual time=2.1..45.3 rows=312 loops=1)  ← 실제
--                            ↑
--              예측 10,000건, 실제 312건 → 통계 문제
```

---

## 한 줄 요약

```
후보 계획 열거 (인덱스 × JOIN순서 × 알고리즘 조합)
      ↓
각 계획의 비용 = I/O 비용 + CPU 비용
      ↓
비용의 입력값 = 카디널리티 추정 = 통계 + 히스토그램
      ↓
추정이 틀리면 → 비용이 틀리면 → 실행 계획이 틀림
      ↓
EXPLAIN ANALYZE로 예측 vs 실제 비교해서 오차 확인
      ↓
오차 크면 → ANALYZE TABLE + 히스토그램으로 교정
```
