# Pop-Con EC2 배포 가이드

## 개요

이 가이드는 EC2에서 Pop-Con 서비스를 Docker로 실행하기 위한 배포 파일을 설명합니다.

### 전체 흐름

```
terraform apply                    EC2에서 deploy.sh 실행
      │                                    │
      ▼                                    ▼
┌──────────────┐               ┌──────────────────┐
│ SSM Parameter │               │ SSM에서 환경변수  │
│ Store에 저장   │ ───────────→  │ 가져와서 .env 생성│
└──────────────┘               └────────┬─────────┘
                                        │
                                        ▼
                               ┌──────────────────┐
                               │ docker-compose로  │
                               │ 컨테이너 실행      │
                               └──────────────────┘
```

---

## 파일 구조

```
terraform-infra/
├── main.tf                        # SSM 파라미터 정의 (terraform apply 대상)
├── variables.tf                   # DB 비밀번호 변수 선언
├── terraform.tfvars.example       # 실제 값 입력 템플릿
├── modules/
│   └── ssm-parameters/            # SSM Parameter Store 모듈
│       ├── main.tf                # String / SecureString 리소스 정의
│       ├── variables.tf           # 모듈 입력 변수
│       └── outputs.tf             # 생성된 파라미터 ARN 출력
└── deploy/                        # EC2 배포 파일
    ├── docker-compose.yml         # 컨테이너 오케스트레이션
    └── deploy.sh                  # SSM → .env → docker-compose 실행
```

---

## 1단계: SSM Parameter Store에 환경변수 등록

### 1-1. terraform.tfvars 생성

```bash
cd terraform-infra
cp terraform.tfvars.example terraform.tfvars
```

`terraform.tfvars`를 열어 실제 값을 입력합니다:

```hcl
db_username      = "popcon"
db_password      = "실제비밀번호입력"
db_root_password = "실제비밀번호입력"
```

> 이 파일은 `.gitignore`에 등록되어 있어 git에 올라가지 않습니다.

### 1-2. main.tf에서 EC2 퍼블릭 IP 수정

```hcl
# main.tf 36번째 줄
"NEXT_PUBLIC_API_URL" = "http://EC2_PUBLIC_IP:8080"  # ← 실제 EC2 IP로 변경
```

### 1-3. terraform apply

```bash
terraform init
terraform plan      # 변경사항 확인
terraform apply     # SSM에 파라미터 등록
```

적용 후 AWS 콘솔에서 확인:
- AWS Console → Systems Manager → Parameter Store
- `/popcon/DB_HOST`, `/popcon/DB_PASSWORD` 등이 생성됨

### 등록되는 파라미터 목록

| SSM 경로 | 타입 | 값 |
|---------|------|-----|
| `/popcon/SPRING_PROFILES_ACTIVE` | String | `prod` |
| `/popcon/DB_HOST` | String | `popcon-mysql` |
| `/popcon/DB_PORT` | String | `3306` |
| `/popcon/DB_NAME` | String | `popcon` |
| `/popcon/NEXT_PUBLIC_API_URL` | String | `http://EC2_IP:8080` |
| `/popcon/JAVA_OPTS` | String | `-Xms512m -Xmx512m` |
| `/popcon/DB_USERNAME` | **SecureString** | (암호화 저장) |
| `/popcon/DB_PASSWORD` | **SecureString** | (암호화 저장) |
| `/popcon/DB_ROOT_PASSWORD` | **SecureString** | (암호화 저장) |

---

## 2단계: EC2 사전 준비

### 2-1. EC2에 필요한 것

- **Docker, Docker Compose**: 컨테이너 실행용
- **AWS CLI**: SSM Parameter Store 조회용
- **Git**: 소스코드 클론용
- **IAM Role**: EC2에 SSM 읽기 권한 (`ssm:GetParametersByPath`, `ssm:GetParameter`)

### 2-2. 소스코드 클론

```bash
# EC2에서 실행
cd /home/ec2-user
mkdir popcon && cd popcon

git clone https://github.com/your-org/pop-con-frontend.git
git clone https://github.com/your-org/pop-con-backend.git
```

### 2-3. 배포 파일 복사

deploy 폴더의 파일을 EC2의 popcon 디렉토리에 복사합니다:

```bash
# EC2의 최종 디렉토리 구조
/home/ec2-user/popcon/
├── pop-con-frontend/        # git clone
├── pop-con-backend/         # git clone
├── docker-compose.yml       # deploy/ 에서 복사
└── deploy.sh                # deploy/ 에서 복사
```

---

## 3단계: 배포 실행

```bash
cd /home/ec2-user/popcon
chmod +x deploy.sh
./deploy.sh
```

### deploy.sh가 하는 일

```
[1/4] SSM Parameter Store에서 환경변수 조회
      → aws ssm get-parameters-by-path로 /popcon/ 하위 모든 파라미터 조회
      → .env 파일 생성 (chmod 600, 소유자만 읽기)

[2/4] 소스코드 업데이트
      → git pull로 최신 코드 반영

[3/4] Docker Compose 빌드 및 실행
      → docker compose --env-file .env up --build -d
      → .env의 값이 docker-compose.yml의 ${변수}에 주입됨

[4/4] 컨테이너 상태 확인
      → 실행 결과 및 접속 URL 출력
```

### 실행 결과 예시

```
=== Pop-Con 배포 시작 ===
[1/4] SSM Parameter Store에서 환경변수 조회 중...
[1/4] 완료 - 9개 환경변수 로드
[2/4] 소스코드 업데이트 중...
[2/4] 완료
[3/4] Docker Compose 빌드 및 실행 중...
[3/4] 완료
[4/4] 컨테이너 상태 확인...
NAME              STATUS
popcon-frontend   Up 5 seconds
popcon-backend    Up 5 seconds
popcon-mysql      Up 10 seconds (healthy)

=== 배포 완료 ===
Frontend: http://3.34.xxx.xxx:3000
Backend:  http://3.34.xxx.xxx:8080/health
```

---

## docker-compose.yml 컨테이너 구성

```
┌─────────────────────────────────────────────────┐
│  popcon-network                                   │
│                                                   │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐      │
│  │frontend │    │backend  │    │ mysql   │      │
│  │ :3000   │───→│ :8080   │───→│ :3306   │      │
│  └─────────┘    └─────────┘    └─────────┘      │
│  Next.js         Spring Boot     MySQL 8.0       │
│                                  │               │
│                           mysql-data (영구 볼륨)  │
└─────────────────────────────────────────────────┘
```

| 서비스 | 이미지 | 포트 | 시작 조건 |
|--------|--------|------|----------|
| frontend | pop-con-frontend Dockerfile로 빌드 | 3000 | backend 시작 후 |
| backend | pop-con-backend Dockerfile로 빌드 | 8080 | mysql 헬스체크 통과 후 |
| mysql | mysql:8.0 공식 이미지 | 3306 | 즉시 |

---

## 환경변수 흐름 (전체)

```
terraform.tfvars (로컬, git 미포함)
    │
    ▼
terraform apply
    │
    ▼
AWS SSM Parameter Store
    │  /popcon/DB_HOST = "popcon-mysql"
    │  /popcon/DB_PASSWORD = "***" (암호화)
    │
    ▼
deploy.sh (EC2에서 실행)
    │  aws ssm get-parameters-by-path
    │
    ▼
.env 파일 (EC2, chmod 600)
    │  DB_HOST=popcon-mysql
    │  DB_PASSWORD=***
    │
    ▼
docker-compose.yml
    │  environment:
    │    DB_HOST: ${DB_HOST}      ← .env에서 읽음
    │    DB_PASSWORD: ${DB_PASSWORD}
    │
    ▼
컨테이너 내부 환경변수
    Spring Boot가 DB_HOST, DB_PASSWORD로 DB 접속
```

> 비밀번호가 코드에 직접 포함되지 않고, SSM → .env → 컨테이너 경로로 안전하게 주입됩니다.

---

## 운영 명령어

```bash
# 컨테이너 상태 확인
docker compose ps

# 로그 확인
docker compose logs -f backend    # 백엔드 로그
docker compose logs -f frontend   # 프론트 로그
docker compose logs -f mysql      # DB 로그

# 재배포 (코드 변경 후)
./deploy.sh

# 서비스 중지
docker compose down

# 서비스 중지 + DB 데이터 삭제
docker compose down -v
```

---

## 주의사항

1. **EC2 IAM Role**: EC2에 `ssm:GetParametersByPath` 권한이 있는 IAM Role이 연결되어야 합니다
2. **보안그룹**: EC2의 보안그룹에서 3000(프론트), 8080(백엔드) 포트를 열어야 합니다
3. **.env 파일**: deploy.sh가 자동 생성하며, `chmod 600`으로 보호됩니다. 수동 편집하지 마세요
4. **DB 데이터**: `docker compose down`으로 중지해도 `mysql-data` 볼륨에 데이터가 유지됩니다. `-v` 옵션을 붙이면 데이터도 삭제됩니다
5. **NEXT_PUBLIC_API_URL**: EC2 퍼블릭 IP가 변경되면 SSM 파라미터를 업데이트하고 재배포해야 합니다
