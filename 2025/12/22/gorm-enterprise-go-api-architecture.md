# GORM 기반 엔터프라이즈 Go API Server 아키텍처

> **작성일**: 2025년 12월 22일
> **카테고리**: Backend, Go, Architecture
> **키워드**: Go, GORM, Clean Architecture, Repository Pattern, PostgreSQL

## 요약

엔터프라이즈급 Go API Server를 구축할 때 GORM을 어떻게 활용해야 하는지 실제 프로젝트 사례를 통해 설명한다. Clean Architecture 기반의 계층 분리, Repository 패턴 구현, Custom Type을 활용한 JSONB 처리, 그리고 멀티테넌시와 감사 로그 설계까지 프로덕션 환경에서 검증된 패턴을 다룬다.

## 프로젝트 구조

Clean Architecture를 적용한 Go API Server의 표준 구조다.

```
api/
├── cmd/server/              # 애플리케이션 진입점
│   └── main.go
├── internal/
│   ├── domain/              # 도메인 계층
│   │   ├── entity/          # GORM 엔티티 정의
│   │   ├── repository/      # Repository 인터페이스
│   │   └── service/         # 도메인 서비스
│   ├── infrastructure/      # 인프라 계층
│   │   ├── persistence/     # Repository 구현, DB 연결
│   │   └── client/          # 외부 서비스 클라이언트
│   ├── interface/           # 프레젠테이션 계층
│   │   └── api/v1/          # HTTP 핸들러
│   └── usecase/             # 비즈니스 로직 오케스트레이션
├── pkg/
│   ├── config/              # 설정 관리
│   └── logger/              # 로깅
└── go.mod
```

### 계층별 책임

| 계층 | 위치 | 책임 |
|------|------|------|
| Domain | `internal/domain` | 엔티티 정의, Repository 인터페이스, 비즈니스 규칙 |
| Infrastructure | `internal/infrastructure` | Repository 구현, DB 연결, 외부 시스템 통합 |
| Interface | `internal/interface` | HTTP 핸들러, 요청/응답 변환 |
| UseCase | `internal/usecase` | 비즈니스 로직 조합, 트랜잭션 경계 |

### 의존성 방향

```
Interface → UseCase → Domain ← Infrastructure
```

Domain 계층은 다른 계층에 의존하지 않으며, Infrastructure가 Domain의 인터페이스를 구현한다.

## 베이스 모델 설계

### UUID 기반 Primary Key

분산 시스템에서는 Auto Increment 대신 UUID를 사용한다.

```go
// internal/domain/entity/base.go
package entity

import (
    "time"

    "github.com/google/uuid"
    "gorm.io/gorm"
)

// gen_random_uuid()는 PostgreSQL 13+ 기본 제공, 이전 버전은 pgcrypto 확장 필요
type BaseModel struct {
    ID        uuid.UUID      `gorm:"type:uuid;default:gen_random_uuid();primaryKey"`
    CreatedAt time.Time      `gorm:"autoCreateTime"`
    UpdatedAt time.Time      `gorm:"autoUpdateTime"`
    DeletedAt gorm.DeletedAt `gorm:"index"`
}
```

**UUID 선택 이유:**

- 분산 환경에서 ID 충돌 없이 생성 가능
- 데이터베이스 간 마이그레이션 용이
- URL에 노출되어도 순서 추론 불가 (보안)

### 타임스탬프 자동화

GORM의 `autoCreateTime`, `autoUpdateTime` 태그를 사용하면 별도 코드 없이 시간이 자동 기록된다.

## Repository 패턴 구현

### 인터페이스 정의

Repository 인터페이스는 Domain 계층에 정의한다.

```go
// internal/domain/repository/user_repository.go
package repository

import (
    "context"

    "github.com/google/uuid"
    "myapp/internal/domain/entity"
)

var ErrNotFound = errors.New("record not found")

type UserRepository interface {
    Create(ctx context.Context, user *entity.User) error
    FindByID(ctx context.Context, id uuid.UUID) (*entity.User, error)
    FindByEmail(ctx context.Context, email string) (*entity.User, error)
    FindAll(ctx context.Context, search string, offset, limit int) ([]*entity.User, int64, error)
    Update(ctx context.Context, user *entity.User) error
    Delete(ctx context.Context, id uuid.UUID) error
    Count(ctx context.Context) (int64, error)
}
```

**설계 원칙:**

- 모든 메서드는 `context.Context`를 첫 번째 인자로 받음
- 표준 에러 정의 (`ErrNotFound`)
- 페이지네이션은 `offset`, `limit` 파라미터와 `total` 반환

### Repository 구현

Infrastructure 계층에서 인터페이스를 구현한다.

```go
// internal/infrastructure/persistence/user_repository.go
package persistence

import (
    "context"
    "errors"

    "github.com/google/uuid"
    "gorm.io/gorm"
    "myapp/internal/domain/entity"
    "myapp/internal/domain/repository"
)

type userRepository struct {
    db *gorm.DB
}

func NewUserRepository(db *Database) repository.UserRepository {
    return &userRepository{db: db.DB}
}

func (r *userRepository) Create(ctx context.Context, user *entity.User) error {
    return r.db.WithContext(ctx).Create(user).Error
}

func (r *userRepository) FindByID(ctx context.Context, id uuid.UUID) (*entity.User, error) {
    var user entity.User
    err := r.db.WithContext(ctx).
        Preload("Organizations", func(db *gorm.DB) *gorm.DB {
            return db.Order("is_primary DESC, created_at ASC")
        }).
        Preload("Organizations.Organization").
        Where("id = ?", id).
        First(&user).Error

    if err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, repository.ErrNotFound
        }
        return nil, err
    }
    return &user, nil
}

func (r *userRepository) FindByEmail(ctx context.Context, email string) (*entity.User, error) {
    var user entity.User
    err := r.db.WithContext(ctx).
        Where("email = ?", email).
        First(&user).Error

    if err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, repository.ErrNotFound
        }
        return nil, err
    }
    return &user, nil
}

func (r *userRepository) FindAll(ctx context.Context, search string, offset, limit int) ([]*entity.User, int64, error) {
    var users []*entity.User
    var total int64

    query := r.db.WithContext(ctx).Model(&entity.User{})

    // PostgreSQL ILIKE: 대소문자 무관 검색
    if search != "" {
        pattern := "%" + search + "%"
        query = query.Where(
            "name ILIKE ? OR email ILIKE ?",
            pattern, pattern,
        )
    }

    if err := query.Count(&total).Error; err != nil {
        return nil, 0, err
    }

    if err := r.db.WithContext(ctx).
        Preload("Organizations", func(db *gorm.DB) *gorm.DB {
            return db.Order("is_primary DESC")
        }).
        Order("created_at DESC").
        Offset(offset).
        Limit(limit).
        Find(&users).Error; err != nil {
        return nil, 0, err
    }

    return users, total, nil
}

func (r *userRepository) Update(ctx context.Context, user *entity.User) error {
    return r.db.WithContext(ctx).Save(user).Error
}

func (r *userRepository) Delete(ctx context.Context, id uuid.UUID) error {
    return r.db.WithContext(ctx).Delete(&entity.User{}, id).Error
}

func (r *userRepository) Count(ctx context.Context) (int64, error) {
    var count int64
    err := r.db.WithContext(ctx).Model(&entity.User{}).Count(&count).Error
    return count, err
}
```

### 핵심 패턴

**1. Context 전파**

모든 쿼리에 `WithContext(ctx)`를 사용하여 요청 스코프의 취소와 타임아웃을 지원한다.

```go
db.WithContext(ctx).First(&user, id)
```

**2. 에러 매핑**

GORM의 에러를 도메인 에러로 변환한다.

```go
if errors.Is(err, gorm.ErrRecordNotFound) {
    return nil, repository.ErrNotFound
}
```

**3. Preload로 N+1 해결**

관계 데이터는 항상 Preload로 로드한다.

```go
db.Preload("Organizations").
   Preload("Organizations.Organization").
   First(&user, id)
```

## Custom Type 구현

PostgreSQL의 JSONB 컬럼을 Go 타입으로 매핑하려면 `driver.Valuer`와 `sql.Scanner` 인터페이스를 구현해야 한다.

### StringList 타입

```go
// internal/domain/entity/types.go
package entity

import (
    "database/sql/driver"
    "encoding/json"
)

type StringList []string

// DB에 저장할 때 호출
func (s StringList) Value() (driver.Value, error) {
    if s == nil {
        return "[]", nil
    }
    return json.Marshal(s)
}

// DB에서 읽을 때 호출
func (s *StringList) Scan(value interface{}) error {
    if value == nil {
        *s = StringList{}
        return nil
    }

    var bytes []byte
    switch v := value.(type) {
    case []byte:
        bytes = v
    case string:
        bytes = []byte(v)
    default:
        return errors.New("unsupported type for StringList")
    }

    return json.Unmarshal(bytes, s)
}
```

### JSONMap 타입

```go
type JSONMap map[string]interface{}

func (j JSONMap) Value() (driver.Value, error) {
    if j == nil {
        return "{}", nil
    }
    return json.Marshal(j)
}

func (j *JSONMap) Scan(value interface{}) error {
    if value == nil {
        *j = JSONMap{}
        return nil
    }

    var bytes []byte
    switch v := value.(type) {
    case []byte:
        bytes = v
    case string:
        bytes = []byte(v)
    default:
        return errors.New("unsupported type for JSONMap")
    }

    return json.Unmarshal(bytes, j)
}
```

### 사용 예시

```go
type User struct {
    BaseModel
    Email       string     `gorm:"type:varchar(255);uniqueIndex;not null"`
    LegalsAgreed StringList `gorm:"type:jsonb;default:'[]'"`
    Preferences  JSONMap    `gorm:"type:jsonb;default:'{}'"`
}
```

## 트랜잭션 패턴

### 상태 전이가 포함된 트랜잭션

```go
// internal/infrastructure/persistence/snapshot_repository.go

func (r *snapshotRepository) MarkDeployed(ctx context.Context, id uuid.UUID) error {
    // 1. 현재 스냅샷 조회
    var snapshot entity.Snapshot
    if err := r.db.WithContext(ctx).Where("id = ?", id).First(&snapshot).Error; err != nil {
        return err
    }

    // 2. 트랜잭션 내에서 상태 전이
    return r.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
        // 이전 Deployed 스냅샷을 Superseded로 변경
        if err := tx.Model(&entity.Snapshot{}).
            Where("environment_id = ? AND status = ? AND id != ?",
                snapshot.EnvironmentID, entity.SnapshotStatusDeployed, id).
            Update("status", entity.SnapshotStatusSuperseded).Error; err != nil {
            return err
        }

        // 현재 스냅샷을 Deployed로 변경
        if err := tx.Model(&entity.Snapshot{}).
            Where("id = ?", id).
            Updates(map[string]interface{}{
                "status":      entity.SnapshotStatusDeployed,
                "deployed_at": time.Now(),
            }).Error; err != nil {
            return err
        }

        return nil // 커밋
    })
}
```

### 부분 업데이트

Zero Value 문제를 피하기 위해 Map을 사용한다.

```go
func (r *snapshotRepository) UpdateStatus(ctx context.Context, id uuid.UUID, status entity.SnapshotStatus) error {
    updates := map[string]interface{}{
        "status": status,
    }
    if status == entity.SnapshotStatusDeployed {
        updates["deployed_at"] = time.Now()
    }
    return r.db.WithContext(ctx).
        Model(&entity.Snapshot{}).
        Where("id = ?", id).
        Updates(updates).Error
}
```

## 멀티테넌시 구현

### Organization 기반 데이터 격리

```go
type Organization struct {
    BaseModel
    Name string `gorm:"type:varchar(100);not null"`
    Slug string `gorm:"type:varchar(50);uniqueIndex;not null"`
}

type Gateway struct {
    BaseModel
    AppID  string     `gorm:"type:varchar(20);uniqueIndex;not null"`
    OrgID  *uuid.UUID `gorm:"type:uuid;index"` // nullable: 시스템 리소스
    // ...
}
```

### 테넌트 필터링

```go
func (r *gatewayRepository) FindByOrganization(ctx context.Context, orgID uuid.UUID, offset, limit int) ([]*entity.Gateway, int64, error) {
    var gateways []*entity.Gateway
    var total int64

    query := r.db.WithContext(ctx).
        Model(&entity.Gateway{}).
        Where("org_id = ?", orgID)

    if err := query.Count(&total).Error; err != nil {
        return nil, 0, err
    }

    if err := r.db.WithContext(ctx).
        Where("org_id = ?", orgID).
        Order("created_at DESC").
        Offset(offset).
        Limit(limit).
        Find(&gateways).Error; err != nil {
        return nil, 0, err
    }

    return gateways, total, nil
}
```

## 감사 로그 설계

### AuditLog 엔티티

```go
type AuditLog struct {
    BaseModel
    ActorID      uuid.UUID `gorm:"type:uuid;index;not null"`
    ActorType    string    `gorm:"type:varchar(50);not null"` // user, system, api_key
    Action       string    `gorm:"type:varchar(50);not null"` // create, update, delete
    ResourceType string    `gorm:"type:varchar(50);index;not null"`
    ResourceID   uuid.UUID `gorm:"type:uuid;index;not null"`
    Changes      JSONMap   `gorm:"type:jsonb"`
    IPAddress    string    `gorm:"type:varchar(45)"`
    UserAgent    string    `gorm:"type:varchar(255)"`
}
```

### 감사 로그 기록

```go
func (r *auditLogRepository) Log(ctx context.Context, log *entity.AuditLog) error {
    return r.db.WithContext(ctx).Create(log).Error
}

// UseCase에서 사용
func (uc *GatewayUseCase) Update(ctx context.Context, id uuid.UUID, input UpdateGatewayInput) error {
    gateway, err := uc.gatewayRepo.FindByID(ctx, id)
    if err != nil {
        return err
    }

    oldValues := map[string]interface{}{
        "display_name": gateway.DisplayName,
        "is_active":    gateway.IsActive,
    }

    gateway.DisplayName = input.DisplayName
    gateway.IsActive = input.IsActive

    if err := uc.gatewayRepo.Update(ctx, gateway); err != nil {
        return err
    }

    // 감사 로그 기록
    return uc.auditRepo.Log(ctx, &entity.AuditLog{
        ActorID:      input.ActorID,
        ActorType:    "user",
        Action:       "update",
        ResourceType: "gateway",
        ResourceID:   gateway.ID,
        Changes: entity.JSONMap{
            "old": oldValues,
            "new": map[string]interface{}{
                "display_name": input.DisplayName,
                "is_active":    input.IsActive,
            },
        },
    })
}
```

## 데이터베이스 연결 래퍼

```go
// internal/infrastructure/persistence/database.go
package persistence

import (
    "time"

    "gorm.io/driver/postgres"
    "gorm.io/gorm"
    "gorm.io/gorm/logger"
    "myapp/pkg/config"
)

type Database struct {
    *gorm.DB
}

func NewDatabase(cfg *config.DatabaseConfig) (*Database, error) {
    dsn := cfg.DSN()

    gormConfig := &gorm.Config{
        Logger: logger.Default.LogMode(logger.Info),
    }

    db, err := gorm.Open(postgres.Open(dsn), gormConfig)
    if err != nil {
        return nil, err
    }

    sqlDB, err := db.DB()
    if err != nil {
        return nil, err
    }

    sqlDB.SetMaxIdleConns(10)
    sqlDB.SetMaxOpenConns(100)
    sqlDB.SetConnMaxLifetime(time.Hour)

    return &Database{DB: db}, nil
}

func (d *Database) AutoMigrate(models ...interface{}) error {
    return d.DB.AutoMigrate(models...)
}

func (d *Database) Close() error {
    sqlDB, err := d.DB.DB()
    if err != nil {
        return err
    }
    return sqlDB.Close()
}
```

## 의존성 주입

```go
// cmd/server/main.go
func main() {
    cfg := config.Load()

    // 1. Database 연결
    db, err := persistence.NewDatabase(&cfg.Database)
    if err != nil {
        log.Fatal("Failed to connect database:", err)
    }
    defer db.Close()

    // 2. AutoMigrate
    if err := db.AutoMigrate(
        &entity.User{},
        &entity.Organization{},
        &entity.Gateway{},
        &entity.AuditLog{},
    ); err != nil {
        log.Fatal("Failed to migrate:", err)
    }

    // 3. Repository 초기화
    userRepo := persistence.NewUserRepository(db)
    orgRepo := persistence.NewOrganizationRepository(db)
    gatewayRepo := persistence.NewGatewayRepository(db)
    auditRepo := persistence.NewAuditLogRepository(db)

    // 4. UseCase 초기화
    gatewayUseCase := usecase.NewGatewayUseCase(gatewayRepo, auditRepo)

    // 5. Handler 초기화
    deps := &api.RouterDeps{
        UserRepo:       userRepo,
        OrgRepo:        orgRepo,
        GatewayUseCase: gatewayUseCase,
    }

    router := api.NewRouter(deps)
    router.Run(":8080")
}
```

## 마이그레이션 전략

### 개발 환경

AutoMigrate 사용:

```go
if cfg.Environment == "development" {
    db.AutoMigrate(&entity.User{}, &entity.Gateway{})
}
```

### 프로덕션 환경

전용 마이그레이션 도구 사용 권장:

```bash
# golang-migrate 사용 예시
migrate -path ./migrations -database "postgres://..." up
```

마이그레이션 파일 구조:

```
migrations/
├── 000001_create_users.up.sql
├── 000001_create_users.down.sql
├── 000002_create_gateways.up.sql
└── 000002_create_gateways.down.sql
```

## 참고 자료

### 공식 문서
- [GORM 공식 문서](https://gorm.io/docs/)
- [Clean Architecture by Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)

### 관련 글
- [GORM 소개: Go 개발자를 위한 ORM 완벽 가이드](https://blog.imprun.dev/94)
- [GORM 실무 트러블슈팅: 운영 환경에서 만난 함정들](https://blog.imprun.dev/96)
