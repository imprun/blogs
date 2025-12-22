# GORM 실무 트러블슈팅: 운영 환경에서 만난 함정들

> **작성일**: 2025년 12월 22일
> **카테고리**: Backend, Go, Troubleshooting
> **키워드**: Go, GORM, PostgreSQL, Troubleshooting, TimeZone

## 요약

GORM을 사용하면서 실제 프로덕션 환경에서 마주친 Edge Cases와 해결 방법을 정리한다. 타임존(KST/UTC) 불일치 문제, Zero Value 업데이트 함정, AutoMigrate와 init.sql의 권한 충돌, 그리고 Upsert, Hook 전파, 연결 끊김 등 실무에서 시간을 허비하기 쉬운 함정들을 다룬다.

## 1. 타임존(TimeZone) 불일치 문제

KST(Asia/Seoul) 환경에서 운영할 때 가장 흔하게 겪는 문제다.

### 문제 상황

```go
// 한국 시간 2025-12-22 15:00:00에 레코드 생성
user := User{Name: "test"}
db.Create(&user)

// DB 조회 결과: created_at = 2025-12-22 06:00:00 (9시간 차이!)
```

또는 반대로:

```go
// DB에 저장된 시간: 2025-12-22 15:00:00 (UTC)
// Go에서 조회 후 출력: 2025-12-23 00:00:00 (KST로 변환됨)
```

### 원인 분석

타임존 불일치는 여러 지점에서 발생한다:

```
[Go 애플리케이션] → [GORM] → [PostgreSQL Driver] → [PostgreSQL Server]
     time.Local       DSN TimeZone      세션 timezone       서버 timezone
```

| 구간 | 설정 위치 | 기본값 |
|------|-----------|--------|
| Go 런타임 | `time.Local`, `TZ` 환경변수 | 시스템 로케일 |
| DSN | `TimeZone=Asia/Seoul` | 서버 기본값 |
| PostgreSQL 세션 | `SET timezone` | postgresql.conf |
| PostgreSQL 서버 | `timezone` in postgresql.conf | OS 설정 |

### 해결 방법

**권장: 백엔드는 UTC, 프론트엔드에서 로컬 타임존 변환**

이 방식이 표준적이고, 글로벌 서비스나 멀티 타임존 환경에서 유리하다.

```go
// Go 백엔드: DSN에 UTC 명시
dsn := "host=localhost user=app dbname=myapp TimeZone=UTC"

// 시간 저장/처리는 항상 UTC
now := time.Now().UTC()
user.CreatedAt = now
```

```typescript
// 프론트엔드 (TypeScript): 로컬 타임존으로 변환
const utcDate = new Date(response.createdAt); // ISO 8601 문자열
const localDate = utcDate.toLocaleString('ko-KR', { timeZone: 'Asia/Seoul' });

// 또는 date-fns-tz 사용
import { formatInTimeZone } from 'date-fns-tz';
const kstDate = formatInTimeZone(utcDate, 'Asia/Seoul', 'yyyy-MM-dd HH:mm:ss');
```

**API 응답 형식:**

```json
{
  "createdAt": "2025-12-22T06:00:00Z"  // ISO 8601 UTC
}
```

**대안: 모든 구간 KST 통일 (한국 전용 서비스)**

글로벌 서비스가 아니고, 한국 사용자만 대상이라면 KST로 통일하는 것도 가능하다.

```go
// DSN
dsn := "host=localhost user=app dbname=myapp TimeZone=Asia/Seoul"

// Go 런타임 (main 또는 init)
func init() {
    loc, err := time.LoadLocation("Asia/Seoul")
    if err != nil {
        panic(err)
    }
    time.Local = loc
}
```

**Docker 환경에서 TZ 설정:**

```yaml
# docker-compose.yml
services:
  api:
    environment:
      TZ: UTC  # 백엔드는 UTC 권장

  postgres:
    environment:
      TZ: UTC
```

### TIMESTAMPTZ vs TIMESTAMP

PostgreSQL에서 타임존 처리:

| 타입 | 저장 방식 | 권장 |
|------|-----------|------|
| `TIMESTAMP` | 타임존 정보 없이 저장 | 비권장 |
| `TIMESTAMPTZ` | UTC로 변환 후 저장, 조회 시 세션 타임존으로 변환 | 권장 |

GORM에서 `time.Time` 필드는 기본적으로 `TIMESTAMPTZ`로 매핑된다.

### 디버깅 방법

```sql
-- PostgreSQL 세션 타임존 확인
SHOW timezone;

-- 서버 타임존 확인
SELECT current_setting('TIMEZONE');

-- 현재 시간 비교
SELECT NOW(), NOW() AT TIME ZONE 'UTC', NOW() AT TIME ZONE 'Asia/Seoul';
```

```go
// Go 타임존 확인
fmt.Println("Local:", time.Local)
fmt.Println("Now:", time.Now())
fmt.Println("Now UTC:", time.Now().UTC())
```

---

## 2. Zero Value 업데이트 문제

### 문제 상황

구조체를 사용하여 업데이트할 때, Go의 Zero Value가 무시되는 현상이 발생한다.

```go
type User struct {
    ID       uuid.UUID
    Name     string
    IsActive bool
}

// IsActive를 false로 변경하고 싶음
user.IsActive = false
db.Model(&user).Updates(user)
```

**기대 결과:** `is_active`가 `false`로 업데이트
**실제 결과:** `is_active`가 변경되지 않음

### 원인 분석

GORM은 구조체 업데이트 시 Zero Value(`false`, `0`, `""`)를 "변경 의도 없음"으로 간주하고 무시한다. 이는 의도하지 않은 필드 덮어쓰기를 방지하기 위한 설계다.

```go
// GORM 내부 동작 (의사 코드)
for field, value := range struct {
    if value == zeroValue {
        continue // 무시
    }
    updateFields[field] = value
}
```

### 해결 방법

**방법 1: Map 사용 (권장)**

```go
db.Model(&user).Updates(map[string]interface{}{
    "is_active": false,
})
```

**방법 2: Select로 필드 명시**

```go
db.Model(&user).Select("IsActive").Updates(user)
```

**방법 3: 포인터 타입 사용**

```go
type User struct {
    ID       uuid.UUID
    Name     string
    IsActive *bool `gorm:"default:true"`
}

falseVal := false
user.IsActive = &falseVal
db.Model(&user).Updates(user)
```

### 권장 패턴

업데이트 함수에서 Map을 일관되게 사용:

```go
func (r *userRepository) UpdateStatus(ctx context.Context, id uuid.UUID, isActive bool) error {
    return r.db.WithContext(ctx).
        Model(&entity.User{}).
        Where("id = ?", id).
        Updates(map[string]interface{}{
            "is_active": isActive,
        }).Error
}
```

---

## 2. AutoMigrate와 init.sql 권한 충돌

### 문제 상황

Docker Compose로 PostgreSQL을 구성할 때, `init.sql`로 테이블을 생성하고 애플리케이션에서 AutoMigrate를 실행하면 다음 에러가 발생한다.

```text
ERROR: permission denied for table users
```

또는 테이블이 존재함에도 재생성을 시도:

```text
ERROR: relation "users" already exists (SQLSTATE 42P07)
```

### 원인 분석

**시나리오 1: 권한 문제**

```yaml
# docker-compose.yml
services:
  postgres:
    environment:
      POSTGRES_USER: postgres      # 슈퍼유저
      POSTGRES_DB: myapp
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
```

```sql
-- init.sql (postgres 슈퍼유저로 실행됨)
CREATE TABLE users (
    id UUID PRIMARY KEY,
    name VARCHAR(100)
);
```

```go
// 애플리케이션 (app_user로 접속)
dsn := "user=app_user password=secret dbname=myapp"
db.AutoMigrate(&User{})  // app_user는 postgres가 만든 테이블 수정 권한 없음
```

테이블 소유자가 `postgres`이고, 애플리케이션이 `app_user`로 접속하면 테이블 스키마를 수정할 권한이 없다.

**시나리오 2: 스키마 불일치**

```sql
-- init.sql
CREATE TABLE myschema.users (...);
```

```go
// 애플리케이션은 public 스키마를 봄
dsn := "dbname=myapp"  // search_path 미지정
db.Migrator().HasTable(&User{})  // public.users 검사 → false
db.AutoMigrate(&User{})          // public.users 생성 시도 → 실패 (이미 myschema에 존재)
```

### 해결 방법

**방법 1: 스키마 관리 단일화 (권장)**

`init.sql`은 데이터베이스와 사용자 생성만 담당하고, 테이블 생성은 애플리케이션에 위임:

```sql
-- init.sql
CREATE USER app_user WITH PASSWORD 'secret';
CREATE DATABASE myapp OWNER app_user;
GRANT ALL PRIVILEGES ON DATABASE myapp TO app_user;
```

```go
// 애플리케이션에서 테이블 생성
db.AutoMigrate(&User{}, &Post{})
```

**방법 2: init.sql에서 권한 부여**

```sql
-- init.sql
CREATE TABLE users (...);
GRANT ALL ON TABLE users TO app_user;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_user;
```

**방법 3: 동일 사용자로 통일**

```yaml
services:
  postgres:
    environment:
      POSTGRES_USER: app_user     # 동일 사용자
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: myapp
```

### 진단 명령어

```sql
-- 테이블 소유자 확인
SELECT tablename, tableowner FROM pg_tables WHERE schemaname = 'public';

-- 현재 사용자 확인
SELECT current_user, session_user;

-- 권한 확인
SELECT grantee, privilege_type
FROM information_schema.role_table_grants
WHERE table_name = 'users';
```

---

## 3. AutoMigrate "relation already exists" (SQLSTATE 42P07)

### 문제 상황

```text
ERROR: relation "regions" already exists (SQLSTATE 42P07)
```

테이블이 이미 존재함에도 GORM이 CREATE TABLE을 시도한다.

### 원인 분석

GORM의 `AutoMigrate`는 내부적으로 `Migrator().HasTable()`로 테이블 존재 여부를 확인한다. 이 검사가 False Negative를 반환하면 CREATE를 시도한다.

**주요 원인:**

1. **search_path 불일치**: 테이블이 다른 스키마에 존재
2. **권한 문제**: information_schema 조회 권한 없음
3. **대소문자 이슈**: PostgreSQL은 기본 소문자, 수동 생성 시 대문자 포함 가능

### 해결 방법

**방법 1: DSN에 search_path 명시**

```go
dsn := "host=localhost user=app dbname=myapp search_path=public"
```

**방법 2: TableName으로 스키마 명시**

```go
func (Region) TableName() string {
    return "public.regions"
}
```

**방법 3: 조건부 마이그레이션**

```go
if !db.Migrator().HasTable(&Region{}) {
    if err := db.AutoMigrate(&Region{}); err != nil {
        log.Printf("Migration skipped or failed: %v", err)
    }
}
```

---

## 4. PostgreSQL DSN 옵션 이해

### 자주 사용하는 DSN 옵션

```go
dsn := "host=localhost port=5432 user=app password=secret dbname=myapp sslmode=disable TimeZone=UTC search_path=public connect_timeout=10"
```

| 옵션 | 설명 | 기본값 |
|------|------|--------|
| `host` | 서버 주소 | `localhost` |
| `port` | 포트 | `5432` |
| `user` | 사용자 | (필수) |
| `password` | 비밀번호 | (없음) |
| `dbname` | 데이터베이스 | (필수) |
| `sslmode` | SSL 모드 | `prefer` |
| `TimeZone` | 세션 타임존 | 서버 기본값 |
| `search_path` | 스키마 검색 경로 | `"$user", public` |
| `connect_timeout` | 연결 타임아웃(초) | `0` (무한) |
| `application_name` | 클라이언트 식별자 | (없음) |

### SSL 모드 옵션

| 값 | 설명 |
|----|------|
| `disable` | SSL 비활성화 |
| `require` | SSL 필수 (인증서 검증 안함) |
| `verify-ca` | CA 인증서 검증 |
| `verify-full` | CA + 호스트명 검증 |

### 환경별 권장 설정

**개발 환경:**

```go
dsn := "host=localhost user=dev password=dev dbname=myapp_dev sslmode=disable"
```

**프로덕션 환경:**

```go
dsn := "host=db.example.com user=app password=*** dbname=myapp sslmode=require connect_timeout=10 application_name=api-server"
```

---

## 5. Preload 순서와 N+1 문제

### 문제 상황

```go
var users []User
db.Find(&users)

for _, user := range users {
    fmt.Println(user.Organization.Name)  // N+1 쿼리 발생!
}
```

### 해결 방법

**Preload 사용:**

```go
db.Preload("Organization").Find(&users)
```

**중첩 Preload:**

```go
db.Preload("Organizations", func(db *gorm.DB) *gorm.DB {
    return db.Order("is_primary DESC")
}).Preload("Organizations.Organization").Find(&users)
```

**조건부 Preload:**

```go
db.Preload("Posts", func(db *gorm.DB) *gorm.DB {
    return db.Where("is_published = ?", true).Limit(5)
}).Find(&users)
```

---

## 6. Soft Delete 주의사항

### 문제 상황

Soft Delete된 레코드가 조회되지 않아 "레코드 없음" 오류가 발생한다.

```go
// 삭제
db.Delete(&user)

// 이후 조회
db.First(&user, id)  // ErrRecordNotFound (삭제된 레코드)
```

### 해결 방법

**삭제된 레코드 포함 조회:**

```go
db.Unscoped().First(&user, id)
```

**영구 삭제:**

```go
db.Unscoped().Delete(&user)
```

**모든 레코드 조회 (삭제 포함):**

```go
db.Unscoped().Find(&users)
```

---

## 7. 복합 조건 쿼리 시 괄호 문제

### 문제 상황

```go
// 의도: (A OR B) AND C
db.Where("status = ? OR status = ?", "active", "pending").
   Where("org_id = ?", orgID).Find(&items)

// 실제 SQL: status = 'active' OR status = 'pending' AND org_id = '...'
// PostgreSQL에서 AND가 OR보다 우선순위 높음 → 의도와 다른 결과
```

### 해결 방법

**OR 그룹화:**

```go
db.Where(
    db.Where("status = ?", "active").Or("status = ?", "pending"),
).Where("org_id = ?", orgID).Find(&items)
```

**Raw 조건 사용:**

```go
db.Where("(status = ? OR status = ?) AND org_id = ?", "active", "pending", orgID).Find(&items)
```

**IN 절 사용 (권장):**

```go
db.Where("status IN ?", []string{"active", "pending"}).
   Where("org_id = ?", orgID).Find(&items)
```

---

## 8. 트랜잭션 내 에러 핸들링

### 문제 상황

트랜잭션 내에서 에러가 발생했는데 롤백되지 않음:

```go
db.Transaction(func(tx *gorm.DB) error {
    tx.Create(&user)
    if someCondition {
        log.Println("Error occurred")
        // return 없음 → 커밋됨!
    }
    return nil
})
```

### 해결 방법

**항상 error 반환:**

```go
db.Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&user).Error; err != nil {
        return err  // 롤백
    }
    if someCondition {
        return errors.New("validation failed")  // 롤백
    }
    return nil  // 커밋
})
```

---

## 9. Upsert (ON CONFLICT) 사용 시 주의사항

### 문제 상황

```go
// 중복 시 업데이트하려고 Clauses 사용
db.Clauses(clause.OnConflict{
    Columns:   []clause.Column{{Name: "email"}},
    DoUpdates: clause.AssignmentColumns([]string{"name", "updated_at"}),
}).Create(&user)

// 결과: updated_at이 업데이트되지 않음
```

### 원인 분석

`AssignmentColumns`는 충돌 시 해당 컬럼을 새 값으로 업데이트하지만, `updated_at`은 GORM의 자동 업데이트 대상이 아닌 경우 갱신되지 않는다.

### 해결 방법

```go
// 방법 1: UpdateAll 사용
db.Clauses(clause.OnConflict{
    Columns:   []clause.Column{{Name: "email"}},
    UpdateAll: true,  // 모든 컬럼 업데이트
}).Create(&user)

// 방법 2: 명시적으로 시간 포함
now := time.Now()
user.UpdatedAt = now
db.Clauses(clause.OnConflict{
    Columns:   []clause.Column{{Name: "email"}},
    DoUpdates: clause.Assignments(map[string]interface{}{
        "name":       user.Name,
        "updated_at": now,
    }),
}).Create(&user)
```

---

## 10. Hook이 Preload에서 동작하지 않음

### 문제 상황

```go
func (u *User) AfterFind(tx *gorm.DB) error {
    u.FullName = u.FirstName + " " + u.LastName
    return nil
}

// Main 쿼리에서는 동작
db.First(&user, id)  // AfterFind 호출됨

// Preload에서는 동작 안함
db.Preload("Author").Find(&posts)  // Author의 AfterFind 호출 안됨
```

### 원인 분석

GORM의 Preload는 별도 쿼리로 실행되며, Hook 호출이 기본적으로 비활성화되어 있다.

### 해결 방법

```go
// Preload 내에서 Hook을 수동 호출하거나, 조회 후 처리
var posts []Post
db.Preload("Author").Find(&posts)

for i := range posts {
    posts[i].Author.AfterFind(db)  // 수동 호출
}

// 또는 Preload 대신 Joins 사용 (단, 구조가 달라짐)
db.Joins("Author").Find(&posts)
```

---

## 11. 연결 끊김과 재연결

### 문제 상황

장시간 운영 후 갑자기 쿼리 실패:

```text
pq: unexpected EOF
pq: connection reset by peer
driver: bad connection
```

### 원인 분석

- PostgreSQL의 `idle_in_transaction_session_timeout`
- 네트워크 타임아웃 (NAT, 로드밸런서)
- PostgreSQL 재시작

### 해결 방법

```go
sqlDB, _ := db.DB()

// 연결 최대 수명 (PostgreSQL idle timeout보다 짧게)
sqlDB.SetConnMaxLifetime(30 * time.Minute)

// 유휴 연결 최대 시간
sqlDB.SetConnMaxIdleTime(10 * time.Minute)

// 연결 풀 설정
sqlDB.SetMaxIdleConns(10)
sqlDB.SetMaxOpenConns(100)
```

**Kubernetes 환경:**

```yaml
# PostgreSQL StatefulSet
env:
  - name: POSTGRES_ARGS
    value: "-c idle_in_transaction_session_timeout=60000"  # 60초
```

---

## 12. FirstOrCreate 동시성 문제

### 문제 상황

```go
// 두 요청이 동시에 실행
db.FirstOrCreate(&user, User{Email: "test@example.com"})

// 결과: 둘 다 "없음"으로 판단 → 중복 생성 시도 → 하나는 실패
```

### 원인 분석

`FirstOrCreate`는 SELECT → INSERT 두 단계로 동작하며, 이 사이에 Race Condition이 발생한다.

### 해결 방법

**방법 1: Upsert 사용**

```go
db.Clauses(clause.OnConflict{
    Columns:   []clause.Column{{Name: "email"}},
    DoNothing: true,
}).Create(&user)

// 이후 조회
db.Where("email = ?", user.Email).First(&user)
```

**방법 2: DB 레벨 Unique 제약 + 에러 핸들링**

```go
err := db.Create(&user).Error
if err != nil {
    if strings.Contains(err.Error(), "duplicate key") {
        db.Where("email = ?", user.Email).First(&user)
        return nil
    }
    return err
}
```

---

## 13. Find vs First vs Take 차이

### 문제 상황

```go
var user User

db.Find(&user, id)   // user가 비어있어도 에러 없음
db.First(&user, id)  // 없으면 ErrRecordNotFound
db.Take(&user, id)   // 없으면 ErrRecordNotFound
```

### 차이점

| 메서드 | 결과 없을 때 | 정렬 | 용도 |
|--------|--------------|------|------|
| `Find` | 에러 없음, 빈 값 | 없음 | 다중 조회, 없어도 OK |
| `First` | `ErrRecordNotFound` | PK ASC | 단일 조회, 존재 필수 |
| `Take` | `ErrRecordNotFound` | 없음 | 아무거나 하나 |

### 올바른 사용

```go
// 존재 여부가 중요한 경우
var user User
err := db.First(&user, id).Error
if errors.Is(err, gorm.ErrRecordNotFound) {
    return nil, repository.ErrNotFound
}

// 없어도 괜찮은 경우
var users []User
db.Where("org_id = ?", orgID).Find(&users)  // 빈 슬라이스 반환
```

---

## 14. Raw SQL과 Scan 타입 불일치

### 문제 상황

```go
type Result struct {
    Count int64
    Total float64
}

var result Result
db.Raw("SELECT COUNT(*) as count, SUM(amount) as total FROM orders").Scan(&result)

// result.Total = 0 (기대: 실제 합계)
```

### 원인 분석

PostgreSQL의 `SUM()`은 `numeric` 타입을 반환하는데, Go의 `float64`와 매핑이 안 될 수 있다.

### 해결 방법

```go
// 방법 1: 타입 캐스팅
db.Raw("SELECT COUNT(*)::bigint as count, SUM(amount)::float as total FROM orders").Scan(&result)

// 방법 2: sql.NullFloat64 사용
type Result struct {
    Count int64
    Total sql.NullFloat64
}
```

---

## 15. 대량 Insert 시 메모리 문제

### 문제 상황

```go
// 100만 건 Insert
var users []User  // 메모리에 100만 개 로드
db.Create(&users)  // OOM 또는 타임아웃
```

### 해결 방법

```go
// CreateInBatches 사용
db.CreateInBatches(users, 1000)  // 1000개씩 나눠서 Insert

// 또는 스트리밍 처리
batchSize := 1000
for i := 0; i < len(users); i += batchSize {
    end := i + batchSize
    if end > len(users) {
        end = len(users)
    }
    if err := db.Create(users[i:end]).Error; err != nil {
        return err
    }
}
```

---

## 체크리스트

새 프로젝트에서 GORM 설정 시 확인 사항:

- [ ] DSN에 `TimeZone` 설정 (UTC 또는 Asia/Seoul 통일)
- [ ] DSN에 `search_path` 설정
- [ ] 연결 풀 설정 (MaxOpenConns, MaxIdleConns, ConnMaxLifetime)
- [ ] Zero Value 업데이트는 Map 사용
- [ ] 관계 조회는 Preload 사용
- [ ] 스키마 관리 책임 단일화 (App 또는 DB)
- [ ] 프로덕션에서는 AutoMigrate 대신 마이그레이션 도구 사용
- [ ] Context 전파 (`WithContext`)
- [ ] 에러 타입 분류 (`ErrRecordNotFound` 체크)
- [ ] FirstOrCreate 대신 Upsert 패턴 고려
- [ ] Docker 환경에서 TZ 환경변수 설정

## 참고 자료

### 공식 문서
- [GORM 공식 문서](https://gorm.io/docs/)
- [PostgreSQL DSN 문서](https://www.postgresql.org/docs/current/libpq-connect.html)

### 관련 글
- [GORM 소개: Go 개발자를 위한 ORM 완벽 가이드](https://blog.imprun.dev/94)
- [GORM 기반 엔터프라이즈 Go API Server 아키텍처](https://blog.imprun.dev/95)
