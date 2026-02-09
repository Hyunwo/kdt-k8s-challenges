# Pop-Con Frontend Dockerfile 가이드

## 사전 작업 (필수)

Dockerfile 사용 전, `next.config.ts`에 아래 설정을 추가해주세요:

```ts
const nextConfig: NextConfig = {
  reactCompiler: true,
  output: 'standalone',  // ← 이 한 줄 추가
};
```

`output: 'standalone'`이 하는 일:
- `next build` 시 `.next/standalone` 폴더에 **실행에 필요한 최소 파일만** 추려서 경량 서버를 생성합니다
- 전체 `node_modules`(수백MB)를 이미지에 넣을 필요가 없어져 **이미지 크기가 대폭 감소**합니다
- 로컬 개발(`npm run dev`)에는 영향 없습니다

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
└─────────────────────────────────┘
```

왜 이렇게 나누나요?
- 최종 이미지에는 **빌드 도구, 소스코드, devDependencies가 포함되지 않습니다**
- 결과적으로 이미지 크기가 작고 보안상 안전합니다

---

## 각 단계 상세 설명

### Stage 1 - 의존성 설치

```dockerfile
FROM node:22-alpine AS deps
RUN apk add --no-cache libc6-compat   # Alpine 호환성 라이브러리
COPY package.json package-lock.json* ./
RUN npm ci                             # lock 파일 기반 정확한 설치
```

- `npm ci`는 `npm install`과 달리 `package-lock.json`을 정확히 따르므로 빌드 재현성이 보장됩니다
- `libc6-compat`는 Alpine Linux에서 일부 네이티브 패키지 호환을 위해 필요합니다

### Stage 2 - 빌드

```dockerfile
FROM node:22-alpine AS builder
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ARG NEXT_PUBLIC_API_URL                # 빌드 시 주입되는 환경변수
RUN npm run build
```

- `NEXT_PUBLIC_` 접두사 변수는 빌드 시점에 코드에 인라인되므로, 여기서 ARG로 받습니다

### Stage 3 - 실행

```dockerfile
FROM node:22-alpine AS runner
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs  # 보안을 위한 전용 유저
COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
USER nextjs
CMD ["node", "server.js"]
```

- root 대신 `nextjs` 유저로 실행하여 보안을 강화합니다
- `standalone` + `static` + `public` 세 가지만 복사합니다

---

## 빌드 & 실행 방법

```bash
# 기본 빌드
docker build -t popcon-frontend .

# 환경변수와 함께 빌드 (API 주소 등)
docker build --build-arg NEXT_PUBLIC_API_URL=https://api.popcon.com -t popcon-frontend .

# 실행
docker run -p 3000:3000 popcon-frontend

# 실행 확인
# 브라우저에서 http://localhost:3000 접속
```

---

## 환경변수 정리

| 변수명 | 주입 시점 | 방법 | 예시 |
|--------|----------|------|------|
| `NEXT_PUBLIC_*` | **빌드 시** | `docker build --build-arg` | `NEXT_PUBLIC_API_URL` |
| 그 외 서버 변수 | **실행 시** | `docker run -e` | `API_SECRET` |

> `NEXT_PUBLIC_` 변수는 브라우저에 노출되므로 **비밀 정보를 절대 넣지 마세요**.

---

## 주의사항

1. `.dockerignore` 파일이 함께 제공됩니다. 삭제하지 마세요 (`.env`, `node_modules` 등이 이미지에 포함되는 것을 방지)
2. `next.config.ts`에 `output: 'standalone'` 이 없으면 빌드가 실패합니다
3. 새로운 `NEXT_PUBLIC_` 환경변수를 추가하면, Dockerfile의 ARG/ENV 섹션에도 추가해주세요
