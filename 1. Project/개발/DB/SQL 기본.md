# SQL 기본 문법 — MySQL 기준

> SQL 기본 문법은 "학습 지침의 단일 개념 키워드"라기보다 **레퍼런스 범주**에 가깝습니다. 따라서 섹션 1·4·5는 문법 항목별로 적용이 다르므로, 전체를 아우르는 형태로 통합 정리합니다. 각 문법 항목의 내부 동작(Mechanics)과 공식 근거는 개별로 명시합니다.

---

## Problem First — SQL이 없던 세계

1970년대 이전, 데이터는 계층형(hierarchical) 또는 네트워크(network) 모델로 저장됐다. 레코드 접근 경로(access path)를 **애플리케이션 코드가 직접 지정**해야 했다.

```
// 계층형 모델 시절 의사 코드
Record emp = db.getRoot("DEPARTMENT")
               .getChild("EMPLOYEE")
               .findByKey("EMP_ID", 1234);
```

문제는 스키마가 바뀌면 **탐색 경로를 쓰는 모든 코드**가 바뀐다는 것이다. Edgar F. Codd(IBM)는 1970년 논문 *"A Relational Model of Data for Large Shared Data Banks"*에서 **"무엇을(what)"만 선언하면 "어떻게(how)"는 엔진이 결정하는** 선언형 언어를 제안했다. 그 결과가 SQL이다.

---

## Mechanics + 공식 근거 + 설계 이유 — 문법별 정리

---

### 0. 실행 순서 (Logical Query Processing Order)

SQL은 **작성 순서 ≠ 실행 순서**다. 이걸 모르면 WHERE에서 별칭(alias)이 왜 안 되는지, GROUP BY 뒤 HAVING이 왜 필요한지 이해 못 한다.

```sql
SELECT   --  6
FROM     --  1
JOIN     --  2
WHERE    --  3
GROUP BY --  4
HAVING   --  5
ORDER BY --  7
LIMIT    --  8
```

> **MySQL 8.0 Reference Manual — The Optimizer** "The server evaluates FROM clauses first to produce a working table, then applies WHERE, then groups rows with GROUP BY…" _→ 서버는 FROM 절을 먼저 평가하여 작업 테이블을 생성하고, WHERE를 적용한 후 GROUP BY로 행을 그룹화한다._

**왜 이 순서인가**

- WHERE는 그룹화 전 개별 행 필터 → 아직 집계 결과가 없으니 집계 함수 사용 불가
- HAVING은 그룹화 후 필터 → 집계 결과 기준 필터 가능
- SELECT의 별칭은 6번째 단계 → WHERE(3번)에서는 아직 별칭이 정의되지 않았으므로 참조 불가 (단, MySQL은 GROUP BY/ORDER BY에서 SELECT 별칭 참조를 **비표준 확장**으로 허용)

---

### 1. DDL — Data Definition Language

#### CREATE TABLE

```sql
CREATE TABLE orders (
    order_id    BIGINT          NOT NULL AUTO_INCREMENT,
    user_id     BIGINT          NOT NULL,
    status      VARCHAR(20)     NOT NULL DEFAULT 'PENDING',
    amount      DECIMAL(10, 2)  NOT NULL,
    created_at  DATETIME(6)     NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    PRIMARY KEY (order_id),
    INDEX idx_user_id (user_id),
    CONSTRAINT fk_user FOREIGN KEY (user_id) REFERENCES users(id)
);
```

**내부 동작 (InnoDB 기준)**

- `PRIMARY KEY` → InnoDB는 PK를 **클러스터드 인덱스(Clustered Index)**로 구성한다. 행 데이터가 PK 순서로 B+Tree 리프 노드에 직접 저장된다.
- `INDEX` → 세컨더리 인덱스. 리프 노드에 인덱스 컬럼 값 + **PK 값**을 저장 (→ 인덱스만으로 행을 못 찾으면 PK로 클러스터드 인덱스를 다시 탐색, 이를 **"더블 룩업"** 혹은 **"인덱스 머지"** 비용이라 부름)
- `FOREIGN KEY` → 참조 무결성을 InnoDB 엔진 레벨에서 강제. MyISAM은 파싱은 하지만 **무시**한다.
- `DECIMAL(10, 2)` → 금액에 `FLOAT`/`DOUBLE` 절대 금지. 부동소수점은 이진 근사값이므로 `0.1 + 0.2 ≠ 0.3` 문제가 발생한다. `DECIMAL`은 **정확한 고정소수점** 연산.

> **MySQL 8.0 Reference — InnoDB and the ACID Model** "Every InnoDB table has a special index called the clustered index that stores row data. If you define a PRIMARY KEY on a table, InnoDB uses it as the clustered index." _→ 모든 InnoDB 테이블에는 행 데이터를 저장하는 클러스터드 인덱스가 존재하며, PRIMARY KEY가 정의된 경우 이를 클러스터드 인덱스로 사용한다._

---

#### ALTER TABLE

```sql
-- 컬럼 추가
ALTER TABLE orders ADD COLUMN updated_at DATETIME(6) NULL;

-- 인덱스 추가
ALTER TABLE orders ADD INDEX idx_status_created (status, created_at);

-- 컬럼 타입 변경 (위험 — 아래 설명)
ALTER TABLE orders MODIFY COLUMN status VARCHAR(50) NOT NULL;
```

**핵심 주의점 — Online DDL**

MySQL 5.6 이전: `ALTER TABLE`은 테이블 전체를 **임시 테이블로 복사** → 수억 건 테이블에서 수 시간 잠금.

MySQL 5.6+ InnoDB: **Online DDL** 도입. `ALGORITHM`과 `LOCK` 옵션으로 동작 제어 가능.

```sql
-- 가장 안전한 방법: INPLACE + 잠금 없음 명시
ALTER TABLE orders
    ADD INDEX idx_status (status),
    ALGORITHM=INPLACE,
    LOCK=NONE;
```

|ALGORITHM|설명|테이블 잠금|
|---|---|---|
|COPY|임시 테이블 복사 (구방식)|쓰기 잠금|
|INPLACE|원본 테이블 직접 수정|최소화|
|INSTANT|메타데이터만 변경 (MySQL 8.0, 컬럼 추가 한정)|없음|

> **MySQL 8.0 Reference — Online DDL Operations** "INSTANT: Only modifies metadata in the data dictionary. No exclusive metadata locks are taken on the table during the preparation and execution phase…" _→ INSTANT 알고리즘은 데이터 딕셔너리의 메타데이터만 수정한다. 준비 및 실행 단계에서 테이블에 배타적 메타데이터 잠금이 걸리지 않는다._

대용량 운영 테이블의 스키마 변경에는 **pt-online-schema-change** 또는 **gh-ost** 같은 툴을 사용하는 것이 업계 관행이다. (공식 근거 수준 아닌 커뮤니티 관행)

---

### 2. DML — Data Manipulation Language

#### SELECT

```sql
-- 기본 구조
SELECT
    u.id,
    u.name,
    COUNT(o.order_id)   AS order_count,
    SUM(o.amount)       AS total_amount
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE o.created_at >= '2024-01-01'
  AND o.status = 'COMPLETED'
GROUP BY u.id, u.name
HAVING total_amount >= 100000
ORDER BY total_amount DESC
LIMIT 20 OFFSET 0;
```

**JOIN 내부 동작 — 세 가지 알고리즘**

MySQL Optimizer는 아래 세 알고리즘 중 비용(cost)이 가장 낮은 것을 선택한다.

|알고리즘|조건|복잡도|
|---|---|---|
|**Nested Loop Join (NLJ)**|항상 가능|O(N×M)|
|**Block Nested Loop (BNL)**|인덱스 없을 때 join_buffer 활용|O(N×M), 메모리 최적화|
|**Hash Join**|MySQL 8.0.18+, 동등 조인(equi-join)|O(N+M) 기대값|

> **MySQL 8.0 Reference — Hash Join Optimization** "Beginning with MySQL 8.0.18, MySQL employs a hash join for any query for which each join has an equi-join condition…" _→ MySQL 8.0.18부터, 각 조인에 동등 조인 조건이 있는 모든 쿼리에 해시 조인을 사용한다._

**LIMIT + OFFSET의 함정**

```sql
-- 100만 번째 페이지 — 실제로 100만+20개 행을 읽고 버린다
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 1000000;

-- 커서 기반 페이지네이션 (No-Offset) — 성능 O(1)
SELECT * FROM orders
WHERE order_id < :last_seen_id
ORDER BY order_id DESC
LIMIT 20;
```

---

#### INSERT

```sql
-- 단건
INSERT INTO orders (user_id, status, amount)
VALUES (1, 'PENDING', 15000.00);

-- 다건 (Batch Insert) — 단건 반복보다 훨씬 빠름
INSERT INTO orders (user_id, status, amount)
VALUES
    (1, 'PENDING', 15000.00),
    (2, 'PENDING', 32000.00),
    (3, 'PENDING',  8500.00);

-- UPSERT — 중복 키 시 UPDATE
INSERT INTO daily_stats (date, count)
VALUES ('2024-01-15', 1)
ON DUPLICATE KEY UPDATE count = count + 1;
```

**Batch Insert가 빠른 이유**

단건 INSERT N번 = 네트워크 라운드트립 N번 + 트랜잭션 커밋 N번 + Redo Log flush N번. 다건 INSERT 1번 = 라운드트립 1번 + 커밋 1번 + flush 1번.

InnoDB의 **Change Buffer**도 관여한다. 세컨더리 인덱스 변경을 바로 디스크에 쓰지 않고 버퍼에 모아 병합한다.

> **MySQL 8.0 Reference — Change Buffer** "The change buffer is a special data structure that caches changes to secondary index pages when those pages are not in the buffer pool." _→ Change Buffer는 보조 인덱스 페이지가 버퍼 풀에 없을 때 해당 페이지의 변경 사항을 캐싱하는 특수 자료구조다._

---

#### UPDATE

```sql
-- 기본
UPDATE orders
SET status = 'CANCELLED', updated_at = NOW(6)
WHERE order_id = 1234
  AND status = 'PENDING';   -- 조건 없는 UPDATE는 전체 테이블 갱신

-- 여러 테이블 JOIN UPDATE (MySQL 확장 문법)
UPDATE orders o
JOIN users u ON o.user_id = u.id
SET o.status = 'VIP_PROCESSING'
WHERE u.grade = 'VIP'
  AND o.status = 'PENDING';
```

**`WHERE` 없는 UPDATE의 위험**

```sql
UPDATE orders SET status = 'CANCELLED';  -- 전체 행 갱신
```

운영 환경에서 실수를 방지하려면:

```sql
SET SQL_SAFE_UPDATES = 1;
-- WHERE 절에 KEY 컬럼이 없으면 에러 발생
```

---

#### DELETE vs TRUNCATE vs DROP

```sql
DELETE FROM orders WHERE created_at < '2020-01-01';  -- 행 단위 삭제, 롤백 가능
TRUNCATE TABLE temp_orders;                           -- 전체 삭제, DDL, 빠름
DROP TABLE temp_orders;                               -- 테이블 자체 제거
```

|구분|분류|롤백|속도|트리거 발동|AUTO_INCREMENT 초기화|
|---|---|---|---|---|---|
|DELETE|DML|가능|느림|O|X|
|TRUNCATE|DDL|**불가**|빠름|X|**O**|
|DROP|DDL|불가|즉시|X|—|

**TRUNCATE가 빠른 이유**: 행을 하나씩 삭제하지 않고 데이터 파일 자체를 재생성(deallocate & reallocate)한다. Undo Log를 생성하지 않으므로 롤백이 불가능하다.

> **MySQL 8.0 Reference — TRUNCATE TABLE Statement** "Logically, TRUNCATE TABLE is similar to a DELETE statement that deletes all rows, or a sequence of DROP TABLE and CREATE TABLE statements." _→ 논리적으로 TRUNCATE TABLE은 모든 행을 삭제하는 DELETE 또는 DROP TABLE과 CREATE TABLE의 연속 실행과 유사하다._

---

### 3. 조건 및 필터링

#### WHERE 조건

```sql
-- 범위
WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31'

-- IN (서브쿼리보다 값 목록이 적을 때)
WHERE status IN ('PENDING', 'PROCESSING')

-- LIKE — 앞 와일드카드는 인덱스 풀스캔
WHERE name LIKE '김%'    -- 인덱스 사용 가능
WHERE name LIKE '%김'    -- 인덱스 사용 불가 (Full Scan)
WHERE name LIKE '%김%'   -- 인덱스 사용 불가

-- NULL 비교 — = NULL은 절대 동작 안 함
WHERE deleted_at IS NULL
WHERE deleted_at IS NOT NULL
```

**NULL의 3중 논리(Three-Valued Logic)**

SQL에서 NULL은 "알 수 없음(unknown)"이다. `NULL = NULL`은 `TRUE`가 아니라 `NULL`(unknown)이다.

> **ISO/IEC 9075 SQL Standard (SQL:2016) — Null Values** "A null value is neither equal to nor not equal to any value, including another null value." _→ NULL 값은 다른 NULL 값을 포함한 어떤 값과도 같지도 않고 다르지도 않다._

이 때문에 `WHERE col = NULL`은 항상 0건을 반환한다. `IS NULL`을 사용해야 한다.

---

#### GROUP BY + 집계 함수

```sql
SELECT
    DATE(created_at)    AS order_date,
    status,
    COUNT(*)            AS cnt,
    COUNT(DISTINCT user_id) AS unique_users,
    SUM(amount)         AS total,
    AVG(amount)         AS avg_amount,
    MAX(amount)         AS max_amount,
    MIN(amount)         AS min_amount
FROM orders
GROUP BY DATE(created_at), status
HAVING cnt >= 10
ORDER BY order_date DESC;
```

**MySQL의 비표준 GROUP BY (ONLY_FULL_GROUP_BY)**

```sql
-- ONLY_FULL_GROUP_BY 활성화 시 에러 발생
SELECT user_id, name, COUNT(*) FROM orders GROUP BY user_id;
-- name이 GROUP BY에 없음 → 어떤 name을 반환할지 불확정

-- 올바른 방법
SELECT user_id, name, COUNT(*)
FROM orders
GROUP BY user_id, name;

-- 또는 ANY_VALUE() — 비결정적임을 명시적으로 인정
SELECT user_id, ANY_VALUE(name), COUNT(*)
FROM orders
GROUP BY user_id;
```

> **MySQL 8.0 Reference — MySQL Handling of GROUP BY** "MySQL 5.7.5 and higher implements detection of functional dependencies. If the ONLY_FULL_GROUP_BY SQL mode is enabled … MySQL rejects queries for which the select list, HAVING condition, or ORDER BY list refer to nonaggregated columns." _→ MySQL 5.7.5 이상은 함수 종속성 감지를 구현한다. ONLY_FULL_GROUP_BY SQL 모드가 활성화된 경우, SELECT 목록·HAVING 조건·ORDER BY 목록이 비집계 컬럼을 참조하는 쿼리를 거부한다._

---

### 4. JOIN 종류

```sql
-- INNER JOIN: 양쪽 모두 매칭되는 행만
SELECT u.name, o.order_id
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- LEFT JOIN: 왼쪽 테이블 전체 + 매칭되는 오른쪽 (없으면 NULL)
SELECT u.name, o.order_id
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;
-- 주문이 없는 유저도 포함, o.order_id는 NULL

-- RIGHT JOIN: 반대 방향 (LEFT JOIN으로 대체 가능, 잘 안 씀)

-- CROSS JOIN: 카테시안 곱 — 의도적 사용 외에는 위험
SELECT a.val, b.val
FROM table_a a
CROSS JOIN table_b b;

-- 안티 조인 패턴 (LEFT JOIN + IS NULL)
SELECT u.*
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.order_id IS NULL;  -- 주문한 적 없는 유저
```

**INNER JOIN vs WHERE 절 조인**

```sql
-- 아래 두 쿼리는 의미·성능 동일 (옵티마이저가 동일하게 처리)
SELECT * FROM a, b WHERE a.id = b.a_id;
SELECT * FROM a JOIN b ON a.id = b.a_id;
```

하지만 **가독성·유지보수**를 이유로 명시적 JOIN 문법이 표준이다. 암시적 조인 문법(`FROM a, b`)에서 `WHERE` 절을 누락하면 의도치 않은 카테시안 곱이 발생한다.

---

### 5. 서브쿼리

```sql
-- 스칼라 서브쿼리 (단일 값 반환)
SELECT
    order_id,
    amount,
    (SELECT AVG(amount) FROM orders) AS overall_avg
FROM orders;

-- IN 서브쿼리
SELECT * FROM users
WHERE id IN (
    SELECT DISTINCT user_id FROM orders WHERE status = 'VIP'
);

-- EXISTS — IN보다 큰 집합에서 빠른 경우 많음
SELECT * FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id AND o.status = 'VIP'
);
```

**IN vs EXISTS 성능 차이**

- `IN`: 서브쿼리 전체를 먼저 실행 → 결과 집합을 메모리에 올린 뒤 비교
- `EXISTS`: 외부 쿼리 각 행에 대해 서브쿼리를 실행 → **첫 번째 매칭 즉시 중단(short-circuit)**

서브쿼리 결과 집합이 클수록 `EXISTS`가 유리하다. 단, MySQL 옵티마이저가 `IN`을 내부적으로 `EXISTS`로 변환하는 경우도 있으므로 **실행 계획(`EXPLAIN`) 확인이 우선**이다.

---

### 6. EXPLAIN — 실행 계획 분석

실무에서 쿼리 작성만큼 중요한 것이 실행 계획 읽기다.

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 1 AND status = 'PENDING';
```

|컬럼|의미|나쁜 신호|
|---|---|---|
|`type`|접근 방식|`ALL` (풀스캔), `index`|
|`key`|실제 사용된 인덱스|`NULL`|
|`rows`|예상 스캔 행 수|수백만|
|`Extra`|추가 정보|`Using filesort`, `Using temporary`|

**type 우선순위** (좋음 → 나쁨)

```
const > eq_ref > ref > range > index > ALL
```

- `const`: PK 또는 유니크 인덱스 단일 값 조회 → 상수 시간
- `ref`: 비유니크 인덱스 동등 조건
- `range`: 인덱스 범위 스캔
- `ALL`: 풀 테이블 스캔 → 인덱스 추가 검토 필요

```sql
-- MySQL 8.0: EXPLAIN ANALYZE로 실제 실행 비용까지 확인
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 1;
```

> **MySQL 8.0 Reference — EXPLAIN Output Format** "The type column of EXPLAIN output describes how tables are joined. The following list describes the join types, ordered from the best type to the worst." _→ EXPLAIN 출력의 type 컬럼은 테이블 조인 방법을 설명한다. 최선에서 최악 순으로 조인 유형이 나열된다._

---

### 7. 트랜잭션 기본

```sql
START TRANSACTION;

UPDATE accounts SET balance = balance - 10000 WHERE id = 1;
UPDATE accounts SET balance = balance + 10000 WHERE id = 2;

-- 이체 성공
COMMIT;

-- 오류 시
ROLLBACK;
```

**ACID와 InnoDB**

| 속성              | InnoDB 구현                                        |
| --------------- | ------------------------------------------------ |
| **Atomicity**   | Undo Log — 실패 시 이전 상태로 복구                        |
| **Consistency** | 제약 조건(FK, UNIQUE), 트리거                           |
| **Isolation**   | MVCC(Multi-Version Concurrency Control) + 락      |
| **Durability**  | Redo Log(WAL) + `innodb_flush_log_at_trx_commit` |

격리 수준(Isolation Level)과 MVCC 동작은 별도의 깊은 주제다 — 여기서는 기본 문법 수준으로 한정.

```sql
-- 격리 수준 조회
SELECT @@transaction_isolation;

-- 세션 격리 수준 변경
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

---

## 이어지는 개념

|순서|개념|이유|
|---|---|---|
|1|**인덱스 내부 구조 (B+Tree, Clustered/Secondary Index)**|위 모든 쿼리의 성능이 결국 인덱스 설계로 귀결된다. 선행 지식 없이는 EXPLAIN이 의미 없음|
|2|**실행 계획 심화 (EXPLAIN ANALYZE, 옵티마이저 힌트)**|기본 문법을 알아도 쿼리가 느린 이유를 진단 못 하면 실무 불가|
|3|**트랜잭션 격리 수준 + MVCC**|동시성 환경에서 데이터 정합성 문제(Phantom Read, Non-Repeatable Read)는 실서비스에서 반드시 발생|
|4|**락 (Row Lock, Gap Lock, Next-Key Lock)**|MVCC 이후 자연스러운 흐름. Deadlock 원인 분석에 필수|
|5|**파티셔닝 + 샤딩**|대용량 데이터 처리 시 기본 문법의 한계에 부딪히는 지점|