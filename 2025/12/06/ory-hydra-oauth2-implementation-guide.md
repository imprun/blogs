# Ory Hydra를 활용한 OAuth2/OIDC 서버 구축 가이드: 실제 프로젝트 적용 사례

> **작성일**: 2025년 12월 6일
> **카테고리**: Authentication, OAuth2, Ory Stack
> **키워드**: Ory Hydra, OAuth2, OIDC, Kratos, Keto, Oathkeeper, API Gateway

## 요약

API Gateway 플랫폼에서 Ory Hydra를 OAuth2/OIDC 서버로 활용한 실제 구현 사례를 공유합니다. Hydra는 Ory 스택(Kratos, Keto, Oathkeeper)과 함께 사용되어, 사용자 인증은 Kratos가, 권한 관리는 Keto가, API 보호는 Oathkeeper가 담당하는 구조입니다. 이 글에서는 Docker 기반 배포, OAuth2 플로우 구현, 다른 Ory 서비스와의 통합 방법을 다룹니다.

---

## 배경

### Ory 스택 소개

Ory는 클라우드 네이티브 인증/인가 오픈소스 생태계입니다:

| 서비스 | 역할 | 포트 |
|--------|------|------|
| **Hydra** | OAuth2/OIDC 서버 | 4444 (public), 4445 (admin) |
| **Kratos** | 사용자 인증 (Identity) | 4433 (public), 4434 (admin) |
| **Keto** | 권한 관리 (Authorization) | 4466 (read), 4467 (write) |
| **Oathkeeper** | API 프록시 (Policy Enforcement) | 4455 (proxy), 4456 (api) |

### Hydra의 역할

Hydra는 **OAuth2/OIDC 프로토콜 서버**입니다. 중요한 점은 Hydra가 사용자 인증을 직접 처리하지 않는다는 것입니다. Login/Consent UI는 별도로 구현해야 하며, Hydra는 토큰 발급과 관리만 담당합니다.

```mermaid
graph LR
    Client[클라이언트] --> Hydra[Hydra]
    Hydra --> LoginUI[Login UI]
    LoginUI --> Kratos[Kratos]
    Kratos --> LoginUI
    LoginUI --> Hydra
    Hydra --> Client

    style Hydra stroke:#2563eb,stroke-width:3px
    style Kratos stroke:#16a34a,stroke-width:3px
    style LoginUI stroke:#ea580c,stroke-width:3px
```

---

## 아키텍처

### 전체 구성

```mermaid
graph TB
    Browser[브라우저] --> Envoy[Envoy Gateway :8080]
    Envoy --> Oathkeeper[Oathkeeper :4455]
    Oathkeeper --> |토큰 검증| Hydra[Hydra :4445]
    Oathkeeper --> API[Go API]
    API --> |권한 확인| Keto[Keto :4466]
    API --> |세션 확인| Kratos[Kratos :4433]
    API --> DB[(PostgreSQL)]

    style Hydra stroke:#2563eb,stroke-width:3px
    style Kratos stroke:#16a34a,stroke-width:3px
    style Keto stroke:#ea580c,stroke-width:3px
    style Oathkeeper stroke:#dc2626,stroke-width:3px
```

### 서비스 역할 분담

1. **Envoy**: 외부 요청의 진입점
2. **Oathkeeper**: Bearer 토큰 검증, 세션 확인
3. **Hydra**: OAuth2 토큰 발급 및 introspection
4. **Kratos**: 사용자 로그인/회원가입 처리
5. **Keto**: 리소스별 권한(RBAC) 관리

---

## Docker Compose 설정

### Hydra 서비스 정의

```yaml
services:
  hydra-migrate:
    image: oryd/hydra:v2.2.0
    environment:
      - DSN=postgres://user:password@postgres:5432/hydra?sslmode=disable
    command: migrate sql -e --yes
    depends_on:
      postgres:
        condition: service_healthy

  hydra:
    image: oryd/hydra:v2.2.0
    container_name: imprun-hydra
    ports:
      - "4444:4444"  # Public API (OAuth2 endpoints)
      - "4445:4445"  # Admin API (토큰 관리, introspection)
    environment:
      - DSN=postgres://user:password@postgres:5432/hydra?sslmode=disable
      - SECRETS_SYSTEM=${HYDRA_SECRET_SYSTEM}
      - SECRETS_COOKIE=${HYDRA_SECRET_COOKIE}
      - URLS_SELF_ISSUER=http://localhost:4444
      - URLS_CONSENT=${APP_URL}/consent
      - URLS_LOGIN=${APP_URL}/login
      - URLS_LOGOUT=${APP_URL}/logout
      - URLS_ERROR=${APP_URL}/error
      - URLS_POST_LOGOUT_REDIRECT=${APP_URL}
      - STRATEGIES_ACCESS_TOKEN=jwt
      - TTL_ACCESS_TOKEN=1h
      - TTL_REFRESH_TOKEN=720h
      - OAUTH2_EXPOSE_INTERNAL_ERRORS=true
    command: serve all --dev
    depends_on:
      hydra-migrate:
        condition: service_completed_successfully
```

### 주요 환경 변수 설명

| 변수 | 설명 |
|------|------|
| `DSN` | PostgreSQL 연결 문자열 |
| `SECRETS_SYSTEM` | 토큰 암호화 키 (32바이트 이상) |
| `SECRETS_COOKIE` | 쿠키 암호화 키 |
| `URLS_SELF_ISSUER` | JWT issuer URL |
| `URLS_LOGIN` | 로그인 UI 리다이렉트 URL |
| `URLS_CONSENT` | 동의 화면 URL |
| `STRATEGIES_ACCESS_TOKEN` | 토큰 형식 (jwt 또는 opaque) |
| `TTL_ACCESS_TOKEN` | 액세스 토큰 유효기간 |
| `TTL_REFRESH_TOKEN` | 리프레시 토큰 유효기간 |

### Secret 생성 방법

```bash
# 32바이트 랜덤 시크릿 생성
openssl rand -hex 32

# .env 파일에 추가
HYDRA_SECRET_SYSTEM=your-generated-secret-here
HYDRA_SECRET_COOKIE=your-generated-cookie-secret-here
```

---

## Oathkeeper와 통합

### Bearer 토큰 Introspection 설정

Oathkeeper는 API 요청의 Bearer 토큰을 Hydra를 통해 검증합니다.

**oathkeeper.yml**:

```yaml
authenticators:
  bearer_token:
    enabled: true
    config:
      check_session_url: http://hydra:4445/oauth2/introspect
      preserve_path: true
      extra_from: "@this"
      subject_from: "sub"
      token_from:
        header: Authorization

  cookie_session:
    enabled: true
    config:
      check_session_url: http://kratos:4433/sessions/whoami
      preserve_path: true
      extra_from: "@this"
      subject_from: "identity.id"

authorizers:
  allow:
    enabled: true

mutators:
  header:
    enabled: true
    config:
      headers:
        X-User-ID: "{{ print .Subject }}"
```

### Access Rule 정의

**access-rules.yml**:

```yaml
- id: api-protected
  upstream:
    url: http://api:8080
  match:
    url: <http|https>://<.*>/api/v1/<.*>
    methods:
      - GET
      - POST
      - PUT
      - DELETE
  authenticators:
    - handler: bearer_token
    - handler: cookie_session
  authorizer:
    handler: allow
  mutators:
    - handler: header
```

---

## Go API에서 토큰 검증

### Hydra Client 설정

```go
// pkg/config/config.go
type Config struct {
    Hydra struct {
        PublicURL string `env:"HYDRA_PUBLIC_URL" envDefault:"http://localhost:4444"`
        AdminURL  string `env:"HYDRA_ADMIN_URL" envDefault:"http://localhost:4445"`
    }
}
```

### Token Introspection 구현

```go
// internal/infrastructure/client/hydra_client.go
package client

import (
    "context"
    "net/http"
    "net/url"
    "strings"

    hydra "github.com/ory/hydra-client-go/v2"
)

type HydraClient struct {
    admin *hydra.APIClient
}

func NewHydraClient(adminURL string) *HydraClient {
    config := hydra.NewConfiguration()
    config.Servers = []hydra.ServerConfiguration{
        {URL: adminURL},
    }

    return &HydraClient{
        admin: hydra.NewAPIClient(config),
    }
}

func (c *HydraClient) IntrospectToken(ctx context.Context, token string) (*hydra.IntrospectedOAuth2Token, error) {
    result, _, err := c.admin.OAuth2API.
        IntrospectOAuth2Token(ctx).
        Token(token).
        Execute()

    if err != nil {
        return nil, err
    }

    return result, nil
}
```

### 인증 미들웨어

```go
// internal/interface/middleware/auth.go
package middleware

import (
    "strings"

    "github.com/gin-gonic/gin"
)

func AuthMiddleware(hydraClient *client.HydraClient, kratosClient *client.KratosClient) gin.HandlerFunc {
    return func(c *gin.Context) {
        // 1. Bearer 토큰 확인
        authHeader := c.GetHeader("Authorization")
        if strings.HasPrefix(authHeader, "Bearer ") {
            token := strings.TrimPrefix(authHeader, "Bearer ")
            result, err := hydraClient.IntrospectToken(c.Request.Context(), token)
            if err == nil && result.GetActive() {
                c.Set("user_id", result.GetSub())
                c.Set("token_type", "bearer")
                c.Next()
                return
            }
        }

        // 2. 세션 쿠키 확인 (Kratos)
        cookie, err := c.Cookie("ory_kratos_session")
        if err == nil && cookie != "" {
            session, err := kratosClient.WhoAmI(c.Request.Context(), cookie)
            if err == nil && session.GetActive() {
                c.Set("user_id", session.Identity.GetId())
                c.Set("token_type", "session")
                c.Next()
                return
            }
        }

        c.AbortWithStatusJSON(401, gin.H{"error": "unauthorized"})
    }
}
```

---

## OAuth2 Client 등록

### CLI를 통한 클라이언트 등록

```bash
# OAuth2 클라이언트 생성
docker exec imprun-hydra hydra create oauth2-client \
    --endpoint http://localhost:4445 \
    --name "API Gateway Client" \
    --grant-type client_credentials \
    --grant-type authorization_code \
    --grant-type refresh_token \
    --response-type code \
    --response-type token \
    --scope openid \
    --scope offline_access \
    --scope profile \
    --redirect-uri http://localhost:3000/callback \
    --token-endpoint-auth-method client_secret_post

# 결과 예시
# CLIENT ID: abc123...
# CLIENT SECRET: xyz789...
```

### Admin API를 통한 클라이언트 등록

```go
func (c *HydraClient) CreateOAuth2Client(ctx context.Context, name string, redirectURIs []string) (*hydra.OAuth2Client, error) {
    client := hydra.NewOAuth2Client()
    client.SetClientName(name)
    client.SetGrantTypes([]string{"authorization_code", "refresh_token", "client_credentials"})
    client.SetResponseTypes([]string{"code", "token"})
    client.SetScope("openid offline_access profile")
    client.SetRedirectUris(redirectURIs)
    client.SetTokenEndpointAuthMethod("client_secret_post")

    result, _, err := c.admin.OAuth2API.
        CreateOAuth2Client(ctx).
        OAuth2Client(*client).
        Execute()

    return result, err
}
```

---

## OAuth2 플로우 구현

### Authorization Code Flow

```mermaid
sequenceDiagram
    participant Client as 클라이언트
    participant Hydra as Hydra
    participant Login as Login UI
    participant Kratos as Kratos
    participant Consent as Consent UI

    Client->>Hydra: GET /oauth2/auth
    Hydra->>Login: 리다이렉트 (login_challenge)
    Login->>Kratos: 사용자 인증
    Kratos->>Login: 세션 생성
    Login->>Hydra: PUT /oauth2/auth/requests/login/accept
    Hydra->>Consent: 리다이렉트 (consent_challenge)
    Consent->>Hydra: PUT /oauth2/auth/requests/consent/accept
    Hydra->>Client: 리다이렉트 (code)
    Client->>Hydra: POST /oauth2/token (code)
    Hydra->>Client: access_token, refresh_token
```

### Login Challenge 처리

```go
// Login UI 백엔드
func HandleLogin(c *gin.Context) {
    challenge := c.Query("login_challenge")

    // 1. Challenge 정보 조회
    loginRequest, _, err := hydraClient.OAuth2API.
        GetOAuth2LoginRequest(c.Request.Context()).
        LoginChallenge(challenge).
        Execute()

    if err != nil {
        c.AbortWithError(500, err)
        return
    }

    // 2. 이미 인증된 사용자인 경우 스킵
    if loginRequest.GetSkip() {
        acceptRequest := hydra.NewAcceptOAuth2LoginRequest(loginRequest.GetSubject())
        result, _, _ := hydraClient.OAuth2API.
            AcceptOAuth2LoginRequest(c.Request.Context()).
            LoginChallenge(challenge).
            AcceptOAuth2LoginRequest(*acceptRequest).
            Execute()

        c.Redirect(302, result.GetRedirectTo())
        return
    }

    // 3. 로그인 폼 표시
    c.HTML(200, "login.html", gin.H{
        "challenge": challenge,
    })
}

func HandleLoginSubmit(c *gin.Context) {
    challenge := c.PostForm("challenge")
    email := c.PostForm("email")
    password := c.PostForm("password")

    // Kratos로 인증
    session, err := kratosClient.Login(c.Request.Context(), email, password)
    if err != nil {
        c.HTML(400, "login.html", gin.H{"error": "Invalid credentials"})
        return
    }

    // Hydra에 로그인 승인
    acceptRequest := hydra.NewAcceptOAuth2LoginRequest(session.Identity.GetId())
    acceptRequest.SetRemember(true)
    acceptRequest.SetRememberFor(3600)

    result, _, _ := hydraClient.OAuth2API.
        AcceptOAuth2LoginRequest(c.Request.Context()).
        LoginChallenge(challenge).
        AcceptOAuth2LoginRequest(*acceptRequest).
        Execute()

    c.Redirect(302, result.GetRedirectTo())
}
```

### Consent Challenge 처리

```go
func HandleConsent(c *gin.Context) {
    challenge := c.Query("consent_challenge")

    // 1. Consent 요청 정보 조회
    consentRequest, _, err := hydraClient.OAuth2API.
        GetOAuth2ConsentRequest(c.Request.Context()).
        ConsentChallenge(challenge).
        Execute()

    if err != nil {
        c.AbortWithError(500, err)
        return
    }

    // 2. 이미 동의한 경우 또는 자사 앱인 경우 스킵
    if consentRequest.GetSkip() || isTrustedClient(consentRequest.GetClient().GetClientId()) {
        acceptRequest := hydra.NewAcceptOAuth2ConsentRequest()
        acceptRequest.SetGrantScope(consentRequest.GetRequestedScope())
        acceptRequest.SetGrantAccessTokenAudience(consentRequest.GetRequestedAccessTokenAudience())

        result, _, _ := hydraClient.OAuth2API.
            AcceptOAuth2ConsentRequest(c.Request.Context()).
            ConsentChallenge(challenge).
            AcceptOAuth2ConsentRequest(*acceptRequest).
            Execute()

        c.Redirect(302, result.GetRedirectTo())
        return
    }

    // 3. 동의 화면 표시
    c.HTML(200, "consent.html", gin.H{
        "challenge":       challenge,
        "requestedScopes": consentRequest.GetRequestedScope(),
        "clientName":      consentRequest.GetClient().GetClientName(),
    })
}
```

---

## Client Credentials Flow

서버 간 통신에 사용되는 플로우입니다.

```go
func GetClientCredentialsToken(clientID, clientSecret string) (string, error) {
    data := url.Values{}
    data.Set("grant_type", "client_credentials")
    data.Set("client_id", clientID)
    data.Set("client_secret", clientSecret)
    data.Set("scope", "openid")

    resp, err := http.PostForm("http://localhost:4444/oauth2/token", data)
    if err != nil {
        return "", err
    }
    defer resp.Body.Close()

    var result struct {
        AccessToken string `json:"access_token"`
        TokenType   string `json:"token_type"`
        ExpiresIn   int    `json:"expires_in"`
    }

    json.NewDecoder(resp.Body).Decode(&result)
    return result.AccessToken, nil
}
```

---

## Keto와 연동한 권한 관리

Hydra로 인증된 사용자의 권한은 Keto로 관리합니다.

### Permission Check

```go
func (m *AuthMiddleware) RequireOrgPermission(permission string) gin.HandlerFunc {
    return func(c *gin.Context) {
        userID := c.GetString("user_id")
        orgID := c.Param("orgId")

        // Keto에서 권한 확인
        hasPermission, err := m.ketoClient.Check(c.Request.Context(), &keto.CheckRequest{
            Namespace: "Organization",
            Object:    orgID,
            Relation:  permission,
            SubjectId: &userID,
        })

        if err != nil || !hasPermission {
            c.AbortWithStatusJSON(403, gin.H{"error": "forbidden"})
            return
        }

        c.Next()
    }
}
```

### 라우터 적용

```go
func SetupRoutes(r *gin.Engine, authMiddleware *middleware.AuthMiddleware) {
    api := r.Group("/api/v1")
    api.Use(authMiddleware.Authenticate())

    orgs := api.Group("/organizations/:orgId")
    {
        orgs.GET("", authMiddleware.RequireOrgPermission("view"), handlers.GetOrganization)
        orgs.PUT("", authMiddleware.RequireOrgPermission("manage"), handlers.UpdateOrganization)
        orgs.DELETE("", authMiddleware.RequireOrgPermission("owner"), handlers.DeleteOrganization)
    }
}
```

---

## 트러블슈팅

### 일반적인 문제와 해결책

| 문제 | 원인 | 해결책 |
|------|------|--------|
| `invalid_client` | 클라이언트 ID/Secret 불일치 | 클라이언트 재등록 또는 확인 |
| `consent_required` | Consent 플로우 미구현 | `/consent` 엔드포인트 구현 |
| Token introspection 실패 | Admin API URL 오류 | 4445 포트 사용 확인 |
| JWT 서명 검증 실패 | JWK 미동기화 | `/.well-known/jwks.json` 확인 |

### 로그 확인

```bash
# Hydra 로그 확인
docker logs imprun-hydra -f

# 특정 요청 디버깅
docker logs imprun-hydra 2>&1 | grep "oauth2"
```

### Health Check

```bash
# Public API
curl http://localhost:4444/health/ready

# Admin API
curl http://localhost:4445/health/ready

# OIDC Discovery
curl http://localhost:4444/.well-known/openid-configuration
```

---

## 교훈

### 1. Hydra는 인증 서버가 아니다

Hydra는 OAuth2/OIDC **프로토콜 서버**입니다. 사용자 인증은 별도 시스템(Kratos 등)이 담당합니다. Login/Consent UI를 직접 구현해야 한다는 점을 처음부터 인지해야 합니다.

### 2. Ory 스택의 역할 분리

각 서비스의 역할이 명확히 분리되어 있습니다:
- Hydra: 토큰 발급/관리
- Kratos: 사용자 인증
- Keto: 권한 관리
- Oathkeeper: API 보호

이 분리를 이해하면 전체 아키텍처 설계가 명확해집니다.

### 3. 개발 모드와 프로덕션 차이

`--dev` 플래그는 개발 편의를 위한 것입니다. 프로덕션에서는:
- HTTPS 필수
- 시크릿 키 안전한 관리
- CORS 설정 제한
- Rate limiting 적용

---

## 참고 자료

### 공식 문서
- [Ory Hydra Documentation](https://www.ory.sh/docs/hydra)
- [Ory Kratos Documentation](https://www.ory.sh/docs/kratos)
- [Ory Keto Documentation](https://www.ory.sh/docs/keto)
- [Ory Oathkeeper Documentation](https://www.ory.sh/docs/oathkeeper)

### 관련 문서
- [Ory Kratos를 활용한 사용자 인증 시스템 구축](https://blog.imprun.dev/77) - Identity Management 상세 구현
- [Ory Keto를 활용한 ReBAC 기반 권한 관리 시스템 구축](https://blog.imprun.dev/78) - 권한 모델 및 Relation Tuple
- [Ory Oathkeeper를 활용한 Zero Trust IAP 구현 가이드](https://blog.imprun.dev/79) - Zero Trust Proxy 설정
- [ImpRun 인증/인가 아키텍처: Ory 스택 통합 구현 가이드](https://blog.imprun.dev/80) - 4개 컴포넌트 통합 아키텍처
