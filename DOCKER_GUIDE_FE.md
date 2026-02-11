# Pop-Con Frontend Dockerfile 가이드

## 제공 파일

| 파일 | 역할 |
|------|------|
| `Dockerfile` | Docker 이미지 빌드 레시피 |
| `.dockerignore` | 빌드 시 `.env`, `node_modules` 등 민감/불필요 파일 제외 |

> 두 파일 모두 삭제하지 마세요. Windows, Mac, Linux 어디서든 동일하게 동작합니다.

---

## 사전 작업 (필수)

Dockerfile 사용 전, `next.config.ts`에 `output: 'standalone'` 설정이 필요합니다:

```ts
const nextConfig: NextConfig = {
  reactCompiler: true,
  output: 'standalone',  // ← 이 줄이 반드시 있어야 함
};
```

이 설정이 하는 일:
- `next build` 시 `.next/standalone` 폴더에 **실행에 필요한 최소 파일만** 추려서 경량 서버를 생성합니다
- 전체 `node_modules`(수백MB)를 이미지에 넣을 필요가 없어 **이미지 크기가 대폭 감소**합니다
- 로컬 개발(`npm run dev`)에는 영향 없습니다
- **이 설정이 없으면 Docker 빌드가 실패합니다**

---

## Dockerfile 구조 설명

이 Dockerfile은 **3단계(Multi-stage)** 로 나뉘어 있습니다:

```
┌─────────────────────────────────┐
│  Stage 1: deps                  │
│  - npm ci로 의존성 설치          │
│  - node_modules 생성            │
└──────────────┬──────────────────┘
               │ node_modules 전달
┌──────────────▼──────────────────┐
│  Stage 2: builder               │
│  - 소스코드 복사                 │
│  - npm run build 실행           │
│  - .next/standalone 생성        │
└──────────────┬──────────────────┘
               │ 빌드 결과물만 복사
┌──────────────▼──────────────────┐
│  Stage 3: runner (최종 이미지)   │
│  - standalone 서버 + static만   │
│  - node server.js로 실행        │
│  - 최종 이미지 크기: ~330MB      │
└─────────────────────────────────┘
```

왜 이렇게 나누나요?
- 최종 이미지에는 **빌드 도구, 소스코드, devDependencies가 포함되지 않습니다**
- 결과적으로 이미지 크기가 작고 보안상 안전합니다

---

## 각 단계 상세 설명

### Stage 1 - 의존성 설치 (deps)

```dockerfile
FROM node:22-alpine AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci
```

- `node:22-alpine`: Node.js 22 LTS + Alpine Linux (경량 이미지)
- `libc6-compat`: Alpine은 musl을 사용하는데, 일부 npm 패키지가 glibc에 의존하므로 호환 라이브러리 필요
- `npm ci`: `package-lock.json`을 정확히 따라 설치 (빌드 재현성 보장, `npm install`보다 빠름)
- **패키지가 추가/삭제되어도 이 단계가 자동으로 처리합니다 (Dockerfile 수정 불필요)**

### Stage 2 - 빌드 (builder)

```dockerfile
FROM node:22-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ARG NEXT_PUBLIC_API_URL
ENV NEXT_PUBLIC_API_URL=$NEXT_PUBLIC_API_URL
RUN npm run build
```

- deps 단계에서 만든 `node_modules`를 가져와서 빌드
- `NEXT_PUBLIC_API_URL`: 빌드 시 백엔드 API 주소를 코드에 인라인 (아래 환경변수 섹션 참고)

### Stage 3 - 실행 (runner)

```dockerfile
FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
USER nextjs
EXPOSE 3000
CMD ["node", "server.js"]
```

- `nextjs` 전용 유저(UID 1001)로 실행하여 보안 강화 (root 실행 방지)
- `standalone` + `static` + `public` 세 폴더만 복사 (최소한의 파일)
- `--chown`: 복사 시 파일 소유권을 `nextjs` 유저로 지정

---

## 빌드 & 실행 방법

```bash
# 이미지 빌드
docker build -t popcon-frontend .

# 환경변수와 함께 빌드 (백엔드 API 주소 주입)
docker build --build-arg NEXT_PUBLIC_API_URL=https://api.popcon.com -t popcon-frontend .

# 컨테이너 실행
docker run -p 3000:3000 popcon-frontend

# 백그라운드 실행
docker run -d -p 3000:3000 popcon-frontend

# 실행 확인 - 브라우저에서 http://localhost:3000 접속

# 컨테이너 종료
docker stop popcon-frontend
```

---

## 환경변수

### 빌드 시 주입 (NEXT_PUBLIC_*)

| 변수명 | 용도 | 주입 방법 |
|--------|------|----------|
| `NEXT_PUBLIC_API_URL` | 백엔드 API 주소 | `docker build --build-arg NEXT_PUBLIC_API_URL=값` |

`NEXT_PUBLIC_` 접두사 변수는 빌드 시 JavaScript에 인라인됩니다.
새로운 `NEXT_PUBLIC_` 변수가 필요하면 Dockerfile의 builder 단계에 아래를 추가해주세요:

```dockerfile
ARG NEXT_PUBLIC_새변수명
ENV NEXT_PUBLIC_새변수명=$NEXT_PUBLIC_새변수명
```

### 실행 시 주입 (서버 전용)

| 변수명 | 용도 | 주입 방법 |
|--------|------|----------|
| 서버 전용 변수 | API 시크릿 등 | `docker run -e API_SECRET=값` |

> `NEXT_PUBLIC_` 변수는 브라우저에 노출되므로 **비밀 정보를 절대 넣지 마세요**.

---

## Dockerfile 수정이 필요한 경우 vs 불필요한 경우

| 상황 | Dockerfile 수정 |
|------|----------------|
| 새 npm 패키지 추가/삭제 | **불필요** (`npm ci`가 자동 처리) |
| 소스코드 수정 | **불필요** |
| 새 `NEXT_PUBLIC_` 환경변수 추가 | **필요** (ARG/ENV 추가) |
| Node.js 버전 변경 | **필요** (FROM 이미지 변경) |
| 포트 변경 | **필요** (EXPOSE, ENV PORT 변경) |

---

## 빌드 실패 시 체크리스트

| 증상 | 원인 | 해결 |
|------|------|------|
| `.next/standalone: not found` | standalone 설정 누락 | `next.config.ts`에 `output: 'standalone'` 추가 |
| `npm ci` 실패 | lock 파일 불일치 | `npm install` 후 `package-lock.json` 커밋 |
| `NEXT_PUBLIC_` 값이 undefined | 빌드 시 ARG 미전달 | `--build-arg` 로 값 전달 |
