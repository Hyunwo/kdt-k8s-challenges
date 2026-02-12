# SSM Parameter Store 모듈

## 개요

AWS SSM Parameter Store에 환경변수를 등록하는 Terraform 모듈입니다.
EC2에서 `deploy.sh` 실행 시 이 파라미터들을 가져와 `.env` 파일을 생성합니다.

## 전체 흐름

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  1. GitHub Secrets에 DB 비밀번호 등록 (사람이 1회)                │
│     ├── DB_USERNAME                                              │
│     ├── DB_PASSWORD                                              │
│     └── DB_ROOT_PASSWORD                                         │
│                                                                  │
│  2. GitHub Actions (terraform apply)                             │
│     ├── OIDC로 AWS 인증                                          │
│     ├── TF_VAR_db_password=${{ secrets.DB_PASSWORD }}             │
│     └── terraform apply → SSM에 파라미터 생성                     │
│                                                                  │
│  3. AWS SSM Parameter Store                                      │
│     ├── /popcon/DB_HOST = "popcon-mysql"         (String)        │
│     ├── /popcon/ECR_REGISTRY = "123456..."       (String)        │
│     ├── /popcon/DB_PASSWORD = "***암호화***"      (SecureString)  │
│     └── ...                                                      │
│                                                                  │
│  4. EC2에서 deploy.sh 실행                                       │
│     ├── aws ssm get-parameters-by-path → .env 자동 생성          │
│     ├── ECR 로그인                                               │
│     └── docker compose up                                        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

## 파일 구조

```
modules/ssm-parameters/
├── main.tf          # SSM 파라미터 리소스 정의
├── variables.tf     # 입력 변수 (project_name, parameters, secrets)
├── outputs.tf       # 출력 값 (ARN 목록)
└── README.md        # 이 파일
```

## 사용 방법

### env/dev/main.tf에서 모듈 호출

```hcl
module "ssm_parameters" {
    source       = "../../modules/ssm-parameters"
    project_name = "popcon"

    # 일반 파라미터 (평문 저장)
    parameters = {
        "ECR_REGISTRY"           = local.ecr_registry
        "SPRING_PROFILES_ACTIVE" = "prod"
        "DB_HOST"                = "popcon-mysql"
        "DB_PORT"                = "3306"
        "DB_NAME"                = "popcon"
        "JAVA_OPTS"              = "-Xms512m -Xmx512m"
        "REDIS_HOST"             = "popcon-redis"
        "REDIS_PORT"             = "6379"
    }

    # 민감 파라미터 (KMS 암호화 저장)
    secrets = {
        "DB_USERNAME"      = var.db_username
        "DB_PASSWORD"      = var.db_password
        "DB_ROOT_PASSWORD" = var.db_root_password
    }
}
```

## parameters vs secrets

| | parameters | secrets |
|--|-----------|---------|
| SSM 타입 | String (평문) | SecureString (KMS 암호화) |
| AWS 콘솔에서 | 값이 바로 보임 | 복호화 권한 필요 |
| 용도 | 호스트명, 포트 등 | 비밀번호, 키 등 |
| 비용 | 무료 | 무료 |

## SSM 경로 규칙

```
/${project_name}/${key}

예시:
/popcon/DB_HOST          → "popcon-mysql"
/popcon/DB_PASSWORD      → "***암호화***"
```

`deploy.sh`에서 `/popcon/` 경로 아래 모든 파라미터를 한 번에 가져옵니다.

## 민감 변수 전달 경로

DB 비밀번호 등 민감한 값은 코드에 직접 쓰지 않습니다.

```
GitHub Secrets (사람이 등록)
    │
    ▼
GitHub Actions 워크플로우 (TF_VAR_ 환경변수)
    │  TF_VAR_db_password: ${{ secrets.DB_PASSWORD }}
    ▼
Terraform (var.db_password로 인식)
    │
    ▼
AWS SSM Parameter Store (SecureString으로 암호화 저장)
    │
    ▼
EC2 deploy.sh (--with-decryption으로 복호화 → .env)
    │
    ▼
docker-compose (컨테이너 환경변수로 주입)
```

## GitHub Actions 워크플로우 변경사항

`terraform-plan.yml`과 `terraform-apply.yml`에 아래 env 블록이 추가되었습니다:

```yaml
env:
  TF_VAR_db_username: ${{ secrets.DB_USERNAME }}
  TF_VAR_db_password: ${{ secrets.DB_PASSWORD }}
  TF_VAR_db_root_password: ${{ secrets.DB_ROOT_PASSWORD }}
```

### GitHub Secrets 등록 필요 (PR 머지 전에)

GitHub 레포 → Settings → Secrets and variables → Actions에 아래 3개를 등록해야 합니다:

| Secret 이름 | 값 |
|-------------|-----|
| `DB_USERNAME` | DB 사용자명 (예: popcon_user) |
| `DB_PASSWORD` | DB 비밀번호 |
| `DB_ROOT_PASSWORD` | DB root 비밀번호 |

> Secrets가 등록되지 않으면 terraform plan/apply 시 빈 값으로 전달되어 실패합니다.

## ECR 레지스트리 자동 추출

ECR 레지스트리 주소를 하드코딩하지 않고 ECR 모듈의 output에서 자동으로 가져옵니다:

```hcl
# ECR 모듈 output
module.ecr.repository_url
# → "123456789.dkr.ecr.ap-northeast-2.amazonaws.com/dev-app"

# "/" 기준으로 앞부분만 추출
locals {
    ecr_registry = split("/", module.ecr.repository_url)[0]
}
# → "123456789.dkr.ecr.ap-northeast-2.amazonaws.com"
```

## 환경변수 추가/변경 시

### 일반 변수 추가 (예: 새로운 설정값)

`env/dev/main.tf`의 `parameters`에 한 줄 추가:

```hcl
parameters = {
    ...
    "NEW_CONFIG" = "new_value"    # ← 추가
}
```

### 민감 변수 추가 (예: Redis 비밀번호)

1. `env/dev/variables.tf`에 변수 선언:
```hcl
variable "redis_password" {
  type      = string
  sensitive = true
}
```

2. `env/dev/main.tf`의 `secrets`에 추가:
```hcl
secrets = {
    ...
    "REDIS_PASSWORD" = var.redis_password    # ← 추가
}
```

3. 워크플로우 yml에 env 추가:
```yaml
TF_VAR_redis_password: ${{ secrets.REDIS_PASSWORD }}
```

4. GitHub Secrets에 `REDIS_PASSWORD` 등록

5. `terraform apply` → SSM에 자동 등록
