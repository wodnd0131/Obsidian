먼저 짚어야 할 게 하나 있어. 이전 GH Actions에서 `cd` job이 `runs-on: [self-hosted, dev]`로 별도의 self-hosted runner를 썼던 이유는, 배포 대상 EC2(dev 서버)에 직접 셸 명령(`docker-compose 복사`, `backup_db.sh`, `deploy.sh`)을 실행해야 했기 때문이야. 

Jenkins로 넘어가면 이 구분 자체가 의미가 없어져 — Jenkins는 애초에 전체가 self-hosted이기 때문에, "self-hosted runner를 안 쓴다"는 표현이 성립하려면 사실상 Jenkins Controller가 배포 대상 EC2와 같은 머신 위에서 동작해야 한다는 뜻이 돼. 

만약 Jenkins Controller를 별도 EC2에 띄울 생각이라면, 그 경우엔 SSH로 배포 대상에 접속해서 명령을 날리는 구조가 필요해지니까, 이 부분은 먼저 결정하고 가는 게 좋아. 

일단은 "Jenkins Controller = 배포 대상 EC2"인 단일 서버 구성을 기준으로 설명할게(소규모 팀 구조엔 이게 제일 단순함).

**1. Jenkins 설치**

EC2(Seoul, ap-northeast-2)에 Docker로 Jenkins를 띄우는 게 관리 측면에서 제일 깔끔해. 호스트의 Docker daemon을 그대로 써야 docker build/push가 가능하니까 소켓을 마운트해줘:

```bash
docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $(which docker):/usr/bin/docker \
  --group-add $(getent group docker | cut -d: -f3) \
  jenkins/jenkins:lts
```

초기 화면에서 admin password 입력하고, Plugin 설치 단계에서 기본 추천 플러그인 외에 추가로 필요한 것들: GitHub Branch Source(PR 자동 감지), Docker Pipeline(docker.build/push를 파이프라인 함수로), Pipeline: GitHub Groovy Libraries, JUnit, Credentials Binding. 이 정도면 GH Actions에서 쓴 액션들을 거의 다 대체할 수 있어.

**2. Credentials 등록 (Manage Jenkins → Credentials)**

GH Actions Secrets에 대응하는 항목들을 여기에 넣어줘: `SUBMODULE_KEY` → GitHub PAT(Username with password 타입), `DOCKER_USERNAME`/`DOCKER_PASSWORD` → Docker Hub 자격증명. Jenkinsfile에서는 `withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')])` 형태로 꺼내 쓰게 돼.

**3. Global Tool Configuration**

JDK 21(Temurin)을 Manage Jenkins → Tools에서 자동 설치로 등록하거나, EC2에 미리 깔아둔 경로를 잡아주면 돼. Gradle은 프로젝트가 wrapper(`./gradlew`)를 쓰고 있어서 별도 Tool 등록은 필요 없어.

**4. GitHub Webhook**

레포 Settings → Webhooks에서 Payload URL을 `http://<jenkins-host>:8080/github-webhook/`으로 등록(push, pull_request 이벤트 체크). 다만 사내망/방화벽 안에 Jenkins가 있으면 GitHub가 직접 호출을 못 하니, 이 경우 ngrok 같은 터널 또는 GitHub→Jenkins 단방향 폴링(`Poll SCM`)으로 우회해야 해.

**5. Job 1 — PR Check (Multibranch Pipeline)**

`back/dev`로 들어오는 PR을 감지하려면 Multibranch Pipeline을 만들고, Branch Source 설정에서 "Discover pull requests" behavior를 추가해줘. 저장소 안에 Jenkinsfile을 두면 되는데, 대략 이런 모양이야:

```groovy
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git submodule update --remote'
            }
        }
        stage('Test') {
            steps {
                dir('backend') { sh './gradlew test' }
            }
        }
        stage('Build') {
            steps {
                dir('backend') { sh './gradlew build -x test' }
            }
        }
    }
    post {
        always {
            junit 'backend/build/test-results/**/*.xml'
        }
    }
}
```

`EnricoMi/publish-unit-test-result-action`이 했던 일은 내장 `junit` 스텝이 그대로 대체해줘 — 빌드 결과에 테스트 트렌드 그래프가 자동으로 붙고, GitHub Branch Source 플러그인이 빌드 성공/실패를 PR에 Check로 자동 게시해줘서 따로 GitHub token 핸들링할 필요도 없어.

**6. Job 2 — Build & Deploy (일반 Pipeline)**

`back/dev` push 시 트리거되는 단일 Pipeline job. "GitHub hook trigger for GITScm polling" 옵션을 켜두면 push webhook이 들어올 때 자동 실행돼.

```groovy
pipeline {
    agent any
    environment {
        DOCKER_CREDS = credentials('dockerhub')
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git submodule update --remote'
                script {
                    env.SHORT_SHA = sh(script: "git rev-parse --short=7 HEAD", returnStdout: true).trim()
                }
            }
        }
        stage('Build JAR') {
            steps {
                dir('backend') { sh './gradlew clean build -x test --no-daemon --build-cache' }
            }
        }
        stage('Docker Build & Push') {
            steps {
                sh """
                  echo \$DOCKER_CREDS_PSW | docker login -u \$DOCKER_CREDS_USR --password-stdin
                  docker buildx build --platform linux/arm64 --push \
                    --build-arg SPRING_ACTIVE_PROFILE=dev \
                    -t \$DOCKER_CREDS_USR/swiipe-backend:latest \
                    -t \$DOCKER_CREDS_USR/swiipe-backend:${SHORT_SHA} \
                    ./backend
                """
            }
        }
        stage('Deploy') {
            steps {
                sh '''
                  sudo cp ./backend/docker/docker-compose.dev.yml /home/ubuntu/project/application/docker-compose.yml
                  [ -f /home/ubuntu/project/database_backup/backup_db.sh ] && sudo /home/ubuntu/project/database_backup/backup_db.sh || echo "skip backup"
                  sudo /home/ubuntu/project/deploy.sh
                '''
            }
        }
    }
}
```

여기서 주의할 점이 하나 있어. GH Actions에서는 `docker/setup-buildx-action`이 호출되는 순간 QEMU 에뮬레이션이 자동으로 설정돼서 x86 러너에서도 `linux/arm64` 빌드가 됐는데, Jenkins 컨테이너에서는 이걸 직접 세팅해줘야 해. Jenkins가 올라간 EC2 인스턴스 자체가 arm64(예: t4g, m7g 계열)라면 네이티브 빌드라 문제없지만, x86 인스턴스라면 호스트에서 한 번 `docker run --privileged --rm tonistiigi/binfmt --install arm64`를 실행하고 `docker buildx create --use --name multiarch`로 빌더를 만들어둬야 buildx 크로스 빌드가 동작해.

**전체적으로 매핑해보면**: GH Actions의 `secrets` → Jenkins Credentials, `actions/setup-java` → Global Tool Config, `EnricoMi/publish-unit-test-result-action` → `junit` 스텝, PR에 자동으로 체크 표시되는 것 → Multibranch Pipeline의 GitHub Branch Source가 대체, `runs-on: [self-hosted, dev]`로 분리했던 배포 환경 → Jenkins Controller가 배포 대상과 같은 머신에 있다면 그냥 로컬 셸, 다르다면 SSH Steps 플러그인으로 대체.

이미 EC2 IAM을 Jenkins용으로 잡아두신 상태면, 그 EC2가 배포 대상 서버와 같은 인스턴스인지 아니면 별도의 Jenkins 전용 인스턴스인지에 따라 Deploy 스테이지 구현이 달라지는데, 어느 쪽으로 가고 있는지 알려주면 그에 맞게 더 구체적으로 짜줄게.

---

docker-compose에서는 `$(...)` 같은 셸 명령 치환이 직접 안 되니까, docker socket 그룹 GID는 `.env` 파일로 분리해서 넘겨주는 게 깔끔해.

먼저 `.env` 파일 생성:

```bash
echo "DOCKER_GID=$(getent group docker | cut -d: -f3)" > .env
```

그리고 `docker-compose.yml`:

```yaml
version: "3.8"

services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    restart: unless-stopped
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
      - /usr/bin/docker:/usr/bin/docker
    group_add:
      - "${DOCKER_GID}"

volumes:
  jenkins_home:
```

실행은 `docker-compose up -d` (같은 디렉토리에 `.env`가 있어야 GID가 주입됨).

원래 명령에서 `$(which docker)`였던 부분은 호스트 docker 바이너리 경로를 그대로 마운트하는 거라, 대부분의 Ubuntu 환경에서는 `/usr/bin/docker`가 맞을 거야. 만약 `which docker` 결과가 다른 경로(예: snap으로 설치했다면 `/snap/bin/docker` 같은)로 나온다면 그 경로로 바꿔줘야 해. `which docker`로 한번 확인해보는 걸 추천해.

그리고 한 가지 더 짚을 점: 이 compose 파일을 EC2에 옮길 때 `jenkins_home`을 named volume으로 쓰면 데이터는 보존되지만 호스트 파일시스템에서 직접 들여다보긴 어려워. 나중에 백업/마이그레이션을 고려한다면 `./jenkins_home:/var/jenkins_home` 같은 bind mount로 바꿔서 호스트 경로에 명시적으로 두는 것도 고려해볼 만해. 어느 쪽을 선호해?