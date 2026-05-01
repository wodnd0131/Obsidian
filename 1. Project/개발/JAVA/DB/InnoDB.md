
MySQL의 기본 스토리지 엔진이다. "스토리지 엔진"이란 데이터를 디스크에 어떻게 저장하고, 읽고, 동시성을 어떻게 처리할지를 담당하는 MySQL 내부 모듈이다.

MySQL은 스토리지 엔진을 교체할 수 있는 구조로 설계되어 있고, InnoDB 외에도 MyISAM, Memory 등이 있다. MySQL 5.5 이후부터 InnoDB가 기본값이다.

---

## InnoDB가 생기기 전 — MyISAM의 고통

초기 MySQL의 기본 엔진인 MyISAM은 구조가 단순했다.

```
MyISAM의 동시성 처리:
쓰기 요청 → 테이블 전체에 Lock
→ 그 동안 모든 읽기/쓰기 대기

게시판 조회 1000명 + 글쓰기 1명
→ 글쓰기 하는 동안 1000명 전부 대기
```

트랜잭션도 없었다. 아래 상황에서 복구 방법이 없었다.

```sql
UPDATE accounts SET balance = balance - 10000 WHERE id = 1;
-- 서버 크래시 발생
UPDATE accounts SET balance = balance + 10000 WHERE id = 2;
-- 두 번째 쿼리는 영영 실행 안 됨 → 돈이 사라짐
```

InnoDB는 이 두 문제 — **동시성**과 **트랜잭션** — 를 해결하기 위해 만들어졌다.

---

## InnoDB의 핵심 구조 세 가지

### 1. MVCC — 읽기와 쓰기가 서로를 안 막는다

InnoDB는 데이터를 수정할 때 기존 데이터를 지우지 않는다. **버전을 쌓는다.**

```
원본 row: { id=1, name="철수", version=1 }

Thread A가 UPDATE 시작 (아직 커밋 안 함)
→ 새 버전: { id=1, name="영희", version=2 } ← Thread A만 봄
→ 구 버전: { id=1, name="철수", version=1 } ← Undo Log에 보관

Thread B가 SELECT
→ "내 트랜잭션 시작 시점에 커밋된 버전만 봐"
→ 구 버전 "철수"를 읽음 → Thread A의 작업에 전혀 영향 없음
```

이게 앞서 나온 **낙관적 락/비관적 락 설명에서 "일반 SELECT는 블로킹 안 된다"고 한 이유**다. MVCC가 그 보장을 만든다.

> **MySQL 공식 문서:** "InnoDB is a multi-versioned storage engine. It keeps information about old versions of changed rows to support transactional features such as concurrency and rollback." _(번역: InnoDB는 멀티버전 스토리지 엔진이다. 동시성과 롤백 같은 트랜잭션 기능을 지원하기 위해 변경된 row의 이전 버전 정보를 보관한다.)_

### 2. 트랜잭션 + Redo/Undo Log — 장애 나도 데이터 안 잃는다

```
트랜잭션 커밋 흐름:

1. Undo Log 기록 (롤백용 — 이전 값 보관)
2. 변경 내용을 메모리(Buffer Pool)에 적용
3. Redo Log(WAL)에 기록 → 디스크 fsync
4. 클라이언트에 "커밋 완료" 응답

서버가 3번 직후 크래시나도:
→ 재시작 시 Redo Log로 커밋된 변경 재적용
→ 데이터 손실 없음
```

이것이 ACID의 **Durability(지속성)** 를 구현하는 방식이다.

### 3. 클러스터드 인덱스 — PK가 곧 데이터 파일이다

InnoDB는 데이터를 **PK 순서로 정렬해서 저장**한다. 인덱스와 데이터가 분리되어 있지 않다.

```
PK로 조회:  SELECT * WHERE id = 100
→ B-Tree 인덱스 탐색 → 그 위치가 곧 실제 데이터
→ 1번의 I/O

보조 인덱스(name)로 조회: SELECT * WHERE name = '철수'
→ name 인덱스 탐색 → PK 값 획득 → PK로 다시 탐색(클러스터드)
→ 2번의 I/O (이를 "더블 룩업"이라 부름)
```

앞서 나온 **넥스트 키 락이 "인덱스 레코드"에 걸린다**고 한 것도 여기서 나온다. InnoDB의 모든 락은 실제로 인덱스 위에 걸린다. 인덱스가 없는 컬럼으로 조회하면 테이블 전체에 락이 걸린다.

---

## 앞서 배운 개념들과의 연결

지금까지 나온 개념들이 InnoDB 위에서 동작한다.

|개념|InnoDB에서 어떻게 구현되나|
|---|---|
|비관적 락 (`FOR UPDATE`)|InnoDB의 넥스트 키 락 (레코드 락 + 갭 락)|
|낙관적 락 (version 체크)|InnoDB의 MVCC + 원자적 UPDATE|
|트랜잭션 격리 수준|MVCC의 버전 가시성 규칙으로 구현|
|SSE 중 "일반 SELECT는 안 막힌다"|MVCC가 읽기에 Lock 불필요하게 만듦|

---

## 이어지는 개념

**트랜잭션 격리 수준 (Isolation Level)** 이 다음이다. InnoDB의 MVCC가 어떤 버전을 "보여줄지" 결정하는 규칙이 격리 수준이다. `READ COMMITTED`와 `REPEATABLE READ`의 차이가 MVCC 버전 스냅샷을 **언제 찍느냐**의 차이라는 걸 알면, 지금까지 나온 락 전략의 동작 경계를 정확하게 이해할 수 있다.