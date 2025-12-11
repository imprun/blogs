# Ory Keto를 활용한 ReBAC 기반 권한 관리 시스템 구축

> **작성일**: 2025년 12월 6일
> **카테고리**: Authorization, Identity, Backend
> **키워드**: Ory Keto, ReBAC, Google Zanzibar, Authorization, Permission, OPL

## 요약

Ory Keto는 Google Zanzibar 논문을 기반으로 구현된 오픈소스 권한 관리 시스템이다. 전통적인 RBAC(Role-Based Access Control)의 한계를 넘어, Relationship-Based Access Control(ReBAC)을 통해 복잡한 권한 모델을 효율적으로 처리한다. 이 글에서는 Keto의 핵심 개념과 실제 프로젝트에서의 적용 방법을 다룬다.

## Google Zanzibar와 ReBAC

### Zanzibar의 배경

Google Zanzibar는 Google 내부에서 Gmail, Drive, YouTube 등 수십억 사용자의 권한을 관리하기 위해 개발된 시스템이다. 2019년 발표된 논문에 따르면, 95번째 백분위 지연 시간이 10ms 미만이며 99.99% 이상의 가용성을 유지한다.

### ReBAC란

ReBAC(Relationship-Based Access Control)는 "관계"를 중심으로 권한을 정의하는 모델이다.

```mermaid
graph LR
    subgraph RBAC["전통적인 RBAC"]
        U1[User] --> R1[Role]
        R1 --> P1[Permission]
    end

    subgraph ReBAC_Model["ReBAC"]
        S1[Subject] <-->|relation| O1[Object]
    end

    style RBAC stroke:#6b7280,stroke-width:2px
    style ReBAC_Model stroke:#16a34a,stroke-width:2px
```

RBAC에서는 "관리자는 문서를 편집할 수 있다"와 같이 역할 단위로 권한을 부여한다. ReBAC에서는 "Alice는 Document:readme.txt의 editor이다"와 같이 개별 객체 단위로 관계를 정의한다.

### ReBAC의 장점

| 항목 | RBAC | ReBAC |
|------|------|-------|
| 권한 단위 | 역할(Role) | 관계(Relation) |
| 세분화 | 리소스 유형 단위 | 개별 리소스 단위 |
| 상속 | 역할 계층 | 관계 그래프 |
| 확장성 | 역할 폭발 문제 발생 | 관계 추가로 해결 |

## Ory Keto 핵심 개념

### Relation Tuple

Keto의 데이터 모델은 Relation Tuple이라는 단위로 구성된다.

```
<namespace>:<object>#<relation>@<subject>
```

예시:
```
Organization:acme#owners@User:alice
Document:readme#viewers@Organization:acme#members
```

첫 번째 예시는 "alice가 acme 조직의 owner"임을 의미한다. 두 번째 예시는 "acme 조직의 members가 readme 문서의 viewers"임을 나타내며, Subject Set을 통한 권한 상속을 보여준다.

### 네 가지 구성 요소

1. **Namespace**: 객체의 타입 (예: Organization, Document, User)
2. **Object**: 권한 대상이 되는 리소스의 ID
3. **Relation**: 주체와 객체 간의 관계 (예: owners, members, viewers)
4. **Subject**: 권한을 가진 주체 (User 또는 Subject Set)

### Subject Set

Subject Set은 다른 관계의 모든 주체를 참조하는 방식이다.

```typescript
// Organization의 members가 모든 하위 Document를 볼 수 있음
Document:doc1#viewers@Organization:org1#members
```

이를 통해 그룹 기반 권한 상속이 가능하다.

## OPL(Ory Permission Language)

### OPL 개요

OPL은 TypeScript의 부분집합으로 설계된 권한 정의 언어다. TypeScript 문법을 사용하므로 별도의 DSL 학습이 불필요하다.

### 기본 구조

```typescript
import { Namespace, Context } from "@ory/permission-language"

class User implements Namespace {}

class Organization implements Namespace {
  related: {
    owners: User[]
    admins: User[]
    members: User[]
  }

  permits = {
    manage: (ctx: Context): boolean =>
      this.related.owners.includes(ctx.subject) ||
      this.related.admins.includes(ctx.subject),

    view: (ctx: Context): boolean =>
      this.related.owners.includes(ctx.subject) ||
      this.related.admins.includes(ctx.subject) ||
      this.related.members.includes(ctx.subject),
  }
}
```

### 권한 상속

```typescript
class Document implements Namespace {
  related: {
    organization: Organization[]
    editors: User[]
    viewers: User[]
  }

  permits = {
    edit: (ctx: Context): boolean =>
      this.related.editors.includes(ctx.subject) ||
      this.related.organization.traverse((org) => org.permits.manage(ctx)),

    view: (ctx: Context): boolean =>
      this.permits.edit(ctx) ||
      this.related.viewers.includes(ctx.subject) ||
      this.related.organization.traverse((org) => org.permits.view(ctx)),
  }
}
```

`traverse` 함수를 통해 상위 객체의 권한을 확인할 수 있다.

## Docker Compose 설정

### keto.yml

```yaml
version: v0.14.0

log:
  level: debug
  format: json

namespaces:
  location: file:///etc/config/keto/namespaces.keto.ts

serve:
  read:
    host: 0.0.0.0
    port: 4466
  write:
    host: 0.0.0.0
    port: 4467
  metrics:
    host: 0.0.0.0
    port: 4468
```

### docker-compose.yml

```yaml
services:
  keto:
    image: oryd/keto:v0.14.0
    ports:
      - "4466:4466"  # Read API
      - "4467:4467"  # Write API
    volumes:
      - ./keto:/etc/config/keto
    environment:
      - DSN=postgres://keto:secret@postgres:5432/keto?sslmode=disable
    command: serve -c /etc/config/keto/keto.yml
```

Keto는 Read API(4466)와 Write API(4467)를 분리하여 제공한다. 프로덕션 환경에서는 Write API에 대한 접근을 내부 네트워크로 제한해야 한다.

## API 사용법

### Relation Tuple 생성

```bash
curl -X PUT "http://localhost:4467/admin/relation-tuples" \
  -H "Content-Type: application/json" \
  -d '{
    "namespace": "Organization",
    "object": "acme",
    "relation": "owners",
    "subject_id": "alice"
  }'
```

### 권한 체크

```bash
curl "http://localhost:4466/relation-tuples/check?namespace=Organization&object=acme&relation=owners&subject_id=alice"
```

응답:
```json
{
  "allowed": true
}
```

### Relation Tuple 조회

```bash
curl "http://localhost:4466/relation-tuples?namespace=Organization&object=acme"
```

### Relation Tuple 삭제

```bash
curl -X DELETE "http://localhost:4467/admin/relation-tuples?namespace=Organization&object=acme&relation=owners&subject_id=alice"
```

## Go 클라이언트 구현

### 클라이언트 구조

```go
type KetoClient struct {
    readURL    string
    writeURL   string
    httpClient *http.Client
}

func NewKetoClient(readURL, writeURL string) *KetoClient {
    return &KetoClient{
        readURL:  readURL,
        writeURL: writeURL,
        httpClient: &http.Client{
            Timeout: 10 * time.Second,
        },
    }
}
```

### Relation 생성

```go
func (c *KetoClient) CreateRelation(ctx context.Context, namespace, object, relation, subjectID string) error {
    tuple := map[string]string{
        "namespace":  namespace,
        "object":     object,
        "relation":   relation,
        "subject_id": subjectID,
    }

    body, _ := json.Marshal(tuple)
    req, _ := http.NewRequestWithContext(ctx, http.MethodPut,
        c.writeURL+"/admin/relation-tuples", bytes.NewReader(body))
    req.Header.Set("Content-Type", "application/json")

    resp, err := c.httpClient.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusCreated && resp.StatusCode != http.StatusOK {
        return fmt.Errorf("failed to create relation: %d", resp.StatusCode)
    }
    return nil
}
```

### 권한 체크

```go
func (c *KetoClient) Check(ctx context.Context, namespace, object, relation, subjectID string) (bool, error) {
    url := fmt.Sprintf("%s/relation-tuples/check?namespace=%s&object=%s&relation=%s&subject_id=%s",
        c.readURL,
        url.QueryEscape(namespace),
        url.QueryEscape(object),
        url.QueryEscape(relation),
        url.QueryEscape(subjectID),
    )

    req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    resp, err := c.httpClient.Do(req)
    if err != nil {
        return false, err
    }
    defer resp.Body.Close()

    if resp.StatusCode == http.StatusForbidden {
        return false, nil
    }

    var result struct {
        Allowed bool `json:"allowed"`
    }
    json.NewDecoder(resp.Body).Decode(&result)
    return result.Allowed, nil
}
```

## ImpRun 프로젝트 적용 사례

ImpRun은 API Gateway 관리 플랫폼으로, 복잡한 권한 모델을 Keto로 구현한다.

### Namespace 계층 구조

```mermaid
graph TB
    User["User<br/>(Kratos Identity)"]
    Org["Organization<br/>(기업/조직)"]
    GW["Gateway<br/>(API Gateway)"]
    Prod["Product<br/>(API 제품)"]
    Env["Environment<br/>(환경: dev/staging/prod)"]
    App["Application<br/>(소비자 앱)"]
    AR["AccessRequest<br/>(접근 요청)"]
    CI["CredentialIssuer<br/>(인증정보 발급자)"]
    Content["Content<br/>(포털 콘텐츠)"]

    User --> Org
    Org --> GW
    GW --> Prod
    GW --> CI
    GW --> Content
    Prod --> Env
    User --> App
    App --> AR
    AR --> Env

    style User stroke:#2563eb,stroke-width:2px
    style Org stroke:#16a34a,stroke-width:2px
    style GW stroke:#ea580c,stroke-width:2px
    style Prod stroke:#8b5cf6,stroke-width:2px
    style Env stroke:#ec4899,stroke-width:2px
    style App stroke:#14b8a6,stroke-width:2px
```

### Organization 권한 모델

```mermaid
graph LR
    subgraph Roles["Relations (역할)"]
        owners["owners"]
        admins["admins"]
        members["members"]
    end

    subgraph Permits["Permissions (권한)"]
        manage["manage<br/>(설정 변경, 구성원 관리)"]
        view["view<br/>(조직 조회)"]
        transfer["transfer<br/>(소유권 양도)"]
        delete["delete<br/>(조직 삭제)"]
    end

    owners --> manage & view & transfer & delete
    admins --> manage & view
    members --> view

    style Roles stroke:#2563eb,stroke-width:2px
    style Permits stroke:#16a34a,stroke-width:2px
```

```typescript
class Organization implements Namespace {
  related: {
    owners: User[]      // 조직 소유자
    admins: User[]      // 조직 관리자
    members: User[]     // 조직 구성원
  }

  permits = {
    manage: (ctx: Context): boolean =>
      this.related.owners.includes(ctx.subject) ||
      this.related.admins.includes(ctx.subject),

    view: (ctx: Context): boolean =>
      this.related.owners.includes(ctx.subject) ||
      this.related.admins.includes(ctx.subject) ||
      this.related.members.includes(ctx.subject),

    transfer: (ctx: Context): boolean =>
      this.related.owners.includes(ctx.subject),

    delete: (ctx: Context): boolean =>
      this.related.owners.includes(ctx.subject),
  }
}
```

### Gateway 권한 모델

Gateway는 ImpRun의 핵심 격리 단위로, 조직과 연동된 복잡한 권한 상속을 지원한다.

```mermaid
graph TB
    subgraph GW_Roles["Gateway Relations"]
        gw_owners["owners"]
        gw_admins["admins"]
        gw_members["members"]
        gw_viewers["viewers"]
    end

    subgraph GW_Permits["Gateway Permissions"]
        gw_manage["manage"]
        gw_view["view"]
        publish_config["publish_config"]
        manage_access["manage_access"]
        publish_content["publish_content"]
        manage_creds["manage_credentials"]
        gw_transfer["transfer"]
        gw_delete["delete"]
    end

    subgraph Org_Inherit["Organization 상속"]
        org_admins["Organization#admins"]
        org_members["Organization#members"]
    end

    org_admins -.->|"SubjectSet"| gw_owners
    org_admins -.->|"SubjectSet"| gw_admins
    org_members -.->|"SubjectSet"| gw_viewers

    gw_owners --> gw_manage & gw_view & publish_config & manage_access & publish_content & manage_creds & gw_transfer & gw_delete
    gw_admins --> gw_manage & gw_view & publish_config & manage_access & publish_content & manage_creds
    gw_members --> gw_view & publish_config & publish_content
    gw_viewers --> gw_view

    style GW_Roles stroke:#ea580c,stroke-width:2px
    style GW_Permits stroke:#8b5cf6,stroke-width:2px
    style Org_Inherit stroke:#16a34a,stroke-width:2px
```

```typescript
class Gateway implements Namespace {
  related: {
    // Role assignments - SubjectSet으로 Organization 상속
    owners: (User | SubjectSet<Organization, "admins">)[]
    admins: (User | SubjectSet<Organization, "admins">)[]
    members: User[]
    viewers: (User | SubjectSet<Organization, "members">)[]
    // Parent organization
    organization: Organization[]
  }

  permits = {
    manage: (ctx: Context): boolean =>
      this.related.owners.includes(ctx.subject) ||
      this.related.admins.includes(ctx.subject),

    view: (ctx: Context): boolean =>
      this.related.owners.includes(ctx.subject) ||
      this.related.admins.includes(ctx.subject) ||
      this.related.members.includes(ctx.subject) ||
      this.related.viewers.includes(ctx.subject),

    publish_config: (ctx: Context): boolean =>
      this.related.owners.includes(ctx.subject) ||
      this.related.admins.includes(ctx.subject) ||
      this.related.members.includes(ctx.subject),

    manage_access: (ctx: Context): boolean =>
      this.related.owners.includes(ctx.subject) ||
      this.related.admins.includes(ctx.subject),

    manage_credentials: (ctx: Context): boolean =>
      this.related.owners.includes(ctx.subject) ||
      this.related.admins.includes(ctx.subject),

    transfer: (ctx: Context): boolean =>
      this.related.owners.includes(ctx.subject),

    delete: (ctx: Context): boolean =>
      this.related.owners.includes(ctx.subject),
  }
}
```

### 권한 상속 패턴

Product와 Environment는 상위 Gateway의 권한을 `traverse`로 상속받는다.

```mermaid
sequenceDiagram
    participant User
    participant Product
    participant Gateway

    User->>Product: permits.manage()?
    Product->>Product: editors에 User 포함?
    alt 직접 권한 있음
        Product-->>User: true
    else 직접 권한 없음
        Product->>Gateway: traverse → permits.manage()
        Gateway->>Gateway: owners/admins에 User 포함?
        Gateway-->>Product: result
        Product-->>User: result
    end
```

```typescript
class Product implements Namespace {
  related: {
    gateway: Gateway[]
    editors: User[]
  }

  permits = {
    // 직접 권한 또는 Gateway에서 상속
    manage: (ctx: Context): boolean =>
      this.related.editors.includes(ctx.subject) ||
      this.related.gateway.traverse((g) => g.permits.manage(ctx)),

    view: (ctx: Context): boolean =>
      this.related.gateway.traverse((g) => g.permits.view(ctx)),
  }
}

class Environment implements Namespace {
  related: {
    product: Product[]
    access_managers: User[]
  }

  permits = {
    manage: (ctx: Context): boolean =>
      this.related.product.traverse((p) => p.permits.manage(ctx)),

    approve_access: (ctx: Context): boolean =>
      this.related.access_managers.includes(ctx.subject) ||
      this.related.product.traverse((p) =>
        p.related.gateway.traverse((g) => g.permits.manage_access(ctx))
      ),
  }
}
```

### 미들웨어 통합

```go
func RequireOrgPermission(ketoClient *KetoClient, permission string) gin.HandlerFunc {
    return func(c *gin.Context) {
        orgID := c.Param("id")
        userID := c.GetString("userId")

        allowed, err := ketoClient.Check(c.Request.Context(),
            "Organization", orgID, permission, userID)

        if err != nil {
            c.AbortWithStatusJSON(500, gin.H{"error": "Permission check failed"})
            return
        }

        if !allowed {
            c.AbortWithStatusJSON(403, gin.H{"error": "Forbidden"})
            return
        }

        c.Next()
    }
}
```

## 운영 고려사항

### 성능

- Keto는 읽기 작업에 최적화되어 있으며, 95번째 백분위 지연 시간이 10ms 미만
- Read API와 Write API가 분리되어 있어 읽기 확장이 용이
- 캐싱은 Keto 내부에서 처리하므로 애플리케이션 레벨 캐싱은 불필요

### 데이터 일관성

Keto는 데이터베이스(PostgreSQL, MySQL, CockroachDB)에 relation tuple을 저장한다. 트랜잭션 처리 시 애플리케이션 데이터와 Keto 데이터의 동기화에 주의해야 한다.

```go
// 권장: 애플리케이션 데이터 저장 후 Keto 업데이트
tx.Begin()
if err := saveToDatabase(data); err != nil {
    tx.Rollback()
    return err
}
tx.Commit()

// Keto 업데이트 실패 시 로그 기록 (재시도 로직 구현 권장)
if err := ketoClient.CreateRelation(...); err != nil {
    log.Error("Failed to sync with Keto", err)
}
```

### 모니터링

Keto는 Prometheus 메트릭을 4468 포트에서 제공한다.

```yaml
scrape_configs:
  - job_name: 'keto'
    static_configs:
      - targets: ['keto:4468']
```

## 참고 자료

### 공식 문서
- [Ory Keto Documentation](https://www.ory.com/docs/keto)
- [Ory Permission Language Reference](https://www.ory.com/docs/keto/reference/ory-permission-language)
- [GitHub - ory/keto](https://github.com/ory/keto)

### 관련 문서
- [Ory Kratos를 활용한 사용자 인증 시스템 구축](https://blog.imprun.dev/77) - Identity Management 상세 구현
- [Ory Hydra OAuth2/OIDC 구현 가이드](https://blog.imprun.dev/76) - OAuth2 토큰 발급 및 클라이언트 관리
- [Ory Oathkeeper를 활용한 Zero Trust IAP 구현 가이드](https://blog.imprun.dev/79) - Zero Trust Proxy 설정
- [ImpRun 인증/인가 아키텍처: Ory 스택 통합 구현 가이드](https://blog.imprun.dev/80) - 4개 컴포넌트 통합 아키텍처

### 관련 자료
- [Zanzibar: Google's Consistent, Global Authorization System](https://research.google/pubs/pub48190/)
- [Top 5 Google Zanzibar open-source implementations](https://workos.com/blog/top-5-google-zanzibar-open-source-implementations-in-2024)
