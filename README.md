# GAMERS Infrastructure

GAMERS 프로젝트의 인프라 구성 (MySQL, Redis, RabbitMQ)을 관리하는 리포지토리입니다.

## 구성 요소

- **MySQL 8.0**: 메인 데이터베이스
- **Redis 7**: 캐싱 및 세션 스토어
- **RabbitMQ 3.13**: 메시지 큐

## GCP 자동 배포 설정

GitHub에 `docker-compose.yaml` 파일을 푸시하면 자동으로 GCP Compute Engine VM에 배포됩니다.

### 1. GCP 설정

#### 1.1 Service Account 생성

```bash
# GCP Console에서 또는 gcloud CLI로 실행
gcloud iam service-accounts create gamers-deploy \
  --display-name="GAMERS Deployment Service Account"

# 필요한 권한 부여
gcloud projects add-iam-policy-binding just-rhythm-482512-q3 \
  --member="serviceAccount:gamers-deploy@just-rhythm-482512-q3.iam.gserviceaccount.com" \
  --role="roles/compute.instanceAdmin.v1"

gcloud projects add-iam-policy-binding just-rhythm-482512-q3 \
  --member="serviceAccount:gamers-deploy@just-rhythm-482512-q3.iam.gserviceaccount.com" \
  --role="roles/compute.osLogin"

# 키 생성 및 다운로드
gcloud iam service-accounts keys create ~/gamers-deploy-key.json \
  --iam-account=gamers-deploy@just-rhythm-482512-q3.iam.gserviceaccount.com
```

#### 1.2 Compute Engine VM 생성

```bash
# VM 인스턴스 생성 (예시)
gcloud compute instances create gamers-infra-vm \
  --project=just-rhythm-482512-q3 \
  --zone=us-central1-a \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --boot-disk-size=50GB \
  --boot-disk-type=pd-balanced \
  --tags=http-server,https-server \
  --metadata=enable-oslogin=TRUE

# 방화벽 규칙 생성 (필요한 포트만 열기)
gcloud compute firewall-rules create allow-gamers-services \
  --project=just-rhythm-482512-q3 \
  --allow=tcp:3306,tcp:6379,tcp:5672,tcp:15672 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=http-server
```

⚠️ **보안 주의사항**: 프로덕션 환경에서는 `--source-ranges`를 특정 IP로 제한하세요.

#### 1.3 VM에 초기 디렉토리 생성

```bash
# VM에 SSH 접속
gcloud compute ssh gamers-infra-vm --zone=us-central1-a

# 디렉토리 생성
mkdir -p ~/gamers-infra

# 로그아웃
exit
```

### 2. GitHub Secrets 설정

Repository Settings → Secrets and variables → Actions → New repository secret

다음 secrets를 추가하세요:

#### GCP 관련
- `GCP_SA_KEY`: Service Account JSON 키 전체 내용 (`~/gamers-deploy-key.json` 파일 내용)
- `GCP_VM_NAME`: VM 인스턴스 이름 (예: `gamers-infra-vm`)
- `GCP_VM_ZONE`: VM 존 (예: `us-central1-a`)

#### 데이터베이스 관련
- `MYSQL_ROOT_PASSWORD`: MySQL root 비밀번호
- `MYSQL_DATABASE`: 데이터베이스 이름 (예: `gamers_db`)
- `MYSQL_USER`: MySQL 사용자 이름
- `MYSQL_PASSWORD`: MySQL 사용자 비밀번호
- `DB_PORT`: 외부에 노출할 MySQL 포트 (예: `3306`)

#### RabbitMQ 관련
- `RABBITMQ_USER`: RabbitMQ 사용자 이름
- `RABBITMQ_PASSWORD`: RabbitMQ 비밀번호
- `RABBITMQ_VHOST`: RabbitMQ Virtual Host (예: `/`)

### 3. 배포 방법

#### 자동 배포
`main` 브랜치에 `docker-compose.yaml`을 푸시하면 자동으로 배포됩니다.

```bash
git add docker-compose.yaml
git commit -m "Update infrastructure configuration"
git push origin main
```

#### 수동 배포
필요시 GitHub Actions 탭에서 워크플로우를 수동으로 실행할 수 있습니다.

### 4. 배포 확인

```bash
# VM에 접속
gcloud compute ssh gamers-infra-vm --zone=us-central1-a

# 컨테이너 상태 확인
cd ~/gamers-infra
docker compose ps

# 로그 확인
docker compose logs -f
```

## 로컬 개발 환경

### 사전 요구사항
- Docker
- Docker Compose

### 실행 방법

1. `.env` 파일 생성

```bash
cp .env.example .env
# .env 파일을 편집하여 환경 변수 설정
```

2. Docker 네트워크 생성

```bash
docker network create gamers-network
```

3. 컨테이너 시작

```bash
docker compose up -d
```

4. 상태 확인

```bash
docker compose ps
```

### 접속 정보

- **MySQL**: `localhost:3306` (설정한 포트)
- **Redis**: `localhost:6379`
- **RabbitMQ**:
  - AMQP: `localhost:5672`
  - Management UI: `http://localhost:15672`

## 트러블슈팅

### GitHub Actions 실패 시

1. **인증 실패**: `GCP_SA_KEY` Secret이 올바른지 확인
2. **VM 접속 실패**: VM 이름과 Zone이 올바른지 확인
3. **Docker 명령 실패**: VM에 Docker가 설치되었는지 확인 (첫 배포 시 자동 설치됨)

### 컨테이너 재시작

```bash
# VM에서
cd ~/gamers-infra
docker compose restart
```

### 로그 확인

```bash
# 모든 서비스 로그
docker compose logs

# 특정 서비스 로그
docker compose logs mysql
docker compose logs redis
docker compose logs rabbitmq
```

### 데이터 초기화

```bash
# 모든 컨테이너와 볼륨 삭제
docker compose down -v

# 다시 시작
docker compose up -d
```

## 보안 권장사항

1. 강력한 비밀번호 사용
2. GCP 방화벽에서 필요한 IP만 허용
3. 프로덕션 환경에서는 VPC 내부 통신 권장
4. 정기적인 보안 업데이트
5. 백업 정책 수립

## 라이선스

MIT