# Pop-Con GitOps 배포 가이드

## 개요

이 레포는 EC2에서 Pop-Con 서비스를 배포하기 위한 설정 파일을 관리합니다.

```
gitops/
├── docker-compose.yml   # 서비스 오케스트레이션 (ECR에서 이미지 pull)
├── deploy.sh            # 배포 스크립트 (SSM → .env → docker compose up)
├── .env.example         # 환경변수 목록 (참고용)
├── .gitignore           # .env 제외
└── README.md            # 이 파일
```

---

## 전체 배포 흐름

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  1. Terraform (terraform-infra)                                 │
│     └── SSM Parameter Store에 환경변수 등록                      │
│                                                                 │
│  2. CI/CD (GitHub Actions) 또는 수동                             │
│     └── Docker 이미지 빌드 → ECR에 push                         │
│                                                                 │
│  3. EC2에서 배포 (이 레포)                                       │
│     └── git pull → deploy.sh 실행                               │
│         ├── SSM에서 환경변수 → .env 자동 생성                    │
│         ├── ECR 로그인                                          │
│         ├── docker compose pull (이미지 가져오기)                │
│         └── docker compose up -d (컨테이너 실행)                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 사전 준비 (최초 1회)

### 1. SSM Parameter Store에 환경변수 등록

terraform-infra에서 SSM 모듈을 통해 등록합니다.

```hcl
# terraform-infra/env/dev/main.tf
module "ssm_parameters" {
  source       = "../../modules/ssm-parameters"
  project_name = "popcon"

  parameters = {
    "ECR_REGISTRY"           = "123456789.dkr.ecr.ap-northeast-2.amazonaws.com"
    "SPRING_PROFILES_ACTIVE" = "prod"
    "DB_HOST"                = "popcon-mysql"
    "DB_PORT"                = "3306"
    "DB_NAME"                = "popcon"
    "JAVA_OPTS"              = "-Xms512m -Xmx512m"
    "REDIS_HOST"             = "popcon-redis"
    "REDIS_PORT"             = "6379"
  }

  secrets = {
    "DB_USERNAME"      = var.db_username
    "DB_PASSWORD"      = var.db_password
    "DB_ROOT_PASSWORD" = var.db_root_password
  }
}
```

```bash
cd terraform-infra/env/dev
terraform apply
```

필요한 환경변수 전체 목록은 `.env.example` 파일을 참고하세요.

### 2. ECR에 이미지 Push

이미지가 ECR에 올라가 있어야 합니다.

```bash
# ECR 로그인
aws ecr get-login-password --region ap-northeast-2 \
  | docker login --username AWS --password-stdin <ECR_REGISTRY>

# 백엔드
cd pop-con-backend
docker build -t <ECR_REGISTRY>/dev-app:backend-latest .
docker push <ECR_REGISTRY>/dev-app:backend-latest

# 프론트엔드
cd pop-con-frontend
docker build -t <ECR_REGISTRY>/dev-app:frontend-latest \
  --build-arg NEXT_PUBLIC_API_URL=http://<EC2_PUBLIC_IP>:8080 .
docker push <ECR_REGISTRY>/dev-app:frontend-latest
```

> 이 과정은 추후 GitHub Actions CI/CD로 자동화됩니다.

### 3. EC2 환경 설정

EC2에 다음이 설치되어 있어야 합니다:

```bash
# Docker 설치
sudo apt-get update
sudo apt-get install -y docker.io docker-compose-plugin

# 현재 유저에게 Docker 권한 부여
sudo usermod -aG docker $USER
newgrp docker

# AWS CLI 설치 (SSM 접근용)
sudo apt-get install -y awscli

# gitops 레포 클론
git clone https://github.com/<org>/gitops.git
cd gitops
chmod +x deploy.sh
```

---

## 배포 방법

### 최초 배포

```bash
cd gitops
./deploy.sh
```

### 업데이트 배포 (코드 변경 후)

```bash
cd gitops
git pull          # docker-compose.yml 변경사항 반영
./deploy.sh       # 새 이미지 pull + 재시작
```

### deploy.sh가 하는 일

```
[1/4] SSM Parameter Store에서 환경변수 가져오기 → .env 생성
[2/4] ECR 로그인
[3/4] ECR에서 최신 이미지 Pull
[4/4] docker compose up -d (컨테이너 실행)
```

---

## 배포 후 확인

```bash
# 컨테이너 상태 확인
docker compose ps

# 전체 로그 확인
docker compose logs -f

# 개별 서비스 로그
docker compose logs -f backend
docker compose logs -f frontend

# 헬스체크
curl http://localhost:8080/health
# → "Pop-Con Server is running"
```

---

## 운영 명령어

### 서비스 중지

```bash
docker compose down           # 컨테이너 중지 + 삭제 (데이터 유지)
docker compose down -v        # 컨테이너 + 볼륨(DB 데이터) 삭제 (주의!)
```

### 특정 서비스만 재시작

```bash
docker compose restart backend
docker compose restart frontend
```

### 로그 확인

```bash
docker compose logs backend --tail 100    # 최근 100줄
docker compose logs -f backend            # 실시간 로그
```

---

## 환경변수 변경 시

### 값 변경 (예: DB 비밀번호 변경)

```
1. terraform-infra에서 변수값 수정
2. terraform apply → SSM 업데이트
3. EC2에서 ./deploy.sh 재실행 → 새 .env 생성 → 반영
```

### 변수 추가 (예: REDIS_PASSWORD 추가)

```
1. terraform-infra SSM 모듈에 변수 추가
2. terraform apply → SSM 등록
3. docker-compose.yml에 환경변수 추가
   environment:
     REDIS_PASSWORD: ${REDIS_PASSWORD}
4. git push (docker-compose.yml 변경사항)
5. EC2에서 git pull → ./deploy.sh 재실행
```

---

## 서비스 구성

| 서비스 | 이미지 | 포트 | 역할 |
|--------|--------|------|------|
| frontend | ECR/dev-app:frontend-latest | 3000 | Next.js 웹 |
| backend | ECR/dev-app:backend-latest | 8080 | Spring Boot API |
| mysql | mysql:8.0 | 3306 | 데이터베이스 |
| redis | redis:7-alpine | 6379 | 캐시/세션 |

### 서비스 시작 순서

```
mysql (healthy) ──┐
                  ├──→ backend (started) ──→ frontend
redis (healthy) ──┘
```

mysql과 redis가 헬스체크를 통과한 후 backend가 시작되고, backend가 시작된 후 frontend가 시작됩니다.

---

## 트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| `deploy.sh: Permission denied` | 실행 권한 없음 | `chmod +x deploy.sh` |
| `SSM 파라미터 0개` | IAM 권한 부족 또는 경로 오류 | EC2 IAM Role에 SSM 읽기 권한 확인 |
| `ECR 로그인 실패` | IAM 권한 부족 | EC2 IAM Role에 ECR 권한 확인 |
| `이미지 pull 실패` | ECR에 이미지 없음 | 이미지가 push되었는지 확인 |
| `backend 연결 실패` | mysql이 아직 준비 안 됨 | `docker compose logs mysql`로 상태 확인 |
| `포트 충돌` | 이전 컨테이너 남아있음 | `docker compose down` 후 재실행 |

### EC2 IAM Role 필요 권한

```
- ssm:GetParametersByPath (SSM 환경변수 조회)
- ecr:GetAuthorizationToken (ECR 로그인)
- ecr:BatchGetImage (이미지 pull)
- ecr:GetDownloadUrlForLayer (이미지 pull)
```
