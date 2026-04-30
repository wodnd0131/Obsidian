**S3 + CloudFront로 분리하면**

- 정적 파일 변경이 서버 재배포 없이 S3 업로드로 끝남
- CloudFront CDN이 엣지에서 캐싱 → API 서버 부하 없음
- 비용도 S3/CloudFront가 EC2보다 훨씬 저렴

# 구현
## TilShareController 동작 방식

### 이 컨트롤러가 하는 일

OG 태그(Open Graph) 처리예요.

카카오톡/슬랙/디스코드에 링크를 공유하면 미리보기가 뜨잖아요. 그게 OG 태그예요. 근데 이 미리보기는 **봇이 서버에 직접 요청해서 HTML을 파싱**해서 만들어요.

```
사용자가 카톡에 링크 공유
→ 카카오 봇이 서버에 GET /share/{id} 요청
→ 서버가 OG 태그 포함된 HTML 반환
→ 카카오가 파싱해서 미리보기 생성
```

근데 프론트가 React/Vue 같은 SPA면 이게 안 돼요.

```
봇이 SPA에 요청
→ 빈 HTML + JS 번들만 받음
→ JS 실행 안 함 (봇은 JS 실행 안 해요)
→ OG 태그 파싱 못함 → 미리보기 없음
```

그래서 API 서버에서 봇 요청만 가로채서 OG 태그 담긴 HTML을 직접 반환하는 거예요.

---

### 코드 흐름

```
GET /share/{id}
       ↓
User-Agent 확인
       ↓
┌──────────────────────────────┐
│ 봇이면                        │
│ DB에서 TIL 조회               │
│ Model에 데이터 담기            │
│ → og-template.html 렌더링    │
└──────────────────────────────┘
       ↓
┌──────────────────────────────┐
│ 사람이면                      │
│ → 프론트 URL로 redirect       │
└──────────────────────────────┘
```

---

### 적용된 방식들

**@Controller + Model + Thymeleaf**

봇한테 반환하는 HTML을 서버에서 렌더링해요.

```java
model.addAttribute("til", response);
model.addAttribute("description", createDescription(response.content()));
return "share/og-template";  // templates/share/og-template.html
```

```html
<!-- og-template.html -->
<meta property="og:title" th:content="${til.title}"/>
<meta property="og:description" th:content="${description}"/>
<meta property="og:image" th:content="${imageUrl}"/>
```

**User-Agent 기반 분기**

```java
// 봇 패턴 목록
"kakaotalk", "slackbot", "discordbot", "linkedinbot"

// 소문자로 통일해서 비교
String lowerUserAgent = userAgent.toLowerCase();
return Arrays.stream(botPatterns)
        .anyMatch(lowerUserAgent::contains);
```

**redirect**

사람이 `/share/{id}`로 직접 접근하면 프론트로 보내요.

```java
return "redirect:" + frontendUrl + "/all-tils?tilId=" + id;
```

---

## SPA에서 OG 태그가 안 되는 이유

### 일반 웹사이트 vs SPA 응답 차이

**일반 웹사이트 (SSR)**

```html
<!-- 서버가 이미 완성된 HTML을 반환 -->
<html>
  <head>
    <meta property="og:title" content="TIL 제목"/>
    <meta property="og:description" content="내용..."/>
  </head>
  <body>실제 컨텐츠</body>
</html>
```

**SPA (React/Vue)**

```html
<!-- 서버가 반환하는 건 이게 전부 -->
<html>
  <head></head>
  <body>
    <div id="root"></div>
    <script src="bundle.js"></script>  <!-- 얘가 실행돼야 컨텐츠 생김 -->
  </body>
</html>
```

---

### 봇은 JS를 실행 안 해요

사람이 브라우저로 접근하면

```
빈 HTML 받음 → bundle.js 실행 → React가 DOM 그림 → 화면 보임
```

카카오/슬랙 봇이 접근하면

```
빈 HTML 받음 → 끝
               JS 실행 안 함
               → OG 태그 없음 → 미리보기 생성 불가
```

---

### 그래서 이 컨트롤러가 하는 것

봇한테만 완성된 HTML을 서버에서 만들어서 줘요.

```
카카오 봇 → GET /share/{id}
              ↓
           User-Agent에 "kakaotalk" 감지
              ↓
           DB에서 TIL 조회 → Thymeleaf로 OG 태그 포함 HTML 렌더링
              ↓
           완성된 HTML 반환 → 카카오가 파싱 → 미리보기 생성

사람      → GET /share/{id}
              ↓
           User-Agent에 봇 패턴 없음
              ↓
           프론트 URL로 redirect → SPA가 정상 렌더링
```

---
## OG 태그

### Open Graph Protocol

페이스북이 2010년에 만든 메타데이터 표준이에요. HTML `<head>`안에 넣는 태그예요.

```html
<head>
  <meta property="og:title" content="제목"/>
  <meta property="og:description" content="설명"/>
  <meta property="og:image" content="이미지 URL"/>
  <meta property="og:url" content="페이지 URL"/>
</head>
```

---

### 하는 일

링크를 공유할 때 미리보기를 만드는 데이터예요.

```
og:title       → 미리보기 제목
og:description → 미리보기 설명
og:image       → 미리보기 썸네일
og:url         → 원본 링크
```

카카오톡/슬랙/디스코드/트위터 전부 이 태그를 읽어서 미리보기를 만들어요.

---

### 실제로 보이는 것

카톡에 링크 보냈을 때 이렇게 뜨는 거요.

```
┌─────────────────────┐
│ 🖼 썸네일 이미지      │
│ 제목                 │
│ 설명 텍스트...        │
│ example.com         │
└─────────────────────┘
```

이게 전부 OG 태그에서 오는 거예요.

---
네 정확해요.

---

봇 입장에서 HTML은 그냥 텍스트 파일이에요.

```
받은 것: <div id="root"></div> + bundle.js
읽을 수 있는 것: <head> 안에 있는 태그들
실행 못하는 것: JS → 그 안에 실제 컨텐츠가 있음
```

그래서 OG 태그를 `<head>`에 **정적으로** 박아줘야 봇이 읽을 수 있어요.

```html
<!-- 봇이 읽을 수 있음 -->
<head>
  <meta property="og:title" content="오늘 배운 것"/>
</head>

<!-- 봇이 읽을 수 없음 -->
<body>
  <div id="root"></div>  ← JS 실행 후에야 내용이 생김
</body>
```

이 컨트롤러가 하는 게 정확히 그거예요. 봇이 요청하면 `<head>`에 OG 태그 꽉 채운 HTML을 서버에서 직접 만들어서 주는 거예요.