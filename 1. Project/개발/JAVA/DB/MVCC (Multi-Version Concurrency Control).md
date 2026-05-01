
MVCC → 읽기/쓰기 동시성 
락 → 쓰기/쓰기 동시성

---
락 기반: 원본 수정 중 → 읽기 대기 
MVCC: 원본 수정 중 → 읽기는 Undo Log에서 내 시점 버전 찾아서 반환 → 대기 없음

---

## 1. Problem First

락 기반 동시성 제어의 한계.

```
락 기반:
Thread A: 데이터 읽는 중 (Read Lock)
Thread B: 데이터 쓰려고 대기... 대기... 대기...

Thread A: 데이터 쓰는 중 (Write Lock)
Thread C: 데이터 읽으려고 대기... 대기... 대기...
```

읽기와 쓰기가 서로를 막는다. 트래픽이 몰리면 Lock 대기가 병목이 된다. 이걸 해결하려고 나온 게 MVCC다.

---

## 2. Mechanics

**핵심 아이디어: 데이터를 덮어쓰지 않는다. 버전을 추가한다.**

```
락 기반: 원본을 직접 수정
MVCC:   원본은 두고 새 버전을 추가, 읽기는 자기 시점 버전을 봄
```

### 트랜잭션 ID

MVCC의 모든 동작은 트랜잭션 ID(이하 TxID) 기반이다. 트랜잭션이 시작될 때 DB가 단조 증가하는 ID를 부여한다.

```
TxID=100: BEGIN
TxID=101: BEGIN
TxID=102: BEGIN
...
```

이 ID로 "어느 트랜잭션이 만든 버전인가", "내가 볼 수 있는 버전인가"를 판단한다.

---

### MySQL InnoDB 구현 — Undo Log

```
실제 데이터 페이지: 항상 최신 버전 1개

[ name='영희', trx_id=200 ]  ← 현재 페이지

Undo Log:
[ name='철수', trx_id=100 ]  ← 이전 버전 (롤백 포인터로 연결)
[ name='김씨', trx_id=50  ]  ← 그 이전 버전
```

```
TxID=150인 읽기 요청:
1. 현재 페이지 확인 → trx_id=200 → 내 시점(150)보다 newer → 못 봄
2. Undo Log 타고 내려감 → trx_id=100 → 150보다 older이고 커밋됨 → 이걸 봄
3. '철수' 반환
```

읽기가 Undo Log를 체인처럼 타고 올라가며 자기 시점에 맞는 버전을 찾는다.

```
장점: 데이터 페이지는 항상 최신 1개 — 구조 단순
단점: 트랜잭션이 길면 Undo Log 체인이 길어짐
      → 읽기가 체인을 오래 탐색해야 함
```

---

### PostgreSQL 구현 — 테이블 내 다중 버전

```
테이블 페이지 안에 모든 버전이 공존:

[ name='김씨', xmin=50,  xmax=100 ]  ← TxID 50~100 사이에 유효
[ name='철수', xmin=100, xmax=200 ]  ← TxID 100~200 사이에 유효
[ name='영희', xmin=200, xmax=null ] ← TxID 200 이후 유효 (현재)

xmin: 이 버전을 만든 트랜잭션 ID
xmax: 이 버전을 삭제/수정한 트랜잭션 ID (null이면 현재 유효)
```

```
TxID=150인 읽기 요청:
→ xmin <= 150 < xmax 인 버전을 찾음
→ [ name='철수', xmin=100, xmax=200 ] 해당
→ '철수' 반환
```

Undo Log 없이 테이블 안에서 바로 찾는다.

```
장점: Undo Log 탐색 없음 — 버전 찾기 단순
단점: 오래된 버전(dead tuple)이 테이블에 계속 쌓임
      → VACUUM이 주기적으로 청소해야 함
      → VACUUM 타이밍 잘못되면 Table Bloat, 성능 저하
```

---

### 두 구현 비교

```
                MySQL InnoDB        PostgreSQL
─────────────────────────────────────────────
버전 저장 위치  Undo Log (별도)     테이블 페이지 (내부)
읽기 방식       체인 탐색           xmin/xmax 조건 검색
쓰기 방식       페이지 직접 수정    새 버전 append
청소 작업       Undo Log purge      VACUUM
긴 트랜잭션     Undo Log 비대화     dead tuple 누적
```

---

## 3. 격리 수준과 MVCC의 관계

MVCC가 "어느 버전을 보여줄지"를 결정하는 규칙이 **격리 수준**이다.

### READ COMMITTED

```
트랜잭션 안에서 SELECT할 때마다 새 스냅샷을 찍는다.

TxID=150 (READ COMMITTED):
  첫 번째 SELECT → 스냅샷: 149까지 커밋된 것만 봄
  ...TxID=145가 커밋...
  두 번째 SELECT → 스냅샷: 149+145 커밋된 것 봄 → 결과가 달라짐

→ Non-Repeatable Read 발생 가능
```

### REPEATABLE READ (MySQL InnoDB 기본값)

```
트랜잭션 시작 시점에 스냅샷을 한 번만 찍는다.

TxID=150 (REPEATABLE READ):
  트랜잭션 시작 → 스냅샷 고정: 149까지 커밋된 것만 봄
  ...TxID=145가 커밋...
  두 번째 SELECT → 여전히 고정된 스냅샷 사용 → 결과 동일

→ Non-Repeatable Read 없음
→ 단, 다른 트랜잭션의 write는 못 막음 → Lost Update 가능
→ 이걸 막으려면 비관적/낙관적 락이 필요 (처음으로 돌아옴)
```

---

## 4. 이 설계를 이렇게 한 이유 — 트레이드오프

**MVCC가 주는 것** 읽기와 쓰기가 서로를 안 막는다. 읽기에 Lock이 필요 없다. 처리량이 올라간다.

**MVCC가 잃는 것**

저장 공간. 버전을 쌓기 때문에 단순 덮어쓰기보다 디스크를 더 쓴다.

긴 트랜잭션이 독이 된다.

```
TxID=100짜리 트랜잭션이 1시간째 열려있음

MySQL: Undo Log를 1시간치 보관해야 함
       → 그 동안 Undo Log purge 불가 → 비대화

PostgreSQL: dead tuple을 1시간치 청소 못 함
            → VACUUM이 해당 버전을 못 지움 → Table Bloat
```

긴 트랜잭션 하나가 시스템 전체 성능을 갉아먹는 이유가 여기 있다.

---

## 5. 이어지는 개념

**트랜잭션 격리 수준** — MVCC가 버전을 만드는 메커니즘이라면, 격리 수준은 그 버전을 언제 어떻게 보여줄지 규칙이다. REPEATABLE READ에서 Phantom Read가 왜 발생하고, InnoDB의 넥스트 키 락이 어떻게 이를 막는지로 이어진다.

**비관적/낙관적 락** — MVCC는 읽기 충돌을 해결하지만, 쓰기 충돌(Lost Update)은 해결하지 못한다. 그 구멍을 막는 게 락 전략이다. 이 대화의 출발점으로 다시 연결된다.