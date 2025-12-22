# GORM 소개: Go 개발자를 위한 ORM 완벽 가이드

> **작성일**: 2025년 12월 22일
> **카테고리**: Backend, Go
> **키워드**: Go, GORM, ORM, PostgreSQL, Database

## 요약

GORM은 Go 생태계에서 가장 널리 사용되는 ORM(Object-Relational Mapping) 라이브러리다. 이 글에서는 GORM의 핵심 개념, 주요 기능, 그리고 실제 프로젝트에서의 활용 방법을 다룬다. 데이터베이스 연결부터 모델 정의, CRUD 작업, 관계 설정까지 실무에서 필요한 내용을 단계별로 설명한다.

## GORM이란

GORM(Go Object Relational Mapper)은 Go 언어를 위한 풀 피처 ORM 라이브러리다. 개발자가 SQL을 직접 작성하지 않고도 Go 구조체를 통해 데이터베이스를 조작할 수 있게 해준다.

### 주요 특징

- **Auto Migration**: 구조체 정의로부터 테이블 스키마 자동 생성 및 변경
- **연관 관계**: Has One, Has Many, Belongs To, Many To Many 지원
- **Hook**: BeforeCreate, AfterUpdate 등 생명주기 콜백
- **트랜잭션**: 중첩 트랜잭션, Save Point 지원
- **Soft Delete**: 실제 삭제 대신 삭제 표시
- **Batch Insert/Update**: 대량 데이터 처리 최적화
- **Prepared Statement 캐싱**: 성능 최적화

### 지원 데이터베이스

- PostgreSQL
- MySQL / MariaDB
- SQLite
- SQL Server
- ClickHouse

## 설치 및 기본 설정

### 패키지 설치

```bash
# GORM 코어
go get -u gorm.io/gorm

# PostgreSQL 드라이버
go get -u gorm.io/driver/postgres

# MySQL 드라이버
go get -u gorm.io/driver/mysql

# SQLite 드라이버
go get -u gorm.io/driver/sqlite
```

### 데이터베이스 연결

```go
package main

import (
    "gorm.io/driver/postgres"
    "gorm.io/gorm"
    "log"
)

func main() {
    dsn := "host=localhost user=postgres password=secret dbname=myapp port=5432 sslmode=disable TimeZone=Asia/Seoul"

    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        log.Fatal("Failed to connect database:", err)
    }

    log.Println("Database connected successfully")
}
```

### DSN 옵션

PostgreSQL DSN에서 자주 사용하는 옵션:

| 옵션 | 설명 | 예시 |
|------|------|------|
| `host` | 데이터베이스 서버 주소 | `localhost` |
| `port` | 포트 번호 | `5432` |
| `user` | 접속 사용자 | `postgres` |
| `password` | 비밀번호 | `secret` |
| `dbname` | 데이터베이스 이름 | `myapp` |
| `sslmode` | SSL 연결 모드 | `disable`, `require` |
| `TimeZone` | 타임존 설정 | `Asia/Seoul`, `UTC` |
| `search_path` | 스키마 검색 경로 | `public` |
| `connect_timeout` | 연결 타임아웃(초) | `10` |

```go
// search_path가 중요한 이유: AutoMigrate 시 테이블 존재 여부 확인에 영향
dsn := "host=localhost user=postgres password=secret dbname=myapp port=5432 sslmode=disable search_path=public"
```

### 연결 풀 설정

프로덕션 환경에서는 연결 풀 설정이 필수다.

```go
db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
if err != nil {
    log.Fatal(err)
}

sqlDB, err := db.DB()
if err != nil {
    log.Fatal(err)
}

// 최대 유휴 연결 수
sqlDB.SetMaxIdleConns(10)

// 최대 열린 연결 수
sqlDB.SetMaxOpenConns(100)

// 연결 최대 수명
sqlDB.SetConnMaxLifetime(time.Hour)
```

## 모델 정의

### 기본 모델

GORM은 `gorm.Model`을 내장 제공하지만, 실무에서는 커스텀 베이스 모델을 정의하는 것이 일반적이다.

```go
// GORM 기본 제공 모델
type Model struct {
    ID        uint           `gorm:"primaryKey"`
    CreatedAt time.Time
    UpdatedAt time.Time
    DeletedAt gorm.DeletedAt `gorm:"index"`
}

// 실무에서 많이 사용하는 UUID 기반 베이스 모델
// gen_random_uuid()는 PostgreSQL 13+ 기본 제공, 이전 버전은 pgcrypto 확장 필요
type BaseModel struct {
    ID        uuid.UUID      `gorm:"type:uuid;default:gen_random_uuid();primaryKey"`
    CreatedAt time.Time      `gorm:"autoCreateTime"`
    UpdatedAt time.Time      `gorm:"autoUpdateTime"`
    DeletedAt gorm.DeletedAt `gorm:"index"`
}
```

### 엔티티 정의

```go
type User struct {
    BaseModel
    Email    string `gorm:"type:varchar(255);uniqueIndex;not null"`
    Name     string `gorm:"type:varchar(100);not null"`
    Password string `gorm:"type:varchar(255);not null"`
    IsActive bool   `gorm:"default:true"`

    // 관계
    Posts []Post `gorm:"foreignKey:UserID"`
}

type Post struct {
    BaseModel
    Title   string    `gorm:"type:varchar(255);not null"`
    Content string    `gorm:"type:text"`
    UserID  uuid.UUID `gorm:"type:uuid;not null;index"`

    // 관계
    User User `gorm:"foreignKey:UserID"`
}
```

### 주요 태그

| 태그 | 설명 | 예시 |
|------|------|------|
| `type` | 컬럼 타입 지정 | `gorm:"type:varchar(255)"` |
| `primaryKey` | Primary Key 지정 | `gorm:"primaryKey"` |
| `uniqueIndex` | Unique 인덱스 | `gorm:"uniqueIndex"` |
| `index` | 일반 인덱스 | `gorm:"index"` |
| `not null` | NOT NULL 제약 | `gorm:"not null"` |
| `default` | 기본값 설정 | `gorm:"default:true"` |
| `foreignKey` | 외래키 지정 | `gorm:"foreignKey:UserID"` |
| `autoCreateTime` | 생성 시간 자동 기록 | `gorm:"autoCreateTime"` |
| `autoUpdateTime` | 수정 시간 자동 기록 | `gorm:"autoUpdateTime"` |
| `column` | 컬럼명 지정 | `gorm:"column:user_name"` |

### 테이블명 커스터마이징

```go
// 기본: 구조체명의 복수형 snake_case (User → users)

// 명시적 테이블명 지정
func (User) TableName() string {
    return "app_users"
}

// 스키마 포함 지정 (멀티 스키마 환경)
func (User) TableName() string {
    return "public.users"
}
```

## Auto Migration

### 기본 사용법

```go
// 단일 모델 마이그레이션
db.AutoMigrate(&User{})

// 다중 모델 마이그레이션
db.AutoMigrate(
    &User{},
    &Post{},
    &Comment{},
)
```

### AutoMigrate의 동작 방식

AutoMigrate는 다음 작업을 수행한다:

1. 테이블이 없으면 생성
2. 누락된 컬럼 추가
3. 누락된 인덱스 추가

AutoMigrate는 다음 작업을 수행하지 **않는다**:

- 기존 컬럼 삭제
- 기존 컬럼 타입 변경
- 기존 인덱스 삭제

```go
// 에러 처리
if err := db.AutoMigrate(&User{}, &Post{}); err != nil {
    log.Fatal("Migration failed:", err)
}
```

### 프로덕션 환경에서의 마이그레이션

프로덕션에서는 AutoMigrate보다 전용 마이그레이션 도구 사용을 권장한다.

- [golang-migrate/migrate](https://github.com/golang-migrate/migrate)
- [Atlas](https://atlasgo.io/)
- [goose](https://github.com/pressly/goose)

## CRUD 작업

### Create

```go
// 단일 레코드 생성
user := User{
    Email: "user@example.com",
    Name:  "John Doe",
}
result := db.Create(&user)
// result.Error: 에러 정보
// result.RowsAffected: 영향받은 행 수
// user.ID: 생성된 레코드의 ID (자동 할당)

// 특정 필드만 생성
db.Select("Email", "Name").Create(&user)

// 배치 생성 (성능 최적화)
users := []User{
    {Email: "user1@example.com", Name: "User 1"},
    {Email: "user2@example.com", Name: "User 2"},
}
db.CreateInBatches(users, 100) // 100개씩 배치 처리
```

### Read

```go
// 단일 조회 (Primary Key)
var user User
db.First(&user, id) // uuid.UUID or uint

// 조건 조회
db.First(&user, "email = ?", "user@example.com")

// Where 절
db.Where("name = ?", "John").First(&user)

// 다중 조회
var users []User
db.Find(&users)

// 조건부 다중 조회
db.Where("is_active = ?", true).Find(&users)

// LIKE 검색 (PostgreSQL ILIKE: 대소문자 무관)
db.Where("name ILIKE ?", "%john%").Find(&users)

// 페이지네이션
db.Offset(0).Limit(10).Find(&users)

// 정렬
db.Order("created_at DESC").Find(&users)

// 카운트
var count int64
db.Model(&User{}).Count(&count)

// 선택 필드
db.Select("id", "name", "email").Find(&users)
```

### Update

```go
// 단일 필드 업데이트
db.Model(&user).Update("name", "New Name")

// 다중 필드 업데이트 (struct)
db.Model(&user).Updates(User{Name: "New Name", IsActive: false})

// 다중 필드 업데이트 (map) - Zero Value 문제 해결
db.Model(&user).Updates(map[string]interface{}{
    "name":      "New Name",
    "is_active": false,
})

// 조건부 업데이트
db.Model(&User{}).Where("is_active = ?", true).Update("is_active", false)

// Save: 전체 필드 저장
user.Name = "Updated Name"
db.Save(&user)
```

### Delete

```go
// Soft Delete (DeletedAt 필드가 있는 경우)
db.Delete(&user)

// 조건부 삭제
db.Where("name = ?", "John").Delete(&User{})

// Hard Delete (영구 삭제)
db.Unscoped().Delete(&user)

// Soft Delete된 레코드 포함 조회
db.Unscoped().Find(&users)
```

## 관계 설정

### Belongs To (N:1)

```go
type Post struct {
    ID     uuid.UUID
    Title  string
    UserID uuid.UUID
    User   User `gorm:"foreignKey:UserID"`
}
```

### Has One (1:1)

```go
type User struct {
    ID      uuid.UUID
    Profile Profile `gorm:"foreignKey:UserID"`
}

type Profile struct {
    ID     uuid.UUID
    UserID uuid.UUID
    Bio    string
}
```

### Has Many (1:N)

```go
type User struct {
    ID    uuid.UUID
    Posts []Post `gorm:"foreignKey:UserID"`
}
```

### Many To Many (M:N)

```go
type User struct {
    ID    uuid.UUID
    Teams []Team `gorm:"many2many:user_teams;"`
}

type Team struct {
    ID    uuid.UUID
    Users []User `gorm:"many2many:user_teams;"`
}
```

### Preload (N+1 문제 해결)

```go
// 기본 Preload
var users []User
db.Preload("Posts").Find(&users)

// 중첩 Preload
db.Preload("Posts.Comments").Find(&users)

// 조건부 Preload
db.Preload("Posts", func(db *gorm.DB) *gorm.DB {
    return db.Where("is_published = ?", true).Order("created_at DESC")
}).Find(&users)

// 다중 Preload
db.Preload("Posts").Preload("Teams").Find(&users)
```

## 트랜잭션

### 기본 트랜잭션

```go
// Transaction 메서드 사용 (권장)
err := db.Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&user).Error; err != nil {
        return err // 롤백
    }

    if err := tx.Create(&post).Error; err != nil {
        return err // 롤백
    }

    return nil // 커밋
})
```

### 수동 트랜잭션

```go
tx := db.Begin()

if err := tx.Create(&user).Error; err != nil {
    tx.Rollback()
    return err
}

if err := tx.Create(&post).Error; err != nil {
    tx.Rollback()
    return err
}

tx.Commit()
```

### 중첩 트랜잭션

```go
db.Transaction(func(tx *gorm.DB) error {
    tx.Create(&user1)

    tx.Transaction(func(tx2 *gorm.DB) error {
        tx2.Create(&user2)
        return nil
    })

    return nil
})
```

## Context 활용

모든 쿼리에 Context를 전달하여 타임아웃과 취소를 지원한다.

```go
// Context가 있는 쿼리
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

var user User
db.WithContext(ctx).First(&user, id)
```

## 로깅 설정

```go
import "gorm.io/gorm/logger"

db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{
    Logger: logger.Default.LogMode(logger.Info),
})

// 로그 레벨
// logger.Silent - 로그 없음
// logger.Error  - 에러만
// logger.Warn   - 경고 이상
// logger.Info   - 모든 쿼리 (개발용)
```

## 다음 단계

이 글에서는 GORM의 기본 개념과 사용법을 다루었다. 다음 글에서는 엔터프라이즈급 Go API Server에서 GORM을 활용한 아키텍처 설계 패턴을 다룬다:

- Clean Architecture와 Repository 패턴
- Custom Type 구현 (JSONB 활용)
- 멀티테넌시 구현
- 감사 로그 설계

## 참고 자료

### 공식 문서
- [GORM 공식 문서](https://gorm.io/docs/)
- [GORM GitHub](https://github.com/go-gorm/gorm)

### 관련 글
- [GORM 기반 엔터프라이즈 Go API Server 아키텍처](https://blog.imprun.dev/95)
- [GORM 실무 트러블슈팅: 운영 환경에서 만난 함정들](https://blog.imprun.dev/96)
