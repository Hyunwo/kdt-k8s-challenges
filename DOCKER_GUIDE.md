# Pop-Con Backend Dockerfile 가이드

## Dockerfile 구조 설명

이 Dockerfile은 **2단계(Multi-stage)** 로 나뉘어 있습니다:

```
┌─────────────────────────────────────┐
│  Stage 1: build                     │
│  - JDK 21 환경                      │
│  - Gradle 의존성 다운로드            │
│  - bootJar로 실행 가능한 JAR 생성    │
└──────────────┬──────────────────────┘
               │ JAR 파일만 복사
┌──────────────▼──────────────────────┐
│  Stage 2: runtime (최종 이미지)      │
│  - JRE 21 환경 (JDK보다 경량)       │
│  - app.jar 실행                     │
└─────────────────────────────────────┘
```

왜 이렇게 나누나요?
- 빌드에는 JDK + Gradle + 소스코드가 필요하지만, 실행에는 **JRE + JAR 파일만** 있으면 됩니다
- 최종 이미지에 빌드 도구와 소스코드가 포함되지 않아 **이미지 크기 감소 + 보안 강화**

---

## 각 단계 상세 설명

### Stage 1 - 빌드

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
- Gradle 관련 파일만 먼저 복사하고 의존성을 다운로드합니다
- **왜?** 소스코드(`src/`)가 변경되어도 이 레이어는 캐시되어 의존성을 다시 받지 않습니다
- 빌드 시간이 크게 단축됩니다

```dockerfile
COPY src src
RUN ./gradlew bootJar -x test --no-daemon
```
- 소스코드를 복사하고 실행 가능한 JAR 파일을 생성합니다
- `-x test`: Docker 빌드 시 테스트를 건너뜁니다 (CI/CD에서 별도로 수행)
- `--no-daemon`: 컨테이너 환경에서는 Gradle 데몬이 불필요하므로 비활성화

### Stage 2 - 실행

```dockerfile
FROM eclipse-temurin:21-jre-jammy
```
- JDK가 아닌 **JRE** 이미지 사용 (컴파일 도구 없이 실행만 가능, 이미지 크기 감소)

```dockerfile
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
COPY --from=build --chown=appuser:appgroup /app/build/libs/*.jar app.jar
USER appuser
```
- root가 아닌 전용 유저(`appuser`)로 실행하여 보안을 강화합니다
- `--chown`으로 복사 시 소유권을 바로 지정합니다

```dockerfile
ENV SPRING_PROFILES_ACTIVE=prod
ENV JAVA_OPTS="-Xms512m -Xmx512m"
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```
- `SPRING_PROFILES_ACTIVE=prod`: `application-prod.yml` 설정을 자동 적용합니다
- `JAVA_OPTS`: JVM 메모리 설정을 외부에서 변경할 수 있습니다
- `sh -c`를 사용하여 환경변수(`$JAVA_OPTS`)가 실행 시점에 치환됩니다

---

## 빌드 & 실행 방법

```bash
# 빌드
docker build -t popcon-backend .

# 실행 (기본 설정)
docker run -p 8080:8080 popcon-backend

# 환경변수와 함께 실행 (DB 접속 정보 등)
docker run -p 8080:8080 \
  -e DB_HOST=your-db-host \
  -e DB_NAME=popcon \
  -e DB_USERNAME=your-username \
  -e DB_PASSWORD=your-password \
  popcon-backend

# JVM 메모리 조정
docker run -p 8080:8080 \
  -e JAVA_OPTS="-Xms256m -Xmx1024m" \
  popcon-backend

# 헬스체크 확인
curl http://localhost:8080/health
```

---

## 환경변수 정리

### Spring 관련

| 변수명 | 용도 | 기본값 |
|--------|------|--------|
| `SPRING_PROFILES_ACTIVE` | 활성 프로필 | `prod` |

### JVM 관련

| 변수명 | 용도 | 기본값 |
|--------|------|--------|
| `JAVA_OPTS` | JVM 옵션 (메모리 등) | `-Xms512m -Xmx512m` |

### DB 관련 (application-prod.yml에서 정의)

DB 접속 정보 등은 `application-prod.yml`에서 `${환경변수명}` 형태로 매핑합니다.
인프라팀에서 운영 환경(ECS Task Definition 등)에 환경변수를 주입합니다.

---

## 주의사항

1. `.dockerignore` 파일이 함께 제공됩니다. 삭제하지 마세요 (`.pem`, `.key` 등 민감 파일이 이미지에 포함되는 것을 방지)
2. `application-prod.yml`은 백엔드팀에서 직접 작성/관리합니다
3. Docker 빌드 시 테스트는 건너뛰므로 (`-x test`), CI/CD 파이프라인에서 별도로 테스트를 수행해주세요
