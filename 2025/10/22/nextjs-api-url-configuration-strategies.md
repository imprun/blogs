# Next.js SSR 환경에서 API URL 환경변수 관리 전략

> **작성일**: 2025-10-22
> **태그**: Next.js, Frontend, Kubernetes, DevOps, Environment Variables
> **난이도**: 중급~고급

## 들어가며

[**imprun.dev**](https://imprun.dev)는 Kubernetes 기반 서버리스 Cloud Function 플랫폼입니다. Next.js 15 App Router 기반의 웹 콘솔을 구축하면서, "환경별로 다른 API 엔드포인트를 어떻게 관리할 것인가?"라는 문제에 직면했습니다.

**우리가 마주한 상황**:
- ✅ **로컬 개발**: `http://localhost:3000` (API 서버)
- ✅ **Kubernetes 스테이징**: `https://api.staging.imprun.dev`
- ✅ **Kubernetes 프로덕션**: `https://api.imprun.dev`
- ❌ **문제**: `NEXT_PUBLIC_*` 환경변수는 빌드 타임에 하드코딩됨
- ❌ **결과**: 환경마다 다른 Docker 이미지를 빌드해야 함

이 글은 **imprun.dev 플랫폼 구축 경험**을 바탕으로, Next.js SSR 환경에서 **런타임에 동적으로 API URL을 설정하는 5가지 전략**을 비교하고, MVP 단계에서의 실전 의사결정 과정을 공유합니다.

---

## 문제 정의: Next.js의 `NEXT_PUBLIC_*` 환경변수

### Next.js 환경변수 동작 방식

Next.js는 클라이언트에서 접근 가능한 환경변수에 `NEXT_PUBLIC_` 접두사를 요구합니다.

```typescript
// frontend/src/lib/constants.ts
export const API_BASE_URL = process.env.NEXT_PUBLIC_API_URL || ""
```

**문제점**: 이 환경변수는 **빌드 타임에 정적으로 치환**됩니다.

```typescript
// 빌드 전 (소스 코드)
const url = process.env.NEXT_PUBLIC_API_URL

// 빌드 후 (번들 JavaScript)
const url = "https://api.imprun.dev"  // ← 하드코딩됨!
```

### 현재 구조의 문제점

**1. Dockerfile에서 빌드 타임 주입**

```dockerfile
# frontend/Dockerfile
ARG NEXT_PUBLIC_API_URL
ENV NEXT_PUBLIC_API_URL=${NEXT_PUBLIC_API_URL}

RUN pnpm build  # 이 시점에 API URL이 고정됨
```

**2. 환경별로 다른 이미지 빌드 필요**

```bash
# 개발 환경
docker build --build-arg NEXT_PUBLIC_API_URL=http://localhost:3000 -t console:dev .

# 스테이징 환경
docker build --build-arg NEXT_PUBLIC_API_URL=https://api.staging.imprun.dev -t console:staging .

# 프로덕션 환경
docker build --build-arg NEXT_PUBLIC_API_URL=https://api.imprun.dev -t console:prod .
```

→ **3개의 서로 다른 이미지 관리**, CI/CD 복잡도 증가

**3. Kubernetes 환경변수는 무시됨**

```yaml
# k8s/imprun/charts/imprun-console/templates/deployment.yaml
env:
  - name: NEXT_PUBLIC_API_URL
    value: "https://api.imprun.dev"  # ← 이미 빌드된 번들에는 영향 없음
```

런타임 환경변수를 변경해도 이미 빌드된 JavaScript 번들 내부의 하드코딩된 값은 바뀌지 않습니다.

### OAuth 리다이렉트 문제

추가로, OAuth 인증 플로우에서는 브라우저가 직접 API 서버로 리다이렉트되므로 **명시적인 API URL이 필수**입니다.

```typescript
// OAuth 로그인
window.location.href = '/v1/auth/github'
// Next.js rewrites는 fetch/axios에만 작동, window.location.href는 작동 안 함
```

---

## 해결 방안 1: 런타임 환경변수 주입 (API 엔드포인트)

### 개념

Next.js 서버에서 제공하는 공개 API를 통해 런타임 설정을 클라이언트에 주입합니다.

### 구현

**1. 공개 설정 API 엔드포인트**

```typescript
// frontend/src/app/api/config/route.ts (신규)
import { NextResponse } from 'next/server'

export async function GET() {
  return NextResponse.json({
    apiUrl: process.env.NEXT_PUBLIC_API_URL || '',
  })
}
```

**2. 클라이언트에서 동적 로드**

```typescript
// frontend/src/lib/config.ts (신규)
let runtimeConfig: { apiUrl: string } | null = null

export async function getRuntimeConfig() {
  if (runtimeConfig) return runtimeConfig

  const res = await fetch('/api/config')
  runtimeConfig = await res.json()
  return runtimeConfig
}
```

**3. httpClient 초기화**

```typescript
// frontend/src/lib/httpclient.ts
export const httpClient = axios.create({
  baseURL: '',  // 초기값 비움
})

export async function initHttpClient() {
  const config = await getRuntimeConfig()
  httpClient.defaults.baseURL = config.apiUrl
}

// frontend/src/app/layout-client.tsx
useEffect(() => {
  initHttpClient()
}, [])
```

**4. Kubernetes 환경변수 활용**

```yaml
# k8s/imprun/charts/imprun-console/templates/deployment.yaml
env:
  - name: NEXT_PUBLIC_API_URL
    value: {{ .Values.env.NEXT_PUBLIC_API_URL | quote }}
```

### 장단점

| 항목 | 평가 |
|------|------|
| 빌드 타임 하드코딩 | ✅ 제거됨 |
| 단일 이미지 배포 | ✅ 가능 |
| 로컬 개발 편의성 | ✅ 우수 |
| 초기 로딩 성능 | ⚠️ API 호출 1회 추가 |
| SSR 페이지 | ⚠️ 서버 환경변수 직접 접근 필요 |

---

## 해결 방안 2: Nginx Ingress 프록시 (가장 깔끔)

### 개념

클라이언트는 상대 경로(`/v1/...`)만 사용하고, Nginx Ingress Controller가 API 서버로 프록시합니다.

### 아키텍처

```
브라우저
  ↓
portal.imprun.dev/v1/applications
  ↓
Nginx Ingress Controller
  ├─ /v1/* → imprun-server (api.imprun.dev)  ← API 프록시
  └─ /*    → imprun-console (Next.js)        ← 정적/SSR 페이지
```

### 구현

**1. Ingress에 API 프록시 규칙 추가**

```yaml
# k8s/imprun/charts/imprun-console/templates/ingress-api-proxy.yaml (신규)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: imprun-console-api-proxy
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - portal.imprun.dev
      secretName: portal-imprun-dev-tls
  rules:
    - host: portal.imprun.dev
      http:
        paths:
          # API 프록시: /v1/* → imprun-server
          - path: /v1/(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: imprun-imprun-server
                port:
                  number: 80
```

**2. Next.js 설정 정리**

```typescript
// frontend/next.config.ts
const nextConfig: NextConfig = {
  output: 'standalone',
  // rewrites 제거! Nginx가 처리함
}
```

**3. httpClient를 상대 경로로**

```typescript
// frontend/src/lib/httpclient.ts
export const httpClient = axios.create({
  baseURL: '',  // 상대 경로 사용
})
```

**4. Dockerfile ARG 제거**

```dockerfile
# frontend/Dockerfile
# ARG NEXT_PUBLIC_API_URL  ← 삭제
# ENV NEXT_PUBLIC_API_URL=${NEXT_PUBLIC_API_URL}  ← 삭제

RUN pnpm build
```

### 장단점

| 항목 | 평가 |
|------|------|
| 빌드 타임 하드코딩 | ✅ 완전 제거 |
| 단일 이미지 배포 | ✅ 가능 |
| 네트워크 효율 | ✅ Nginx 직접 프록시 (Next.js 경유 안 함) |
| CORS 문제 | ✅ Same-Origin |
| 로컬 개발 | ❌ Kubernetes 필요 |
| OAuth 리다이렉트 | ⚠️ 브라우저 직접 접근 시 프록시 안 됨 |

**치명적 단점: 로컬 개발 시 불편함**

```bash
# 로컬 개발 시
cd frontend
pnpm dev

# http://localhost:3001 접속
# → /v1/applications 호출 시 404 (Nginx 없음)
# → 개발자가 직접 localhost:3000/v1/applications 호출해야 함
```

---

## 해결 방안 3: Runtime Config Injection (고급)

### 개념

컨테이너 시작 시 JavaScript 파일에 환경변수를 주입합니다.

### 구현

**1. 설정 파일 템플릿**

```javascript
// frontend/public/config.js.template
window.__RUNTIME_CONFIG__ = {
  apiUrl: '${NEXT_PUBLIC_API_URL}'
}
```

**2. Entrypoint 스크립트**

```bash
#!/bin/sh
# frontend/docker-entrypoint.sh
envsubst < /app/public/config.js.template > /app/public/config.js
exec node frontend/server.js
```

**3. HTML에서 로드**

```tsx
// frontend/src/app/layout.tsx
<script src="/config.js"></script>
```

**4. TypeScript에서 사용**

```typescript
// frontend/src/lib/config.ts
export const getApiUrl = () => {
  return window.__RUNTIME_CONFIG__?.apiUrl || ''
}
```

### 장단점

| 항목 | 평가 |
|------|------|
| 빌드 타임 하드코딩 | ✅ 완전 제거 |
| 런타임 설정 | ✅ 완전한 동적 설정 |
| 추가 API 호출 | ✅ 없음 |
| 복잡도 | ⚠️ Entrypoint 스크립트 관리 필요 |
| SSR 페이지 | ❌ 사용 불가 (브라우저 전용) |

---

## 해결 방안 4: All-in-One 컨테이너 + Nginx (Admin Service 패턴)

### 개념

**단일 컨테이너 내부에 Nginx, API Server, Frontend를 모두 포함**시키고, Nginx가 내부 프록시 역할을 수행합니다. `supervisord`로 3개 프로세스를 동시에 관리합니다.

### 아키텍처

```
Docker Container (Port 3000)
  ↓
Nginx (localhost:3000)
  ├─ /api/* → Admin Server (localhost:3001)
  └─ /*     → Next.js Frontend (localhost:4001)
```

**핵심**: 브라우저는 항상 `http://localhost:3000`만 보며, 컨테이너 내부에서 라우팅이 완결됩니다.

### 구현

**1. Multi-stage Dockerfile**

```dockerfile
# services/admin/Dockerfile

# Stage 1: API Server 빌드
FROM node:24-bookworm-slim AS server-builder
WORKDIR /app
COPY services/admin/server ./services/admin/server
RUN pnpm build

# Stage 2: Frontend 빌드
FROM node:24-bookworm-slim AS frontend-builder
WORKDIR /app
COPY services/admin/frontend ./services/admin/frontend

# ✨ 핵심: API URL을 /api로 하드코딩 (Nginx 프록시 경로)
ARG NEXT_PUBLIC_API_URL=/api
ENV NEXT_PUBLIC_API_URL=${NEXT_PUBLIC_API_URL}
RUN pnpm build

# Stage 3: All-in-One Runtime
FROM node:24-bookworm-slim AS runtime

# Nginx + Supervisor 설치
RUN apt-get update && \
    apt-get install -y nginx supervisor

# API Server 복사
COPY --from=server-builder /app/services/admin/server/dist ./services/admin/server/dist

# Frontend 복사
COPY --from=frontend-builder /app/services/admin/frontend/.next/standalone ./

# Nginx 설정 (Heredoc 사용)
COPY --chown=root:root <<EOF /etc/nginx/sites-available/admin
server {
    listen 3000;

    # API 프록시 (Admin Server)
    location /api/ {
        proxy_pass http://localhost:3001/;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
    }

    # Frontend (Next.js Standalone)
    location / {
        proxy_pass http://localhost:4001;
        proxy_set_header Host \$host;
    }
}
EOF

RUN ln -s /etc/nginx/sites-available/admin /etc/nginx/sites-enabled/admin

# Supervisord 설정
COPY --chown=root:root <<EOF /etc/supervisor/conf.d/admin.conf
[program:nginx]
command=/usr/sbin/nginx -g 'daemon off;'
priority=10

[program:admin-server]
command=/usr/local/bin/node dist/main.js
directory=/app/services/admin/server
environment=PORT="3001"
priority=20

[program:admin-frontend]
command=/usr/local/bin/node server.js
directory=/app/services/admin/frontend
environment=PORT="4001"
priority=20
EOF

EXPOSE 3000
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/admin.conf"]
```

**2. Frontend httpClient**

```typescript
// services/admin/frontend/src/lib/httpclient.ts
export const httpClient = axios.create({
  baseURL: '/api',  // ← Nginx가 localhost:3001로 프록시
})
```

**3. Kubernetes Deployment (단순함)**

```yaml
# k8s/imprun/charts/admin-service/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: admin-service
spec:
  containers:
    - name: admin
      image: junsik/imprun-admin:latest
      ports:
        - containerPort: 3000
      # 환경변수 불필요! 모든 것이 컨테이너 내부에서 해결됨
```

### 장단점

| 항목 | 평가 |
|------|------|
| 빌드 타임 하드코딩 | ✅ `/api`로 고정 (문제 없음) |
| 단일 이미지 배포 | ✅ 가능 |
| 로컬 개발 | ✅ Docker Compose로 쉽게 재현 |
| 네트워크 효율 | ✅ 컨테이너 내부 localhost 통신 |
| CORS 문제 | ✅ Same-Origin |
| 컨테이너 크기 | ⚠️ Nginx + 3개 프로세스로 증가 |
| 모니터링 | ⚠️ Supervisord 로그 관리 필요 |
| 독립 스케일링 | ❌ Frontend/Backend 개별 확장 불가 |

### 언제 사용하는가?

**적합한 경우**:
- ✅ 내부 Admin 도구 (트래픽 적음)
- ✅ Frontend/Backend가 항상 1:1로 배포
- ✅ 간단한 배포 파이프라인 선호
- ✅ 환경변수 관리 최소화

**부적합한 경우**:
- ❌ 높은 트래픽 (Frontend/Backend 독립 스케일링 필요)
- ❌ 마이크로서비스 아키텍처 (서비스 분리 필요)
- ❌ 여러 Frontend가 동일 Backend 공유

### 실제 사례: imprun.dev Admin Service

imprun.dev의 Admin Service는 **내부 운영 도구**로, 다음과 같은 이유로 All-in-One 패턴을 선택했습니다:

1. **낮은 트래픽**: 운영팀만 사용 (동시 사용자 < 10명)
2. **간단한 배포**: 단일 Pod만 관리
3. **환경변수 제거**: API URL 걱정 없음
4. **빠른 개발**: 로컬에서 `docker run` 한 번으로 전체 스택 실행

```bash
# 로컬 개발 (Docker만 있으면 됨)
docker build -t admin:dev ./services/admin
docker run -p 3000:3000 admin:dev

# http://localhost:3000 접속
# → Nginx → Frontend (4001) 또는 API (3001)
```

---

## 해결 방안 5: 하이브리드 접근법 (로컬/프로덕션 분리)

### 개념

**개발 환경**에서는 `next.config.ts` rewrites 사용, **프로덕션**에서는 Nginx Ingress 사용.

### 구현

**1. 환경별 rewrites 활성화**

```typescript
// frontend/next.config.ts
const isDevelopment = process.env.NODE_ENV === 'development'
const apiDestination = process.env.NEXT_PUBLIC_API_URL

const nextConfig: NextConfig = {
  output: 'standalone',

  async rewrites() {
    if (!isDevelopment || !apiDestination) {
      return []  // 프로덕션: Nginx가 처리
    }

    // 개발 환경: Next.js가 프록시
    return [
      {
        source: '/v1/:path*',
        destination: `${apiDestination}/v1/:path*`,
      },
    ]
  },
}
```

**2. 환경별 baseURL**

```typescript
// frontend/src/lib/httpclient.ts
const getBaseURL = () => {
  if (typeof window !== 'undefined') {
    if (process.env.NODE_ENV === 'production') {
      return ''  // 프로덕션: 상대 경로 (Nginx 프록시)
    }
    return API_BASE_URL || ''  // 개발: 명시적 URL
  }
  return API_BASE_URL || ''
}

export const httpClient = axios.create({
  baseURL: getBaseURL(),
})
```

**3. Dockerfile: NODE_ENV=production**

```dockerfile
# frontend/Dockerfile
ENV NODE_ENV=production
RUN pnpm build  # rewrites 비활성화됨
```

**4. Kubernetes: Nginx Ingress 프록시**

```yaml
# k8s/imprun/charts/imprun-console/templates/ingress-api-proxy.yaml
# (방안 2와 동일)
```

### 장단점

| 항목 | 평가 |
|------|------|
| 로컬 개발 편의성 | ✅ 우수 (Kubernetes 불필요) |
| 프로덕션 최적화 | ✅ Nginx 프록시 |
| 빌드 타임 하드코딩 | ✅ 제거 |
| 단일 이미지 배포 | ✅ 가능 |
| 코드 복잡도 | ⚠️ 환경별 분기 로직 필요 |

---

## 방안 비교 매트릭스

| 특성 | 방안 1<br>(API 엔드포인트) | 방안 2<br>(Nginx Ingress) | 방안 3<br>(Config Injection) | 방안 4<br>(All-in-One) | 방안 5<br>(하이브리드) |
|------|--------------------------|-------------------------|---------------------------|----------------------|----------------------|
| 빌드 하드코딩 제거 | ✅ | ✅ | ✅ | ⚠️ `/api` 고정 | ✅ |
| 단일 이미지 배포 | ✅ | ✅ | ✅ | ✅ | ✅ |
| 로컬 개발 편의성 | ✅ | ❌ | ⚠️ | ✅ | ✅ |
| 네트워크 효율 | ⚠️ | ✅ | ✅ | ✅✅ | ✅ |
| 구현 복잡도 | 낮음 | 중간 | 높음 | 중간 | 중간 |
| SSR 호환성 | ⚠️ | ✅ | ❌ | ✅ | ✅ |
| OAuth 리다이렉트 | ✅ | ⚠️ | ✅ | ✅ | ✅ |
| 독립 스케일링 | ✅ | ✅ | ✅ | ❌ | ✅ |
| 적합한 용도 | 범용 | 고트래픽 | 정적 호스팅 | 내부 도구 | 범용 |

---

## MVP 단계 의사결정: 현재 방식 유지

imprun.dev 프로젝트는 **현재 MVP 단계**이므로, 다음과 같은 이유로 **기존 방식(빌드 타임 ARG)을 유지**하기로 결정했습니다.

### 유지 이유

1. **단순성 우선**: MVP는 빠른 검증이 목적
2. **이미 작동하는 코드**: 추가 리팩토링 없이 개발 집중
3. **환경 수 제한**: 현재 dev/production 2개만 관리
4. **CI/CD 미구축**: 수동 빌드/배포로 환경별 이미지 생성 부담 적음

### 현재 구조

```dockerfile
# frontend/Dockerfile
ARG NEXT_PUBLIC_API_URL
ENV NEXT_PUBLIC_API_URL=${NEXT_PUBLIC_API_URL}
RUN pnpm build
```

```yaml
# k8s/imprun/values.yaml
imprun-console:
  env:
    NEXT_PUBLIC_API_URL: "https://api.imprun.dev"
```

### 향후 마이그레이션 시점

프로젝트 성격에 따라 다음 방안을 추천합니다:

#### **방안 4 (All-in-One 컨테이너)** - 내부 도구
다음 조건에 해당하면 Admin Service 패턴 사용:
- ✅ 내부 운영 도구 (사용자 < 50명)
- ✅ Frontend/Backend 1:1 배포
- ✅ 간단한 배포 선호
- ✅ 독립 스케일링 불필요

#### **방안 5 (하이브리드 접근법)** - 일반 서비스
다음 조건이 충족되면 전환 권장:
- ✅ 환경 3개 이상 (dev/staging/production)
- ✅ CI/CD 파이프라인 구축 완료
- ✅ 멀티 리전 배포 시작
- ✅ 개발자 3명 이상 (로컬 개발 경험 중요도 증가)
- ✅ Frontend/Backend 독립 스케일링 필요

---

## 실전 구현 가이드: 하이브리드 접근법 (향후 참고용)

MVP 이후 마이그레이션을 고려 중이라면, 아래 체크리스트를 참고하세요.

### 마이그레이션 체크리스트

#### Phase 1: 로컬 개발 환경 유지

- [ ] `frontend/next.config.ts` 수정
  ```typescript
  async rewrites() {
    if (process.env.NODE_ENV !== 'development') return []
    // ...
  }
  ```

- [ ] `frontend/src/lib/httpclient.ts` 환경별 baseURL
  ```typescript
  const baseURL = process.env.NODE_ENV === 'production' ? '' : API_BASE_URL
  ```

- [ ] 로컬 테스트
  ```bash
  cd frontend
  pnpm dev
  # http://localhost:3001 접속 → API 호출 정상 확인
  ```

#### Phase 2: 프로덕션 Nginx 프록시 설정

- [ ] `k8s/imprun/charts/imprun-console/templates/ingress-api-proxy.yaml` 생성
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: imprun-console-api-proxy
  # ...
  ```

- [ ] `k8s/imprun/charts/imprun-console/values.yaml` 설정 추가
  ```yaml
  apiProxy:
    enabled: true
    serviceName: imprun-imprun-server
    servicePort: 80
  ```

- [ ] Helm 업그레이드
  ```bash
  helm upgrade imprun ./k8s/imprun -f values-production.yaml
  ```

- [ ] Ingress 생성 확인
  ```bash
  kubectl get ingress -n imprun-system
  ```

#### Phase 3: Dockerfile 정리

- [ ] `frontend/Dockerfile`에서 ARG 제거
  ```dockerfile
  # ARG NEXT_PUBLIC_API_URL  ← 삭제
  ENV NODE_ENV=production
  RUN pnpm build
  ```

- [ ] 새 이미지 빌드 및 푸시
  ```bash
  docker build -t imprun-console:v2.0.0 ./frontend
  docker push imprun-console:v2.0.0
  ```

- [ ] Kubernetes Deployment 이미지 태그 업데이트

#### Phase 4: 검증

- [ ] 프로덕션 접속: `https://portal.imprun.dev`
- [ ] 브라우저 개발자 도구 네트워크 탭 확인
  - `/v1/applications` → `200 OK`
  - Request URL: `https://portal.imprun.dev/v1/applications` (Same-Origin)
- [ ] OAuth 로그인 테스트
- [ ] 환경변수 변경 테스트
  ```bash
  # values-production.yaml 수정 없이 Ingress annotation만 변경
  kubectl annotate ingress imprun-console-api-proxy \
    nginx.ingress.kubernetes.io/upstream-vhost=new-api.imprun.dev
  ```

---

## 결론

Next.js SSR 환경에서 API URL을 관리하는 것은 단순해 보이지만, **빌드 타임 하드코딩, 로컬 개발 편의성, 프로덕션 최적화**라는 세 가지 목표를 동시에 만족시키기 어렵습니다.

### 핵심 교훈

1. **MVP는 단순함이 승리**: 완벽한 구조보다 빠른 검증이 우선
2. **프로젝트 성격에 따른 선택**:
   - 일반 서비스: 하이브리드 접근법 (로컬 rewrites + 프로덕션 Nginx Ingress)
   - 내부 도구: All-in-One 컨테이너 (Nginx + API + Frontend)
3. **마이그레이션 시점 명확화**: 환경 수, 트래픽, 개발자 수 고려
4. **문서화의 중요성**: 기술 부채를 명시하고 해결 방안 기록

### 방안 선택 가이드

```
트래픽 많음 + 독립 스케일링 필요
  → 방안 2 (Nginx Ingress) 또는 방안 5 (하이브리드)

내부 도구 + 단순 배포 선호
  → 방안 4 (All-in-One 컨테이너)

로컬 개발 + 프로덕션 최적화 모두 중요
  → 방안 5 (하이브리드)

MVP 단계 + 빠른 검증 우선
  → 현재 방식 유지 (빌드 타임 ARG)
```

### 참고 자료

- [Next.js Environment Variables 공식 문서](https://nextjs.org/docs/pages/building-your-application/configuring/environment-variables)
- [Nginx Ingress Controller Rewrite 설정](https://kubernetes.github.io/ingress-nginx/examples/rewrite/)
- [Helm Umbrella Chart Best Practices](./helm-chart-best-practices.md)

---

> 💡 **TIP**: 이 글에서 논의한 내용은 프로젝트 요구사항에 따라 달라질 수 있습니다.
> 자신의 프로젝트에서는 환경 수, 개발자 수, CI/CD 성숙도를 고려하여 최적의 방안을 선택하세요.
