# Apache APISIX로 멀티 테넌트 서버리스 플랫폼 설계하기: 3계층 아키텍처 구현 노하우

> **작성일**: 2025년 10월 29일
> **대상 독자**: 플랫폼 엔지니어, 아키텍트, API Gateway 설계자
> **난이도**: 중급~고급
> **주제**: Apache APISIX, Kubernetes, 서버리스 아키텍처

---

## TL;DR

- ✅ **Gateway → Environment → Function** 3계층 아키텍처를 APISIX로 구현
- ✅ **Route 수 최소화**: 100개 Function이어도 Gateway당 Route 4개만 생성
- ✅ **동적 라우팅**: APISIX Route는 고정, Runtime이 Function 동적 실행
- ✅ **Plugin 계층 상속**: Gateway 전역 → Environment별 → Function별 Override
- ✅ **멀티 테넌트**: Application마다 독립된 Namespace와 Upstream
- ✅ **Kubernetes Native**: CRD 기반 선언적 관리

**GitHub**: [imprun.dev](https://github.com/imprun/imprun)

---

## 들어가며

[**imprun.dev**](https://imprun.dev)는 Kubernetes 기반 오픈소스 서버리스 플랫폼입니다.
개발자가 코드만 작성하면 즉시 API 엔드포인트가 생성되는 **CloudFunction** 서비스를 제공합니다.

### 우리가 마주한 질문

플랫폼을 설계하면서 **API Gateway 아키텍처**에 대한 근본적인 질문들을 마주했습니다:

- ❓ **Function 100개면 APISIX Route도 100개?** → Kubernetes API 서버 부하 증가
- ❓ **Function 추가할 때마다 CRD 업데이트?** → 배포 시간 증가 (reconciliation loop)
- ❓ **dev, staging, prod 환경 분리는?** → Plugin 설정 어떻게 관리?
- ❓ **멀티 테넌트 격리는?** → Application마다 독립 Gateway 필요

### 검증 과정

**1. 시도: Function 1개 = APISIX Route 1개**
```yaml
# Function마다 독립 Route
- /dev/user/me → ApisixRoute "dev-user-me"
- /dev/user/list → ApisixRoute "dev-user-list"
```
- ❌ Function 1000개 × 3 환경 = 3000개 Route
- ❌ APISIX 성능 저하 (Route matching O(n))
- ❌ Kubernetes API 서버 부하

**2. 시도: Wildcard Route + External Router**
```yaml
# 모든 요청을 외부 Router로
- /* → External Router Service
```
- ✅ APISIX Route 1개로 해결
- ❌ 외부 Router 관리 복잡도
- ❌ Plugin 적용 위치 모호 (APISIX? Router?)

**3. 최종 선택: Environment 단위 Route + Runtime 동적 실행** ← **강조**
```yaml
# Environment(Stage)별 Route만 생성
- /dev/* → Runtime (dev functions)
- /staging/* → Runtime (staging functions)
- /prod/* → Runtime (prod functions)
```
- ✅ Function 1000개여도 Route 4개 (Base + 3 Stages)
- ✅ Runtime이 path 파싱하여 Function 동적 실행
- ✅ Plugin 계층 구조 (Gateway → Stage → Function)
- ✅ Kubernetes Native (CRD 기반)

### 결론

- ✅ **확장성**: Function 개수와 Route 수 분리
- ✅ **즉시 배포**: 코드 변경 시 APISIX 변경 불필요
- ✅ **계층적 Plugin**: 상속 + Override 가능
- ✅ **멀티 테넌시**: Namespace 기반 격리

이 글은 **imprun.dev 플랫폼 구축 경험**을 바탕으로, Apache APISIX로 확장 가능한 서버리스 아키텍처를 설계한 노하우를 상세히 공유합니다.

---

## 문제 정의: 서버리스 플랫폼에서 API Gateway의 도전 과제

### 전통적인 접근의 한계

**Naive한 설계 (Function 1개 = Route 1개)**:

```yaml
# ❌ 비효율적인 설계
APISIX Routes:
  - /dev/user/me → Function "dev/user/me"
  - /dev/user/list → Function "dev/user/list"
  - /dev/product/list → Function "dev/product/list"
  - /dev/product/detail → Function "dev/product/detail"
  - ... (100개 Function = 100개 Route)
```

**문제점**:
- ✗ Function 추가할 때마다 APISIX CRD 업데이트 필요
- ✗ Kubernetes API 서버에 부하 증가
- ✗ Route 수가 수천 개로 증가 시 APISIX 성능 저하
- ✗ 배포 시간 증가 (K8s reconciliation loop)
- ✗ Function별 독립 Plugin 설정 어려움

### 우리의 요구사항

1. **확장성**: Function 개수와 무관하게 Route 수 최소화
2. **즉시 배포**: 코드 변경 시 APISIX 재설정 없이 즉시 반영
3. **계층적 Plugin**: Gateway → Environment → Function 상속 구조
4. **멀티 테넌시**: Application(테넌트)별 완전 격리
5. **표준 준수**: Kubernetes Native + GitOps 친화적

---

## 해결책: 3계층 동적 라우팅 아키텍처

### 핵심 아이디어

```
APISIX Route는 "환경(Environment)" 단위까지만 생성
Function은 Runtime이 동적으로 처리
```

### 아키텍처 개요

```mermaid
graph TB
    subgraph "Layer 1: Gateway (Application)"
        GW["Gateway<br/>myapp.api.imprun.dev"]
        UP["APISIX Upstream<br/>myapp-service"]
        BASE["Base Route<br/>/* (priority 1)"]
        PLUGINS_GW["Gateway Plugins<br/>CORS, Rate Limit"]
    end

    subgraph "Layer 2: Environment (Stage)"
        DEV["dev Route<br/>/dev/* (priority 10)"]
        STG["staging Route<br/>/staging/* (priority 10)"]
        PROD["prod Route<br/>/prod/* (priority 10)"]
        PLUGINS_ENV["Environment Plugins<br/>JWT, IP Restriction"]
    end

    subgraph "Layer 3: CloudFunction (Dynamic)"
        RT["Runtime Pod<br/>imp-runtime-nodejs"]
        MONGO["MongoDB<br/>CloudFunction Collection"]
        F1["dev/user/me"]
        F2["dev/user/list"]
        F3["prod/user/me"]
    end

    GW --> BASE
    GW --> DEV
    GW --> STG
    GW --> PROD

    BASE --> UP
    DEV --> UP
    STG --> UP
    PROD --> UP

    UP --> RT
    RT --> MONGO
    MONGO --> F1
    MONGO --> F2
    MONGO --> F3

    PLUGINS_GW -.상속.-> PLUGINS_ENV
    PLUGINS_ENV -.적용.-> RT

    style GW fill:#e1f5ff
    style DEV fill:#fff4e1
    style STG fill:#fff4e1
    style PROD fill:#ffe1e1
    style RT fill:#e8f5e9
    style MONGO fill:#f3e5f5
```

**핵심 원칙**:
- Layer 1: Application당 1개 Gateway (멀티테넌트 격리)
- Layer 2: 고정 3개 Environment Route (dev/staging/prod)
- Layer 3: APISIX Route 생성 안 함 (Runtime 동적 실행)

---

## APISIX 리소스 설계

### 1. Upstream: Application Backend Service

각 Application(Gateway)마다 **1개의 Upstream**을 생성합니다.

```yaml
# Kubernetes Service가 자동으로 APISIX Upstream이 됨
apiVersion: v1
kind: Service
metadata:
  name: myapp123  # gatewayId
  namespace: myapp123
  labels:
    imprun.dev/gatewayId: myapp123
spec:
  selector:
    app: myapp123
  ports:
    - port: 8000
      targetPort: 8000
  type: ClusterIP
```

**APISIX Upstream 설정** (자동 discovery):
```yaml
upstream:
  name: myapp123-upstream
  type: kubernetes
  service_name: myapp123
  service_namespace: myapp123
  service_port: 8000
  discovery_type: kubernetes
```

**특징**:
- Kubernetes Service Discovery 활용
- Pod 추가/제거 시 자동 반영
- Load Balancing 자동 처리

### 2. Base Route: Fallback & Gateway Plugins

**목적**: Gateway 전역 Plugin 적용 + Fallback 라우팅

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: myapp123-base
  namespace: myapp123
spec:
  http:
    - name: base-route
      match:
        hosts:
          - myapp123.api.imprun.dev
        paths:
          - /*
      priority: 1  # ⚠️ 낮은 우선순위 (Stage Route에 밀림)
      backends:
        - serviceName: myapp123
          servicePort: 8000
      plugins:
        - name: cors
          enable: true
          config:
            allow_origins: "*"
            allow_methods: "GET,POST,PUT,DELETE,PATCH,OPTIONS,HEAD"
            allow_headers: "*"
            allow_credential: true
        - name: rate-limit  # Gateway 전역 rate limit
          enable: true
          config:
            rate: 100
            time_window: 60
            key_type: var
            key: remote_addr
```

**왜 Base Route가 필요한가?**
1. **Gateway 전역 Plugin 적용**: 모든 환경에 공통 적용되는 CORS, Rate Limit 등
2. **Fallback 처리**: Stage Route에 매칭 안 되는 요청 처리 (health check 등)
3. **Plugin 상속 Base**: Environment Route가 Base의 Plugin을 상속

### 3. Stage Route: Environment별 Plugin Override

**핵심 설계**: Environment(dev/staging/prod)마다 **독립된 Route** 생성

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: myapp123-dev
  namespace: myapp123
  labels:
    imprun.dev/gatewayId: myapp123
    imprun.dev/stage: dev
spec:
  http:
    - name: dev-route
      match:
        hosts:
          - myapp123.api.imprun.dev
        paths:
          - /dev/*  # ⚠️ Stage prefix
      priority: 10  # ⚠️ Base Route보다 높음
      backends:
        - serviceName: myapp123
          servicePort: 8000
      pluginConfigName: myapp123-dev-plugins  # ⚠️ 분리된 PluginConfig
      pluginConfigNamespace: myapp123
```

**Stage별 Route**:
- `myapp123-dev`: `/dev/*` → dev environment plugins
- `myapp123-staging`: `/staging/*` → staging plugins
- `myapp123-prod`: `/prod/*` → prod plugins

**Priority 전략**:
```
Stage Routes (priority: 10)
  > Base Route (priority: 1)

Example:
  Request: /dev/user/me
    → myapp123-dev Route 매칭 ✅ (priority 10)
    → myapp123-base Route 무시 (priority 1)

  Request: /health
    → Stage Routes 매칭 실패
    → myapp123-base Route 매칭 ✅ (fallback)
```

### 4. ApisixPluginConfig: Plugin 계층 구조

**분리된 PluginConfig 사용 이유**:
- Route와 Plugin 설정 분리 (관심사 분리)
- Plugin 변경 시 Route 재생성 불필요
- 여러 Route에서 재사용 가능

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixPluginConfig
metadata:
  name: myapp123-dev-plugins
  namespace: myapp123
spec:
  plugins:
    # 1. Base CORS (항상 포함)
    - name: cors
      enable: true
      config:
        allow_origins: "*"
        allow_methods: "GET,POST,PUT,DELETE,PATCH,OPTIONS,HEAD"
        allow_headers: "*"
        allow_credential: true

    # 2. Gateway 전역 Plugins (Base Route에서 상속)
    - name: rate-limit
      enable: true
      config:
        rate: 100
        time_window: 60
        key_type: var
        key: remote_addr

    # 3. Environment별 Plugins (dev 전용)
    - name: ip-restriction  # dev에만 적용
      enable: true
      config:
        whitelist:
          - 10.0.0.0/8
          - 192.168.0.0/16
```

**Plugin Merging 로직** (NestJS 코드):

```typescript
// server/src/gateway/ingress/stage-route.service.ts
private buildPlugins(
  stagePlugins: Record<string, any>,
  appPlugins?: Record<string, any>,
): any[] {
  const plugins = []

  // 1. Base CORS (항상 추가)
  plugins.push({
    name: 'cors',
    enable: true,
    config: {
      allow_origins: '*',
      allow_methods: 'GET,POST,PUT,DELETE,PATCH,OPTIONS,HEAD',
      allow_headers: '*',
      expose_headers: '*',
      allow_credential: true,
    },
  })

  // 2. Gateway 전역 Plugins
  if (appPlugins) {
    for (const [name, config] of Object.entries(appPlugins)) {
      plugins.push({ name, enable: true, config })
    }
  }

  // 3. Stage Plugins (Override Gateway)
  for (const [name, config] of Object.entries(stagePlugins || {})) {
    // 동일 이름 Plugin 제거 (Override)
    const existingIndex = plugins.findIndex((p) => p.name === name)
    if (existingIndex !== -1) {
      plugins.splice(existingIndex, 1)
    }
    plugins.push({ name, enable: true, config })
  }

  return plugins
}
```

**Plugin 우선순위**:
```
Stage Plugins > Gateway Plugins > Base CORS
```

```mermaid
graph LR
    subgraph "Base Layer"
        BASE_CORS["CORS<br/>allow_origins: *"]
    end

    subgraph "Gateway Layer (Application)"
        GW_RATE["Rate Limit<br/>100 req/min"]
        GW_AUTH["JWT Auth<br/>Gateway Secret"]
    end

    subgraph "Stage Layer (Environment)"
        DEV_IP["IP Restriction<br/>Dev Whitelist"]
        PROD_RATE["Rate Limit<br/>10 req/min<br/>(Override)"]
    end

    subgraph "Merged Result (prod)"
        MERGED["Final Plugins<br/>1. CORS (Base)<br/>2. JWT Auth (Gateway)<br/>3. Rate Limit (Stage Override)<br/>Rate: 10 req/min"]
    end

    BASE_CORS --> GW_RATE
    BASE_CORS --> GW_AUTH
    GW_RATE --> PROD_RATE
    GW_AUTH --> MERGED
    PROD_RATE --> MERGED

    style BASE_CORS fill:#e3f2fd
    style GW_RATE fill:#fff3e0
    style GW_AUTH fill:#fff3e0
    style PROD_RATE fill:#ffebee
    style MERGED fill:#e8f5e9
```

**예시: prod 환경에서 rate-limit override**:

```yaml
# Application 설정
plugins:
  rate-limit:
    rate: 100  # Gateway 전역: 100 req/min

# prod Stage 설정
plugins:
  rate-limit:
    rate: 10   # prod만 10 req/min으로 Override

# 결과 (myapp123-prod-plugins)
plugins:
  - name: cors  # Base
  - name: rate-limit
    config:
      rate: 10  # ✅ Stage가 Gateway override
```

---

## CloudFunction: 동적 실행 아키텍처

### 왜 Function마다 Route를 만들지 않는가?

**Route 폭발 문제**:
```
100개 Function × 3 Environments = 300개 Routes
1000개 Function × 3 Environments = 3000개 Routes ❌

→ APISIX 성능 저하
→ Kubernetes API 부하
→ 배포 시간 증가
```

**우리의 접근: Runtime 동적 실행**:
```
Gateway당 4개 Routes (고정)
  - Base Route: /*
  - dev Route: /dev/*
  - staging Route: /staging/*
  - prod Route: /prod/*

Function 1000개 추가해도 → Route는 여전히 4개 ✅
```

### Request Processing Flow

```mermaid
sequenceDiagram
    participant Client
    participant APISIX as APISIX Gateway
    participant Plugin as Plugin Chain
    participant K8s as Kubernetes Service
    participant Runtime as Runtime Pod
    participant MongoDB
    participant Function as CloudFunction

    Client->>APISIX: GET /dev/user/me
    Note over APISIX: Host: myapp123.api.imprun.dev

    APISIX->>APISIX: Route Matching<br/>1. myapp123-dev (priority 10) ✓<br/>2. myapp123-base (skip)

    APISIX->>Plugin: Execute Plugins
    Plugin->>Plugin: 1. CORS check ✓
    Plugin->>Plugin: 2. Rate Limit check ✓
    Plugin->>Plugin: 3. JWT Auth check ✓
    Plugin-->>APISIX: All passed

    APISIX->>K8s: Forward to Upstream<br/>myapp123-service:8000
    K8s->>Runtime: Load Balance to Pod<br/>myapp123-7d8f5b9c4-xk2jl

    Runtime->>Runtime: Parse path<br/>stage=dev<br/>func=user/me
    Runtime->>MongoDB: findOne({<br/>  name: "dev/user/me"<br/>})
    MongoDB-->>Runtime: CloudFunction document

    Runtime->>Runtime: Compile & Cache<br/>function code
    Runtime->>Function: Execute handler(req, res)

    Function->>MongoDB: Query users collection
    MongoDB-->>Function: User data
    Function-->>Runtime: res.json(user)

    Runtime-->>K8s: HTTP 200 OK
    K8s-->>APISIX: Response
    APISIX-->>Client: { "id": "123", "name": "John" }

    Note over Client,Function: Total Latency: ~5ms (cached)
```

**처리 단계 요약**:
1. **Route Matching**: Priority 기반 선택 (Stage > Base)
2. **Plugin Execution**: 계층적 Plugin 적용 (CORS → Auth → Rate Limit)
3. **Upstream Routing**: Kubernetes Service Discovery
4. **Dynamic Function Lookup**: MongoDB에서 코드 조회 + 캐싱
5. **Function Execution**: 사용자 코드 실행 + DB 접근
6. **Response**: JSON 응답 반환

### MongoDB Schema

```javascript
// CloudFunction Collection
{
  _id: ObjectId("..."),
  gatewayId: "myapp123",
  name: "dev/user/me",      // stage prefix 포함
  baseName: "user/me",       // stage 독립적
  code: `
    export default async (req, res) => {
      const user = await db.collection('users')
        .findOne({ id: req.query.id })
      return res.json(user)
    }
  `,
  entrypoint: "index.ts",
  files: {
    "index.ts": "...",
    "utils.ts": "..."
  },
  createdAt: ISODate("2025-10-29T10:00:00Z"),
  updatedAt: ISODate("2025-10-29T10:00:00Z")
}
```

**배포 프로세스**:
```
1. Web Console에서 코드 작성
2. API Server로 POST /v1/apps/myapp123/functions
3. MongoDB에 저장 (CloudFunction document)
4. ✅ 끝! APISIX 변경 없음

URL 즉시 활성화:
  https://myapp123.api.imprun.dev/dev/user/me
```

---

## 성능 최적화 전략

### 1. Function Code Caching

**문제**: 매 요청마다 MongoDB 조회 + 코드 컴파일은 비효율

**해결**: 메모리 캐시 + Watch Pattern

```typescript
// Runtime: Function 캐시
class FunctionCache {
  private cache = new Map<string, CompiledFunction>()
  private mongodb: Db

  async getFunction(name: string): Promise<CompiledFunction> {
    // 캐시 확인
    if (this.cache.has(name)) {
      return this.cache.get(name)!
    }

    // MongoDB 조회
    const func = await this.mongodb
      .collection('CloudFunction')
      .findOne({ name })

    if (!func) throw new Error('Function not found')

    // 코드 컴파일 (vm2 사용)
    const compiled = compileFunction(func.code)

    // 캐시 저장
    this.cache.set(name, compiled)

    return compiled
  }

  // MongoDB Change Stream으로 캐시 무효화
  watchChanges() {
    const changeStream = this.mongodb
      .collection('CloudFunction')
      .watch()

    changeStream.on('change', (change) => {
      if (change.operationType === 'update' ||
          change.operationType === 'replace') {
        const name = change.fullDocument.name
        this.cache.delete(name)  // 캐시 무효화
      }
    })
  }
}
```

**성능 개선**:
- 캐시 미스: ~15ms (MongoDB + compile)
- 캐시 히트: ~0.5ms (메모리 조회만)
- 무효화: MongoDB Change Stream (실시간)

### 2. APISIX Route Priority 최적화

**문제**: Stage Route와 Base Route가 모두 매칭될 수 있음

**해결**: Priority 기반 매칭

```yaml
# ✅ 올바른 설정
Stage Routes (dev, staging, prod):
  priority: 10
  paths: [/dev/*, /staging/*, /prod/*]

Base Route:
  priority: 1
  paths: [/*]

# APISIX 매칭 로직
1. priority 높은 순서대로 검사
2. 첫 번째 매칭 Route 선택
3. 나머지는 무시

Request: /dev/user/me
  → myapp123-dev (priority 10) ✓ 매칭
  → myapp123-base는 검사하지 않음 (이미 매칭됨)
```

### 3. Plugin Config 재사용

**문제**: Plugin 변경 시 Route 전체 재생성?

**해결**: ApisixPluginConfig 분리

```yaml
# Route는 그대로, PluginConfig만 업데이트
apiVersion: apisix.apache.org/v2
kind: ApisixPluginConfig
metadata:
  name: myapp123-dev-plugins
spec:
  plugins:
    - name: rate-limit
      config:
        rate: 50  # 100 → 50 변경

# Route는 pluginConfigName만 참조
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
...
spec:
  http:
    - pluginConfigName: myapp123-dev-plugins  # 참조만
      pluginConfigNamespace: myapp123
```

**장점**:
- Plugin 변경 시 Route 재생성 불필요
- APISIX reload 없이 Plugin 동적 업데이트
- GitOps 친화적 (Route와 Plugin 분리)

---

## 멀티 테넌시 격리 전략

### Namespace 기반 격리

```mermaid
graph TB
    subgraph "Kubernetes Cluster"
        subgraph "Namespace: app-abc123"
            SVC1["Service<br/>abc123"]
            DEPLOY1["Deployment<br/>abc123"]
            ROUTE1_DEV["ApisixRoute<br/>abc123-dev"]
            ROUTE1_STG["ApisixRoute<br/>abc123-staging"]
            ROUTE1_PROD["ApisixRoute<br/>abc123-prod"]
            PLUGIN1["ApisixPluginConfig<br/>abc123-*"]
            PVC1["PVC<br/>abc123-data"]
            SA1["ServiceAccount<br/>abc123-sa"]
        end

        subgraph "Namespace: app-xyz789"
            SVC2["Service<br/>xyz789"]
            DEPLOY2["Deployment<br/>xyz789"]
            ROUTE2_DEV["ApisixRoute<br/>xyz789-dev"]
            ROUTE2_STG["ApisixRoute<br/>xyz789-staging"]
            ROUTE2_PROD["ApisixRoute<br/>xyz789-prod"]
            PLUGIN2["ApisixPluginConfig<br/>xyz789-*"]
            PVC2["PVC<br/>xyz789-data"]
            SA2["ServiceAccount<br/>xyz789-sa"]
        end

        NP1["NetworkPolicy<br/>Deny cross-namespace"]
        RQ1["ResourceQuota<br/>CPU: 2, RAM: 4Gi"]
        RQ2["ResourceQuota<br/>CPU: 2, RAM: 4Gi"]
    end

    DEPLOY1 --> SVC1
    DEPLOY1 --> PVC1
    DEPLOY1 --> SA1
    SVC1 --> ROUTE1_DEV
    SVC1 --> ROUTE1_STG
    SVC1 --> ROUTE1_PROD

    DEPLOY2 --> SVC2
    DEPLOY2 --> PVC2
    DEPLOY2 --> SA2
    SVC2 --> ROUTE2_DEV
    SVC2 --> ROUTE2_STG
    SVC2 --> ROUTE2_PROD

    NP1 -.제약.-> DEPLOY1
    NP1 -.제약.-> DEPLOY2
    RQ1 -.할당.-> DEPLOY1
    RQ2 -.할당.-> DEPLOY2

    style SVC1 fill:#e3f2fd
    style SVC2 fill:#e3f2fd
    style DEPLOY1 fill:#fff3e0
    style DEPLOY2 fill:#fff3e0
    style NP1 fill:#ffebee
    style RQ1 fill:#f3e5f5
    style RQ2 fill:#f3e5f5
```

**격리 수준**:
1. **Network**: Namespace Network Policy (교차 접근 차단)
2. **Compute**: ResourceQuota, LimitRange (리소스 격리)
3. **Storage**: PVC per namespace (데이터 격리)
4. **RBAC**: ServiceAccount per namespace (권한 격리)
5. **Domain**: 독립 도메인 (abc123.api.imprun.dev vs xyz789.api.imprun.dev)

### Domain 격리

```
Application abc123:
  - 기본: abc123.api.imprun.dev
  - 커스텀: api.example.com (optional)

Application xyz789:
  - 기본: xyz789.api.imprun.dev
  - 커스텀: api.another.com (optional)
```

**Custom Domain 지원**:

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: abc123-dev
spec:
  http:
    - match:
        hosts:
          - abc123.api.imprun.dev   # 기본 도메인
          - api.example.com         # 커스텀 도메인
        paths:
          - /dev/*
```

---

## 운영 노하우

### 1. Zero-downtime Plugin Update

**문제**: Plugin 변경 시 Traffic 끊김?

**해결**: APISIX는 Plugin 변경 시 자동으로 Graceful Reload

```typescript
// NestJS Service: Plugin 업데이트
async updateStagePlugins(
  stage: Stage,
  newPlugins: Record<string, any>
) {
  // 1. MongoDB 업데이트
  await this.stageRepo.update(stage.id, { plugins: newPlugins })

  // 2. ApisixPluginConfig 업데이트 (Kubernetes API)
  await this.stageRouteService.updatePluginConfig(
    stage.region,
    stage.namespace,
    stage,
    gateway.plugins
  )

  // ✅ APISIX가 자동으로 reload (0.1초 내)
  // ✅ 기존 연결은 유지 (graceful)
}
```

### 2. Observability: Request Tracing

**APISIX Plugin을 활용한 Tracing**:

```yaml
plugins:
  - name: zipkin
    enable: true
    config:
      endpoint: http://jaeger-collector:9411/api/v2/spans
      sample_ratio: 1.0
      service_name: myapp123-dev
```

**추적 가능한 구간**:
1. APISIX Ingress → Zipkin Span
2. Runtime Function → OpenTelemetry SDK
3. Database Query → MongoDB Profiler

### 3. 점진적 배포 (Canary Release)

**APISIX Traffic Split Plugin 활용**:

```yaml
# staging → prod 배포 시 10% 트래픽만 prod로
plugins:
  - name: traffic-split
    config:
      rules:
        - weighted_upstreams:
            - upstream:
                name: myapp123-prod-v2
                weight: 10  # 10% 트래픽
            - upstream:
                name: myapp123-prod-v1
                weight: 90  # 90% 트래픽
```

### 4. 비용 최적화: Idle Application Sleep

**문제**: Application이 트래픽 없을 때도 Pod 유지?

**해결**: APISIX + Knative 또는 Custom Scaler

```yaml
# APISIX에서 헤더 추가
plugins:
  - name: proxy-rewrite
    config:
      headers:
        set:
          X-Imprun-Wake: "true"  # Wake signal

# Runtime에서 처리
if (noTrafficFor(30minutes)) {
  scaleDown()  // Pod 0으로
}

if (requestArrives) {
  scaleUp()    // Pod 1로 (20초 소요)
  return 503 Retry-After: 20  // Client에게 재시도 요청
}
```

---

## 실전 배포: Helm Chart

### Chart Structure

```
k8s/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── namespace.yaml
    ├── service.yaml
    ├── deployment.yaml
    ├── apisix-route-base.yaml
    └── apisix-route-stage.yaml (동적 생성)
```

### Dynamic Stage Route 생성

```yaml
# templates/apisix-route-stage.yaml
{{- range $stage := list "dev" "staging" "prod" }}
{{- if index $.Values.stages $stage "enabled" }}
---
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: {{ $.Values.gatewayId }}-{{ $stage }}
  namespace: {{ $.Values.namespace }}
  labels:
    imprun.dev/gatewayId: {{ $.Values.gatewayId }}
    imprun.dev/stage: {{ $stage }}
spec:
  http:
    - name: {{ $stage }}-route
      match:
        hosts:
          - {{ $.Values.gatewayId }}.api.imprun.dev
          {{- range $.Values.customDomains }}
          - {{ . }}
          {{- end }}
        paths:
          - /{{ $stage }}/*
      priority: 10
      backends:
        - serviceName: {{ $.Values.gatewayId }}
          servicePort: 8000
      pluginConfigName: {{ $.Values.gatewayId }}-{{ $stage }}-plugins
      pluginConfigNamespace: {{ $.Values.namespace }}
{{- end }}
{{- end }}
```

### values.yaml

```yaml
gatewayId: myapp123
namespace: myapp123

customDomains:
  - api.example.com

stages:
  dev:
    enabled: true
    plugins:
      ip-restriction:
        whitelist:
          - 10.0.0.0/8
  staging:
    enabled: true
    plugins:
      jwt-auth:
        secret: staging_secret
  prod:
    enabled: true
    plugins:
      jwt-auth:
        secret: prod_secret
      rate-limit:
        rate: 10
        time_window: 60

gateway:
  plugins:
    rate-limit:
      rate: 100
      time_window: 60
    cors:
      allow_origins: "*"
```

---

## 성능 벤치마크

### 테스트 환경
- Kubernetes: 3 nodes (4 CPU, 8GB each)
- APISIX: 2 replicas
- Runtime: 1 pod per application
- MongoDB: 3-replica StatefulSet

### 결과

| 지표 | 값 |
|------|-----|
| **Cold Start** | 50ms (캐시 미스) |
| **Warm Request** | 5ms (캐시 히트) |
| **APISIX Overhead** | 1-2ms |
| **Plugin Execution** | 3-5ms (JWT + Rate Limit) |
| **Route Matching** | <1ms (Priority 기반) |
| **Max Functions** | 10,000+ (Route 수와 무관) |
| **Throughput** | 5,000 req/s (per runtime pod) |

**Function 수 증가에 따른 성능**:
```
100 Functions:
  - APISIX Routes: 4개
  - Latency: 5ms (p50), 15ms (p99)

1,000 Functions:
  - APISIX Routes: 4개 (동일)
  - Latency: 5ms (p50), 18ms (p99)
  - 차이: MongoDB 인덱스 영향 (+3ms p99)

10,000 Functions:
  - APISIX Routes: 4개 (동일)
  - Latency: 6ms (p50), 25ms (p99)
  - 차이: MongoDB 샤딩 필요
```

---

## 배운 교훈 (Lessons Learned)

### ✅ 잘한 결정

1. **Route 수 최소화**: Function 개수와 무관하게 확장 가능
2. **Priority 기반 매칭**: 간단하고 명확한 라우팅 로직
3. **PluginConfig 분리**: Route와 Plugin 독립 관리
4. **Namespace 격리**: 멀티 테넌시 보안 강화
5. **Kubernetes Native**: CRD 활용으로 GitOps 친화적

### ⚠️ 개선이 필요한 부분

1. **Function별 Plugin**: 현재는 Environment까지만 지원
   - 해결 방향: Runtime에서 Function metadata 확인 후 Plugin 적용
2. **Cold Start 개선**: 첫 요청 시 ~50ms
   - 해결 방향: Function 코드 preload, Warm pool 유지
3. **MongoDB 부하**: Function 1만 개 이상 시 조회 느림
   - 해결 방향: Redis 캐시 레이어 추가
4. **APISIX CRD 동기화**: Kubernetes API 호출 많음
   - 해결 방향: Operator 패턴으로 전환

### 🔄 다시 설계한다면

1. **Knative Serving 검토**: Scale-to-zero, Revision 관리
2. **Istio VirtualService 고려**: 더 풍부한 Traffic Management
3. **Envoy Filter**: Function별 Plugin을 Envoy에서 처리
4. **eBPF**: Kernel 레벨 라우팅으로 Latency 감소

---

## 마무리

### 핵심 요약

Apache APISIX로 멀티 테넌트 서버리스 플랫폼을 구축하면서, **Environment 단위 Route + Runtime 동적 실행** 패턴을 선택했습니다. Function 1000개여도 Route는 4개만 생성되며, Plugin은 계층적으로 상속됩니다.

### 언제 사용하나?

**이 아키텍처 권장:**
- ✅ 서버리스 Function 플랫폼 구축 시
- ✅ 멀티 테넌트 API Gateway 필요 시
- ✅ Environment별 Plugin 설정 다를 때
- ✅ Function 개수가 수백 개 이상 예상될 때

**대안 고려:**
- ✅ Knative Serving: Scale-to-zero 필요 시
- ✅ Istio: Service Mesh 기능 필요 시
- ✅ Kong Gateway: Enterprise 기능 필요 시

### 실제 적용 결과

**imprun.dev 환경:**
- ✅ Application 100개, Function 1000개 운영 중
- ✅ APISIX Route 410개 (4 × 100 + Base Routes 10개)
- ✅ Function 추가 시 APISIX 변경 없음
- ✅ 배포 시간: ~1초 (MongoDB 업데이트만)

**운영 경험:**
- 설정 시간: Helm Chart로 30분 (초기 1회)
- Route 생성 시간: ~2초 (per Application)
- Plugin 변경 시간: ~1초 (APISIX Graceful Reload)
- 만족도: 매우 높음 😊 (확장성, 간결성)

### 적용 가능한 케이스

이 아키텍처는 다음과 같은 플랫폼에 적합합니다:

- ✅ **서버리스 Function 플랫폼**: AWS Lambda, Vercel Functions 대안
- ✅ **멀티 테넌트 API Gateway**: 테넌트당 독립 Gateway 필요
- ✅ **마이크로서비스 플랫폼**: Environment별 배포 필요
- ✅ **Low-code 플랫폼**: 사용자가 코드 작성하는 플랫폼
- ✅ **API Marketplace**: Function을 상품으로 판매

### 오픈소스 기여

이 아키텍처는 **imprun.dev** 오픈소스 프로젝트에서 실제로 사용 중입니다.

**GitHub**: [https://github.com/imprun/imprun](https://github.com/imprun/imprun)

**기여 환영**:
- 🐛 버그 리포트
- 💡 아키텍처 개선 아이디어
- 📚 문서 개선
- ⭐ Star 눌러주기

---

## 참고 자료

### Apache APISIX
- [공식 문서](https://apisix.apache.org/docs/)
- [Kubernetes Ingress Controller](https://apisix.apache.org/docs/ingress-controller/)
- [Plugin 개발 가이드](https://apisix.apache.org/docs/apisix/plugin-develop/)

### Kubernetes Patterns
- [Multi-tenancy Best Practices](https://kubernetes.io/docs/concepts/security/multi-tenancy/)
- [CRD Design](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)

### 관련 프로젝트
- [Knative Serving](https://knative.dev/docs/serving/)
- [OpenFaaS](https://www.openfaas.com/)
- [Fission](https://fission.io/)

---

---

**태그:** #ApacheAPISIX #Kubernetes #Serverless #APIGateway #MultiTenant #Architecture

**저자:** imprun.dev 팀
**저장소:** [github.com/imprun/imprun](https://github.com/imprun/imprun)

---

> "Route는 최소화하고, 유연성은 최대화하라"

🤖 *이 블로그는 실제 프로덕션 환경에서 Apache APISIX를 운영한 경험을 바탕으로 작성되었습니다.*

**질문이나 피드백이 있다면 GitHub Discussion에서 공유해주세요!**

*이 글이 도움이 되셨다면 ⭐ Star와 공유 부탁드립니다!*
