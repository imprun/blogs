# 분산 환경에서 Optimistic Lock으로 동시성 제어하기

> Cron Job 기반 Task Processing에서 타임스탬프를 활용한 동시성 제어 구현

## TL;DR

- ✅ **Optimistic Locking**: DB 트랜잭션 없이 타임스탬프로 동시성 제어
- ✅ **Lock/Unlock 패턴**: 각 단계마다 unlock하여 순차 처리
- ✅ **Progressive Backoff**: 대기 시간을 점진적으로 증가시켜 재시도
- ✅ **분산 환경 안전**: 여러 서버에서 동시 실행해도 중복 처리 방지
- ✅ **자동 복구**: Lock 타임아웃으로 서버 장애 시에도 자동 재시도

이 글은 **[imprun.dev](https://imprun.dev)** 플랫폼에서 여러 서버에서 동시에 실행되는 Cron Job 기반 Task Processing을 안전하게 구현한 경험을 공유합니다.

---

## 들어가며: 여러 서버에서 동시에 Cron Job 실행하기

[imprun.dev](https://imprun.dev)는 Kubernetes 기반 플랫폼으로, HA(High Availability)를 위해 **여러 Pod에서 API 서버를 실행**합니다.

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Pod 1      │  │  Pod 2      │  │  Pod 3      │
│  API Server │  │  API Server │  │  API Server │
│             │  │             │  │             │
│  @Cron      │  │  @Cron      │  │  @Cron      │
│  EVERY_SEC  │  │  EVERY_SEC  │  │  EVERY_SEC  │
└─────────────┘  └─────────────┘  └─────────────┘
       │                │                │
       └────────────────┼────────────────┘
                        ↓
                  MongoDB (shared)
```

**문제**: 모든 Pod에서 1초마다 같은 Cron Job이 실행됩니다!

```typescript
@Cron(CronExpression.EVERY_SECOND)
async tick() {
  // Pod 1, Pod 2, Pod 3 모두 실행!
  this.handleDeletingPhase()
}

async handleDeletingPhase() {
  // 같은 Gateway를 Pod 1, 2, 3가 동시에 처리하면?
  const gateway = await db.findOne({ phase: 'Deleting' })
  await deleteResources(gateway)  // ← 중복 삭제 시도!
}
```

**발생 가능한 문제**:
- ❌ **중복 처리**: Pod 1, 2가 같은 Gateway를 동시에 삭제 시도
- ❌ **Race Condition**: Pod 1이 삭제 중인데 Pod 2가 또 삭제
- ❌ **리소스 낭비**: 같은 작업을 여러 번 실행

---

## Optimistic Lock 패턴으로 해결

### 핵심 아이디어: `lockedAt` 타임스탬프

각 Document에 `lockedAt` 필드를 추가하고, **타임스탬프 기반으로 Lock을 획득**합니다.

```typescript
export class ApiGateway {
  gatewayId: string
  state: 'Running' | 'Stopped' | 'Deleted'
  phase: 'Creating' | 'Created' | 'Deleting' | 'Deleted'

  lockedAt: Date       // ← 이 필드!
  createdAt: Date
  updatedAt: Date
}
```

### findAndLock() 구현

```typescript
async handleDeletingPhase() {
  const lockTimeout = 30  // 30초

  // 1. Lock 획득 (Atomic Operation)
  const res = await db.collection('ApiGateway').findOneAndUpdate(
    {
      phase: 'Deleting',
      lockedAt: { $lt: new Date(Date.now() - 1000 * lockTimeout) }
      //          ↑ 30초 이상 지난 문서만 선택
    },
    { $set: { lockedAt: new Date() } },  // 현재 시간으로 Lock
    { sort: { lockedAt: 1, updatedAt: 1 }, returnDocument: 'after' }
  )

  if (!res.value) return  // Lock 획득 실패 (다른 Pod가 처리 중)

  const gateway = res.value

  // 2. 작업 수행
  await deleteResources(gateway)

  // 3. Lock 해제 (다음 단계 처리 가능하도록)
  await unlock(gateway.gatewayId)
}
```

### 동작 원리

**시나리오**: Pod 1, 2가 동시에 `handleDeletingPhase()` 실행

```
Time: 0초
─────────────────────────────────────────────
MongoDB: gateway { phase: 'Deleting', lockedAt: 1970-01-01 }
                                       ↑ 과거 시간 (unlocked 상태)

Pod 1 실행: findOneAndUpdate() 시작
Pod 2 실행: findOneAndUpdate() 시작

Time: 0.1초
─────────────────────────────────────────────
Pod 1: Lock 획득 성공!
  → gateway { lockedAt: 2025-11-01 10:00:00 }

Pod 2: Lock 획득 실패 (lockedAt이 0초 전이라 조건 불충족)
  → return (처리 안 함)

Time: 0.1 ~ 30초
─────────────────────────────────────────────
Pod 1: 리소스 삭제 작업 수행 중...

Pod 2: 다음 tick에서 계속 시도
  → lockedAt이 30초 이상 지나지 않아서 계속 실패
  → Pod 1이 작업을 완료할 때까지 대기

Time: 5초
─────────────────────────────────────────────
Pod 1: 작업 완료 → unlock()
  → gateway { lockedAt: 1970-01-01 }
                      ↑ 과거 시간으로 재설정

Pod 2: Lock 획득 성공!
  → 다음 단계 처리 시작
```

---

## Lock/Unlock/Relock 패턴

### Unlock: 다음 단계로 진행

```typescript
async unlock(gatewayId: string) {
  await db.collection('ApiGateway').updateOne(
    { gatewayId },
    { $set: { lockedAt: TASK_LOCK_INIT_TIME } }
    //                   ↑ new Date('1970-01-01')
  )
}
```

**언제 사용하나요?**
- 한 단계 작업 완료 시
- 다음 tick에서 즉시 다음 단계 처리하도록

**예시**:
```typescript
async handleDeletingPhase() {
  const gateway = await findAndLock()

  // 1단계: Trigger 삭제
  if (hadTriggers > 0) {
    await this.triggerService.removeAll(gatewayId)
    return await this.unlock(gatewayId)  // ← 즉시 다음 단계로
  }

  // 2단계: Function 삭제
  if (hadFunctions > 0) {
    await this.functionService.removeAll(gatewayId)
    return await this.unlock(gatewayId)
  }

  // ...
}
```

### Relock: 대기 후 재시도

```typescript
async relock(gatewayId: string, waitingTime = 0) {
  // Progressive Backoff: 대기 시간 점진적 증가
  if (waitingTime <= 2 * 60 * 1000) {  // 2분 이하
    waitingTime = Math.ceil(waitingTime / 10)  // 10% 증가
  }

  if (waitingTime > 2 * 60 * 1000) {  // 2분 초과
    waitingTime = this.lockTimeout * 1000  // 30초 고정
  }

  const lockedAt = new Date(Date.now() - 1000 * this.lockTimeout + waitingTime)
  await db.collection('ApiGateway').updateOne(
    { gatewayId },
    { $set: { lockedAt } }
  )
}
```

**언제 사용하나요?**
- 비동기 작업 대기 시 (예: Kubernetes Pod 생성 중)
- 외부 리소스가 준비될 때까지 대기

**예시**:
```typescript
async handleStartingPhase() {
  const gateway = await findAndLock()

  // Pod 생성
  await instanceService.create(gatewayId)

  // Pod Ready 확인
  const instance = await instanceService.get(gatewayId)
  if (instance.deployment?.status?.unavailableReplicas > 0) {
    // Pod 아직 준비 안됨 → 대기 후 재시도
    const waitingTime = Date.now() - gateway.updatedAt.getTime()
    return await this.relock(gatewayId, waitingTime)
  }

  // Pod 준비 완료 → 다음 단계
  gateway.phase = 'Started'
}
```

### Progressive Backoff 전략

```typescript
// 대기 시간 증가 패턴
waitingTime:  0초 → 0초 대기
waitingTime:  1초 → 0.1초 대기
waitingTime: 10초 → 1초 대기
waitingTime: 30초 → 3초 대기
waitingTime:  2분 → 12초 대기
waitingTime:  3분 → 30초 대기 (최대값 고정)
```

**왜 이렇게 설계했나요?**
1. **초기에는 빠르게 재시도**: Pod 생성은 보통 10초 이내
2. **대기가 길어지면 천천히**: 2분 이상 걸리면 뭔가 문제가 있음
3. **최대 30초로 제한**: 너무 오래 대기하지 않음

---

## 실제 구현 사례

### ApiGatewayTaskService: Deleting Phase

```typescript
@Injectable()
export class ApiGatewayTaskService {
  readonly lockTimeout = 30  // 30초

  async handleDeletingPhase() {
    const db = SystemDatabase.db

    // 1. Lock 획득
    const res = await db.collection<ApiGateway>('ApiGateway')
      .findOneAndUpdate(
        {
          phase: ApiGatewayPhase.Deleting,
          lockedAt: { $lt: new Date(Date.now() - 1000 * this.lockTimeout) }
        },
        { $set: { lockedAt: new Date() } },
        { sort: { lockedAt: 1, updatedAt: 1 }, returnDocument: 'after' }
      )

    if (!res.value) return  // Lock 획득 실패
    const gateway = res.value

    // 2. 순차 삭제 (각 단계마다 unlock)
    const hadTriggers = await db.collection('CronTrigger')
      .countDocuments({ gatewayId: gateway.gatewayId })
    if (hadTriggers > 0) {
      await this.triggerService.removeAll(gateway.gatewayId)
      return await this.unlock(gateway.gatewayId)
    }

    const hadFunctions = await db.collection('CloudFunction')
      .countDocuments({ gatewayId: gateway.gatewayId })
    if (hadFunctions > 0) {
      await this.functionService.removeAll(gateway.gatewayId)
      return await this.unlock(gateway.gatewayId)
    }

    // ... (Stage, RuntimeDomain, Database 순차 삭제)

    // 3. 모든 작업 완료
    await db.collection('ApiGateway').updateOne(
      { _id: gateway._id },
      { $set: { phase: ApiGatewayPhase.Deleted } }
    )
  }

  async unlock(gatewayId: string) {
    await db.collection('ApiGateway').updateOne(
      { gatewayId },
      { $set: { lockedAt: TASK_LOCK_INIT_TIME } }
    )
  }
}
```

### RuntimeDomainTaskService: Creating Phase with Relock

```typescript
@Injectable()
export class RuntimeDomainTaskService {
  readonly lockTimeout = 30

  async handleCreatingPhase() {
    const db = SystemDatabase.db

    // 1. Lock 획득
    const res = await db.collection<RuntimeDomain>('RuntimeDomain')
      .findOneAndUpdate(
        {
          phase: DomainPhase.Creating,
          lockedAt: { $lt: new Date(Date.now() - 1000 * this.lockTimeout) }
        },
        { $set: { lockedAt: new Date() } },
        { returnDocument: 'after' }
      )

    if (!res.value) return
    const doc = res.value

    // 2. Certificate 생성 (customDomain인 경우)
    if (doc.customDomain && region.gatewayConf.tls?.enabled) {
      const waitingTime = Date.now() - doc.updatedAt.getTime()

      // Certificate 생성
      let cert = await this.certService.getRuntimeCertificate(region, doc)
      if (!cert) {
        cert = await this.certService.createRuntimeCertificate(region, doc)
        this.logger.log(`Creating certificate: ${doc.gatewayId}`)
        // Certificate 생성 요청 → cert-manager가 처리 중
        return await this.relock(doc.gatewayId, waitingTime)
      }

      // Certificate Ready 확인
      const conditions = (cert as any).status?.conditions || []
      if (!isConditionTrue('Ready', conditions)) {
        this.logger.log(`Certificate not ready: ${doc.gatewayId}`)
        // 아직 준비 안됨 → 대기 후 재시도
        return await this.relock(doc.gatewayId, waitingTime)
      }
    }

    // 3. Ingress 생성
    const ingress = await this.runtimeGateway.getIngress(region, doc)
    if (!ingress) {
      await this.runtimeGateway.createIngress(region, doc)
      this.logger.log(`Ingress created: ${doc.gatewayId}`)
    }

    // 4. Phase 완료
    await db.collection('RuntimeDomain').updateOne(
      { _id: doc._id, phase: DomainPhase.Creating },
      {
        $set: {
          phase: DomainPhase.Created,
          lockedAt: TASK_LOCK_INIT_TIME,
          updatedAt: new Date()
        }
      }
    )
  }

  async relock(gatewayId: string, waitingTime = 0) {
    if (waitingTime <= 2 * 60 * 1000) {
      waitingTime = Math.ceil(waitingTime / 10)
    }

    if (waitingTime > 2 * 60 * 1000) {
      waitingTime = this.lockTimeout * 1000
    }

    const db = SystemDatabase.db
    const lockedAt = new Date(Date.now() - 1000 * this.lockTimeout + waitingTime)
    await db.collection('RuntimeDomain').updateOne(
      { gatewayId },
      { $set: { lockedAt } }
    )
  }
}
```

---

## 이 패턴의 장점

### 1. 분산 환경 안전

```typescript
// ❌ Lock 없이 구현하면?
async handleDeletingPhase() {
  const gateway = await db.findOne({ phase: 'Deleting' })
  // Pod 1, 2, 3 모두 같은 gateway 선택!

  await deleteResources(gateway)
  // 중복 삭제 시도 → 에러 발생
}

// ✅ Optimistic Lock으로 안전
async handleDeletingPhase() {
  const gateway = await findAndLock()
  // Pod 1만 Lock 획득, Pod 2/3는 null 반환

  if (!gateway) return  // 다른 Pod가 처리 중

  await deleteResources(gateway)
  // 단 하나의 Pod만 실행
}
```

### 2. DB 트랜잭션 불필요

```typescript
// ❌ Pessimistic Lock (트랜잭션 필요)
const session = await client.startSession()
session.startTransaction()
try {
  const gateway = await db.findOne({ ... }, { session })
  await gateway.lock()  // ← 다른 요청은 블로킹됨
  await deleteResources(gateway)
  await session.commitTransaction()
} catch (err) {
  await session.abortTransaction()
}

// ✅ Optimistic Lock (트랜잭션 불필요)
const gateway = await findAndLock()  // Atomic Operation
if (!gateway) return  // 실패 시 그냥 return

await deleteResources(gateway)  // 트랜잭션 없이 실행
```

**MongoDB 트랜잭션의 문제**:
- Replica Set 필요
- 성능 오버헤드 (MVCC)
- 복잡한 에러 처리

**Optimistic Lock의 장점**:
- 단일 Document Update (Atomic)
- 트랜잭션 불필요
- 간단한 구현

### 3. 자동 Lock 해제 (Timeout)

```typescript
// Lock 획득 조건
lockedAt: { $lt: new Date(Date.now() - 1000 * lockTimeout) }
//                                      ↑ 30초

// 시나리오: Pod 1이 작업 중 장애 발생
Time: 0초 - Pod 1 Lock 획득 (lockedAt: 2025-11-01 10:00:00)
Time: 10초 - Pod 1 서버 장애! (작업 중단)
Time: 35초 - Pod 2 Lock 획득 (30초 경과)
Time: 36초 - Pod 2 작업 재시작
```

**수동 복구 불필요**: 30초 후 자동으로 다른 Pod가 처리

### 4. 순차 처리 (Sequential Processing)

```typescript
async handleDeletingPhase() {
  const gateway = await findAndLock()

  // 각 단계마다 unlock → 다음 tick에서 계속
  if (hadTriggers) {
    await deleteTriggers()
    return await unlock()  // ← 1단계 완료
  }

  if (hadFunctions) {
    await deleteFunctions()
    return await unlock()  // ← 2단계 완료
  }

  if (hadStages) {
    await deleteStages()
    return await unlock()  // ← 3단계 완료
  }
}
```

**왜 unlock을 단계별로?**
1. **부분 실패 복구**: 2단계 실패 시 1단계는 재실행 안 함
2. **타임아웃 방지**: 각 단계가 30초 이내에 완료
3. **디버깅 용이**: 어느 단계에서 멈췄는지 확인 가능

### 5. Progressive Backoff로 효율성

```typescript
// Certificate 생성 대기
waitingTime = Date.now() - doc.updatedAt.getTime()

if (waitingTime === 0초)  → 즉시 재시도 (0초 대기)
if (waitingTime === 10초) → 1초 후 재시도
if (waitingTime === 60초) → 6초 후 재시도
if (waitingTime === 3분)  → 30초 후 재시도
```

**효과**:
- 빠르게 완료되는 작업은 즉시 처리
- 오래 걸리는 작업은 서버 부담 감소

---

## vs Pessimistic Lock

### Pessimistic Lock

```typescript
// Redis 기반 Distributed Lock
const lock = await redis.lock('gateway:abc123', { ttl: 30000 })

try {
  await deleteResources()
} finally {
  await lock.unlock()
}
```

**장점**:
- ✅ 강력한 동시성 제어
- ✅ 즉시 Lock 획득 실패 감지

**단점**:
- ❌ 외부 의존성 (Redis, Memcached)
- ❌ Lock 해제 실패 시 Deadlock
- ❌ 복잡한 에러 처리 (unlock 누락 방지)

### Optimistic Lock

```typescript
// MongoDB 기반 Optimistic Lock
const gateway = await findAndLock()
if (!gateway) return

await deleteResources()
await unlock()
```

**장점**:
- ✅ 외부 의존성 없음 (MongoDB만 사용)
- ✅ 자동 Lock 해제 (Timeout)
- ✅ 간단한 구현

**단점**:
- ❌ Lock 획득 재시도 (polling)
- ❌ Lock 경합 시 지연 (최대 30초)

### 언제 어떤 패턴을 쓸까?

| 상황 | Pessimistic Lock | Optimistic Lock |
|------|------------------|-----------------|
| 높은 동시성 | ⭐⭐⭐ | ⭐⭐ |
| 긴 작업 시간 (>30초) | ⭐⭐⭐ | ⭐ |
| 간단한 구조 | ⭐ | ⭐⭐⭐ |
| 자동 복구 | ⭐⭐ | ⭐⭐⭐ |
| Task Processing | ⭐⭐ | ⭐⭐⭐ |
| API 요청 처리 | ⭐⭐⭐ | ⭐ |

**imprun.dev는 Optimistic Lock 선택**:
- Task Processing (Cron Job)
- MongoDB 이미 사용 중
- 자동 복구 필요
- 간단한 구조 선호

---

## PostgreSQL이었다면?

imprun.dev는 MongoDB를 사용하기 때문에 구현이 매우 간단했습니다. 만약 **PostgreSQL**을 사용했다면 어땠을까요?

### MongoDB의 장점

```typescript
// MongoDB: 단일 Operation으로 Atomic Lock
const gateway = await db.collection('ApiGateway').findOneAndUpdate(
  { phase: 'Deleting', lockedAt: { $lt: pastTime } },
  { $set: { lockedAt: new Date() } },
  { returnDocument: 'after' }
)
// ✅ 트랜잭션 불필요
// ✅ 한 줄로 Lock 획득
// ✅ Connection Pool 신경 안 써도 됨
```

### PostgreSQL 구현 옵션

**방법 1: SELECT FOR UPDATE SKIP LOCKED** (권장)

```typescript
// PostgreSQL의 경우
const client = await pool.connect()
try {
  await client.query('BEGIN')

  // Lock 획득 시도
  const result = await client.query(`
    SELECT * FROM api_gateways
    WHERE phase = 'Deleting'
      AND locked_at < NOW() - INTERVAL '30 seconds'
    ORDER BY locked_at ASC, updated_at ASC
    LIMIT 1
    FOR UPDATE SKIP LOCKED  -- ← 다른 트랜잭션이 Lock 중이면 skip
  `)

  if (result.rows.length === 0) {
    await client.query('ROLLBACK')
    return null  // Lock 획득 실패
  }

  // Lock 시간 업데이트
  await client.query(`
    UPDATE api_gateways
    SET locked_at = NOW()
    WHERE id = $1
  `, [result.rows[0].id])

  await client.query('COMMIT')
  return result.rows[0]
} catch (err) {
  await client.query('ROLLBACK')
  throw err
} finally {
  client.release()  // Connection Pool에 반환
}
```

**복잡도 증가**:
- ❌ BEGIN/COMMIT 트랜잭션 필요
- ❌ Connection Pool에서 client 획득/해제
- ❌ Rollback 처리
- ❌ 코드가 10배 길어짐
- ❌ 트랜잭션 격리 수준 고려 필요

**방법 2: Optimistic Lock with Version**

```typescript
// version 필드 기반 Optimistic Lock
async function findAndLock(currentVersion: number) {
  const result = await pool.query(`
    UPDATE api_gateways
    SET
      locked_at = NOW(),
      version = version + 1
    WHERE phase = 'Deleting'
      AND locked_at < NOW() - INTERVAL '30 seconds'
      AND version = $1
    RETURNING *
  `, [currentVersion])

  if (result.rows.length === 0) {
    // 다른 프로세스가 먼저 Lock 획득했거나
    // version이 변경됨 (Optimistic Lock 실패)
    return null
  }

  return result.rows[0]
}

// 사용
const gateway = await findAndLock(currentVersion)
if (!gateway) {
  // 재시도 로직 필요
  return
}
```

**추가 고려사항**:
- ✅ 트랜잭션 불필요 (단일 UPDATE)
- ❌ version 필드 추가 필요
- ❌ version 관리 복잡도
- ❌ UPDATE가 0건이면 재시도 필요

### Redis를 추가한다면?

```typescript
// Redlock 알고리즘 (Distributed Lock)
import Redlock from 'redlock'

const redlock = new Redlock([redisClient], {
  retryCount: 3,
  retryDelay: 200
})

async function handleDeletingPhase() {
  const gateway = await db.findOne({ phase: 'Deleting' })
  if (!gateway) return

  // Redis Lock 획득
  const lock = await redlock.lock(
    `gateway:${gateway.gatewayId}`,
    30000  // 30초 TTL
  )

  try {
    await deleteResources(gateway)
  } finally {
    await lock.unlock()
  }
}
```

**장점**:
- ✅ 더 강력한 분산 Lock
- ✅ TTL 자동 관리
- ✅ DB 부하 감소 (Lock을 DB에서 분리)

**단점**:
- ❌ Redis 인프라 추가 필요
- ❌ 네트워크 hop 증가 (latency)
- ❌ Redis 장애 시 전체 시스템 영향
- ❌ Redlock 알고리즘 복잡도

### 결론: MongoDB 선택의 이유

imprun.dev가 MongoDB를 선택한 이유는:

1. **이미 MongoDB 사용 중**: 추가 인프라 불필요
2. **Atomic Operation 지원**: `findOneAndUpdate()` 한 줄로 해결
3. **Task Processing 특성**: 높은 동시성이 필요하지 않음
4. **간단한 구조**: 트랜잭션, Connection Pool 관리 불필요

**만약 PostgreSQL을 사용했다면**:
- `FOR UPDATE SKIP LOCKED` 사용 (트랜잭션 필요)
- 또는 Redis 추가 고려

**만약 높은 동시성이 필요했다면**:
- Redis + Redlock 패턴 고려
- MongoDB/PostgreSQL은 Lock Store로만 사용

MongoDB의 단순함 덕분에 **외부 의존성 없이** 동시성 제어를 구현할 수 있었습니다.

---

## 주의사항

### 1. Lock Timeout 설정

```typescript
readonly lockTimeout = 30  // 30초

// ❌ Timeout이 너무 짧으면?
readonly lockTimeout = 5   // 5초
// → Kubernetes API 호출이 5초 이상 걸리면?
// → 작업 중인데 다른 Pod가 Lock 획득 (중복 처리)

// ❌ Timeout이 너무 길면?
readonly lockTimeout = 300  // 5분
// → Pod 장애 시 5분 동안 작업 중단
// → 복구가 너무 늦음

// ✅ 적절한 Timeout
readonly lockTimeout = 30  // 30초
// → 대부분의 작업은 30초 이내 완료
// → Pod 장애 시 30초 후 자동 복구
```

### 2. findAndLock() 정렬 순서

```typescript
findOneAndUpdate(
  { ... },
  { ... },
  { sort: { lockedAt: 1, updatedAt: 1 } }
  //        ↑ lockedAt이 오래된 순서
  //                      ↑ updatedAt이 오래된 순서
)
```

**왜 이렇게 정렬?**
1. `lockedAt` 오래된 순: Timeout 임박한 작업 우선 처리
2. `updatedAt` 오래된 순: 오래 대기한 작업 우선 처리

### 3. unlock() 누락 방지

```typescript
// ❌ unlock() 누락
async handleDeletingPhase() {
  const gateway = await findAndLock()

  await deleteTriggers()
  // unlock() 호출 안 함!
  // → 30초 동안 다음 단계 진행 안 됨
}

// ✅ 항상 unlock() 호출
async handleDeletingPhase() {
  const gateway = await findAndLock()

  if (hadTriggers) {
    await deleteTriggers()
    return await this.unlock(gatewayId)  // ← 필수!
  }

  // 또는 try-finally
  try {
    await deleteResources()
  } finally {
    await this.unlock(gatewayId)  // ← 보장
  }
}
```

### 4. lockedAt 초기화

```typescript
// 새 Document 생성 시
await db.collection('ApiGateway').insertOne({
  gatewayId,
  state: 'Running',
  phase: 'Creating',
  lockedAt: TASK_LOCK_INIT_TIME,  // ← new Date('1970-01-01')
  createdAt: new Date(),
  updatedAt: new Date()
})
```

**왜 1970-01-01인가?**
- `$lt: new Date(Date.now() - 1000 * 30)` 조건을 항상 만족
- 즉시 Lock 획득 가능

---

## State Machine과의 조합

이 블로그에서는 Optimistic Lock 패턴에 집중했습니다. 하지만 imprun.dev에서는 **State Machine 패턴과 함께** 사용합니다:

```typescript
// State Machine: 무엇을 처리할지
@Cron(CronExpression.EVERY_SECOND)
async tick() {
  this.handleCreatingPhase()   // Phase: Creating 처리
  this.handleDeletingPhase()   // Phase: Deleting 처리
  this.handleDeletedState()    // State: Deleted 처리
}

// Optimistic Lock: 누가 처리할지
async handleDeletingPhase() {
  const gateway = await findAndLock()  // ← 동시성 제어
  if (!gateway) return  // 다른 Pod가 처리 중

  // State Machine에 따라 작업 수행
  if (hadTriggers) await deleteTriggers()
  if (hadFunctions) await deleteFunctions()
  // ...
}
```

State Machine에 대한 자세한 내용은:
- [State Machine 패턴으로 Kubernetes 리소스 생명주기 관리하기](https://blog.imprun.dev/46)

---

## 결론

분산 환경에서 Cron Job 기반 Task Processing을 구현할 때 동시성 제어는 필수입니다.

imprun.dev는 **Optimistic Lock 패턴**으로:
- ✅ 여러 Pod에서 안전한 동시 실행
- ✅ DB 트랜잭션 없이 간단한 구현
- ✅ 자동 Lock 해제로 장애 복구
- ✅ Progressive Backoff로 효율성

**핵심 구현**:
1. **`lockedAt` 필드**: 타임스탬프 기반 Lock
2. **findAndLock()**: Atomic Operation으로 Lock 획득
3. **unlock()/relock()**: 단계별 제어
4. **Timeout 기반 자동 해제**: 서버 장애 시에도 복구

외부 의존성(Redis) 없이도 MongoDB만으로 **안전하고 효율적인 동시성 제어**가 가능합니다.

---

**다음 읽을거리**:
- [State Machine 패턴으로 Kubernetes 리소스 생명주기 관리하기](https://blog.imprun.dev/46) - 무엇을 처리할지
- [imprun.dev GitHub](https://github.com/imprun-dev/imprun)
- [Optimistic Locking - Martin Fowler](https://martinfowler.com/eaaCatalog/optimisticOfflineLock.html)
- [MongoDB Atomic Operations](https://www.mongodb.com/docs/manual/core/write-operations-atomicity/)

---

> "Optimistic Lock은 외부 의존성 없이 MongoDB만으로 분산 환경의 동시성 제어를 가능하게 한다"

🤖 *이 블로그는 [**imprun.dev**](https://imprun.dev) 플랫폼 개발 과정에서 실제로 구현한 Optimistic Lock 기반 동시성 제어 시스템을 소개합니다.*
