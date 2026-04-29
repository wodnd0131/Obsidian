### Problem First — ETag 없이 PUT을 쓰면 생기는 문제

두 명이 동시에 같은 문서를 수정한다.

```
서버의 초기 상태: { id: 1, content: "원본 내용", version: 1 }
```

```
시간순서:

1. 철수가 GET /docs/1 → { content: "원본 내용" } 받음
2. 영희가 GET /docs/1 → { content: "원본 내용" } 받음
3. 철수가 편집 완료 → PUT /docs/1 { content: "철수가 수정" } → 성공
4. 영희가 편집 완료 → PUT /docs/1 { content: "영희가 수정" } → 성공

최종 서버 상태: { content: "영희가 수정" }
```

철수의 수정이 영희의 덮어쓰기로 **조용히 사라진다.**  
두 사람 모두 성공 응답을 받았다.  
이게 **Lost Update 문제**다.

### ETag의 역할

ETag(Entity Tag)는 서버가 리소스의 **특정 버전에 붙이는 식별자**다.  
리소스가 바뀌면 ETag도 바뀐다.

```
GET /docs/1
← 200 OK
← ETag: "v1-abc123"
← { content: "원본 내용" }
```

구현 방식은 서버마다 다르지만 보통:

- 내용의 해시값 (MD5, SHA 등)
- DB의 버전 컬럼 값
- 마지막 수정 타임스탬프 해시

```java
// Spring에서 ETag 응답 예시
@GetMapping("/docs/{id}")
public ResponseEntity<Doc> getDoc(@PathVariable Long id) {
    Doc doc = docRepository.findById(id).orElseThrow();
    String etag = "\"" + doc.getVersion() + "\""; // 버전을 ETag로
    
    return ResponseEntity.ok()
        .eTag(etag)
        .body(doc);
}
```

---

## 4. If-Match 헤더 — ETag로 조건부 PUT

클라이언트는 PUT을 보낼 때 **"내가 본 버전이 아직 유효할 때만 적용해라"** 고 조건을 건다.

```
GET /docs/1
← ETag: "v1-abc123"
← { content: "원본 내용" }

PUT /docs/1
If-Match: "v1-abc123"   ← "이 버전일 때만 적용해라"
{ content: "수정된 내용" }
```

서버는 현재 ETag와 If-Match 값을 비교한다.

```java
@PutMapping("/docs/{id}")
public ResponseEntity<Doc> updateDoc(
    @PathVariable Long id,
    @RequestHeader("If-Match") String ifMatch,
    @RequestBody DocUpdateRequest request
) {
    Doc doc = docRepository.findById(id).orElseThrow();
    String currentEtag = "\"" + doc.getVersion() + "\"";
    
    if (!currentEtag.equals(ifMatch)) {
        // 내가 본 버전이 아님 → 누군가 먼저 수정했음
        return ResponseEntity.status(412).build(); // 412 Precondition Failed
    }
    
    doc.setContent(request.getContent());
    doc.setVersion(doc.getVersion() + 1);
    docRepository.save(doc);
    
    return ResponseEntity.ok(doc);
}
```

### 동시 수정 시나리오 재연

```
1. 철수  GET /docs/1  → ETag: "v1", content: "원본"
2. 영희  GET /docs/1  → ETag: "v1", content: "원본"

3. 철수  PUT /docs/1
         If-Match: "v1"
         { content: "철수가 수정" }
         → 서버: 현재 ETag "v1" == "v1" ✓ → 적용
         → 서버 ETag가 "v2"로 바뀜

4. 영희  PUT /docs/1
         If-Match: "v1"          ← 영희가 본 버전
         { content: "영희가 수정" }
         → 서버: 현재 ETag "v2" != "v1" ✗
         → 412 Precondition Failed 반환

영희: "수정 충돌이 발생했습니다. 최신 버전을 다시 불러오세요."
```

철수의 수정이 사라지지 않는다. 영희는 충돌을 명시적으로 인지하고 다시 시도한다.

---

## 5. 낙관적 잠금이란 이름이 붙은 이유

### 비관적 잠금(Pessimistic Lock)과 비교

```
비관적: "충돌이 날 거라고 가정" → 미리 잠근다
낙관적: "충돌이 안 날 거라고 가정" → 일단 진행, 마지막에 확인
```

```
비관적 잠금:
철수가 /docs/1을 열면 → 서버가 잠금
영희가 /docs/1을 열려 하면 → 대기 (또는 실패)
철수가 저장하면 → 잠금 해제

낙관적 잠금:
철수와 영희가 동시에 열 수 있음
저장 시점에 "내가 본 버전이 맞는지" 확인
틀리면 → 충돌 처리
```

HTTP는 비연결형 프로토콜이라 비관적 잠금을 HTTP 레벨에서 구현하기 어렵다.  
(세션을 유지해야 하는데, HTTP는 요청마다 독립적이다)

ETag + If-Match가 낙관적 잠금을 HTTP 레벨에서 구현하는 표준 방법이다.

### 낙관적 잠금이 잃는 것

| |낙관적 잠금|비관적 잠금|
|---|---|---|
|충돌이 드물 때|빠르다|불필요한 대기 발생|
|충돌이 잦을 때|재시도 반복 → 처리량 저하|순서대로 처리|
|구현|클라이언트가 재시도 로직 필요|서버가 잠금 관리|

문서 편집처럼 **같은 리소스를 여러 명이 동시에 수정하는 빈도가 낮은** 상황에 낙관적 잠금이 맞다.  
재고 차감처럼 **충돌이 빈번하고 정합성이 치명적인** 경우엔 DB 레벨의 비관적 잠금이 낫다.

---

## 6. 이어지는 개념

1. **HTTP 캐시 (Cache-Control, ETag의 If-None-Match)** — ETag는 충돌 방지뿐 아니라 캐시 갱신에도 쓰인다. `If-None-Match`로 "변경된 게 없으면 304만 줘라" 구현. 지금 배운 ETag의 다른 활용이다.
    
2. **DB 낙관적 잠금 (@Version in JPA)** — HTTP ETag와 동일한 개념이 DB 레이어에도 있다. Spring Data JPA의 `@Version`이 이걸 구현한다. HTTP 레이어와 DB 레이어에서 같은 패턴이 어떻게 조합되는지 이어진다.
    
3. **분산 환경에서의 동시성 — Redis 기반 분산 락** — 낙관적 잠금으로 해결 안 되는 충돌 빈도가 높거나, 여러 서비스가 같은 리소스를 건드리는 경우. 다음 레벨의 동시성 제어다.