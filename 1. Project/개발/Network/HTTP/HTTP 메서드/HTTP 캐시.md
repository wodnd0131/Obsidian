#Network/HTTP 
## 1. Problem First — 캐시가 없을 때 어떤 일이 생기나

### 시나리오: 변하지 않는 데이터를 매번 새로 만든다

```
사용자 1000명이 메인 페이지를 동시에 요청한다.
메인 페이지에는 오늘의 추천 상품 목록이 있다.
이 목록은 하루에 한 번 바뀐다.
```

```
요청 1   → DB 쿼리 → 상품 목록 조회 → JSON 직렬화 → 응답
요청 2   → DB 쿼리 → 상품 목록 조회 → JSON 직렬화 → 응답
요청 3   → DB 쿼리 → 상품 목록 조회 → JSON 직렬화 → 응답
...
요청 1000 → DB 쿼리 → 상품 목록 조회 → JSON 직렬화 → 응답
```

1000번 전부 동일한 결과다. 그런데 1000번 전부 DB에 갔다.

**이게 근본적으로 문제인 이유:**  
"이미 만든 답을 버리고 또 만드는" 낭비가 아니라,  
트래픽 급증 시 DB가 병목이 되어 **전체 시스템이 응답 불능**이 된다.  
캐시는 성능 최적화가 아니라 **시스템 생존 전략**이다.

### 또 다른 시나리오: 네트워크 비용

```
클라이언트(모바일)
    |
    | 매 요청마다 500KB짜리 이미지를 새로 받는다
    | 이미지는 바뀌지 않았는데
    |
    서버
```

바뀌지 않은 리소스를 매번 전송하는 것은  
사용자 데이터 낭비 + 서버 대역폭 낭비 + 응답 지연이다.

---

## 2. HTTP 캐시의 두 가지 질문

HTTP 캐시는 항상 두 가지를 판단한다.

```
1. 이 응답을 저장해도 되는가?         → 캐시 가능성 (Cacheability)
2. 저장한 응답이 아직 유효한가?       → 신선도 (Freshness)
```

이 두 질문에 답하는 메커니즘이 `Cache-Control`과 `ETag`다.

---

## 3. Mechanics

### 3-1. Cache-Control — "얼마나 오래 믿어도 되는가"

서버가 응답에 붙이는 헤더다.

```
HTTP/1.1 200 OK
Cache-Control: max-age=3600
Content-Type: application/json

{ "products": [...] }
```

`max-age=3600` → "이 응답은 3600초(1시간) 동안 신선하다. 그 안에는 서버에 묻지 말고 저장해둔 걸 써라."

```
클라이언트                    서버
    |                                     |
    |--- GET /products ------->|
    |<-- 200 OK (max-age=3600)-|
    |    [응답 저장]                       |
    |                                                |
    | (30분 후)                            |
    |--- GET /products         | ← 서버에 요청 안 보냄
    |    [저장된 응답 사용]     |
    |                                     |
    | (61분 후)                            |
    |--- GET /products ------->| ← 만료됐으므로 서버에 요청
    |<-- 200 OK (max-age=3600)-|
```

### 주요 디렉티브

```
Cache-Control: max-age=3600
```

3600초 동안 캐시 유효.

```
Cache-Control: no-cache
```

이름이 헷갈린다. "캐시하지 마라"가 아니다.  
"저장은 해도 되지만, **쓰기 전에 서버에 유효한지 물어봐라**"다.

```
Cache-Control: no-store
```

"저장 자체를 하지 마라." 진짜로 캐시 안 함.  
민감한 데이터(인증 토큰, 개인정보)에 쓴다.

```
Cache-Control: private
```

"중간 프록시/CDN에는 저장하지 말고, 이 클라이언트(브라우저)에만 저장해라."  
로그인한 사용자 개인화 응답에 쓴다.

```
Cache-Control: public
```

"CDN 포함 어디든 저장해도 된다."

```
Cache-Control: s-maxage=3600
```

CDN 같은 공유 캐시에만 적용되는 max-age.  
브라우저 캐시와 CDN 캐시를 다르게 설정할 때 쓴다.

### 3-2. ETag — "내용이 바뀌었는가"

앞에서 충돌 방지 용도로 봤던 ETag가 캐시에서도 쓰인다.  
용도가 다르고 사용하는 헤더가 다르다.

|용도|요청 헤더|
|---|---|
|충돌 방지 (PUT)|`If-Match`|
|캐시 갱신 확인 (GET)|`If-None-Match`|

```
1차 요청:
GET /products
← 200 OK
← ETag: "abc123"
← Cache-Control: no-cache   ← 매번 확인하라
← { "products": [...] }
   [브라우저: ETag "abc123" 와 함께 응답 저장]

2차 요청 (캐시가 있지만 no-cache라 확인해야 함):
GET /products
→ If-None-Match: "abc123"   ← "이 버전 아직 유효해?"

서버가 확인:
  현재 ETag == "abc123" → 안 바뀜
← 304 Not Modified          ← body 없음, 헤더만
   [브라우저: 저장해둔 응답 그대로 사용]
```

304 응답은 body가 없다.  
500KB짜리 응답을 다시 받는 대신 수백 바이트짜리 304만 받는다.

```
서버에서 상품 목록이 바뀌었다면:

GET /products
→ If-None-Match: "abc123"

서버: 현재 ETag == "xyz789" → 바뀜
← 200 OK
← ETag: "xyz789"
← { "products": [...새 목록...] }
```

### 3-3. 캐시가 저장되는 위치

```
클라이언트(브라우저)  →  CDN/프록시  →  Origin 서버
     [브라우저 캐시]     [공유 캐시]
```

**브라우저 캐시 (Private Cache)**  
이 사용자의 브라우저에만 저장된다.  
`private` 디렉티브가 적용되는 곳.

**CDN / 리버스 프록시 (Shared Cache)**  
모든 사용자의 요청이 여기서 처리될 수 있다.  
CloudFront, Nginx, Varnish 등.  
`public` 디렉티브가 적용되는 곳.

```
사용자 1000명이 /products 요청
    ↓
CDN이 캐시 가지고 있으면 → 1000명 전원 CDN에서 응답
Origin 서버에는 요청 0개
```

이게 CDN의 핵심 가치다.

---

## 4. Spring에서 구현

### Cache-Control 응답 헤더 설정

```java
@GetMapping("/products")
public ResponseEntity<List<Product>> getProducts() {
    List<Product> products = productService.getAll();
    
    return ResponseEntity.ok()
        .cacheControl(CacheControl.maxAge(1, TimeUnit.HOURS).cachePublic())
        .body(products);
}
```

### ETag + If-None-Match 처리

Spring은 `ShallowEtagHeaderFilter`로 자동화할 수 있다.

```java
@Bean
public FilterRegistrationBean<ShallowEtagHeaderFilter> etagFilter() {
    FilterRegistrationBean<ShallowEtagHeaderFilter> bean = 
        new FilterRegistrationBean<>(new ShallowEtagHeaderFilter());
    bean.addUrlPatterns("/api/*");
    return bean;
}
```

이걸 등록하면:

- 서버가 응답 body의 MD5 해시를 계산해서 ETag로 붙인다
- 요청에 `If-None-Match`가 있으면 비교해서 304 자동 반환

**단점:** body를 다 만들고 나서 해시를 계산한다.  
DB 쿼리는 매번 일어난다. 네트워크 전송만 줄어드는 것이다.  
진짜 DB 부하를 줄이려면 `Cache-Control: max-age`와 조합하거나  
직접 버전 컬럼 기반 ETag를 구현해야 한다.

```java
// 버전 컬럼 기반 ETag — DB 쿼리도 줄이는 방식
@GetMapping("/products/{id}")
public ResponseEntity<Product> getProduct(
    @PathVariable Long id,
    @RequestHeader(value = "If-None-Match", required = false) String ifNoneMatch
) {
    Product product = productRepository.findById(id).orElseThrow();
    String etag = "\"" + product.getVersion() + "\"";
    
    if (etag.equals(ifNoneMatch)) {
        return ResponseEntity.status(304).eTag(etag).build(); // body 없이
    }
    
    return ResponseEntity.ok()
        .eTag(etag)
        .cacheControl(CacheControl.noCache()) // 매번 확인, 단 304 가능
        .body(product);
}
```

---

## 5. 설계 트레이드오프 — 무엇을 잃는가

### max-age를 길게 잡으면

```
Cache-Control: max-age=86400  (24시간)
```

서버 부하 감소, 응답 빠름.

그런데 배포를 했다.

```
서버: 상품 가격이 바뀐 새 버전 배포 완료
클라이언트: 아직 캐시에 있는 이전 가격을 24시간 동안 본다
```

**캐시 무효화(Cache Invalidation)** 문제다.  
캐시 기간 동안 서버가 응답을 "취소"할 방법이 HTTP 표준에는 없다.

> Phil Karlton의 유명한 말: "There are only two hard things in Computer Science: cache invalidation and naming things."
> 
> _→ 컴퓨터 과학에서 어려운 것은 딱 두 가지다: 캐시 무효화와 이름 짓기._

### 실무에서 쓰는 해결법: URL에 버전을 박는다

```html
<!-- 나쁜 예: 파일명이 같으면 캐시가 안 깨짐 -->
<script src="/app.js"></script>

<!-- 좋은 예: 배포마다 파일명이 바뀜 → 자동으로 새로 받음 -->
<script src="/app.abc123.js"></script>
```

Webpack, Vite 같은 번들러가 자동으로 해준다.  
파일 내용의 해시를 파일명에 넣는다.  
내용이 바뀌면 파일명이 바뀌고, 브라우저는 새 파일로 인식해서 캐시를 안 쓴다.

API 응답은 URL에 버전을 박기 어렵다. 그래서:

```
정적 파일 (JS, CSS, 이미지): max-age 길게 + 파일명 해시
API 응답 (자주 바뀌는 데이터): no-cache + ETag 조합
API 응답 (안 바뀌는 데이터): max-age 짧게 + CDN
```

---

## 6. 정리 — 헤더 조합별 동작

|설정|동작|쓰는 상황|
|---|---|---|
|`max-age=3600`|1시간 동안 서버 요청 없음|자주 안 바뀌는 공개 데이터|
|`no-cache`|매번 서버 확인, 304 가능|바뀔 수 있지만 304 최적화 원할 때|
|`no-store`|아예 저장 안 함|민감한 데이터|
|`private, max-age=3600`|브라우저만 캐시, CDN은 안 함|로그인 사용자 개인화 응답|
|`public, max-age=86400`|CDN 포함 전부 캐시|정적 리소스|
|`no-cache` + ETag|매번 확인하되 안 바뀌면 304|API 응답 최적화|

---

## 7. 이어지는 개념

1. **CDN 동작 원리 (CloudFront)** — 브라우저 캐시의 다음 레이어다. `Cache-Control: public`을 붙였을 때 실제로 CDN에서 어떻게 캐시되고 무효화되는지. 지금 배운 헤더들이 CDN에서 어떻게 해석되는지 직접 이어진다.
    
2. **Spring Cache 추상화 (`@Cacheable`)** — HTTP 캐시는 클라이언트-서버 사이의 캐시다. `@Cacheable`은 서버 내부에서 DB 부하를 줄이는 캐시다. 레이어가 다르고 무효화 전략도 다르다. HTTP 캐시만으로 해결 안 되는 부분을 어디서 채우는지 이어진다.
    
3. **캐시 스탬피드 (Cache Stampede) / 썬더링 허드** — 캐시가 만료되는 순간 동시에 수천 개의 요청이 DB로 쏟아지는 문제. 캐시를 배웠으면 반드시 알아야 하는 장애 패턴이다.