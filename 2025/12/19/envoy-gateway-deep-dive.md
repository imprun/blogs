# Envoy Gateway 분석: Gateway API 확장의 실체

> **작성일**: 2025년 12월 19일
> **카테고리**: Kubernetes, API Gateway, Envoy
> **키워드**: Envoy Gateway, Gateway API, Kubernetes, Policy Attachment, xDS

## 요약

Envoy Gateway는 Envoy Proxy를 Kubernetes Gateway API로 제어하는 구현체다. 표준 Gateway API만으로는 부족한 기능(Rate Limiting, 인증, Circuit Breaker 등)을 Policy Attachment 모델로 확장한다. 이 글에서는 Envoy Gateway의 아키텍처와 5가지 핵심 정책, 그리고 확장 메커니즘을 분석한다.

## Envoy Gateway란

Envoy Proxy는 강력하지만 직접 설정하기 복잡하다. xDS 설정을 YAML로 작성하면 수백 줄이 되기 쉽다.

Envoy Gateway는 이 문제를 해결한다:

| 구분 | 직접 Envoy 설정 | Envoy Gateway |
|------|---------------|---------------|
| 설정 방식 | xDS YAML 직접 작성 | Gateway API 리소스 |
| 복잡도 | 높음 | 낮음 |
| Kubernetes 통합 | 별도 구현 필요 | 네이티브 |
| 표준화 | 없음 | Gateway API 표준 |

Envoy Gateway는 Gateway, HTTPRoute 같은 표준 리소스를 xDS 설정으로 변환하여 Envoy Proxy에 전달한다.

## Gateway API만으로는 부족한 이유

Gateway API는 라우팅의 기본만 정의한다:

- Host/Path 기반 라우팅
- Header 매칭
- Traffic Splitting
- TLS Termination

하지만 실제 운영에는 더 많은 기능이 필요하다:

| 필요 기능 | Gateway API 표준 | Envoy Gateway 확장 |
|----------|-----------------|-------------------|
| Rate Limiting | 없음 | BackendTrafficPolicy |
| JWT 인증 | 없음 | SecurityPolicy |
| Circuit Breaker | 없음 | BackendTrafficPolicy |
| CORS | 없음 | SecurityPolicy |
| IP 필터링 | 없음 | SecurityPolicy |
| 커스텀 로직 | 없음 | EnvoyExtensionPolicy |

Envoy Gateway는 이러한 기능을 **Policy Attachment** 모델로 제공한다.

## Policy Attachment 모델

Gateway API의 핵심 설계 원칙 중 하나다. 표준 리소스(Gateway, HTTPRoute)를 수정하지 않고 별도의 Policy 리소스를 "부착"하여 기능을 확장한다.

```
┌─────────────────────────────────────────────────────────┐
│                      Gateway                            │
│  ┌─────────────────────────────────────────────────┐    │
│  │ ClientTrafficPolicy (클라이언트 → Gateway)       │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                     HTTPRoute                           │
│  ┌─────────────────────────────────────────────────┐    │
│  │ SecurityPolicy (인증/인가)                       │    │
│  │ BackendTrafficPolicy (Gateway → Backend)        │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────┐
│                      Backend                            │
└─────────────────────────────────────────────────────────┘
```

### 정책 우선순위

여러 정책이 같은 리소스에 적용될 때:

1. **Route Rule 레벨** (HTTPRoute의 특정 rule) - 최우선
2. **Route 레벨** (HTTPRoute 전체)
3. **Listener 레벨** (Gateway의 특정 listener)
4. **Gateway 레벨** (Gateway 전체) - 최하위

같은 레벨에서 충돌하면 생성 시간이 빠른 정책이 우선한다.

## 5가지 핵심 정책

### 1. ClientTrafficPolicy

클라이언트와 Gateway 간 연결을 제어한다.

| 기능 | 설명 |
|------|------|
| TLS 설정 | mTLS, 인증서 검증 |
| TCP Keepalive | 연결 유지 설정 |
| HTTP 프로토콜 | HTTP/1, HTTP/2, HTTP/3 설정 |
| Client IP | 프록시 체인에서 원본 IP 추출 |
| Path 정규화 | 백엔드 호환성을 위한 경로 정리 |

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: ClientTrafficPolicy
metadata:
  name: client-policy
spec:
  targetRefs:
    - kind: Gateway
      name: my-gateway
  timeout:
    http:
      idleTimeout: 30s
```

### 2. BackendTrafficPolicy

Gateway와 Backend 간 트래픽을 제어한다.

| 기능 | 설명 |
|------|------|
| Rate Limiting | Global/Local Rate Limit |
| Load Balancing | Round Robin, Least Request, Consistent Hashing |
| Circuit Breaker | 연결 수 기반 서킷 브레이커 |
| Retry | 재시도 정책 |
| Fault Injection | 테스트용 장애 주입 |

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: BackendTrafficPolicy
metadata:
  name: backend-policy
spec:
  targetRefs:
    - kind: HTTPRoute
      name: my-route
  circuitBreaker:
    maxConnections: 100
  rateLimit:
    local:
      requests: 10
      unit: Second
```

### 3. SecurityPolicy

인증과 인가를 담당한다.

| 기능 | 설명 |
|------|------|
| JWT 인증 | JWT 토큰 검증 |
| OIDC | OAuth 2.0/OIDC 인증 |
| Basic Auth | 기본 인증 |
| API Key | API 키 인증 |
| CORS | Cross-Origin 설정 |
| IP 필터링 | Allow/Deny 리스트 |
| External Auth | 외부 인가 서비스 연동 |

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: SecurityPolicy
metadata:
  name: security-policy
spec:
  targetRefs:
    - kind: HTTPRoute
      name: api-route
  jwt:
    providers:
      - name: auth0
        issuer: https://example.auth0.com/
        audiences:
          - my-api
```

### 4. EnvoyExtensionPolicy

커스텀 로직을 주입한다.

| 방식 | 설명 |
|------|------|
| Wasm | WebAssembly 확장 (OCI 이미지 또는 HTTP URL) |
| External Process | 외부 프로세스와 RPC 통신 |

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyExtensionPolicy
metadata:
  name: wasm-extension
spec:
  targetRefs:
    - kind: HTTPRoute
      name: backend
  wasm:
    - name: custom-filter
      code:
        type: HTTP
        http:
          url: https://example.com/filter.wasm
          sha256: abc123...
```

### 5. EnvoyPatchPolicy

생성된 xDS 설정을 직접 수정한다. JSON Patch 문법을 사용한다.

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyPatchPolicy
metadata:
  name: custom-patch
spec:
  targetRef:
    kind: Gateway
    name: eg
  type: JSONPatch
  jsonPatches:
    - type: "type.googleapis.com/envoy.config.listener.v3.Listener"
      name: default/eg/http
      operation:
        op: add
        path: "/default_filter_chain/filters/0/typed_config/local_reply_config"
        value:
          mappers:
            - filter:
                status_code_filter:
                  comparison:
                    op: EQ
                    value:
                      default_value: 404
              body:
                inline_string: "Not Found"
```

EnvoyPatchPolicy는 기본적으로 비활성화되어 있다. xDS에 익숙한 고급 사용자를 위한 기능이다.

## 정책 병합 (Policy Merging)

BackendTrafficPolicy는 상위 정책과 병합할 수 있다. 플랫폼 팀이 Gateway 레벨에서 기본 정책을 설정하고, 애플리케이션 팀이 Route 레벨에서 추가 정책을 적용하는 시나리오에 유용하다.

```yaml
# 플랫폼 팀: Gateway 레벨 정책
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: BackendTrafficPolicy
metadata:
  name: global-policy
spec:
  targetRefs:
    - kind: Gateway
      name: eg
  rateLimit:
    global:
      rules:
        - limit:
            requests: 100
            unit: Second

---
# 애플리케이션 팀: Route 레벨 정책 (병합)
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: BackendTrafficPolicy
metadata:
  name: route-policy
spec:
  mergeType: StrategicMerge  # 병합 활성화
  targetRefs:
    - kind: HTTPRoute
      name: signup-service
  rateLimit:
    global:
      rules:
        - limit:
            requests: 5
            unit: Minute
```

두 Rate Limit이 모두 적용된다: 전체 100 req/s + signup 5 req/min.

## Ingress API와의 비교

| 기능 | Ingress | Gateway API + Envoy Gateway |
|------|---------|---------------------------|
| Path 매칭 | Prefix, Exact만 | Regex 지원 |
| Header 매칭 | 어노테이션 의존 | 네이티브 지원 |
| Traffic Split | 어노테이션 의존 | 네이티브 지원 |
| 확장 방식 | 비표준 어노테이션 | 타입 안전한 CRD |
| 역할 분리 | 없음 | GatewayClass → Gateway → Route |
| 이식성 | 구현체마다 다름 | 표준화됨 |

Ingress의 어노테이션 기반 확장은 타입 안전성이 없고 구현체마다 다르다. Gateway API의 Policy Attachment는 CRD로 정의되어 스키마 검증이 가능하다.

## 언제 Envoy Gateway를 선택하는가

### 적합한 경우

- Kubernetes 네이티브 API Gateway가 필요할 때
- Envoy의 고급 기능(Rate Limiting, Circuit Breaker)이 필요할 때
- Gateway API 표준을 따르고 싶을 때
- 플랫폼/애플리케이션 팀 간 역할 분리가 필요할 때

### 고려할 점

| 항목 | 설명 |
|------|------|
| 학습 곡선 | Gateway API + Envoy Gateway 확장 모두 이해 필요 |
| 운영 복잡도 | Envoy Gateway Controller + Envoy Proxy 운영 |
| 디버깅 | xDS 변환 과정 이해 필요 |

## 결론

Envoy Gateway는 Gateway API의 실용적인 확장이다:

| 구분 | 설명 |
|------|------|
| 표준 준수 | Gateway API 완전 구현 |
| 확장성 | 5가지 Policy CRD로 Envoy 기능 노출 |
| 유연성 | EnvoyPatchPolicy로 xDS 직접 수정 가능 |
| 역할 분리 | 플랫폼/애플리케이션 팀 책임 분리 |

단순한 라우팅 이상의 기능이 필요하고, Kubernetes 네이티브 방식을 선호한다면 Envoy Gateway는 좋은 선택이다.

## 참고 자료

- [Envoy Gateway 공식 문서](https://gateway.envoyproxy.io/)
- [Introducing Envoy Gateway's Gateway API Extensions](https://www.zhaohuabing.com/post/2024-09-22-introducing-envoy-gateways-gateway-api-extensions-en/)
- [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/)
- [Envoy Proxy xDS](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/operations/dynamic_configuration)
