# MongoDB 인덱스 생성 베스트 프랙티스: 수동 vs 자동, 그리고 Hybrid 접근

**작성일:** 2025-11-02
**카테고리:** MongoDB, Database, Performance, Backend, NestJS
**난이도:** 중급

---

## TL;DR

- **문제**: MongoDB 인덱스를 애플리케이션 시작 시 자동 생성? 수동 생성? 어느 것이 맞을까?
- **해결**: 환경에 따라 다른 전략 사용 - Development는 자동, Production은 수동 + 환경 변수 제어
- **핵심**: 대규모 컬렉션에서 인덱스 빌드는 **시간과 리소스 소요**가 크므로 Production에서는 신중하게 계획
- **결과**: 개발 속도 ↑, Production 안정성 ↑, 유연한 인덱스 관리
- **구현**: NestJS onModuleInit + AUTO_CREATE_INDEXES 환경 변수로 Hybrid 접근

---

## 들어가며

[**imprun.dev**](https://imprun.dev)는 "API 개발부터 AI 통합까지, 모든 것을 하나로 제공"하는 Kubernetes 기반 API 플랫폼입니다. CloudFunction Directory API를 개발하면서 **MongoDB 인덱스를 언제, 어떻게 생성해야 하는가**라는 질문에 직면했습니다.

**우리가 마주한 질문**:
- ❓ NestJS 서버 시작 시 자동으로 인덱스를 생성해도 될까?
- ❓ Production 환경에서 createIndex는 안전한가?
- ❓ 인덱스 생성 실패 시 서버는 어떻게 동작해야 하나?
- ❓ Text Index의 weights는 어떻게 설정해야 하나?

**검증 과정**:
1. **수동 생성 (MongoDB Shell)**
   - ✅ Production 안전성 보장
   - ❌ 개발 속도 저하 (매번 수동 실행)
   - ❌ 팀원 간 인덱스 불일치 가능

2. **자동 생성 (Application Startup)**
   - ✅ 개발 속도 향상 (코드로 관리)
   - ❌ Production에서 blocking 위험
   - ❌ 대규모 컬렉션에서 서버 시작 지연

3. **Hybrid 접근 (환경 변수 제어)** ← **최종 선택**
   - ✅ Development는 자동, Production은 수동
   - ✅ 코드로 인덱스 스펙 관리
   - ✅ 유연한 운영 전략

**결론**:
- ✅ NestJS onModuleInit에서 인덱스 생성 로직 구현
- ✅ `AUTO_CREATE_INDEXES` 환경 변수로 제어
- ✅ 실패 시에도 서버 정상 동작 (warn 로깅)

이 글은 **imprun.dev 플랫폼 구축 경험**을 바탕으로, MongoDB 인덱스 생성 전략과 실전 구현 방법을 공유합니다.

---

## MongoDB 인덱스 생성 방법

MongoDB에서 인덱스를 생성하는 방법은 크게 3가지입니다.

### 1. MongoDB Shell (수동)

```javascript
// MongoDB Shell에서 직접 실행
db.CloudFunction.createIndexes([
  { key: { gatewayId: 1 }, name: 'gatewayId_1' },
  { key: { updatedAt: -1 }, name: 'updatedAt_-1' },
  {
    key: { name: 'text', desc: 'text', tags: 'text' },
    name: 'search_text',
    weights: { name: 3, desc: 2, tags: 1 }
  }
])
```

**장점**:
- ✅ Production 환경에서 안전 (롤링 인덱스 빌드 가능)
- ✅ 인덱스 빌드 진행 상황 모니터링 가능
- ✅ 문제 발생 시 즉시 중단 가능

**단점**:
- ❌ 매번 수동 실행 필요
- ❌ 팀원 간 인덱스 스펙 불일치 위험
- ❌ 코드와 인덱스 스펙이 분리

### 2. Application Startup (자동)

```typescript
// NestJS Service
async onModuleInit() {
  await this.db.collection<CloudFunction>('CloudFunction').createIndexes([
    { key: { gatewayId: 1 }, name: 'gatewayId_1' },
    { key: { updatedAt: -1 }, name: 'updatedAt_-1' }
  ])
  this.logger.log('CloudFunction indexes created successfully')
}
```

**장점**:
- ✅ 개발 속도 향상 (코드로 관리)
- ✅ 팀원 간 인덱스 스펙 일치 (Git으로 관리)
- ✅ 로컬 개발 환경 자동 세팅

**단점**:
- ❌ Production에서 인덱스 빌드 시 시스템 리소스 사용 증가
- ❌ 대규모 컬렉션에서 서버 시작 지연 가능성
- ❌ 인덱스 빌드 실패 시 서버 시작 실패 위험

### 3. MongoDB Atlas Auto-Index (클라우드)

Atlas는 Performance Advisor가 자동으로 인덱스를 제안하고 생성할 수 있습니다.

**장점**:
- ✅ 쿼리 패턴 기반 자동 최적화
- ✅ 수동 관리 부담 감소

**단점**:
- ❌ Atlas 전용 (Self-hosted 불가)
- ❌ 인덱스 스펙 예측 불가

---

## createIndex vs createIndexes: 무엇이 다른가?

MongoDB는 두 가지 인덱스 생성 메서드를 제공합니다.

### createIndex (단일 인덱스)

```typescript
await collection.createIndex({ gatewayId: 1 }, { name: 'gatewayId_1' })
```

- 한 번에 하나의 인덱스만 생성
- 각 createIndex마다 별도의 명령 실행

### createIndexes (다중 인덱스) ← **권장**

```typescript
await collection.createIndexes([
  { key: { gatewayId: 1 }, name: 'gatewayId_1' },
  { key: { updatedAt: -1 }, name: 'updatedAt_-1' }
])
```

- 한 번에 여러 인덱스 생성
- 단일 데이터베이스 명령으로 실행
- **더 효율적** (네트워크 왕복 횟수 감소)

`★ Insight ─────────────────────────────────────`
**createIndexes의 장점:**
- 여러 인덱스를 원자적으로 생성
- 네트워크 왕복 횟수 감소 (성능 향상)
- MongoDB 공식 문서 권장 방법
`─────────────────────────────────────────────────`

---

## createIndexes는 Blocking Operation인가?

**주의**: Index build 동작은 MongoDB 버전과 환경에 따라 다를 수 있습니다.

### Index Build의 특성

MongoDB에서 인덱스를 생성할 때 고려해야 할 점:

**일반적인 특성**:
- ⚠️ **대규모 컬렉션에서 인덱스 빌드는 시간 소요**
- ⚠️ **인덱스 빌드 중 시스템 리소스 사용 증가**
- ⚠️ **Production 환경에서는 신중하게 계획 필요**

**MongoDB 4.2 이전 (Legacy)**:
- `background: true` 옵션 사용 가능 (현재는 deprecated)
```typescript
// ❌ MongoDB 4.2+ 에서 deprecated
await collection.createIndex({ field: 1 }, { background: true })
```

**MongoDB 4.2 이후**:
- Index build 프로세스가 개선됨
- 자세한 동작은 MongoDB 공식 문서 참고
- Production 환경에서는 Rolling Index Build 권장

### Rolling Index Build (Production 권장)

```bash
# MongoDB Atlas CLI
atlas api rollingIndex createRollingIndex \
  --clusterName myCluster \
  --groupId <project-id> \
  --file index-spec.json
```

- ✅ 한 번에 하나의 replica set member에만 인덱스 빌드
- ✅ 무중단 인덱스 생성
- ⚠️ Atlas 또는 Ops Manager 필요

---

## imprun.dev의 선택: Hybrid 접근

[**imprun.dev**](https://imprun.dev)는 **환경 변수로 제어하는 Hybrid 접근**을 선택했습니다.

### 구현: FunctionService

**파일**: `server/src/function/function.service.ts:60-89`

```typescript
import { Injectable, Logger, OnModuleInit } from '@nestjs/common'
import { SystemDatabase } from '../system-database'

@Injectable()
export class FunctionService implements OnModuleInit {
  private readonly logger = new Logger(FunctionService.name)
  private readonly db = SystemDatabase.db

  /**
   * Initialize MongoDB indexes for CloudFunction collection
   * Called automatically when the module is initialized
   */
  async onModuleInit() {
    // 환경 변수로 자동 인덱스 생성 비활성화 가능 (기본값: true)
    const autoCreateIndexes = process.env.AUTO_CREATE_INDEXES !== 'false'

    if (!autoCreateIndexes) {
      this.logger.log('CloudFunction index auto-creation is disabled (AUTO_CREATE_INDEXES=false)')
      return
    }

    try {
      await this.db.collection<CloudFunction>('CloudFunction').createIndexes([
        // Index for filtering by gatewayId (used in getFunctionsDirectory)
        { key: { gatewayId: 1 }, name: 'gatewayId_1' },
        // Index for sorting by updatedAt (used in getFunctionsDirectory)
        { key: { updatedAt: -1 }, name: 'updatedAt_-1' },
        // Text index for searching name, desc, and tags
        {
          key: { name: 'text', desc: 'text', tags: 'text' },
          name: 'search_text',
          weights: { name: 3, desc: 2, tags: 1 },
        },
        // Compound index for gatewayId + updatedAt (optimizes filtered + sorted queries)
        { key: { gatewayId: 1, updatedAt: -1 }, name: 'gatewayId_updatedAt' },
      ])
      this.logger.log('CloudFunction indexes created successfully')
    } catch (error) {
      // 인덱스 생성 실패해도 서버는 계속 실행 (쿼리는 느리지만 작동)
      this.logger.warn(`Failed to create CloudFunction indexes: ${error.message}`)
    }
  }
}
```

### 구현: DeviceCodeService (TTL + Unique Index)

**파일**: `server/src/authentication/device-code.service.ts:34-40`

```typescript
async onModuleInit() {
  await this.collection.createIndexes([
    // Unique index for device code
    { key: { deviceCode: 1 }, unique: true, name: 'device_code_unique' },
    // Unique index for user code
    { key: { userCode: 1 }, unique: true, name: 'user_code_unique' },
    // TTL index for automatic document expiration
    { key: { expiresAt: 1 }, expireAfterSeconds: 0, name: 'expires_at_ttl' },
  ])
}
```

### 환경별 설정

**Development (.env.local)**:
```bash
# 개발 환경: 자동 인덱스 생성 활성화 (기본값)
# AUTO_CREATE_INDEXES=true  # 생략 가능
```

**Production (.env.production)**:
```bash
# 프로덕션 환경: 자동 인덱스 생성 비활성화
AUTO_CREATE_INDEXES=false
```

**Production 수동 인덱스 생성**:
```bash
# Kubernetes Job으로 실행
kubectl apply -f k8s/jobs/create-indexes.yaml

# 또는 MongoDB Shell 직접 실행
mongosh "mongodb://..." --eval "db.CloudFunction.createIndexes([...])"
```

---

## 인덱스 타입별 사용 사례

### 1. 단일 필드 인덱스

```typescript
{ key: { gatewayId: 1 }, name: 'gatewayId_1' }
```

**사용 사례**: 특정 필드로 필터링
```typescript
// getFunctionsDirectory에서 사용
const functions = await db.collection('CloudFunction')
  .find({ gatewayId: 'gw-123' })  // ← gatewayId_1 인덱스 사용
  .toArray()
```

### 2. 정렬 인덱스

```typescript
{ key: { updatedAt: -1 }, name: 'updatedAt_-1' }
```

**사용 사례**: 최신 순 정렬
```typescript
const functions = await db.collection('CloudFunction')
  .find()
  .sort({ updatedAt: -1 })  // ← updatedAt_-1 인덱스 사용
  .limit(20)
  .toArray()
```

### 3. 텍스트 인덱스 (Full-Text Search)

```typescript
{
  key: { name: 'text', desc: 'text', tags: 'text' },
  name: 'search_text',
  weights: { name: 3, desc: 2, tags: 1 }
}
```

**사용 사례**: 여러 필드에서 텍스트 검색
```typescript
const functions = await db.collection('CloudFunction')
  .find({ $text: { $search: 'hello world' } })  // ← search_text 인덱스 사용
  .toArray()
```

**Weights 설명**:
- `name: 3` - name 필드 매칭 시 점수 3배
- `desc: 2` - desc 필드 매칭 시 점수 2배
- `tags: 1` - tags 필드 매칭 시 점수 1배

### 4. 복합 인덱스 (Compound Index)

```typescript
{ key: { gatewayId: 1, updatedAt: -1 }, name: 'gatewayId_updatedAt' }
```

**사용 사례**: 필터링 + 정렬
```typescript
const functions = await db.collection('CloudFunction')
  .find({ gatewayId: 'gw-123' })  // ← gatewayId로 필터링
  .sort({ updatedAt: -1 })        // ← updatedAt로 정렬
  .toArray()
// gatewayId_updatedAt 복합 인덱스 사용 (매우 효율적!)
```

`★ Insight ─────────────────────────────────────`
**복합 인덱스의 순서가 중요:**
- `{ gatewayId: 1, updatedAt: -1 }` ✅ gatewayId 필터링 + updatedAt 정렬 가능
- `{ updatedAt: -1, gatewayId: 1 }` ❌ gatewayId 필터링에는 비효율적
- **원칙**: 등호 조건 → 범위 조건 → 정렬 순서로 배치
`─────────────────────────────────────────────────`

### 5. TTL 인덱스 (Time-To-Live)

```typescript
{ key: { expiresAt: 1 }, expireAfterSeconds: 0, name: 'expires_at_ttl' }
```

**사용 사례**: 일정 시간 후 자동 삭제
```typescript
// Device Code는 expiresAt 시간에 자동 삭제
const deviceCode = {
  code: 'ABC123',
  expiresAt: new Date(Date.now() + 15 * 60 * 1000)  // 15분 후
}
await collection.insertOne(deviceCode)
// 15분 후 MongoDB가 자동으로 삭제
```

### 6. 유니크 인덱스 (Unique Index)

```typescript
{ key: { deviceCode: 1 }, unique: true, name: 'device_code_unique' }
```

**사용 사례**: 중복 방지
```typescript
// deviceCode 중복 시 에러 발생
await collection.insertOne({ deviceCode: 'ABC123' })  // ✅ 성공
await collection.insertOne({ deviceCode: 'ABC123' })  // ❌ E11000 duplicate key error
```

---

## 주의사항 및 트러블슈팅

### 1. 인덱스 생성 실패 시 서버 동작

**❌ 나쁜 예** (서버 시작 실패):
```typescript
async onModuleInit() {
  await this.db.collection('CloudFunction').createIndexes([...])
  // 인덱스 생성 실패 시 서버 시작 실패!
}
```

**✅ 좋은 예** (warn 로깅 후 계속 실행):
```typescript
async onModuleInit() {
  try {
    await this.db.collection('CloudFunction').createIndexes([...])
    this.logger.log('Indexes created successfully')
  } catch (error) {
    // 인덱스 없어도 서버는 작동 (느리지만)
    this.logger.warn(`Failed to create indexes: ${error.message}`)
  }
}
```

### 2. 중복 인덱스 생성 시도

MongoDB는 이미 존재하는 인덱스는 재생성하지 않습니다.

```typescript
// 첫 번째 실행
await collection.createIndexes([{ key: { gatewayId: 1 }, name: 'gatewayId_1' }])
// ✅ 인덱스 생성

// 두 번째 실행 (서버 재시작)
await collection.createIndexes([{ key: { gatewayId: 1 }, name: 'gatewayId_1' }])
// ✅ 이미 존재함 - 아무 작업 안함 (매우 빠름)
```

### 3. 인덱스 이름 충돌

**문제**:
```typescript
// 첫 번째 배포
await collection.createIndexes([
  { key: { gatewayId: 1 }, name: 'gatewayId_1' }
])

// 두 번째 배포 (key는 같지만 name 변경)
await collection.createIndexes([
  { key: { gatewayId: 1 }, name: 'gateway_id_index' }  // ← 다른 이름!
])
// ❌ 에러: 같은 key에 다른 이름의 인덱스 존재
```

**해결**:
```bash
# 기존 인덱스 삭제 후 재생성
db.CloudFunction.dropIndex('gatewayId_1')
db.CloudFunction.createIndex({ gatewayId: 1 }, { name: 'gateway_id_index' })
```

### 4. Text Index는 컬렉션당 하나만

```typescript
// ❌ 에러 발생
await collection.createIndexes([
  { key: { name: 'text' }, name: 'name_text' },
  { key: { desc: 'text' }, name: 'desc_text' }  // ← 두 번째 text 인덱스!
])
```

**해결**: 복합 Text Index 사용
```typescript
// ✅ 하나의 text 인덱스에 여러 필드 포함
await collection.createIndexes([
  {
    key: { name: 'text', desc: 'text', tags: 'text' },
    name: 'search_text'
  }
])
```

### 5. 대규모 컬렉션에서의 인덱스 빌드

**문제**: 수백만 건의 문서가 있는 컬렉션에서 인덱스 생성 시 시간 소요

**해결**:
```bash
# Production에서는 Rolling Index Build 사용 (Atlas)
# 또는 오프피크 시간에 인덱스 생성
atlas api rollingIndex createRollingIndex --file index.json

# 또는 직접 MongoDB Shell에서 모니터링하며 생성
mongosh "mongodb://..." --eval "db.collection.createIndex(...)"
```

---

## 베스트 프랙티스 요약

### Development 환경

```typescript
// ✅ onModuleInit에서 자동 생성
async onModuleInit() {
  const autoCreate = process.env.AUTO_CREATE_INDEXES !== 'false'
  if (!autoCreate) return

  try {
    await this.db.collection('CloudFunction').createIndexes([...])
    this.logger.log('Indexes created')
  } catch (error) {
    this.logger.warn(`Failed: ${error.message}`)
  }
}
```

**장점**:
- ✅ 로컬 개발 환경 자동 세팅
- ✅ 팀원 간 인덱스 스펙 일치
- ✅ Git으로 인덱스 히스토리 관리

### Production 환경

```bash
# .env.production
AUTO_CREATE_INDEXES=false
```

```bash
# Kubernetes Job으로 수동 생성
apiVersion: batch/v1
kind: Job
metadata:
  name: create-mongodb-indexes
spec:
  template:
    spec:
      containers:
      - name: create-indexes
        image: mongo:7
        command:
        - mongosh
        - "mongodb://..."
        - --eval
        - "db.CloudFunction.createIndexes([...])"
      restartPolicy: OnFailure
```

**장점**:
- ✅ 서버 시작 시간 단축
- ✅ 인덱스 빌드 실패 시 서버 영향 없음
- ✅ Rolling Index Build 가능 (Atlas)

### 인덱스 설계 원칙

1. **쿼리 패턴 분석**
   - 자주 사용하는 필터링 필드 → 인덱스
   - 정렬 필드 → 인덱스
   - 등호 조건 + 정렬 → 복합 인덱스

2. **복합 인덱스 순서**
   - 등호 조건 필드 먼저
   - 범위 조건 필드 중간
   - 정렬 필드 마지막

3. **인덱스 개수 최소화**
   - 쓰기 성능 고려 (인덱스마다 업데이트 비용)
   - 스토리지 공간 고려

4. **Text Index Weights**
   - 중요한 필드에 높은 가중치
   - 검색 relevance 향상

---

## 마무리

### 핵심 요약

MongoDB 인덱스 생성의 최적 전략은 **환경별로 다른 접근을 사용하는 Hybrid 방식**입니다.

1. **Development**: onModuleInit에서 자동 생성 (코드로 관리)
2. **Production**: 환경 변수로 비활성화 + 수동 생성 (안정성 우선)
3. **인덱스 스펙**: TypeScript 코드로 관리 (Git 버전 관리)
4. **에러 처리**: 실패해도 서버 계속 실행 (warn 로깅)

### 환경별 권장 전략

**Development 환경**:
- ✅ `AUTO_CREATE_INDEXES=true` (기본값)
- ✅ NestJS onModuleInit 자동 생성
- ✅ 개발 속도 우선

**Staging 환경**:
- ✅ `AUTO_CREATE_INDEXES=true` or `false` (팀 정책에 따라)
- ✅ Production과 동일한 환경 테스트

**Production 환경**:
- ✅ `AUTO_CREATE_INDEXES=false`
- ✅ Kubernetes Job 또는 MongoDB Shell로 수동 생성
- ✅ Rolling Index Build 사용 (Atlas)
- ✅ 안정성 우선

### 실제 적용 결과

**imprun.dev 환경** (Kubernetes 3노드, MongoDB 7.0):
- ✅ **개발 속도 향상**: 팀원이 코드만 받으면 인덱스 자동 생성
- ✅ **Production 안정성**: 수동 생성으로 서버 시작 시간 영향 없음
- ✅ **인덱스 히스토리**: Git으로 모든 변경 사항 추적
- ✅ **유연한 운영**: 환경 변수 하나로 전략 전환

**개발 경험**:
- NestJS의 Lifecycle Hook (onModuleInit) 활용
- 환경 변수로 간단한 on/off 제어
- Try-catch로 안전한 에러 처리
- 만족도: 매우 높음 😊

---

## 참고 자료

### 공식 문서
- [MongoDB createIndexes Command](https://www.mongodb.com/docs/manual/reference/command/createIndexes/)
- [MongoDB Index Build Operations](https://www.mongodb.com/docs/manual/core/index-creation/)
- [MongoDB Rolling Index Builds](https://www.mongodb.com/docs/atlas/reference/api-resources-spec/#tag/Rolling-Index)

### 관련 코드
- [function.service.ts:60-89](../../../server/src/function/function.service.ts#L60-L89) - CloudFunction 인덱스 생성
- [device-code.service.ts:34-40](../../../server/src/authentication/device-code.service.ts#L34-L40) - DeviceCode TTL 인덱스

### 관련 글
- [MongoDB Aggregation Pipeline로 N+1 문제 해결하기](https://blog.imprun.dev/54)
- [MongoDB 연결 타임아웃 50% 해결기](https://blog.imprun.dev/53)

---

**태그:** #MongoDB #Index #Performance #NestJS #Backend #Database #BestPractices

**저자:** imprun.dev 팀

---

> "MongoDB 인덱스 관리의 핵심은 환경에 맞는 전략을 선택하는 것입니다. Development는 자동으로 빠르게, Production은 수동으로 안전하게."

🤖 *이 블로그는 실제 프로덕션 환경에서 MongoDB 인덱스를 운영한 경험을 바탕으로 작성되었습니다.*

---

**질문이나 피드백은 블로그 댓글에 남겨주세요!**
