# Environment-Agnostic Architecture 구현기: "baseName만 저장한다"는 착각에서 벗어나기

**작성일:** 2025-11-03
**카테고리:** Implementation, Troubleshooting, Architecture
**난이도:** 중급

---

## TL;DR

- **문제**: "Environment-Agnostic = MongoDB에 baseName만 저장"이라는 착각으로 404 에러 폭탄
- **해결**: 환경별 독립 CloudFunction 존재 (`dev/hello`, `staging/hello`, `prod/hello`), `extractBaseName()` 패턴 전면 적용
- **핵심**: "Frontend는 환경 몰라도 됨" ≠ "MongoDB도 환경 몰라도 됨"
- **결과**: Function 생성 실패, History 404, Promote 실패 → 모두 해결 (4시간 디버깅)

---

## 들어가며

[**imprun.dev**](https://imprun.dev)는 "API 개발부터 AI 통합까지, 모든 것을 하나로 제공"하는 Kubernetes 기반 API 플랫폼입니다.

[Environment-Agnostic Architecture (1탄)](https://blog.imprun.dev/58)에서 이론적인 설계를 공유했습니다. "Frontend는 환경을 모른다"는 아름다운 원칙이었죠.

그런데 **실제 구현 과정은 지옥**이었습니다.

**우리가 마주한 질문**:
- ❓ POST로 "hello" 생성 성공 → GET으로 목록 조회하면 `[]` 빈 배열?
- ❓ Frontend에 "dev/234523452345" 표시 → MongoDB에 `dev/dev/234523452345` 저장됨?
- ❓ History 조회 404 에러 → "environment 파라미터는 왜 필요한데?"

**시행착오 과정**:

1. **"baseName만 저장한다"고 착각**
   - ✅ 이론: Frontend는 baseName만 사용
   - ❌ 현실: MongoDB 조회 실패, History 404, Promote 불가능

2. **"환경은 Subdomain으로만 구분"이라 착각**
   - ✅ 이론: URL은 Subdomain으로 환경 구분
   - ❌ 현실: MongoDB에 환경별 독립 Function 존재

3. **"extractBaseName() 몇 곳만 적용하면 됨"이라 착각** ← **최종 깨달음**
   - ✅ 현실: Controller의 **모든** 메서드에 패턴 적용 필요
   - ✅ 현실: MongoDB는 `dev/hello`, `staging/hello`, `prod/hello` 각각 독립 저장
   - ✅ 현실: History는 environment 쿼리 필수 (각 환경별 독립 functionId)

**결론**:
- ✅ MongoDB 이중 prefix 데이터 정리 (마이그레이션 스크립트)
- ✅ Controller 전체 메서드에 `extractBaseName()` 패턴 적용
- ✅ History 엔드포인트에 environment 쿼리 파라미터 추가

이 글은 **[imprun.dev](https://imprun.dev) 플랫폼 구축 경험**을 바탕으로, Environment-Agnostic Architecture를 구현하면서 겪은 **모든 시행착오**를 솔직하게 공유합니다.

---

## 1. 참사의 시작: "dev/dev/234523452345"

### 증상

Frontend에 이렇게 표시되고 있었습니다:

```
API 경로: dev/234523452345
```

"이상한데? Environment-Agnostic인데 왜 dev/가 붙어있지?"

MongoDB를 확인했습니다:

```javascript
// mongosh
db.CloudFunction.findOne({ name: /234523452345/ })

// 결과
{
  _id: ObjectId("..."),
  name: "dev/dev/234523452345"  // ← 이중 prefix!
}
```

### 원인 분석

**FunctionService.create()**가 무조건 `dev/` prefix를 추가하고 있었습니다:

```typescript
// ❌ 문제 코드
async create(gatewayId: string, dto: CreateFunctionDto) {
  const devFunctionName = `dev/${dto.name}`  // 입력이 이미 "dev/hello"면?

  await this.db.collection('CloudFunction').insertOne({
    gatewayId,
    name: devFunctionName,  // "dev/dev/hello" 저장!
    // ...
  })
}
```

**Frontend에서 이미 "dev/hello"를 보내면** → MongoDB에 `dev/dev/hello` 저장!

### 해결

**extractBaseName()로 먼저 정리**:

```typescript
// ✅ 수정된 코드
import { extractBaseName } from '@/utils/getter'

async create(gatewayId: string, dto: CreateFunctionDto) {
  // 1. 입력에서 환경 prefix 제거 (있으면)
  const baseName = extractBaseName(dto.name)  // "dev/hello" → "hello"

  // 2. dev/ prefix 추가
  const devFunctionName = `dev/${baseName}`  // "dev/hello"

  await this.db.collection('CloudFunction').insertOne({
    gatewayId,
    name: devFunctionName,
    // ...
  })
}
```

### 데이터 정리 (마이그레이션 스크립트)

기존 데이터에 이미 이중 prefix가 있었습니다:

```javascript
// server/scripts/fix-function-names.js

const { MongoClient } = require('mongodb')

async function fixFunctionNames() {
  const client = new MongoClient(process.env.DATABASE_URL)
  await client.connect()

  const db = client.db('sys_db')
  const collection = db.collection('CloudFunction')

  const functions = await collection.find({}).toArray()

  let updated = 0
  for (const fn of functions) {
    const originalName = fn.name

    // dev/dev/hello → hello
    let baseName = originalName.replace(/^(dev|staging|prod)\//, '')
    baseName = baseName.replace(/^(dev|staging|prod)\//, '')  // 이중 제거

    if (baseName !== originalName) {
      await collection.updateOne(
        { _id: fn._id },
        { $set: { name: baseName } }
      )
      console.log(`Updated: ${originalName} → ${baseName}`)
      updated++
    }
  }

  console.log(`Total updated: ${updated}`)
  await client.close()
}

fixFunctionNames().catch(console.error)
```

**실행 결과**:
```
Updated: dev/dev/234523452345 → 234523452345
Total updated: 1
```

---

## 2. Function 생성 후 목록 조회 실패

### 증상

```
1. POST /v1/api-gateways/ueigjz/functions
   Body: { name: "1" }
   Response: 200 OK

2. GET /v1/api-gateways/ueigjz/functions
   Response: { data: [] }  // ← 빈 배열!

3. GET /v1/api-gateways/ueigjz/functions/1
   Response: 404 Not Found  // ← Function이 없다고?
```

### 원인 분석

**FunctionController.findOne()**이 baseName을 그대로 조회:

```typescript
// ❌ 문제 코드
@Get(':baseName')
async findOne(@Param('baseName') baseName: string) {
  // baseName = "1"
  const fullName = baseName.includes('/') ? baseName : `dev/${baseName}`
  // fullName = "dev/1"

  const func = await this.functionsService.findOne(gatewayId, fullName)
  // MongoDB 조회: name = "dev/1" ← 존재함!

  if (!func) {
    throw new NotFoundException()
  }

  return ResponseUtil.ok(func)  // ← name: "dev/1" 그대로 반환
}
```

**문제**: `extractBaseName()`을 **응답 시에만 적용**해야 하는데, 아예 적용 안 함!

### 해결

**모든 Controller 메서드에 extractBaseName() 패턴 적용**:

#### GET /functions/:baseName

```typescript
// ✅ 수정된 코드
import { extractBaseName } from '@/utils/getter'

@Get(':baseName')
async findOne(
  @Param('gatewayId') gatewayId: string,
  @Param('baseName') baseName: string,
) {
  // 1. baseName 정리 (혹시 모를 prefix 제거)
  const cleanBaseName = extractBaseName(baseName)  // "dev/1" → "1"

  // 2. dev/ prefix 추가
  const fullName = `dev/${cleanBaseName}`  // "dev/1"

  // 3. MongoDB 조회
  const func = await this.functionsService.findOne(gatewayId, fullName)

  if (!func) {
    throw new NotFoundException(`Function not found: ${cleanBaseName}`)
  }

  // 4. ✅ 응답 시 baseName 변환
  return ResponseUtil.ok({
    ...func,
    name: extractBaseName(func.name)  // "dev/1" → "1"
  })
}
```

#### PATCH /functions/:baseName

```typescript
// ✅ update, remove, typeCheck, getOpenAPISpec 모두 동일 패턴
@Patch(':baseName')
async update(
  @Param('baseName') baseName: string,
  @Body() dto: UpdateFunctionDto,
) {
  const cleanBaseName = extractBaseName(baseName)
  const fullName = `dev/${cleanBaseName}`

  const func = await this.functionsService.findOne(gatewayId, fullName)
  if (!func) {
    throw new NotFoundException(`Function not found: ${cleanBaseName}`)
  }

  const updated = await this.functionsService.updateOne(func, dto)

  return ResponseUtil.ok({
    ...updated,
    name: extractBaseName(updated.name)  // ✅ baseName 반환
  })
}
```

**적용 메서드 (6곳)**:
- `findOne()` - GET /functions/:baseName
- `update()` - PATCH /functions/:baseName
- `updateDebug()` - PATCH /functions/:baseName/debug
- `remove()` - DELETE /functions/:baseName
- `typeCheck()` - POST /functions/:baseName/typecheck
- `getOpenAPISpec()` - GET /functions/:baseName/openapi

### 교훈

> **"Controller의 모든 메서드에서 입력 정리 + 응답 변환 패턴 적용"**

---

## 3. History 조회 404 에러

### 증상

```
GET /v1/api-gateways/ueigjz/functions/1/history?environment=dev
Response: 404 Not Found
```

### 초기 오해

저는 이렇게 생각했습니다:

> "Environment-Agnostic이니까 MongoDB에 baseName만 저장하겠지?"

```typescript
// ❌ 잘못된 이해
{
  name: "hello"  // baseName만 저장?
}
```

그래서 History 엔드포인트를 이렇게 구현:

```typescript
// ❌ 잘못된 구현
@Get(':baseName/history')
async getHistory(@Param('baseName') baseName: string) {
  // baseName으로만 조회
  const func = await this.functionsService.findOne(gatewayId, baseName)
  // MongoDB 조회: name = "1" ← 존재하지 않음!

  if (!func) {
    throw new NotFoundException()  // ← 404!
  }
}
```

### 사용자의 설명

> "히스토리는 staging, prod 도 조회해야 하기 때문에 쿼리스트링으로 환경정보를 제공했다."

**아하! environment 쿼리 파라미터가 필요한 이유**:
- baseName "hello"에 대해 **환경별로 독립 CloudFunction**이 존재
- `dev/hello`, `staging/hello`, `prod/hello` 각각 다른 **functionId**
- History는 **functionId**로 조회하므로 어떤 환경인지 알아야 함

### 최종 이해

**MongoDB 실제 구조**:

```typescript
// ✅ 실제 MongoDB
// baseName "hello" → 최대 3개 Function
{
  _id: ObjectId("aaa"),
  name: "dev/hello",
  source: { version: 1 }
}
{
  _id: ObjectId("bbb"),
  name: "staging/hello",
  source: { version: 1 }
}
{
  _id: ObjectId("ccc"),
  name: "prod/hello",
  source: { version: 1 }
}
```

**각 환경은 독립적인 functionId를 가짐**!

### 해결

```typescript
// ✅ 올바른 구현
@Get(':baseName/history')
async getHistory(
  @Param('gatewayId') gatewayId: string,
  @Param('baseName') baseName: string,
  @Query('environment') environment?: string,  // ✅ 환경 쿼리 추가
) {
  // ⚠️ 중요: 환경별로 독립 CloudFunction 존재
  const env = environment || 'dev'
  const fullName = `${env}/${baseName}`  // "dev/hello", "staging/hello", "prod/hello"

  const func = await this.functionsService.findOne(gatewayId, fullName)
  if (!func) {
    throw new NotFoundException(
      `Function not found in ${env} environment: ${baseName}`
    )
  }

  // History는 functionId로 조회 (각 환경별로 독립)
  const history = await this.functionsService.getHistory(func)

  return ResponseUtil.ok(history)
}
```

### 사용자 확인

> "수정한 코드가 맞아, 다만 'staging, prod에 매칭되는 function Id는 존재하지 않습니다'는 틀렸어. 있긴 하지만 frontend 에서 알 필요가 없는 것이지."

**핵심 깨달음**:
- functionId는 **각 환경별로 존재**함
- Frontend는 **baseName만 알면 됨**
- Backend가 **environment 쿼리로 적절한 Function 조회**

---

## 4. Promote 실패: Semantic Version 제한 에러

### 증상

FunctionEditor에서 "배포하기" 클릭 시:

```
Error: Semantic version can only be set in dev environment
```

### 원인 분석

**FunctionService.updateOne()**이 환경 추출 시도:

```typescript
// ❌ 문제 코드
async updateOne(func: CloudFunction, dto: UpdateFunctionDto) {
  // Environment 추출 시도
  const stage = func.name.split('/')[0]  // "dev"
  const isDevEnvironment = stage === 'dev'

  // Semantic version은 dev만 허용?
  if (dto.semver && !isDevEnvironment) {
    throw new Error('Semantic version can only be set in dev environment')
  }

  // ...
}
```

**문제**: Environment-Agnostic인데 **환경 기반 제약**을 두고 있음!

### 해결

```typescript
// ✅ 수정된 코드
async updateOne(func: CloudFunction, dto: UpdateFunctionDto) {
  // Environment-Agnostic: baseName만 사용, stage 정보 없음
  const stage = 'dev'  // Fixed for bundling purposes

  // ✅ Semantic version 제한 제거
  // 모든 환경에서 semver 허용

  // Bundling 로직
  const bundled = await this.bundler.bundle({
    files: dto.files || func.source.files,
    entrypoint: dto.entrypoint || func.source.entrypoint,
    stage,  // dev로 고정 (bundling용)
  })

  // Update
  await this.db.collection('CloudFunction').updateOne(
    { _id: func._id },
    {
      $set: {
        'source.files': bundled.files,
        'source.semver': dto.semver || func.source.semver,
        // ...
      }
    }
  )
}
```

---

## 5. Frontend Pipeline 에러

### 증상

```
PromoteFunctionSheet.tsx:71
Uncaught ReferenceError: isPipelineEnabled is not defined
```

```
PromoteFunctionSheet.tsx:185
Uncaught ReferenceError: currentStage is not defined
```

### 원인

Environment-Agnostic 마이그레이션 과정에서 **Pipeline 관련 코드**를 삭제했는데, 일부 참조가 남아있었습니다.

### 해결

**모든 Pipeline 변수 제거**:

```typescript
// ❌ 삭제된 코드
const isPipelineEnabled = gateway?.pipeline?.enabled
const currentStage = devFunction?.name.split("/")[0] || "dev"
const pipelineStages = [...getStageStatus()]

// ✅ 수정된 코드
const activeStages = stages.filter((stage) => stage.state === "Active")
const availableStages = activeStages  // 모든 활성 환경으로 배포 가능
```

**targetFunction 로직 단순화**:

```typescript
// ❌ 삭제된 코드
const targetName = `${selectedStage}/${functionName}`
const targetFunction = allFunctions.find((f) => f.name === targetName)

// ✅ 수정된 코드
// Environment-Agnostic: baseName으로 찾기
const targetFunction = allFunctions.find((f) => f.name === functionName)
```

---

## 6. 실전 체크리스트

### Backend 변경 사항

**필수 패턴 (모든 Controller 메서드)**:

```typescript
import { extractBaseName } from '@/utils/getter'

// 1. 입력 파라미터 정리
const cleanBaseName = extractBaseName(baseName)

// 2. dev/ prefix 추가 (또는 environment 기반)
const fullName = `dev/${cleanBaseName}`

// 3. MongoDB 조회
const func = await this.functionsService.findOne(gatewayId, fullName)

// 4. 응답 시 baseName 변환
return ResponseUtil.ok({
  ...func,
  name: extractBaseName(func.name)
})
```

**적용 메서드 목록**:
- [x] `create()` - POST /functions
- [x] `findOne()` - GET /functions/:baseName
- [x] `update()` - PATCH /functions/:baseName
- [x] `updateDebug()` - PATCH /functions/:baseName/debug
- [x] `remove()` - DELETE /functions/:baseName
- [x] `typeCheck()` - POST /functions/:baseName/typecheck
- [x] `getOpenAPISpec()` - GET /functions/:baseName/openapi
- [x] `getHistory()` - GET /functions/:baseName/history (environment 쿼리 추가)

### Frontend 변경 사항

**제거된 코드**:
- [x] `getDisplayName()` 함수 (9개 파일에서 제거)
- [x] Pipeline 관련 변수 (`isPipelineEnabled`, `currentStage`, `pipelineStages`)
- [x] 환경 prefix 제거 로직

**추가된 코드**:
- [x] History 조회 시 environment 파라미터 전달
- [x] Promote 시 source environment 포함 전송

---

## 마무리

### 핵심 요약

> **"Frontend는 환경 몰라도 됨" ≠ "MongoDB도 환경 몰라도 됨"**

**실전에서 배운 4가지**:
1. **MongoDB 구조 확인이 최우선**: baseName당 환경별 독립 CloudFunction 존재
2. **패턴의 일관성**: Controller **모든** 메서드에서 extractBaseName 패턴 적용
3. **데이터 마이그레이션**: 기존 이중 prefix 데이터 정리 필수
4. **점진적 제거**: Pipeline 등 삭제된 기능의 참조 완전 제거

### 언제 사용하나?

**Environment-Agnostic Architecture 구현 시 필수 체크**:
- ✅ MongoDB 구조를 정확히 이해했는가?
- ✅ Controller 모든 메서드에 패턴 적용했는가?
- ✅ 기존 데이터 마이그레이션은 필요한가?
- ✅ 환경별 조회 엔드포인트에 environment 쿼리 추가했는가?

### 실제 적용 결과

**[imprun.dev](https://imprun.dev) 환경:**
- ✅ 데이터 정리: 이중 prefix Function 1개 수정
- ✅ Controller: 8개 메서드 패턴 적용
- ✅ Frontend: Pipeline 변수 완전 제거
- ✅ History: environment 쿼리로 환경별 조회 가능

**운영 경험:**
- 디버깅 시간: 약 4시간 (실제 경험)
- 주요 시간 소모: MongoDB 구조 오해 해소 (2시간)
- ROI: 404 에러 완전 해소, 개발자 혼란 제거
- 만족도: 매우 높음 😊 (모든 CRUD 정상 동작)

---

## 관련 글

- [Environment-Agnostic Architecture: Frontend와 Backend의 환경 분리 패턴 (1탄)](https://blog.imprun.dev/58)
- [imprun.dev GitHub Repository](https://github.com/imprun/imprun)

---

**태그:** Implementation, Troubleshooting, EnvironmentAgnostic, MongoDB, Backend, RealExperience

---

> "이론은 아름다웠다. 하지만 현실은... MongoDB에 환경별 독립 Function이 존재한다는 걸 4시간 후에 깨달았다."

🤖 *이 글은 **[imprun.dev](https://imprun.dev) 플랫폼 구축 과정**에서 실제로 겪은 모든 시행착오와 디버깅 과정을 솔직하게 공유한 것입니다.*

---

**질문이나 피드백은 블로그 댓글에 남겨주세요!**
