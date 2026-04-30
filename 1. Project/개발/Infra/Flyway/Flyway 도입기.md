# Flyway 정리

## 핵심 개념

Flyway는 **데이터베이스 스키마 버전 관리 도구**로, Git이 소스코드를 관리하듯이 데이터베이스 구조 변경을 체계적으로 관리합니다.

## 주요 기능

### 1. 자동 상태 추적 및 관리

- `flyway_schema_history` 테이블에서 실행 기록 자동 관리
- 현재 DB 버전 상태를 정확히 파악
- 어떤 마이그레이션이 실행되었는지/안되었는지 자동 판단

### 2. 순차적 마이그레이션 실행

- 버전 번호 순서대로 자동 실행 (V1 → V2 → V3...)
- 이미 실행된 마이그레이션은 스킵
- 의존성 문제 방지

### 3. 다양한 스키마 요소 관리

- **테이블**: 생성/수정/삭제
- **컬럼**: 추가/삭제/타입 변경
- **인덱스**: 생성/삭제/수정
- **제약조건**: 외래키, 유니크, 체크 제약조건
- **뷰, 프로시저, 함수, 트리거**
- **권한 관리**

### 4. 무결성 검증

- 체크섬으로 마이그레이션 파일 변경 감지
- 실행된 파일이 수정되면 에러 발생
- `flyway validate` 명령으로 검증 가능

### 5. 환경별 일관성 보장

- 개발/스테이징/프로덕션 모든 환경에서 동일한 방식으로 적용
- 환경마다 다른 상태여도 자동으로 동기화

### 6. CI/CD 통합

- 자동화된 배포 파이프라인에 쉽게 통합
- 명령어 한 줄로 마이그레이션 실행

## 왜 필요한가?

### 문제 상황들

**1. 팀 협업시 DB 동기화 문제**

```
상황: 개발자 A가 새 테이블을 추가했는데, 개발자 B가 모름
결과: 개발자 B의 로컬에서 애플리케이션 에러 발생

```

**2. 배포시 스키마 변경 누락**

```
상황: 스테이징에서는 잘 되는데 프로덕션 배포 후 에러
원인: 프로덕션에 스키마 변경사항 적용 안됨

```

**3. 수동 스크립트 실행의 위험성**

```
상황: DBA가 수동으로 DDL 스크립트 실행
위험: 실행 순서 틀림, 중복 실행, 누락 등

```

**4. 현재 DB 상태 파악 어려움**

```
질문: "현재 DB에 어떤 스키마 변경이 적용되어 있지?"
문제: 확인할 방법이 없어서 추측에 의존

```

### Flyway가 제공하는 해결책

**1. 자동화된 동기화**

- `flyway migrate` 한 번이면 모든 환경이 최신 상태로 동기화
- 사람의 실수 요소 제거

**2. 명확한 상태 파악**

```bash
flyway info
# 현재 버전, 실행 가능한 마이그레이션 목록 즉시 확인

```

**3. 안전한 실행**

- 이미 실행된 것은 자동 스킵
- 순서 보장으로 의존성 문제 방지
- 체크섬으로 파일 무결성 검증

**4. 협업 효율성**

- 새로운 팀원도 `flyway migrate`만 실행하면 환경 설정 완료
- 코드 리뷰처럼 마이그레이션도 리뷰 가능

**5. 배포 안정성**

- CI/CD 파이프라인에서 자동 실행
- 배포 실패 위험 대폭 감소

## 실제 도입 효과

### Before (Flyway 없이)

```
1. 개발자가 수동으로 DDL 스크립트 작성
2. 팀 챗에 "DB 스키마 변경했으니 XXX 스크립트 실행해주세요"
3. 각자 수동으로 실행 (누락, 순서 틀림 등 문제 발생)
4. 배포시 DBA가 수동으로 프로덕션에 적용
5. 문제 발생시 어떤 상태인지 파악하기 어려움

```

### After (Flyway 사용)

```
1. 개발자가 Flyway 마이그레이션 파일 작성
2. Git에 커밋/푸시
3. 팀원들은 git pull 후 flyway migrate 실행
4. CI/CD에서 자동으로 스테이징/프로덕션에 적용
5. 모든 환경의 상태를 명확히 파악 가능

```

## 결론

Flyway는 단순한 DDL 관리 도구가 아니라, **데이터베이스 변경의 전체 생명주기를 자동화하고 안전하게 관리하는 시스템**입니다.

특히 팀 규모가 커질수록, 환경이 많아질수록, 배포 빈도가 높아질수록 그 가치가 더욱 커집니다. 코드에 Git이 필수인 것처럼, 현대적인 애플리케이션 개발에서 데이터베이스 마이그레이션 도구는 거의 필수가 되었다고 볼 수 있습니다.

[WAS/DB 분리 환경에서 Flyway 도입 가이드](https://www.notion.so/WAS-DB-Flyway-24e0c7355c2a803d8d30fefd5218493d?pvs=21)

엥 붙이는건 개쉬운데 아직 장점을 모르겠다?

---

적용의 핵심은 JPA DDL Auto가 확실하지 않다는 점과 운영 서버의 데이터를 보호하기 위함이다.

# JPA DDL Auto Update의 운영 환경 문제점과 Flyway 해결법

## 🚨 DDL Auto Update의 직관적 문제점

### 1. **갑작스런 데이터 손실**

```java
// 개발자가 이렇게 수정했을 때
@Entity
public class User {
    @Column(length = 20)  // 기존 50자 → 20자로 축소
    private String username;
}

```

**서버 재시작 시 자동 실행**:

```sql
ALTER TABLE users ALTER COLUMN username TYPE VARCHAR(20);

```

**결과**: 20자 이상 사용자명 **강제 잘림** → 데이터 영구 손실

### 2. **예측 불가능한 서비스 중단**

```java
@Entity
public class UserActivity {
    @Index  // 인덱스 추가
    private LocalDateTime createdAt;
}

```

**대용량 테이블에서**:

- 5000만 건 테이블에 인덱스 생성
- **2-3시간 테이블 락**
- 관련 모든 서비스 완전 마비

### 3. **롤링 배포 시 서버 충돌**

```java
// v2.0에서 컬럼 추가
@Entity
public class Product {
    private String name;
    private String category;  // 새 컬럼
}

```

**롤링 배포 중**:

- v2.0 서버: `category` 컬럼 생성
- v1.0 서버: `category` 컬럼 모름
- **서버 간 SQL 충돌** → 에러 발생

## ✅ Flyway의 직관적 해결법

### 1. **데이터 보호를 위한 사전 검증**

```sql
-- V1__Safe_username_resize.sql
-- 먼저 확인하고
DO $$
BEGIN
    IF EXISTS (SELECT 1 FROM users WHERE LENGTH(username) > 20) THEN
        RAISE EXCEPTION '20자 초과 사용자명이 %개 존재합니다',
            (SELECT COUNT(*) FROM users WHERE LENGTH(username) > 20);
    END IF;
END $$;

-- 안전하면 변경
ALTER TABLE users ALTER COLUMN username TYPE VARCHAR(20);

```

**결과**: 데이터 손실 사전 방지

### 2. **무중단 인덱스 생성**

```sql
-- V2__Add_activity_index.sql
-- 동시성 인덱스로 서비스 영향 없음
CREATE INDEX CONCURRENTLY idx_user_activity_created_at
ON user_activity(created_at);

```

**결과**: 백그라운드에서 인덱스 생성, 서비스 정상 운영

### 3. **호환 가능한 점진적 변경**

```sql
-- V3__Add_product_category.sql
-- 1단계: 호환되는 컬럼 추가 (NULL 허용)
ALTER TABLE products ADD COLUMN category VARCHAR(100);

```

```java
// 애플리케이션도 호환 코드로
@Entity
public class Product {
    private String name;

    @Column(name = "category")
    private String category;  // NULL 허용, 기존 버전과 호환

    public String getCategory() {
        return category != null ? category : "미분류";
    }
}

```

```sql
-- V4__Populate_category.sql
-- 2단계: 나중에 데이터 채우기
UPDATE products SET category = '기본값' WHERE category IS NULL;

```

**결과**: 롤링 배포 시에도 모든 서버 정상 동작

## 📊 한눈에 보는 차이점

|상황|DDL Auto Update|Flyway|
|---|---|---|
|**컬럼 크기 축소**|❌ 즉시 잘림 → 데이터 손실|✅ 사전 검증 → 안전한 변경|
|**인덱스 추가**|❌ 테이블 락 → 서비스 중단|✅ CONCURRENTLY → 무중단|
|**롤링 배포**|❌ 서버 충돌 → 에러 발생|✅ 호환 변경 → 안전한 배포|
|**제어 가능성**|❌ 자동 실행 → 예측 불가|✅ 명시적 관리 → 완전 통제|

## 🎯 핵심 메시지

### **DDL Auto Update = 운영 환경의 시한폭탄**

- 언제 터질지 모르는 데이터 손실
- 예고 없는 서비스 중단
- 개발팀 새벽 긴급 대응

### **Flyway = 운영 환경의 안전망**

- 모든 변경 사전 검토
- 데이터 보호 우선
- 무중단 서비스 보장

---

---

# CICD 추가

```java
name: CI Check Pipeline

on:
  pull_request:
    branches: [ "main" ]

jobs:
  ci-check:
    runs-on: ubuntu-24.04
    permissions:
      contents: read
      checks: write
      pull-requests: write
    environment: prod
    
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: test
          MYSQL_DATABASE: testdb
          MYSQL_USER: test
          MYSQL_PASSWORD: test
        options: >-
          --health-cmd="mysqladmin ping --silent"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 3
        ports:
          - 3306:3306

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.SUBMODULE_KEY }}
          submodules: true

      - name: Update Submodule
        run: git submodule update --remote

      - name: Setup JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: gradle

      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@af1da67850ed9a4cedd57bfd976089dd991e2582
        with:
          cache-read-only: false

      - name: Wait for MySQL
        run: |
          until mysqladmin ping -h localhost -P 3306 -u test -ptest --silent; do
            echo 'Waiting for MySQL...'
            sleep 2
          done

      - name: Check Flyway Migration
        run: ./gradlew flywayMigrate -Pflyway.url=jdbc:mysql://localhost:3306/testdb -Pflyway.user=test -Pflyway.password=test
        working-directory: ./backend

      - name: Validate Flyway Migration
        run: ./gradlew flywayValidate -Pflyway.url=jdbc:mysql://localhost:3306/testdb -Pflyway.user=test -Pflyway.password=test
        working-directory: ./backend

      - name: Check Test Success
        run: ./gradlew test --parallel --build-cache
        working-directory: ./backend
        env:
          SPRING_DATASOURCE_URL: jdbc:mysql://localhost:3306/testdb
          SPRING_DATASOURCE_USERNAME: test
          SPRING_DATASOURCE_PASSWORD: test

      - name: Check Gradle Build Success
        run: ./gradlew build -x test --parallel --build-cache
        working-directory: ./backend

      - name: Publish Test Results
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: always()
        with:
          files: |
            backend/build/test-results/**/*.xml
          github_token: ${{ secrets.GITHUB_TOKEN }}
```

```java
name: Build and Deploy

on:
  pull_request:
    branches: [ "main" ]

jobs:
  ci:
    runs-on: ubuntu-24.04
    environment: prod
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.SUBMODULE_KEY }}
          submodules: recursive

      - name: Update Submodule
        run: git submodule update --remote

      - name: Generate short SHA
        id: meta
        run: echo "short_sha=$(echo ${{ github.sha }} | cut -c1-7)" >> $GITHUB_OUTPUT

      - name: Set up JDK
        uses: actions/setup-java@v3
        with:
          distribution: 'temurin'
          java-version: 21
          cache: gradle

      - name: Build JAR
        working-directory: ./backend
        run: ./gradlew clean build -x test --no-daemon --build-cache --parallel

      - name: Login Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and Push Docker Image
        uses: docker/build-push-action@v5
        with:
          context: ./backend
          platforms: linux/amd64,linux/arm64
          push: true
          cache-from: type=gha
          cache-to: type=gha,mode=max
          tags: |
            ${{ secrets.DOCKER_USERNAME }}/${{ secrets.DOCKER_REPOSITORY }}:release
            ${{ secrets.DOCKER_USERNAME }}/${{ secrets.DOCKER_REPOSITORY }}:${{ steps.meta.outputs.short_sha }}
          build-args: |
            SPRING_ACTIVE_PROFILE=prod

  database-migration:
    needs: ci
    runs-on: [ self-hosted, prod ]
    environment: prod
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.SUBMODULE_KEY }}
          submodules: true

      - name: Set up JDK
        uses: actions/setup-java@v3
        with:
          distribution: 'temurin'
          java-version: 21
          cache: gradle

      - name: Run Flyway Migration
        working-directory: ./backend
        run: |
          ./gradlew flywayMigrate \\
            -Pflyway.url=${{ secrets.PROD_DB_URL }} \\
            -Pflyway.user=${{ secrets.PROD_DB_USERNAME }} \\
            -Pflyway.password=${{ secrets.PROD_DB_PASSWORD }} \\
            --no-daemon

      - name: Validate Migration
        working-directory: ./backend
        run: |
          ./gradlew flywayValidate \\
            -Pflyway.url=${{ secrets.PROD_DB_URL }} \\
            -Pflyway.user=${{ secrets.PROD_DB_USERNAME }} \\
            -Pflyway.password=${{ secrets.PROD_DB_PASSWORD }} \\
            --no-daemon

  cd:
    needs: [ci, database-migration]
    runs-on: [ self-hosted, prod ]
    environment: prod
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.SUBMODULE_KEY }}
          submodules: true

      - name: Copy Docker Compose
        run: |
          sudo cp ./backend/docker/docker-compose.prod.yml /home/ubuntu/docker/docker-compose.yml

      - name: Deploy EC2
        run: |
          cd /home/ubuntu/docker
          sudo docker-compose pull backend
          sudo docker-compose up -d --build backend
          sudo docker image prune -f
```