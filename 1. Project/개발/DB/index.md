## 1. Problem First — 인덱스가 없었다면

```sql
SELECT * FROM orders WHERE user_id = 1234;
```

인덱스 없는 테이블에서 이 쿼리의 유일한 실행 방법:

```
페이지 1 읽기 → 모든 row 검사 → user_id = 1234인가?
페이지 2 읽기 → 모든 row 검사 → user_id = 1234인가?
페이지 3 읽기 → ...
...
페이지 100,000 읽기까지 반복
```

**O(N) — 데이터가 늘어날수록 선형으로 느려진다.**

"전용 테이블을 만든다"는 이해가 맞는 방향인데, 정확히는 **별도의 B+Tree 자료구조를 만든다**는 것이다. 왜 하필 B+Tree인지, 그 구조가 어떻게 생겼는지가 인덱스 이해의 핵심이다.

---

## 2. Mechanics

### 왜 B+Tree인가 — 자료구조 선택 근거

인덱스에 쓸 자료구조의 요구사항:

```
① 단일 값 조회:  WHERE id = 1234         → O(log N)
② 범위 조회:     WHERE id BETWEEN 100 AND 200  → 연속 접근 가능
③ 정렬:          ORDER BY id             → 이미 정렬된 상태
④ 디스크 친화적: 페이지(16KB) 단위로 효율적 읽기
```

다른 자료구조와 비교:

```
해시 테이블:  ① O(1) ✅  ②범위 불가 ❌  ③정렬 불가 ❌
이진 탐색 트리: 균형 깨지면 O(N)으로 퇴화 ❌
B Tree:      ②범위 조회 시 리프 아닌 노드도 순회해야 함 ❌
B+Tree:      ①②③④ 모두 충족 ✅
```

**B+Tree가 선택된 이유는 범위 조회 + 디스크 I/O 최적화를 동시에 만족하기 때문이다.**

---

### B+Tree 구조

```
                    [루트 노드]
                   [50 | 100]
                  /     |     \
         [내부 노드]  [내부 노드]  [내부 노드]
         [20 | 35]   [60 | 80]  [110 | 130]
        /    |    \
[리프] [리프] [리프] [리프] ...
[1,2,3][20,25][35,40][50,55] ...
  ↕      ↕      ↕      ↕
 (리프 노드들이 LinkedList로 연결)
```

**핵심 특징 두 가지:**

**① 실제 데이터(또는 PK)는 리프 노드에만 있다**

```
루트/내부 노드: 탐색 경로만 저장 (키 값 + 자식 포인터)
리프 노드:     실제 데이터 저장
```

내부 노드가 가볍기 때문에 하나의 페이지(16KB)에 수백~수천 개의 키를 넣을 수 있다. → 트리 높이가 낮아진다 → 루트에서 리프까지 I/O 횟수가 적다.

```
1억 건 테이블:
B+Tree 높이 ≈ 3~4 레벨
→ 단일 값 조회 = 디스크 I/O 3~4번으로 끝
```

**② 리프 노드가 LinkedList로 연결된다**

```
[리프1: 1,2,3] → [리프2: 4,5,6] → [리프3: 7,8,9] → ...
```

범위 조회 시 루트부터 다시 내려올 필요 없이 **리프를 따라 순차 스캔**한다.

```sql
WHERE id BETWEEN 4 AND 8
→ id=4가 있는 리프 찾기 (트리 탐색, O(log N))
→ 리프를 오른쪽으로 따라가며 id=8까지 읽기 (순차 I/O)
```

---

### InnoDB의 두 가지 인덱스

InnoDB는 인덱스가 두 종류이고 **구조가 근본적으로 다르다.**

#### 클러스터드 인덱스 (Clustered Index)

```
= PK 그 자체가 인덱스
= 리프 노드에 실제 row 데이터 전체가 들어있음
= 테이블당 1개만 존재
```

```
[클러스터드 인덱스 리프 노드]

[id=1 | name="Alice" | status="ACTIVE" | created_at=2024-01-01 | ...]
[id=2 | name="Bob"   | status="BANNED" | created_at=2024-01-02 | ...]
[id=3 | name="Carol" | status="ACTIVE" | created_at=2024-01-03 | ...]
```

**PK 순서대로 실제 데이터가 물리적으로 정렬되어 저장된다.**

```sql
-- 이 쿼리가 가장 빠른 이유
SELECT * FROM users WHERE id = 1234;
-- 클러스터드 인덱스 탐색 = 데이터 직접 접근, 추가 I/O 없음
```

#### 세컨더리 인덱스 (Secondary Index)

```
= PK 외의 컬럼에 만드는 인덱스
= 리프 노드에 인덱스 컬럼 값 + PK 값만 들어있음
= 테이블당 여러 개 가능
```

```
[created_at 세컨더리 인덱스 리프 노드]

[created_at=2024-01-01 | PK=9001]
[created_at=2024-01-01 | PK=234 ]
[created_at=2024-01-02 | PK=5678]
[created_at=2024-01-02 | PK=123 ]
```

created_at 순으로 정렬되어 있고, **실제 row 데이터 없이 PK만 저장**된다.

**세컨더리 인덱스 조회 = 2단계:**

```
1단계: 세컨더리 인덱스 탐색
       created_at = '2024-01-01' 찾기
       → PK = 9001, 234 획득

2단계: 클러스터드 인덱스 탐색 (테이블 조회, Table Lookup)
       PK = 9001로 클러스터드 인덱스 탐색 → 실제 row
       PK = 234로  클러스터드 인덱스 탐색 → 실제 row
```

이 2단계 접근이 바로 앞에서 설명한 **랜덤 I/O의 원인**이다.

```
PK=9001 → 디스크 A 위치
PK=234  → 디스크 B 위치  (A와 전혀 다른 곳)
PK=5678 → 디스크 C 위치
→ 랜덤 I/O
```

---

### 커버링 인덱스 — 2단계를 없애는 방법

세컨더리 인덱스의 2단계(Table Lookup)를 없애려면 **쿼리에 필요한 컬럼이 인덱스 안에 전부 있으면 된다.**

```sql
-- 이 쿼리:
SELECT user_id, created_at FROM orders WHERE created_at >= '2024-01-01';

-- created_at 단일 인덱스로는:
-- 1단계: 인덱스에서 created_at 조건 만족하는 PK 찾기
-- 2단계: PK로 클러스터드 인덱스 접근해서 user_id 가져오기 (랜덤 I/O)

-- 복합 인덱스 (created_at, user_id) 로는:
-- 1단계: 인덱스에서 created_at 조건 + user_id 바로 읽기
-- 2단계: 없음 ← 클러스터드 인덱스 접근 불필요
```

```sql
CREATE INDEX idx_covering ON orders (created_at, user_id);
```

EXPLAIN에서 `Extra: Using index`가 보이면 커버링 인덱스가 작동 중이다.

---

### 복합 인덱스 — 컬럼 순서가 전부다

```sql
CREATE INDEX idx_multi ON orders (status, created_at, user_id);
```

이 인덱스의 내부 정렬:

```
[status="ACTIVE"   | created_at=2024-01-01 | user_id=1  | PK]
[status="ACTIVE"   | created_at=2024-01-01 | user_id=5  | PK]
[status="ACTIVE"   | created_at=2024-01-02 | user_id=3  | PK]
[status="BANNED"   | created_at=2024-01-01 | user_id=2  | PK]
[status="INACTIVE" | created_at=2024-01-03 | user_id=7  | PK]
```

**앞 컬럼 기준으로 정렬 → 그 안에서 다음 컬럼 기준 정렬 → 재귀적**

#### 선두 컬럼 원칙 (Left-most Prefix Rule)

```sql
-- 인덱스: (status, created_at, user_id)

WHERE status = 'ACTIVE'                           -- ✅ 인덱스 사용 (status)
WHERE status = 'ACTIVE' AND created_at >= '2024'  -- ✅ 인덱스 사용 (status + created_at)
WHERE status = 'ACTIVE' AND user_id = 1           -- ✅ status만 인덱스, user_id는 필터
WHERE created_at >= '2024'                        -- ❌ 인덱스 사용 불가
WHERE user_id = 1                                 -- ❌ 인덱스 사용 불가
```

**왜 중간 컬럼을 건너뛰면 안 되는가:**

```
status 없이 created_at만 조건으로 주면:
인덱스는 [ACTIVE+2024-01-01], [ACTIVE+2024-01-02], [BANNED+2024-01-01]...
이렇게 status별로 섞여서 정렬되어 있음

created_at만으로 연속된 범위를 찾을 수 없음
→ 리프 LinkedList를 순차적으로 따라가는 게 불가능
→ 인덱스 의미 없음
```

#### 범위 조건 이후 컬럼은 인덱스 효과 없음

```sql
-- 인덱스: (status, created_at, user_id)

WHERE status = 'ACTIVE'          -- = 조건, 다음 컬럼 계속 사용 가능
  AND created_at >= '2024-01-01' -- 범위 조건, 여기서 인덱스 range 끝
  AND user_id = 1                -- 인덱스 사용 불가, 별도 필터링
```

**이유:**

```
status='ACTIVE' AND created_at >= '2024-01-01' 까지는
인덱스에서 연속된 범위를 찾을 수 있음

그런데 그 범위 안에서 user_id 순서는?
[ACTIVE | 2024-01-01 | user_id=5 ]
[ACTIVE | 2024-01-01 | user_id=1 ]  ← user_id 정렬 안 됨
[ACTIVE | 2024-01-02 | user_id=3 ]
→ user_id로 범위 탐색 불가, 매 row마다 필터링만 가능
```

따라서 복합 인덱스 설계 원칙:

```
1. = 조건 컬럼을 앞으로
2. 범위 조건 컬럼을 뒤로
3. 자주 쓰는 조합 기준으로 순서 결정
4. 커버링 인덱스 목적이면 SELECT 컬럼도 포함
```

---

### 인덱스가 있어도 안 쓰이는 케이스

```sql
-- ① 컬럼을 함수/연산으로 가공
WHERE YEAR(created_at) = 2024        -- ❌ 인덱스 못 씀
WHERE created_at >= '2024-01-01'     -- ✅

-- ② 묵시적 타입 변환
-- user_id 컬럼이 INT인데
WHERE user_id = '1234'               -- ❌ 문자열로 비교, 인덱스 못 씀
WHERE user_id = 1234                 -- ✅

-- ③ LIKE 앞쪽 와일드카드
WHERE name LIKE '%Alice'             -- ❌ 앞이 불확정이라 범위 특정 불가
WHERE name LIKE 'Alice%'             -- ✅

-- ④ OR 조건 (일부 경우)
WHERE status = 'ACTIVE' OR user_id = 1  -- 옵티마이저에 따라 풀스캔 선택
```

모두 같은 이유다. **인덱스는 정렬된 B+Tree인데, 가공/변환된 값은 그 순서를 파괴한다.**

---

## 3. 공식 근거

- **MySQL 8.0 — InnoDB Index Types**: [https://dev.mysql.com/doc/refman/8.0/en/innodb-index-types.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-index-types.html)
    
    > "Every InnoDB table has a special index called the clustered index that stores row data." (번역: 모든 InnoDB 테이블은 row 데이터를 저장하는 클러스터드 인덱스라는 특수 인덱스를 가진다.)
    
- **MySQL 8.0 — Multiple-Column Indexes**: [https://dev.mysql.com/doc/refman/8.0/en/multiple-column-indexes.html](https://dev.mysql.com/doc/refman/8.0/en/multiple-column-indexes.html)
    
- **MySQL 8.0 — Covering Index / Index Extensions**: [https://dev.mysql.com/doc/refman/8.0/en/index-extensions.html](https://dev.mysql.com/doc/refman/8.0/en/index-extensions.html)
    
- _High Performance MySQL 4th Ed._ — Chapter 8 "Indexing for High Performance" (3순위)
    

---

## 4. 트레이드오프

### 인덱스를 많이 만들면 생기는 비용

```
쓰기 작업(INSERT / UPDATE / DELETE) 시:
  → 테이블 데이터 변경
  → 관련된 모든 인덱스 B+Tree도 재조정 (노드 분할 등)
  → 인덱스 수에 비례해서 쓰기 비용 증가

저장 공간:
  → 인덱스마다 별도 B+Tree 공간 차지
  → 대형 테이블에서 인덱스가 테이블보다 클 수 있음
```

### PK 설계가 중요한 이유

클러스터드 인덱스 = PK 순서 = 물리적 저장 순서이기 때문에:

```
PK가 AUTO_INCREMENT(순차) 일 때:
  INSERT → 항상 B+Tree의 오른쪽 끝에 추가
  → 페이지 분할 최소화, 순차 I/O 유리

PK가 UUID(랜덤) 일 때:
  INSERT → B+Tree의 무작위 위치에 삽입
  → 페이지 분할 빈번, 물리적 단편화 발생
  → 쓰기 성능 저하, 범위 조회 성능 저하
```

Spring에서 `@GeneratedValue(strategy = GenerationType.IDENTITY)` 를 권장하는 이유가 여기 있다.

---

## 5. 이어지는 개념

|순서|개념|이유|
|---|---|---|
|1|**EXPLAIN 읽는 법 심화**|지금 배운 인덱스 구조가 EXPLAIN의 type/key/Extra에 정확히 매핑됨. 이론을 실무 도구로 연결하는 단계|
|2|**인덱스와 트랜잭션 — 갭 락(Gap Lock)**|인덱스 range scan이 락 범위를 결정함. 소프트딜리트 + 범위 조건 쿼리에서 데드락이 발생하는 원인|
|3|**MVCC와 인덱스**|SELECT가 인덱스를 통해 어느 버전의 row를 읽는지 — 트랜잭션 격리 수준과 연결|


