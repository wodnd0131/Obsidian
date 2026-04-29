# REST API

---

## 1. Problem First

REST 이전, 또는 REST를 무시하고 만든 API가 실제로 어떤 문제를 만드는지.

### 시나리오 1 — 행위가 URL에 난무한다

```
POST /getUserInfo
POST /createUser
POST /deleteUser
POST /updateUserStatus
POST /getUserList
POST /searchUsers
```

팀이 10명이면 10가지 네이밍 방식이 나옵니다. 6개월 뒤 신규 입사자는 `/getUserInfo`와 `/getUser`가 뭐가 다른지 코드를 열어봐야 압니다. API가 100개가 넘으면 **문서 없이는 쓸 수 없는 API**가 됩니다.

### 시나리오 2 — HTTP를 택배 봉투로만 쓴다

```java
// 요청
POST /api/users
{ "action": "delete", "userId": 123 }

// 응답 — 항상 200
HTTP/1.1 200 OK
{ "result": "fail", "message": "User not found" }
```

- 삭제 실패인데 HTTP 200이 옵니다
- API Gateway, 로드밸런서, CDN이 200을 **성공**으로 캐싱합니다
- 모니터링 도구가 에러율 0%를 보고합니다
- 클라이언트는 `result` 필드를 매번 파싱해서 실패를 감지해야 합니다

HTTP가 이미 설계해둔 상태 표현 체계를 전부 버리고 직접 재구현하는 것입니다.

### 시나리오 3 — 캐시가 불가능한 구조

```
GET /getProductList  (POST로 바꿔도 동작 동일)
```

HTTP GET은 브라우저, CDN, 프록시가 캐시할 수 있습니다. POST는 캐시하지 않습니다. 조회 API를 POST로 만드는 순간 **인프라 레벨 캐시를 포기**합니다. 트래픽 100% 가 오리진 서버까지 도달합니다.

---

## 2. Mechanics

### REST가 뭔지 먼저 정확히

REST는 Roy Fielding이 2000년 박사 논문 _"Architectural Styles and the Design of Network-based Software Architectures"_ 에서 정의한 **아키텍처 스타일**입니다.

프로토콜도, 표준도, 스펙도 아닙니다. **6가지 제약 조건의 집합**입니다. 이 제약을 전부 만족하면 RESTful, 아니면 REST가 아닙니다.

---

### 6가지 제약 조건 — 표면 말고 이유까지

**① Client-Server**

클라이언트와 서버를 분리합니다.

- 클라이언트는 UI/UX에 집중
- 서버는 데이터 저장/비즈니스 로직에 집중
- 둘이 독립적으로 진화 가능

실무 임팩트: React 앱과 Spring 서버가 독립 배포 가능한 이유의 아키텍처 근거입니다.

---

**② Stateless**

서버가 클라이언트의 세션 상태를 저장하지 않습니다. **모든 요청은 그 자체로 완결**되어야 합니다.

```java
// Stateful — 서버가 로그인 상태를 기억
GET /next-page          // "이전에 어디 있었는지 서버가 앎"
Authorization: (없음)

// Stateless — 요청 자체에 모든 컨텍스트 포함
GET /products?page=2
Authorization: Bearer eyJhbGci...
```

왜 이게 중요한가:

서버가 상태를 가지면 **특정 서버 인스턴스에 요청이 고정**됩니다. 오토스케일링으로 서버를 10대로 늘려도, 세션이 1번 서버에 있으면 1번 서버로만 가야 합니다. Sticky Session 문제입니다.

Stateless면 어떤 서버 인스턴스도 요청을 처리할 수 있습니다. 로드밸런서가 자유롭게 분산할 수 있고, 서버가 죽어도 다른 서버가 즉시 대체합니다.

---

**③ Cacheable**

응답은 캐시 가능 여부를 명시해야 합니다.

```
HTTP/1.1 200 OK
Cache-Control: max-age=3600        # 1시간 캐시 가능
ETag: "33a64df551425fcc55e"        # 리소스 버전 식별자
```

HTTP 메서드별 기본 캐시 가능 여부:

|Method|캐시 가능 여부|
|---|---|
|GET|✅ 기본 캐시 가능|
|HEAD|✅ 기본 캐시 가능|
|POST|⚠️ 명시적 헤더 있을 때만|
|PUT/DELETE/PATCH|❌ 캐시 불가|

---

**④ Uniform Interface**

REST의 핵심이자 가장 많이 오해되는 제약입니다. 4개의 하위 제약으로 구성됩니다.

**4-1. Resource Identification in Requests**

리소스는 URI로 식별합니다. 리소스는 명사입니다.

```
# 나쁜 예 — 행위가 URI에
/getUser/123
/deleteUser/123
/updateUser/123

# 좋은 예 — 리소스가 URI에, 행위는 HTTP 메서드에
GET    /users/123
DELETE /users/123
PATCH  /users/123
```

**4-2. Manipulation of Resources Through Representations**

클라이언트는 리소스 자체를 갖는 게 아니라 **표현(Representation)** 을 받습니다. 같은 리소스를 JSON으로도, XML로도, HTML로도 표현할 수 있습니다.

```
GET /users/123
Accept: application/json     → JSON 응답
Accept: application/xml      → XML 응답
Accept: text/html            → HTML 응답
```

**4-3. Self-descriptive Messages**

메시지는 자기 자신을 설명해야 합니다. Content-Type, 상태 코드 등이 그 수단입니다.

```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /users/123

{"id": 123, "name": "홍길동"}
```

**4-4. HATEOAS** ← 실무에서 가장 많이 무시되는 제약

Hypermedia As The Engine Of Application State.

응답에 **다음에 할 수 있는 행동의 링크**를 포함해야 합니다.

```json
{
  "id": 123,
  "name": "홍길동",
  "status": "ACTIVE",
  "_links": {
    "self":   { "href": "/users/123" },
    "orders": { "href": "/users/123/orders" },
    "deactivate": { "href": "/users/123/deactivate", "method": "POST" }
  }
}
```

클라이언트가 API 구조를 **하드코딩하지 않아도** 됩니다. 응답을 보고 다음 URL을 동적으로 따라갑니다. GitHub REST API, PayPal API 등이 부분적으로 구현합니다.

현실에서 대부분의 API는 HATEOAS를 구현하지 않습니다. 그래서 Fielding은 2008년 블로그에서 _"HATEOAS 없으면 REST가 아니다"_ 라고 명시적으로 지적했습니다.

> "If the engine of application state (and hence the API) is not being driven by hypertext, then it cannot be RESTful and cannot be a REST API." — Roy Fielding, 2008 (https://roy.gbiv.com/untangled/2008/rest-apis-must-be-hypertext-driven)
> 
> (한국어) "애플리케이션 상태의 엔진이 하이퍼텍스트에 의해 구동되지 않는다면, RESTful하지 않은 것이며 REST API라 부를 수 없다."

---

**⑤ Layered System**

클라이언트는 자신이 서버와 직접 통신하는지, 중간 레이어(프록시, 게이트웨이, 로드밸런서)를 통하는지 알 수 없어야 합니다.

```
Client → CDN → API Gateway → Load Balancer → Server
```

클라이언트 입장에서는 그냥 `https://api.example.com` 입니다. 중간에 무엇이 있는지 모릅니다. 이 제약 덕분에 인프라를 클라이언트 변경 없이 자유롭게 바꿀 수 있습니다.

---

**⑥ Code on Demand (선택적)**

서버가 실행 가능한 코드를 클라이언트에 전송할 수 있습니다. JavaScript가 대표적입니다. 유일한 선택적 제약입니다.

---

### HTTP 메서드 — 놓치기 쉬운 시맨틱

```
GET    — 조회. 안전(Safe) + 멱등(Idempotent)
POST   — 생성/처리. 안전 ❌, 멱등 ❌
PUT    — 전체 교체. 안전 ❌, 멱등 ✅
PATCH  — 부분 수정. 안전 ❌, 멱등 ⚠️
DELETE — 삭제. 안전 ❌, 멱등 ✅
HEAD   — GET인데 바디 없음. 안전 ✅, 멱등 ✅
```

**Safe(안전)**: 서버 상태를 변경하지 않음 **Idempotent(멱등)**: 동일 요청을 N번 해도 결과가 같음

멱등성이 왜 중요한가:

```java
// DELETE /users/123 — 멱등
// 1번 호출: 삭제됨 → 204 No Content
// 2번 호출: 이미 없음 → 404 Not Found
// 결과(리소스 상태): 동일 — 없음

// POST /orders — 멱등 아님
// 1번 호출: 주문 생성 → 201 Created
// 2번 호출: 주문 또 생성 → 201 Created
// 결과: 주문 2개 생성됨
```

네트워크 타임아웃 발생 시, 클라이언트는 재시도를 해야 할지 말지 판단해야 합니다. **멱등한 메서드는 재시도가 안전**합니다. POST는 재시도하면 중복 생성 위험이 있습니다.

**PATCH의 멱등성 ⚠️:**

```java
// 멱등한 PATCH — 특정 값으로 설정
PATCH /users/123
{ "status": "INACTIVE" }   // N번 해도 결과 동일

// 멱등하지 않은 PATCH — 상대값 변경
PATCH /users/123
{ "point": "+100" }        // 호출할 때마다 포인트 증가
```

RFC 5789는 PATCH가 멱등일 수도 아닐 수도 있다고 명시합니다. 설계에 따라 달라집니다.

> RFC 5789, Section 2: "A PATCH request can be issued in such a way as to be idempotent, which also helps prevent bad outcomes from collisions between two PATCH requests on the same resource in a similar time frame."
> 
> (한국어) "PATCH 요청은 멱등하게 설계될 수 있으며, 이는 동일 리소스에 대한 두 PATCH 요청의 충돌로 인한 문제를 방지하는 데 도움이 된다."

---

### HTTP 상태 코드 — 자주 틀리는 것들

```
200 OK          — 조회/수정 성공
201 Created     — 생성 성공. Location 헤더에 생성된 리소스 URI 포함
204 No Content  — 성공인데 응답 바디 없음 (DELETE 후 등)
400 Bad Request — 클라이언트 요청 자체가 잘못됨 (유효성 검증 실패)
401 Unauthorized — 인증 안 됨 (로그인 안 함)
403 Forbidden   — 인증은 됐는데 권한 없음
404 Not Found   — 리소스 없음
405 Method Not Allowed — URI는 맞는데 메서드가 잘못됨
409 Conflict    — 리소스 충돌 (중복 생성 시도 등)
422 Unprocessable Entity — 문법은 맞는데 비즈니스 규칙 위반
429 Too Many Requests — Rate limit 초과
500 Internal Server Error — 서버 내부 오류
503 Service Unavailable   — 서버 일시적 불가 (점검, 과부하)
```

**401 vs 403 — 실무에서 자주 혼동:**

```
401: "당신이 누군지 모릅니다. 로그인하세요"
403: "당신이 누군지는 아는데, 이건 못 합니다"
```

**404 vs 403의 보안 트레이드오프:**

권한 없는 리소스에 접근할 때 403을 주면 "리소스가 존재함"을 알려줍니다. 보안상 민감한 경우 404를 주기도 합니다. GitHub private repo에 접근하면 403이 아니라 404를 반환하는 이유입니다.

---

### URI 설계 — 놓치기 쉬운 규칙들

```
# 계층 관계는 /로 표현
GET /users/123/orders/456

# 동사 금지, 명사 복수형 사용
❌ /getUsers
✅ /users

# 소문자만
❌ /Users/123
✅ /users/123

# 언더스코어 대신 하이픈
❌ /user_orders
✅ /user-orders

# 파일 확장자 금지
❌ /users/123.json
✅ /users/123  (Accept 헤더로 포맷 협상)

# 필터/정렬/페이징은 쿼리 파라미터
GET /users?status=active&sort=createdAt&page=2&size=20
```

**행위를 표현해야 할 때:**

```
# 리소스로 추상화
POST /users/123/deactivate      # 비활성화라는 "행위"를 리소스로
POST /orders/456/cancel         # 취소
POST /payments/789/refund       # 환불
```

완전히 CRUD로 맞아 떨어지지 않는 행위는 **행위 자체를 리소스화**하는 게 현실적인 타협점입니다.

---

## 3. 공식 근거

- Roy Fielding 박사 논문 (2000): https://ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm
- Roy Fielding 블로그 (2008), HATEOAS 필수성: https://roy.gbiv.com/untangled/2008/rest-apis-must-be-hypertext-driven
- RFC 7231 — HTTP/1.1 Semantics (GET/POST/PUT/DELETE 등 메서드 정의)
- RFC 5789 — PATCH Method for HTTP
- RFC 7807 — Problem Details for HTTP APIs (에러 응답 표준 포맷)

---

## 4. 설계 트레이드오프

**HATEOAS를 구현하면:**

- 클라이언트가 서버 URL 구조에 종속되지 않음
- 응답 크기 증가, 클라이언트 파싱 복잡도 증가
- 팀 간 합의 비용이 큼
- 그래서 대부분 실무에서 포기합니다

**Stateless를 지키면:**

- 수평 확장이 자유로움
- 모든 요청에 인증 토큰을 포함해야 하므로 **요청 크기 증가**
- 서버가 컨텍스트를 캐싱할 수 없으므로 **반복 조회 비용 발생**

**REST를 엄격하게 지키면:**

- API가 예측 가능하고 일관성 있음
- 실시간 양방향 통신(WebSocket), 복잡한 쿼리(GraphQL), 고성능 바이너리 통신(gRPC)이 필요한 상황에서는 REST가 오히려 부적합합니다

---

## 5. 이어지는 개념

|순서|개념|이유|
|---|---|---|
|1|**HTTP 상세** (헤더, 캐시, 커넥션)|REST가 HTTP 위에 정의되므로, HTTP를 모르면 REST 설계 근거를 이해 못 함|
|2|**인증/인가** (JWT, OAuth 2.0)|Stateless 제약을 지키면서 인증을 어떻게 구현하는지가 바로 이어지는 실무 문제|
|3|**API 버전 관리**|외부에 공개하거나 클라이언트가 여럿이면 즉시 직면하는 문제|
|4|**GraphQL / gRPC**|REST의 한계를 어떤 상황에서 다른 것으로 대체하는지 이해하려면 REST를 먼저 알아야 함|
