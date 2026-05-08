
## 1. Problem First — 옵티마이저가 없었다면

### 시나리오: 순진한 실행 계획

```sql
SELECT o.id, o.amount, u.name, u.email
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.status = 'ACTIVE'
  AND o.created_at >= '2024-01-01'
  AND o.deleted_at IS NULL;
```

옵티마이저가 없다면, DBMS는 SQL이 작성된 순서 그대로 실행한다.

```
1. orders 전체 풀스캔 (1,000만 건)
2. 각 row마다 users를 풀스캔하며 JOIN (1,000만 × 100만 = 10조 번 비교)
3. WHERE 필터 적용
```

**이게 단순히 "느리다"가 아닌 근본 문제인 이유:**

SQL은 _선언형(declarative)_ 언어다. "무엇을 원하는가"만 기술하고 "어떻게 가져올 것인가"는 기술하지 않는다. 동일한 결과를 내는 실행 경로가 수십~수천 가지 존재한다.

```
경로 A: orders → (nested loop) → users  →  index on orders.created_at 사용
경로 B: users  → (nested loop) → orders  →  index on users.status 사용
경로 C: 두 테이블 hash join 후 필터
경로 D: orders를 created_at 인덱스로 range scan 후 user_id로 users 조회
...
```

비용 차이가 수백만 배까지 벌어질 수 있다. 이걸 개발자가 매 쿼리마다 직접 결정한다는 건 불가능하다. 옵티마이저는 이 결정을 자동화한다. 그리고 그 결정의 품질이 곧 DB 성능이다.

---

## 2. Mechanics — 옵티마이저 내부 동작

MySQL 옵티마이저는 **비용 기반 옵티마이저(Cost-Based Optimizer, CBO)** 다. "가장 빠를 것 같은" 계획이 아니라 **비용 모델에 따라 가장 낮은 추정 비용의 계획**을 선택한다.

### 파이프라인 전체 그림

```
SQL 텍스트
    │
    ▼
┌─────────────────────────────┐
│  1. Parser                  │  SQL → Parse Tree (AST)
│     문법 오류 검출           │
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│  2. Preprocessor            │  의미 검증 (테이블/컬럼 존재 여부)
│     권한 확인, 뷰 확장       │  뷰 → 서브쿼리로 인라인화
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│  3. Optimizer               │  ← 여기가 핵심
│     논리적 변환 (Rewrite)    │
│     물리적 계획 탐색         │
│     비용 추정 & 선택         │
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│  4. Executor                │  선택된 계획을 실행
│     Storage Engine 호출     │  InnoDB API 통해 실제 I/O
└─────────────────────────────┘
```

---

### 옵티마이저 내부 — 3단계 상세

#### 단계 1: 논리적 변환 (Query Rewrite / Transformation)

실행 전에 SQL 자체를 **의미론적으로 동등하지만 더 효율적인 형태**로 변환한다.

**① 조건 푸시다운 (Predicate Pushdown)**

```sql
-- 원본
SELECT * FROM (SELECT * FROM orders) AS sub WHERE sub.user_id = 1;

-- 변환 후 (서브쿼리 안으로 조건을 밀어넣음)
SELECT * FROM orders WHERE user_id = 1;
```

서브쿼리 전체를 물질화(materialize)하기 전에 필터를 먼저 적용해서 데이터 볼륨을 줄인다.

**② 상수 폴딩 (Constant Folding)**

```sql
WHERE price > 10 * 5   →   WHERE price > 50
WHERE '2024-01-01' < created_at AND created_at < '2024-12-31'
-- 실행 시마다 계산하지 않고 컴파일 타임에 상수로 치환
```

**③ 불필요 조건 제거 (Trivially True/False)**

```sql
WHERE 1 = 1        →  제거
WHERE 1 = 2        →  빈 결과 반환 (실행 자체 안 함)
WHERE id = 1 AND id = 2  →  빈 결과
```

**④ 서브쿼리 → JOIN 변환 (Subquery Unnesting)**

```sql
-- 원본: 서브쿼리 = 매 row마다 반복 실행될 위험
SELECT * FROM orders
WHERE user_id IN (SELECT id FROM users WHERE status = 'ACTIVE');

-- 변환 후: semi-join으로 재작성 (한 번만 실행)
SELECT DISTINCT o.* FROM orders o
INNER JOIN users u ON o.user_id = u.id
WHERE u.status = 'ACTIVE';
```

MySQL 8.0부터 이 변환이 대폭 강화됐다. (공식 레퍼런스: [MySQL 8.0 Subquery Optimization](https://dev.mysql.com/doc/refman/8.0/en/subquery-optimization.html))

---

#### 단계 2: 통계 수집 및 카디널리티 추정

옵티마이저가 비용을 계산하려면 "이 조건으로 몇 건이 나올 것인가"를 알아야 한다. 이 추정치를 **카디널리티(Cardinality)** 라고 하고, 여기서 오차가 생기면 실행 계획이 망가진다.

**통계 정보 저장 위치**

```sql
-- 테이블/인덱스 통계
SELECT * FROM information_schema.INNODB_TABLE_STATS WHERE table_name = 'orders';
SELECT * FROM information_schema.INNODB_INDEX_STATS WHERE table_name = 'orders';

-- 컬럼 통계 (MySQL 8.0 히스토그램)
SELECT * FROM information_schema.COLUMN_STATISTICS;
```

**통계 항목 상세**

| 항목                     | 의미               | 사용처                 |
| ---------------------- | ---------------- | ------------------- |
| `n_rows`               | 테이블 예상 행 수       | 풀스캔 비용 기준           |
| `clustered_index_size` | 클러스터드 인덱스 페이지 수  | I/O 비용              |
| `n_diff_pfx01..N`      | 인덱스 컬럼 조합별 고유값 수 | 선택도(Selectivity) 계산 |
| 히스토그램                  | 컬럼 값 분포          | 비균등 분포 처리           |

**선택도(Selectivity) 계산 원리**

```
선택도 = (조건에 맞는 행 수) / (전체 행 수)
       = 1 / cardinality

예: users.status 컬럼의 고유값이 3개 (ACTIVE, INACTIVE, BANNED)
    → 선택도 ≈ 1/3 ≈ 0.33
    → 인덱스 사용해도 33% 풀스캔이므로 옵티마이저가 풀스캔 선택할 수 있음
```

**히스토그램이 필요한 이유 (MySQL 8.0 도입)**

균등 분포를 가정하면 틀린 경우:

```
orders.status: COMPLETED(95%), CANCELLED(4%), PENDING(1%)

히스토그램 없음 → 선택도 = 1/3 = 33% 로 추정
히스토그램 있음 → COMPLETED는 95%, PENDING은 1% 로 정확히 추정
```

```sql
-- 히스토그램 생성
ANALYZE TABLE orders UPDATE HISTOGRAM ON status WITH 100 BUCKETS;
```

(공식 레퍼런스: [MySQL 8.0 Histograms](https://dev.mysql.com/doc/refman/8.0/en/analyze-table.html#analyze-table-histogram-statistics-analysis))

---

#### 단계 3: 실행 계획 탐색 및 비용 계산

**비용 모델**

MySQL의 비용 단위는 추상화된 "cost unit"이다.

```sql
-- 비용 상수 확인 (mysql.server_cost, mysql.engine_cost)
SELECT * FROM mysql.server_cost;
SELECT * FROM mysql.engine_cost;
```

| 항목                           | 기본값  | 의미            |
| ---------------------------- | ---- | ------------- |
| `row_evaluate_cost`          | 0.1  | 행 하나 평가 비용    |
| `disk_temptable_create_cost` | 20.0 | 디스크 임시 테이블 생성 |
| `io_block_read_cost`         | 1.0  | 디스크 I/O 1블록   |
| `memory_block_read_cost`     | 0.25 | 메모리 I/O 1블록   |
|                              |      |               |

**인덱스 선택 비용 계산 예시**

```
테이블: orders (1,000만 행)
조건: created_at >= '2024-01-01'  (전체의 10% = 100만 건 예상)

[인덱스 range scan 비용]
= (읽을 리프 페이지 수 × io_block_read_cost)
+ (100만 행 × row_evaluate_cost)
+ (랜덤 I/O: 클러스터드 인덱스 접근 비용)  ← 여기가 중요
≈ 상당히 높음 (랜덤 I/O 때문)

[풀스캔 비용]
= (전체 페이지 수 × io_block_read_cost × 순차 I/O 보정)
+ (1,000만 행 × row_evaluate_cost)
≈ 순차 I/O라 예상보다 낮을 수 있음
```

> 이 때문에 **인덱스가 있어도 옵티마이저가 풀스캔을 선택**하는 경우가 생긴다. 선택도가 낮을수록 랜덤 I/O 비용이 순차 풀스캔 비용을 역전한다.

**JOIN 순서 탐색 (핵심)**

테이블 N개를 JOIN할 때 가능한 순서는 **N! (팩토리얼)** 이다.

```
5개 테이블 → 120가지
10개 테이블 → 3,628,800가지
```

MySQL은 기본적으로 `optimizer_search_depth = 62` 로 설정되어 있고, 테이블 수가 일정 이상이면 **Greedy Search**로 전환해 완전 탐색을 포기한다.

```sql
SHOW VARIABLES LIKE 'optimizer_search_depth';
-- 0: 자동 결정, 1~62: 탐색 깊이 제한
```

이 제한 때문에 **최적이 아닌 계획이 선택될 수 있다.**

---

### EXPLAIN 출력과 옵티마이저 결정의 매핑

직접 EXPLAIN을 써봤으니 내부 의미를 짚고 넘어간다.

```sql
EXPLAIN FORMAT=JSON SELECT ...;  -- 비용까지 보려면 JSON 포맷
EXPLAIN ANALYZE SELECT ...;      -- MySQL 8.0.18+, 실제 실행 후 예측 vs 실제 비교
```

| EXPLAIN 컬럼      | 옵티마이저 결정 내용                                      |
| --------------- | ------------------------------------------------ |
| `type`          | 접근 방식 (풀스캔/인덱스/range 등)                          |
| `possible_keys` | 옵티마이저가 후보로 고려한 인덱스                               |
| `key`           | 실제 선택한 인덱스                                       |
| `key_len`       | 인덱스의 몇 바이트까지 사용했는가                               |
| `rows`          | 옵티마이저의 카디널리티 추정치                                 |
| `filtered`      | rows 중 WHERE 조건 통과 비율 추정                         |
| `Extra`         | Using index / Using filesort / Using temporary 등 |

```
type 성능 순서:
system > const > eq_ref > ref > range > index > ALL

ALL = 풀스캔, 대부분의 경우 피해야 함
index = 인덱스 풀스캔, ALL보다는 낫지만 여전히 전체 탐색
range = 인덱스로 범위 탐색, 일반적으로 목표 수준
ref = 비유니크 인덱스 동등 조건, 좋음
eq_ref = 유니크 인덱스/PK 동등 조건 (JOIN에서), 매우 좋음
const = PK/유니크 단일 값 조건, 상수 취급
```

---

## 3. 공식 근거

- **MySQL 8.0 Reference — The Optimizer** : [https://dev.mysql.com/doc/refman/8.0/en/optimization.html](https://dev.mysql.com/doc/refman/8.0/en/optimization.html)
- **Cost Model 공식 문서** : [https://dev.mysql.com/doc/refman/8.0/en/cost-model.html](https://dev.mysql.com/doc/refman/8.0/en/cost-model.html)
    
    > "The optimizer uses the cost model to choose among multiple query execution plans by estimating the cost of each plan." (번역: 옵티마이저는 비용 모델을 사용해 각 실행 계획의 비용을 추정하고 여러 계획 중 하나를 선택한다.)
    
- **MySQL 8.0 Histograms** : [https://dev.mysql.com/doc/refman/8.0/en/analyze-table.html](https://dev.mysql.com/doc/refman/8.0/en/analyze-table.html)
- **Subquery Optimization** : [https://dev.mysql.com/doc/refman/8.0/en/subquery-optimization.html](https://dev.mysql.com/doc/refman/8.0/en/subquery-optimization.html)
- **Morgan Tocker (MySQL Product Manager), "Understanding the Query Optimizer"** — MySQL Blog, 공식 블로그 수준
- _High Performance MySQL, 4th Ed._ — Schwartz, Zaitsev (3순위에 준하는 MySQL 전문서)

---

## 4. 설계 트레이드오프 — 옵티마이저를 이렇게 만든 이유와 그 비용

### ✅ CBO(비용 기반)를 선택한 이유

과거 RBO(Rule-Based Optimizer)는 고정된 규칙으로 계획을 선택했다. (예: "인덱스가 있으면 항상 인덱스 사용")

**RBO의 실패 사례:**

```sql
-- 인덱스 있어도 데이터 99%가 status='ACTIVE'인 경우
SELECT * FROM users WHERE status = 'ACTIVE';
-- RBO: 인덱스 있음 → 인덱스 사용 → 99% 행에 랜덤 I/O → 풀스캔보다 느림
-- CBO: 선택도 0.99 → 풀스캔이 저렴 → 풀스캔 선택
```

CBO는 데이터 분포를 반영하므로 더 현실적이다.

### ❌ CBO의 구조적 한계

**한계 1: 통계 오래되면 계획 망가짐**

```sql
-- 대량 INSERT/DELETE 후 통계 갱신 안 됐을 때
-- rows 추정: 1,000건  /  실제: 10,000,000건
-- → 인덱스 nested loop 선택 → 실제로는 1,000만 번 랜덤 I/O → 장애
ANALYZE TABLE orders;  -- 수동 갱신
```

InnoDB는 백그라운드에서 자동으로 통계를 수집하지만, `innodb_stats_auto_recalc = ON` 이고 변경 행이 10% 초과할 때만 트리거된다. 대량 배치 후에는 수동 ANALYZE가 필요하다.

**한계 2: 비용 모델이 실제 하드웨어와 괴리**

기본 `io_block_read_cost = 1.0` 은 HDD 기준으로 설계됐다. SSD/NVMe 환경에서는 랜덤 I/O와 순차 I/O의 비용 차이가 훨씬 줄어든다. → SSD 환경에서 옵티마이저가 풀스캔을 과도하게 기피하거나 반대로 인덱스를 과소평가할 수 있다.

```sql
-- SSD 환경에서 비용 상수 조정 고려
UPDATE mysql.engine_cost
SET cost_value = 0.25
WHERE cost_name = 'io_block_read_cost';
FLUSH OPTIMIZER_COSTS;
```

> 단, 이 조정은 **모든 쿼리에 전역 영향**을 미치므로 프로덕션에서는 신중하게 적용해야 한다.

**한계 3: JOIN 순서 탐색 한계 (이미 언급)**

10개 이상 테이블 JOIN에서 Greedy Search로 전환하면 최적 계획을 못 찾을 수 있다. 이 경우 `STRAIGHT_JOIN` 힌트로 순서를 강제하거나 쿼리를 분리해야 한다.

**한계 4: 옵티마이저는 "지금 이 쿼리"만 본다**

동시에 실행 중인 다른 쿼리의 부하를 모른다. 풀스캔이 비용상 최적이어도, 그게 100개 동시에 들어오면 시스템 전체가 망가진다. → 옵티마이저 튜닝만으로 해결 안 됨. 인덱스/아키텍처 설계가 함께 필요.

---

## 5. 실무 튜닝 포인트 — 옵티마이저에 신경써야 할 것들

### ① 통계 신뢰성 확보

```sql
-- 통계 갱신 시점 확인
SELECT last_update, n_rows, clustered_index_size
FROM mysql.innodb_table_stats
WHERE table_name = 'orders';

-- 대량 배치 후 명시적 갱신
ANALYZE TABLE orders;

-- 통계 샘플 페이지 수 조정 (기본 20, 정확도 vs 갱신 비용 트레이드오프)
innodb_stats_persistent_sample_pages = 200  -- my.cnf
```

### ② 히스토그램 활용 (MySQL 8.0+)

인덱스 없는 컬럼에 범위 조건이 많을 때, 또는 값 분포가 극단적으로 편향된 컬럼에 적용한다.

```sql
ANALYZE TABLE orders
UPDATE HISTOGRAM ON status, payment_method WITH 100 BUCKETS;

-- 확인
SELECT * FROM information_schema.COLUMN_STATISTICS
WHERE table_name = 'orders';
```

### ③ EXPLAIN ANALYZE로 추정 vs 실제 비교

EXPLAIN의 `rows`는 추정치다. 실제와 크게 다르면 통계 문제다.

```sql
EXPLAIN ANALYZE
SELECT o.id, u.name
FROM orders o JOIN users u ON o.user_id = u.id
WHERE o.created_at >= '2024-01-01';

-- 출력 예시
-- -> Nested loop inner join  (cost=12345 rows=8900) (actual time=0.5..230 rows=95000 loops=1)
--    추정 8,900건인데 실제 95,000건 → 통계 갱신 필요
```

### ④ 옵티마이저가 틀렸을 때 힌트 사용

```sql
-- 특정 인덱스 강제
SELECT * FROM orders USE INDEX (idx_created_at) WHERE ...;
SELECT * FROM orders FORCE INDEX (idx_created_at) WHERE ...;  -- 더 강제적

-- JOIN 순서 강제
SELECT STRAIGHT_JOIN o.id, u.name FROM orders o JOIN users u ...;

-- MySQL 8.0 옵티마이저 힌트 (더 세밀한 제어)
SELECT /*+ NO_RANGE_OPTIMIZATION(o idx_created_at) */ * FROM orders o WHERE ...;
SELECT /*+ JOIN_ORDER(u, o) */ o.id FROM orders o JOIN users u ON ...;
```

> 힌트는 **최후 수단**이다. 통계 갱신, 인덱스 재설계로 해결 못 했을 때만 쓴다. 힌트가 박힌 쿼리는 데이터 분포가 바뀌어도 고정된 계획을 강제하므로 나중에 독이 될 수 있다.

### ⑤ 소프트딜리트 환경에서의 함정

직접 소프트딜리트 아카이빙을 해봤다고 했으니 이 부분은 특히 중요하다.

```sql
-- 이 인덱스는 deleted_at IS NULL 조건에서 비효율적
CREATE INDEX idx_status ON orders (status);

-- WHERE status = 'ACTIVE' AND deleted_at IS NULL
-- → deleted_at IS NULL이 전체의 99%라면 선택도 매우 낮음
-- → 옵티마이저가 인덱스 무시하고 풀스캔 선택 가능

-- 해결: 부분 인덱스(MySQL은 지원 안 함)이 없으므로
-- 복합 인덱스로 deleted_at을 포함시켜 커버링 유도
CREATE INDEX idx_active_orders ON orders (deleted_at, status, created_at);
-- deleted_at IS NULL → 인덱스 앞부분에서 걸러짐
-- status, created_at → 이후 조건 처리
```

또한 아카이빙 배치 실행 후에는:

```sql
-- 대량 DELETE/INSERT 후 통계 강제 갱신 필수
ANALYZE TABLE orders;
ANALYZE TABLE orders_archive;
```

---

## 이어지는 개념

|순서|개념|이유|
|---|---|---|
|1|**InnoDB 클러스터드 인덱스 & 세컨더리 인덱스 구조**|옵티마이저가 "랜덤 I/O 비용"을 계산하는 근거가 B+Tree 구조에서 나옴. 이걸 모르면 인덱스 선택 이유를 온전히 이해 못 함|
|2|**커버링 인덱스 (Covering Index)**|`Using index` Extra 조건이 왜 극적인 성능 차이를 만드는지 — 직전 학습의 즉각적 응용|
|3|**트랜잭션 격리 수준 & MVCC**|InnoDB가 EXPLAIN의 rows와 다르게 보이는 버전을 읽는 이유, 소프트딜리트 아카이빙 중 락 경합 원인|
|4|**실행 계획 안정성 — Plan Cache & Adaptive Hash Index**|배포 후 갑자기 쿼리가 느려지는 이유 (plan이 바뀐 것), AHI가 옵티마이저 바깥에서 개입하는 구조|