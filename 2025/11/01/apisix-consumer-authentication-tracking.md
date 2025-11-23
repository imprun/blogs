# APISIX Consumer 인증 아키텍처 완벽 이해하기: X-Consumer-Username 헤더와 사용자 추적

> **작성일**: 2025년 11월 1일
> **대상 독자**: 플랫폼 엔지니어, API Gateway 개발자, 백엔드 아키텍트
> **난이도**: 중급
> **주제**: Apache APISIX, Consumer 인증, API Key 관리, 사용량 추적

---

## TL;DR

- **핵심**: APISIX Consumer의 `metadata.name`이 `X-Consumer-Username` 헤더의 소스
- **Multi-auth 패턴**: 하나의 Consumer가 key-auth, basic-auth, jwt-auth를 동시에 가질 수 있음
- **두 개의 Username**: Consumer username (식별) ≠ Credential username (인증)
- **사용량 추적**: `X-Consumer-Username` 헤더로 Backend에서 API 호출 추적 가능
- **Critical Bug 발견**: [imprun](https://imprun.dev)에서 API Key 대신 MongoDB ObjectId를 APISIX에 주입하는 버그 발견

**GitHub**: [imprun.dev](https://github.com/imprun/imprun)

---

## 들어가며

[**imprun.dev**](https://imprun.dev)는 Kubernetes 기반 오픈소스 API Gateway 플랫폼입니다.
CloudFunction으로 REST API를 개발하고, 이를 AI 에이전트의 MCP Tool로 연결할 수 있습니다.

### 우리가 마주한 질문

ClientApp(API Consumer) 생성 플로우를 개선하던 중, APISIX 인증 아키텍처에 대한 근본적인 질문들을 마주했습니다:

- ❓ **왜 별도의 API Key 모듈이 필요한가?** → APISIX가 인증을 처리하는데 왜 서버에서 API Key를 별도 관리?
- ❓ **인증 정보는 어떻게 Backend까지 도달하는가?** → APISIX 인증 성공 후 어떤 정보가 전달?
- ❓ **key-auth, basic-auth, jwt-auth의 차이는?** → 각각 어떻게 다르게 처리?
- ❓ **사용량 추적은 어떻게 구현하는가?** → API 호출 통계와 과금을 위한 구조?

### 검증 과정

**1. 시도: 공식 문서만 읽고 이해**
- ✅ 기본 구조 파악
- ❌ Consumer username과 Credential username 구분 불명확
- ❌ 실제 사용량 추적 구현 방법 부재

**2. 시도: APISIX 소스코드 분석**
- ✅ Consumer 매칭 로직 이해
- ✅ X-Consumer-Username 생성 원리 발견
- ❌ Kubernetes CRD 구조와의 연결 불명확

**3. 최종: imprun 코드베이스 전체 분석** ← **강조**
- ✅ ApisixConsumer CRD v2 구조 완벽 이해
- ✅ metadata.name → X-Consumer-Username 매핑 확인
- ✅ **Critical Bug 발견**: ObjectId 주입 버그
- ✅ 사용량 추적 아키텍처 설계 완성

### 결론

- ✅ **Consumer 식별 원리**: `metadata.name`이 모든 것의 핵심
- ✅ **Multi-auth 지원**: 하나의 Consumer에 여러 인증 방식 동시 적용
- ✅ **Backend 추적 가능**: APISIX 헤더로 사용량 추적 및 과금 구현 가능
- ✅ **실제 버그 발견**: Production 환경에서 작동하지 않던 인증 문제 원인 파악

이 글은 **imprun.dev 플랫폼 구축 경험**을 바탕으로, APISIX Consumer 인증의 작동 원리와 사용자 추적 아키텍처를 상세히 공유합니다.

---

## APISIX Consumer의 핵심: metadata.name

### ApisixConsumer CRD 구조

```yaml
apiVersion: apisix.apache.org/v2
kind: ApisixConsumer
metadata:
  name: mygateway-dev-507f1f77bcf86cd799439011  # ← 이것이 X-Consumer-Username!
  namespace: my-gateway
spec:
  authParameter:
    keyAuth:
      value:
        key: "sk-imprun-abc123..."  # Plaintext API Key
```

**핵심 포인트**: `metadata.name`이 Backend로 전달되는 `X-Consumer-Username` 헤더 값이 됩니다.

### 인증 플로우

```mermaid
sequenceDiagram
    participant Client as 클라이언트
    participant APISIX as APISIX Gateway
    participant Backend as Backend Runtime

    Client->>APISIX: GET /api/hello<br/>X-API-Key: sk-imprun-abc123

    Note over APISIX: Consumer 매칭<br/>모든 Consumer의 authParameter 검색

    APISIX->>APISIX: keyAuth.value.key와 비교

    Note over APISIX: Consumer 발견!<br/>metadata.name: mygateway-dev-507f1f77bcf86cd799439011

    APISIX->>Backend: GET /api/hello<br/>X-Consumer-Username: mygateway-dev-507f1f77bcf86cd799439011<br/>X-Credential-Identifier: sk-imprun-abc123

    Backend-->>APISIX: 응답
    APISIX-->>Client: 응답

    style Client stroke:#2563eb,stroke-width:2px
    style APISIX stroke:#ea580c,stroke-width:3px
    style Backend stroke:#16a34a,stroke-width:2px
```

**동작 순서**:
1. 클라이언트가 `X-API-Key: sk-imprun-abc123...` 헤더로 요청
2. APISIX가 모든 Consumer의 `authParameter.keyAuth.value.key`와 비교
3. 일치하는 Consumer 발견 (예: `mygateway-dev-507f1f77bcf86cd799439011`)
4. **Backend로 요청 전달 시 헤더 추가**:
   ```
   X-Consumer-Username: mygateway-dev-507f1f77bcf86cd799439011
   X-Credential-Identifier: sk-imprun-abc123...
   X-Consumer-Custom-Id: (설정 시)
   ```

---

## 인증 방식별 차이점

### 1. Key-Auth (API Key 인증)

**imprun 현재 구현**:

```typescript
// Consumer 이름 = X-Consumer-Username 헤더 값
const consumerName = `${stage.gatewayId}-${stage.name}-${consumer._id}`
// 예: "mygateway-dev-507f1f77bcf86cd799439011"

const apisixConsumer = {
  kind: 'ApisixConsumer',
  metadata: {
    name: consumerName,  // ← X-Consumer-Username
  },
  spec: {
    authParameter: {
      keyAuth: {
        value: {
          key: consumer.apiKeyId  // ⚠️ 버그: ObjectId 주입!
        }
      }
    }
  }
}
```

**클라이언트 요청**:
```http
GET /api/functions/hello
Host: mygateway.api.imprun.dev
X-API-Key: sk-imprun-abc123...
```

**Backend가 받는 헤더**:
```
X-Consumer-Username: mygateway-dev-507f1f77bcf86cd799439011
X-Credential-Identifier: sk-imprun-abc123...
```

### 2. Basic-Auth (Username/Password 인증)

**Consumer 정의**:
```yaml
metadata:
  name: mygateway-dev-abc123  # ← X-Consumer-Username 소스
spec:
  authParameter:
    basicAuth:
      value:
        username: "mobile-app"  # ← 인증용 username (별개!)
        password: "hashed-password"
```

**클라이언트 요청**:
```http
GET /api/functions/hello
Host: mygateway.api.imprun.dev
Authorization: Basic bW9iaWxlLWFwcDpzZWNyZXQtcGFzcw==
```

**Backend가 받는 헤더**:
```
X-Consumer-Username: mygateway-dev-abc123  ← metadata.name
X-Credential-Identifier: mobile-app        ← basicAuth username
```

### 3. JWT-Auth (JWT 토큰 인증)

**Consumer 정의**:
```yaml
metadata:
  name: mygateway-dev-abc123
spec:
  authParameter:
    jwtAuth:
      value:
        key: "consumer-key"
        secret: "jwt-secret"  # HS256
        algorithm: "HS256"
```

**클라이언트 요청**:
```http
GET /api/functions/hello
Host: mygateway.api.imprun.dev
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Backend가 받는 헤더**:
```
X-Consumer-Username: mygateway-dev-abc123
X-Credential-Identifier: consumer-key
```

---

## 두 개의 Username 개념

이것이 가장 헷갈리는 부분입니다:

| 항목 | 역할 | 예시 | 헤더 |
|------|------|------|------|
| **Consumer username**<br>(`metadata.name`) | APISIX가 Consumer를 식별하는 ID | `mygateway-dev-507f1f77bcf86cd799439011` | `X-Consumer-Username` |
| **Credential username**<br>(`authParameter.basicAuth.value.username`) | HTTP Basic Auth에서 실제 사용하는 username | `mobile-app` | `X-Credential-Identifier` |

### 왜 이렇게 나뉘어져 있을까?

APISIX는 **하나의 Consumer가 여러 인증 방식을 동시에 가질 수 있는 Multi-auth 패턴**을 지원합니다:

```yaml
metadata:
  name: mygateway-dev-abc123  # 동일한 Consumer
spec:
  authParameter:
    keyAuth:
      value:
        key: "sk-imprun-xyz789"
    basicAuth:
      value:
        username: "mobile-app"
        password: "hashed-password"
    jwtAuth:
      value:
        key: "jwt-key"
        secret: "jwt-secret"
```

이 경우 클라이언트가 **어떤 인증 방식을 사용하든** `X-Consumer-Username`은 항상 `mygateway-dev-abc123`으로 동일합니다.

**Multi-auth 인증 로직**:
```mermaid
graph TB
    Request["클라이언트 요청"]
    KeyAuth["key-auth 확인"]
    BasicAuth["basic-auth 확인"]
    JwtAuth["jwt-auth 확인"]
    Success["인증 성공<br/>X-Consumer-Username 추가"]
    Fail["403 Forbidden"]

    Request --> KeyAuth
    Request --> BasicAuth
    Request --> JwtAuth

    KeyAuth -->|통과| Success
    BasicAuth -->|통과| Success
    JwtAuth -->|통과| Success

    KeyAuth -->|실패| BasicAuth
    BasicAuth -->|실패| JwtAuth
    JwtAuth -->|실패| Fail

    style Request stroke:#2563eb,stroke-width:2px
    style Success stroke:#16a34a,stroke-width:3px
    style Fail stroke:#dc2626,stroke-width:3px
```

Backend는 `X-Consumer-Username`을 통해 "이 요청은 동일한 Consumer에서 온 것"을 알 수 있습니다.

---

## imprun의 현재 구현과 발견된 버그

### Critical Bug 발견

**현재 코드** (server/src/gateway/apisix/apisix-route.service.ts:258):

```typescript
const apisixConsumer = {
  kind: 'ApisixConsumer',
  metadata: {
    name: consumerName,  // ✅ 올바른 패턴
  },
  spec: {
    authParameter: {
      keyAuth: {
        value: {
          key: consumer.apiKeyId  // ⚠️ 버그: MongoDB ObjectId!
        }
      }
    }
  }
}
```

**문제점**:
- ❌ `consumer.apiKeyId`는 MongoDB ObjectId (예: `507f1f77bcf86cd799439011`)
- ❌ APISIX는 plaintext API Key를 기대 (예: `sk-imprun-abc123...`)
- ❌ **결과**: 현재 key-auth 인증이 작동하지 않음

**올바른 코드**:
```typescript
const apisixConsumer = {
  spec: {
    authParameter: {
      keyAuth: {
        value: {
          key: consumer.credentials?.apiKeyPlaintext  // ✅ Plaintext key
        }
      }
    }
  }
}
```

### 버그 발견 과정

```mermaid
graph TB
    Question["❓ 왜 별도 API Key 모듈?"]
    Search1["코드 검색: apiKeyId"]
    Found["apisix-route.service.ts:258 발견"]
    Analysis["ObjectId vs Plaintext 비교"]
    Bug["🚨 Critical Bug!"]

    Question --> Search1
    Search1 --> Found
    Found --> Analysis
    Analysis --> Bug

    style Question stroke:#2563eb,stroke-width:2px
    style Bug stroke:#dc2626,stroke-width:3px,stroke-dasharray: 5 5
```

**트러블슈팅 경험**:
1. grep으로 `apiKeyId` 검색
2. APISIX Consumer 동기화 코드 발견
3. APISIX key-auth 문서와 비교
4. MongoDB ObjectId가 주입되고 있음을 확인
5. 실제 API Key는 `apiKey.plaintextKey`에 있음을 발견

---

## 왜 별도의 API Key 모듈이 필요한가?

APISIX가 인증을 처리하는데 왜 imprun 서버에서 API Key를 별도로 관리할까요?

### 1. 보안 강화

**APISIX etcd는 plaintext로 저장**하지만, imprun API Key 모듈은 **SHA-256 해싱**을 사용합니다:

```typescript
// server/src/api-keys/api-keys.service.ts
async create(userId: ObjectId, dto: CreateApiKeyDto) {
  const plaintextKey = `sk-imprun-${generateRandomString(32)}`

  const apiKey = new this.apiKeyModel({
    hash: hashApiKey(plaintextKey),  // SHA-256 해싱
    scopes: dto.scopes,
    userId,
  })

  await apiKey.save()

  return {
    _id: apiKey._id,
    plaintextKey,  // 한 번만 반환
  }
}
```

### 2. 키 재발급 지원

Consumer를 재생성하지 않고도 키를 교체할 수 있습니다.

### 3. 감사 추적 (Audit Trail)

누가, 언제, 어떤 키를 생성/삭제했는지 이력 관리가 가능합니다.

### 4. 고급 기능

```typescript
interface ApiKey {
  scopes: string[]         // 권한 범위
  rateLimit?: number       // 속도 제한
  ipWhitelist?: string[]   // IP 화이트리스트
  expiresAt?: Date         // 만료일
}
```

### 5. 향후 사용량 추적 및 과금

**가장 중요한 이유**: API Key 모듈에는 이미 `updateUsageStats()` 메서드가 구현되어 있습니다:

```typescript
async updateUsageStats(apiKeyId: ObjectId) {
  await this.apiKeyModel.updateOne(
    { _id: apiKeyId },
    {
      $inc: { 'usage.requests': 1 },
      $set: { 'usage.lastUsedAt': new Date() },
    }
  )
}
```

**하지만**: 이 메서드를 호출하는 코드가 **전체 코드베이스에 없습니다**. 즉, 사용량 추적 인프라는 준비되어 있지만 실제로는 사용되지 않습니다.

---

## 사용량 추적 아키텍처 설계

APISIX가 전달하는 `X-Consumer-Username` 헤더를 활용하면 사용량 추적이 가능합니다:

### 아키텍처 다이어그램

```mermaid
graph TB
    Client["클라이언트"]
    APISIX["APISIX Gateway"]
    Runtime["CloudFunction Runtime"]
    ImprunServer["imprun Server<br/>API Key Module"]

    Client -->|"1. X-API-Key: sk-imprun-abc123"| APISIX
    APISIX -->|"2. X-Consumer-Username: mygateway-dev-507f1f77bcf86cd799439011"| Runtime
    Runtime -->|"3. Function 실행"| Runtime
    Runtime -.->|"4. POST /api/v1/api-keys/{id}/usage<br/>(비동기)"| ImprunServer
    ImprunServer -.->|"5. updateUsageStats()"| ImprunServer

    style Client stroke:#2563eb,stroke-width:2px
    style APISIX stroke:#ea580c,stroke-width:3px
    style Runtime stroke:#16a34a,stroke-width:3px
    style ImprunServer stroke:#0891b2,stroke-width:2px
```

### 구현 예시

**Runtime에서 Consumer 정보 추출**:

```typescript
// runtimes/imp-runtime-nodejs/src/handler/invoke.ts
export async function handleInvokeFunction(req: IRequest, res: Response) {
  // 1. APISIX에서 전달받은 Consumer 정보
  const consumerUsername = req.headers['x-consumer-username']
  // "mygateway-dev-507f1f77bcf86cd799439011"

  const credentialId = req.headers['x-credential-identifier']
  // "sk-imprun-abc123..." (key-auth) 또는 "mobile-app" (basic-auth)

  // 2. Consumer ID 추출
  const consumerId = consumerUsername?.split('-').pop()
  // "507f1f77bcf86cd799439011"

  // 3. CloudFunction 실행
  const ctx: FunctionContext = {
    __function_name: name,
    requestId: req.requestId,
    query: req.query,
    body: req.body,
    headers: req.headers,
    method: req.method,
    auth: req['auth'],
    user: req.user,
    consumerId,  // Consumer ID 추가 ✅
  }

  const result = await executeFunction(ctx)

  // 4. 사용량 업데이트 (비동기, Function 실행 결과에 영향 없음)
  if (consumerId) {
    updateApiKeyUsage(new ObjectId(consumerId)).catch(console.error)
  }

  return result
}

async function updateApiKeyUsage(consumerId: ObjectId) {
  // imprun 서버의 사용량 업데이트 API 호출
  await fetch(`${IMPRUN_SERVER_URL}/api/v1/api-keys/${consumerId}/usage`, {
    method: 'POST',
    headers: {
      'X-Internal-Token': INTERNAL_TOKEN,
      'Content-Type': 'application/json',
    },
  })
}
```

**imprun 서버에 사용량 업데이트 API 추가**:

```typescript
// server/src/api-keys/api-keys.controller.ts
@Controller('api/v1/api-keys')
export class ApiKeysController {
  @Post(':id/usage')
  @UseGuards(InternalAuthGuard)  // Runtime에서만 호출 가능
  async updateUsage(@Param('id') id: string) {
    await this.apiKeysService.updateUsageStats(new ObjectId(id))
    return { success: true }
  }
}
```

---

## 구현 계획

### Phase 1: Critical Bug Fix (긴급)

**목표**: APISIX key-auth 인증 작동하도록 버그 수정

**작업 내용**:

1. **StageConsumer 스키마 확장** (environment.entity.ts):
```typescript
export interface StageConsumer {
  _id?: ObjectId
  name: string
  credentials?: {
    // key-auth
    apiKeyId?: ObjectId
    apiKeyPlaintext?: string  // APISIX 동기화용 ✅

    // basic-auth (향후 확장)
    username?: string
    passwordHash?: string

    // jwt-auth (향후 확장)
    jwtKey?: string
    jwtSecret?: string
  }
}
```

2. **ClientApp 생성 시 plaintext 저장** (environment.service.ts):
```typescript
async addClient(gatewayId: string, stageName: string, dto: any, userId: ObjectId) {
  const apiKey = await this.apiKeysService.create(userId, {
    name: `${dto.name}-${stageName}-key`,
    scopes: ['functions.invoke'],
  })

  const newClient = {
    _id: new ObjectId(),
    name: dto.name,
    credentials: {
      apiKeyId: apiKey._id,
      apiKeyPlaintext: apiKey.plaintextKey,  // ✅ Plaintext 저장
    },
  }

  return {
    ...newClient,
    apiKey: apiKey.plaintextKey,
  }
}
```

3. **APISIX Consumer 동기화 수정** (apisix-route.service.ts:258):
```typescript
// Before
key: consumer.apiKeyId  // ❌ MongoDB ObjectId

// After
key: consumer.credentials?.apiKeyPlaintext  // ✅ Plaintext key
```

**예상 작업 시간**: 1-2시간

### Phase 2: Consumer Tracking Infrastructure

**목표**: Runtime에서 Consumer 정보 추적

**작업 내용**:

1. Runtime에서 `X-Consumer-Username` 헤더 읽기
2. FunctionContext에 `consumerId` 추가
3. 사용량 업데이트 API 추가 (Internal 인증)
4. Runtime에서 사용량 리포트 (비동기)

**예상 작업 시간**: 2-3시간

### Phase 3: Multi-Auth Support (선택사항)

**목표**: basic-auth, jwt-auth 지원

**작업 내용**:

1. StageConsumer 스키마 완전 확장
2. Backend에서 authType별 credential 생성
3. APISIX Consumer 동기화 로직 확장

**예상 작업 시간**: 4-5시간

---

## 마무리

### 핵심 요약

APISIX Consumer 인증 아키텍처는 다음과 같이 설계되어 있습니다:

1. **APISIX**: 인증 처리 + Consumer 식별
2. **Consumer Metadata**: `metadata.name` = `X-Consumer-Username` (Backend 전달)
3. **Multi-auth 패턴**: 하나의 Consumer에 여러 인증 방식 동시 적용
4. **API Key 모듈**: 보안 저장 + 사용량 통계 준비
5. **Runtime**: Consumer 헤더 읽기 → 사용량 추적 (구현 필요)

### 언제 사용하나?

**APISIX Consumer 패턴 권장:**
- ✅ API Gateway에서 통합 인증 필요
- ✅ 여러 인증 방식을 동시 지원
- ✅ Backend에서 Consumer 식별 필요
- ✅ 사용량 추적 및 과금 계획

**별도 API Key 모듈 권장:**
- ✅ 보안 강화 (해싱, 재발급)
- ✅ 감사 추적 필요
- ✅ 고급 기능 (Scopes, Rate Limit, IP Whitelist)
- ✅ 향후 사용량 추적/과금 계획

### 실제 적용 결과

**imprun.dev 환경:**
- ✅ APISIX Consumer CRD v2 구조 완벽 이해
- ✅ Critical Bug 발견 (ObjectId 주입 문제)
- ✅ 사용량 추적 아키텍처 설계 완료
- ⚠️ Phase 1 버그 수정 예정

**트러블슈팅 경험:**
- 조사 시간: 약 4시간
- 버그 발견: APISIX Consumer 동기화 코드 분석
- 해결 방법: Plaintext API Key 저장 및 동기화

---

## 참고 자료

### 공식 문서
- [APISIX Basic-Auth Plugin](https://apisix.apache.org/docs/apisix/plugins/basic-auth/)
- [APISIX Key-Auth Plugin](https://apisix.apache.org/docs/apisix/plugins/key-auth/)
- [APISIX JWT-Auth Plugin](https://apisix.apache.org/docs/apisix/plugins/jwt-auth/)
- [APISIX Consumer Concept](https://apisix.apache.org/docs/apisix/terminology/consumer/)
- [APISIX Ingress Controller v2 API](https://apisix.apache.org/docs/ingress-controller/references/v2/)

### 관련 글
- [Apache APISIX로 멀티 테넌트 서버리스 플랫폼 설계하기](https://blog.imprun.dev/41) (예시 - 실제 엔트리 번호는 tistory.md에 등록 필요)

---

**태그:** #APISIX #Consumer #Authentication #APIKey #APIGateway #Kubernetes #CloudNative #Tracking #Billing

**저자:** imprun.dev 팀
**저장소:** [github.com/imprun/imprun](https://github.com/imprun/imprun)

---

> "APISIX Consumer의 `metadata.name`이 모든 것의 핵심입니다. 이것을 이해하면 사용자 추적부터 과금까지 모든 것이 명확해집니다."

🤖 *이 블로그는 [imprun.dev](https://imprun.dev) 플랫폼에서 APISIX Consumer 인증을 구현하며 발견한 버그와 해결 과정을 바탕으로 작성되었습니다.*
