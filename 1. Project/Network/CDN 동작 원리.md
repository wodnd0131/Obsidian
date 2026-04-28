# CDN 동작 원리 (CloudFront)

---

## 1. Problem First — CDN이 없을 때 어떤 일이 생기나

### 시나리오 A: 지리적 거리

```
서울 사용자 → 요청 → 미국 버지니아 Origin 서버
                      (왕복 약 200~300ms RTT)

도쿄 사용자 → 요청 → 미국 버지니아 Origin 서버
                      (왕복 약 150~200ms RTT)
```

빛의 속도는 물리 법칙이다. 거리를 줄이지 않으면 해결 안 된다.

### 시나리오 B: 트래픽 집중

```
이벤트 시작 → 동시 접속자 10만 명 → 전부 Origin 서버로
                                      → 서버 다운
```

앞서 HTTP 캐시에서 본 것처럼, 캐시가 없으면 Origin이 모든 부하를 받는다.  
브라우저 캐시는 **각 사용자의 브라우저에만** 저장된다.  
사용자 10만 명이 처음 요청할 때는 전부 Origin으로 간다.

**CDN이 해결하는 것:**  
사용자 가까운 곳에 응답을 미리 저장해두고,  
Origin 대신 CDN이 응답한다.

```
서울 사용자 → CDN 서울 엣지 → (캐시 있으면) 즉시 응답
                              → (캐시 없으면) Origin에서 가져와서 저장 후 응답
```

---

## 2. Mechanics — CDN 내부에서 어떻게 동작하는가

### 2-1. 엣지 로케이션(Edge Location)

CloudFront는 전 세계에 **엣지 로케이션**을 두고 있다.  
사용자의 요청은 DNS 레벨에서 가장 가까운 엣지로 라우팅된다.

```
사용자가 cdn.example.com 요청
    ↓
DNS: "이 사용자에게 가장 가까운 CloudFront 엣지 IP를 알려줘"
    ↓
서울 엣지로 연결
```

이게 **Anycast 라우팅**이다.  
같은 도메인이지만 사용자 위치에 따라 다른 서버로 연결된다.

### 2-2. 캐시 히트 / 미스 흐름

```
┌─────────────────────────────────────────────────┐
│                  요청 흐름                        │
│                                                   │
│  사용자 ──→ 엣지 로케이션                         │
│                   │                               │
│            캐시 있음? ──Yes──→ 응답 (Cache HIT)   │
│                   │                               │
│                  No                               │
│                   ↓                               │
│            Origin 서버에 요청                     │
│                   │                               │
│            응답 받아서 저장 (Cache MISS)           │
│                   │                               │
│            사용자에게 응답                         │
└─────────────────────────────────────────────────┘
```

**Cache HIT:** 엣지에서 바로 응답. Origin 요청 없음.  
**Cache MISS:** Origin까지 갔다옴. 이후 요청은 HIT.

### 2-3. 캐시 키(Cache Key) — 무엇을 기준으로 저장하나

CloudFront는 기본적으로 **URL(Host + Path + Query String)** 을 캐시 키로 쓴다.

```
GET /products?category=shoes  → 캐시 키: /products?category=shoes
GET /products?category=bags   → 캐시 키: /products?category=bags  (다른 캐시)
```

**중요한 함정:** 헤더나 쿠키가 다르면 같은 URL이어도 다른 응답일 수 있다.

```java
// 서버 코드
@GetMapping("/products")
public List<Product> getProducts(
    @RequestHeader("Accept-Language") String lang  // 언어별로 다른 응답
) {
    return productService.getByLanguage(lang);
}
```

```
GET /products (Accept-Language: ko)  → 한국어 응답
GET /products (Accept-Language: en)  → 영어 응답
```

CloudFront 기본 설정에서 헤더는 캐시 키에 포함되지 않는다.  
→ 한국어 응답이 캐시된 상태에서 영어 사용자가 오면 한국어 응답을 받는다.

이걸 해결하려면 CloudFront에서 캐시 키에 헤더를 포함하도록 설정해야 한다.

```
CloudFront Cache Policy 설정:
  Headers: Accept-Language 포함
```

그러면:

```
GET /products (Accept-Language: ko)  → 캐시 키: /products + ko
GET /products (Accept-Language: en)  → 캐시 키: /products + en  (다른 캐시)
```

### 2-4. TTL — CloudFront가 Cache-Control을 어떻게 해석하는가

여기서 앞에서 배운 `Cache-Control`이 직접 연결된다.

CloudFront는 Origin 응답의 `Cache-Control` 헤더를 읽어서 TTL을 결정한다.

```
Origin 응답:
Cache-Control: max-age=3600, public

→ CloudFront: "3600초 동안 이 응답을 엣지에 저장하겠다"
```

|Origin 응답 헤더|CloudFront 동작|
|---|---|
|`Cache-Control: max-age=3600`|3600초 캐시|
|`Cache-Control: no-cache`|기본 TTL 적용 (설정값, 보통 86400초)|
|`Cache-Control: no-store`|캐시 안 함|
|`Cache-Control: private`|캐시 안 함|
|`s-maxage=7200, max-age=3600`|CloudFront는 7200초, 브라우저는 3600초|

**`s-maxage`가 중요한 이유:**  
브라우저 캐시와 CDN 캐시를 다르게 설정할 수 있다.

```
Cache-Control: s-maxage=86400, max-age=300
```

```
브라우저: 5분(300초) 후 만료 → 자주 최신 확인
CDN: 24시간(86400초) 유지 → Origin 부하 최소화
```

사용자는 5분 주기로 CDN에 확인 요청을 보내지만,  
CDN은 Origin에는 하루에 한 번만 간다.

---

## 3. 캐시 무효화 — 배포했을 때 어떻게 캐시를 깨는가

HTTP 캐시에서 짚었던 핵심 문제다.

```
max-age=86400으로 캐시된 상태에서 긴급 배포 → 사용자들이 24시간 동안 구버전을 본다
```

### 방법 1: CloudFront Invalidation API

```bash
aws cloudfront create-invalidation \
  --distribution-id EDFDVBD6EXAMPLE \
  --paths "/products/*" "/index.html"
```

CloudFront에 "이 경로의 캐시를 지워라"고 명령한다.  
엣지 로케이션 전체에 전파되는 데 보통 수십 초~수 분 걸린다.

**잃는 것:**

- 요청당 비용 발생 (AWS 과금)
- 전파 시간 동안 엣지마다 캐시 상태가 다를 수 있음
- 무효화 직후 모든 엣지에서 Cache MISS → Origin 트래픽 급증 가능

### 방법 2: URL에 버전 박기 (권장)

```
배포 전: /app.js           → CloudFront에 캐시됨
배포 후: /app.abc123.js    → 새 URL → 자동으로 Cache MISS → 새 파일 받음
```

URL이 바뀌면 캐시 키가 바뀐다.  
무효화 API 호출 없이 자동으로 새 버전이 서빙된다.

```
# Webpack 빌드 결과
app.3f4a1b.js    ← 내용 해시가 파일명에 포함
vendor.8c2d3e.js
```

HTML은 `no-cache` + ETag로 설정해서 항상 최신 파일명을 참조하게 한다.

```
index.html        → Cache-Control: no-cache (항상 확인)
app.3f4a1b.js    → Cache-Control: max-age=31536000 (1년, 파일명이 버전)
```

### 방법 3: `stale-while-revalidate`

```
Cache-Control: max-age=300, stale-while-revalidate=86400
```

> RFC 5861에 정의된 확장 디렉티브.

```
0~300초:   캐시 신선함 → 캐시 사용
300~86400초: 캐시 만료됐지만 백그라운드에서 갱신 중
             → 일단 만료된 캐시로 응답 (사용자는 기다리지 않음)
             → 동시에 Origin에 새 버전 요청
86400초 후: 완전 만료 → 반드시 Origin에서 받아야 함
```

사용자는 만료된 캐시를 잠깐 볼 수 있지만 응답 지연이 없다.  
엄격한 최신성보다 응답 속도가 중요한 경우에 쓴다.

---

## 4. Origin Shield — CloudFront 심화

```
엣지 로케이션이 전 세계에 수백 개 있다.
각 엣지가 Cache MISS일 때 전부 Origin으로 요청하면?
→ 동시에 수백 개의 엣지가 Origin을 두드린다
```

**Origin Shield**는 엣지와 Origin 사이에 중간 캐시 레이어를 추가한다.

```
[서울 엣지] ─┐
[도쿄 엣지] ─┤→ [Origin Shield (서울 리전)] → Origin 서버
[싱가포르엣지]─┘
```

모든 엣지의 MISS가 Origin Shield를 거친다.  
Origin Shield에 캐시가 있으면 Origin까지 가지 않는다.  
Origin이 받는 요청 수가 극적으로 줄어든다.

---

## 5. 설계 트레이드오프

### CDN을 쓰면 잃는 것

**디버깅 난이도 상승**

```
"왜 내 배포가 반영이 안 되지?"
→ CDN 캐시가 아직 살아있음
→ 엣지마다 캐시 상태가 다를 수 있음
→ 사용자마다 다른 버전을 볼 수 있음
```

**캐시 히트율과 최신성은 반비례**

```
TTL 길게 → 히트율 높음, Origin 부하 낮음, 최신성 낮음
TTL 짧게 → 히트율 낮음, Origin 부하 높음, 최신성 높음
```

은탄환이 없다. 데이터의 특성에 따라 TTL을 다르게 설정해야 한다.

```
정적 파일 (JS, CSS): TTL 1년 + URL 버전 해시
API 응답 (상품 목록): TTL 5분
API 응답 (재고 수량): TTL 30초 또는 캐시 안 함
API 응답 (개인화 데이터): no-store (CDN 캐시 안 함)
```

**지역별 법규 문제**

개인정보 데이터가 특정 국가 외 엣지에 캐시되면 법적 문제가 될 수 있다.  
CloudFront는 특정 지역 엣지만 허용하거나 제외하는 설정을 제공한다.

---

## 6. 이어지는 개념

1. **Spring Cache 추상화 (`@Cacheable`, `@CacheEvict`)** — HTTP/CDN 캐시는 클라이언트-네트워크 레이어 캐시다. `@Cacheable`은 서버 내부, DB 앞단의 캐시다. CDN을 뚫고 들어온 요청이 Origin 서버에 도달했을 때 이 레이어가 DB 부하를 막는다. 캐시 레이어 전체 그림을 완성하는 다음 단계다.
    
2. **캐시 스탬피드 (Cache Stampede)** — CDN TTL이 만료되는 순간, 수천 개의 엣지에서 동시에 Origin으로 요청이 몰린다. CDN을 배웠으면 이 장애 패턴이 어디서 어떻게 터지는지 알아야 한다. `@Cacheable`과 묶어서 봐야 완성된다.