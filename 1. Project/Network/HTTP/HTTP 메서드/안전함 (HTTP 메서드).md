HTTP 메서드가 서버의 상태를 바꾸지 않으면 그 메서드가 **안전**하다고 말합니다. 다른 말로 하면, 읽기 작업만 수행하는 메서드는 안전합니다. 흔히 쓰이는 HTTP 메서드 중에서는 [`GET`](https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Methods/GET), [`HEAD`](https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Methods/HEAD), [`OPTIONS`](https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Methods/OPTIONS)가 안전합니다. 모든 안전한 메서드는 [[멱등성]]또한 갖지만, 모든 멱등성을 지닌 메서드가 안전한 것은 아닙니다. 
예컨대 [`PUT`](https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Methods/PUT)과 [`DELETE`](https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Methods/DELETE)는 둘 다 멱등성을 가졌지만 안전하지는 않은 메서드입니다.

그러나, 안전한 메서드가 읽기 전용의 의미를 내포하긴 하지만, 서버가 요청 정보와 통계 등을 기록함으로써 자신의 상태에 변경을 가하는 것도 가능합니다. 안전한 메서드 호출의 중요한 부분은 그 메서드를 호출해도 클라이언트가 서버의 상태 변화를 직접 요청하는 것이 아니므로 서버에 불필요한 부하를 주지 않을 것이란 점입니다. 브라우저는 안전한 메서드라면 서버에 해를 끼치지 않을 것임을 알 수 있기 때문에 자유롭게 호출할 수 있습니다. 이런 점을 활용해서 브라우저가 별다른 위험 없이도 프리페칭과 같은 동작을 수행할 수 있는 것입니다. 웹 크롤러 역시 안전한 메서드의 호출에 의존합니다.

안전한 메서드가 정적 파일만 제공해야 할 필요는 없습니다. 생성된 스크립트가 안전함을 보장하는 한, 서버는 안전한 메서드에 대한 응답을 즉시 생성할 수 있습니다. 다른 전자 상거래 웹 사이트에 주문을 넣는 것과 같이 외부 효과를 유발하는 것은 안됩니다.

메서드의 안전함을 준수하는 것은 온전히 서버 애플리케이션의 책임으로, Apache, Nginx, IIS 등 웹 서버 스스로는 안전함을 강제하지 못합니다. 서버 애플리케이션은 특히 [`GET`](https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Methods/GET) 요청을 받았을 때 자신의 상태가 바뀌지 않도록 해야 합니다.

다음은 서버 상태를 바꾸지 않는, 안전한 메서드의 호출입니다.

```
GET /pageX.html HTTP/1.1
```

다음은 서버 상태를 바꿀 수도 있는, 안전하지 않은 메서드의 호출입니다.

```
POST /pageX.html HTTP/1.1
```

마지막으로 멱등성을 가졌지만 안전하지는 않은 메서드의 호출입니다.

```
DELETE /idX/delete HTTP/1.1
```
## 같이 보기

- HTTP 명세에서 [안전함](https://httpwg.org/specs/rfc9110.html#safe.methods)의 정의.
- 일반적으로 쓰이는 안전한 메서드: [`GET`](https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Methods/GET), [`HEAD`](https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Methods/HEAD), [`OPTIONS`](https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Methods/OPTIONS)
- 일반적으로 쓰이는 안전하지 않은 메서드: [`PUT`](https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Methods/PUT), [`DELETE`](https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Methods/DELETE), [`POST`](https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Methods/POST)
---
# 프리페칭 / 웹 크롤러 / "정적 파일 아니어도 됨"

---

## 1. 프리페칭(Prefetching)이 뭔가

### Problem First

브라우저가 링크를 **클릭한 다음에야** 다음 페이지 요청을 보내면,  
사용자는 항상 네트워크 왕복(RTT) + 서버 처리 시간을 기다려야 한다.

```
사용자 클릭 → GET /product/123 → 서버 처리 → 응답 → 렌더링
              ←── 이 시간 동안 사용자는 흰 화면 ──→
```

### Mechanics

브라우저는 사용자가 **아직 클릭하지 않은** 링크의 리소스를 미리 가져올 수 있다.

```html
<link rel="prefetch" href="/product/123">
```

또는 브라우저가 자체 판단으로 `<a href>` 링크를 미리 GET 요청하기도 한다.

**여기서 핵심 조건:** 브라우저가 이걸 할 수 있는 전제가  
_"GET은 안전하다 — 내가 요청해도 서버 상태를 바꾸지 않는다"_ 는 보장이다.

만약 GET이 안전하다는 보장이 없다면?

```
브라우저가 /order/confirm을 미리 GET했는데
→ 서버가 그 GET 요청에서 주문을 확정해버림
→ 사용자는 클릭도 안 했는데 주문이 나감
```

이게 실제로 2005년에 일어난 사건이다.  
Google Web Accelerator가 페이지의 링크를 미리 GET으로 긁었는데,  
일부 사이트가 `GET /post/delete?id=123` 같은 구조를 쓰고 있었고  
→ 데이터가 대량 삭제됐다.

**RFC 9110이 "안전한 메서드" 개념을 명시하는 이유가 바로 이것이다.**

> RFC 9110 §9.2.1: "Request methods are considered 'safe' if their defined semantics are essentially read-only ... This distinction is important because automated retrieval agents (crawlers) and cache pre-population need to make requests without worrying about causing harm."
> 
> _→ 자동화 에이전트와 캐시 선제 요청이 "해를 끼칠 걱정 없이" 요청할 수 있으려면 안전성 구분이 필요하다._

---

## 2. 웹 크롤러가 "의존"한다는 게 무슨 뜻인가

크롤러(Googlebot 등)는 사이트를 탐색할 때 **모든 링크를 GET으로 요청**한다.

크롤러 입장에서는:

- 이 사이트가 내부적으로 어떤 로직을 돌리는지 모른다
- 그냥 링크가 있으면 따라간다

크롤러가 안전하게 동작할 수 있는 전제가  
_"GET/HEAD 요청은 서버 상태를 바꾸지 않는다"_ 는 **관례적 약속**이다.

이 약속이 깨지면:

```
Googlebot이 /admin/user/delete?id=모든사용자 를 GET으로 방문
→ 서버가 실제로 삭제 처리
→ 사이트 데이터 전부 날아감
```

실제로 이런 구조("GET으로 삭제")를 짠 사이트들이 크롤러나 프리페처에 의해  
의도치 않게 데이터가 날아간 사례가 있었다.

크롤러는 사이트마다 별도 계약을 맺지 않는다.  
"GET은 읽기다"라는 HTTP 명세의 약속을 믿고 동작한다.  
그래서 **의존**한다는 표현이 쓰인 것이다.

---

## 3. "안전한 메서드가 정적 파일만 제공해야 할 필요는 없다"

이 문장이 말하는 맥락을 이해하려면  
"안전하다 = 정적 파일 서빙"이라는 오해를 먼저 짚어야 한다.

### 오해: 안전하다 = 파일을 그냥 읽어서 돌려준다

```
GET /index.html → 디스크에서 파일 읽어서 응답
```

이건 당연히 안전하다. 상태 변경이 없으니까.

### 실제: 안전하다 = 서버 상태를 바꾸지 않는다 (생성 방식은 무관)

```java
// 이것도 안전한 GET이다
@GetMapping("/products")
public List<Product> getProducts() {
    // DB 조회 → 결과를 계산 → JSON 생성해서 응답
    // 서버/DB 상태는 변경 없음
    return productRepository.findAll();
}
```

응답이 매번 동적으로 생성되더라도,  
**그 과정에서 서버 상태를 변경하지 않으면** 안전한 메서드다.

반대로 이건 안전하지 않다:

```java
@GetMapping("/products")  // ← GET인데
public List<Product> getProductsAndLog() {
    List<Product> result = productRepository.findAll();
    auditLog.record("조회됨", userId);  // ← 외부 시스템 상태 변경
    emailService.sendViewAlert();       // ← 외부 효과 발생
    return result;
}
```

단순 로깅(서버 내부 로그 파일 기록)은 RFC 9110이 허용하는 범위로 본다.  
하지만 외부 시스템에 주문을 넣거나, 이메일을 보내거나, 결제를 트리거하는 건  
"클라이언트가 직접 요청한 상태 변경"이 아니더라도 안전하지 않은 것으로 봐야 한다.

> RFC 9110 §9.2.1: "a safe method need not be completely safe ... It is this distinction between a client-requested state change and an incidental side-effect of a response that separates safe methods from unsafe ones"
> 
> _→ 클라이언트가 요청한 상태 변경인지 아닌지가 핵심이지, 응답이 동적으로 생성됐는지가 기준이 아니다._

### 한 줄 요약

|구분|안전 여부|
|---|---|
|정적 파일 서빙|안전|
|동적 쿼리 후 JSON 응답|**안전** (상태 변경 없으면)|
|GET으로 DB insert|**안전하지 않음**|
|GET으로 외부 API 호출(주문 등)|**안전하지 않음**|

"정적 파일이어야 안전하다"는 말은 틀렸고,  
**"상태를 바꾸지 않으면 안전하다"** 가 올바른 정의다.  
그 문장은 이 오해를 교정하는 것이다.