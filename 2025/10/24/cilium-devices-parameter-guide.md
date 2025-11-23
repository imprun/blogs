# Cilium devices 파라미터 완벽 가이드: Tailscale 환경에서의 핵심

> **작성일**: 2025-10-24
> **환경**: Kubernetes 1.34, Cilium 1.18.2, Tailscale/Headscale, Oracle Cloud
> **난이도**: 고급
> **핵심**: devices 파라미터 하나가 모든 네트워크 통신을 좌우한다

---

## 들어가며

[**imprun.dev**](https://imprun.dev)는 Kubernetes 기반 서버리스 Cloud Function 플랫폼입니다. Tailscale 메시 네트워크 위에 Cilium CNI를 구축하면서, **Pod 간 통신은 되는데 외부 통신이 안 되는** 기이한 문제에 직면했습니다.

**우리가 겪은 문제**:
- ✅ Pod 간 통신 정상 (10.244.x.x)
- ❌ **외부 인터넷 통신 불가** (DNS 조회 실패, apt update 실패)
- ❌ `devices=enp0s6` (물리 인터페이스)만 지정했을 때
- ✅ `devices=enp0s6,tailscale0` (둘 다 지정)으로 해결

**핵심 발견**:
- Cilium의 `devices` 파라미터는 **eBPF를 적용할 인터페이스를 지정**
- **외부 통신**은 물리 인터페이스(enp0s6)를 통해 나감
- **노드 간 Pod 통신**은 Tailscale 인터페이스(tailscale0)를 통해 전달됨
- **두 인터페이스 모두 지정해야** 완전한 네트워크 동작

이 글은 **imprun.dev 플랫폼 구축 경험**을 바탕으로, Tailscale 환경에서 Cilium의 `devices` 파라미터가 실제로 무엇을 제어하는지 완벽히 분석합니다.

---

## 🎯 이 문서를 읽어야 하는 이유

**"Cilium 설치했는데 Pod 간 통신은 되는데 외부 통신이 안 돼요"**
**"외부 통신은 되는데 노드 간 Pod 통신이 안 돼요"**
**"왜 `devices=enp0s6` 하나만 지정하면 안 되나요?"**

이 문서는 **Tailscale/Headscale 환경에서 Cilium의 `devices` 파라미터가 실제로 무엇을 제어하는지**, 그리고 **왜 두 개의 인터페이스가 모두 필요한지**를 완벽히 설명합니다.

---

## 📋 목차

1. [배경: 서로 다른 네트워크의 Kubernetes 클러스터](#1-배경-서로-다른-네트워크의-kubernetes-클러스터)
2. [devices 파라미터란 무엇인가](#2-devices-파라미터란-무엇인가)
3. [시나리오별 증상 분석](#3-시나리오별-증상-분석)
4. [근본 원리: Cilium의 Interface Selection](#4-근본-원리-ciliums-interface-selection)
5. [최종 권장사항](#5-최종-권장사항)
6. [트러블슈팅](#6-트러블슈팅)

---

## 1. 배경: 서로 다른 네트워크의 Kubernetes 클러스터

### 1.1 환경 구성

```
Oracle Cloud 계정 A              Oracle Cloud 계정 B
┌─────────────────────┐         ┌─────────────────────┐
│ Master Node         │         │ Worker Node         │
│ VCN: 10.0.0.239/24  │         │ VCN: 10.0.0.24/24   │
│ Tailscale: 100.64.0.1│◄───────►│ Tailscale: 100.64.0.2│
└─────────────────────┘         └─────────────────────┘
        ▲                               ▲
        │ enp0s6                        │ enp0s6
        │                               │
        ▼                               ▼
  Internet (NAT)                  Internet (NAT)
```

**핵심 제약사항**:
- 서로 다른 Oracle 계정 → **10.0.0.x 대역으로 직접 통신 불가**
- Tailscale mesh network → **100.64.0.x 대역으로만 통신 가능**
- 외부 인터넷 접근 → **Oracle VCN Gateway (enp0s6) 경유 필수**

### 1.2 네트워크 인터페이스

각 노드는 **두 개의 네트워크 인터페이스**를 가집니다:

```bash
# enp0s6: Oracle VCN Physical NIC
2: enp0s6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000
    inet 10.0.0.239/24 brd 10.0.0.255 scope global
    # 역할: 외부 인터넷 통신 (Oracle VCN Gateway 경유)

# tailscale0: Tailscale VPN Interface
3: tailscale0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1280
    inet 100.64.0.1/32 scope global
    # 역할: 노드 간 통신 (WireGuard 터널)
```

---

## 2. devices 파라미터란 무엇인가

### 2.1 공식 정의

Cilium 공식 문서:
> "The `devices` option specifies which network interfaces Cilium uses for NodePort and LoadBalancer services."

**하지만 실제로는**:
1. **노드 IP 결정**에 영향
2. **Masquerade (SNAT) 인터페이스** 선택
3. **VXLAN 터널 엔드포인트** IP 결정

### 2.2 설정 방법

```yaml
# Helm values.yaml
devices: "enp0s6,tailscale0"  # 문자열로 묶기

# 또는 리스트 형식
devices:
  - enp0s6
  - tailscale0

# Helm CLI
--set devices="enp0s6,tailscale0"
--set devices="{enp0s6,tailscale0}"  # 중괄호 + 따옴표
```

---

## 3. 시나리오별 증상 분석

### 📊 비교표

| devices 설정 | Cilium Node IP | Pod → Pod (노드 간) | Pod → 외부 (인터넷) | 증상 |
|-------------|---------------|-------------------|------------------|------|
| **생략** (auto) | ✅ 100.64.0.1 | ✅ 정상 | ✅ 정상 | 완벽 작동 |
| **enp0s6,tailscale0** | ✅ 100.64.0.1 | ✅ 정상 | ✅ 정상 | 완벽 작동 (명시적) |
| **enp0s6만** | ❌ 10.0.0.239 | ❌ 실패 | ✅ 정상 | 노드 간 통신 불가 |
| **tailscale0만** | ✅ 100.64.0.1 | ✅ 정상 | ❌ 실패 | 외부 통신 불가 |

---

## 3.1 시나리오 A: devices=enp0s6 (한 개만)

### 설정

```yaml
devices: enp0s6  # ❌ 잘못된 설정
```

### 증상

```bash
# ❌ Cilium이 enp0s6 IP를 노드 IP로 오인
kubectl -n kube-system exec ds/cilium -- cilium node list
Name                     IPv4 Address   Endpoint CIDR
instance-master          10.0.0.239     10.244.0.0/24  ← 잘못됨!
instance-worker          10.0.0.24      10.244.1.0/24  ← 잘못됨!

# ❌ Pod 간 통신 실패
kubectl run test-master --image=busybox --restart=Never -- sleep 3600
kubectl run test-worker --image=busybox --restart=Never -- sleep 3600

kubectl exec test-master -- ping 10.244.1.x
# ping: bad address '10.244.1.x'
# 또는
# PING 10.244.1.10 (10.244.1.10): 56 data bytes
# (timeout... no response)

# ✅ 외부 통신은 정상
kubectl exec test-master -- ping -c 3 8.8.8.8
# 64 bytes from 8.8.8.8: seq=0 ttl=54 time=... ✅
```

### 원인 분석

**Cilium의 동작**:
```
1. devices=enp0s6 설정 확인
2. enp0s6 인터페이스만 스캔
3. enp0s6의 IP (10.0.0.239) 감지
4. 노드 IP를 10.0.0.239로 설정 (kubelet --node-ip 무시!)
5. VXLAN 터널 엔드포인트도 10.0.0.239 사용
```

**패킷 흐름**:
```
Master Node Pod → Worker Node Pod 통신 시도:

[Master - 10.0.0.239]
  Pod A (10.244.0.5)
    │ dst: 10.244.1.10
    ↓
  Cilium VXLAN 캡슐화
    │ Outer Header:
    │   src: 10.0.0.239
    │   dst: 10.0.0.24 (Worker enp0s6 IP)
    ↓
  enp0s6 인터페이스로 송신 시도
    ↓
  ❌ Oracle VCN 라우팅 실패
     "10.0.0.24는 다른 계정/VPC입니다"
    ↓
  DROP!
```

### 실제 에러 로그

```bash
# Cilium agent 로그
kubectl -n kube-system logs ds/cilium | grep -i error
# level=warning msg="Unable to reach node" node=instance-worker ip=10.0.0.24
# level=error msg="VXLAN tunnel endpoint unreachable" remote=10.0.0.24

# tcpdump로 패킷 확인
tcpdump -i enp0s6 host 10.0.0.24
# (패킷 나가지만 응답 없음)
```

---

## 3.2 시나리오 B: devices=tailscale0 (한 개만)

### 설정

```yaml
devices: tailscale0  # ❌ 잘못된 설정
```

### 증상

```bash
# ✅ Cilium이 Tailscale IP를 노드 IP로 인식 (정상)
kubectl -n kube-system exec ds/cilium -- cilium node list
Name                     IPv4 Address   Endpoint CIDR
instance-master          100.64.0.1     10.244.0.0/24  ← 정상!
instance-worker          100.64.0.2     10.244.1.0/24  ← 정상!

# ✅ Pod 간 통신 정상
kubectl exec test-master -- ping 10.244.1.10
# 64 bytes from 10.244.1.10: seq=0 ttl=62 time=... ✅

# ❌ 외부 통신 실패
kubectl exec test-master -- ping -c 3 8.8.8.8
# PING 8.8.8.8 (8.8.8.8): 56 data bytes
# (timeout... no response)

# ❌ DNS도 실패
kubectl exec test-master -- nslookup google.com
# ;; connection timed out; no servers could be reached

# ❌ CoreDNS 재시작 반복
kubectl get pods -n kube-system -l k8s-app=kube-dns
NAME                       READY   STATUS    RESTARTS   AGE
coredns-xxx                0/1     Running   5          2m  ← 계속 재시작!
coredns-yyy                1/1     Running   0          2m
```

### 원인 분석

**Cilium의 동작**:
```
1. devices=tailscale0 설정 확인
2. tailscale0 인터페이스만 스캔
3. Masquerade도 tailscale0 사용
4. 외부 트래픽 SNAT: 100.64.0.1
5. Tailscale은 Peer-to-Peer VPN
   → 8.8.8.8은 Peer가 아님
   → 라우팅 경로 없음 → DROP
```

**패킷 흐름**:
```
Pod → 8.8.8.8 (Google DNS) 통신 시도:

[Master Node]
  Pod A (10.244.0.5)
    │ dst: 8.8.8.8
    ↓
  Cilium eBPF Masquerade
    │ SNAT: 100.64.0.1 (tailscale0 IP)
    ↓
  tailscale0 인터페이스로 송신
    ↓
  Tailscale 라우팅 테이블 확인:
    Peer 목록: [100.64.0.1, 100.64.0.2]
    8.8.8.8: ??? (Peer 아님)
    ↓
  ❌ DROP! (No route to host)
```

### 실제 에러 로그

```bash
# CoreDNS 로그
kubectl logs -n kube-system coredns-xxx
# [ERROR] plugin/errors: 2 google.com. A: read udp 10.244.0.x:xxxxx->8.8.8.8:53: i/o timeout

# Cilium BPF NAT 테이블 확인
kubectl -n kube-system exec ds/cilium -- cilium bpf nat list | grep 8.8.8.8
# UDP OUT 10.244.0.x:53 -> 8.8.8.8:53 XLATE_SRC 100.64.0.1:53
# (SNAT은 되지만 응답이 안 옴)

# Tailscale 상태 확인
tailscale status
# 100.64.0.1   master    ...
# 100.64.0.2   worker    ...
# (8.8.8.8은 목록에 없음)
```

---

## 3.3 시나리오 C: devices=enp0s6,tailscale0 (둘 다) ✅

### 설정

```yaml
devices: "enp0s6,tailscale0"  # ✅ 올바른 설정
# 또는 생략 (auto-detect)
```

### 증상

```bash
# ✅ Cilium이 두 인터페이스 모두 감지
kubectl -n kube-system exec ds/cilium -- cilium status --verbose | grep Masq
Masquerading: BPF [enp0s6, tailscale0]  10.244.0.0/24  [IPv4: Enabled]

# ✅ 노드 IP (Tailscale)
kubectl -n kube-system exec ds/cilium -- cilium node list
Name                     IPv4 Address   Endpoint CIDR
instance-master          100.64.0.1     10.244.0.0/24  ✅
instance-worker          100.64.0.2     10.244.1.0/24  ✅

# ✅ Pod 간 통신 정상
kubectl exec test-master -- ping -c 3 10.244.1.10
# 64 bytes from 10.244.1.10: seq=0 ttl=62 time=2.1 ms ✅

# ✅ 외부 통신 정상
kubectl exec test-master -- ping -c 3 8.8.8.8
# 64 bytes from 8.8.8.8: seq=0 ttl=54 time=1.8 ms ✅

# ✅ DNS 정상
kubectl exec test-master -- nslookup google.com
# Server:    10.96.0.10
# Address:   10.96.0.10:53
# Name:      google.com
# Address:   142.250.x.x ✅

# ✅ HTTPS 통신 정상
kubectl exec test-master -- wget -qO- https://api.ipify.org
# 168.107.2.60  ← Oracle Cloud Public IP ✅
```

### 원인 분석

**Cilium의 동작**:
```
1. devices=[enp0s6, tailscale0] 설정 확인
2. 두 인터페이스 모두 스캔
3. 노드 IP: kubelet --node-ip (100.64.0.1) 우선
4. Masquerade: 트래픽 목적지에 따라 자동 선택
   - Pod CIDR (10.244.x.x) → Node IP 경로 → tailscale0
   - 외부 (8.8.8.8) → Default gateway 경로 → enp0s6
```

### 패킷 흐름

#### Pod → Pod (노드 간 통신)

```
[Master Node - 100.64.0.1]
  Pod A (10.244.0.5)
    │ dst: 10.244.1.10 (Worker Pod)
    ↓
  Cilium VXLAN 캡슐화
    │ Outer Header:
    │   src: 100.64.0.1 (Master Tailscale IP)
    │   dst: 100.64.0.2 (Worker Tailscale IP)
    ↓
  커널 라우팅 테이블 확인:
    ip route | grep 100.64.0.2
    → 100.64.0.0/10 dev tailscale0
    ↓
  tailscale0 인터페이스로 송신
    ↓ WireGuard 암호화 터널
  [Worker Node - 100.64.0.2]
    tailscale0 수신
    ↓ VXLAN 디캡슐화
  Pod B (10.244.1.10) ✅
```

#### Pod → 외부 (인터넷)

```
[Master Node]
  Pod A (10.244.0.5)
    │ dst: 8.8.8.8
    ↓
  Cilium eBPF Masquerade
    │ 인터페이스 선택 로직:
    │ - 8.8.8.8은 Pod CIDR 아님
    │ - Default route 확인: dev enp0s6
    │ - enp0s6 IP 사용
    │
    │ SNAT: 10.0.0.239 (enp0s6 IP)
    ↓
  enp0s6 인터페이스로 송신
    ↓
  Oracle VCN Gateway
    │ NAT: 10.0.0.239 → 168.107.2.60 (Public IP)
    ↓
  Internet ✅
```

### BPF NAT 테이블 확인

```bash
kubectl -n kube-system exec ds/cilium -- cilium bpf nat list | head -20

# Pod → 외부 트래픽
ICMP OUT 10.244.0.62:7 -> 8.8.8.8:0 XLATE_SRC 10.0.0.239:7
                                               ^^^^^^^^^^^ enp0s6 IP!

# 응답 패킷
ICMP IN 8.8.8.8:0 -> 10.0.0.239:7 XLATE_DST 10.244.0.62:7

# Pod → Pod 트래픽 (VXLAN)
TCP OUT 100.64.0.1:51074 -> 100.64.0.2:4240 XLATE_SRC 100.64.0.1:51074
        ^^^^^^^^^             ^^^^^^^^^^^ Tailscale IPs!
```

---

## 4. 근본 원리: Cilium의 Interface Selection

### 4.1 devices 파라미터의 실제 역할

```yaml
devices: "interface1,interface2,..."

# Cilium이 이 정보로 결정하는 것:
1. 노드 IP 감지 우선순위
2. Masquerade (SNAT) 인터페이스 pool
3. NodePort/LoadBalancer 바인딩 인터페이스
4. Direct routing device (Native mode)
```

### 4.2 Masquerade Interface Selection Logic

**Cilium 내부 로직 (의사 코드)**:

```python
def select_masquerade_interface(packet):
    dst_ip = packet.destination

    # 1. Pod CIDR 범위 체크
    if dst_ip in all_pod_cidr_ranges:
        # Pod 간 통신: Node IP를 찾아서 해당 인터페이스 사용
        target_node_ip = lookup_node_ip_for_pod(dst_ip)
        return find_interface_for_ip(target_node_ip)
        # 예: 100.64.0.2 → tailscale0

    # 2. Service CIDR 체크
    elif dst_ip in service_cidr:
        # eBPF load balancing, SNAT 필요 없음
        return None

    # 3. 외부 트래픽
    else:
        # Default gateway 경로의 인터페이스 사용
        default_route = get_default_route()  # ip route | grep default
        return default_route.interface
        # 예: enp0s6
```

### 4.3 Node IP Detection Priority

```
1. devices 파라미터에 명시된 인터페이스들 스캔
2. 각 인터페이스의 IP 주소 수집
3. kubelet --node-ip와 매칭되는 IP 찾기
4. 매칭되는 인터페이스를 주 인터페이스로 선택
5. 해당 인터페이스를 VXLAN 터널 엔드포인트로 사용
```

**예시**:

```bash
# kubelet 설정
KUBELET_EXTRA_ARGS=--node-ip=100.64.0.1

# devices 설정에 따른 동작:

# Case 1: devices=enp0s6
# - enp0s6 IP: 10.0.0.239
# - 100.64.0.1과 매칭 안 됨
# - enp0s6 IP를 강제로 노드 IP로 사용 (잘못됨!) ❌

# Case 2: devices=tailscale0
# - tailscale0 IP: 100.64.0.1
# - 100.64.0.1과 매칭됨
# - tailscale0를 노드 IP로 사용 (정상) ✅
# - 하지만 Masquerade도 tailscale0만 사용 (외부 통신 실패) ❌

# Case 3: devices=enp0s6,tailscale0
# - tailscale0 IP: 100.64.0.1 ← 매칭!
# - 노드 IP: 100.64.0.1 사용 (정상) ✅
# - Masquerade: 두 인터페이스 모두 사용 가능
#   → 트래픽에 따라 자동 선택 ✅
```

### 4.4 Auto-Detection 메커니즘

**devices를 생략하면**:

```bash
# Cilium이 자동으로:
1. 모든 네트워크 인터페이스 스캔
   - loopback 제외
   - down 상태 제외

2. 각 인터페이스 특성 분석
   - IP 주소
   - Default route 여부
   - MTU

3. 노드 IP 매칭
   - kubelet --node-ip와 매칭되는 인터페이스 찾기

4. 결과
   - 찾은 인터페이스들을 devices로 설정
   - 예: devices=[enp0s6, tailscale0]
```

**실제 감지 결과 확인**:

```bash
kubectl -n kube-system exec ds/cilium -- cilium status --verbose | grep -A 2 Masq

# devices 생략 시 출력:
Masquerading: BPF [enp0s6, tailscale0]
              ^^^  ^^^^^^^^^^^^^^^^^^^^
              자동 감지된 인터페이스들!
```

---

## 5. 최종 권장사항

### 5.1 옵션 비교

| 방식 | 장점 | 단점 | 추천도 |
|-----|------|------|--------|
| **명시적 지정**<br/>`devices: "enp0s6,tailscale0"` | • 명확한 의도<br/>• 예측 가능<br/>• 문서화 용이 | • 환경 변경 시 수정 필요 | ⭐⭐⭐⭐⭐ |
| **생략 (auto)**<br/>`# devices 없음` | • 유연성 높음<br/>• 환경 변화 대응 | • 예측 어려움<br/>• 디버깅 복잡 | ⭐⭐⭐⭐ |
| **와일드카드**<br/>`devices: "enp+,tailscale+"` | • 인터페이스 이름 변화 대응 | • 의도하지 않은 인터페이스 포함 가능 | ⭐⭐⭐ |

### 5.2 권장 설정 (Tailscale 환경)

```yaml
# Helm values.yaml
devices: "enp0s6,tailscale0"

# 또는 Helm CLI
helm install cilium cilium/cilium \
  --version 1.18.2 \
  --namespace kube-system \
  --set devices="enp0s6,tailscale0" \
  --set routingMode=tunnel \
  --set tunnelProtocol=vxlan \
  --set kubeProxyReplacement=true \
  --set bpf.masquerade=true \
  --set enableIPv4Masquerade=true \
  --set autoDirectNodeRoutes=false
```

**필수 조건**:
```bash
# 1. kubelet이 Tailscale IP로 node-ip 설정되어 있어야 함
cat /etc/sysconfig/kubelet  # 또는 /etc/default/kubelet
# KUBELET_EXTRA_ARGS=--node-ip=100.64.0.1

# 2. Tailscale이 정상 작동 중
tailscale status

# 3. Default route가 enp0s6를 통하도록 설정
ip route | grep default
# default via 10.0.0.1 dev enp0s6
```

### 5.3 검증 방법

```bash
# 1. Cilium 노드 IP 확인
kubectl -n kube-system exec ds/cilium -- cilium node list
# IPv4 Address가 100.64.0.x여야 함 ✅

# 2. Masquerade 인터페이스 확인
kubectl -n kube-system exec ds/cilium -- cilium status --verbose | grep Masq
# Masquerading: BPF [enp0s6, tailscale0] ✅

# 3. Pod 간 통신 테스트
kubectl run test-a --image=busybox --restart=Never -- sleep 3600
kubectl run test-b --image=busybox --restart=Never -- sleep 3600
kubectl exec test-a -- ping -c 3 $(kubectl get pod test-b -o jsonpath='{.status.podIP}')
# 64 bytes from ... ✅

# 4. 외부 통신 테스트
kubectl exec test-a -- ping -c 3 8.8.8.8
# 64 bytes from 8.8.8.8 ✅

# 5. 실제 Public IP 확인
kubectl exec test-a -- wget -qO- https://api.ipify.org
# 168.107.x.x (Oracle Cloud Public IP) ✅
```

---

## 6. 트러블슈팅

### 6.1 Pod 간 통신 안 됨

**증상**:
```bash
kubectl exec test-pod-a -- ping 10.244.1.x
# timeout 또는 network unreachable
```

**원인**: Cilium Node IP가 잘못 설정됨

**해결**:
```bash
# 1. Cilium Node IP 확인
kubectl -n kube-system exec ds/cilium -- cilium node list

# 2. 10.0.0.x로 표시되면 devices 설정 문제
# devices=enp0s6만 지정했거나, kubelet --node-ip 미설정

# 3. 해결책
# Option A: devices에 tailscale0 추가
helm upgrade cilium cilium/cilium \
  --reuse-values \
  --set devices="enp0s6,tailscale0"

# Option B: devices 생략
helm upgrade cilium cilium/cilium \
  --reuse-values \
  --set devices=null

# 4. Cilium pod 재시작
kubectl -n kube-system delete pod -l k8s-app=cilium
```

### 6.2 외부 통신 안 됨

**증상**:
```bash
kubectl exec test-pod -- ping 8.8.8.8
# timeout
```

**원인**: Masquerade가 tailscale0만 사용

**해결**:
```bash
# 1. Masquerade 인터페이스 확인
kubectl -n kube-system exec ds/cilium -- cilium status | grep Masq

# 2. [tailscale0]만 표시되면 devices 설정 문제

# 3. 해결책: devices에 enp0s6 추가
helm upgrade cilium cilium/cilium \
  --reuse-values \
  --set devices="enp0s6,tailscale0"

# 4. BPF NAT 테이블 확인
kubectl -n kube-system exec ds/cilium -- cilium bpf nat list | grep 8.8.8.8
# XLATE_SRC가 10.0.0.x (enp0s6 IP)여야 함 ✅
```

### 6.3 CoreDNS 재시작 반복

**증상**:
```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
NAME                       READY   STATUS    RESTARTS
coredns-xxx                0/1     Running   5  ← 계속 재시작
```

**원인**: CoreDNS가 외부 DNS (8.8.8.8) 접근 불가

**해결**:
```bash
# 외부 통신 문제 해결 (위 6.2 참고)

# CoreDNS 로그 확인
kubectl logs -n kube-system coredns-xxx
# [ERROR] ... read udp ... 8.8.8.8:53: i/o timeout

# 해결 후 재시작
kubectl rollout restart deployment/coredns -n kube-system
```

### 6.4 devices 설정이 적용 안 됨

**증상**:
```bash
# devices 변경했는데 이전 설정 그대로
kubectl -n kube-system exec ds/cilium -- cilium status | grep Masq
# Masquerading: BPF [old_interface]
```

**해결**:
```bash
# 1. ConfigMap 확인
kubectl -n kube-system get cm cilium-config -o yaml | grep device

# 2. ConfigMap에 반영되었는데도 안 바뀌면
# Cilium pod를 완전히 재시작
kubectl -n kube-system delete pod -l k8s-app=cilium

# 3. 또는 Helm으로 완전 재설치
helm uninstall cilium -n kube-system
helm install cilium cilium/cilium -f values.yaml
```

---

## 7. 요약

### 7.1 핵심 원칙

```
┌─────────────────────────────────────────────────────────┐
│  Tailscale 환경에서는 두 인터페이스가 모두 필요합니다!   │
│                                                         │
│  devices: "enp0s6,tailscale0"                           │
│            ^^^^^^  ^^^^^^^^^^                           │
│            외부      노드 간                              │
│            통신      통신                                │
└─────────────────────────────────────────────────────────┘
```

### 7.2 트래픽별 인터페이스 매핑

| 트래픽 유형 | 사용 인터페이스 | SNAT IP | 이유 |
|------------|---------------|---------|------|
| Pod → Pod (같은 노드) | cilium_host | - | 로컬 라우팅 |
| Pod → Pod (다른 노드) | tailscale0 | - | VXLAN over Tailscale |
| Pod → Service | eBPF | - | Load balancing |
| Pod → 외부 | enp0s6 | 10.0.0.x | Oracle VCN Gateway |

### 7.3 잘못된 설정과 결과

```yaml
# ❌ 잘못된 설정 1
devices: enp0s6
결과: Pod 간 통신 불가 (노드 IP가 10.0.0.x로 설정됨)

# ❌ 잘못된 설정 2
devices: tailscale0
결과: 외부 통신 불가 (SNAT IP가 100.64.0.x)

# ✅ 올바른 설정
devices: "enp0s6,tailscale0"
결과: 모든 통신 정상!
```

---

## 8. 참고 자료

### 관련 문서
- [Oracle Cloud Kubernetes 완벽 가이드](./oracle-kubernetes-complete-guide.md)
- [Tailscale + Cilium Hybrid Routing](./tailscale-cilium-hybrid-routing.md)
- [Cilium 공식 문서: Kube-Proxy Free](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/)

### 공식 문서
- [Cilium Network Device Configuration](https://docs.cilium.io/en/stable/operations/performance/tuning/)
- [Tailscale Subnet Routes](https://tailscale.com/kb/1019/subnets/)

---

**작성자**: Claude (AI Assistant)
**최종 수정**: 2025-10-24
**버전**: 1.0

> 💡 **Tip**: 이 문서를 다른 Tailscale + Kubernetes 환경에도 적용할 수 있습니다. 인터페이스 이름만 환경에 맞게 변경하세요.
