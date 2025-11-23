# MongoDB 연결 타임아웃 50% 해결기: Connection Pool 분리가 부른 나비효과

**작성일:** 2025-11-02
**카테고리:** MongoDB, NestJS, Connection Pool, Debugging
**난이도:** 중급

---

## TL;DR

- **문제**: 로컬 개발 환경에서 MongoDB 연결 시 50% 확률로 30초 타임아웃 발생
- **원인**: 로컬 개발을 위한 Connection Pool 분리 설계가 Hot-Reload 환경에서 **673,098개의 연결 생성** 초래
- **핵심**: `minPoolSize=1`이어도 **여러 MongoClient 인스턴스**가 동시에 초기화되면 연결 경합(Connection Race) 발생
- **해결**: Module-level Singleton 패턴 + Connection Pool 통합으로 타임아웃 0%로 감소

---

## 들어가며

[**imprun.dev**](https://imprun.dev)는 "API 개발부터 AI 통합까지, 모든 것을 하나로 제공"하는 Kubernetes 기반 API 플랫폼입니다. CloudFunction과 독립적인 Gateway 데이터베이스를 제공하기 위해 MongoDB 연결을 **환경별로 분리**하는 아키텍처를 설계했습니다.

**우리가 마주한 질문**:
- ❓ `minPoolSize=1`인데 왜 연결 경합이 발생하나?
- ❓ 네트워크는 정상(ping 10ms)인데 왜 타임아웃이 50%나 나올까?
- ❓ 로컬 개발을 위한 연결 분리가 오히려 독이 된 건 아닐까?

**검증 과정**:

1. **타임아웃 값 증가 (30초 → 60초)**: ❌ 동일한 현상, 시간만 더 걸림
2. **Tailscale 네트워크 최적화**: ❌ ping 10ms로 정상, 원인 아님
3. **MongoDB 로그 분석**: ✅ `totalCreated: 673,098` 발견! 🚨
4. **Connection Pool 통합 + Hot-Reload 안전성**: ✅ 타임아웃 0%로 개선

**결론**:
- ✅ Hot-Reload 환경에서 Module-level Singleton으로 연결 재사용
- ✅ 여러 곳에 흩어진 MongoClient 생성 지점 통합
- ✅ Connection Pool 설정 최적화로 안정성 확보

이 글은 **imprun.dev 플랫폼 구축 경험**을 바탕으로, 로컬 개발 최적화가 오히려 프로덕션 품질을 저하시킨 사례와 해결 과정을 공유합니다.

---

## 배경: 연결을 분리한 이유

### 원래 설계: 단순했던 시절

처음에는 MongoDB 연결이 2가지만 있었습니다:

```typescript
// 초기 설계 (2가지 연결)
- controlConnection: API 서버가 직접 접근
- defaultConnection: Runtime Pod가 접근
```

**목적**:
- API 서버와 Runtime Pod의 네트워크 경로 분리
- 개발 환경에서 포트포워딩/NodePort 유연하게 사용

### 진화: 요구사항의 증가

프로젝트가 성장하면서 연결 종류가 **4가지**로 증가했습니다:

```typescript
// 현재 설계 (4가지 연결)
1. SystemDatabase    - 사용자, Gateway, Function 메타데이터
2. TrafficDatabase   - 트래픽 로그 (선택적)
3. AppDatabase       - Gateway별 독립 데이터베이스
4. RuntimeDatabase   - CloudFunction Runtime Pod 전용
```

**각 연결마다 별도 MongoClient + Connection Pool**이 생성되었습니다.

### 문제의 시작

```typescript
// system-database.ts
export class SystemDatabase {
  static ready = this.initialize()  // ⚠️ 클래스 로드 시 즉시 실행

  static async initialize() {
    this._client = new MongoClient(DATABASE_URL, {
      maxPoolSize: 3,
      minPoolSize: 1,  // 최소 1개만 생성하면 되는데?
    })
    await this._client.connect()
  }
}

// database.service.ts - 권한 관리용 임시 연결
async grantWritePermission(...) {
  const client = new MongoClient(connectionUri)  // ⚠️ 또 다른 MongoClient
  await client.connect()
  // ...
  await client.close()
}
```

**문제점**:
1. **4가지 연결** × **각각 MongoClient** = 최소 4개의 Connection Pool
2. **Hot-Reload**: NestJS `pnpm dev` 시 파일 변경마다 모듈 재로드
3. **임시 연결**: 권한 관리 등 일시적 작업마다 새 MongoClient 생성

결과: **연결 생성이 기하급수적으로 증가** 🔥

---

## 증상: 50% 확률의 타임아웃

### 에러 로그

```bash
[Nest] 89532  - 2025. 11. 02. 오전 9:21:50     LOG [SystemDatabase] Connecting to system database
[Nest] 89532  - 2025. 11. 02. 오전 9:21:50   DEBUG [SystemDatabase] Attempting to connect...

# 3분 15초 대기...

[Nest] 89532  - 2025. 11. 02. 오전 9:25:04   ERROR [SystemDatabase] Failed to connect after 194135ms
[Nest] 89532  - 2025. 11. 02. 오전 9:25:04   ERROR MongoServerSelectionError: connection timed out
```

### 환경 정보

**개발 환경**:
- OS: Windows 11
- 네트워크: Tailscale VPN (원격 Kubernetes 접근)
- MongoDB: Kubernetes NodePort (30017)
- Connection String: `mongodb://root:***@100.64.0.2:30017/sys_db?authSource=admin&directConnection=true`

**네트워크 상태**:
```bash
$ ping 100.64.0.2 -n 5
평균 = 10ms  # ✅ 네트워크 정상!
```

**혼란스러운 점**:
- 네트워크는 정상인데 왜 타임아웃?
- 가끔은 1초 만에 연결되는데 왜 50%는 실패?
- `minPoolSize=1`인데 왜 Pool 경합?

---

## 디버깅: 진실을 찾아서

### 1단계: MongoDB 로그 확인

```bash
$ kubectl logs -n imprun-system mongodb-0 --tail=50 | grep connection

# 연결이 계속 끊어지고 재연결되는 패턴 발견
{"msg":"Connection ended","connectionId":673024,"connectionCount":36}
{"msg":"Connection accepted","connectionId":673059,"connectionCount":37}
{"msg":"Connection ended","connectionId":673025,"connectionCount":36}
{"msg":"Connection accepted","connectionId":673060,"connectionCount":37}
...
```

**의심스러운 점**: 왜 이렇게 자주 연결이 끊어질까?

### 2단계: MongoDB 연결 통계 확인

```bash
$ kubectl exec -n imprun-system mongodb-0 -- \
  mongosh -u root -p *** --authenticationDatabase admin --quiet \
  --eval "db.serverStatus().connections"

{
  current: 41,           # 현재 활성 연결
  available: 999959,     # 사용 가능한 연결
  totalCreated: 673098,  # 🚨 이게 뭐야?!
  active: 5
}
```

**충격적인 발견**: `totalCreated: 673,098`

- MongoDB가 시작된 이후 **67만 개 이상의 연결**이 생성됨
- 현재는 41개만 활성화
- 즉, **연결이 생성되었다가 즉시 폐기되는 패턴** 반복

### 3단계: 연결 생성 지점 추적

```bash
$ grep -r "new MongoClient" server/src/ --include="*.ts"

server/src/system-database.ts:27:    this._client = new MongoClient(...)
server/src/system-database.ts:90:    this._client = new MongoClient(...)  # TrafficDatabase
server/src/database/mongo.service.ts:46:    const client = new MongoClient(...)
server/src/database/mongo.service.ts:82:    const client = new MongoClient(...)
server/src/database/database.service.ts:159:    const client = new MongoClient(...)  # revokeWritePermission
server/src/database/database.service.ts:185:    const client = new MongoClient(...)  # grantWritePermission
server/src/database/database.service.ts:211:    const client = new MongoClient(...)  # getUserPermission
```

**발견된 문제**:
1. **7곳에서 MongoClient 생성**
2. **Hot-Reload**: `system-database.ts`가 재로드될 때마다 새 client 생성
3. **임시 연결**: 권한 관리 메서드 3개가 각각 새 client 생성

---

## 근본 원인: Connection Race Condition

### 문제 1: Hot-Reload와 Static Initialization

```typescript
// ❌ Before: 클래스 로드 시마다 새 연결 생성
export class SystemDatabase {
  static ready = this.initialize()  // 파일이 import될 때마다 실행!

  static async initialize() {
    this._client = new MongoClient(DATABASE_URL, {
      maxPoolSize: 3,
      minPoolSize: 1,
    })
    await this._client.connect()
  }
}
```

**Hot-Reload 시나리오**:
1. 파일 저장 (예: `user.service.ts` 수정)
2. NestJS가 변경된 모듈 재로드
3. `SystemDatabase` import → `static ready = this.initialize()` 재실행
4. **새로운 MongoClient 생성** (이전 client는 정리되지 않음)
5. 반복...

### 문제 2: 여러 MongoClient의 동시 초기화

```mermaid
sequenceDiagram
    participant App as NestJS App
    participant Sys as SystemDatabase
    participant Trf as TrafficDatabase
    participant Svc as DatabaseService
    participant Mongo as MongoDB

    App->>Sys: await SystemDatabase.ready
    App->>Trf: await TrafficDatabase.ready
    App->>Svc: initService.init()

    Note over Sys,Trf: 동시에 connect() 호출
    Sys->>Mongo: new MongoClient + connect() (pool: 3)
    Trf->>Mongo: new MongoClient + connect() (pool: 3)
    Svc->>Mongo: new MongoClient + connect() (임시)

    Note over Mongo: 연결 대기열 폭주!
    Mongo-->>Sys: ⏰ Timeout
    Mongo-->>Trf: ⏰ Timeout
```

**핵심 인사이트**:
- `minPoolSize=1`은 **각 MongoClient당** 1개
- **7개의 MongoClient**가 있으면 최소 **7개의 연결** 동시 시도
- Hot-Reload로 이전 client들이 정리되지 않으면 **기하급수적 증가**

### 문제 3: Connection Pool이 재사용되지 않음

```typescript
// database.service.ts - 권한 관리 시 매번 새 연결
async revokeWritePermission(name: string, username: string, region: Region) {
  const client = new MongoClient(region.databaseConf.connectionUri)  // ❌ 새 Pool

  try {
    await client.connect()
    // 권한 변경...
  } finally {
    await client.close()  // Pool 폐기
  }
}
```

**왜 이렇게 설계했나?**
- 임시 작업이니까 연결을 즉시 닫아야 한다고 생각
- 하지만 **새 Pool 생성 비용 >> 기존 Pool 재사용 비용**

---

## 해결책: Connection Pool 통합과 Singleton

### 1. Module-Level Singleton (Hot-Reload 안전)

```typescript
// ✅ After: 모듈 레벨 변수로 Singleton 보장
// Global singleton instances (hot-reload safe)
let _systemClient: MongoClient | null = null
let _systemReady: Promise<MongoClient> | null = null
let _systemInitializing = false

export class SystemDatabase {
  private static readonly logger = new Logger(SystemDatabase.name)

  static get ready(): Promise<MongoClient> {
    if (!_systemReady) {
      _systemReady = this.initialize()
    }
    return _systemReady  // 항상 동일한 Promise 반환
  }

  private static async initialize() {
    // 이미 초기화 중이면 대기
    if (_systemInitializing && _systemReady) {
      this.logger.debug('System database initialization already in progress, waiting...')
      return _systemReady
    }

    // 이미 연결되어 있으면 재사용
    if (_systemClient) {
      try {
        await _systemClient.db().admin().ping()
        this.logger.debug('Reusing existing system database connection')
        return _systemClient
      } catch (err) {
        this.logger.warn('Existing connection is dead, reconnecting...')
        await _systemClient.close()
        _systemClient = null
      }
    }

    _systemInitializing = true
    _systemClient = new MongoClient(DATABASE_URL, {
      connectTimeoutMS: 30000,
      serverSelectionTimeoutMS: 30000,
      maxPoolSize: 3,        // 환경 변수로 제어
      minPoolSize: 1,
      maxIdleTimeMS: 300000, // 5분 - 재연결 빈도 감소
    })

    try {
      const client = await _systemClient.connect()
      this.logger.log(`Connected to system database`)
      _systemInitializing = false
      return client
    } catch (err) {
      _systemClient = null
      _systemReady = null
      _systemInitializing = false
      throw err
    }
  }
}
```

**핵심 개선 사항**:
1. ✅ **Module-level 변수**: 클래스 재로드와 무관하게 유지
2. ✅ **Race Condition 방지**: `_systemInitializing` 플래그로 중복 초기화 차단
3. ✅ **연결 재사용**: 기존 연결 상태 확인 후 재사용
4. ✅ **Lazy Initialization**: `ready` getter로 필요 시에만 초기화

### 2. Connection Pool 설정 최적화

```typescript
// Before: 짧은 타임아웃 + 작은 Pool
{
  connectTimeoutMS: 10000,
  serverSelectionTimeoutMS: 15000,
  maxPoolSize: 3,
  minPoolSize: 1,
  maxIdleTimeMS: 120000,  // 2분
}

// After: 안정적인 타임아웃 + 최적화된 Pool
{
  connectTimeoutMS: 30000,           // 30초 (NodePort 환경 고려)
  serverSelectionTimeoutMS: 30000,   // 30초
  maxPoolSize: 3,                    // 개발 환경 적정 크기
  minPoolSize: 1,
  maxIdleTimeMS: 300000,             // 5분 - 재연결 빈도 대폭 감소
  waitQueueTimeoutMS: 10000,         // 대기열 타임아웃
  retryWrites: true,                 // 쓰기 재시도
  retryReads: true,                  // 읽기 재시도
  directConnection: true,            // Replica Set discovery 건너뛰기
}
```

**변경 이유**:
- `maxIdleTimeMS: 5분`: 개발 중 유휴 상태에서도 연결 유지
- `connectTimeoutMS: 30초`: Windows + NodePort + Tailscale 환경 고려
- `retryReads/Writes`: 일시적 네트워크 오류 대응

### 3. 임시 연결 Pool 크기 축소

```typescript
// database.service.ts - 권한 관리용 임시 연결
async revokeWritePermission(name: string, username: string, region: Region) {
  // ✅ Pool 크기 축소 + 빠른 정리
  const client = new MongoClient(connectionUri, {
    maxPoolSize: 3,        // 임시 작업용 작은 풀
    minPoolSize: 0,        // 필요할 때만 생성
    maxIdleTimeMS: 30000,  // 30초 후 자동 정리
  })

  try {
    await client.connect()
    // 권한 변경...
  } finally {
    await client.close()
  }
}
```

**개선 효과**:
- 임시 연결이 필요한 경우에도 Connection Pool 크기 최소화
- `maxIdleTimeMS: 30초`로 빠른 정리

### 4. MongoService 옵션 통일

```typescript
// mongo.service.ts - 일시적 작업용 공통 옵션
private readonly mongoOptions = {
  connectTimeoutMS: 30000,
  serverSelectionTimeoutMS: 30000,
  socketTimeoutMS: 60000,
  maxPoolSize: 3,              // 작은 풀
  minPoolSize: 0,              // 필요할 때만 생성
  maxIdleTimeMS: 30000,        // 빠른 정리
  waitQueueTimeoutMS: 10000,
  retryWrites: true,
  retryReads: true,
  directConnection: true,
}
```

---

## 결과: 타임아웃 0%로 개선

### Before (수정 전)

**MongoDB 통계**:
```javascript
{
  current: 41,
  totalCreated: 673098,  // 🚨 67만 개!
  active: 5
}
```

**서버 시작**:
```bash
[Nest] LOG [SystemDatabase] Connecting to system database
[Nest] DEBUG [SystemDatabase] Attempting to connect...

# 30초 ~ 3분 대기 (50% 확률)

[Nest] ERROR Failed to connect after 194135ms
```

**개발 경험**:
- ❌ 타임아웃 발생률: **50%**
- ❌ 성공 시에도 10-30초 소요
- ❌ Hot-Reload 시마다 불안정

### After (수정 후)

**MongoDB 통계** (서버 5회 재시작 후):
```javascript
{
  current: 5,              // 대폭 감소!
  totalCreated: 673123,    // +25개 (5회 × 5개)
  active: 3
}
```

**서버 시작**:
```bash
[Nest] LOG [SystemDatabase] Connecting to system database
[Nest] DEBUG [SystemDatabase] Pool size: 3/1 (max/min)
[Nest] DEBUG [SystemDatabase] Attempting to connect...
[Nest] LOG [SystemDatabase] Connected to system database (523ms)  # ✅ 0.5초!
[Nest] DEBUG [SystemDatabase] Database ping successful
```

**Hot-Reload 시**:
```bash
[Nest] DEBUG [SystemDatabase] Reusing existing system database connection  # ✅ 재사용!
```

**개발 경험**:
- ✅ 타임아웃 발생률: **0%**
- ✅ 연결 시간: **0.5초** (이전 10-30초)
- ✅ Hot-Reload 안정성: **기존 연결 재사용**

---

## 교훈: 로컬 최적화의 함정

### 설계 의도는 좋았다

**Connection Pool 분리 설계**:
```typescript
// 의도: 개발 환경과 프로덕션 환경 분리
- controlConnection   → API 서버 (포트포워딩/로컬)
- runtimeConnection   → Runtime Pod (클러스터 내부)
- appConnection       → Gateway별 독립 DB
- trafficConnection   → 트래픽 로그 (선택적)
```

**장점**:
- ✅ 유연한 로컬 개발
- ✅ 명확한 책임 분리
- ✅ 프로덕션 보안 (최소 권한)

**하지만...**

### 함정 1: Hot-Reload를 고려하지 않음

```typescript
// ❌ 문제: 파일 저장할 때마다 새 연결 생성
export class SystemDatabase {
  static ready = this.initialize()  // 클래스 로드 = 새 연결
}
```

**교훈**:
> **Hot-Reload 환경에서는 Module-level Singleton을 사용하라**

### 함정 2: minPoolSize=1의 착각

```
minPoolSize=1 ≠ 전체 연결 1개
minPoolSize=1 = 각 MongoClient당 1개

7개 MongoClient × minPoolSize=1 = 최소 7개 연결
```

**교훈**:
> **Connection Pool 크기는 MongoClient 인스턴스 개수와 곱셈 관계**

### 함정 3: "임시 연결"이라는 착각

```typescript
// ❌ 매번 새 Pool 생성 (비효율)
async tempOperation() {
  const client = new MongoClient(url)
  await client.connect()  // Pool 생성 (느림)
  // 작업...
  await client.close()    // Pool 폐기
}
```

**교훈**:
> **임시 작업이어도 Connection Pool은 재사용하는 것이 효율적**

---

## 마무리

### 핵심 요약

1. **로컬 개발 최적화가 오히려 독**: Connection Pool 분리 설계가 Hot-Reload 환경에서 연결 폭주 초래
2. **Module-level Singleton**: Hot-Reload에도 안전한 연결 재사용 패턴
3. **Connection Pool은 곱셈**: 여러 MongoClient 인스턴스의 Pool 크기 합산 주의
4. **임시 연결도 Pool 재사용**: 새 Pool 생성 비용 >> 기존 Pool 재사용 비용

### 언제 Connection Pool을 분리할까?

**❌ 분리하지 말아야 할 경우:**
- 같은 MongoDB 서버에 접근
- Hot-Reload가 빈번한 개발 환경
- 연결 설정이 거의 동일

**✅ 분리해야 할 경우:**
- 완전히 다른 MongoDB 클러스터
- 보안 요구사항 (격리된 권한)
- 트래픽 특성이 극단적으로 다름 (예: 실시간 vs 배치)

### 실제 적용 결과

**imprun.dev 환경:**
- ✅ 타임아웃 발생률: 50% → **0%**
- ✅ 연결 시간: 10-30초 → **0.5초**
- ✅ MongoDB 연결 생성: 67만 개 → **5회 재시작당 25개**

**개발 경험:**
- 서버 시작 대기: **3분 → 5초**
- Hot-Reload 안정성: **매우 높음**
- 스트레스 수준: **😫 → 😊**

---

## 참고 자료

### 공식 문서
- [MongoDB Node.js Driver - Connection Pooling](https://www.mongodb.com/docs/drivers/node/current/fundamentals/connection/connection-pooling/)
- [NestJS - Hot Reload](https://docs.nestjs.com/recipes/hot-reload)

### 관련 글
- [Kubernetes 환경에서 MongoDB 연결 최적화](https://blog.imprun.dev/50)
- [NestJS Dependency Injection 완벽 가이드](https://blog.imprun.dev/45)

---

**태그:** #MongoDB #ConnectionPool #NestJS #Debugging #HotReload #Performance

**저자:** imprun.dev 팀

---

> "로컬 개발 최적화가 프로덕션 품질을 저하시킬 수 있다. Hot-Reload와 Connection Pool은 신중하게 다루자."

🤖 *이 블로그는 실제 프로덕션 환경에서 MongoDB 연결 타임아웃을 디버깅한 경험을 바탕으로 작성되었습니다.*

---

**질문이나 피드백은 블로그 댓글에 남겨주세요!**
