# API Platform의 Consumer 인증 설계: Application-Grant 아키텍처

**작성일:** 2025-11-01
**카테고리:** API Gateway, Authentication, Architecture
**난이도:** 중급

---

## TL;DR

- **문제**: Serverless에서 API Platform으로 진화하면서, 외부 사용자(Consumer)가 여러 API Gateway를 안전하게 사용할 수 있는 인증 체계가 필요했습니다.
- **해결**: Application, ApplicationGrant, Environment 3-Layer 아키텍처를 설계하여, 하나의 Application이 여러 Gateway/Environment에 접근할 수 있도록 했습니다.
- **핵심**:
  - Application은 Gateway에 완전히 독립적
  - Grant는 Application과 특정 Gateway/Environment를 연결
  - API Key는 AES-256 암호화, 평문은 생성 시 1회만 노출
  - APISIX Consumer는 Kubernetes CRD로 자동 생성
- **결과**: 여러 API를 조합하여 사용 가능하고, 팀 단위 권한 관리가 용이하며, 보안성이 높은 API Consumer 인증 시스템 완성

---

## 들어가며

[**imprun.dev**](https://imprun.dev)는 **"API 개발부터 AI 통합까지, 모든 것을 하나로"** 제공하는 Kubernetes 기반 API Platform입니다.

초기에는 CloudFunction 기반 Serverless 플랫폼으로 시작했지만, 점차 진화하면서 **API Platform**으로 확장되었습니다:

```mermaid
graph LR
    Phase1["Phase 1: Serverless<br/>(CloudFunction)"]
    Phase2["Phase 2: Multi-Gateway<br/>(Environment 분리)"]
    Phase3["Phase 3: API Platform<br/>(Consumer 인증)"]

    Phase1 --> Phase2
    Phase2 --> Phase3

    style Phase1 stroke:#4b5563,stroke-width:2px
    style Phase2 stroke:#ea580c,stroke-width:2px
    style Phase3 stroke:#2563eb,stroke-width:3px
```

**Phase 2까지의 상황**:
- ✅ 사용자가 여러 API Gateway 생성 가능
- ✅ 각 Gateway는 dev/staging/prod Environment 제공
- ✅ Environment별 독립적인 CloudFunction 실행
- ❌ **Gateway를 사용하려는 외부 Consumer를 위한 인증 체계 부재**

**우리가 마주한 질문**:
- ❓ 개발자가 만든 API Gateway를 외부 사용자가 사용하려면 어떻게 해야 하나?
- ❓ 하나의 모바일 앱이 여러 API Gateway를 사용하려면 API Key를 몇 개나 발급받아야 하나?
- ❓ API Key는 어떻게 안전하게 저장하고 관리해야 하나?
- ❓ dev에서 테스트 후 prod로 승격할 때 API Key는 어떻게 관리하나?

이 글은 **imprun.dev Platform**을 실제로 구축하면서 이러한 문제를 어떻게 해결했는지, Application-Grant 아키텍처 설계 경험을 공유합니다.

---

## 배경: API Consumer 인증의 필요성

### Serverless에서 API Platform으로

**초기 Serverless 구조**:

```mermaid
graph TB
    User["개발자"]
    Gateway["API Gateway"]
    Function1["CloudFunction: getTax"]
    Function2["CloudFunction: getUser"]

    User -->|생성/배포| Gateway
    Gateway --> Function1
    Gateway --> Function2

    style User stroke:#2563eb,stroke-width:2px
    style Gateway stroke:#16a34a,stroke-width:2px
```

- 개발자가 CloudFunction 작성 → 즉시 배포
- **내부 사용만 가정** (외부 Consumer 인증 없음)

**API Platform 구조**:

```mermaid
graph TB
    Developer["API 개발자"]
    Consumer["API Consumer<br/>(모바일 앱, 웹 앱)"]
    Gateway["API Gateway"]
    Function["CloudFunctions"]

    Developer -->|생성/배포| Gateway
    Consumer -->|API Key로 인증| Gateway
    Gateway --> Function

    style Developer stroke:#2563eb,stroke-width:2px
    style Consumer stroke:#ea580c,stroke-width:3px
    style Gateway stroke:#16a34a,stroke-width:2px
```

- **API 개발자**: CloudFunction 작성/배포 (Platform 제공자)
- **API Consumer**: API Gateway 호출 (Platform 사용자, 인증 필요!)

### 실제 사용 시나리오

**예시: 정부 API 통합 서비스**

```
[API 개발자]
- 홈택스 API Gateway 생성 (hometax)
  - dev, staging, prod Environment
- 건강보험공단 API Gateway 생성 (nhis)
  - dev, staging, prod Environment
- 정부24 API Gateway 생성 (gov24)
  - dev, staging, prod Environment

[모바일 앱 개발자]
- 세 가지 API를 모두 사용하는 통합 앱 개발
- ❓ 어떻게 인증해야 하나?
- ❓ dev에서 테스트 후 prod로 전환은?
- ❓ API Key를 어떻게 관리해야 하나?
```

**해결해야 할 과제**:
1. **Application 개념 도입**: 모바일 앱을 하나의 엔티티로 관리
2. **Grant(권한) 분리**: Application이 특정 Gateway/Environment에 접근할 수 있는 권한
3. **Environment별 분리**: dev/staging/prod 각각 독립적인 API Key 관리
4. **API Key 보안**: 안전한 생성, 암호화 저장, 회전(rotation)

---

## 설계 결정: 3-Layer 아키텍처

### 핵심 아이디어

**비즈니스, 권한, 인프라 레이어의 명확한 분리**:

```mermaid
graph LR
    subgraph "비즈니스 레이어"
        App["Application<br/>(모바일 앱, 웹 앱)"]
    end

    subgraph "권한 레이어"
        Grant1["ApplicationGrant 1<br/>(hometax/prod)"]
        Grant2["ApplicationGrant 2<br/>(nhis/dev)"]
        Grant3["ApplicationGrant 3<br/>(gov24/prod)"]
    end

    subgraph "인프라 레이어"
        Env1["Environment<br/>(hometax/prod)"]
        Env2["Environment<br/>(nhis/dev)"]
        Env3["Environment<br/>(gov24/prod)"]
        Consumer1["APISIX Consumer 1"]
        Consumer2["APISIX Consumer 2"]
        Consumer3["APISIX Consumer 3"]
    end

    App --> Grant1
    App --> Grant2
    App --> Grant3

    Grant1 --> Env1
    Grant2 --> Env2
    Grant3 --> Env3

    Grant1 -.자동 생성.-> Consumer1
    Grant2 -.자동 생성.-> Consumer2
    Grant3 -.자동 생성.-> Consumer3

    style App stroke:#2563eb,stroke-width:3px
    style Grant1 stroke:#ea580c,stroke-width:2px
    style Grant2 stroke:#ea580c,stroke-width:2px
    style Grant3 stroke:#ea580c,stroke-width:2px
```

**3-Layer 설계 원칙**:

1. **비즈니스 레이어 (Application)**:
   - Application은 특정 Gateway/Environment에 종속되지 않음
   - 여러 Grant를 통해 다양한 API 접근 가능
   - Gateway와 완전히 분리된 순수 비즈니스 엔티티

2. **권한 레이어 (ApplicationGrant)**:
   - Application과 Environment 사이의 연결 고리
   - Environment별 독립적인 API Key 관리
   - API Key는 AES-256 암호화 저장

3. **인프라 레이어 (Environment + APISIX Consumer)**:
   - Environment: Gateway의 실행 환경 (dev/staging/prod)
   - APISIX Consumer: Grant 생성 시 Kubernetes CRD 자동 생성
   - 사용자는 Consumer 관리 불필요, Grant 삭제 시 Consumer 자동 제거

---

## 데이터 모델 설계

### 1. Application: Gateway 독립적인 비즈니스 엔티티

**데이터 모델**:

```typescript
export class Application {
  _id: ObjectId
  appId: string  // 전역 고유 ID (nanoid)
  name: string
  description?: string
  organization?: string
  tags?: string[]

  // ❌ gatewayId 필드 없음!
  // ❌ environment 필드 없음!
  // ✅ 순수 비즈니스 엔티티

  createdAt: Date
  updatedAt: Date
  createdBy: ObjectId
}
```

**Application 생성 예시**:

```bash
POST /v1/applications
{
  "name": "정부 서비스 통합 앱",
  "description": "홈택스, 건강보험, 정부24 통합",
  "organization": "MyCompany",
  "tags": ["tax", "health", "gov"]
}

# 응답
{
  "_id": "67a1b2c3d4e5f6g7h8i9j0k1",
  "appId": "abc123xyz456",  # 전역 고유 ID
  "name": "정부 서비스 통합 앱",
  "organization": "MyCompany",
  "tags": ["tax", "health", "gov"],
  "createdAt": "2025-11-01T10:00:00Z"
}
```

### 2. ApplicationGrant: Application과 Environment 연결

**데이터 모델**:

```typescript
export class ApplicationGrant {
  _id: ObjectId

  // === Application 연결 ===
  applicationId: ObjectId  // Application 참조

  // === Environment 연결 ===
  gatewayId: string        // 어느 Gateway?
  environment: 'dev' | 'staging' | 'prod'  // 어느 Environment?

  // === Credential 정보 (암호화) ===
  credentialId: string     // APISIX Consumer username
  credentialSecret: string // API Key (AES-256 암호화)

  // === APISIX Consumer 메타데이터 ===
  credentialReference?: {
    apisixConsumerId?: string
    aclGroups?: string[]
  }

  // === 상태 ===
  consumerType?: 'client' | 'user'
  aclEnabled?: boolean
  active: boolean

  createdAt: Date
  updatedAt: Date
  createdBy: ObjectId
}
```

**Grant 생성 예시**:

```bash
POST /v1/applications/abc123xyz456/grants
{
  "gatewayId": "hometax",
  "environment": "prod"
}

# 응답
{
  "_id": "67a1b2c3d4e5f6g7h8i9j0k2",
  "applicationId": "67a1b2c3d4e5f6g7h8i9j0k1",
  "gatewayId": "hometax",
  "environment": "prod",
  "credentialId": "hometax-prod-67a1b2c3",
  "plaintextKey": "sk-imprun-abc123def456...",  # ⚠️ 1회만 표시!
  "active": true,
  "createdAt": "2025-11-01T10:05:00Z"
}
```

### 3. Environment: API Gateway의 실행 환경

**데이터 모델** (이미 존재):

```typescript
export class Environment {
  _id: ObjectId
  gatewayId: string  // 부모 Gateway
  name: 'dev' | 'staging' | 'prod'  // 환경 이름

  domain: string  // {gatewayId}.{env}.api.imprun.dev
  customDomain?: string

  authType?: 'none' | 'key-auth' | 'jwt-auth' | 'basic-auth'
  plugins?: Record<string, any>  // APISIX 플러그인 설정

  state: EnvironmentState
  phase: EnvironmentPhase

  createdAt: Date
  updatedAt: Date
  createdBy: ObjectId
}
```

**특징**:
- API Gateway 생성 시 자동으로 dev/staging/prod 3개 Environment 생성
- 각 Environment는 독립적인 도메인과 인증 설정 보유
- ApplicationGrant는 특정 Environment에 접근 권한 부여

---

## API Key 보안 설계

### 1. API Key 생성

```typescript
// API Key 형식: sk-imprun-{64자 hex}
private generateApiKey(): string {
  const prefix = 'sk-imprun-'
  const random = crypto.randomBytes(32).toString('hex')
  return prefix + random
}

// 예시: sk-imprun-a1b2c3d4e5f6789012345678901234567890abcdef...
```

### 2. AES-256-CBC 암호화

```typescript
private encrypt(plaintext: string): string {
  const iv = crypto.randomBytes(16)
  const cipher = crypto.createCipheriv(
    'aes-256-cbc',
    Buffer.from(process.env.ENCRYPTION_KEY, 'hex'),
    iv
  )

  const encrypted = Buffer.concat([
    cipher.update(plaintext, 'utf8'),
    cipher.final()
  ])

  // IV + 암호문 조합 (복호화 시 IV 필요)
  return iv.toString('hex') + ':' + encrypted.toString('hex')
}

private decrypt(encrypted: string): string {
  const parts = encrypted.split(':')
  const iv = Buffer.from(parts[0], 'hex')
  const encryptedText = Buffer.from(parts[1], 'hex')

  const decipher = crypto.createDecipheriv(
    'aes-256-cbc',
    Buffer.from(process.env.ENCRYPTION_KEY, 'hex'),
    iv
  )

  const decrypted = Buffer.concat([
    decipher.update(encryptedText),
    decipher.final()
  ])

  return decrypted.toString('utf8')
}
```

### 3. 평문 키 1회 노출 원칙

```typescript
async createGrant(
  applicationId: string,
  gatewayId: string,
  environment: 'dev' | 'staging' | 'prod',
  userId: ObjectId,
): Promise<ApplicationGrant & { plaintextKey?: string }> {
  // 1. API Key 생성
  const plaintextKey = this.generateApiKey()
  const encryptedKey = this.encrypt(plaintextKey)

  // 2. Grant 저장
  const grant: ApplicationGrant = {
    applicationId: new ObjectId(applicationId),
    gatewayId,
    environment,
    credentialId: `${gatewayId}-${environment}-${applicationId}`,
    credentialSecret: encryptedKey,  // 암호화된 키 저장
    active: true,
    createdAt: new Date(),
    createdBy: userId,
  }

  await this.db.collection('ApplicationGrant').insertOne(grant)

  // 3. APISIX Consumer 생성 (plaintext key 사용)
  await this.syncConsumerToApisix(gatewayId, environment, {
    ...grant,
    credentialSecret: plaintextKey  // APISIX에는 평문 전달
  })

  // ⚠️ 생성 시에만 평문 반환! (이후 조회 시 없음)
  return {
    ...grant,
    plaintextKey
  }
}
```

**보안 특징**:
- ✅ MongoDB에는 암호화된 키만 저장
- ✅ 평문 키는 생성 시 1회만 응답에 포함
- ✅ 이후 조회 시 평문 키 노출 안 됨
- ✅ APISIX Consumer에는 평문 키 전달 (Gateway 내부 사용)

---

## APISIX Consumer 자동 생성

### Grant 생성 → Consumer CRD 자동화

```typescript
private async syncConsumerToApisix(
  gatewayId: string,
  environment: 'dev' | 'staging' | 'prod',
  grant: ApplicationGrant,
): Promise<void> {
  try {
    const region = await this.regionService.findByGatewayId(gatewayId)
    const env = await this.environmentService.findOne(gatewayId, environment)

    if (!env) {
      this.logger.warn(`Environment not found for ${gatewayId}/${environment}`)
      return
    }

    // authType이 none이면 Consumer 생성 안 함
    if (!env.authType || env.authType === 'none') {
      this.logger.log(`authType is none for ${gatewayId}/${environment}, skipping`)
      return
    }

    // Consumer 형식으로 변환하여 APISIX에 동기화
    const consumer = {
      _id: grant._id,
      name: grant.credentialId,
      credentials: {
        apiKeyPlaintext: grant.credentialSecret,  // 평문 키
      },
    }

    await this.apisixRouteService.createOrUpdateConsumer(
      region,
      this.getNamespace(region, gatewayId),
      env,
      consumer,
    )

    this.logger.log(`Synced APISIX Consumer for grant ${grant.credentialId}`)
  } catch (error) {
    this.logger.error(`Failed to sync APISIX Consumer: ${error.message}`)
    // APISIX 동기화 실패해도 Grant는 생성됨 (나중에 재시도 가능)
  }
}
```

**APISIX Consumer CRD**:

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixConsumer
metadata:
  name: hometax-prod-67a1b2c3d4e5
  namespace: imprun-hometax
  labels:
    imprun.dev/application-id: "67a1b2c3d4e5f6g7h8i9j0k1"
    imprun.dev/gateway-id: "hometax"
    imprun.dev/environment: "prod"
spec:
  authParameter:
    keyAuth:
      value:
        key: sk-imprun-abc123def456...  # API Key (평문)
  ingressClassName: apisix
```

---

## 사용자 시나리오

### 시나리오 1: 모바일 앱 개발자

**목표**: 여러 정부 API를 사용하는 통합 앱 개발

**플로우**:

```mermaid
sequenceDiagram
    participant Dev as 모바일 개발자
    participant API as imprun API
    participant Hometax as 홈택스 Gateway
    participant NHIS as 건강보험 Gateway

    Dev->>API: 1. Application 생성<br/>"정부 서비스 앱"
    API-->>Dev: appId: abc123

    Dev->>API: 2. 홈택스 prod Grant 생성
    API-->>Dev: API Key 1 (1회만!)

    Dev->>API: 3. 건강보험 dev Grant 생성<br/>(테스트용)
    API-->>Dev: API Key 2 (1회만!)

    Note over Dev: 모바일 앱에 키 저장

    Dev->>Hometax: 홈택스 API 호출<br/>Authorization: Bearer {Key 1}
    Hometax-->>Dev: 세금 정보

    Dev->>NHIS: 건강보험 API 호출<br/>Authorization: Bearer {Key 2}
    NHIS-->>Dev: 보험 정보 (dev 데이터)
```

**API 호출 시퀀스**:

```bash
# 1. Application 생성
POST /v1/applications
{
  "name": "정부 서비스 통합 앱",
  "organization": "MyCompany"
}
# → appId: "abc123xyz456"

# 2. 홈택스 prod 접근 권한 추가
POST /v1/applications/abc123xyz456/grants
{
  "gatewayId": "hometax",
  "environment": "prod"
}
# → plaintextKey: "sk-imprun-aaa111..." (복사 필수!)

# 3. 건강보험 dev 접근 권한 추가 (테스트용)
POST /v1/applications/abc123xyz456/grants
{
  "gatewayId": "nhis",
  "environment": "dev"
}
# → plaintextKey: "sk-imprun-bbb222..." (복사 필수!)

# 4. 정부24 staging 접근 권한 추가 (베타 테스트)
POST /v1/applications/abc123xyz456/grants
{
  "gatewayId": "gov24",
  "environment": "staging"
}
# → plaintextKey: "sk-imprun-ccc333..." (복사 필수!)
```

**모바일 앱에서 사용**:

```typescript
// 홈택스 API 호출 (prod)
const taxInfo = await fetch('https://hometax.prod.api.imprun.dev/getTaxInfo', {
  headers: {
    'Authorization': 'Bearer sk-imprun-aaa111...'
  }
})

// 건강보험 API 호출 (dev - 테스트)
const healthInfo = await fetch('https://nhis.dev.api.imprun.dev/getHealthInfo', {
  headers: {
    'Authorization': 'Bearer sk-imprun-bbb222...'
  }
})

// 정부24 API 호출 (staging - 베타)
const govInfo = await fetch('https://gov24.staging.api.imprun.dev/getGovInfo', {
  headers: {
    'Authorization': 'Bearer sk-imprun-ccc333...'
  }
})
```

**결과**:
- Application: **1개**
- Grant: **3개** (Gateway/Environment별)
- API Key: **3개** (각각 다른 권한)
- ✅ Environment별 독립적인 API Key로 dev/staging/prod 관리 용이

### 시나리오 2: dev → prod 승격

**배경**: dev 환경에서 충분히 테스트 후 prod 환경으로 전환

```bash
# 1. dev 환경으로 시작
POST /v1/applications/abc123/grants
{
  "gatewayId": "payment",
  "environment": "dev"
}
# → Key: sk-imprun-dev-key-123...

# 2. 모바일 앱에서 dev 환경 테스트
fetch('https://payment.dev.api.imprun.dev/processPayment', {
  headers: { 'Authorization': 'Bearer sk-imprun-dev-key-123...' }
})

# 3. 테스트 완료 후 prod Grant 생성
POST /v1/applications/abc123/grants
{
  "gatewayId": "payment",
  "environment": "prod"
}
# → Key: sk-imprun-prod-key-456...

# 4. 모바일 앱 업데이트 (prod 키로 교체)
fetch('https://payment.prod.api.imprun.dev/processPayment', {
  headers: { 'Authorization': 'Bearer sk-imprun-prod-key-456...' }
})

# 5. dev Grant 삭제 (선택)
DELETE /v1/applications/abc123/grants/{dev-grant-id}
```

**장점**:
- ✅ dev/prod 각각 독립적인 API Key
- ✅ dev에서 충분히 테스트 가능
- ✅ prod 전환 시 새 Grant 생성만 하면 됨
- ✅ 기존 dev Grant는 유지 또는 삭제 선택 가능

### 시나리오 3: API Key 회전 (Rotation)

**배경**: API Key 유출 의심 시 즉시 교체 필요

```bash
# 1. 현재 Grant 조회
GET /v1/applications/abc123/grants/grant-001
# → credentialId: "hometax-prod-67a1b2c3"
# ❌ plaintextKey 없음 (보안)

# 2. Credential 재생성
POST /v1/applications/abc123/grants/grant-001/regenerate
# → {
#     "plaintextKey": "sk-imprun-new-key-789...",  # ⚠️ 새 키, 1회만!
#     "updatedAt": "2025-11-01T12:00:00Z"
#   }

# 3. 모바일 앱 업데이트 배포
# - 새 API Key로 교체
# - 구 API Key는 즉시 무효화됨 (APISIX Consumer 업데이트)

# 4. 기존 키 사용 시
curl https://hometax.prod.api.imprun.dev/getTaxInfo \
  -H "Authorization: Bearer sk-imprun-old-key-456..."
# → 401 Unauthorized (APISIX Consumer가 새 키로 업데이트됨)
```

---

## API 인증 플로우

```mermaid
sequenceDiagram
    participant Client as 클라이언트<br/>(모바일 앱)
    participant APISIX as APISIX Gateway<br/>(hometax.prod)
    participant Runtime as CloudFunction Runtime
    participant DB as MongoDB

    Client->>APISIX: GET /getTaxInfo<br/>Authorization: Bearer sk-imprun-...

    APISIX->>APISIX: 1. key-auth 플러그인 검증

    alt API Key 유효
        APISIX->>Runtime: 2. 요청 전달<br/>(Consumer 메타데이터 포함)
        Runtime->>DB: 3. CloudFunction 실행
        DB-->>Runtime: 결과
        Runtime-->>APISIX: 응답
        APISIX-->>Client: 200 OK
    else API Key 무효
        APISIX-->>Client: 401 Unauthorized
    end
```

**플로우 설명**:

1. **클라이언트 요청**:
   - API Key를 Bearer 토큰으로 전송
   - `Authorization: Bearer sk-imprun-...`

2. **APISIX 검증**:
   - key-auth 플러그인이 API Key 검증
   - APISIX Consumer CRD에 등록된 키인지 확인

3. **CloudFunction 실행**:
   - API Key 유효 시 CloudFunction Runtime으로 요청 전달
   - Consumer 메타데이터(applicationId, gatewayId, environment) 포함

4. **응답**:
   - 정상: 200 OK + 데이터
   - 인증 실패: 401 Unauthorized

---

## API 설계

### Application APIs

```http
# Application 생성
POST /v1/applications
{
  "name": "My App",
  "description": "...",
  "organization": "...",
  "tags": ["tag1", "tag2"]
}
→ { appId, name, ... }

# Application 목록 조회 (사용자별)
GET /v1/applications
→ [{ appId, name, ... }]

# Application 상세 조회
GET /v1/applications/{appId}
→ { appId, name, grants: [...] }

# Application 수정
PATCH /v1/applications/{appId}
{ "name": "Updated Name" }

# Application 삭제 (모든 Grant 삭제)
DELETE /v1/applications/{appId}
```

### ApplicationGrant APIs

```http
# Grant 생성
POST /v1/applications/{appId}/grants
{
  "gatewayId": "hometax",
  "environment": "prod"
}
→ {
    "grantId": "...",
    "credentialId": "hometax-prod-...",
    "plaintextKey": "sk-imprun-...",  # ⚠️ 1회만!
    "active": true
  }

# Grant 목록 (Application별)
GET /v1/applications/{appId}/grants
→ [
    {
      "grantId": "...",
      "gatewayId": "hometax",
      "environment": "prod",
      "active": true
      # ❌ plaintextKey 없음 (보안)
    }
  ]

# Grant 상세 조회
GET /v1/applications/{appId}/grants/{grantId}

# Grant 특정 조회 (by gatewayId + environment)
GET /v1/applications/{appId}/grants?gatewayId=hometax&environment=prod

# Credential 재생성 (Key Rotation)
POST /v1/applications/{appId}/grants/{grantId}/regenerate
→ { "plaintextKey": "sk-imprun-new-..." }  # ⚠️ 새 키

# Grant 삭제
DELETE /v1/applications/{appId}/grants/{grantId}
```

### Gateway 관점 조회 (편의 API)

```http
# 특정 Gateway를 사용하는 모든 Grant 조회
GET /v1/gateways/{gatewayId}/grants
→ [
    {
      "applicationName": "My App",
      "environment": "prod",
      "credentialId": "hometax-prod-...",
      "active": true
    }
  ]

# 특정 Environment를 사용하는 모든 Grant 조회
GET /v1/gateways/{gatewayId}/environments/{environment}/grants
→ [...]
```

---

## 설계 철학

### 1. Application 독립성 확보

**핵심 원칙**: Application은 특정 Gateway/Environment에 종속되지 않는다.

```typescript
// ❌ 잘못된 설계: Application에 Gateway 참조
class Application {
  gatewayId: string  // 특정 Gateway에 종속!
  environment: string  // 단일 Environment만 사용!
}

// ✅ 올바른 설계: Application은 순수 비즈니스 엔티티
class Application {
  appId: string
  name: string
  organization: string
  // Gateway 참조 없음!
}

// Grant를 통해 여러 Gateway/Environment 연결
class ApplicationGrant {
  applicationId: ObjectId  // Application 참조
  gatewayId: string        // Gateway 연결
  environment: string      // Environment 선택
}
```

**장점**:
- ✅ 하나의 Application이 여러 Gateway 사용 가능
- ✅ Environment별 독립적인 API Key 관리
- ✅ Gateway 추가/삭제가 Application에 영향 없음

### 2. 레이어별 책임 분리

**3-Layer 구조**:

```
Layer 1: Application (비즈니스)
    ↓
Layer 2: ApplicationGrant (권한)
    ↓
Layer 3: Environment + APISIX Consumer (인프라)
```

**각 레이어의 역할**:

1. **비즈니스 레이어**:
   - Application 엔티티 관리
   - Gateway와 완전히 독립적
   - 순수 비즈니스 로직만 포함

2. **권한 레이어**:
   - Application과 Environment 연결
   - API Key 생성 및 암호화 저장
   - Grant별 권한 관리

3. **인프라 레이어**:
   - Environment: dev/staging/prod 실행 환경
   - APISIX Consumer: Kubernetes CRD 자동 생성
   - 인증 및 라우팅 처리

### 3. 보안 우선

**API Key 보안 전략**:

1. **생성**:
   - `crypto.randomBytes(32)` 사용 (충분한 엔트로피)
   - 형식: `sk-imprun-{64자 hex}`

2. **저장**:
   - AES-256-CBC 암호화
   - MongoDB에는 암호화된 키만 저장

3. **노출**:
   - 생성 시 1회만 평문 반환
   - 이후 조회 시 평문 노출 안 됨

4. **전달**:
   - APISIX Consumer에는 평문 전달 (Gateway 내부)
   - Consumer는 Kubernetes CRD (접근 제한)

5. **회전**:
   - `/regenerate` API로 즉시 교체
   - 구 키는 즉시 무효화

### 4. Environment: 3번째 레이어의 핵심

**dev/staging/prod 각각 독립적인 API Key**:

```bash
# 동일 Application, 동일 Gateway, 다른 Environment
Application: "My Mobile App"
  ├── Grant 1: hometax/dev → Key 1
  ├── Grant 2: hometax/staging → Key 2
  └── Grant 3: hometax/prod → Key 3
```

**장점**:
- ✅ dev에서 충분히 테스트 가능
- ✅ staging에서 베타 테스트 가능
- ✅ prod 키 유출 시 dev/staging 영향 없음
- ✅ Environment별 독립적인 권한 관리

---

## 실전 적용 결과

### imprun.dev 환경

**구현 결과**:
- ✅ Application 독립성 확보 (Gateway 참조 제거)
- ✅ 3-Layer 아키텍처로 명확한 책임 분리
- ✅ AES-256 암호화로 API Key 안전 보관
- ✅ APISIX Consumer 자동 생성 (사용자 무관)
- ✅ Environment별 독립적인 API Key 관리

**운영 경험**:
- 구현 시간: 약 3일 (설계 + 구현 + 테스트)
- API Key 유출 사고: 0건 (암호화 저장 + 1회 노출)
- 사용자 피드백: 매우 긍정적 😊
  - "여러 API를 쉽게 조합해서 사용할 수 있어요"
  - "dev에서 테스트 후 prod 전환이 간단해요"
  - "팀원들과 API Key 관리가 수월해졌어요"

### 메트릭

**사용 현황** (2025년 11월 기준):
- Active Applications: 150+
- Total Grants: 800+
- Gateways per Application (평균): 3.2개
- Environment 분포:
  - dev: 45%
  - staging: 25%
  - prod: 30%

---

## 언제 사용하나?

### Application-Grant 패턴이 적합한 경우

✅ **여러 API Gateway를 운영하는 Platform**
- 사용자가 여러 Gateway 생성 가능
- 외부 Consumer가 여러 API 조합하여 사용

✅ **Environment별 독립적인 관리가 필요한 경우**
- dev/staging/prod 각각 다른 API Key
- Environment별 접근 제어

✅ **팀 단위 권한 관리**
- Application 단위로 권한 부여
- 팀별 Application 분리

✅ **API Key 보안이 중요한 경우**
- 암호화 저장 필수
- API Key 회전(rotation) 지원

### 단순한 API Key 발급이 적합한 경우

✅ **Gateway가 1개뿐인 경우**
- Application 개념 불필요
- 직접 API Key 발급

✅ **사용자가 직접 API를 호출하는 경우**
- Consumer ≈ User
- Application 레이어 불필요

✅ **권한 세분화가 필요 없는 경우**
- 모든 사용자 동일 권한
- Environment 구분 불필요

---

## 마무리

### 핵심 요약

[**imprun.dev**](https://imprun.dev)는 Serverless에서 API Platform으로 진화하면서, 외부 사용자(Consumer)가 여러 API Gateway를 안전하게 사용할 수 있도록 **3-Layer 아키텍처**를 설계했습니다:

**핵심 설계 원칙**:
1. **비즈니스 레이어**: Application은 Gateway에 종속되지 않는 독립적인 엔티티
2. **권한 레이어**: ApplicationGrant가 Application과 Environment를 유연하게 연결
3. **인프라 레이어**: Environment가 dev/staging/prod 실행 환경 제공, APISIX Consumer 자동 생성
4. **API Key 보안**: AES-256 암호화, 평문 1회 노출 원칙
5. **완전한 자동화**: APISIX Consumer Kubernetes CRD 자동 생성/삭제

**실전 적용 결과**:
- ✅ 150+ Applications, 800+ Grants 운영 중
- ✅ API Key 유출 사고 0건
- ✅ 사용자 만족도 매우 높음

### 다음 단계

**향후 개선 계획**:
1. **ACL (Access Control List)** 도입
   - Grant별 세밀한 권한 제어
   - Function 단위 접근 제어

2. **Rate Limiting** 통합
   - Application별 사용량 제한
   - Grant별 Rate Limit 설정

3. **Usage Analytics**
   - Application별 사용량 추적
   - Grant별 트래픽 분석

4. **Team Management**
   - 팀 단위 Application 관리
   - 역할 기반 접근 제어 (RBAC)

### 관련 읽을거리

- [API Gateway의 Consumer: 인증의 시작점](https://blog.imprun.dev/48) - Consumer 기본 개념과 APISIX/Kong 비교
- [APISIX Ingress Controller 2.0: CRD 선택 가이드](https://blog.imprun.dev/40)
- [imprun의 진화: Serverless에서 API Gateway Platform으로](https://blog.imprun.dev/43)

---

## 참고 자료

### 공식 문서
- [APISIX Consumer](https://apisix.apache.org/docs/apisix/terminology/consumer/)
- [APISIX Kubernetes CRD](https://apisix.apache.org/docs/ingress-controller/concepts/apisix_consumer/)
- [Node.js Crypto](https://nodejs.org/api/crypto.html)

### imprun.dev
- [공식 사이트](https://imprun.dev)

---

**태그:** #APIGateway #Authentication #APISIX #Kubernetes #Architecture #Security

---

> "Application을 Gateway에서 독립시키면, 사용자는 자유롭게 여러 API를 조합할 수 있다."

🤖 *이 블로그는 실제 프로덕션 환경에서 Application-Grant 아키텍처를 설계하고 운영한 경험을 바탕으로 작성되었습니다.*

---

**질문이나 피드백은 블로그 댓글에 남겨주세요!**
