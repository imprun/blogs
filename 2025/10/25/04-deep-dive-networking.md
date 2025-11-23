# 4단계: 네트워킹 심화 이해

> **시리즈**: [Oracle Cloud + Tailscale + Kubernetes 완벽 가이드](README.md)
> ← **이전**: [3단계: Kubernetes + Cilium 구축](03-setup-kubernetes-cilium.md) | **다음**: [5단계: 실전 경험과 교훈](05-lessons-learned.md) →

---

> Tailscale + Cilium 네트워킹 스택의 동작 원리 완벽 분석

## 📋 이 문서의 목적

Kubernetes 클러스터가 정상 동작한다면, 이제 **왜** 이렇게 구성했는지 이해할 차례입니다.
패킷이 어떻게 흐르는지, 각 구성요소가 어떤 역할을 하는지 깊이 있게 살펴봅니다.

## 🏗️ 전체 네트워킹 스택

```
┌──────────────────────────────────────────────┐
│              Application Layer                │
│                  (Pod/Container)              │
├──────────────────────────────────────────────┤
│             Cilium eBPF (CNI)                │
│         • kube-proxy replacement              │
│         • VXLAN encapsulation                 │
│         • Service load balancing              │
├──────────────────────────────────────────────┤
│          Tailscale (WireGuard VPN)           │
│         • Peer-to-peer mesh                   │
│         • NAT traversal                       │
├──────────────────────────────────────────────┤
│           Linux Network Stack                 │
│         • netfilter/iptables                  │
│         • routing tables                      │
├──────────────────────────────────────────────┤
│          Physical Network (Oracle VCN)        │
│         • Public IP                           │
│         • Security Lists                      │
└──────────────────────────────────────────────┘
```

## 🔄 Phase 1: Tailscale 네트워킹

### WireGuard 터널링

Tailscale은 WireGuard 프로토콜을 사용하여 암호화 터널을 생성합니다:

```
Node A (100.64.0.1)          Node B (100.64.0.2)
       │                            │
       │   WireGuard Handshake     │
       │<─────────────────────────>│
       │                            │
       │   Encrypted UDP Tunnel    │
       │   (Port 41641)            │
       │<═════════════════════════>│
```

### Tailscale 인터페이스 분석

```bash
# Tailscale 인터페이스 정보
ip addr show tailscale0
# 3: tailscale0: <POINTOPOINT,MULTICAST,NOARP,UP>
#     inet 100.64.0.1/32 scope global tailscale0

# 라우팅 테이블
ip route | grep tailscale
# 100.64.0.2 dev tailscale0 scope link
# 100.64.0.3 dev tailscale0 scope link
```

**특징**:
- **Point-to-Point**: 각 피어와 직접 연결
- **/32 주소**: 단일 IP만 할당
- **동적 라우팅**: 피어 추가/제거 시 자동 업데이트

### NAT Traversal (STUN/DERP)

```
실제 네트워크 경로:
Oracle VCN A (NAT)     Internet      Oracle VCN B (NAT)
     │                    │                │
  Node A ──────> STUN Server <────── Node B
     │           (연결 조정)              │
     │                                    │
     └──────────> Direct P2P <───────────┘
              (최종 직접 연결)
```

## 🌐 Phase 2: Cilium CNI 동작

### VXLAN 오버레이 네트워크

Cilium이 VXLAN을 사용하는 이유:
- Tailscale 네트워크와 **독립적인** Pod 네트워크 구성
- 다양한 네트워크 토폴로지 지원

```
VXLAN 캡슐화:
┌──────────────┐
│ Original L2  │  Pod → Pod 원본 프레임
├──────────────┤
│   VXLAN      │  VNI: 1234 (가상 네트워크 ID)
├──────────────┤
│   UDP        │  Port: 8472
├──────────────┤
│   IP         │  Src: 100.64.0.1, Dst: 100.64.0.2
├──────────────┤
│   WireGuard  │  Tailscale 암호화
└──────────────┘
```

### eBPF 프로그램 위치

```bash
# eBPF 프로그램 확인
cilium bpf list
# Loaded BPF programs:
# - cil_from_container (TC ingress)
# - cil_to_container (TC egress)
# - cil_sock_ops (socket operations)
# - cil_lb (load balancer)
```

각 프로그램의 역할:

| 프로그램 | 위치 | 역할 |
|---------|------|------|
| `cil_from_container` | veth ingress | Pod에서 나가는 패킷 처리 |
| `cil_to_container` | veth egress | Pod로 들어오는 패킷 처리 |
| `cil_sock_ops` | Socket layer | 연결 추적, 정책 적용 |
| `cil_lb` | XDP/TC | Service 로드밸런싱 |

### kube-proxy 대체 동작

```
Service (ClusterIP) 접근 흐름:

1. Pod A → Service IP (10.96.0.10)
           ↓
2. eBPF LB Map lookup
   ┌──────────────────────┐
   │ 10.96.0.10:80 →      │
   │   - 10.244.0.5:80    │
   │   - 10.244.1.6:80    │
   └──────────────────────┘
           ↓
3. DNAT to selected endpoint
           ↓
4. VXLAN encapsulation
           ↓
5. Tailscale tunneling
```

## 🔀 Phase 3: 패킷 플로우 상세 분석

### 시나리오 1: 같은 노드 내 Pod 통신

```
Pod A (10.244.0.5) → Pod B (10.244.0.6)

1. Pod A veth → cilium_host
2. eBPF redirect (no kernel stack)
3. Direct to Pod B veth
4. Pod B receives

Total hops: 0 (eBPF fast path)
```

### 시나리오 2: 다른 노드 Pod 통신

```
Node 1: Pod A (10.244.0.5) → Node 2: Pod B (10.244.1.10)

1. Pod A → cilium_vxlan
   └─ eBPF: policy check, VXLAN encap

2. cilium_vxlan → tailscale0
   └─ Routing: 10.244.1.0/24 via 100.64.0.2

3. tailscale0 → WireGuard tunnel
   └─ Encryption, UDP 41641

4. Internet traversal
   └─ Oracle VCN → Public IP → Oracle VCN

5. Node 2: tailscale0 → cilium_vxlan
   └─ Decryption, VXLAN decap

6. cilium_vxlan → Pod B
   └─ eBPF: policy enforcement
```

### 시나리오 3: Pod → 외부 인터넷

```
Pod (10.244.0.5) → google.com (142.250.x.x)

1. Pod → cilium_host
   └─ Default route via node

2. eBPF Masquerade
   └─ SNAT: 10.244.0.5 → Node IP

3. Node routing decision
   └─ Default route via enp0s6 (NOT tailscale0)

4. Oracle VCN NAT Gateway
   └─ SNAT: Private IP → Public IP

5. Internet
```

## 🔍 Phase 4: 디버깅과 관찰

### tcpdump로 패킷 캡처

```bash
# VXLAN 트래픽 확인
sudo tcpdump -i tailscale0 -nn 'udp port 8472' -c 10

# 특정 Pod의 트래픽
sudo tcpdump -i cali12345678 -nn

# WireGuard 트래픽
sudo tcpdump -i any -nn 'udp port 41641'
```

### eBPF 맵 확인

```bash
# Service 엔드포인트 맵
cilium bpf lb list

# 연결 추적 테이블
cilium bpf ct list global

# 정책 맵
cilium bpf policy get
```

### 네트워크 경로 추적

```bash
# Pod에서 경로 확인
kubectl run debug --rm -it --image=nicolaka/netshoot -- bash

# netshoot Pod 내부에서
traceroute 100.64.0.2  # 다른 노드
traceroute 8.8.8.8     # 외부

# MTU 경로 발견
ping -M do -s 1472 100.64.0.2
```

## ⚙️ Phase 5: 성능 최적화

### MTU 최적화

```
최적 MTU 계산:
Base Ethernet MTU:     1500
- WireGuard overhead:   -60
- VXLAN overhead:       -50
- Safety margin:        -10
─────────────────────────────
Optimal MTU:           1380

실제 설정: 1200 (보수적)
```

### eBPF 최적화

```yaml
# Cilium 성능 튜닝
bpf:
  preallocateMaps: true     # BPF 맵 사전 할당
  masquerade: true          # eBPF 기반 마스커레이드
  clockSource: ktime        # 정확한 타임스탬프

# 연결 추적 최적화
conntrack:
  accounting: false         # 통계 비활성화 (성능)
  maxEntries: 524288       # 최대 연결 수 증가
```

### Tailscale 최적화

```bash
# UDP 수신 버퍼 증가
sudo sysctl -w net.core.rmem_default=26214400
sudo sysctl -w net.core.rmem_max=26214400

# WireGuard 키 교환 간격
sudo tailscale set --key-expiry-warning=24h
```

## 🛠️ Phase 6: 트러블슈팅 가이드

### 문제: Pod가 외부 접속 불가

**증상**: Pod → google.com 실패

**진단 체크리스트**:
1. ✓ CoreDNS 동작 확인
2. ✓ Masquerade 설정 확인
3. ✓ Default route 확인
4. ✓ Oracle VCN NAT Gateway

```bash
# 진단 명령어
kubectl exec <pod> -- nslookup google.com
kubectl exec <pod> -- ping 8.8.8.8
kubectl exec <pod> -- wget -O- http://checkip.amazonaws.com
```

### 문제: 노드 간 통신 불안정

**증상**: 간헐적 패킷 로스

**진단 체크리스트**:
1. ✓ Tailscale 직접 연결 확인
2. ✓ VXLAN 인터페이스 상태
3. ✓ MTU 불일치 확인

```bash
# Tailscale 연결 품질
tailscale ping --verbose 100.64.0.2

# VXLAN 인터페이스 통계
ip -s link show cilium_vxlan
```

## 📊 모니터링 메트릭

### 주요 관찰 지표

| 메트릭 | 명령어 | 정상 범위 |
|--------|--------|-----------|
| Tailscale 지연시간 | `tailscale ping` | < 50ms |
| VXLAN 패킷 드롭 | `ip -s link show` | 0 |
| eBPF 프로그램 오류 | `cilium metrics list` | 0 |
| 연결 추적 테이블 사용률 | `conntrack -C` | < 80% |

### Prometheus 메트릭 (선택)

```yaml
# Cilium 메트릭 활성화
prometheus:
  enabled: true
  serviceMonitor:
    enabled: true

# 주요 메트릭
- cilium_bpf_syscall_duration_seconds
- cilium_datapath_errors_total
- cilium_endpoint_state
- cilium_kubernetes_events_total
```

## 🎯 핵심 이해 포인트

### 1. 왜 VXLAN over Tailscale?

- **격리**: Pod 네트워크와 노드 네트워크 분리
- **유연성**: 다양한 underlay 네트워크 지원
- **호환성**: 표준 Kubernetes 네트워킹 모델

### 2. 왜 kube-proxy 대체?

- **성능**: iptables 규칙 대신 eBPF 사용
- **확장성**: O(1) 룩업 시간
- **기능**: 고급 로드밸런싱 알고리즘

### 3. 왜 devices 설정이 중요한가?

```yaml
# ❌ 잘못된 설정
devices: tailscale0  # 외부 통신 불가

# ✅ 올바른 설정
devices: ""  # 자동 감지 (enp0s6, tailscale0)
```

- Cilium은 지정된 인터페이스만 사용
- 외부 통신은 물리 인터페이스 필요
- Tailscale은 피어 간 통신만 처리

## 📚 참고 자료

### 공식 문서
- [Cilium eBPF Datapath](https://docs.cilium.io/en/stable/concepts/ebpf/)
- [Tailscale How It Works](https://tailscale.com/blog/how-tailscale-works/)
- [VXLAN RFC 7348](https://tools.ietf.org/html/rfc7348)

### 추가 학습
- eBPF 프로그래밍 기초
- WireGuard 프로토콜 상세
- Kubernetes 네트워킹 모델

## 🔄 다음 단계

네트워킹을 완전히 이해했다면:
- Ingress Controller 추가 (Traefik/Nginx)
- 모니터링 스택 구성 (Prometheus/Grafana)
- GitOps 파이프라인 구축

---

*이제 당신은 Tailscale + Cilium 네트워킹의 전문가입니다!*