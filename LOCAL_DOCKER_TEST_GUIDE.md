# Pop-Con 로컬 Docker 빌드 테스트 가이드

## 사전 준비

- Docker Desktop 설치 및 실행 ([다운로드](https://www.docker.com/products/docker-desktop))
- Docker Desktop이 실행 중인지 확인:

```bash
docker --version
```

---

## 백엔드 빌드 테스트

### 1. 이미지 빌드

```bash
cd pop-con-backend
docker build -t popcon-backend .
```

빌드 과정:
```
[build 1/6] JDK 21 이미지 다운로드
[build 2/6] Gradle 파일 복사
[build 3/6] 의존성 다운로드         ← 최초 빌드 시 오래 걸림 (이후 캐시됨)
[build 4/6] 소스코드 복사
[build 5/6] bootJar 빌드
[stage-1]   JRE 이미지로 JAR 복사   ← 최종 이미지
```

### 2. 컨테이너 실행

```bash
# 기본 실행 (H2 인메모리 DB, dev 프로필)
docker run -p 8080:8080 -e SPRING_PROFILES_ACTIVE=default popcon-backend
```

### 3. 동작 확인

브라우저 또는 터미널에서:

```bash
curl http://localhost:8080/health
```

응답:
```
Pop-Con Server is running
```

### 4. 종료

`Ctrl + C` 로 종료하거나:

```bash
docker stop popcon-backend
```

---

## 프론트엔드 빌드 테스트

### 사전 작업 (필수)

`next.config.ts`에 `output: 'standalone'`이 있는지 확인:

```ts
const nextConfig: NextConfig = {
  reactCompiler: true,
  output: 'standalone',  // ← 이 줄이 반드시 있어야 함
};
```

### 1. 이미지 빌드

```bash
cd pop-con-frontend
docker build -t popcon-frontend .
```

빌드 과정:
```
[deps 1/3]    Node 22 이미지 다운로드
[deps 2/3]    package.json 복사
[deps 3/3]    npm ci 실행             ← 최초 빌드 시 오래 걸림 (이후 캐시됨)
[builder 1/2] 소스코드 복사
[builder 2/2] next build 실행
[runner]      standalone 결과물 복사   ← 최종 이미지
```

### 2. 컨테이너 실행

```bash
docker run -p 3000:3000 popcon-frontend
```

### 3. 동작 확인

브라우저에서 접속:

```
http://localhost:3000
```

Next.js 기본 페이지가 보이면 성공입니다.

### 4. 종료

`Ctrl + C` 로 종료하거나:

```bash
docker stop popcon-frontend
```

---

## 빌드 결과 확인

```bash
# 이미지 목록 확인
docker images | findstr popcon
```

정상 결과 예시:
```
popcon-backend    latest    516MB
popcon-frontend   latest    330MB
```

---

## 빌드 실패 시 체크리스트

### 백엔드

| 증상 | 원인 | 해결 |
|------|------|------|
| `gradlew: Permission denied` | 실행 권한 없음 | Dockerfile에 `chmod +x gradlew` 있는지 확인 |
| `bootJar FAILED` | 컴파일 에러 | `./gradlew bootJar -x test`로 로컬에서 먼저 확인 |

### 프론트엔드

| 증상 | 원인 | 해결 |
|------|------|------|
| `.next/standalone: not found` | standalone 설정 누락 | `next.config.ts`에 `output: 'standalone'` 추가 |
| `npm ci` 실패 | lock 파일 불일치 | `package-lock.json` 최신화 (`npm install` 후 커밋) |

---

## 테스트 후 정리

```bash
# 컨테이너 전부 정지
docker stop popcon-backend popcon-frontend

# 이미지 삭제 (디스크 공간 확보)
docker rmi popcon-backend popcon-frontend

# 빌드 캐시 정리 (선택)
docker builder prune
```
