# Pop-Con 인프라 전체 흐름 가이드

## 프로젝트 구조

```
4개 레포가 각자 역할을 가지고 협업합니다.

┌─────────────────────────────────────────────────────────────┐
│  pop-con-frontend        프론트엔드 소스코드 + Dockerfile   │
│  pop-con-backend         백엔드 소스코드 + Dockerfile       │
│  terraform-infra         AWS 인프라 코드 (IaC)              │
│  gitops                  배포 설정 (docker-compose, deploy) │
└─────────────────────────────────────────────────────────────┘
```

---

## 전체 흐름 한눈에 보기

```
[개발팀]                    [클라우드팀]                   [AWS]

코드 작성                   Terraform 코드 작성
    │                           │
    ▼                           ▼
GitHub에 push               GitHub에 push (PR)
    │                           │
    ▼                           ▼
(GitHub Actions)            (GitHub Actions)
Docker 이미지 빌드              │
    │                      terraform plan
    │                      terraform apply ──────────────→ VPC 생성
    │                           │                          ECR 생성
    │                           │                          SSM 파라미터 생성
    ▼                           │                              │
ECR에 이미지 push ─────────────────────────────────────→ ECR에 저장
                                │                              │
                                ▼                              │
                           EC2에 SSM 접속                      │
                           git pull (gitops)                   │
                           ./deploy.sh ───────────────→ SSM에서 환경변수 가져옴
                                │                        ECR에서 이미지 pull
                                ▼                              │
                           docker compose up                   │
                                │                              ▼
                                └──────────────────→ 서비스 실행!
                                                     (frontend, backend,
                                                      mysql, redis)
```

---

## 단계별 상세 설명

### 1단계: AWS 인프라 생성 (terraform-infra)

**누가**: 클라우드팀
**언제**: 최초 1회 (이후 변경 시에만)

```
terraform-infra/
├── modules/
│   ├── vpc/               # VPC, 서브넷, IGW
│   ├── ecr/               # Docker 이미지 저장소
│   └── ssm-parameters/    # 환경변수 저장소
└── env/dev/
    └── main.tf            # 위 모듈들을 호출
```

#### 실행 방법

```
코드 수정 → feature 브랜치 push → PR 생성
→ GitHub Actions가 자동으로 terraform plan (PR 코멘트에 결과 표시)
→ 리뷰 후 main에 머지
→ GitHub Actions가 자동으로 terraform apply
→ AWS에 리소스 생성됨
```

#### 생성되는 AWS 리소스

| 리소스 | 용도 |
|--------|------|
| VPC | 네트워크 격리 |
| Public/Private Subnet | EC2, RDS 배치 |
| Internet Gateway | 외부 통신 |
| ECR | Docker 이미지 저장소 |
| SSM Parameter Store | 환경변수 (DB 비밀번호 등) |

---

### 2단계: Docker 이미지 빌드 & ECR Push

**누가**: 개발팀 (추후 CI/CD로 자동화)
**언제**: 코드 변경 후 배포할 때마다

#### 프론트엔드 이미지

```
pop-con-frontend/
├── Dockerfile          # 3단계 빌드 (deps → builder → runner)
├── .dockerignore       # 불필요 파일 제외
└── next.config.ts      # output: 'standalone' 필수
```

```
npm ci → npm run build → node server.js
결과: ~330MB 경량 이미지
```

#### 백엔드 이미지

```
pop-con-backend/
├── Dockerfile          # 2단계 빌드 (build → runtime)
├── .dockerignore       # 민감 파일 제외
└── build.gradle        # 의존성 정의
```

```
gradlew dependencies → gradlew bootJar → java -jar app.jar
결과: ~516MB 이미지
```

#### ECR에 Push (수동)

```bash
# ECR 로그인
aws ecr get-login-password --region ap-northeast-2 \
  | docker login --username AWS --password-stdin <ECR_REGISTRY>

# 빌드 & Push (백엔드)
docker build -t <ECR_REGISTRY>/dev-app:backend-latest .
docker push <ECR_REGISTRY>/dev-app:backend-latest

# 빌드 & Push (프론트엔드)
docker build -t <ECR_REGISTRY>/dev-app:frontend-latest \
  --build-arg NEXT_PUBLIC_API_URL=http://<EC2_IP>:8080 .
docker push <ECR_REGISTRY>/dev-app:frontend-latest
```

> 추후 GitHub Actions CI/CD로 자동화 예정

---

### 3단계: 환경변수 관리 (SSM Parameter Store)

**누가**: 클라우드팀
**언제**: 1단계와 동시 (terraform apply 시 자동)

#### 환경변수 저장 경로

```
AWS SSM Parameter Store

/popcon/ECR_REGISTRY           = "123456789.dkr.ecr..."    (String)
/popcon/SPRING_PROFILES_ACTIVE = "prod"                     (String)
/popcon/DB_HOST                = "popcon-mysql"              (String)
/popcon/DB_PORT                = "3306"                      (String)
/popcon/DB_NAME                = "popcon"                    (String)
/popcon/JAVA_OPTS              = "-Xms512m -Xmx512m"        (String)
/popcon/REDIS_HOST             = "popcon-redis"              (String)
/popcon/REDIS_PORT             = "6379"                      (String)
/popcon/DB_USERNAME            = "***"                       (SecureString)
/popcon/DB_PASSWORD            = "***"                       (SecureString)
/popcon/DB_ROOT_PASSWORD       = "***"                       (SecureString)
```

#### 민감 변수 전달 경로

```
GitHub Secrets (사람이 1회 등록)
    │  DB_PASSWORD = "실제비밀번호"
    ▼
GitHub Actions 워크플로우
    │  TF_VAR_db_password = "실제비밀번호"
    ▼
Terraform (var.db_password)
    │
    ▼
AWS SSM Parameter Store (SecureString, KMS 암호화)
    │
    ▼
EC2 deploy.sh (--with-decryption → 복호화)
    │
    ▼
.env 파일 → docker-compose → 컨테이너
```

---

### 4단계: EC2 배포 (gitops)

**누가**: 클라우드팀
**언제**: 이미지가 ECR에 올라간 후

```
gitops/
├── docker-compose.yml   # 서비스 오케스트레이션
├── deploy.sh            # 배포 자동화 스크립트
├── .env.example         # 환경변수 목록 (참고용)
└── .gitignore           # .env 제외
```

#### 배포 명령어

```bash
# EC2에 SSM 접속 (SSH 대신 SSM Session Manager 사용)
aws ssm start-session --target <instance-id>

# 최초
git clone https://github.com/kt-cloud-TECHUP-T1/gitops.git
cd gitops
chmod +x deploy.sh

# 배포
./deploy.sh
```

#### deploy.sh가 하는 일

```
[1/4] SSM에서 환경변수 가져와 .env 생성
      aws ssm get-parameters-by-path --path /popcon --with-decryption

[2/4] ECR 로그인
      aws ecr get-login-password | docker login

[3/4] ECR에서 이미지 Pull
      docker compose pull

[4/4] 컨테이너 실행
      docker compose up -d
```

#### 실행되는 서비스

```
┌─────────────────── popcon-network ───────────────────┐
│                                                       │
│  frontend (3000) ──→ backend (8080) ──→ mysql (3306)  │
│                          │                            │
│                          └──→ redis (6379)            │
│                                                       │
└───────────────────────────────────────────────────────┘

시작 순서:
mysql (healthy) ──┐
                  ├──→ backend (started) ──→ frontend
redis (healthy) ──┘
```

---

## 레포별 역할 요약

| 레포 | 담당 | 내용 |
|------|------|------|
| pop-con-frontend | 프론트팀 | Next.js 코드 + Dockerfile |
| pop-con-backend | 백엔드팀 | Spring Boot 코드 + Dockerfile |
| terraform-infra | 클라우드팀 | VPC, ECR, SSM 등 AWS 리소스 |
| gitops | 클라우드팀 | docker-compose, deploy.sh |

---

## 자주 하는 작업

### 코드 변경 후 재배포

```
1. 개발팀: 코드 수정 → Docker 이미지 빌드 → ECR push
2. EC2에서: ./deploy.sh (새 이미지 자동 pull)
```

### 환경변수 변경

```
1. terraform-infra에서 env/dev/main.tf 수정
2. PR → 머지 → terraform apply (자동)
3. EC2에서: ./deploy.sh (새 .env 자동 생성)
```

### docker-compose 변경 (서비스 추가 등)

```
1. gitops에서 docker-compose.yml 수정
2. push
3. EC2에서: git pull → ./deploy.sh
```

### 인프라 변경 (서브넷 추가 등)

```
1. terraform-infra에서 모듈/변수 수정
2. PR → 팀원 리뷰 → 머지 → terraform apply (자동)
```

---

## 향후 단계

| 단계 | 환경 | 변경 사항 |
|------|------|----------|
| 1단계 (현재) | EC2 + docker-compose | 수동 배포 |
| 1.5단계 | EC2 + GitHub Actions | deploy.sh 자동 실행 |
| 2단계 | K3s | docker-compose → K3s manifest |
| 3단계 | EKS + MSA | K3s → Helm/ArgoCD |

각 단계에서 **gitops 레포의 내용만 바뀌고**, Dockerfile과 ECR은 그대로 유지됩니다.
