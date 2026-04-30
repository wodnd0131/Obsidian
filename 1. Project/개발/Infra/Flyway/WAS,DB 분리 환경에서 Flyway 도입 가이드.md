## 1. 아키텍처 개요
![[../../../../Repository/WAS,DB 분리 환경에서 Flyway 도입 가이드-1.png]]

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   WAS Server    │    │  DB Server      │    │ CI/CD Pipeline  │
│                 │    │                 │    │                 │
│ ┌─────────────┐ │    │ ┌─────────────┐ │    │ ┌─────────────┐ │
│ │     App     │◄┼────┼►│   MySQL     │ │    │ │   Flyway    │ │
│ │             │ │    │ │             │ │    │ │ Migration   │ │
│ └─────────────┘ │    │ └─────────────┘ │    │ └─────────────┘ │
│                 │    │                 │    │       │         │
└─────────────────┘    └─────────────────┘    └───────┼─────────┘
                                                       │
                                                  Network Access

```

## 2. DB 서버 설정

### docker-compose.db.yml (DB 서버용)

```yaml
version: '3.8'

services:
  mysql:
    image: mysql:8.0
    container_name: production-mysql
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
      - ./mysql/conf.d:/etc/mysql/conf.d
    networks:
      - db-network
    restart: unless-stopped

volumes:
  mysql_data:
    driver: local

networks:
  db-network:
    driver: bridge

```

### DB 서버 환경변수 (.env.db)

```
MYSQL_ROOT_PASSWORD=super_secure_root_password
MYSQL_DATABASE=myapp_production
MYSQL_USER=app_user
MYSQL_PASSWORD=secure_app_password

```

### MySQL 설정 (mysql/conf.d/my.cnf)

```
[mysqld]
# 네트워크 접근 허용
bind-address = 0.0.0.0

# 성능 최적화
innodb_buffer_pool_size = 1G
max_connections = 200

# 로깅
general_log = 1
general_log_file = /var/log/mysql/general.log
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2

# 보안
local_infile = 0

```

## 3. WAS 서버 설정

### docker-compose.was.yml (WAS 서버용)

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: myapp-was
    ports:
      - "8080:8080"
      - "8443:8443"
    environment:
      # 외부 DB 연결 정보
      DB_HOST: ${DB_HOST}
      DB_PORT: ${DB_PORT}
      DB_NAME: ${DB_NAME}
      DB_USER: ${DB_USER}
      DB_PASSWORD: ${DB_PASSWORD}

      # 애플리케이션 설정
      SPRING_PROFILES_ACTIVE: production
      JVM_OPTS: "-Xms512m -Xmx2g"
    volumes:
      - ./logs:/app/logs
      - ./config:/app/config
    networks:
      - was-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "<http://localhost:8080/health>"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  was-network:
    driver: bridge

```

### WAS 서버 환경변수 (.env.was)

```
DB_HOST=192.168.1.100  # DB 서버 IP
DB_PORT=3306
DB_NAME=myapp_production
DB_USER=app_user
DB_PASSWORD=secure_app_password

```

## 4. Flyway 전용 설정

### flyway/ 디렉토리 구조

```
flyway/
├── docker-compose.flyway.yml
├── conf/
│   ├── flyway.production.conf
│   ├── flyway.staging.conf
│   └── flyway.development.conf
├── sql/
│   ├── V1__Create_users_table.sql
│   ├── V2__Add_email_column.sql
│   └── V3__Create_orders_table.sql
└── scripts/
    ├── migrate.sh
    ├── rollback.sh
    └── status.sh

```

### docker-compose.flyway.yml

```yaml
version: '3.8'

services:
  flyway:
    image: flyway/flyway:latest
    container_name: flyway-migration
    volumes:
      - ./sql:/flyway/sql
      - ./conf:/flyway/conf
      - ./scripts:/flyway/scripts
    environment:
      # 환경별로 다른 설정 파일 사용
      FLYWAY_CONFIG_FILES: /flyway/conf/flyway.${ENV:-production}.conf
    networks:
      - migration-network

networks:
  migration-network:
    driver: bridge

```

### 환경별 Flyway 설정

### flyway/conf/flyway.production.conf

```
# 프로덕션 DB 연결
flyway.url=jdbc:mysql://192.168.1.100:3306/myapp_production
flyway.user=app_user
flyway.password=secure_app_password

# 마이그레이션 설정
flyway.locations=filesystem:/flyway/sql
flyway.schemas=myapp_production
flyway.table=flyway_schema_history

# 안전 설정
flyway.validateOnMigrate=true
flyway.cleanDisabled=true
flyway.baselineOnMigrate=true
flyway.outOfOrder=false

# 연결 설정
flyway.connectRetries=60
flyway.connectRetriesInterval=1

```

### flyway/conf/flyway.staging.conf

```
flyway.url=jdbc:mysql://192.168.1.101:3306/myapp_staging
flyway.user=app_user
flyway.password=staging_password

flyway.locations=filesystem:/flyway/sql
flyway.schemas=myapp_staging
flyway.table=flyway_schema_history
flyway.validateOnMigrate=true
flyway.cleanDisabled=false
flyway.baselineOnMigrate=true

```

## 5. 마이그레이션 실행 스크립트

### scripts/migrate.sh

```bash
#!/bin/bash

# 환경 설정
ENV=${1:-production}
ACTION=${2:-migrate}

echo "=== Flyway Migration ==="
echo "Environment: $ENV"
echo "Action: $ACTION"
echo "========================"

# DB 연결 테스트
echo "Testing database connection..."
docker run --rm --network host \\
  -v "$(pwd)/conf:/flyway/conf" \\
  flyway/flyway:latest \\
  -configFiles="/flyway/conf/flyway.${ENV}.conf" \\
  info

if [ $? -ne 0 ]; then
    echo "❌ Database connection failed!"
    exit 1
fi

echo "✅ Database connection successful!"

# 마이그레이션 실행
echo "Running migration..."
docker run --rm --network host \\
  -v "$(pwd)/sql:/flyway/sql" \\
  -v "$(pwd)/conf:/flyway/conf" \\
  flyway/flyway:latest \\
  -configFiles="/flyway/conf/flyway.${ENV}.conf" \\
  $ACTION

if [ $? -eq 0 ]; then
    echo "✅ Migration completed successfully!"
else
    echo "❌ Migration failed!"
    exit 1
fi

```

### scripts/status.sh

```bash
#!/bin/bash

ENV=${1:-production}

echo "=== Flyway Status Check ==="
echo "Environment: $ENV"
echo "============================"

docker run --rm --network host \\
  -v "$(pwd)/conf:/flyway/conf" \\
  flyway/flyway:latest \\
  -configFiles="/flyway/conf/flyway.${ENV}.conf" \\
  info

echo ""
echo "=== Migration History ==="
docker run --rm --network host \\
  -v "$(pwd)/conf:/flyway/conf" \\
  mysql:8.0 mysql \\
  -h 192.168.1.100 \\
  -u app_user \\
  -psecure_app_password \\
  myapp_production \\
  -e "SELECT installed_rank, version, description, installed_on, success FROM flyway_schema_history ORDER BY installed_rank;"

```

## 6. CI/CD 파이프라인 통합

### GitHub Actions (.github/workflows/deploy.yml)

```yaml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  database-migration:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Flyway
        run: |
          cd flyway
          chmod +x scripts/*.sh

      - name: Run database migration
        env:
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
        run: |
          cd flyway
          # 설정 파일에 실제 비밀번호 주입
          sed -i "s/secure_app_password/$DB_PASSWORD/g" conf/flyway.production.conf
          ./scripts/migrate.sh production migrate

      - name: Verify migration
        run: |
          cd flyway
          ./scripts/status.sh production

  deploy-was:
    needs: database-migration
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to WAS server
        run: |
          # WAS 서버에 SSH 접속하여 애플리케이션 배포
          ssh ${{ secrets.WAS_USER }}@${{ secrets.WAS_HOST }} << 'EOF'
            cd /app
            docker-compose -f docker-compose.was.yml pull
            docker-compose -f docker-compose.was.yml up -d
          EOF

```

## 7. 네트워크 보안 설정

### DB 서버 방화벽 (iptables)

```bash
# DB 서버에서 실행
# WAS 서버와 CI/CD 서버에서만 접근 허용
iptables -A INPUT -p tcp --dport 3306 -s 192.168.1.110 -j ACCEPT  # WAS 서버
iptables -A INPUT -p tcp --dport 3306 -s 192.168.1.120 -j ACCEPT  # CI/CD 서버
iptables -A INPUT -p tcp --dport 3306 -j DROP  # 기타 모든 접근 차단

```

### MySQL 사용자 권한 설정

```sql
-- 애플리케이션용 사용자 (WAS에서 사용)
CREATE USER 'app_user'@'192.168.1.110' IDENTIFIED BY 'secure_app_password';
GRANT SELECT, INSERT, UPDATE, DELETE ON myapp_production.* TO 'app_user'@'192.168.1.110';

-- 마이그레이션용 사용자 (CI/CD에서 사용)
CREATE USER 'migration_user'@'192.168.1.120' IDENTIFIED BY 'migration_password';
GRANT ALL PRIVILEGES ON myapp_production.* TO 'migration_user'@'192.168.1.120';

-- 권한 적용
FLUSH PRIVILEGES;

```

## 8. 운영 방법

### 일반적인 배포 프로세스

```bash
# 1. 개발환경에서 마이그레이션 테스트
cd flyway
./scripts/migrate.sh development

# 2. 스테이징 환경에서 테스트
./scripts/migrate.sh staging

# 3. 프로덕션 배포 (CI/CD 또는 수동)
./scripts/migrate.sh production

# 4. WAS 서버 배포
ssh was-server "cd /app && docker-compose -f docker-compose.was.yml up -d"

```

### 응급 상황 대응

```bash
# 1. 마이그레이션 실패시 상태 확인
./scripts/status.sh production

# 2. 실패한 마이그레이션 수정
docker run --rm --network host \\
  -v "$(pwd)/conf:/flyway/conf" \\
  flyway/flyway:latest \\
  -configFiles="/flyway/conf/flyway.production.conf" \\
  repair

# 3. 특정 버전으로 베이스라인 설정
docker run --rm --network host \\
  -v "$(pwd)/conf:/flyway/conf" \\
  flyway/flyway:latest \\
  -configFiles="/flyway/conf/flyway.production.conf" \\
  baseline -baselineVersion=5

```

이렇게 설정하면 WAS와 DB가 분리된 환경에서도 안전하고 체계적으로 데이터베이스 마이그레이션을 관리할 수 있습니다.