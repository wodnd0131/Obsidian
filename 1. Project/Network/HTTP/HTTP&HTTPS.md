#Network/HTTP 

**HTTP는 HTML 문서와 같은 리소스들을 가져올 수 있도록 해주는** [프로토콜](https://developer.mozilla.org/ko/docs/Glossary/Protocol)입니다. HTTP는 웹에서 이루어지는 모든 데이터 교환의 기초이며, 클라이언트-서버 프로토콜이기도 합니다. 클라이언트-서버 프로토콜이란 (보통 웹브라우저인) 수신자 측에 의해 요청이 초기화되는 프로토콜을 의미합니다. 하나의 완전한 문서는 텍스트, 레이아웃 설명, 이미지, 비디오, 스크립트 등 불러온(fetched) 하위 문서들로 재구성됩니다.
![[../../../Repository/HTTP&HTTPS-1.png]]

클라이언트와 서버들은 (데이터 스트림과 대조적으로) 개별적인 메시지 교환에 의해 통신합니다. 보통 브라우저인 클라이언트에 의해 전송되는 메시지를 요청(_requests_)이라고 부르며, 그에 대해 서버에서 응답으로 전송되는 메시지를 응답(_responses_)이라고 부릅니다.

|**구분**|**HTTP 메시지 교환 (Message-based)**|**데이터 스트림 (Stream-based)**|
|---|---|---|
|**통신 단위**|요청 하나에 응답 하나 (독립적 단위)|연속적인 데이터 조각들의 흐름|
|**구조**|헤더(Header)와 바디(Body)가 명확히 구분됨|데이터가 잘게 쪼개져 계속 흘러감|
|**비유**|**택배:** 상자를 보내고, 받는 사람이 확인|**수도꼭지:** 틀면 물이 계속 나오고, 잠그면 멈춤|
|**상태**|요청이 끝나면 연결이 종료되거나 대기함|연결이 유지된 채 데이터가 계속 오감|
![[../../../Repository/HTTP&HTTPS-2.png]]

1990년대 초에 설계된 HTTP는 거듭하여 진화해온 확장 가능한 프로토콜입니다. HTTP는 애플리케이션 계층의 프로토콜로, 신뢰 가능한 전송 프로토콜이라면 이론상으로는 무엇이든 사용할 수 있으나 [[TCP (전송 제어 프로토콜)]] 혹은 암호화된 TCP 연결인 [[TCP (전송 제어 프로토콜)#TLS|TLS]] 를 통해 전송됩니다. HTTP의 확장성 덕분에, 오늘날 하이퍼텍스트 문서 뿐만 아니라 이미지와 비디오 혹은 HTML 폼 결과와 같은 내용을 서버로 포스트(POST)하기 위해서도 사용됩니다. HTTP는 또한 필요할 때마다 웹 페이지를 갱신하기 위해 문서의 일부를 가져오는데 사용될 수도 있습니다.

## HTTP 기반 시스템의 구성요소
HTTP는 클라이언트-서버 프로토콜입니다. 요청은 하나의 개체, 사용자 에이전트(또는 그것을 대신하는 프록시)에 의해 전송됩니다. 대부분의 경우, 사용자 에이전트는 브라우저지만, 무엇이든 될 수 있습니다. 예를 들어, 검색 엔진 인덱스를 채워넣고 유지하기 위해 웹을 돌아다니는 로봇이 그러한 경우입니다.

각각의 개별적인 요청들은 서버로 보내지며, 서버는 요청을 처리하고 `response`라고 불리는 응답을 제공합니다. 이 요청과 응답 사이에는 여러 개체들이 있는데, 예를 들면 다양한 작업을 수행하는 게이트웨이 또는 캐시 역할을 하는 [[../프록시]] 등이 있습니다.
![[../../../Repository/HTTP&HTTPS-3.png]]
### 클라이언트: 사용자 에이전트
사용자 에이전트는 사용자를 대신하여 동작하는 모든 도구입니다. 이 역할은 주로 브라우저에 의해 수행됩니다; 엔지니어들과 자신들의 애플리케이션을 디버그하는 웹 개발자들이 사용하는 프로그램들은 예외입니다.

브라우저는 **항상** 요청을 보내는 개체입니다. 그것은 결코 서버가 될 수 없습니다(수년에 걸쳐 서버 초기화된 메시지를 시뮬레이션하기 위해 몇 가지 메커니즘이 추가되어 왔지만).

웹 페이지를 표시하기 위해, 브라우저는 페이지의 HTML 문서를 가져오기 위한 요청을 전송한 뒤, 파일을 구문 분석하여 실행해야 할 스크립트 그리고 페이지 내 포함된 하위 리소스들(보통 이미지와 비디오)을 잘 표시하기 위한 레이아웃 정보(CSS)에 대응하는 추가적인 요청들을 가져옵니다. 그런 뒤에 브라우저는 완전한 문서인 웹 페이지를 표시하기 위해 그런 리소스들을 혼합합니다. 브라우저에 의해 실행된 스크립트는 이후 단계에서 좀 더 많은 리소스들을 가져올 수 있으며 브라우저는 그에 따라 웹 페이지를 갱신하게 됩니다.

웹 페이지는 하이퍼텍스트 문서로, 표시된 텍스트의 일부는 사용자가 사용자 에이전트를 제어하고 웹을 돌아다닐 수 있도록 새로운 웹 페이지를 가져오기 위해 실행(보통 마우스 클릭에 의해)될 수 있는 링크임을 뜻합니다. 브라우저는 HTTP 요청 내에서 이런 지시 사항들을 변환하고 HTTP 응답을 해석하여 사용자에게 명확한 응답을 표시합니다.
### 웹 서버
통신 채널의 반대편에는 클라이언트에 의한 요청에 대한 문서를 _제공_하는 서버가 존재합니다. 서버는 사실 상 논리적으로 단일 기계입니다.이는 로드(로드 밸런싱) 혹은 그때 그때 다른 컴퓨터(캐시, DB 서버, e-커머스 서버 등과 같은)들의 정보를 얻고 완전하게 혹은 부분적으로 문서를 생성하는 소프트웨어의 복잡한 부분을 공유하는 서버들의 집합일 수도 있기 때문입니다.

서버는 반드시 단일 머신일 필요는 없지만, 여러 개의 서버를 동일한 머신 위에서 호스팅 할 수는 있습니다. HTTP/1.1과 [`Host`](https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Headers/Host) 헤더를 이용하여, 동일한 IP 주소를 공유할 수도 있습니다.
### [[../프록시]]
웹 브라우저와 서버 사이에서는 수많은 컴퓨터와 머신이 HTTP 메시지를 이어 받고 전달합니다. 여러 계층으로 이루어진 웹 스택 구조에서 이러한 컴퓨터/머신들은 대부분은 전송, 네트워크 혹은 물리 계층에서 동작하며, 성능에 상당히 큰 영향을 주지만 HTTP 계층에서는 이들이 어떻게 동작하는지 눈에 보이지 않습니다. 이러한 컴퓨터/머신 중에서도 애플리케이션 계층에서 동작하는 것들을 일반적으로 **프록시**라고 부릅니다. 프록시는 눈에 보이거나 그렇지 않을 수도 있으며(프록시를 통해 요청이 변경되거나 변경되지 않는 경우를 말함) 다양한 기능들을 수행할 수 있습니다:

- 캐싱 (캐시는 공개 또는 비공개가 될 수 있습니다 (예: 브라우저 캐시))
- 필터링 (바이러스 백신 스캔, 유해 컨텐츠 차단(자녀 보호) 기능)
- 로드 밸런싱 (여러 서버들이 서로 다른 요청을 처리하도록 허용)
- 인증 (다양한 리소스에 대한 접근 제어)
- 로깅 (이력 정보를 저장)
## HTTP (HyperText Transfer Protocol) 개요

### 1. HTTP의 주요 특징

- **간결함:** 사람이 읽을 수 있는 텍스트 기반 메시지로 설계되었습니다. HTTP/2에서 프레임 단위로 캡슐화되어 복잡해졌지만, 기본 메시지 구조는 여전히 간결함을 유지합니다.
    
- **확장성:** **HTTP 헤더**를 통해 기능을 쉽게 확장할 수 있습니다. 클라이언트와 서버 간의 약속만 있다면 새로운 기능을 언제든 추가할 수 있습니다.
    
- **[[무상태(Stateless)와 세션]]:** 기본적으로 요청 간의 상태를 저장하지 않습니다. 하지만 **HTTP 쿠키**를 사용하여 쇼핑 바구니와 같은 '상태가 있는 세션'을 구현할 수 있습니다.
    
- **연결성:** 신뢰할 수 있는 전송 프로토콜(주로 TCP)에 의존합니다. HTTP/1.0은 매 요청마다 연결을 새로 맺었으나, HTTP/1.1은 지속 연결(Persistent Connection)을 도입했고, HTTP/2는 다중화(Multiplexing)를 통해 효율을 높였습니다.
    

### 2. HTTP로 제어할 수 있는 것들

1. **캐시(Cache):** 서버가 문서의 캐시 여부와 유효 기간을 지시할 수 있습니다.
    
2. **Origin 제약 완화(CORS):** 보안을 위한 동일 출처 정책(Same-origin policy)을 헤더를 통해 제어하여 다른 도메인의 자원을 안전하게 가져올 수 있습니다.
    
3. **인증(Authentication):** 특정 사용자만 접근할 수 있도록 인증 정보를 주고받습니다.
    
4. **프록시와 터널링:** 실제 주소를 숨기거나 네트워크 장벽을 넘기 위해 프록시 서버를 거칠 수 있습니다.
    

---

### 3. HTTP 통신 흐름 (HTTP Flow)

클라이언트와 서버가 통신하는 단계는 다음과 같습니다.

1. **TCP 연결 설정:** 요청을 보내기 위해 연결을 엽니다. (기존 연결 재사용 가능)
    
2. **HTTP 요청 메시지 전송:** ```http
    
    GET / HTTP/1.1
    
    Host: developer.mozilla.org
    
    Accept-Language: fr
    
3. **HTTP 응답 메시지 수신:**
    
    HTTP
    
    ```
    HTTP/1.1 200 OK
    Date: Sat, 09 Oct 2010 14:28:02 GMT
    Server: Apache
    Last-Modified: Tue, 01 Dec 2009 20:18:22 GMT
    ETag: "51142bc1-7449-479b075b2891b"
    Accept-Ranges: bytes
    Content-Length: 29769
    Content-Type: text/html
    
    <!DOCTYPE html>
    <html>
      </html>
    ```
    
4. **연결 종료 또는 재사용:** 통신이 끝나면 연결을 닫거나 다음 요청을 위해 유지합니다.
    

---

### 4. 기술의 발전: 파이프라이닝에서 HTTP/2까지

- **HTTP 파이프라이닝:** 응답을 기다리지 않고 여러 요청을 연속해서 보내는 방식이었으나, 실제 네트워크 환경에서 구현의 어려움이 있었습니다.
    
- **HTTP/2 (Multiplexing):** 파이프라이닝의 단점을 극복하고, 단일 연결 상에서 여러 메시지를 동시에 전송할 수 있도록 개선되었습니다.
    
- **QUIC:** 구글에서 실험 중인 UDP 기반 프로토콜로, 더 빠르고 신뢰성 있는 전송을 목표로 합니다.


![[../../../Repository/HTTP&HTTPS-httpStructure.png]]

- HTTP [메서드](https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Methods), 보통 클라이언트가 수행하고자 하는 동작을 정의한 [`GET`](https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Methods/GET), [`POST`](https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Methods/POST) 같은 동사나 [`OPTIONS`](https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Methods/OPTIONS)나 [`HEAD`](https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Methods/HEAD)와 같은 명사입니다. 일반적으로, 클라이언트는 리소스를 가져오거나(`GET`을 사용하여) [HTML 폼](https://developer.mozilla.org/ko/docs/Learn_web_development/Extensions/Forms)의 데이터를 전송(`POST`를 사용하여)하려고 하지만, 다른 경우에는 다른 동작이 요구될 수도 있습니다.
- 가져오려는 리소스의 경로; 예를 들면 [프로토콜](https://developer.mozilla.org/ko/docs/Glossary/Protocol) (`http://`), [도메인](https://developer.mozilla.org/ko/docs/Glossary/Domain) (여기서는 `developer.mozilla.org`), 또는 TCP [포트](https://developer.mozilla.org/ko/docs/Glossary/Port) (여기서는 `80`)인 요소들을 제거한 리소스의 URL입니다.
- HTTP 프로토콜의 버전.
- 서버에 대한 추가 정보를 전달하는 선택적 [헤더들](https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Headers).
- `POST`와 같은 몇 가지 메서드를 위한, 전송된 리소스를 포함하는 응답의 본문과 유사한 본문.
## HTTP 기반 API

HTTP 기반으로 가장 일반적으로 사용된 API는 [user agent](https://developer.mozilla.org/ko/docs/Glossary/User_agent)와 서버간에 데이터를 교환하는데 사용될 수 있는 [`XMLHttpRequest`](https://developer.mozilla.org/ko/docs/Web/API/XMLHttpRequest) API 입니다. 최신 [`Fetch API`](https://developer.mozilla.org/ko/docs/Web/API/Fetch_API)는 보다 강력하고 유연한 기능을 제공합니다.

또 다른 API인 [서버-전송 이벤트](https://developer.mozilla.org/ko/docs/Web/API/Server-sent_events)는 서버가 전송 메커니즘으로 HTTP를 사용하여, 클라이언트로 이벤트를 보낼 수 있도록 하는 단방향 서비스입니다. 클라이언트는 [`EventSource`](https://developer.mozilla.org/en-US/docs/Web/API/EventSource) 인터페이스를 사용하여, 연결을 맺고 이벤트 핸들러를 설정합니다. 클라이언트 브라우저는 HTTP 스트림으로 도착한 메시지를 적절한 [`Event`](https://developer.mozilla.org/ko/docs/Web/API/Event) 객체로 자동 변환하여, 알려진 경우 해당 이벤트 [`type`](https://developer.mozilla.org/ko/docs/Web/API/Event/type "type")에 대해 등록된 이벤트 핸들러로 전달하거나 또는 특정 유형의 이벤트가 설정되지 않은 경우에는 [`onmessage`](https://developer.mozilla.org/ko/docs/Web/API/EventSource/message_event "onmessage") 이벤트 핸들러로 전달합니다.
## 결론
HTTP는 사용이 쉬운 확장 가능한 프로토콜입니다. 헤더를 쉽게 추가하는 능력을 지닌 클라이언트-서버 구조는 HTTP가 웹의 확장된 수용력과 함께 발전할 수 있게 합니다.

HTTP/2가 성능 향상을 위해 HTTP 메시지를 프레임 내로 임베드하여 약간의 복잡함을 더했을지라도, 애플리케이션의 관점에서 볼 때, 메시지의 기본적인 구조는 HTTP/1.0이 릴리즈된 이후와 동일합니다. 세션의 흐름은 여전히 단순하여, 간단한 [HTTP 메시지 모니터](https://firefox-source-docs.mozilla.org/devtools-user/network_monitor/index.html)를 이용한 조사와 디버그를 가능하게 해줍니다.