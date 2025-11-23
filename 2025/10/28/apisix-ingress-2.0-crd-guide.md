# APISIX Ingress Controller 2.0: CRD 선택 가이드

**작성일**: 2025-10-28
**대상**: APISIX Ingress Controller 2.0 사용자

## TL;DR 🎯

**2025년 현재, Kubernetes Ingress + annotations 사용을 권장합니다.**

- ✅ **ApisixPluginConfig는 deprecated 아님** - 활발히 사용 중
- ❌ **ApisixClusterConfig만 삭제됨** - ApisixGlobalRule로 대체
- 🚧 **Gateway API는 Alpha** - CORS, 플러그인 미지원

## APISIX Ingress Controller 2.0 CRD 현황

### 공식 지원 CRD (2025-10-28 기준)

#### v2 (APISIX 네이티브) - ✅ Stable

| CRD | 용도 | 상태 | 비고 |
|-----|------|------|------|
| **ApisixRoute** | 라우팅 규칙 | ✅ Stable | ApisixRoute/v2 |
| **ApisixPluginConfig** | 재사용 플러그인 설정 | ✅ Stable | **Deprecated 아님** |
| **ApisixTls** | TLS 인증서 매핑 | ✅ Stable | Secret 연결 |
| **ApisixUpstream** | 업스트림 설정 | ✅ Stable | 부하 분산 |
| **ApisixConsumer** | Consumer 인증 | ✅ Stable | JWT, Key-Auth 등 |
| **ApisixGlobalRule** | 전역 규칙 | ✅ Stable | ApisixClusterConfig 대체 |

#### v1alpha1 (Gateway API 관련) - 🚧 Alpha

| CRD | 용도 | 상태 | 비고 |
|-----|------|------|------|
| **HTTPRoutePolicy** | HTTPRoute 정책 | 🚧 Alpha | 플러그인 미지원 |
| **PluginConfig** | Gateway API 플러그인 | 🚧 Alpha | HTTPRoute용 |
| **GatewayProxy** | Data Plane 연결 | 🚧 Alpha | 2.0 신규 |
| **BackendTrafficPolicy** | 백엔드 정책 | 🚧 Alpha | - |
| **Consumer** | Gateway API Consumer | 🚧 Alpha | - |

#### 삭제된 CRD - ❌ Removed

| CRD | 대체 | 비고 |
|-----|------|------|
| **ApisixClusterConfig** | ApisixGlobalRule | 2.0에서 삭제 |

## CORS 설정 방법 비교 (2025년 현재)

### 방법 1: Kubernetes Ingress ⭐⭐⭐ (권장)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-service
  annotations:
    # cert-manager 자동 인증서
    cert-manager.io/cluster-issuer: letsencrypt-cloudflare

    # CORS 플러그인 (YAML)
    k8s.apisix.apache.org/plugins: |
      - name: cors
        enable: true
        config:
          allow_origins: "*"
          allow_methods: "GET,POST,PUT,DELETE,PATCH,OPTIONS,HEAD"
          allow_headers: "*"
          expose_headers: "*"
          allow_credential: true
          max_age: 3600

      - name: redirect
        enable: true
        config:
          http_to_https: true

spec:
  ingressClassName: apisix
  tls:
    - hosts:
        - example.com
      secretName: example-com-tls  # cert-manager가 자동 생성
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80
```

**장점**:
- ✅ Kubernetes 표준 Ingress
- ✅ cert-manager 완벽 통합 (annotation 한 줄)
- ✅ 다른 Ingress Controller로 전환 용이
- ✅ 배포 빠르고 간단
- ✅ CORS 완벽 작동

**단점**:
- ⚠️ annotation에 YAML 문자열 (타입 체크 없음)
- ⚠️ 복잡한 플러그인 설정 시 가독성 떨어짐

**사용 시나리오**:
- MVP / 빠른 프로토타이핑
- 표준 준수 필요
- 단순~중간 복잡도 설정

---

### 방법 2: ApisixRoute + ApisixPluginConfig ⭐⭐

```yaml
# 1. 재사용 가능한 플러그인 설정
apiVersion: apisix.apache.org/v2
kind: ApisixPluginConfig
metadata:
  name: cors-https
  namespace: default
spec:
  plugins:
    - name: cors
      enable: true
      config:
        allow_origins: "*"
        allow_methods: "GET,POST,PUT,DELETE,PATCH,OPTIONS,HEAD"
        allow_headers: "*"
        expose_headers: "*"
        allow_credential: true
        max_age: 3600

    - name: redirect
      enable: true
      config:
        http_to_https: true
---
# 2. 라우팅 규칙
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: my-service
  namespace: default
spec:
  http:
    - name: my-route
      match:
        hosts:
          - example.com
        paths:
          - /*
      backends:
        - serviceName: my-service
          servicePort: 80

      # PluginConfig 참조 (cross-namespace 가능)
      pluginConfigName: cors-https
      pluginConfigNamespace: default

      websocket: true
      timeout:
        connect: 60s
        send: 300s
        read: 300s
---
# 3. TLS 인증서 (cert-manager Secret 연결)
apiVersion: apisix.apache.org/v2
kind: ApisixTls
metadata:
  name: example-com-tls
  namespace: default
spec:
  hosts:
    - example.com
  secret:
    name: example-com-tls  # cert-manager가 생성한 Secret
    namespace: default
---
# 4. cert-manager Certificate (별도 관리)
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com
  namespace: default
spec:
  secretName: example-com-tls
  issuerRef:
    name: letsencrypt-cloudflare
    kind: ClusterIssuer
  dnsNames:
    - example.com
```

**장점**:
- ✅ Type-safe (CRD validation)
- ✅ ApisixPluginConfig 재사용 가능 (cross-namespace)
- ✅ 깔끔한 YAML 구조
- ✅ APISIX 고급 기능 100% 사용
- ✅ CORS 완벽 작동

**단점**:
- ❌ APISIX 전용 (벤더 종속)
- ⚠️ cert-manager Certificate 별도 관리 필요
- ⚠️ 리소스 4개 관리 (ApisixRoute, ApisixPluginConfig, ApisixTls, Certificate)

**사용 시나리오**:
- APISIX 장기 사용 확정
- 복잡한 플러그인 설정 많음
- 여러 서비스에서 플러그인 재사용
- Type-safe 배포 원함

---

### 방법 3: Gateway API ❌ (미래, 아직 안 됨)

```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: HTTPRoute
metadata:
  name: my-service
spec:
  hostnames:
    - example.com
  rules:
    - backendRefs:
        - name: my-service
          port: 80
      matches:
        - path:
            type: PathPrefix
            value: /

# ❌ CORS 설정 방법 없음
# ❌ 플러그인 설정 방법 없음
# ❌ HTTPS 리다이렉트 불가
```

**현재 상태 (2025-10-28)**:
- 🚧 Alpha 단계 (v1alpha2)
- ❌ CORS 미지원 (GEP 1767 제안 단계)
- ❌ 플러그인 설정 방법 제공 안 됨
- ❌ HTTPRoutePolicy로도 CORS 설정 불가
- ⏳ 2026년 이후 Stable 예상

**장점** (미래):
- ✅ Kubernetes 표준 (Gateway API)
- ✅ 벤더 중립적

**단점** (현재):
- ❌ CORS, HTTPS 리다이렉트 등 기본 기능 미지원
- ❌ 프로덕션 사용 불가

---

## 아키텍처별 권장 방식

### MVP / 스타트업

**Kubernetes Ingress + annotations** ⭐⭐⭐

```yaml
# 한 파일에 모든 설정
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-cloudflare
    k8s.apisix.apache.org/plugins: |
      - name: cors
        config: {...}
```

**이유**:
- 빠른 배포
- cert-manager 간단 통합
- 표준 준수
- 나중에 전환 가능

### 중소 규모 프로덕션

**Kubernetes Ingress 또는 ApisixRoute** ⭐⭐⭐

- 단순 설정: Ingress
- 복잡한 플러그인: ApisixRoute + ApisixPluginConfig

### 대규모 엔터프라이즈

**ApisixRoute + ApisixPluginConfig** ⭐⭐⭐

```yaml
# 플러그인 재사용 (100개 서비스에 동일 CORS)
apiVersion: apisix.apache.org/v2
kind: ApisixPluginConfig
metadata:
  name: standard-cors
  namespace: common
---
# 각 서비스에서 참조
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
spec:
  pluginConfigName: standard-cors
  pluginConfigNamespace: common
```

**이유**:
- 플러그인 재사용 (DRY 원칙)
- Type-safe 배포
- 중앙 관리

---

## 실전 예시: Runtime 도메인 CORS 설정

### 현재 방식 (Ingress)

**서버 코드** (server/src/gateway/ingress/runtime-ingress.service.ts):

```typescript
if (ingressClassName === 'apisix') {
  Object.assign(annotations, {
    "k8s.apisix.apache.org/plugins": `
- name: cors
  enable: true
  config:
    allow_origins: "*"
    allow_methods: "GET,POST,PUT,DELETE,PATCH,OPTIONS,HEAD"
    allow_headers: "*"
    expose_headers: "*"
    allow_credential: true
    max_age: 3600

- name: redirect
  enable: true
  config:
    http_to_https: true
`.trim()
  });
}
```

**생성되는 Ingress**:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rponue
  namespace: app-rponue
  annotations:
    k8s.apisix.apache.org/plugins: |
      - name: cors
        enable: true
        config:
          allow_origins: "*"
          allow_methods: "GET,POST,PUT,DELETE,PATCH,OPTIONS,HEAD"
          allow_headers: "*"
          expose_headers: "*"
          allow_credential: true
          max_age: 3600

      - name: redirect
        enable: true
        config:
          http_to_https: true
spec:
  ingressClassName: apisix
  tls:
    - hosts:
        - rponue.api.imprun.dev
      secretName: wildcard-api-imprun-dev-tls
  rules:
    - host: rponue.api.imprun.dev
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: rponue
                port:
                  number: 8000
```

**결과**: ✅ 작동 완벽

---

## 자주 묻는 질문 (FAQ)

### Q1. ApisixPluginConfig가 deprecated 되었나요?

**A**: ❌ **아닙니다.** ApisixPluginConfig는 여전히 활발히 사용 중이며, APISIX Ingress Controller 2.0 공식 API Reference에 포함되어 있습니다.

- ✅ ApisixPluginConfig (v2) - Stable
- ❌ ApisixClusterConfig - 삭제됨 (ApisixGlobalRule로 대체)

### Q2. Gateway API를 언제 사용해야 하나요?

**A**: ⏳ **아직 아닙니다.** Gateway API는 현재 Alpha 단계이며, CORS와 같은 기본 기능이 지원되지 않습니다. 2026년 이후 Stable 버전이 나오면 고려하세요.

### Q3. Ingress와 ApisixRoute 중 무엇을 선택해야 하나요?

**A**: 대부분의 경우 **Kubernetes Ingress**를 권장합니다.

| 시나리오 | 권장 방식 |
|---------|----------|
| MVP / 빠른 배포 | Kubernetes Ingress |
| 표준 준수 필요 | Kubernetes Ingress |
| APISIX 장기 사용 확정 + 복잡한 플러그인 | ApisixRoute + ApisixPluginConfig |
| Gateway 전환 가능성 | Kubernetes Ingress |

### Q4. cert-manager는 ApisixRoute와 어떻게 연동되나요?

**A**: ApisixRoute는 cert-manager annotation을 직접 지원하지 않습니다. Certificate 리소스를 별도로 관리해야 합니다.

```yaml
# 1. Certificate 생성 (cert-manager)
apiVersion: cert-manager.io/v1
kind: Certificate
spec:
  secretName: example-tls

# 2. ApisixTls로 연결
apiVersion: apisix.apache.org/v2
kind: ApisixTls
spec:
  secret:
    name: example-tls
```

Ingress는 annotation 한 줄로 끝:
```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-cloudflare
```

### Q5. 플러그인을 여러 서비스에서 재사용하려면?

**A**: ApisixPluginConfig 사용을 권장합니다.

```yaml
# common namespace에 공통 플러그인
apiVersion: apisix.apache.org/v2
kind: ApisixPluginConfig
metadata:
  name: standard-cors
  namespace: common
---
# app-1 namespace에서 참조
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  namespace: app-1
spec:
  pluginConfigName: standard-cors
  pluginConfigNamespace: common  # cross-namespace 참조
---
# app-2 namespace에서도 동일하게 참조
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  namespace: app-2
spec:
  pluginConfigName: standard-cors
  pluginConfigNamespace: common
```

---

## 마이그레이션 시나리오

### Ingress → ApisixRoute

```bash
# 1. PluginConfig 생성
kubectl apply -f apisix-plugin-config.yaml

# 2. ApisixTls 생성
kubectl apply -f apisix-tls.yaml

# 3. ApisixRoute 생성
kubectl apply -f apisix-route.yaml

# 4. 기존 Ingress 삭제
kubectl delete ingress my-service

# 5. 테스트
curl https://example.com -v
```

### Kong → APISIX (Ingress)

```bash
# 1. APISIX Ingress 생성 (Kong과 동일한 spec)
kubectl apply -f apisix-ingress.yaml

# 2. 테스트
curl https://example.com -v

# 3. Kong Ingress 삭제
kubectl delete ingress my-service-kong
```

---

## 결론

### 2025년 현재 권장사항

**대부분의 경우: Kubernetes Ingress + annotations** ⭐⭐⭐

- ✅ 표준 준수
- ✅ 빠른 배포
- ✅ cert-manager 완벽 통합
- ✅ CORS 완벽 작동
- ✅ 다른 Gateway로 전환 가능

**복잡한 엔터프라이즈 환경: ApisixRoute + ApisixPluginConfig** ⭐⭐

- ✅ Type-safe
- ✅ 플러그인 재사용
- ✅ APISIX 기능 100%
- ❌ 벤더 종속

**Gateway API: 아직 사용 불가** ❌

- 🚧 Alpha 단계
- ❌ CORS 미지원
- ⏳ 2026년 이후 고려

### ApisixPluginConfig는 Deprecated 아님! ✅

- ✅ APISIX Ingress Controller 2.0 공식 CRD
- ✅ 활발히 사용 중
- ✅ 안심하고 사용 가능
- ❌ ApisixClusterConfig만 삭제됨 (→ ApisixGlobalRule)

---

**작성자**: Claude (APISIX Ingress Controller 공식 문서 기반)
**최종 업데이트**: 2025-10-28
**참고 문서**:
- [APISIX Ingress Controller API Reference](https://apisix.apache.org/docs/ingress-controller/reference/apisix-ingress-controller/api-reference)
- [APISIX Ingress Controller Upgrade Guide](https://apisix.apache.org/docs/ingress-controller/upgrade-guide/)
- [Gateway API Documentation](https://gateway-api.sigs.k8s.io/)
