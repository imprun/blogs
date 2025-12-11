# ImpRun 인증/인가 아키텍처: Ory 스택 통합 구현 가이드

> **작성일**: 2025년 12월 6일
> **카테고리**: Authentication, Authorization, Architecture, Backend
> **키워드**: Ory Kratos, Ory Hydra, Ory Keto, Ory Oathkeeper, Zero Trust, OAuth2, ReBAC

## 요약

ImpRun v3 플랫폼은 Ory 스택 전체(Kratos, Hydra, Keto, Oathkeeper)를 통합하여 엔터프라이즈급 인증/인가 시스템을 구축했다. 각 컴포넌트가 명확한 역할을 담당하며, Zero Trust 아키텍처를 기반으로 웹 애플리케이션과 외부 API 클라이언트를 모두 지원한다. 이 글에서는 4개 Ory 컴포넌트의 조합 방식과 실제 구현 사례를 다룬다.

## Ory 스택 구성

### 컴포넌트별 역할

| 컴포넌트 | 역할 | 포트 |
|----------|------|------|
| Ory Kratos | Identity Management - 사용자 등록, 로그인, 세션 관리 | 4433 (Public), 4434 (Admin) |
| Ory Hydra | OAuth2/OIDC Server - 토큰 발급, 클라이언트 관리 | 4444 (Public), 4445 (Admin) |
| Ory Keto | Permissions - ReBAC 기반 권한 관리 | 4466 (Read), 4467 (Write) |
| Ory Oathkeeper | Identity & Access Proxy - 인증/인가 게이트키퍼 | 4455 (Proxy), 4456 (API) |

### 전체 아키텍처

```mermaid
flowchart TB
    subgraph clients["Clients"]
        Browser["Browser<br/>(Next.js)"]
        External["External API<br/>Client"]
        M2M["M2M Service"]
    end

    subgraph gateway["Gateway Layer"]
        Envoy["Envoy Gateway"]
        Oathkeeper["Oathkeeper<br/>(IAP)"]
    end

    subgraph auth["Ory Auth Stack"]
        Kratos["Kratos<br/>(Identity)"]
        Hydra["Hydra<br/>(OAuth2)"]
        Keto["Keto<br/>(Permissions)"]
    end

    subgraph backend["Backend"]
        GoAPI["Go API Server"]
        DB["PostgreSQL"]
    end

    Browser -->|"Session Cookie"| Envoy
    External -->|"Bearer Token"| Envoy
    M2M -->|"Client Credentials"| Hydra

    Envoy <-->|"ext_authz"| Oathkeeper
    Oathkeeper -->|"whoami"| Kratos
    Oathkeeper -->|"introspect"| Hydra

    Envoy -->|"X-User-* headers"| GoAPI
    GoAPI -->|"Permission Check"| Keto
    GoAPI --> DB

    Kratos --> DB
    Hydra --> DB
    Keto --> DB

    style gateway stroke:#dc2626,stroke-width:2px
    style auth stroke:#16a34a,stroke-width:2px
    style backend stroke:#2563eb,stroke-width:2px
```

## 인증 방식별 플로우

### 클라이언트 유형별 인증

| 클라이언트 | 인증 방식 | 토큰 타입 | Oathkeeper Handler |
|------------|----------|----------|-------------------|
| Web App (Next.js) | Kratos Session | Cookie | `cookie_session` |
| External API | Hydra OAuth2 | Bearer Token | `bearer_token` |
| M2M | Client Credentials | Bearer Token | `bearer_token` |

### 1. 웹 로그인 플로우 (Kratos Session)

```mermaid
sequenceDiagram
    participant Browser
    participant NextJS as Next.js
    participant Kratos
    participant Envoy
    participant Oathkeeper
    participant GoAPI as Go API

    Note over Browser,GoAPI: 로그인 Flow
    Browser->>NextJS: 1. /login 접속
    NextJS->>Kratos: 2. init flow
    Kratos-->>NextJS: flow data
    NextJS-->>Browser: render form

    Browser->>NextJS: 3. submit credentials
    NextJS->>Kratos: submit to Kratos
    Kratos-->>Browser: 4. 세션 쿠키 발급<br/>(ory_kratos_session)

    Note over Browser,GoAPI: API 요청 Flow
    Browser->>Envoy: 5. API 요청 (with cookie)
    Envoy->>Oathkeeper: 6. ext_authz
    Oathkeeper->>Kratos: 7. /sessions/whoami
    Kratos-->>Oathkeeper: session info
    Oathkeeper-->>Envoy: 8. OK + X-User-* headers
    Envoy->>GoAPI: 9. Forward request
    GoAPI-->>Browser: response
```

### 2. 외부 API 플로우 (OAuth2 Bearer Token)

```mermaid
sequenceDiagram
    participant Client as API Client
    participant Hydra
    participant Envoy
    participant Oathkeeper
    participant GoAPI as Go API

    Note over Client,GoAPI: 토큰 발급
    Client->>Hydra: 1. POST /oauth2/token<br/>(client_credentials)
    Hydra-->>Client: access_token

    Note over Client,GoAPI: API 요청
    Client->>Envoy: 2. API 요청<br/>(Authorization: Bearer ...)
    Envoy->>Oathkeeper: 3. ext_authz
    Oathkeeper->>Hydra: 4. POST /oauth2/introspect
    Hydra-->>Oathkeeper: token info (active, scope, sub)
    Oathkeeper-->>Envoy: 5. OK + headers
    Envoy->>GoAPI: 6. Forward request
    GoAPI-->>Client: response
```

### 3. 권한 검사 플로우 (Keto ReBAC)

```mermaid
sequenceDiagram
    participant GoAPI as Go API
    participant Keto
    participant DB

    Note over GoAPI,DB: Gateway 접근 권한 검사
    GoAPI->>Keto: 1. Check permission<br/>(Gateway:gw-id#view@User:user-id)

    Note over Keto: Relation Tuple 검색<br/>+ SubjectSet 확장
    Keto->>DB: 2. Query relation tuples
    DB-->>Keto: tuples

    alt 직접 권한
        Keto-->>GoAPI: 3a. allowed: true
    else 조직 상속 권한
        Note over Keto: Organization:org-id#members<br/>→ Gateway:gw-id#viewers
        Keto-->>GoAPI: 3b. allowed: true
    else 권한 없음
        Keto-->>GoAPI: 3c. allowed: false
    end
```

## Ory 컴포넌트 상세 설정

### Kratos 설정 (Identity Management)

```yaml
# kratos.yml
version: v1.1.0

dsn: postgres://kratos:secret@postgres:5432/kratos

serve:
  public:
    base_url: http://localhost:3000/.ory/
    cors:
      enabled: true
      allowed_origins:
        - http://localhost:3000

selfservice:
  default_browser_return_url: http://localhost:3000/

  methods:
    password:
      enabled: true
    oidc:
      enabled: true
      config:
        providers:
          - id: google
            provider: google
            client_id: ${GOOGLE_CLIENT_ID}
            client_secret: ${GOOGLE_CLIENT_SECRET}
            scope: [email, profile]

  flows:
    login:
      ui_url: http://localhost:3000/auth/login
    registration:
      ui_url: http://localhost:3000/auth/registration
      after:
        password:
          hooks:
            - hook: session  # 회원가입 후 자동 로그인

session:
  lifespan: 24h
  cookie:
    name: ory_kratos_session
    same_site: Lax

identity:
  default_schema_id: user
  schemas:
    - id: user
      url: file:///etc/config/kratos/schemas/user.schema.json
```

### Hydra 설정 (OAuth2/OIDC)

```yaml
# hydra.yml
version: v2.2.0

dsn: postgres://hydra:secret@postgres:5432/hydra

serve:
  public:
    cors:
      enabled: true
      allowed_origins:
        - http://localhost:3000

urls:
  self:
    issuer: http://localhost:4444/
  consent: http://localhost:3000/auth/consent
  login: http://localhost:3000/auth/login
  logout: http://localhost:3000/auth/logout

strategies:
  access_token: jwt

oauth2:
  allowed_top_level_claims:
    - namespace
    - roles
    - scopes
```

### Keto 설정 (Permissions)

```yaml
# keto.yml
version: v0.14.0

dsn: postgres://keto:secret@postgres:5432/keto

namespaces:
  location: file:///etc/config/keto/namespaces.keto.ts

serve:
  read:
    host: 0.0.0.0
    port: 4466
  write:
    host: 0.0.0.0
    port: 4467
```

**OPL Namespace 정의:**

```typescript
// namespaces.keto.ts
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

class Gateway implements Namespace {
  related: {
    owners: (User | SubjectSet<Organization, "admins">)[]
    admins: (User | SubjectSet<Organization, "admins">)[]
    members: User[]
    viewers: (User | SubjectSet<Organization, "members">)[]
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

    delete: (ctx: Context): boolean =>
      this.related.owners.includes(ctx.subject),
  }
}
```

### Oathkeeper 설정 (IAP)

```yaml
# oathkeeper.yml
serve:
  proxy:
    port: 4455
  api:
    port: 4456

access_rules:
  repositories:
    - file:///etc/config/oathkeeper/access-rules.yml

authenticators:
  noop:
    enabled: true
  cookie_session:
    enabled: true
    config:
      check_session_url: http://kratos:4433/sessions/whoami
      preserve_path: true
      extra_from: "@this"
      subject_from: "identity.id"
      only:
        - ory_kratos_session
  bearer_token:
    enabled: true
    config:
      check_session_url: http://hydra:4445/oauth2/introspect
      token_from:
        header: Authorization

authorizers:
  allow:
    enabled: true

mutators:
  header:
    enabled: true
    config:
      headers:
        X-User-Id: "{{ print .Subject }}"
        X-User-Email: "{{ print .Extra.identity.traits.email }}"

errors:
  fallback:
    - json
  handlers:
    json:
      enabled: true
    redirect:
      enabled: true
      config:
        to: http://localhost:3000/auth/login
```

## Go 백엔드 통합

### SDK 설치

```bash
# Kratos Client
go get github.com/ory/kratos-client-go

# Hydra Client
go get github.com/ory/hydra-client-go

# Keto Client (REST API)
go get github.com/ory/keto-client-go

# OAuth2 토큰 발급용 (공식 권장)
go get golang.org/x/oauth2
```

공식 문서에서 `/oauth2/auth`와 `/oauth2/token` 엔드포인트는 Hydra SDK가 아닌 `golang.org/x/oauth2` 패키지 사용을 권장한다.

### 클라이언트 구조

```go
// internal/infrastructure/ory/clients.go
package ory

import (
    kratosclient "github.com/ory/kratos-client-go"
    hydraclient "github.com/ory/hydra-client-go"
    ketoclient "github.com/ory/keto-client-go"
)

type OryClients struct {
    Kratos *KratosClient
    Hydra  *HydraClient
    Keto   *KetoClient
}

func NewOryClients(cfg *config.OryConfig) *OryClients {
    return &OryClients{
        Kratos: NewKratosClient(cfg.Kratos.PublicURL, cfg.Kratos.AdminURL),
        Hydra:  NewHydraClient(cfg.Hydra.AdminURL),
        Keto:   NewKetoClient(cfg.Keto.ReadURL, cfg.Keto.WriteURL),
    }
}
```

### Kratos Client

```go
// internal/infrastructure/ory/kratos.go
package ory

import (
    "context"
    kratosclient "github.com/ory/kratos-client-go"
)

type KratosClient struct {
    frontendAPI *kratosclient.FrontendApi
    identityAPI *kratosclient.IdentityApi
}

func NewKratosClient(publicURL, adminURL string) *KratosClient {
    // Public API (포트 4433) - 세션 검증용
    publicConfig := kratosclient.NewConfiguration()
    publicConfig.Servers = []kratosclient.ServerConfiguration{{URL: publicURL}}
    publicClient := kratosclient.NewAPIClient(publicConfig)

    // Admin API (포트 4434) - Identity 관리용
    adminConfig := kratosclient.NewConfiguration()
    adminConfig.Servers = []kratosclient.ServerConfiguration{{URL: adminURL}}
    adminClient := kratosclient.NewAPIClient(adminConfig)

    return &KratosClient{
        frontendAPI: publicClient.FrontendApi,
        identityAPI: adminClient.IdentityApi,
    }
}

// ToSession: Cookie로 세션 검증
func (c *KratosClient) ToSession(ctx context.Context, cookie string) (*kratosclient.Session, error) {
    session, _, err := c.frontendAPI.ToSession(ctx).
        Cookie(cookie).
        Execute()
    return session, err
}

// GetIdentity: Identity 정보 조회
func (c *KratosClient) GetIdentity(ctx context.Context, id string) (*kratosclient.Identity, error) {
    identity, _, err := c.identityAPI.GetIdentity(ctx, id).Execute()
    return identity, err
}
```

### Hydra Client

```go
// internal/infrastructure/ory/hydra.go
package ory

import (
    "context"
    hydraclient "github.com/ory/hydra-client-go"
    "golang.org/x/oauth2/clientcredentials"
)

type HydraClient struct {
    oauth2API *hydraclient.OAuth2Api
}

func NewHydraClient(adminURL string) *HydraClient {
    config := hydraclient.NewConfiguration()
    config.Servers = []hydraclient.ServerConfiguration{{URL: adminURL}}
    client := hydraclient.NewAPIClient(config)

    return &HydraClient{
        oauth2API: client.OAuth2Api,
    }
}

// IntrospectToken: Access Token 검증
func (c *HydraClient) IntrospectToken(ctx context.Context, token string) (*hydraclient.IntrospectedOAuth2Token, error) {
    result, _, err := c.oauth2API.IntrospectOAuth2Token(ctx).
        Token(token).
        Execute()
    return result, err
}

// CreateOAuth2Client: Service Account용 OAuth2 클라이언트 생성
func (c *HydraClient) CreateOAuth2Client(ctx context.Context, name string, scopes []string) (*hydraclient.OAuth2Client, error) {
    body := *hydraclient.NewOAuth2Client()
    body.SetClientName(name)
    body.SetGrantTypes([]string{"client_credentials"})
    body.SetScope(strings.Join(scopes, " "))
    body.SetTokenEndpointAuthMethod("client_secret_basic")

    client, _, err := c.oauth2API.CreateOAuth2Client(ctx).
        OAuth2Client(body).
        Execute()
    return client, err
}

// GetAccessToken: Client Credentials로 토큰 발급 (golang.org/x/oauth2 사용)
func GetAccessToken(ctx context.Context, tokenURL, clientID, clientSecret string, scopes []string) (string, error) {
    config := clientcredentials.Config{
        TokenURL:     tokenURL,
        ClientID:     clientID,
        ClientSecret: clientSecret,
        Scopes:       scopes,
    }
    token, err := config.Token(ctx)
    if err != nil {
        return "", err
    }
    return token.AccessToken, nil
}
```

### Keto Client

```go
// internal/infrastructure/ory/keto.go
package ory

import (
    "context"
    ketoclient "github.com/ory/keto-client-go"
)

type KetoClient struct {
    permissionAPI   *ketoclient.PermissionApi
    relationshipAPI *ketoclient.RelationshipApi
}

func NewKetoClient(readURL, writeURL string) *KetoClient {
    // Read API (포트 4466) - 권한 체크용
    readConfig := ketoclient.NewConfiguration()
    readConfig.Servers = []ketoclient.ServerConfiguration{{URL: readURL}}
    readClient := ketoclient.NewAPIClient(readConfig)

    // Write API (포트 4467) - Relation Tuple 관리용
    writeConfig := ketoclient.NewConfiguration()
    writeConfig.Servers = []ketoclient.ServerConfiguration{{URL: writeURL}}
    writeClient := ketoclient.NewAPIClient(writeConfig)

    return &KetoClient{
        permissionAPI:   readClient.PermissionApi,
        relationshipAPI: writeClient.RelationshipApi,
    }
}

// CheckPermission: 권한 체크
func (c *KetoClient) CheckPermission(ctx context.Context, namespace, object, relation, subjectID string) (bool, error) {
    result, _, err := c.permissionAPI.CheckPermission(ctx).
        Namespace(namespace).
        Object(object).
        Relation(relation).
        SubjectId(subjectID).
        Execute()
    if err != nil {
        return false, err
    }
    return result.GetAllowed(), nil
}

// CreateRelationship: Relation Tuple 생성 (직접 역할 부여)
func (c *KetoClient) CreateRelationship(ctx context.Context, namespace, object, relation, subjectID string) error {
    body := ketoclient.CreateRelationshipBody{
        Namespace: ketoclient.PtrString(namespace),
        Object:    ketoclient.PtrString(object),
        Relation:  ketoclient.PtrString(relation),
        SubjectId: ketoclient.PtrString(subjectID),
    }
    _, _, err := c.relationshipAPI.CreateRelationship(ctx).
        CreateRelationshipBody(body).
        Execute()
    return err
}

// CreateRelationshipWithSubjectSet: 팀 역할 상속용
func (c *KetoClient) CreateRelationshipWithSubjectSet(
    ctx context.Context,
    namespace, object, relation string,
    subjectNamespace, subjectObject, subjectRelation string,
) error {
    body := ketoclient.CreateRelationshipBody{
        Namespace: ketoclient.PtrString(namespace),
        Object:    ketoclient.PtrString(object),
        Relation:  ketoclient.PtrString(relation),
        SubjectSet: &ketoclient.SubjectSet{
            Namespace: subjectNamespace,
            Object:    subjectObject,
            Relation:  subjectRelation,
        },
    }
    _, _, err := c.relationshipAPI.CreateRelationship(ctx).
        CreateRelationshipBody(body).
        Execute()
    return err
}
```

### 통합 인증 미들웨어

Oathkeeper가 전달한 헤더를 사용하거나, 직접 Kratos/Hydra를 호출하는 미들웨어:

```go
// internal/interface/middleware/auth.go
func AuthMiddleware(oryClients *client.OryClients, useOathkeeper bool) gin.HandlerFunc {
    return func(c *gin.Context) {
        // 1. Oathkeeper 모드: 헤더에서 사용자 정보 추출
        if useOathkeeper {
            userID := c.GetHeader("X-User-Id")
            if userID != "" {
                user := &entity.UserSession{
                    ID:        uuid.MustParse(userID),
                    Email:     c.GetHeader("X-User-Email"),
                    TokenType: "oathkeeper",
                }
                c.Set("user", user)
                c.Next()
                return
            }
        }

        // 2. Bearer Token 검증 (Hydra)
        authHeader := c.GetHeader("Authorization")
        if strings.HasPrefix(authHeader, "Bearer ") {
            token := strings.TrimPrefix(authHeader, "Bearer ")
            introspection, err := oryClients.Hydra.Introspect(c.Request.Context(), token)
            if err == nil && introspection.Active {
                user := &entity.UserSession{
                    ID:        uuid.MustParse(introspection.Subject),
                    TokenType: "oauth2",
                    Scopes:    introspection.Scope,
                }
                c.Set("user", user)
                c.Next()
                return
            }
        }

        // 3. Session Cookie 검증 (Kratos)
        sessionCookie, err := c.Cookie("ory_kratos_session")
        if err == nil && sessionCookie != "" {
            session, err := oryClients.Kratos.WhoAmI(c.Request.Context(), sessionCookie)
            if err == nil && session.Active {
                user := &entity.UserSession{
                    ID:        uuid.MustParse(session.Identity.ID),
                    Email:     session.Identity.Traits.Email,
                    TokenType: "session",
                }
                c.Set("user", user)
                c.Next()
                return
            }
        }

        c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
            "error": "No valid authentication provided",
        })
    }
}
```

### Keto 권한 검사 미들웨어

```go
// internal/interface/middleware/permission.go
func GatewayPermissionMiddleware(ketoClient *client.KetoClient, permission string) gin.HandlerFunc {
    return func(c *gin.Context) {
        gatewayID := c.Param("gw")
        if gatewayID == "" {
            c.Next()
            return
        }

        user, exists := c.Get("user")
        if !exists {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "Unauthorized",
            })
            return
        }

        userSession := user.(*entity.UserSession)

        // Keto 권한 검사
        allowed, err := ketoClient.Check(c.Request.Context(), KetoCheckRequest{
            Namespace: "Gateway",
            Object:    gatewayID,
            Relation:  permission,
            SubjectID: userSession.ID.String(),
        })

        if err != nil {
            c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{
                "error": "Permission check failed",
            })
            return
        }

        if !allowed {
            c.AbortWithStatusJSON(http.StatusForbidden, gin.H{
                "error":    "Forbidden",
                "resource": "Gateway",
                "action":   permission,
            })
            return
        }

        c.Set("currentGateway", gatewayID)
        c.Next()
    }
}
```

### 라우터 설정

```go
// internal/interface/router/router.go
func SetupRouter(oryClients *client.OryClients) *gin.Engine {
    r := gin.Default()

    // Public endpoints
    public := r.Group("/api/public")
    {
        public.GET("/health", healthCheck)
        public.GET("/directory/*path", getDirectory)
    }

    // Protected endpoints
    protected := r.Group("/api")
    protected.Use(AuthMiddleware(oryClients, true))
    {
        // User endpoints
        protected.GET("/me", getMe)
        protected.GET("/organizations", listOrganizations)

        // Gateway endpoints with permission checks
        gw := protected.Group("/gateways/:gw")
        {
            gw.GET("", GatewayPermissionMiddleware(oryClients.Keto, "view"), getGateway)
            gw.PUT("", GatewayPermissionMiddleware(oryClients.Keto, "manage"), updateGateway)
            gw.DELETE("", GatewayPermissionMiddleware(oryClients.Keto, "delete"), deleteGateway)

            // Products
            gw.GET("/products", GatewayPermissionMiddleware(oryClients.Keto, "view"), listProducts)
            gw.POST("/products", GatewayPermissionMiddleware(oryClients.Keto, "manage"), createProduct)

            // Config
            gw.PUT("/config", GatewayPermissionMiddleware(oryClients.Keto, "publish_config"), publishConfig)
        }
    }

    return r
}
```

## Next.js 프론트엔드 통합

### Ory Client 설정

```typescript
// shared/api/ory.ts
import { Configuration, FrontendApi } from '@ory/client';

const KRATOS_URL = '/.ory';

export const kratosClient = new FrontendApi(
  new Configuration({
    basePath: KRATOS_URL,
    baseOptions: {
      withCredentials: true,
    },
  })
);
```

### Next.js Proxy 설정

```typescript
// next.config.ts
const nextConfig: NextConfig = {
  async rewrites() {
    return [
      {
        source: '/.ory/:path*',
        destination: `${process.env.KRATOS_URL || 'http://localhost:4433'}/:path*`,
      },
    ];
  },
};
```

### Auth Provider

```typescript
// entities/auth/model/auth-provider.tsx
'use client';

import { createContext, useContext, useEffect, useState } from 'react';
import { Session } from '@ory/client';
import { kratosClient } from '@/shared/api/ory';

interface AuthContextType {
  session: Session | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  logout: () => Promise<void>;
}

const AuthContext = createContext<AuthContextType | null>(null);

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [session, setSession] = useState<Session | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    kratosClient.toSession()
      .then(({ data }) => {
        setSession(data);
      })
      .catch(() => {
        setSession(null);
      })
      .finally(() => {
        setIsLoading(false);
      });
  }, []);

  const logout = async () => {
    const { data } = await kratosClient.createBrowserLogoutFlow();
    window.location.href = data.logout_url;
  };

  return (
    <AuthContext.Provider value={{
      session,
      isAuthenticated: !!session,
      isLoading,
      logout,
    }}>
      {children}
    </AuthContext.Provider>
  );
}

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) throw new Error('useAuth must be used within AuthProvider');
  return context;
};
```

## Docker Compose 전체 구성

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: root
      POSTGRES_PASSWORD: secret
    volumes:
      - ./docker/postgres/init:/docker-entrypoint-initdb.d
      - postgres_data:/var/lib/postgresql/data

  kratos:
    image: oryd/kratos:v1.1.0
    environment:
      DSN: postgres://kratos:secret@postgres:5432/kratos
    ports:
      - "4433:4433"
      - "4434:4434"
    volumes:
      - ./docker/ory/kratos:/etc/config/kratos
    command: serve -c /etc/config/kratos/kratos.yml --dev --watch-courier
    depends_on:
      - postgres

  hydra:
    image: oryd/hydra:v2.2.0
    environment:
      DSN: postgres://hydra:secret@postgres:5432/hydra
      URLS_CONSENT: http://localhost:3000/auth/consent
      URLS_LOGIN: http://localhost:3000/auth/login
    ports:
      - "4444:4444"
      - "4445:4445"
    volumes:
      - ./docker/ory/hydra:/etc/config/hydra
    command: serve all -c /etc/config/hydra/hydra.yml --dev
    depends_on:
      - postgres

  keto:
    image: oryd/keto:v0.14.0
    environment:
      DSN: postgres://keto:secret@postgres:5432/keto
    ports:
      - "4466:4466"
      - "4467:4467"
    volumes:
      - ./docker/ory/keto:/etc/config/keto
    command: serve -c /etc/config/keto/keto.yml
    depends_on:
      - postgres

  oathkeeper:
    image: oryd/oathkeeper:v0.40.7
    ports:
      - "4455:4455"
      - "4456:4456"
    volumes:
      - ./docker/ory/oathkeeper:/etc/config/oathkeeper
    command: serve --config /etc/config/oathkeeper/oathkeeper.yml
    depends_on:
      - kratos
      - hydra

  envoy:
    image: envoyproxy/envoy:v1.32-latest
    ports:
      - "8080:8080"
      - "9901:9901"
    volumes:
      - ./docker/envoy/envoy.yaml:/etc/envoy/envoy.yaml
    depends_on:
      - oathkeeper

  api:
    build:
      context: ./apps/api
      dockerfile: Dockerfile
    environment:
      - KRATOS_PUBLIC_URL=http://kratos:4433
      - KRATOS_ADMIN_URL=http://kratos:4434
      - HYDRA_PUBLIC_URL=http://hydra:4444
      - HYDRA_ADMIN_URL=http://hydra:4445
      - KETO_READ_URL=http://keto:4466
      - KETO_WRITE_URL=http://keto:4467
      - USE_OATHKEEPER=true
    depends_on:
      - kratos
      - hydra
      - keto

  web:
    build:
      context: ./apps/web
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - KRATOS_URL=http://kratos:4433
    depends_on:
      - api
      - kratos

  mailhog:
    image: mailhog/mailhog
    ports:
      - "8025:8025"

volumes:
  postgres_data:
```

## 데이터 흐름 요약

### 인증 데이터 흐름

```
1. 사용자 로그인 (Kratos)
   Browser → Next.js → Kratos → Session Cookie

2. API 요청
   Browser (Cookie) → Envoy → Oathkeeper → Kratos (whoami)
                                         → Headers 추가
                                         → Go API

3. 권한 검사
   Go API → Keto (check) → allowed/denied

4. 사용자 데이터 동기화
   Go API (첫 요청) → Kratos Identity → User 테이블 생성
```

### 컴포넌트별 책임

| 컴포넌트 | 핵심 질문 | 담당 기능 |
|----------|----------|----------|
| **Kratos** | WHO - 사용자가 누구인가? | 회원가입, 로그인, 세션 관리, 소셜 로그인 (OIDC), 이메일 인증, 비밀번호 복구 |
| **Hydra** | TOKEN - 접근 토큰은 무엇인가? | Access Token / Refresh Token 발급, Client Credentials (M2M), Token Introspection |
| **Keto** | CAN - 무엇을 할 수 있는가? | ReBAC 기반 권한 모델, Relation Tuple 관리, 권한 상속 (Organization → Gateway) |
| **Oathkeeper** | GATE - 통과할 수 있는가? | 모든 요청 검증, Kratos/Hydra 연동, 사용자 정보 헤더 주입 |

## DB와 Keto 동기화

### Transactional Outbox 패턴

DB와 Keto 간 일관성을 보장하기 위해 Transactional Outbox 패턴을 사용한다.

```mermaid
sequenceDiagram
    participant API as Go API
    participant DB as PostgreSQL
    participant Worker as Outbox Worker
    participant Keto

    Note over API,Keto: 역할 부여 요청
    API->>DB: 1. BEGIN TRANSACTION
    API->>DB: 2. INSERT user_gateway_roles
    API->>DB: 3. INSERT keto_outbox (pending)
    API->>DB: 4. COMMIT

    Note over Worker,Keto: 비동기 동기화
    Worker->>DB: 5. SELECT pending events
    Worker->>Keto: 6. CreateRelationship
    Worker->>DB: 7. UPDATE status = completed
```

### Outbox 테이블

```sql
CREATE TABLE keto_outbox (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type  VARCHAR(50) NOT NULL,  -- create_tuple, delete_tuple
    payload     JSONB NOT NULL,
    status      VARCHAR(20) DEFAULT 'pending',  -- pending, completed, failed
    retry_count INT DEFAULT 0,
    error_message TEXT,
    created_at  TIMESTAMP DEFAULT now(),
    processed_at TIMESTAMP
);

CREATE INDEX idx_keto_outbox_pending ON keto_outbox(status) WHERE status = 'pending';
```

### Outbox Worker 구현

```go
// internal/worker/keto_outbox.go
type KetoOutboxWorker struct {
    ketoClient *ory.KetoClient
    db         *gorm.DB
}

func (w *KetoOutboxWorker) ProcessPendingEvents(ctx context.Context) error {
    var events []entity.KetoOutbox
    w.db.Where("status = ?", "pending").Limit(100).Find(&events)

    for _, event := range events {
        if err := w.processEvent(ctx, &event); err != nil {
            event.RetryCount++
            event.ErrorMessage = err.Error()
            if event.RetryCount >= 3 {
                event.Status = "failed"
            }
        } else {
            event.Status = "completed"
            event.ProcessedAt = time.Now()
        }
        w.db.Save(&event)
    }
    return nil
}

func (w *KetoOutboxWorker) processEvent(ctx context.Context, event *entity.KetoOutbox) error {
    payload := event.Payload

    switch event.EventType {
    case "create_tuple":
        if subjectNS, ok := payload["subject_namespace"]; ok {
            return w.ketoClient.CreateRelationshipWithSubjectSet(ctx,
                payload["namespace"].(string),
                payload["object"].(string),
                payload["relation"].(string),
                subjectNS.(string),
                payload["subject_object"].(string),
                payload["subject_relation"].(string),
            )
        }
        return w.ketoClient.CreateRelationship(ctx,
            payload["namespace"].(string),
            payload["object"].(string),
            payload["relation"].(string),
            payload["subject_id"].(string),
        )

    case "delete_tuple":
        // 삭제 로직
    }
    return nil
}
```

## 운영 고려사항

### 성능

- 각 Ory 컴포넌트는 stateless → 수평 확장 가능
- PostgreSQL 연결 풀 적절히 설정
- Oathkeeper 세션 캐싱으로 Kratos/Hydra 부하 감소

### Keto 권한 캐싱

매 요청마다 Keto API 호출은 부하를 유발할 수 있다. Redis 캐싱으로 최적화:

```go
// internal/interface/middleware/permission_cached.go
func CachedGatewayPermissionMiddleware(
    ketoClient *ory.KetoClient,
    redis *redis.Client,
    permission string,
) gin.HandlerFunc {
    return func(c *gin.Context) {
        userID := c.GetString("user_id")
        gatewayID := c.Param("gw")

        cacheKey := fmt.Sprintf("perm:%s:%s:%s", userID, gatewayID, permission)

        // 1. 캐시 확인
        if cached, err := redis.Get(c, cacheKey).Result(); err == nil {
            if cached == "1" {
                c.Next()
                return
            }
            c.AbortWithStatusJSON(403, gin.H{"error": "Forbidden"})
            return
        }

        // 2. Keto 조회
        allowed, err := ketoClient.CheckPermission(c, "Gateway", gatewayID, permission, userID)
        if err != nil {
            c.AbortWithStatusJSON(500, gin.H{"error": "Permission check failed"})
            return
        }

        // 3. 캐싱 (TTL 5분)
        value := "0"
        if allowed {
            value = "1"
        }
        redis.Set(c, cacheKey, value, 5*time.Minute)

        if !allowed {
            c.AbortWithStatusJSON(403, gin.H{"error": "Forbidden"})
            return
        }

        c.Next()
    }
}
```

**캐시 무효화**: 권한 변경 시 관련 캐시 키 삭제

```go
func InvalidatePermissionCache(redis *redis.Client, userID, gatewayID string) {
    pattern := fmt.Sprintf("perm:%s:%s:*", userID, gatewayID)
    keys, _ := redis.Keys(context.Background(), pattern).Result()
    if len(keys) > 0 {
        redis.Del(context.Background(), keys...)
    }
}
```

### 보안

1. **Admin API 격리**: 4434, 4445, 4467 포트는 내부 네트워크에서만 접근
2. **HTTPS 필수**: 프로덕션에서 모든 통신 암호화
3. **Secret 관리**: 환경 변수 또는 Vault 사용

### 모니터링

각 컴포넌트는 Prometheus 메트릭 제공:

```yaml
# 메트릭 엔드포인트
Kratos:     http://kratos:4434/metrics
Hydra:      http://hydra:4445/metrics
Keto:       http://keto:4468/metrics
Oathkeeper: http://oathkeeper:9000/metrics
```

## 참고 자료

### 공식 문서
- [Ory Documentation](https://www.ory.sh/docs)
- [Ory Kratos](https://www.ory.sh/docs/kratos)
- [Ory Hydra](https://www.ory.sh/docs/hydra)
- [Ory Keto](https://www.ory.sh/docs/keto)
- [Ory Oathkeeper](https://www.ory.sh/docs/oathkeeper)

### Go SDK 문서
- [Ory Kratos Go SDK](https://www.ory.com/docs/kratos/self-hosted/go) - Kratos 클라이언트 설정 및 API 사용법
- [Ory Hydra Go SDK](https://www.ory.com/docs/hydra/self-hosted/go) - Hydra 클라이언트 설정, Token Introspection
- [Ory Keto Go SDK](https://www.ory.com/docs/keto/self-hosted/go) - Keto 클라이언트 설정, Permission Check

### 관련 문서
- [Ory Kratos를 활용한 사용자 인증 시스템 구축](https://blog.imprun.dev/77) - Identity Management 상세 구현
- [Ory Hydra OAuth2/OIDC 구현 가이드](https://blog.imprun.dev/76) - OAuth2 토큰 발급 및 클라이언트 관리
- [Ory Keto를 활용한 ReBAC 기반 권한 관리 시스템 구축](https://blog.imprun.dev/78) - 권한 모델 및 Relation Tuple
- [Ory Oathkeeper를 활용한 Zero Trust IAP 구현 가이드](https://blog.imprun.dev/79) - Zero Trust Proxy 설정
