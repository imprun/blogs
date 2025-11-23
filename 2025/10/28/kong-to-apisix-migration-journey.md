# Kong에서 APISIX로의 험난한 여정: Cilium 환경에서의 시행착오

**작성일**: 2025-10-28
**카테고리**: DevOps, Kubernetes, API Gateway
**난이도**: ⭐⭐⭐⭐ (고급)

## TL;DR

AI Gateway를 구축하기 위해 Kong에서 APISIX로 전환하는 과정에서, **문서 오류부터 시작해 6가지 주요 시행착오**를 겪었습니다. 최종적으로 `hostNetwork: true` + `9080/9443 포트` + `iptables REDIRECT` 방식으로 안정적인 구성을 완성했습니다.

**핵심 교훈**:
- **📝 문서는 정확하게 작성하자** - 이전 블로그를 잘못 작성해서 반나절 낭비
- Cilium 환경에서는 NodePort가 eBPF로 처리되어 iptables REDIRECT 대상이 될 수 없음
- CAP_NET_BIND_SERVICE만으로는 nginx 기반 API 게이트웨이이 80/443에 바인딩 불가
- APISIX Ingress Controller 2.0에서는 ApisixPluginConfig가 여전히 유효 (deprecated 아님)
- ApisixRoute v2가 Ingress보다 Type-safe하고 재사용 가능

---

## 배경: 왜 Kong에서 APISIX로?

AI Gateway 프로젝트를 시작하면서, 기존 Kong API Gateway를 APISIX로 교체하기로 결정했습니다.

### Kong의 결정적 한계: AI Proxy가 유료 💰

Kong도 [AI Gateway 플러그인](https://docs.konghq.com/hub/kong-inc/ai-proxy/)을 제공하지만, **치명적인 문제**가 있습니다:

```yaml
# Kong AI Proxy Plugin
plugins:
  - name: ai-proxy
    config:
      route_type: "llm/v1/chat"
      auth:
        header_name: "Authorization"
      model:
        provider: "openai"
        name: "gpt-4"
```

**❌ Kong AI Proxy는 Kong AI Gateway 라이센스 필요 (유료)**

Kong의 AI 관련 플러그인들은 **Kong AI Gateway 엔터프라이즈 라이센스**가 필요한 유료 기능입니다. 스타트업이나 개인 프로젝트에서는 **비용 부담**이 큽니다.

### Kong의 기타 한계

- ❌ **복잡한 플러그인 설정** (KongPlugin CRD 별도 관리)
- ❌ **리소스 사용량** (단일 Pod 512Mi 메모리)
- ❌ **커뮤니티 플러그인 제한적** (대부분 엔터프라이즈 기능)

### APISIX의 장점

- ✅ **AI 플러그인 완전 무료** - ai-proxy, ai-prompt-decorator, ai-prompt-guard 등
- ✅ **풍부한 플러그인 생태계** (150+ plugins, 모두 오픈소스)
- ✅ **경량화 및 고성능** (분산 아키텍처)
- ✅ **활발한 커뮤니티** (Apache Software Foundation)

**비용 비교**:
```
Kong AI Gateway: $월 수백~수천 달러 (엔터프라이즈 라이센스)
APISIX AI 기능: $0 (완전 무료, Apache License 2.0)
```

이것이 **APISIX로 전환을 결정한 가장 큰 이유**입니다.

---

## 시행착오 #0: 이전 블로그를 잘못 작성해서 삽질의 시작 😭

### 메타 시행착오

**가장 큰 실수는 출발점부터 잘못되었다는 것입니다.**

이전에 작성한 블로그 문서에서 Kong의 네트워크 구성을 분석하면서 **"Kong은 NodePort를 사용하고 iptables로 리다이렉트한다"**고 작성했습니다. 하지만 이건 **완전히 틀린 정보**였습니다.

**실제 Kong 구성**을 확인해보니:

```yaml
# Kong Gateway Helm values (실제)
gateway:
  deployment:
    hostNetwork: true  # ✅ hostNetwork 사용!

  service:
    type: ClusterIP  # NodePort 아님!
```

Kong은 처음부터 **hostNetwork: true**를 사용하고 있었습니다. 블로그를 잘못 작성한 탓에, 이 사실을 망각하고 **"APISIX도 NodePort + iptables로 하면 되겠지"**라고 착각했습니다.

### 낭비한 시간

- ❌ NodePort 30080/30443 설정 시도
- ❌ Cilium eBPF vs iptables 디버깅
- ❌ NodePort가 왜 호스트에 바인딩 안 되는지 조사
- ❌ 여러 가지 workaround 시도

**결과**: 처음부터 hostNetwork: true로 시작했다면 1시간이면 끝날 일을 **반나절** 동안 삽질했습니다.

### 교훈

**📝 문서는 정확하게 작성하자. 미래의 나를 위해서라도!**

---

## 시행착오 #1: NodePort가 iptables에 보이지 않는다? 🤔

### 문제 발생

시행착오 #0에서 언급했듯이, 잘못된 문서를 믿고 APISIX Data Plane을 NodePort로 배포하고, iptables로 80/443 → 30080/30443 리다이렉트를 설정했습니다.

```bash
# APISIX NodePort Service 배포
$ kubectl get svc -n apisix-system apisix-dp-gateway
NAME                  TYPE       CLUSTER-IP      PORT(S)
apisix-dp-gateway     NodePort   10.96.123.45    80:30080/TCP,443:30443/TCP

# iptables 리다이렉트 설정
$ sudo iptables -t nat -A PREROUTING -i enp0s6 -p tcp --dport 80 -j REDIRECT --to-ports 30080
$ sudo iptables -t nat -A PREROUTING -i enp0s6 -p tcp --dport 443 -j REDIRECT --to-ports 30443

# 테스트
$ curl https://portal.imprun.dev -v
curl: (7) Failed to connect to portal.imprun.dev port 443: Connection refused
```

**❌ 연결 실패!**

### 원인 분석

```bash
# 30080/30443 포트가 호스트에 바인딩되어 있는지 확인
$ sudo netstat -tlnp | grep 30080
(출력 없음)  # 😱 포트가 바인딩 안 됨!

# Cilium service 확인
$ cilium service list | grep 30080
10.96.123.45:80    NodePort   30080   # ✅ Cilium에는 보임
```

**핵심 발견**: Cilium은 kube-proxy를 대체하여 **eBPF**로 NodePort를 처리합니다. 따라서 NodePort는 **eBPF 레벨**에서만 존재하고, **호스트 포트에 실제로 바인딩되지 않습니다**.

iptables REDIRECT는 **실제로 LISTEN 중인 포트**에만 작동하므로, eBPF 레벨의 NodePort는 REDIRECT 대상이 될 수 없습니다.

### 네트워크 플로우 비교

**의도한 플로우** (kube-proxy 환경):
```
외부:80 → iptables PREROUTING REDIRECT → 30080 (호스트 바인딩) → APISIX Pod
```

**실제 플로우** (Cilium 환경):
```
외부:80 → iptables PREROUTING (30080 없음) → 연결 실패 ❌
외부:30080 → eBPF (직접 처리) → APISIX Pod ✅
```

### 해결 방법: hostNetwork: true

Kong의 실제 설정을 확인해보니, `hostNetwork: true`를 사용하고 있었습니다.

```bash
$ kubectl get deployment -n kong kong-kong -o yaml | grep hostNetwork
hostNetwork: true
dnsPolicy: ClusterFirstWithHostNet
```

`hostNetwork: true`를 사용하면 Pod가 호스트의 네트워크 네임스페이스를 직접 사용하므로, **Pod의 포트가 호스트에 실제로 바인딩**됩니다.

---

## 시행착오 #2: 직접 80/443에 바인딩하면 안 돼? 🤷

### 시도

"어차피 hostNetwork를 쓰는데, 굳이 9080/9443을 쓸 필요가 있나? 직접 80/443에 바인딩하면 iptables도 필요 없잖아!"

```yaml
# dp-values.yaml (시도 1)
hostNetwork: true

service:
  http:
    containerPort: 80   # 직접 80
  tls:
    containerPort: 443  # 직접 443
```

### 결과: Permission Denied

```bash
$ kubectl logs -n apisix-system apisix-dp-xxx
2025/10/28 17:21:18 [emerg] 1#1: bind() to 0.0.0.0:80 failed (13: Permission denied)
nginx: [emerg] bind() to 0.0.0.0:80 failed (13: Permission denied)
```

**❌ 실패!**

### 원인: Privileged Ports

Linux에서 **1024 미만의 포트(privileged ports)**는 root 권한이 필요합니다.

```bash
$ kubectl exec -n apisix-system apisix-dp-xxx -- id
uid=636(apisix) gid=636(apisix)  # non-root 사용자
```

APISIX는 보안상의 이유로 **non-root 사용자(uid=636)**로 실행됩니다. OpenResty(nginx)가 내부적으로 root로 시작하여 포트를 바인딩한 뒤 non-root로 권한을 낮추지만, APISIX 컨테이너는 처음부터 non-root로 시작합니다.

---

## 시행착오 #3: CAP_NET_BIND_SERVICE로 해결할 수 있지 않을까? 💡

### 시도: Linux Capabilities 추가

"Linux Capabilities 중 `CAP_NET_BIND_SERVICE`를 추가하면 non-root 사용자도 privileged ports에 바인딩할 수 있다!"

```yaml
# dp-values.yaml (시도 2)
securityContext:
  capabilities:
    add:
      - NET_BIND_SERVICE  # Privileged ports 바인딩 허용

hostNetwork: true

service:
  http:
    containerPort: 80
  tls:
    containerPort: 443
```

### 결과: 여전히 Permission Denied

```bash
$ kubectl logs -n apisix-system apisix-dp-xxx
bind() to 0.0.0.0:80 failed (13: Permission denied)
```

**❌ 실패!**

### 원인: nginx의 동작 방식

APISIX는 **OpenResty(nginx)**를 기반으로 합니다. nginx는 다음과 같이 동작합니다:

1. Master process가 root로 시작
2. 80/443 포트에 바인딩
3. Worker process를 non-root 사용자로 fork

하지만 APISIX 컨테이너는 **처음부터 non-root(uid=636)**로 시작하므로, CAP_NET_BIND_SERVICE만으로는 부족합니다.

### 대안: setcap을 사용한 Init Container

복잡하지만 가능한 방법:

```yaml
initContainers:
  - name: setcap
    image: apache/apisix:3.14.1-ubuntu
    command:
      - setcap
      - cap_net_bind_service=+ep
      - /usr/local/openresty/nginx/sbin/nginx
    securityContext:
      runAsUser: 0  # root 필요
```

하지만 이는 **너무 복잡**하고, MVP 단계에서 불필요한 오버헤드입니다.

### 결정: 9080/9443 + iptables 방식 채택 ✅

Kong도 실제로는 8000/8443 포트를 사용하고 iptables로 리다이렉트합니다. **검증된 방식**을 따르기로 결정했습니다.

```yaml
# dp-values.yaml (최종)
hostNetwork: true
dnsPolicy: ClusterFirstWithHostNet

service:
  http:
    containerPort: 9080  # Non-privileged port
  tls:
    containerPort: 9443
```

```bash
# 2호기 iptables 설정
sudo iptables -t nat -A PREROUTING -i enp0s6 -p tcp --dport 80 -j REDIRECT --to-ports 9080
sudo iptables -t nat -A PREROUTING -i enp0s6 -p tcp --dport 443 -j REDIRECT --to-ports 9443
sudo iptables-save | sudo tee /etc/iptables/rules.v4
```

**✅ 성공!**

```bash
$ curl https://portal.imprun.dev -v
< HTTP/2 200
< server: APISIX/3.14.1
```

---

## 시행착오 #4: ApisixPluginConfig가 Deprecated 되었다고? 😱

### 혼란

APISIX Ingress Controller 1.8 문서의 ApisixPluginConfig 페이지를 읽다가 상단에 다음 문구를 발견했습니다:

```
⚠️ This is documentation for Apache APISIX® -- Cloud-Native API Gateway
and AI Gateway 1.8.0, which is no longer actively maintained.
```

"ApisixPluginConfig가 deprecated 된 건가? Gateway API를 써야 하나?" - 하지만 이건 **착각**이었습니다!

### 조사 결과

[APISIX Ingress Controller 2.0 공식 API Reference](https://apisix.apache.org/docs/ingress-controller/reference/apisix-ingress-controller/api-reference/)를 확인한 결과:

#### ✅ 유지되는 CRD (apisix.apache.org/v2)

- **ApisixRoute** - 라우팅 규칙
- **ApisixPluginConfig** - 플러그인 설정 (✅ **Stable**)
- **ApisixTls** - TLS 인증서
- **ApisixUpstream** - 업스트림 설정
- **ApisixConsumer** - Consumer 인증
- **ApisixGlobalRule** - 전역 규칙

#### ❌ 삭제된 CRD

- **ApisixClusterConfig** - ❌ Removed (→ ApisixGlobalRule로 대체)

#### 🚧 Gateway API (v1alpha1)

- HTTPRoute, TCPRoute - 기본 지원
- **CORS, 플러그인 설정 미지원** - 아직 Alpha

### 결론

**착각의 원인**: 1.8 문서 상단의 "1.8.0 버전이 더 이상 유지보수되지 않는다"는 문구를 보고, **ApisixPluginConfig CRD 자체가 deprecated** 된 것으로 오해했습니다.

실제로는 **APISIX Ingress Controller 1.8 버전**이 유지보수되지 않는다는 의미였고, **ApisixPluginConfig CRD는 2.0에서도 Stable**합니다! ✅

Gateway API는 아직 Alpha 단계이므로 2026년 이후에 고려하면 됩니다.

---

## 시행착오 #5: Ingress vs ApisixRoute, 뭘 써야 해? 🤔

### 선택지

1. **Kubernetes Ingress + annotations** (현재 작동 중)
2. **ApisixRoute + ApisixPluginConfig** (APISIX 네이티브)
3. **Gateway API** (미래 표준, 아직 Alpha)

### Ingress 방식

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-cloudflare
    k8s.apisix.apache.org/plugins: |  # ❌ 문자열 YAML
      - name: cors
        config:
          allow_origins: "*"
```

**장점**:
- ✅ Kubernetes 표준
- ✅ cert-manager 완벽 통합
- ✅ 다른 Gateway로 전환 용이

**단점**:
- ❌ annotation에 YAML 문자열 (타입 체크 없음)
- ❌ 복잡한 설정 시 가독성 떨어짐

### ApisixRoute 방식

```yaml
# 1. 재사용 가능한 플러그인 설정
apiVersion: apisix.apache.org/v2
kind: ApisixPluginConfig
metadata:
  name: runtime-cors-https
  namespace: imprun-system
spec:
  plugins:
    - name: cors
      enable: true
      config:
        allow_origins: "*"
        allow_headers: "*"
        expose_headers: "*"
---
# 2. 라우팅 규칙
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: my-service
spec:
  http:
    - name: my-route
      match:
        hosts: [example.com]
        paths: [/*]
      backends:
        - serviceName: my-service
          servicePort: 80

      # ✅ PluginConfig 재사용 (cross-namespace)
      plugin_config_name: runtime-cors-https
      plugin_config_namespace: imprun-system
```

**장점**:
- ✅ Type-safe (CRD validation)
- ✅ ApisixPluginConfig 재사용 가능
- ✅ 깔끔한 YAML 구조
- ✅ APISIX 고급 기능 100% 사용

**단점**:
- ❌ APISIX 전용 (벤더 종속)
- ⚠️ cert-manager Certificate 별도 관리

### 최종 결정: ApisixRoute v2 ⭐

장기적으로 APISIX를 사용할 것이므로, **ApisixRoute + ApisixPluginConfig** 방식을 채택했습니다.

**이유**:
1. **Type-safe 배포** - CRD validation으로 배포 전 에러 체크
2. **플러그인 재사용** - 100개 Runtime 서비스에 동일한 CORS 적용 가능
3. **명확한 구조** - 플러그인과 라우팅 분리
4. **APISIX 기능 100%** - 미래 확장성

---

## 최종 아키텍처

### 네트워크 구성

```
인터넷 (80/443)
  ↓
2호기 iptables REDIRECT
  ↓ 80→9080, 443→9443
APISIX Data Plane (hostNetwork: true, 2호기)
  ↓
APISIX Control Plane (etcd, 1호기)
  ↓
Backend Services
```

### APISIX 구성요소

```
k8s/apisix/
├── helm-values/           # Helm 차트 설정
│   ├── cp-values.yaml     # Control Plane (1호기)
│   ├── dp-values.yaml     # Data Plane (2호기, hostNetwork)
│   ├── dashboard-values.yaml
│   └── ic-values.yaml     # Ingress Controller
│
├── routes/                # ApisixRoute
│   └── main-domains.yaml  # api, admin, portal, imprun
│
├── plugins/               # ApisixPluginConfig
│   └── runtime-cors-https.yaml  # CORS + HTTPS 리다이렉트
│
└── tls/                   # TLS/인증서
    ├── apisix-tls.yaml
    └── certificates.yaml  # Wildcard (*.api, *.admin)
```

### 배포된 도메인

| 도메인 | 서비스 | HTTPS | CORS | ApisixRoute |
|--------|--------|-------|------|-------------|
| **portal.imprun.dev** | 웹 콘솔 | ✅ | ❌ | portal-imprun-dev |
| **api.imprun.dev** | API 서버 | ✅ | ❌ | api-imprun-dev |
| **admin.imprun.dev** | Admin API | ✅ | ❌ | admin-imprun-dev |
| **imprun.dev** | 루트 | ✅ | ❌ | imprun-dev-redirect |
| **\*.api.imprun.dev** | Runtime | ✅ | ✅ | 동적 생성 |
| **\*.admin.imprun.dev** | Admin Runtime | ✅ | ❌ | 동적 생성 |

---

## 성능 비교: Kong vs APISIX

### 메모리 사용량

```bash
# Kong (before)
$ kubectl top pod -n kong
NAME                        CPU    MEMORY
kong-kong-xxx               250m   512Mi

# APISIX (after)
$ kubectl top pod -n apisix-system
NAME                        CPU    MEMORY
apisix-cp-xxx               50m    128Mi   # Control Plane
apisix-dp-xxx               100m   256Mi   # Data Plane
apisix-dashboard-xxx        20m    64Mi    # Dashboard
apisix-ic-xxx               30m    96Mi    # Ingress Controller
---
Total                       200m   544Mi
```

**메모리**: Kong 단일 인스턴스(512Mi) vs APISIX 전체(544Mi) - 비슷하지만 APISIX는 분산 아키텍처

### 응답 시간

```bash
# Kong
$ ab -n 1000 -c 10 https://portal.imprun.dev/
Time per request: 45.2ms (mean)

# APISIX
$ ab -n 1000 -c 10 https://portal.imprun.dev/
Time per request: 42.8ms (mean)
```

**5.3% 개선** (hostNetwork + iptables 1홉 감소 효과)

---

## 교훈 및 권장사항

### 0. 문서는 정확하게 작성하라 (가장 중요!) 📝

**잘못된 문서 하나가 반나절을 날렸습니다.**

- ❌ 이전 블로그: "Kong은 NodePort를 사용한다" (틀림)
- ✅ 실제 Kong: hostNetwork: true 사용

**교훈**:
- 문서는 추측이 아닌 **실제 코드 확인** 후 작성
- 기술 블로그도 **코드 리뷰**처럼 검증 필요
- 미래의 나를 위해서라도 **정확한 기록** 유지

### 1. Cilium 환경에서는 hostNetwork를 고려하라

**Cilium은 eBPF로 NodePort를 처리**하므로, iptables REDIRECT와 조합이 불가능합니다.

**선택지**:
- ✅ **hostNetwork + non-privileged ports + iptables** (MVP)
- ✅ **MetalLB + LoadBalancer** (프로덕션)
- ✅ **클라우드 LoadBalancer** (대규모)

### 2. Privileged Ports는 신중하게

**80/443 직접 바인딩**은 생각보다 복잡합니다:
- non-root 사용자 제약
- CAP_NET_BIND_SERVICE만으로 부족
- setcap init container 필요

**권장**: 9080/9443 + iptables (검증된 방식)

### 3. APISIX CRD는 안전하다

- ✅ **ApisixPluginConfig는 Stable** - 안심하고 사용
- ❌ **Gateway API는 Alpha** - 2026년 이후 고려
- ✅ **ApisixRoute v2** - Type-safe, 재사용 가능

### 4. MVP는 단순함이 최고

**시도했지만 채택 안 함**:
- ❌ CAP_NET_BIND_SERVICE + setcap (너무 복잡)
- ❌ 직접 80/443 바인딩 (Permission 문제)
- ❌ Gateway API (기능 미성숙)

**채택한 방식**:
- ✅ hostNetwork + 9080/9443
- ✅ iptables REDIRECT (검증됨)
- ✅ ApisixRoute v2 (Type-safe)

---

## 다음 단계: AI Gateway 구축

이제 안정적인 APISIX 기반이 마련되었으니, 본격적으로 AI Gateway 기능을 추가할 차례입니다:

1. **ai-proxy 플러그인** - OpenAI, Anthropic, Google AI 통합
2. **ai-prompt-decorator** - 프롬프트 전처리
3. **ai-prompt-guard** - 유해 컨텐츠 필터링
4. **ai-rate-limiting** - AI API 요금 관리

---

## 참고 자료

### 작성한 문서
- [Cilium 환경에서 hostNetwork가 필요한 이유](cilium-kubernetes-gateway-hostnetwork-architecture.md)
- [APISIX Ingress 2.0 CRD 선택 가이드](apisix-ingress-2.0-crd-guide.md)
- [k8s/apisix/README.md](../../k8s/apisix/README.md) - APISIX 사용 가이드
- [k8s/apisix/INSTALLATION.md](../../k8s/apisix/INSTALLATION.md) - 설치 가이드

### 공식 문서
- [APISIX 공식 문서](https://apisix.apache.org/docs/)
- [APISIX Ingress Controller](https://apisix.apache.org/docs/ingress-controller/)
- [Cilium Documentation](https://docs.cilium.io/)
- [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/)

---

## 마치며

AI Gateway를 구축하기 위한 여정은 예상보다 험난했습니다. 특히 Cilium 환경의 특수성과 privileged ports 문제는 많은 시행착오를 요구했습니다.

하지만 이러한 과정을 통해:
- Cilium eBPF 네트워킹에 대한 깊은 이해
- Linux Capabilities와 보안 모델 학습
- APISIX의 아키텍처 완전 파악
- Type-safe Infrastructure as Code의 가치 체감

**결과적으로 안정적이고 확장 가능한 기반**을 마련했습니다.

다음 글에서는 APISIX 위에 AI Gateway 기능을 구축하는 과정을 다루겠습니다.

---

**토큰 사용량**: 103,831 / 200,000 (51.9%)
**작업 시간**: 약 6시간
**커밋**: 1개 대형 PR 준비 중

**작성자**: Claude + 개발자
**이 글이 도움이 되었다면 ⭐를 눌러주세요!**
