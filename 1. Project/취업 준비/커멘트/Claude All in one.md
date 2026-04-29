### 인적성 / 인성 관련

**1분 자기소개 / 마지막 어필**

- `이력서` → "문제를 생산성으로 전환하는 백엔드 개발자" 한 줄 정체성
- `Pickeat` → DoS 시뮬레이션 → 메트릭 기반 분석 → Thread/Pool 튜닝 흐름
- `Matilda` → Redis 대신 인메모리 선택 이유를 설명할 수 있는 기술 의사결정 사례
- `이력서` → 우아한테크코스 7기, 공간정보 공모전 우수상, 창업유망팀 300 등 수상 이력

---

**지원 동기**

- `키위즈` → 첫 프로젝트 시작 계기: 외국인 대학생 소통 문제를 직접 발견하고 창업까지 연결
- `Pickeat` → 우아한테크코스 팀 프로젝트로 실서비스 운영 경험까지 이어짐
- `이력서` → "관찰 가능성과 메트릭" 지식 공유 세션 발표: 학습을 팀으로 확장하는 태도

---

**협업할 때 어떤 역할인지 / 팀 갈등 경험**

- `보험사 전사 시스템 개편` → 회의 비효율 문제 → "고객 가치" 기준 의사결정 체계 수립, 1:1 미팅 정례화
- `키위즈` → 개발팀 ↔ 기획팀 간극 → 작업 우선순위 3단계(필수/가능/여유) 체계화
- `키위즈` → 인프라 담당 공백 → 본인이 자발적으로 AWS 마이그레이션 인수

---

**기술 스택이 다른 팀원과 어떻게 할 것인지**

- `리뷰캔버스` → Next.js 기반 프론트 개발, 현업 개발자와 협업 → GitHub 레포 분석, Slack 대화 분석으로 적응
- `키위즈` → 백엔드 개발자임에도 React Native 프론트 일부 담당
- `Pickeat` → API 버저닝(v1/v2 병렬 제공)으로 프론트엔드와의 의존성 해소

---

**코드 리뷰 방식**

- `리뷰캔버스` → 현업 코드 리뷰 문화 적응, 커밋 메시지 작성법 연구, 피드백 기반 품질 개선
- `Matilda` → SonarCloud 도입 → 34개 코드 스멜 중 21개 해결, 정기 리팩터링 데이 운영
- `이력서` → 우아한테크코스: 페어 프로그래밍, 코드 리뷰 협업 중심 학습

---

### 기술 CS 관련

**MVC 패턴 / DispatcherServlet / Spring MVC**

- `보험사 전사 시스템 개편` → Spring Security + Main Server 구조 설계 담당
- `키위즈` → Spring Boot API 서버 구축, Spring Security 토큰 기반 인증/인가
- `Pickeat` → ApiSpec Interface 도입: 컨트롤러에서 명세 로직 분리 → MVC 관심사 분리 실사례

---

**DI / IoC / AOP**

- `Matilda` → `@TransactionalEventListener` 비동기 처리 도입: Spring 이벤트 기반 설계 활용
- `Pickeat` → 인터셉터를 활용한 구버전 API 만료 안내: AOP성 횡단 관심사 처리 사례
- `보험사 전사 시스템 개편` → Spring Security 필터 체인 구성

---

**JWT / 인증·인가**

- `키위즈` → Spring Security + Redis를 서브 DB로 활용한 토큰 관리
- `보험사 전사 시스템 개편` → Security 처리 + Redis Token 저장 구조 설계 담당
- `이력서` → "인증과 인가 관련" 자소서 항목 존재

---

**JPA / N+1 문제**

- `Matilda` → `/api/tag/recent`에서 N+1 발생 → `LEFT JOIN FETCH` 페치 조인으로 해결
- `Matilda` → Offset 기반 페이지네이션 → 커서 기반으로 전환, 복합 인덱스 추가

---

**JPQL vs QueryDSL**

- 직접 언급된 포트폴리오 없음 → "이 부분은 포트폴리오에 명시된 경험 없음, 별도 학습 내용으로 답변 준비 필요"

---

**DB 트랜잭션 / 인덱스**

- `Matilda` → `(is_public, is_deleted, createAt DESC)` 복합 인덱스 추가로 조회 성능 개선
- `Pickeat` → Flyway로 DB 형상 V1→V2→V3 체계화, `flyway_schema_history`로 실행 이력 관리
- `Pickeat` → 무중단 마이그레이션: 구/신규 스키마 병행 운영 → 트랜잭션 안전성 확보 사례

---

**Docker / 배포 방식**

- `Pickeat` → Docker 이미지 리태깅 기반 자동 롤백 (Dev: latest/previous, Prod: release/pre-release)
- `Pickeat` → DB 마이그레이션 전 타임스탬프 기반 자동 백업
- `Matilda` → Docker + Nginx + OCI 기반 배포 구성
- `키위즈` → EC2 ubuntu 기반 배포, Docker Chat Server 구성
- `이력서` → "배포 방식 관련" 자소서 항목 존재

---

**성능 최적화 / HikariCP / Thread 튜닝**

- `Pickeat` → Tomcat Thread 200 → 32로 조정: p99 응답시간 63% 개선
- `Pickeat` → HikariCP Pool 확장 시 오히려 악화 확인 → PreparedStatement Cache 활성화로 해결
- `Pickeat` → HikariCP 공식 공식(`connections = core_count × 2 + effective_spindle_count`) 적용 검토 언급

---

**TDD / 테스트**

- `Pickeat` → JUnit 5 사용
- `이력서` → 우아한테크코스: OOP, TDD, Spring 기반 미션 수행
- 구체적 TDD 흐름 사례는 포트폴리오에 명시 없음 → 별도 준비 필요

---

**Java Stream API**

- 포트폴리오에 직접 언급 없음 → 별도 준비 필요
---

**REST API / HTTP 메서드**

- `Pickeat` → API 버저닝 전략(v1/v2), ApiSpec Interface로 Swagger 명세 표준화
- `Matilda` → 목록 조회 API 분리 설계: Reference 정보를 별도 API로 분리

---

### 요약: 답변 밀도가 높은 질문 TOP 5

|질문|연결 가능한 포트폴리오 수|
|---|---|
|팀 갈등 경험과 해결 방식|보험사, 키위즈, 리뷰캔버스|
|DB 트랜잭션 / 인덱스|Pickeat, Matilda|
|배포 방식|Pickeat, Matilda, 키위즈|
|성능 최적화 경험|Pickeat, Matilda|
|인증/인가|키위즈, 보험사|
