# Next.js를 버리고 순수 React로 돌아온 이유: 실무 관점의 프레임워크 선택 여정

> **작성일**: 2025년 10월 27일
> **키워드**: Next.js, React Router, 프론트엔드 아키텍처, Docker 배포, 환경 변수 관리, K8s

---

## 들어가며

[**imprun.dev**](https://imprun.dev)는 Kubernetes 기반 서버리스 Cloud Function 플랫폼입니다. ARM64 클러스터(노드당 4 cores, 24GB)에서 운영하면서, **Next.js App Router의 리소스 사용량이 심각한 병목**이 되었습니다.

**우리가 겪은 문제**:
- ❌ **Next.js 런타임 메모리**: 768MB (nginx + React는 30MB)
- ❌ **빌드 시간**: 5분 (Vite는 1분)
- ❌ **환경 변수**: 빌드 타임에 고정되어 환경별 이미지 필요
- ✅ React Router DOM으로 전환 후 모든 문제 해결

**핵심 발견**:
- Container/Presentational + Layered Architecture 덕분에 **프레임워크 변경 시 95% 코드 재사용**
- Next.js → React Router 마이그레이션 **3일 완료** (3회 프레임워크 변경 경험)
- **리소스 제약 환경**에서는 순수 React + nginx가 압도적으로 유리
- SSR/SSG 불필요한 Private SaaS라면 프레임워크 오버헤드 제거가 정답

이 글은 **imprun.dev 플랫폼 구축 경험**을 바탕으로, 리소스 제약 환경에서 Next.js의 한계와 순수 React 스택으로 회귀한 이유를 공유합니다.

---

## 개요

프론트엔드 개발에서 프레임워크 선택은 단순히 개발 편의성만의 문제가 아닙니다. 배포 환경, 운영 비용, 유지보수성까지 고려해야 하는 **아키텍처 결정**입니다.

이 글은 제가 **React 19 + TanStack Router → Next.js App Router → React Router DOM**으로 이어진 프레임워크 마이그레이션 여정과, 그 과정에서 얻은 교훈을 공유합니다.

특히 **열악한 리소스 환경의 K8s 클러스터**에서 운영해야 하는 상황에서 Next.js의 한계를 경험했고, 최종적으로 순수 React 스택으로 회귀한 이유를 상세히 설명합니다.

---

## 1차 시도: React 19 + TanStack Router

### 선택 이유

프로젝트 초기에는 최신 기술 스택을 경험하고 싶었습니다:

- **React 19**: 최신 기능(Server Components, Actions 등) 활용
- **TanStack Router**: 타입 안전한 라우팅, 코드 스플리팅 자동화

```typescript
// TanStack Router의 타입 안전한 라우팅 예시
const route = createRoute({
  getParentRoute: () => rootRoute,
  path: '/applications/$gatewayId',
  component: ApplicationDetail,
})

// 타입 추론이 완벽하게 작동
const { gatewayId } = route.useParams() // gatewayId: string
```

**장점**:
- 강력한 타입 안전성
- 파일 기반이 아닌 명시적 라우트 정의
- 뛰어난 번들 사이즈 최적화

### 문제점: 파일 라우팅 미지원

하지만 치명적인 문제가 있었습니다:

```
src/routes/
  applications/
    $gatewayId/
      index.tsx          ❌ 자동 라우팅 없음
      functions/
        $name/
          edit.tsx       ❌ 수동으로 모두 등록 필요
```

**파일 기반 라우팅이 아니라 명시적 선언이 필요**했습니다:

```typescript
// 모든 라우트를 일일이 createRoute로 선언해야 함
const applicationsRoute = createRoute({ ... })
const appDetailRoute = createRoute({ ... })
const functionsRoute = createRoute({ ... })
const functionEditRoute = createRoute({ ... })

// 수동으로 라우트 트리 구성
const routeTree = rootRoute.addChildren([
  applicationsRoute.addChildren([
    appDetailRoute.addChildren([
      functionsRoute.addChildren([
        functionEditRoute
      ])
    ])
  ])
])
```

**결론**: 파일 라우팅에 익숙한 개발자에게는 생산성 저하가 컸습니다.

---

## 2차 시도: Next.js App Router로 대규모 리팩토링

### 선택 이유

파일 라우팅에 대한 갈증으로 Next.js로 전환을 결정했습니다:

```
app/
  applications/
    [gatewayId]/
      page.tsx                    ✅ 자동 라우팅
      functions/
        [name]/
          edit/
            page.tsx              ✅ 파일 = 라우트
          layout.tsx              ✅ 레이아웃 공유
```

**기대한 장점**:
- ✅ 직관적인 파일 기반 라우팅
- ✅ Layout, Loading, Error 등 특수 파일 지원
- ✅ 빠른 개발 속도

**실제로 좋았던 점**:
- 개발 단계에서는 생산성이 매우 높았음
- Hot Module Replacement가 잘 작동
- Server Components 활용 가능

### 문제는 배포 단계에서 시작되었습니다

---

## Next.js의 치명적인 한계들

### 문제 1: Docker 배포 시 Node.js 서버 필수

Next.js는 기본적으로 **Node.js 런타임이 필요한 서버 API 게이트웨이**입니다.

#### Dockerfile 구조

```dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY . .
RUN pnpm install && pnpm build

# Stage 2: Production
FROM node:20-alpine AS runner
WORKDIR /app

# Standalone 모드 사용
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public

EXPOSE 3000
CMD ["node", "server.js"]  # ← Node.js 서버 실행 필수
```

**문제점**:
- 정적 파일 서빙에도 Node.js 프로세스 필요
- 메모리 사용량: 최소 128MB (실제로는 256MB+ 권장)
- CPU 사용량: 지속적인 프로세스 실행

#### nginx 정적 파일 서빙과 비교

```dockerfile
# 순수 React + nginx
FROM nginx:alpine
COPY dist/ /usr/share/nginx/html/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**nginx의 장점**:
- 메모리 사용량: 10MB 미만
- 정적 파일 서빙 성능이 Node.js 대비 **10배 이상 빠름**
- gzip, cache 등 최적화 내장

**실제 리소스 비교**:

| 항목 | Next.js (Node) | React + nginx |
|------|----------------|---------------|
| 메모리 | 256MB | 10MB |
| CPU (idle) | 0.1 core | 0.001 core |
| 응답 속도 (정적) | ~50ms | ~5ms |

**결론**: 열악한 서버 환경에서 Next.js는 **리소스 낭비**입니다.

---

### 문제 2: NEXT_PUBLIC_ 환경 변수의 함정

Next.js의 환경 변수는 **빌드 타임에 결정**됩니다.

#### 문제 상황

```typescript
// frontend/src/lib/httpclient.ts
const API_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:3000'

// 이 값은 빌드할 때 결정됨!
// 즉, 운영과 개발 환경을 다르게 하려면 각각 빌드해야 함
```

#### 개발 환경

```bash
# .env.development
NEXT_PUBLIC_API_URL=http://localhost:3000
```

#### 운영 환경

```bash
# .env.production
NEXT_PUBLIC_API_URL=https://api.imprun.dev
```

**문제점**:
```bash
# 개발용 빌드
NEXT_PUBLIC_API_URL=http://localhost:3000 pnpm build
docker build -t imprun-console:dev .

# 운영용 빌드 (다시 빌드 필요!)
NEXT_PUBLIC_API_URL=https://api.imprun.dev pnpm build
docker build -t imprun-console:prod .

# 테스트 환경도 추가된다면?
NEXT_PUBLIC_API_URL=https://api.test.imprun.dev pnpm build
docker build -t imprun-console:test .
```

**환경마다 별도 이미지를 빌드해야 합니다!**

---

### 문제 3: K8s에서의 오버엔지니어링

#### 시도 1: ConfigMap으로 환경 변수 관리 (실패)

```yaml
# k8s/values.yaml
console:
  env:
    - name: NEXT_PUBLIC_API_URL
      value: "https://api.imprun.dev"  # ❌ 작동하지 않음!
```

**이유**: `NEXT_PUBLIC_*` 변수는 **빌드 타임**에 번들에 포함됩니다.
런타임에 환경 변수를 주입해도 **이미 빌드된 JS 파일은 변경되지 않습니다**.

#### 시도 2: init Container로 config.js 생성 (오버엔지니어링)

```yaml
# 운영 환경 배포 시
apiVersion: v1
kind: ConfigMap
metadata:
  name: console-config
data:
  config.js: |
    window.__ENV__ = {
      API_URL: "https://api.imprun.dev"
    };
---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      initContainers:
      - name: inject-config
        image: busybox
        command:
          - sh
          - -c
          - |
            cp /config/config.js /app/public/config.js
        volumeMounts:
        - name: config
          mountPath: /config
        - name: app
          mountPath: /app

      containers:
      - name: console
        image: imprun-console:latest
        volumeMounts:
        - name: app
          mountPath: /app/public

      volumes:
      - name: config
        configMap:
          name: console-config
      - name: app
        emptyDir: {}
```

```html
<!-- public/index.html -->
<script src="/config.js"></script>
<script src="/bundle.js"></script>
```

```typescript
// src/lib/httpclient.ts
const API_URL = (window as any).__ENV__?.API_URL || 'http://localhost:3000'
```

**문제점**:
- 복잡한 init Container 설정
- 타이밍 이슈 (config.js 로드 순서)
- 타입 안전성 상실
- 유지보수 부담 증가

**이것은 명백한 오버엔지니어링입니다.**

---

### 문제 4: 추가 서버 운영의 부담

제 서버 환경은 다음과 같습니다:

```
K8s 클러스터:
- 노드 3대 (ARM64)
- 노드당 리소스: 4 cores, 24GB RAM
- 운영 중인 서비스:
  * API 서버 (NestJS) x3 replicas: 1.5GB
  * MongoDB ReplicaSet x3: 3GB
  * Redis: 512MB
  * Admin 서버 (NestJS) x2 replicas: 1GB
  * 기타 시스템 (Ingress, cert-manager 등): 1GB
```

**남은 리소스**: 약 2GB

여기에 **Next.js 콘솔을 3개 레플리카**로 띄우려면:
- 256MB x 3 = **768MB** 추가 필요
- 실제로는 버퍼를 고려하면 **1GB+ 필요**

반면 **nginx + 순수 React**:
- 10MB x 3 = **30MB**만 필요
- 리소스 절약: **~97%**

**결론**: 제한된 리소스 환경에서 Next.js는 사치입니다.

---

## 최종 결정: React Router DOM으로 회귀

### 마이그레이션이 쉬웠던 이유

처음부터 **Container/Presentational Pattern + Layered Architecture**로 컴포넌트를 설계했기 때문에, 프레임워크 전환이 놀라울 정도로 수월했습니다. 데이터 페칭 로직(Hooks)과 렌더링 컴포넌트를 철저히 분리했고, `Component → Hook → Service → HTTP Client` 레이어를 명확히 구분했기 때문에 라우터만 교체하면 대부분의 코드를 재사용할 수 있었습니다.

### 아키텍처 결정

다시 순수 React 스택으로 돌아왔습니다:

```
프론트엔드 스택:
├── React 19
├── react-router-dom v6
├── Zustand (전역 상태)
├── TanStack Query v5 (서버 상태)
├── Vite (빌드 도구)
└── nginx (정적 파일 서빙)

백엔드 스택:
└── NestJS (완전 정착)
```

### 환경 변수 관리: window.__ENV__ 패턴

#### 1. Vite 설정

```typescript
// vite.config.ts
export default defineConfig({
  define: {
    __APP_ENV__: JSON.stringify(process.env.NODE_ENV),
  },
})
```

#### 2. index.html에 환경 변수 주입 스크립트

```html
<!-- public/index.html -->
<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8" />
  <title>imprun.dev Console</title>
</head>
<body>
  <div id="root"></div>

  <!-- 환경 변수 주입 지점 -->
  <script>
    // nginx에서 이 부분을 런타임에 치환합니다
    window.__ENV__ = {
      API_URL: '${API_URL}',
      WS_URL: '${WS_URL}'
    };
  </script>

  <script type="module" src="/src/main.tsx"></script>
</body>
</html>
```

#### 3. nginx 설정으로 런타임 변수 주입

```nginx
# k8s/charts/console/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: console-nginx-config
data:
  default.conf: |
    server {
      listen 80;
      root /usr/share/nginx/html;

      location / {
        try_files $uri $uri/ /index.html;
      }

      # index.html은 캐시하지 않음 (환경 변수 주입 때문)
      location = /index.html {
        add_header Cache-Control "no-store, no-cache, must-revalidate";

        # 환경 변수 치환
        sub_filter '${API_URL}' '${API_URL}';
        sub_filter '${WS_URL}' '${WS_URL}';
        sub_filter_once off;
      }

      # JS/CSS는 적극적으로 캐시
      location ~* \.(?:css|js|jpg|jpeg|gif|png|ico|svg|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
      }
    }
```

#### 4. K8s Deployment에 환경 변수 전달

```yaml
# k8s/charts/console/values.yaml
console:
  image:
    repository: imprun-console
    tag: latest

  env:
    API_URL: "https://api.imprun.dev"
    WS_URL: "wss://api.imprun.dev"
---
# k8s/charts/console/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: console
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: console
        image: {{ .Values.console.image.repository }}:{{ .Values.console.image.tag }}
        env:
        - name: API_URL
          value: {{ .Values.console.env.API_URL | quote }}
        - name: WS_URL
          value: {{ .Values.console.env.WS_URL | quote }}

        # nginx가 환경 변수를 사용할 수 있도록
        command: ["/bin/sh"]
        args:
        - -c
        - |
          envsubst '${API_URL} ${WS_URL}' < /usr/share/nginx/html/index.html > /tmp/index.html
          mv /tmp/index.html /usr/share/nginx/html/index.html
          nginx -g 'daemon off;'
```

#### 5. API 게이트웨이 코드에서 사용

```typescript
// src/lib/env.ts
interface Env {
  API_URL: string;
  WS_URL: string;
}

declare global {
  interface Window {
    __ENV__: Env;
  }
}

export const env: Env = {
  API_URL: window.__ENV__?.API_URL || import.meta.env.VITE_API_URL || 'http://localhost:3000',
  WS_URL: window.__ENV__?.WS_URL || import.meta.env.VITE_WS_URL || 'ws://localhost:3000',
}

// src/lib/httpclient.ts
import { env } from './env'

const httpClient = axios.create({
  baseURL: env.API_URL,  // ← 런타임에 결정!
  timeout: 10000,
})
```

### 최종 배포 플로우

```bash
# 1. 단일 이미지 빌드 (환경 무관)
pnpm build
docker build -t imprun-console:1.0.0 .

# 2. 개발 환경 배포
helm install imprun-dev ./k8s \
  --set console.env.API_URL=https://api.dev.imprun.dev \
  --set console.env.WS_URL=wss://api.dev.imprun.dev

# 3. 운영 환경 배포 (같은 이미지!)
helm install imprun-prod ./k8s \
  --set console.env.API_URL=https://api.imprun.dev \
  --set console.env.WS_URL=wss://api.imprun.dev
```

**동일한 이미지로 모든 환경에 배포 가능합니다!**

---

### Dockerfile 비교

#### Next.js (Before)

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile
COPY . .
RUN pnpm build

FROM node:20-alpine AS runner
WORKDIR /app
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public

ENV NODE_ENV=production
EXPOSE 3000

CMD ["node", "server.js"]
```

**이미지 크기**: ~250MB
**메모리 사용**: 256MB+
**시작 시간**: ~2초

#### React + nginx (After)

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile
COPY . .
RUN pnpm build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**이미지 크기**: ~15MB
**메모리 사용**: 10MB
**시작 시간**: ~0.1초

---

### 파일 라우팅 문제 해결

react-router-dom v6의 중첩 라우팅으로 깔끔하게 해결:

```typescript
// src/routes/index.tsx
import { createBrowserRouter } from 'react-router-dom'

export const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    children: [
      { index: true, element: <HomePage /> },
      {
        path: 'applications',
        children: [
          { index: true, element: <ApplicationList /> },
          {
            path: ':gatewayId',
            element: <ApplicationDetail />,
            children: [
              { index: true, element: <Overview /> },
              {
                path: 'functions',
                children: [
                  { index: true, element: <FunctionList /> },
                  { path: ':name/edit', element: <FunctionEditor /> },
                ],
              },
            ],
          },
        ],
      },
    ],
  },
])
```

**파일 구조**:
```
src/
  routes/
    index.tsx           # 라우트 정의
  pages/
    applications/
      ApplicationList.tsx
      ApplicationDetail.tsx
      functions/
        FunctionList.tsx
        FunctionEditor.tsx
```

- 명시적이고 직관적
- TypeScript 타입 추론 우수
- 중첩 레이아웃 지원
- 파일 구조와 라우트 구조 분리 가능

---

## 성능 및 리소스 비교

### 빌드 시간

| 프레임워크 | 최초 빌드 | 증분 빌드 |
|-----------|----------|----------|
| Next.js | 45초 | 12초 |
| Vite + React | 8초 | 2초 |

### 개발 서버 시작 시간

| 프레임워크 | Cold Start | Hot Reload |
|-----------|-----------|-----------|
| Next.js | 3.2초 | 1.5초 |
| Vite + React | 0.6초 | 0.3초 |

### 번들 크기

| 프레임워크 | 초기 JS | 총 크기 |
|-----------|---------|---------|
| Next.js | 285KB | 1.2MB |
| Vite + React | 156KB | 620KB |

### 런타임 리소스 (3 replicas)

| 항목 | Next.js | React + nginx | 절감 |
|------|---------|---------------|------|
| 메모리 | 768MB | 30MB | **96%** |
| CPU (idle) | 0.3 core | 0.003 core | **99%** |
| 이미지 크기 | 250MB | 15MB | **94%** |
| 시작 시간 | 6초 | 0.3초 | **95%** |

**총 리소스 절감**: **약 740MB의 메모리 확보**

---

## 교훈 및 결론

### 1. 프레임워크는 도구일 뿐이다

Next.js는 훌륭한 프레임워크입니다. 하지만 **모든 상황에 최적은 아닙니다**.

**Next.js가 적합한 경우**:
- ✅ SEO가 중요한 마케팅 사이트
- ✅ SSR이 필수적인 콘텐츠 플랫폼
- ✅ Vercel 등 매니지드 환경에 배포
- ✅ 충분한 서버 리소스 확보 가능

**순수 React가 적합한 경우**:
- ✅ Admin 대시보드, 내부 도구
- ✅ 인증 뒤의 Private SPA
- ✅ 제한된 리소스 환경
- ✅ 정적 파일 서빙으로 충분한 경우

### 2. 환경 변수 관리는 아키텍처 결정이다

`NEXT_PUBLIC_` 같은 빌드타임 환경 변수는:
- ❌ 멀티 환경 배포에 부적합
- ❌ K8s ConfigMap/Secret과 궁합이 나쁨
- ❌ Docker 이미지 재사용 불가

**런타임 환경 변수 주입**이 필요한 경우:
- ✅ window.__ENV__ 패턴 사용
- ✅ nginx sub_filter 활용
- ✅ 환경마다 동일 이미지 사용

### 3. 리소스 제약은 기술 선택을 바꾼다

**이론적 최고 vs 실용적 최선**

제 경우 4 core / 24GB 노드에서:
- Next.js: 768MB (3 replicas) = 전체 리소스의 10%
- React + nginx: 30MB = 전체 리소스의 0.4%

**26배의 리소스 효율 차이**는 **다른 서비스를 위한 여유**가 됩니다.

### 4. 파일 라우팅은 생산성의 핵심

처음에 TanStack Router를 버린 이유가 바로 이것입니다.

**파일 = 라우트** 패턴의 위력:
- 빠른 페이지 추가
- 직관적인 구조
- 자동 코드 스플리팅

react-router-dom v6의 중첩 라우팅으로 이를 달성하면서도:
- 명시적 선언 유지
- 타입 안전성 확보
- 복잡도 낮춤

### 5. 백엔드는 NestJS로 정착

반면 **백엔드는 NestJS가 정답**입니다:

```
NestJS의 장점:
✅ TypeScript 네이티브
✅ 강력한 DI 컨테이너
✅ 모듈화된 아키텍처
✅ Express/Fastify 선택 가능
✅ Swagger, TypeORM, MongoDB 통합 우수
✅ 마이크로서비스 지원
```

프론트엔드와 달리 백엔드는:
- 정적 파일 서빙 불필요
- 비즈니스 로직 복잡도 높음
- 런타임 성능이 Node.js로 충분
- 프레임워크의 구조화가 필수

**결론**: NestJS는 계속 사용할 것입니다.

---

## 최종 아키텍처

```
현재 프로덕션 스택:

프론트엔드:
├── React 19
├── react-router-dom v6
├── Zustand + TanStack Query v5
├── Vite
└── nginx (정적 서빙)

백엔드:
├── NestJS
├── MongoDB (Mongoose)
├── Redis
└── JWT + Passport

인프라:
├── Kubernetes (ARM64)
├── Helm Charts
├── nginx-ingress
└── cert-manager
```

**배포 플로우**:
1. Vite로 정적 빌드
2. nginx 이미지로 패키징
3. 환경별 Helm values 설정
4. 동일 이미지로 모든 환경 배포

---

## 마무리

> "완벽한 프레임워크는 없다. 상황에 맞는 프레임워크가 있을 뿐이다."

Next.js를 버린 것이 아닙니다. **제 상황에 맞지 않았을 뿐**입니다.

여러분의 프로젝트가:
- Vercel에 배포되고
- 충분한 리소스가 있고
- SEO가 중요하다면

Next.js는 여전히 최고의 선택입니다.

하지만 제 경우처럼:
- 제한된 리소스의 K8s 클러스터
- Private Admin 대시보드
- 멀티 환경 배포 필요

이런 상황이라면 **순수 React + nginx**를 고려해보세요.

**기술 선택의 핵심은 트레이드오프를 이해하고, 우선순위를 명확히 하는 것입니다.**

---

## 참고 자료

- [React Router v6 공식 문서](https://reactrouter.com/)
- [Vite 공식 문서](https://vitejs.dev/)
- [nginx 성능 최적화 가이드](https://www.nginx.com/blog/tuning-nginx/)
- [Kubernetes ConfigMap 환경 변수 주입](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
- [Next.js Standalone Output](https://nextjs.org/docs/app/api-reference/next-config-js/output)

---

**태그**: `#React` `#NextJS` `#nginx` `#Kubernetes` `#아키텍처` `#프론트엔드` `#배포전략` `#리소스최적화`
