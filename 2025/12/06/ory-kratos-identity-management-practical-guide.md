# Ory Kratos를 활용한 사용자 인증 시스템 구축: ImpRun 프로젝트 적용 사례

> **작성일**: 2025년 12월 6일
> **카테고리**: Authentication, Identity, Backend, Frontend
> **키워드**: Ory Kratos, Authentication, Identity Management, OAuth, OIDC, Session

## 요약

Ory Kratos는 클라우드 네이티브 환경을 위한 오픈소스 ID 관리 시스템이다. 자체적인 로그인/회원가입 UI를 제공하지 않고, API 기반으로 동작하여 프론트엔드에서 완전한 UI 커스터마이징이 가능하다. 이 글에서는 ImpRun 프로젝트에서 Kratos를 Next.js 프론트엔드와 Go 백엔드에 통합한 실제 구현 사례를 다룬다.

## Ory Kratos 개요

### Kratos란

Kratos는 Ory 프로젝트의 ID 관리 컴포넌트로, 다음 기능을 제공한다:

- 사용자 등록 및 로그인
- 비밀번호 및 소셜 로그인 (OIDC)
- 이메일/SMS 인증
- 비밀번호 복구
- 프로필 관리
- 세션 관리
- MFA (TOTP)

### Headless 아키텍처

Kratos는 UI를 제공하지 않는 Headless 아키텍처를 채택했다.

```mermaid
graph LR
    Frontend["Frontend<br/>(Next.js)"]
    Kratos["Kratos<br/>(API Only)"]
    Database["Database<br/>(PostgreSQL)"]
    SMTP["SMTP Server<br/>(이메일 발송)"]
    Backend["Backend API<br/>(Go/Gin)"]

    Frontend --> Kratos
    Kratos --> Database
    Kratos --> SMTP
    Frontend --> Backend

    style Frontend stroke:#2563eb,stroke-width:2px
    style Kratos stroke:#16a34a,stroke-width:2px
    style Database stroke:#4b5563,stroke-width:2px
    style SMTP stroke:#4b5563,stroke-width:2px
    style Backend stroke:#ea580c,stroke-width:2px
```

이 구조의 장점:
- 프론트엔드에서 완전한 UI 제어
- 브랜드에 맞는 디자인 적용 가능
- SPA, SSR, 모바일 앱 등 다양한 클라이언트 지원

## Self-Service Flows

### Flow 개념

Kratos는 인증 과정을 "Self-Service Flow"로 모델링한다. 각 Flow는 고유 ID를 가지며, 상태 머신처럼 동작한다.

| Flow | 설명 |
|------|------|
| Login | 기존 사용자 로그인 |
| Registration | 신규 사용자 등록 |
| Recovery | 비밀번호 복구 |
| Verification | 이메일/전화번호 인증 |
| Settings | 프로필 및 비밀번호 변경 |
| Logout | 로그아웃 |

### Flow 처리 과정

```
1. 클라이언트가 Flow 생성 요청
   GET /self-service/login/browser

2. Kratos가 Flow ID와 UI 노드 반환
   {
     "id": "flow-uuid",
     "ui": {
       "nodes": [
         { "type": "input", "attributes": { "name": "identifier" } },
         { "type": "input", "attributes": { "name": "password" } }
       ]
     }
   }

3. 클라이언트가 폼 데이터와 함께 Flow 제출
   POST /self-service/login?flow=flow-uuid

4. Kratos가 세션 생성 및 쿠키 설정
```

## ImpRun 프로젝트 설정

### Docker Compose 구성

```yaml
services:
  kratos:
    image: oryd/kratos:v1.1.0
    ports:
      - "4433:4433"  # Public API
      - "4434:4434"  # Admin API
    volumes:
      - ./docker/ory/kratos:/etc/config/kratos
    environment:
      - DSN=postgres://kratos:secret@postgres:5432/kratos?sslmode=disable
    command: serve -c /etc/config/kratos/kratos.yml --dev --watch-courier
```

### kratos.yml 설정

```yaml
version: v1.1.0

dsn: postgres://kratos:secret@postgres:5432/kratos?sslmode=disable

serve:
  public:
    base_url: http://localhost:3000/.ory/
    cors:
      enabled: true
      allowed_origins:
        - http://localhost:3000
      allowed_methods:
        - POST
        - GET
        - PUT
        - PATCH
        - DELETE
      allowed_headers:
        - Authorization
        - Cookie
        - Content-Type
      exposed_headers:
        - Content-Type
        - Set-Cookie
  admin:
    base_url: http://kratos:4434/

selfservice:
  default_browser_return_url: http://localhost:3000/
  allowed_return_urls:
    - http://localhost:3000

  methods:
    password:
      enabled: true
      config:
        haveibeenpwned_enabled: false
        min_password_length: 8
    totp:
      enabled: true
      config:
        issuer: ImpRun
    oidc:
      enabled: true
      config:
        providers:
          - id: google
            provider: google
            mapper_url: file:///etc/config/kratos/oidc.google.jsonnet
            client_id: ${GOOGLE_CLIENT_ID}
            client_secret: ${GOOGLE_CLIENT_SECRET}
            scope:
              - email
              - profile

  flows:
    error:
      ui_url: http://localhost:3000/error
    settings:
      ui_url: http://localhost:3000/settings
      privileged_session_max_age: 15m
    recovery:
      enabled: true
      ui_url: http://localhost:3000/recovery
      use: code
    verification:
      enabled: true
      ui_url: http://localhost:3000/verification
      use: code
    logout:
      after:
        default_browser_return_url: http://localhost:3000/login
    login:
      ui_url: http://localhost:3000/login
      lifespan: 10m
    registration:
      enabled: true
      ui_url: http://localhost:3000/register
      lifespan: 10m
      after:
        password:
          hooks:
            - hook: session  # 회원가입 후 자동 로그인

session:
  lifespan: 24h
  cookie:
    name: ory_kratos_session
    path: /
    same_site: Lax

identity:
  default_schema_id: user
  schemas:
    - id: user
      url: file:///etc/config/kratos/schemas/user.schema.json

courier:
  smtp:
    connection_uri: smtp://mailhog:1025/?skip_ssl_verify=true
    from_name: ImpRun Platform
    from_address: noreply@imprun.io
```

### Identity Schema

사용자 속성을 정의하는 JSON Schema:

```json
{
  "$id": "https://schemas.imprun.io/user.schema.json",
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "User",
  "type": "object",
  "properties": {
    "traits": {
      "type": "object",
      "properties": {
        "email": {
          "type": "string",
          "format": "email",
          "title": "Email",
          "ory.sh/kratos": {
            "credentials": {
              "password": {
                "identifier": true
              }
            },
            "verification": {
              "via": "email"
            },
            "recovery": {
              "via": "email"
            }
          }
        },
        "name": {
          "type": "string",
          "title": "Name",
          "minLength": 1,
          "maxLength": 128
        },
        "username": {
          "type": "string",
          "title": "Username",
          "minLength": 3,
          "maxLength": 32,
          "pattern": "^[a-zA-Z0-9_-]+$"
        }
      },
      "required": ["email"],
      "additionalProperties": false
    }
  }
}
```

`ory.sh/kratos` 확장을 통해 email을 로그인 식별자로 지정하고, 복구 및 인증에 사용하도록 설정한다.

### Google OIDC Mapper

소셜 로그인 시 Kratos Identity로 매핑하는 Jsonnet 파일:

```jsonnet
local claims = std.extVar('claims');

{
  identity: {
    traits: {
      email: claims.email,
      name: if "name" in claims then claims.name
            else (if "given_name" in claims && "family_name" in claims
                  then claims.given_name + " " + claims.family_name
                  else claims.email),
    },
  },
}
```

## Next.js 프론트엔드 통합

### Proxy 설정

CORS 이슈를 피하기 위해 Next.js rewrites를 사용한다:

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

프론트엔드는 `/.ory/` 경로로 요청하고, Next.js가 Kratos로 프록시한다.

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

export async function getSession() {
  try {
    const { data: session } = await kratosClient.toSession();
    return session;
  } catch {
    return null;
  }
}
```

### Auth API Layer

FSD 아키텍처에 따른 API 계층:

```typescript
// entities/auth/api/auth-api.ts
import { UpdateLoginFlowBody } from '@ory/client';
import { kratosClient } from '@/shared/api/ory';

export const authApi = {
  // Login Flow
  async getLoginFlow(returnTo?: string) {
    return kratosClient.createBrowserLoginFlow({ returnTo });
  },

  async getLoginFlowById(flowId: string) {
    return kratosClient.getLoginFlow({ id: flowId });
  },

  async updateLoginFlow(flowId: string, body: UpdateLoginFlowBody) {
    return kratosClient.updateLoginFlow({
      flow: flowId,
      updateLoginFlowBody: body,
    });
  },

  // Registration Flow
  async getRegistrationFlow(returnTo?: string) {
    return kratosClient.createBrowserRegistrationFlow({ returnTo });
  },

  // ... Recovery, Verification, Settings flows

  // Session
  async toSession() {
    return kratosClient.toSession();
  },

  async logout() {
    return kratosClient.createBrowserLogoutFlow();
  },
};
```

### Login Form 구현

```tsx
// features/auth/ui/login-form.tsx
'use client';

import { useForm } from 'react-hook-form';
import { useLoginFlow, useLoginMutation } from '@/entities/auth';
import type { UiNodeInputAttributes } from '@ory/client';

export function LoginForm() {
  const searchParams = useSearchParams();
  const flowId = searchParams.get('flow');
  const returnTo = searchParams.get('return_to') || '/';

  // Flow 조회
  const { data: flow, isLoading } = useLoginFlow(flowId || undefined, returnTo);

  const { register, handleSubmit } = useForm<LoginValues>();
  const loginMutation = useLoginMutation();

  const onSubmit = (values: LoginValues) => {
    if (!flow?.id) return;

    // CSRF 토큰 추출
    const csrfNode = flow.ui.nodes.find((node) => {
      const attrs = node.attributes as { name?: string };
      return attrs.name === 'csrf_token';
    });
    const csrfToken = (csrfNode?.attributes as UiNodeInputAttributes)?.value;

    loginMutation.mutate({
      flowId: flow.id,
      body: {
        csrf_token: csrfToken,
        method: 'password',
        identifier: values.identifier,
        password: values.password,
      },
    }, {
      onSuccess: () => {
        window.location.href = returnTo;
      },
    });
  };

  // Social Login (OIDC)
  const handleSocialLogin = (provider: string) => {
    const csrfNode = flow.ui.nodes.find(
      (n) => n.attributes.name === 'csrf_token'
    );
    const csrfToken = (csrfNode?.attributes as UiNodeInputAttributes)?.value;

    loginMutation.mutate({
      flowId: flow.id,
      body: {
        method: 'oidc',
        provider: provider,
        csrf_token: csrfToken,
      },
    }, {
      onError: (err: any) => {
        // OIDC는 redirect 응답을 에러로 반환 (422 또는 433)
        // 422: Unprocessable Entity - 브라우저 리다이렉트 필요
        // 433: 커스텀 에러 코드 - 동일하게 리다이렉트 처리
        if (err.response?.status === 422 || err.response?.status === 433) {
          const redirectUrl = err.response?.data?.redirect_browser_to;
          if (redirectUrl) {
            window.location.href = redirectUrl;
            return;
          }
        }
        // 일반 에러 처리
        console.error('OIDC login failed:', err);
      },
    });
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {/* OIDC 버튼 */}
      {flow?.ui.nodes.some((node) => node.group === 'oidc') && (
        <Button onClick={() => handleSocialLogin('google')}>
          Google로 로그인
        </Button>
      )}

      {/* 이메일/비밀번호 폼 */}
      <Input {...register('identifier')} placeholder="이메일" />
      <Input {...register('password')} type="password" placeholder="비밀번호" />
      <Button type="submit">로그인</Button>
    </form>
  );
}
```

## TypeScript 타입 캐스팅 이슈

### 문제 상황

`@ory/client`의 타입 정의는 런타임 데이터 구조와 완벽하게 일치하지 않는 경우가 있다. VSCode의 TypeScript 검사와 Next.js Turbo 빌드의 strict 타입 체크에서 타입 에러가 발생한다.

### 실제 코드에서의 타입 캐스팅

다음은 ImpRun 프로젝트의 실제 로그인 폼 구현에서 사용된 타입 캐스팅 패턴이다:

```typescript
// features/auth/ui/login-form.tsx
import type { UiNodeInputAttributes } from '@ory/client';

// 1. CSRF 토큰 추출 시 타입 캐스팅
const csrfNode = flow.ui.nodes.find((node) => {
  // node.attributes의 타입이 union type이므로 캐스팅 필요
  const attrs = node.attributes as { name?: string };
  return attrs.name === 'csrf_token';
});
const csrfToken = (csrfNode?.attributes as UiNodeInputAttributes)?.value;

// 2. OIDC 버튼 렌더링 시 - any 캐스팅
{flow.ui.nodes
  .filter((node) => node.group === 'oidc')
  .map((node) => {
    // UiNodeAttributes는 여러 타입의 union이므로 단순화를 위해 any 사용
    const attrs = node.attributes as any;
    return (
      <Button key={attrs.value} onClick={() => handleOidcLogin(attrs.value)}>
        {node.meta.label?.text}
      </Button>
    );
  })}

// 3. 중첩 캐스팅이 필요한 경우
const csrfNode = flow.ui.nodes.find(
  (n) => n.attributes.node_type === 'input' && n.attributes.name === 'csrf_token'
);
// unknown을 거쳐 UiNodeInputAttributes로 캐스팅
const csrfToken = (csrfNode?.attributes as unknown as UiNodeInputAttributes)?.value;
```

### 타입 캐스팅이 필요한 이유

`@ory/client`의 `UiNode.attributes`는 다음과 같은 union type으로 정의되어 있다:

```typescript
type UiNodeAttributes =
  | UiNodeInputAttributes
  | UiNodeTextAttributes
  | UiNodeImageAttributes
  | UiNodeAnchorAttributes
  | UiNodeScriptAttributes;
```

각 노드 타입에 따라 `attributes`의 구조가 다르므로, 특정 속성(예: `name`, `value`)에 접근하려면 적절한 타입으로 캐스팅이 필요하다.

### 권장 패턴

```typescript
// 타입 가드 함수 정의
function isInputNode(node: UiNode): node is UiNode & { attributes: UiNodeInputAttributes } {
  return node.attributes.node_type === 'input';
}

// 사용
const csrfNode = flow.ui.nodes.find(
  (node) => isInputNode(node) && node.attributes.name === 'csrf_token'
);

if (csrfNode && isInputNode(csrfNode)) {
  const csrfToken = csrfNode.attributes.value;  // 타입 안전
}
```

다만 실무에서는 간결함을 위해 `as` 캐스팅을 사용하는 경우가 많다.

## Go 백엔드 통합

### 설계 원칙: Kratos Identity와 백엔드 User 테이블 분리

Kratos는 인증(Authentication)만 담당하고, 애플리케이션 고유의 사용자 데이터는 백엔드 DB에서 별도 관리한다.

```
Kratos Identity (인증 데이터)     백엔드 User 테이블 (비즈니스 데이터)
┌─────────────────────────┐      ┌─────────────────────────────────┐
│ id (UUID)               │──────│ kratos_identity_id (FK)         │
│ email                   │      │ id (UUID, PK)                   │
│ password (hashed)       │      │ email                           │
│ oidc_credentials        │      │ name                            │
│ traits (JSON)           │      │ username                        │
│ metadata_public         │      │ avatar_url                      │
│ created_at              │      │ is_admin                        │
│ updated_at              │      │ subscription_plan               │
└─────────────────────────┘      │ created_projects (relation)     │
                                 │ team_memberships (relation)     │
                                 │ created_at                      │
                                 │ updated_at                      │
                                 └─────────────────────────────────┘
```

이 구조의 장점:
- Kratos는 인증에만 집중, 비즈니스 로직과 분리
- 백엔드 User 테이블에서 애플리케이션 고유 데이터 관리 (구독, 프로젝트, 팀 등)
- `kratos_identity_id`로 연결하여 인증된 사용자와 비즈니스 데이터 매핑

### Kratos Client

```go
// internal/infrastructure/client/kratos_client.go
package client

type KratosClient struct {
    publicURL  string
    adminURL   string
    httpClient *http.Client
}

type KratosSession struct {
    ID       string         `json:"id"`
    Active   bool           `json:"active"`
    Identity KratosIdentity `json:"identity"`
}

type KratosIdentity struct {
    ID             string                 `json:"id"`
    Traits         KratosTraits           `json:"traits"`
    MetadataPublic map[string]interface{} `json:"metadata_public"`
}

type KratosTraits struct {
    Email    string  `json:"email"`
    Name     string  `json:"name,omitempty"`
    Username *string `json:"username,omitempty"`
    Picture  *string `json:"picture,omitempty"`
}

func NewKratosClient(cfg *config.KratosConfig) *KratosClient {
    return &KratosClient{
        publicURL: cfg.PublicURL,
        adminURL:  cfg.AdminURL,
        httpClient: &http.Client{
            Timeout: 10 * time.Second,
        },
    }
}

// WhoAmI validates a session cookie and returns session info
func (c *KratosClient) WhoAmI(ctx context.Context, sessionCookie string) (*KratosSession, error) {
    url := fmt.Sprintf("%s/sessions/whoami", c.publicURL)

    req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    if err != nil {
        return nil, err
    }

    req.Header.Set("Accept", "application/json")
    req.Header.Set("Cookie", fmt.Sprintf("ory_kratos_session=%s", sessionCookie))

    resp, err := c.httpClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    if resp.StatusCode == http.StatusUnauthorized {
        return nil, ErrSessionInvalid
    }

    var session KratosSession
    if err := json.NewDecoder(resp.Body).Decode(&session); err != nil {
        return nil, err
    }

    if !session.Active {
        return nil, ErrSessionInvalid
    }

    return &session, nil
}
```

### Auth Middleware

```go
// internal/interface/middleware/auth.go
package middleware

const SessionCookieName = "ory_kratos_session"

func AuthMiddleware(kratosClient *client.KratosClient, userRepo repository.UserRepository) gin.HandlerFunc {
    return func(c *gin.Context) {
        // 세션 쿠키 확인
        sessionCookie, err := c.Cookie(SessionCookieName)
        if err != nil || sessionCookie == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "No valid authentication provided",
            })
            return
        }

        // Kratos에서 세션 검증
        session, err := kratosClient.WhoAmI(c.Request.Context(), sessionCookie)
        if err != nil {
            if errors.Is(err, client.ErrSessionInvalid) {
                c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                    "error": "Session is invalid or expired",
                })
                return
            }
            c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{
                "error": "Failed to validate session",
            })
            return
        }

        // DB에서 사용자 조회 또는 생성
        user, err := getOrCreateUser(c, userRepo, session)
        if err != nil {
            c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{
                "error": "Failed to process user",
            })
            return
        }

        // Context에 사용자 정보 저장
        userSession := &entity.UserSession{
            ID:        user.ID,
            KratosID:  user.KratosIdentityID,
            Email:     user.Email,
            Name:      user.Name,
            IsAdmin:   user.IsAdmin,
            TokenType: "session",
        }
        c.Set("user", userSession)
        c.Next()
    }
}

// getOrCreateUser: Kratos 세션 기반 User 테이블 자동 동기화
//
// 핵심 로직:
// 1. Kratos Identity ID로 기존 사용자 조회
// 2. 없으면 신규 생성 (Just-In-Time Provisioning)
// 3. 있으면 Kratos에서 변경된 정보 동기화
//
// 이 방식의 장점:
// - 별도의 회원가입 API 없이 Kratos 인증만으로 사용자 생성
// - Kratos에서 이메일/이름 변경 시 자동 동기화
// - 첫 로그인 시점에 자동으로 User 레코드 생성
func getOrCreateUser(c *gin.Context, userRepo repository.UserRepository, session *client.KratosSession) (*entity.User, error) {
    ctx := c.Request.Context()

    // 1. kratos_identity_id로 기존 사용자 조회
    user, err := userRepo.FindByKratosID(ctx, session.Identity.ID)
    if err == nil {
        // 2-A. 기존 사용자: Kratos 정보 변경 시 동기화
        needsUpdate := false
        if user.Email != session.Identity.Traits.Email {
            user.Email = session.Identity.Traits.Email
            needsUpdate = true
        }
        if user.Name != session.Identity.Traits.Name {
            user.Name = session.Identity.Traits.Name
            needsUpdate = true
        }
        if needsUpdate {
            userRepo.Update(ctx, user)
        }
        return user, nil
    }

    // 2-B. 신규 사용자: User 테이블에 레코드 생성
    // Kratos에서 받은 정보를 기반으로 초기 데이터 설정
    user = &entity.User{
        KratosIdentityID: session.Identity.ID,  // Kratos와 연결 키
        Email:            session.Identity.Traits.Email,
        Name:             session.Identity.Traits.Name,
        IsAdmin:          false,  // 기본값
    }

    // 3. MetadataPublic에서 관리자 역할 확인
    // Kratos Admin API로 설정한 metadata_public.roles 참조
    if session.Identity.MetadataPublic != nil {
        if roles, ok := session.Identity.MetadataPublic["roles"].([]interface{}); ok {
            for _, role := range roles {
                if role == "system-admin" {
                    user.IsAdmin = true
                    break
                }
            }
        }
    }

    // 4. DB에 신규 사용자 저장
    if err := userRepo.Create(ctx, user); err != nil {
        return nil, err
    }

    return user, nil
}
```

### 데이터 모델

```go
// internal/domain/entity/user.go
type User struct {
    ID               uuid.UUID `gorm:"type:uuid;primaryKey;default:gen_random_uuid()"`
    KratosIdentityID string    `gorm:"uniqueIndex;not null"`
    Email            string    `gorm:"not null"`
    Name             string
    Username         *string
    AvatarURL        *string
    IsAdmin          bool      `gorm:"default:false"`
    CreatedAt        time.Time
    UpdatedAt        time.Time
}

type UserSession struct {
    ID        uuid.UUID
    KratosID  string
    Email     string
    Name      string
    Username  *string
    AvatarURL *string
    IsAdmin   bool
    TokenType string  // "session" or "token"
}
```

## 인증 아키텍처

### 전체 흐름

```mermaid
sequenceDiagram
    participant Browser
    participant NextJS as Next.js<br/>(Frontend)
    participant Backend as Go Backend<br/>(API Server)
    participant Kratos as Ory Kratos

    Note over Browser,Kratos: 로그인 Flow
    Browser->>NextJS: 1. /login 접속
    NextJS->>Kratos: 2. /.ory/self-service/login/browser
    Kratos-->>NextJS: 3. Flow 생성 및 반환
    NextJS-->>Browser: Login Form 렌더링
    Browser->>NextJS: 이메일/비밀번호 제출
    NextJS->>Kratos: POST /.ory/self-service/login
    Kratos-->>Browser: 4. 세션 쿠키 발급<br/>(ory_kratos_session)

    Note over Browser,Kratos: API 요청 Flow
    Browser->>Backend: 5. API 요청 (with cookie)
    Backend->>Kratos: 6. /sessions/whoami
    Kratos-->>Backend: 7. 세션 검증 및 Identity 반환
    Backend-->>Browser: API 응답
```

### 세션 관리

Kratos는 `ory_kratos_session` 쿠키를 통해 세션을 관리한다.

| 설정 | 값 | 설명 |
|------|-----|------|
| lifespan | 24h | 세션 유효 시간 |
| cookie.name | ory_kratos_session | 쿠키 이름 |
| cookie.same_site | Lax | CSRF 보호 |
| cookie.path | / | 쿠키 경로 |

## Admin API 활용

### Identity 조회

```go
func (c *KratosClient) GetIdentity(ctx context.Context, identityID string) (*KratosIdentity, error) {
    url := fmt.Sprintf("%s/admin/identities/%s?include_credential=oidc", c.adminURL, identityID)

    req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    req.Header.Set("Accept", "application/json")

    resp, err := c.httpClient.Do(req)
    // ...
}
```

### 연결된 소셜 계정 조회

```go
func (c *KratosClient) GetLinkedAccounts(ctx context.Context, identityID string) ([]LinkedAccount, error) {
    identity, err := c.GetIdentity(ctx, identityID)
    if err != nil {
        return nil, err
    }

    var accounts []LinkedAccount
    if oidcCred, ok := identity.Credentials["oidc"]; ok {
        for _, provider := range oidcCred.Config.Providers {
            accounts = append(accounts, LinkedAccount{
                Provider: provider.Provider,
                Subject:  provider.Subject,
            })
        }
    }
    return accounts, nil
}
```

### Identity 삭제 (회원 탈퇴)

```go
func (c *KratosClient) DeleteIdentity(ctx context.Context, identityID string) error {
    url := fmt.Sprintf("%s/admin/identities/%s", c.adminURL, identityID)

    req, _ := http.NewRequestWithContext(ctx, http.MethodDelete, url, nil)
    resp, err := c.httpClient.Do(req)

    if resp.StatusCode != http.StatusNoContent {
        return fmt.Errorf("failed to delete identity")
    }
    return nil
}
```

## 운영 고려사항

### 보안

1. **Admin API 접근 제한**: 내부 네트워크에서만 접근 가능하도록 설정
2. **CSRF 토큰**: 모든 Flow 제출 시 csrf_token 필수
3. **Same-Site Cookie**: Lax 또는 Strict 사용
4. **HTTPS**: 프로덕션에서 필수

### 확장성

Kratos는 stateless하게 설계되어 있어 수평 확장이 가능하다. 세션 데이터는 PostgreSQL에 저장된다.

```yaml
# 멀티 인스턴스 배포
replicas: 3
```

### 모니터링

Kratos는 Prometheus 메트릭을 제공한다.

```yaml
# kratos.yml
serve:
  admin:
    base_url: http://kratos:4434/
```

주요 메트릭:
- `ory_kratos_login_request_duration_seconds`
- `ory_kratos_session_validation_duration_seconds`
- `ory_kratos_identity_creation_total`

## 고급 기능: 2FA (TOTP) 설정 및 강제 적용

### TOTP 활성화

`kratos.yml`에서 TOTP를 활성화한다:

```yaml
selfservice:
  methods:
    totp:
      enabled: true
      config:
        issuer: ImpRun  # 인증 앱에 표시될 서비스 이름
```

### 2FA 강제 적용

TOTP를 설정한 사용자에게 2FA를 강제하려면 `required_aal` 설정을 사용한다:

```yaml
selfservice:
  flows:
    settings:
      ui_url: http://localhost:3000/account/settings/security
      privileged_session_max_age: 15m
      required_aal: highest_available  # 2FA 필수화

session:
  lifespan: 24h
  cookie:
    name: ory_kratos_session
    path: /
    same_site: Lax
  whoami:
    required_aal: highest_available  # API 요청 시 2FA 검증
```

`required_aal: highest_available`의 의미:
- TOTP가 설정된 사용자: AAL2 (2단계 인증) 필수
- TOTP가 없는 사용자: AAL1 (비밀번호만)으로 접근 가능
- 설정 변경 시 2FA 재인증 요구

### 2FA 관련 에러 처리

2FA가 필요한 상황에서 Kratos는 `session_aal2_required` 에러와 함께 `redirect_browser_to` URL을 반환한다:

```typescript
case 'session_aal2_required':
    // 2FA 필요 - Kratos 리다이렉트 URL 사용
    if (redirectTo) {
        window.location.href = redirectTo;
        return;
    }
    break;
```

## 범용 Kratos Flow UI 컴포넌트

### 설계 원칙

Kratos가 반환하는 `flow.ui.nodes`를 동적으로 렌더링하는 범용 컴포넌트를 구현한다. 하드코딩된 폼 필드 대신, Kratos가 제공하는 스키마를 기반으로 UI를 생성한다.

```
features/auth/ui/flow/
├── index.ts
├── kratos-flow.tsx      # Flow 컨테이너 (폼 제출 처리)
├── kratos-node.tsx      # Node 라우터 (타입별 분기)
├── kratos-messages.tsx  # 에러/성공 메시지 표시
├── node-input.tsx       # Input 노드 (text, password, email 등)
├── node-button.tsx      # Button/Submit 노드
├── node-checkbox.tsx    # Checkbox 노드
├── node-hidden.tsx      # Hidden input (csrf_token 등)
├── node-text.tsx        # Text 노드 (복구 코드 표시 등)
├── node-image.tsx       # Image 노드 (QR 코드 등)
└── types.ts             # 타입 정의
```

### 핵심 타입 정의

```typescript
// types.ts
import { LoginFlow, RegistrationFlow, SettingsFlow, RecoveryFlow, VerificationFlow } from '@ory/client';

export type Flow =
    | LoginFlow
    | RegistrationFlow
    | SettingsFlow
    | RecoveryFlow
    | VerificationFlow;

export type Methods =
    | 'password'
    | 'profile'
    | 'oidc'
    | 'totp'
    | 'webauthn'
    | 'passkey'
    | 'lookup_secret'
    | 'link'
    | 'code';

export interface KratosFlowProps<T extends FlowValues = FlowValues> {
    flow?: Flow;
    only?: Methods;
    onSubmit: (values: T) => Promise<void>;
    hideGlobalMessages?: boolean;
}
```

### KratosFlow 컴포넌트

폼 제출과 상태 관리를 담당하는 컨테이너 컴포넌트:

```typescript
// kratos-flow.tsx
'use client';

import { UiNode } from '@ory/client';
import { getNodeId, isUiNodeInputAttributes } from '@ory/integrations/ui';
import { FormEvent, useCallback, useEffect, useState } from 'react';

import { KratosMessages } from './kratos-messages';
import { KratosNode } from './kratos-node';
import { Flow, KratosFlowProps, Methods } from './types';

// 그룹별 노드 필터링
function filterNodes(flow: Flow | undefined, only?: Methods): UiNode[] {
    if (!flow) return [];

    return flow.ui.nodes.filter(({ group }) => {
        if (!only) return true;
        return group === 'default' || group === only;
    });
}

// 노드에서 초기값 추출
function initializeValues(nodes: UiNode[]): Record<string, unknown> {
    const values: Record<string, unknown> = {};

    nodes.forEach((node) => {
        if (isUiNodeInputAttributes(node.attributes)) {
            // 버튼/제출 값은 클릭 시에만 포함
            if (node.attributes.type === 'button' || node.attributes.type === 'submit') {
                return;
            }
            values[node.attributes.name] = node.attributes.value;
        }
    });

    return values;
}

export function KratosFlow<T extends FlowValues>({
    flow,
    only,
    onSubmit,
    hideGlobalMessages,
}: KratosFlowProps<T>) {
    const [values, setValues] = useState<Record<string, unknown>>({});
    const [isLoading, setIsLoading] = useState(false);

    const nodes = filterNodes(flow, only);

    // Flow 변경 시 값 재초기화
    useEffect(() => {
        setValues(initializeValues(nodes));
    }, [flow]);

    const handleSubmit = useCallback(
        async (event: FormEvent<HTMLFormElement>) => {
            event.preventDefault();
            if (isLoading) return;

            const form = event.currentTarget;
            const formData = new FormData(form);
            let body = Object.fromEntries(formData) as T;

            // 여러 제출 버튼 중 어떤 것이 클릭되었는지 확인
            const nativeEvent = event.nativeEvent as SubmitEvent;
            if (nativeEvent.submitter instanceof HTMLButtonElement) {
                const method = nativeEvent.submitter;
                if (method.name && method.value) {
                    body = { ...body, [method.name]: method.value };
                }
            }

            setIsLoading(true);
            try {
                await onSubmit({ ...body, ...values } as T);
            } finally {
                setIsLoading(false);
            }
        },
        [isLoading, onSubmit, values]
    );

    if (!flow) return null;

    return (
        <form
            action={flow.ui.action}
            method={flow.ui.method}
            onSubmit={handleSubmit}
            className="space-y-4"
        >
            {!hideGlobalMessages && <KratosMessages messages={flow.ui.messages} />}

            {nodes.map((node, k) => {
                const id = getNodeId(node);
                return (
                    <KratosNode
                        key={`${id}-${k}`}
                        disabled={isLoading}
                        node={node}
                        value={values[id]}
                        setValue={(value) => setValues((prev) => ({ ...prev, [id]: value }))}
                    />
                );
            })}
        </form>
    );
}
```

### 보안 설정 페이지에서의 활용

하나의 Settings Flow로 모든 보안 설정을 관리한다:

```tsx
// app/account/settings/security/page.tsx
'use client';

export default function SecurityPage() {
    const { data: flow, isLoading } = useSettingsFlow();
    const mutation = useSettingsMutation();

    const onSubmit = async (values: UpdateSettingsFlowBody) => {
        if (!flow?.id) return;
        await mutation.mutateAsync({ flowId: flow.id, body: values });
    };

    return (
        <PageContainer title="보안" description="비밀번호와 연결된 계정을 관리합니다">
            <div className="max-w-2xl space-y-6">
                {/* 비밀번호 변경 */}
                <Card>
                    <CardHeader>
                        <CardTitle>비밀번호 변경</CardTitle>
                    </CardHeader>
                    <CardContent>
                        <KratosFlow flow={flow} only="password" onSubmit={onSubmit} />
                    </CardContent>
                </Card>

                {/* 소셜 계정 연결 */}
                <Card>
                    <CardHeader>
                        <CardTitle>연결된 계정</CardTitle>
                    </CardHeader>
                    <CardContent>
                        <KratosFlow flow={flow} only="oidc" onSubmit={onSubmit} />
                    </CardContent>
                </Card>

                {/* TOTP 2FA */}
                <Card>
                    <CardHeader>
                        <CardTitle>인증 앱 (TOTP)</CardTitle>
                    </CardHeader>
                    <CardContent>
                        <KratosFlow flow={flow} only="totp" onSubmit={onSubmit} />
                    </CardContent>
                </Card>

                {/* 복구 코드 */}
                <Card>
                    <CardHeader>
                        <CardTitle>복구 코드</CardTitle>
                    </CardHeader>
                    <CardContent>
                        <KratosFlow flow={flow} only="lookup_secret" onSubmit={onSubmit} />
                    </CardContent>
                </Card>
            </div>
        </PageContainer>
    );
}
```

## 중앙화된 한국어 i18n 시스템

### 설계 배경

Kratos는 영어 메시지를 반환하므로, 프론트엔드에서 한국어로 번역해야 한다. `@ory/elements`의 한국어 번역을 참고하여 중앙화된 i18n 시스템을 구현한다.

### 번역 구조

```typescript
// features/auth/lib/kratos-i18n.ts

// 1. ID 기반 번역 (Kratos UiText.id 사용)
const messageTranslations: Record<number, string> = {
    // 로그인 메시지 (1010xxx)
    1010001: '로그인',
    1010002: '{provider}로 로그인',

    // 설정 메시지 (1050xxx)
    1050001: '변경 사항이 저장되었습니다!',
    1050002: '{provider} 연결',
    1050003: '{provider} 연결 해제',
    1050004: 'TOTP 인증 앱 연결 해제',

    // 에러 메시지 (4000xxx)
    4000006: '자격 증명이 올바르지 않습니다. 비밀번호, 이메일을 확인하세요.',
    4000007: '동일한 이메일로 등록된 계정이 이미 존재합니다.',
    // ...
};

// 2. 텍스트 기반 번역 (폴백)
const textTranslations: Record<string, string> = {
    'Your changes have been saved!': '변경 사항이 저장되었습니다!',
    'The provided credentials are invalid.': '제공된 자격 증명이 올바르지 않습니다.',
    // ...
};

// 3. 라벨 번역
const labelTranslations: Record<string, string> = {
    'Password': '비밀번호',
    'New password': '새 비밀번호',
    'Save': '저장',
    'Link totp': 'TOTP 등록',
    'Unlink totp': 'TOTP 해제',
    // ...
};
```

### 번역 함수

```typescript
/**
 * Kratos 메시지 ID로 번역
 * 플레이스홀더 치환 지원 ({provider}, {secret} 등)
 */
export function translateMessageById(
    id: number,
    context?: Record<string, unknown>
): string | null {
    const template = messageTranslations[id];
    if (!template) return null;

    if (context) {
        return template.replace(/\{(\w+)\}/g, (_, key) => {
            return String(context[key] ?? `{${key}}`);
        });
    }

    return template;
}

/**
 * 라벨 텍스트 번역
 * OIDC 동적 라벨 처리 포함 (Link google → Google 연결)
 */
export function translateLabel(label: string): string {
    if (labelTranslations[label]) {
        return labelTranslations[label];
    }

    // OIDC 동적 라벨 처리
    const linkMatch = label.match(/^(Link|Unlink)\s+(\w+)$/i);
    if (linkMatch) {
        const [, action, provider] = linkMatch;
        const providerName = provider.charAt(0).toUpperCase() + provider.slice(1);
        return action.toLowerCase() === 'link'
            ? `${providerName} 연결`
            : `${providerName} 연결 해제`;
    }

    return label;
}
```

### 컴포넌트에서 활용

```typescript
// node-button.tsx
import { translateLabel, getProviderDisplayName } from '../../lib/kratos-i18n';

function getButtonLabel(node: UiNode, attrs: UiNodeInputAttributes): string {
    // OIDC 버튼: 아이콘 + 프로바이더명
    if (node.group === 'oidc' && attrs.value) {
        const providerName = getProviderDisplayName(attrs.value);
        const isUnlink = attrs.name === 'unlink';
        return isUnlink ? `${providerName} 연결 해제` : `${providerName} 연결`;
    }

    // 일반 버튼: 라벨 번역
    const label = node.meta.label?.text || attrs.value || 'Submit';
    return translateLabel(label);
}
```

## 에러 핸들링 패턴

### 핵심 원칙: redirect_browser_to 사용

Kratos 공식 샘플의 핵심 패턴은 에러 응답의 `redirect_browser_to` URL을 그대로 사용하는 것이다. URL을 직접 생성하지 않는다.

```typescript
// features/auth/lib/handle-flow-error.ts

export function handleFlowError<S>(
    router: AppRouterInstance,
    flowType: FlowType,
    resetFlow: (flow: S | undefined) => void
) {
    return async (err: AxiosError<KratosErrorResponse>): Promise<void> => {
        const errorId = err.response?.data?.error?.id;
        const redirectTo = err.response?.data?.redirect_browser_to;
        const status = err.response?.status;

        switch (errorId) {
            case 'session_refresh_required':
                // 권한 있는 작업에 재인증 필요
                // CRITICAL: redirect_browser_to 직접 사용
                if (redirectTo) {
                    window.location.href = redirectTo;
                    return;
                }
                break;

            case 'session_aal2_required':
                // 2FA 필요
                if (redirectTo) {
                    window.location.href = redirectTo;
                    return;
                }
                break;

            case 'self_service_flow_expired':
                // Flow 만료 - 새 Flow 생성
                toast.error('세션이 만료되었습니다.');
                resetFlow(undefined);
                window.location.href = getFlowPath(flowType);
                return;
        }

        // HTTP 상태 코드 처리
        switch (status) {
            case 422:
                // 브라우저 리다이렉트 필요 (OIDC, 2FA 등)
                if (redirectTo) {
                    window.location.href = redirectTo;
                    return;
                }
                break;

            case 403:
                // 세션 검증 실패
                if (redirectTo) {
                    window.location.href = redirectTo;
                    return;
                }
                break;
        }

        // 처리되지 않은 에러는 호출자에게 전달
        return Promise.reject(err);
    };
}
```

### 400 에러 처리 (검증 에러)

400 에러는 폼 검증 실패를 의미한다. Flow를 업데이트하여 에러 메시지를 표시한다:

```typescript
export function handleSubmitError<S>(setFlow: (flow: S) => void) {
    return async (err: AxiosError<S>) => {
        if (err.response?.status === 400 && err.response.data) {
            // 검증 에러 - Flow 업데이트로 에러 표시
            setFlow(err.response.data);
            return;
        }
        return Promise.reject(err);
    };
}
```

### 사용 예시

```typescript
// Settings 페이지에서의 에러 처리
const onSubmit = async (values: UpdateSettingsFlowBody) => {
    if (!flow?.id) return;

    try {
        await mutation.mutateAsync({ flowId: flow.id, body: values });
        await refetch();
    } catch (err) {
        // Flow 에러 처리 (세션, 리다이렉트 등)
        try {
            await handleFlowError(router, 'settings', () => {})(err as never);
        } catch (flowErr) {
            // 검증 에러 처리
            await handleSubmitError(() => refetch())(flowErr as never);
        }
    }
};
```

## 참고 자료

### 공식 문서
- [Ory Kratos Documentation](https://www.ory.sh/docs/kratos)
- [Ory Kratos GitHub](https://github.com/ory/kratos)
- [Identity Schema Reference](https://www.ory.sh/docs/kratos/reference/configuration)

### 관련 문서
- [Ory Hydra OAuth2/OIDC 구현 가이드](https://blog.imprun.dev/76) - OAuth2 토큰 발급 및 클라이언트 관리
- [Ory Keto를 활용한 ReBAC 기반 권한 관리 시스템 구축](https://blog.imprun.dev/78) - 권한 모델 및 Relation Tuple
- [Ory Oathkeeper를 활용한 Zero Trust IAP 구현 가이드](https://blog.imprun.dev/79) - Zero Trust Proxy 설정
- [ImpRun 인증/인가 아키텍처: Ory 스택 통합 구현 가이드](https://blog.imprun.dev/80) - 4개 컴포넌트 통합 아키텍처

### 클라이언트 라이브러리
- [@ory/client (npm)](https://www.npmjs.com/package/@ory/client)
- [Ory Go SDK](https://github.com/ory/client-go)
