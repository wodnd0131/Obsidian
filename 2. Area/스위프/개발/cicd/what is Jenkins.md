젠킨스부터 보면, Java로 작성된 오픈소스 automation 서버야. 2004년 Hudson이라는 이름으로 시작했다가 2011년에 Jenkins로 포크된 꽤 오래된 도구고, GitHub Actions(2019년 출시)보다 훨씬 먼저 CI/CD 영역을 장악하고 있었던 물건이야. 핵심 구조는 Controller(예전엔 Master라고 불렀던) + Agent(예전엔 Slave) 모델인데, Controller가 빌드 트리거를 받아서 스케줄링하고 실제 빌드/테스트/배포 작업을 Agent에게 위임하는 구조야. 작은 규모면 Controller 혼자서 Agent 역할까지 다 하기도 해.

가장 큰 차이는 **호스팅 모델**이야. GitHub Actions는 `ubuntu-24.04`처럼 GitHub가 직접 운영하는 호스팅 러너를 기본 제공하고, 필요할 때만 self-hosted runner를 옵션으로 추가하는 구조잖아. 반면 Jenkins는 처음부터 "GitHub-hosted" 옵션이라는 게 존재하지 않아 — Jenkins 서버 자체를 어딘가에(보통 EC2 같은 본인 인프라에) 직접 올려야만 돌아가. 즉 Jenkins는 정의상 전체가 self-hosted야. 그래서 지금 쓰신 "이번엔 self-hosted runner 안 쓸 예정"이라는 표현은 GH Actions 맥락에서나 의미가 있고, Jenkins로 넘어가면 이 구분 자체가 사라져. Jenkins Controller가 이미 EC2에 떠 있으면, 거기서 deploy 스텝을 그냥 로컬 셸로 실행하면 되니까 별도의 "self-hosted runner 등록" 절차 자체가 필요 없어져.

그 외 차이를 짚어보면:

- **파이프라인 정의**: GH Actions는 YAML, Jenkins는 Groovy 기반 Jenkinsfile(Declarative 또는 Scripted). Groovy라서 반복문/조건문/함수를 자유롭게 짤 수 있어 복잡한 로직엔 유리하지만, YAML보다는 진입장벽이 좀 있음.
- **트리거 통합**: GH Actions의 `on: pull_request`, `on: push`는 네이티브라 깔끔함. Jenkins는 GitHub 저장소에 Webhook을 직접 등록해줘야 하고, PR 단위 빌드를 받으려면 GitHub Branch Source 플러그인 기반 Multibranch Pipeline job을 만들어야 함. 지금처럼 "PR 체크용"과 "push 시 배포용"을 나누는 패턴은 Jenkins에서는 보통 Job을 두 개로 분리해서(예: `swiipe-pr-check` Multibranch Pipeline, `swiipe-back-dev-deploy` 일반 Pipeline) 각각 다른 webhook 이벤트를 구독하는 식으로 구현해.
- **Secret 관리**: GH Actions Secrets vs Jenkins Credentials Store(UI에서 등록 후 Jenkinsfile에서 `withCredentials` 블록으로 사용).
- **PR에 체크 결과 표시**: GH Actions는 기본 내장. Jenkins는 GitHub Checks 플러그인이나 Commit Status 연동 플러그인을 추가로 깔아야 PR에 ✅/❌가 붙음.
- **UI**: GH Actions는 GitHub 안에 통합돼 있어 따로 볼 필요 없음. Jenkins는 별도 대시보드(보통 `:8080`)에 들어가야 하는데, 대신 빌드 트렌드 그래프, Blue Ocean UI 등 시각화는 더 풍부한 편.
- **운영 비용**: GH Actions는 호스팅 러너 쓰면 별도 서버 관리가 없음. Jenkins는 Controller 자체가 항시 떠있는 EC2 인스턴스라서 그 비용/관리 부담을 본인이 짊어져야 함.

위 두 워크플로우를 Jenkins로 옮기는 건 거의 1:1로 가능해. 단계별로 매핑하면:

- Submodule checkout: Jenkins Git 플러그인이 submodule(recursive 포함) checkout을 네이티브 지원하거나, 그냥 `sh 'git submodule update --init --recursive'`로 셸 처리해도 됨.
- JDK 21: Global Tool Configuration에서 자동 설치시키거나, EC2(Agent)에 미리 JDK 21을 박아두는 방식. 혹은 `agent { docker { image 'eclipse-temurin:21-jdk' } }`로 컨테이너 안에서 빌드하는 것도 흔한 패턴.
- `./gradlew test` / `./gradlew build -x test`: 그대로 `sh` 스텝.
- 테스트 결과 publish: `junit 'backend/build/test-results/**/*.xml'` 스텝이 내장돼 있어서 EnricoMi 액션과 같은 역할(Jenkins UI에 테스트 트렌드 그래프로 표시)을 해줌.
- Docker build & push: Docker Pipeline 플러그인의 `docker.build()` / `docker.withRegistry()`로 동일하게 가능하고, `linux/arm64` 멀티아키 빌드도 그냥 셸에서 `docker buildx build --platform linux/arm64 --push ...` 호출하면 됨.
- Blue-Green deploy: Jenkins Controller가 배포 대상 EC2와 같은 머신이면 그냥 같은 셸 스텝(`sh '/home/ubuntu/project/deploy.sh'`)으로 끝남. 다른 머신에 배포해야 하면 SSH Pipeline Steps 플러그인으로 ssh 접속해서 실행.

이미 Jenkins용으로 EC2 IAM 정책까지 설정해두신 거 보면, 보통 소규모 팀은 Controller 한 대(EC2 하나)에서 빌드부터 배포까지 다 처리하는 단일 노드 구성으로 시작하고, 나중에 빌드량이 늘면 Controller/Agent를 분리하는 식으로 가는 경우가 많아. 지금 스케일이면 단일 노드로 시작해도 충분할 거야.

---
# 젠킨스 우대사항의 의미?
그 질문이 진짜로 묻고 있는 건 "CI/CD가 뭔지 아세요?"가 아니라 거의 항상 "우리 회사에 이미 있는 Jenkins 인프라를 던져줘도 바로 들어가서 운영할 수 있나요?"야. 이 차이가 핵심이고, 몇 가지 구체적인 이유로 나눠볼 수 있어.

가장 큰 이유는 **레거시 인프라**야. Jenkins는 2011년부터 있었고 GH Actions는 2019년에 나왔어. 그 사이 10년 가까이 거의 모든 회사의 CI/CD가 Jenkins로 구축됐고, 이미 수십~수백 개 Job, Shared Library, 플러그인 조합이 얽혀있는 시스템을 새 도구로 마이그레이션할 동기가 별로 없어. 그래서 "Jenkins 써봤냐"는 사실상 "우리가 이미 깔아둔 그 거미줄 같은 시스템을 보고 당황하지 않을 사람인가"를 묻는 거야.

두 번째는 **운영 책임의 범위**가 완전히 다르다는 점. GH Actions는 YAML만 잘 쓰면 러너 관리, 스케일링, 가용성은 GitHub가 다 알아서 해줘. Jenkins는 Controller 자체가 본인이 띄우고 패치하고 백업하고 장애 대응까지 해야 하는 서버야. 메모리 부족으로 Controller가 죽거나, 플러그인 버전 충돌로 빌드가 깨지거나, Credential Store 권한 설정을 잘못해서 시크릿이 노출되는 류의 문제는 GH Actions에서는 거의 마주칠 일이 없는데 Jenkins에서는 일상이야. 그래서 면접관이 듣고 싶은 건 "Jenkinsfile 작성 경험"보다는 "Jenkins 자체를 운영하다가 터진 문제를 해결해본 경험"인 경우가 많아.

세 번째, **Groovy 기반 파이프라인과 Shared Library**. 회사 규모가 커지면 마이크로서비스 수십 개에 똑같은 Jenkinsfile을 복붙하지 않고, 공통 로직을 Shared Library로 뽑아서 여러 레포가 같은 파이프라인 함수를 호출하는 구조를 만들어. 이건 YAML로는 구현이 어렵고 Groovy의 스크립팅 능력이 필요한 영역이라, "Jenkins 경험"이 있다는 건 곧 이런 식의 파이프라인 추상화 설계 경험이 있다는 신호로 읽히기도 해.

네 번째는 한국 회사 맥락에서 특히 중요한데, **망분리/온프레미스 환경**. 금융권이나 일부 대기업, 공공기관 인접 회사는 보안 정책상 외부 클라우드 러너(GitHub-hosted)에 접근 자체가 막혀있어. 이런 환경에서는 GH Actions를 아예 못 쓰고 Jenkins처럼 내부망에 직접 떠 있는 self-hosted 도구만 쓸 수 있어. 그래서 "Jenkins 써봤냐"가 사실은 "폐쇄망 환경에서 CI/CD 돌려본 경험 있냐"를 묻는 질문으로 기능하는 경우도 꽤 많아.

정리하면, 면접 답변을 준비한다면 "Jenkinsfile 작성해봤다" 수준보다는 Agent 구성을 직접 해봤는지, Credential 관리나 플러그인 충돌 같은 운영 이슈를 다뤄봤는지, 또는 여러 Job/레포를 엮는 파이프라인 구조를 설계해봤는지를 보여주는 쪽이 회사가 실제로 듣고 싶어하는 답일 거야. 지금처럼 EC2에 Jenkins Controller를 직접 올리고 IAM부터 잡아가는 과정 자체가 이미 그 방향의 경험이고.