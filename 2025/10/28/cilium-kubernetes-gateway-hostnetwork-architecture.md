# Cilium 환경에서 API Gateway 배포 시 hostNetwork가 필요한 이유

**작성일**: 2025-10-28
**대상**: Kubernetes + Cilium 환경에서 Kong, APISIX 등 API Gateway 운영자

## 개요

Kubernetes에서 API Gateway(Kong, APISIX 등)를 배포할 때, kube-proxy 대신 **Cilium**을 사용하는 환경에서는 외부 트래픽 라우팅 전략이 달라집니다.

이 문서는 Cilium 환경에서 NodePort + iptables 조합이 작동하지 않는 이유와, 상황별 최적 아키텍처 선택 가이드를 제공합니다.

## 목차

1. [문제 상황: NodePort가 iptables에서 보이지 않음](#문제-상황-nodeport가-iptables에서-보이지-않음)
2. [근본 원인: Cilium의 eBPF 기반 네트워킹](#근본-원인-cilium의-ebpf-기반-네트워킹)
3. [해결 방법: hostNetwork 사용](#해결-방법-hostnetwork-사용)
4. [아키텍처 선택 가이드: MVP vs 프로덕션](#아키텍처-선택-가이드-mvp-vs-프로덕션)
5. [실전 구성 예시](#실전-구성-예시)

---

## 문제 상황: NodePort가 iptables에서 보이지 않음

### 증상

API Gateway를 NodePort로 배포하고, iptables로 80/443 → NodePort 30080/30443 리다이렉트를 설정했지만 연결이 실패합니다.

```bash
# NodePort Service 배포됨
$ kubectl get svc apisix-gateway
NAME             TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
apisix-gateway   NodePort   10.96.123.45    <none>        80:30080/TCP,443:30443/TCP   5m

# iptables 규칙 설정
$ sudo iptables -t nat -A PREROUTING -i enp0s6 -p tcp --dport 80 -j REDIRECT --to-ports 30080
$ sudo iptables -t nat -A PREROUTING -i enp0s6 -p tcp --dport 443 -j REDIRECT --to-ports 30443

# 그러나 연결 실패
$ curl http://localhost:80
curl: (7) Failed to connect to localhost port 80: Connection refused
```

### 혼란의 원인

Cilium CLI로 확인하면 NodePort가 정상적으로 표시됩니다:

```bash
$ cilium service list | grep 30080
10.96.123.45:80    NodePort   30080   ...
```

하지만 **iptables에는 NodePort 바인딩이 전혀 보이지 않습니다**:

```bash
$ sudo netstat -tlnp | grep 30080
# (출력 없음)
```

---

## 근본 원인: Cilium의 eBPF 기반 네트워킹

### kube-proxy vs Cilium 비교

| 구분 | kube-proxy | Cilium |
|-----|-----------|---------|
| **구현 방식** | iptables / IPVS | eBPF (kernel) |
| **NodePort 처리** | iptables NAT 규칙 생성 | eBPF 프로그램으로 처리 |
| **호스트 포트 바인딩** | ✅ 실제 포트 LISTEN | ❌ eBPF 레벨에서만 존재 |
| **iptables 가시성** | ✅ 규칙 확인 가능 | ❌ eBPF는 iptables 우회 |

### Cilium의 NodePort 처리 흐름

```
외부 패킷 → NIC → eBPF XDP/TC Hook → NodePort 매칭 → Pod로 전달
                          ↑
                   iptables는 거치지 않음!
```

Cilium은 커널의 eBPF hook을 통해 패킷을 조기에 가로채어 처리하므로, **iptables PREROUTING 체인에 도달하기 전에 NodePort로 라우팅**됩니다.

### 왜 iptables REDIRECT가 작동하지 않는가?

```
# 의도한 흐름
외부:80 → iptables PREROUTING REDIRECT → 30080 → eBPF → Pod

# 실제 흐름 (Cilium 환경)
외부:80 → iptables PREROUTING (30080 포트는 바인딩 안 됨) → 연결 실패
외부:30080 → eBPF (직접 처리) → Pod (정상 작동)
```

**핵심**: iptables REDIRECT는 **실제로 LISTEN 중인 포트**에만 작동합니다. Cilium의 NodePort는 eBPF 레벨에서만 존재하므로 iptables에서 redirect 대상이 될 수 없습니다.

---

## 해결 방법: hostNetwork 사용

### 솔루션 개요

API Gateway Pod를 **hostNetwork: true**로 배포하면, Pod가 호스트의 네트워크 네임스페이스를 직접 사용합니다. 이렇게 하면 **실제 호스트 포트에 바인딩**되어 iptables REDIRECT가 정상 작동합니다.

### 구성 예시 (APISIX)

```yaml
# apisix-values.yaml
replicaCount: 1

# hostNetwork 활성화
hostNetwork: true
dnsPolicy: ClusterFirstWithHostNet

# 특정 노드에 배포 (단일 노드 바인딩)
nodeSelector:
  kubernetes.io/hostname: gateway-node-01

apisix:
  ssl:
    enabled: true
    containerPort: 9443  # APISIX가 9443 포트에 LISTEN

service:
  type: ClusterIP  # hostNetwork 사용 시 ClusterIP
  http:
    enabled: true
    servicePort: 80
    containerPort: 9080  # APISIX가 9080 포트에 LISTEN
  tls:
    enabled: true
    servicePort: 443
    containerPort: 9443
```

### iptables 설정 (Gateway 노드에서 실행)

```bash
# 80 → 9080, 443 → 9443 리다이렉트
sudo iptables -t nat -A PREROUTING -i enp0s6 -p tcp --dport 80 -j REDIRECT --to-ports 9080
sudo iptables -t nat -A PREROUTING -i enp0s6 -p tcp --dport 443 -j REDIRECT --to-ports 9443

# 규칙 영구 저장
sudo iptables-save | sudo tee /etc/iptables/rules.v4
```

### 검증

```bash
# 1. Pod가 호스트 포트에 바인딩되었는지 확인
$ sudo netstat -tlnp | grep -E ':(9080|9443)'
tcp6  0  0 :::9080   :::*   LISTEN  12345/nginx: master
tcp6  0  0 :::9443   :::*   LISTEN  12345/nginx: master

# 2. 외부 접속 테스트
$ curl http://gateway-node-01:80 -v
< HTTP/1.1 404 Not Found
< Server: APISIX/3.14.1

$ curl https://api.example.com -v
< HTTP/1.1 200 OK
```

---

## 아키텍처 선택 가이드: MVP vs 프로덕션

### 환경별 권장 아키텍처

| 환경 | 트래픽 처리 방식 | 고가용성 | 복잡도 | 비용 |
|-----|---------------|---------|-------|------|
| **MVP / 개발** | hostNetwork + iptables | ❌ (단일 장애점) | ⭐ 낮음 | 무료 |
| **소규모 프로덕션** | MetalLB + LoadBalancer | ✅ (여러 노드) | ⭐⭐ 중간 | 무료 (온프레미스) |
| **대규모 프로덕션** | 클라우드 LoadBalancer | ✅ (관리형) | ⭐ 낮음 | 💰 유료 |

### 1. MVP / 개발 환경: hostNetwork + iptables

**사용 시나리오**:
- 단일 서버 또는 소규모 클러스터
- 트래픽이 낮고 downtime 허용 가능
- 비용 절감이 최우선

**장점**:
- ✅ 추가 인프라 불필요 (0원)
- ✅ 설정 간단 (iptables 규칙만)
- ✅ 낮은 지연시간 (네트워크 홉 최소화)

**단점**:
- ❌ **단일 장애점**: Gateway 노드 다운 시 전체 서비스 중단
- ❌ **수평 확장 불가**: 단일 노드에만 바인딩
- ❌ **롤링 업데이트 불가**: Pod 재시작 시 downtime 발생

**구성도**:
```
인터넷
  ↓ 80/443
[Gateway 노드] iptables REDIRECT → 9080/9443 → APISIX Pod (hostNetwork)
                                         ↓
                                    Backend Pods
```

### 2. 소규모 프로덕션: MetalLB + LoadBalancer

**사용 시나리오**:
- 온프레미스 환경
- 고가용성 필요
- 클라우드 비용 부담

**장점**:
- ✅ **고가용성**: 여러 노드에 Pod 분산
- ✅ **롤링 업데이트**: 무중단 배포 가능
- ✅ **수평 확장**: replica 증가로 트래픽 분산

**단점**:
- ⚠️ MetalLB 설치 및 관리 필요
- ⚠️ BGP 또는 L2 모드 네트워크 설정 필요

**구성도**:
```
인터넷
  ↓ 80/443
[MetalLB VIP: 192.168.1.100]
  ↓ ↓ ↓
[Node1] [Node2] [Node3]
  ↓       ↓       ↓
APISIX  APISIX  APISIX (replicas: 3)
  ↓       ↓       ↓
     Backend Pods
```

**MetalLB 설정 예시**:

```yaml
# metallb-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.1.100-192.168.1.110
```

```yaml
# apisix-values.yaml (LoadBalancer 모드)
replicaCount: 3

hostNetwork: false  # LoadBalancer 사용 시 불필요

podAntiAffinity:  # 여러 노드에 분산
  preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app.kubernetes.io/name: apisix
        topologyKey: kubernetes.io/hostname

service:
  type: LoadBalancer
  loadBalancerIP: 192.168.1.100  # MetalLB VIP
  http:
    enabled: true
    servicePort: 80
    containerPort: 9080
  tls:
    enabled: true
    servicePort: 443
    containerPort: 9443
```

### 3. 대규모 프로덕션: 클라우드 LoadBalancer

**사용 시나리오**:
- AWS, GCP, Azure 등 클라우드 환경
- 글로벌 트래픽 처리
- 관리 부담 최소화

**장점**:
- ✅ **완전 관리형**: 헬스체크, 오토스케일링 자동
- ✅ **글로벌 분산**: CDN, DDoS 방어 등 추가 기능
- ✅ **멀티 AZ 지원**: 높은 가용성

**단점**:
- 💰 **비용**: 시간당 과금 ($15-50/month)
- 🔒 **벤더 종속**: 클라우드 제공자에 의존

**구성도**:
```
인터넷
  ↓
[AWS ALB / GCP GLB] (관리형 LoadBalancer)
  ↓ ↓ ↓
[AZ-1]  [AZ-2]  [AZ-3]
  ↓       ↓       ↓
APISIX  APISIX  APISIX (replicas: 6+)
  ↓       ↓       ↓
     Backend Pods
```

**Kubernetes 설정**:

```yaml
# apisix-values.yaml (클라우드 LoadBalancer)
replicaCount: 6

service:
  type: LoadBalancer
  annotations:
    # AWS ALB
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"

    # GCP
    # cloud.google.com/load-balancer-type: "External"

    # Azure
    # service.beta.kubernetes.io/azure-load-balancer-internal: "false"

  http:
    enabled: true
    servicePort: 80
    containerPort: 9080
  tls:
    enabled: true
    servicePort: 443
    containerPort: 9443
```

---

## 실전 구성 예시

### Kong 예시 (hostNetwork 모드)

실제 운영 중인 Kong 구성을 확인하면 hostNetwork: true를 사용합니다:

```bash
$ kubectl get deployment kong-kong -n kong -o yaml | grep -A 5 hostNetwork
hostNetwork: true
dnsPolicy: ClusterFirstWithHostNet
nodeSelector:
  kubernetes.io/hostname: gateway-node-01
```

```bash
$ kubectl get pods -n kong -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP             NODE
kong-kong-5d7c8c9b-x7p9q    1/1     Running   0          10d   192.168.1.10   gateway-node-01
```

### APISIX 예시 (hostNetwork 모드)

```yaml
# k8s/apisix/dp-values.yaml
replicaCount: 1

hostNetwork: true
dnsPolicy: ClusterFirstWithHostNet

nodeSelector:
  kubernetes.io/hostname: gateway-node-01

apisix:
  deployment:
    mode: decoupled
    role: data_plane

  ssl:
    enabled: true
    containerPort: 9443

service:
  type: ClusterIP
  http:
    enabled: true
    servicePort: 80
    containerPort: 9080
  tls:
    enabled: true
    servicePort: 443
    containerPort: 9443
```

```bash
# Gateway 노드 iptables 설정
sudo iptables -t nat -A PREROUTING -i enp0s6 -p tcp --dport 80 -j REDIRECT --to-ports 9080
sudo iptables -t nat -A PREROUTING -i enp0s6 -p tcp --dport 443 -j REDIRECT --to-ports 9443
sudo iptables-save | sudo tee /etc/iptables/rules.v4
```

---

## 주요 결론

### Cilium 환경에서 알아야 할 것

1. **NodePort + iptables 조합은 작동하지 않음**
   - Cilium은 eBPF로 NodePort를 처리하므로 iptables에서 보이지 않음
   - iptables REDIRECT는 실제로 LISTEN 중인 포트에만 작동

2. **MVP는 hostNetwork, 프로덕션은 LoadBalancer**
   - 개발/테스트: hostNetwork + iptables (간단, 저비용, 낮은 가용성)
   - 소규모: MetalLB (고가용성, 온프레미스)
   - 대규모: 클라우드 LoadBalancer (관리형, 고비용)

3. **hostNetwork의 제약사항**
   - 단일 노드에만 배포 가능 (포트 충돌 방지)
   - Pod 재시작 시 downtime 발생
   - 롤링 업데이트 불가

### 마이그레이션 경로

```
단계 1: MVP (hostNetwork)
  ↓ 트래픽 증가
단계 2: 온프레미스 프로덕션 (MetalLB)
  ↓ 글로벌 확장
단계 3: 클라우드 프로덕션 (AWS ALB / GCP GLB)
```

---

## 참고 자료

- [Cilium Service Mesh](https://docs.cilium.io/en/stable/network/servicemesh/)
- [Kubernetes hostNetwork Documentation](https://kubernetes.io/docs/concepts/policy/pod-security-policy/#host-namespaces)
- [MetalLB Installation Guide](https://metallb.universe.tf/installation/)
- [APISIX Deployment Modes](https://apisix.apache.org/docs/apisix/deployment-modes/)

---

**작성자 노트**: 이 문서는 실제 APISIX 배포 과정에서 겪은 시행착오를 바탕으로 작성되었습니다. Cilium 환경에서 "왜 NodePort가 안 보이지?"라는 질문으로 시작해, hostNetwork 방식으로 해결한 경험을 공유합니다.
