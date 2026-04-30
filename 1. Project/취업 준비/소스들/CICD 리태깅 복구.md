
```
      - name: Backup Current Image Before Deploy
        run: |
          CURRENT_IMAGE=$(docker inspect docker-backend-1 --format='{{.Config.Image}}' 2>/dev/null || echo "none")
          echo "Current running image: $CURRENT_IMAGE"

          if [ "$CURRENT_IMAGE" != "none" ]; then
            docker tag "$CURRENT_IMAGE" ${{ secrets.DOCKER_USERNAME }}/${{ secrets.DOCKER_REPOSITORY }}:previous || true
            docker push ${{ secrets.DOCKER_USERNAME }}/${{ secrets.DOCKER_REPOSITORY }}:previous || true
            echo "✅ Backed up current running image as previous"
          else
            echo "⚠️ No running container found"
          fi
      - name: Deploy EC2
        run: |
          cd /home/ubuntu/docker
          docker-compose pull backend
          docker-compose up -d --build backend
          docker image prune -f
```
제공해주신 GitHub Actions 워크플로우 코드에서 **리태깅(Retagging)** 작업이 일어나는 부분은 크게 세 군데입니다.

각 단계별로 어떤 의도로 리태깅이 발생하는지 정리해 드립니다.

---

### 1. 배포 전: 현재 버전을 `previous`로 백업

가장 먼저 현재 실행 중인 안정적인 이미지를 나중을 위해 백업하는 단계입니다.

YAML

```
      - name: Backup Current Image Before Deploy
        run: |
          # 1. 현재 실행 중인 컨테이너의 이미지 이름을 가져옴
          CURRENT_IMAGE=$(docker inspect docker-backend-1 --format='{{.Config.Image}}' 2>/dev/null || echo "none")
          
          if [ "$CURRENT_IMAGE" != "none" ]; then
            # 2. 리태깅 발생: 현재 이미지를 'previous' 태그로 변경
            docker tag "$CURRENT_IMAGE" ${{ secrets.DOCKER_USERNAME }}/${{ secrets.DOCKER_REPOSITORY }}:previous || true
            
            # 3. 변경된 태그를 저장소에 푸시
            docker push ... :previous || true
```

### 2. 롤백 시: `latest`를 `previous`로 교체 (파일 수정)

이 부분은 이미지 자체를 리태깅한다기보다, 배포 정의서(`docker-compose.yml`) 내의 태그 이름을 리태깅된 버전으로 치환하여 실행하는 단계입니다.

YAML

```
      - name: Rollback if Health Check fails
        if: failure()
        run: |
          # docker-compose.yml 파일 내의 latest 문자열을 previous로 변경 (치환)
          sed -i 's/pickeat\/pickeat:latest/pickeat\/pickeat:previous/g' docker-compose.yml
```

### 3. 롤백 완료 후: `latest` 태그 정상화

롤백에 성공한 후, 저장소에 있는 `latest`가 실패한 버전이 아닌, 현재 복구된 버전(`previous`)을 가리키도록 다시 맞추는 작업입니다.

YAML

```
      - name: Update latest tag after successful rollback
        if: failure()
        run: |
          # 1. 백업해뒀던 previous 이미지를 가져옴
          docker pull ... :previous
          
          # 2. 리태깅 발생: previous 이미지를 다시 'latest'로 명명
          docker tag ... :previous ... :latest
          
          # 3. 최신화된 'latest'를 저장소에 다시 푸시
          docker push ... :latest
```

---

### 💡 요약하자면

- **백업 시:** `운영 중인 이미지` → `previous` (리태깅)
    
- **복구 시:** `previous` → `latest` (리태깅 후 덮어쓰기)
    

이 프로세스를 통해 배포가 실패하더라도 즉시 이전 단계의 이미지로 서비스를 복구할 수 있게 됩니다. 작성하신 자기소개서의 "Docker 리태깅 기반 자동 롤백"이 바로 이 코드들을 의미합니다.

---
"물론 블루-그린 배포가 가용성 측면에서 유리하지만, 한정된 리소스 내에서 **비용 효율성**을 극대화하면서도 **휴먼 에러를 방지**하는 데 집중했습니다. 컨테이너의 빠른 재구동 특성을 활용해 이미지 리태깅 전략을 세웠고, 여기에 Flyway와 사전 DB 백업 프로세스를 결합하여 **단순한 서버 복구를 넘어 데이터 정합성까지 보장되는 확실한 롤백 체계**를 구축하는 것이 프로젝트 상황에 더 적합한 엔지니어링적 선택이라고 판단했습니다."


"단일 인스턴스 내 포트 스위칭 방식도 검토했으나, 배포 시점의 **리소스 경합(Resource Contention)으로 인한 전체 시스템 다운 리스크**를 방지하고자 했습니다. 대신 Docker 리태깅과 Flyway를 조합하여, 자원 효율성을 지키면서도 **데이터 정합성까지 고려한 신속한 복구 체계**를 구축하는 방향을 선택했습니다."

블루-그린은 '성공적인 배포'를 무중단으로 하는 데 강점이 있지만, 본인은 '실패했을 때의 확실한 복구'에 더 비중을 두신 겁니다.

---
https://github.com/woowacourse-teams/2025-pick-eat/pull/284