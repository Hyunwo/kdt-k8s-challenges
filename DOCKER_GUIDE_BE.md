# Pop-Con Backend Dockerfile 가이드

## 제공 파일

| 파일 | 역할 |
|------|------|
| `Dockerfile` | Docker 이미지 빌드 레시피 |
| `.dockerignore` | 빌드 시 `.pem`, `.key` 등 민감/불필요 파일 제외 |

> 두 파일 모두 삭제하지 마세요. Windows, Mac, Linux 어디서든 동일하게 동작합니다.

---

## 기술 스택

| 항목 | 버전/이름 |
|------|----------|
| Language | Java 21 (LTS) |
| Framework | Spring Boot 4.0.2 |
| Build Tool | Gradle 9.3.0 |
| ORM | Spring Data JPA |
| DB (개발) | H2 (인메모리) |
| DB (운영) | MySQL 8.0 |

---

## Dockerfile 구조 설명

이 Dockerfile은 **2단계(Multi-stage)** 로 나뉘어 있습니다:

```
┌───────────────────────────────────────┐
│  Stage 1: build                       │
│  - JDK 21 환경                        │
│  - Gradle 의존성 다운로드 (캐시됨)     │
│  - bootJar로 실행 가능한 JAR 생성      │
└──────────────┬────────────────────────┘
               │ JAR 파일만 복사
┌──────────────▼────────────────────────┐
│  Stage 2: runtime (최종 이미지)        │
│  - JRE 21 환경 (JDK보다 경량)         │
│  - app.jar 실행                       │
│  - 최종 이미지 크기: ~516MB            │
└───────────────────────────────────────┘
```

왜 이렇게 나누나요?
- 빌드에는 JDK + Gradle + 소스코드가 필요하지만, 실행에는 **JRE + JAR 파일만** 있으면 됩니다
- 최종 이미지에 빌드 도구와 소스코드가 포함되지 않아 **이미지 크기 감소 + 보안 강화**

---

## 각 단계 상세 설명

### Stage 1 - 빌드 (build)

```dockerfile
FROM eclipse-temurin:21-jdk-jammy AS build
WORKDIR /app
```

- `eclipse-temurin:21-jdk-jammy`: Java 21 JDK + Ubuntu 22.04(Jammy) 기반 이미지
- `-jammy`를 명시하여 OS 버전을 고정 (빌드 재현성 보장)

```dockerfile
COPY gradlew .
COPY gradle gradle
COPY build.gradle .
COPY settings.gradle .
RUN chmod +x gradlew && ./gradlew dependencies --no-daemon
```

- **Gradle 래퍼(wrapper)**: Gradle을 별도 설치하지 않아도 빌드할 수 있게 해주는 프로젝트 내장 도구입니다
  - `gradlew`: Gradle 실행 스크립트
  - `gradle/`: Gradle 버전 정보가 담긴 wrapper 파일
  - `build.gradle`: 의존성, 플러그인 정의
  - `settings.gradle`: 프로젝트 이름 설정
- **캐싱을 위해 먼저 복사**: 이 파일들을 소스코드보다 먼저 복사하고 의존성을 다운로드합니다. 소스코드(`src/`)가 변경되어도 `build.gradle`이 안 바뀌면 이 레이어는 캐시되어 **의존성을 다시 받지 않습니다** (빌드 시간 대폭 단축)
- `--no-daemon`: 컨테이너 환경에서는 Gradle 데몬이 불필요하므로 비활성화

```dockerfile
COPY src src
RUN ./gradlew bootJar -x test --no-daemon
```

- 소스코드를 복사하고 실행 가능한 JAR 파일을 생성합니다
- `-x test`: Docker 빌드 시 테스트를 건너뜁니다 (CI/CD에서 별도로 수행)

### Stage 2 - 실행 (runtime)

```dockerfile
FROM eclipse-temurin:21-jre-jammy
WORKDIR /app
```

- JDK가 아닌 **JRE** 이미지 사용 (컴파일 도구 없이 실행만 가능, 이미지 크기 감소)

```dockerfile
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
COPY --from=build --chown=appuser:appgroup /app/build/libs/*.jar app.jar
USER appuser
```

- `appuser` 전용 유저로 실행하여 보안 강화 (root 실행 방지)
- `--chown`: 복사 시 파일 소유권을 `appuser`로 지정

```dockerfile
ENV SPRING_PROFILES_ACTIVE=prod
ENV JAVA_OPTS="-Xms512m -Xmx512m"
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

- `SPRING_PROFILES_ACTIVE=prod`: `application-prod.yml` 설정을 자동 적용합니다
- `JAVA_OPTS`: JVM 메모리 설정 (아래 환경변수 섹션에서 상세 설명)
- `sh -c`를 사용하여 환경변수(`$JAVA_OPTS`)가 실행 시점에 치환됩니다

---

## 사전 준비

- Docker Desktop 설치 및 실행 ([다운로드](https://www.docker.com/products/docker-desktop))
- Docker Desktop이 실행 중인지 확인 (트레이에 고래 아이콘):

```bash
docker --version
# Docker version 28.x.x 같은 출력이 나오면 정상
```

---

## 백엔드팀이 해야 할 것

Dockerfile이 정상적으로 빌드되는지 확인하는 것이 목적입니다.
**실제 배포는 인프라(클라우드)팀에서 진행합니다.**

```bash
# 1. 이미지 빌드
docker build -t popcon-backend .

# 2. 컨테이너 실행 (H2 인메모리 DB로 테스트)
docker run --name popcon-backend -p 8080:8080 -e SPRING_PROFILES_ACTIVE=default popcon-backend

# 3. 헬스체크 확인 (새 터미널에서)
curl http://localhost:8080/health
# "Pop-Con Server is running" 응답이 오면 성공

# 4. 종료 (Ctrl+C 또는 아래 명령어)
docker stop popcon-backend

# 5. 컨테이너 정리
docker rm popcon-backend
```

> MySQL/Redis 연결 없이 H2 인메모리 DB로 테스트합니다.
> DB 연결 등 실제 배포 환경 설정은 인프라팀에서 처리합니다.

---

## 실행 예시 (상황별 - 하나만 선택)

아래는 **상황별 예시**입니다. 동시에 실행하는 것이 아니라 **하나만 골라서** 실행합니다.
(같은 포트를 사용하므로 동시 실행 시 포트 충돌이 발생합니다)

```bash
# [상황 A] 빌드 테스트 - DB 없이 H2로 실행
docker run --name popcon-backend -p 8080:8080 -e SPRING_PROFILES_ACTIVE=default popcon-backend

# [상황 B] MySQL 연결 - application-prod.yml이 있을 때
docker run --name popcon-backend -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=prod \
  -e DB_HOST=your-db-host \
  -e DB_NAME=popcon \
  -e DB_USERNAME=your-username \
  -e DB_PASSWORD=your-password \
  popcon-backend

# [상황 C] JVM 메모리 조정
docker run --name popcon-backend -p 8080:8080 \
  -e JAVA_OPTS="-Xms256m -Xmx1024m" \
  popcon-backend
```

> 다른 상황으로 재실행하려면 먼저 기존 컨테이너를 정리해야 합니다:
> `docker stop popcon-backend && docker rm popcon-backend`

---

## 환경변수

### Spring 관련

| 변수명 | 용도 | 기본값 |
|--------|------|--------|
| `SPRING_PROFILES_ACTIVE` | 활성 프로필 | `prod` |

### JVM 관련

| 변수명 | 용도 | 기본값 |
|--------|------|--------|
| `JAVA_OPTS` | JVM 메모리 등 | `-Xms512m -Xmx512m` |

`-Xms512m -Xmx512m`의 의미:
- `-Xms512m`: JVM 시작 시 메모리 512MB 확보
- `-Xmx512m`: JVM 최대 메모리 512MB 제한
- 둘을 동일하게 설정하면 메모리를 미리 확보하여 컨테이너 환경에서 안정적으로 동작합니다
- 서버 스펙에 따라 `docker run -e JAVA_OPTS="-Xms1g -Xmx1g"`으로 조정 가능

### DB 관련

DB 접속 정보는 `application-prod.yml`에서 `${환경변수명}` 형태로 매핑합니다.
`application-prod.yml`은 백엔드팀에서 직접 작성/관리합니다.
인프라팀에서 운영 환경에 환경변수를 주입합니다.

---

## Dockerfile 수정이 필요한 경우 vs 불필요한 경우

| 상황 | Dockerfile 수정 |
|------|----------------|
| 새 Gradle 의존성 추가/삭제 | **불필요** (`gradlew dependencies`가 자동 처리) |
| 소스코드 수정 | **불필요** |
| Java 소스 파일 추가 | **불필요** (`bootJar`가 자동 컴파일) |
| Java 버전 변경 | **필요** (FROM 이미지 변경) |
| 포트 변경 | **필요** (EXPOSE 변경) |
| 시스템 패키지 필요 시 | **필요** (RUN apt-get 추가) |

---

## 빌드/실행 실패 시 체크리스트

| 증상 | 원인 | 해결 |
|------|------|------|
| `docker: command not found` | Docker 미설치 | Docker Desktop 설치 및 실행 |
| `Cannot connect to the Docker daemon` | Docker Desktop 미실행 | Docker Desktop 실행 (트레이 고래 아이콘 확인) |
| `gradlew: Permission denied` | 실행 권한 없음 | Dockerfile에 `chmod +x gradlew` 있는지 확인 |
| `bootJar FAILED` | 컴파일 에러 | `./gradlew bootJar -x test`로 로컬에서 먼저 확인 |
| `port is already allocated` | 8080 포트 사용 중 | 아래 정리 명령어 실행 후 재시도 |

**포트 충돌 해결:**
```bash
# 실행 중인 컨테이너 확인
docker ps

# 해당 컨테이너 정지 및 삭제
docker stop popcon-backend && docker rm popcon-backend

# 다시 실행
docker run --name popcon-backend -p 8080:8080 -e SPRING_PROFILES_ACTIVE=default popcon-backend
```

---

## 자주 쓰는 명령어

```bash
# 이미지 빌드
docker build -t popcon-backend .

# 실행
docker run --name popcon-backend -p 8080:8080 -e SPRING_PROFILES_ACTIVE=default popcon-backend

# 백그라운드 실행
docker run -d --name popcon-backend -p 8080:8080 -e SPRING_PROFILES_ACTIVE=default popcon-backend

# 컨테이너 상태 확인
docker ps

# 로그 확인
docker logs popcon-backend

# 종료
docker stop popcon-backend

# 컨테이너 삭제 (종료 후)
docker rm popcon-backend

# 이미지 삭제
docker rmi popcon-backend
```

---

## 주의사항

1. `.dockerignore` 파일을 삭제하지 마세요 (`.pem`, `.key` 등 민감 파일이 이미지에 포함되는 것을 방지)
2. `application-prod.yml`은 백엔드팀에서 직접 작성/관리합니다
3. Docker 빌드 시 테스트는 건너뛰므로 (`-x test`), CI/CD 파이프라인에서 별도로 테스트를 수행해주세요
