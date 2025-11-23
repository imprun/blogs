# 5단계: 실전 경험과 아키텍처 결정 배경

> **시리즈**: [Oracle Cloud + Tailscale + Kubernetes 완벽 가이드](README.md)
> ← **이전**: [4단계: 네트워킹 심화 이해](04-deep-dive-networking.md) | **처음으로**: [README](README.md)

---

> 며칠간의 삽질에서 얻은 교훈 - "다음에는 이렇게 하지 말자"

## 📋 이 문서의 목적

이 가이드의 최종 아키텍처는 처음부터 완벽하게 설계된 것이 아닙니다.
여러 번의 시도와 실패를 거쳐 현재의 구성에 도달했습니다.

**왜 이 문서가 필요한가?**
- 실패한 방법을 기록하여 같은 함정에 빠지지 않도록
- 아키텍처 결정의 배경과 근거 공유
- "왜 이렇게 했나?"에 대한 솔직한 답변

### 전체 여정 타임라인

```mermaid
graph TD
    Start[목표: Tailscale + Kubernetes 클러스터] --> Attempt1{시도 1: Tailscale in K8s}

    Attempt1 -->|DaemonSet 배포| Problem1[❌ 문제 발생]
    Problem1 --> Issue1a[네트워크 네임스페이스 충돌]
    Problem1 --> Issue1b[Pod 재시작 시 터널 끊김]
    Problem1 --> Issue1c[hostNetwork 보안 문제]

    Issue1a --> Decision1[🔄 systemd 서비스로 변경]
    Issue1b --> Decision1
    Issue1c --> Decision1

    Decision1 --> Attempt2{시도 2: Cilium Native Routing}

    Attempt2 -->|성능 최적화 시도| Problem2[❌ 문제 발생]
    Problem2 --> Issue2a[Tailscale이 Pod 라우팅 안함]
    Problem2 --> Issue2b[수동 라우팅 관리 악몽]
    Problem2 --> Issue2c[자동화 실패]

    Issue2a --> Decision2[🔄 VXLAN으로 회귀]
    Issue2b --> Decision2
    Issue2c --> Decision2

    Decision2 --> Final[✅ 최종 아키텍처]

    Final --> Layer1[Tailscale: systemd 서비스]
    Final --> Layer2[Cilium: VXLAN 모드]
    Final --> Layer3[eBPF: kube-proxy 대체]

    Layer1 --> Result[안정적 운영 중]
    Layer2 --> Result
    Layer3 --> Result

    style Start stroke:#00bfff,stroke-width:3px
    style Problem1 stroke:#ff6b6b,stroke-width:3px
    style Problem2 stroke:#ff6b6b,stroke-width:3px
    style Decision1 stroke:#ffa500,stroke-width:3px
    style Decision2 stroke:#ffa500,stroke-width:3px
    style Final stroke:#4ecdc4,stroke-width:3px
    style Result stroke:#95e1d3,stroke-width:3px
```

## 🚫 실패담 1: Tailscale을 Kubernetes 내부에서 실행

### 시도한 방법

Tailscale을 Kubernetes 방식으로 관리하려고 시도했습니다:

```yaml
# 시도했던 방법 (실패)
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: tailscale
  namespace: kube-system
spec:
  template:
    spec:
      hostNetwork: true  # ← 이미 여기서 문제의 냄새
      containers:
      - name: tailscale
        image: tailscale/tailscale:latest
        securityContext:
          privileged: true  # ← 보안 문제
```

**왜 시도했나?**
- Kubernetes 방식으로 통합 관리
- `kubectl` 명령어로 모든 것 제어
- GitOps 워크플로우에 포함

### 겪은 문제

#### 1. **CNI와 네트워크 네임스페이스 충돌**

```bash
# Pod 네트워크 네임스페이스
ip netns exec cni-xxx ip route
# 10.244.0.0/16 via cilium_host

# 호스트 네트워크 네임스페이스
ip route
# 100.64.0.0/10 via tailscale0

# → 두 세계가 서로를 인식하지 못함!
```

**증상:**
- Tailscale Pod 시작은 되지만 라우팅 작동 안 함
- `tailscale status`는 정상, 실제 통신은 실패
- 패킷 드롭 발생

#### 2. **Pod 재시작 시 터널 끊김**

```bash
# Scenario
1. Tailscale Pod 정상 실행 → 터널 연결 OK
2. Pod 재시작 (업데이트, 노드 이동 등)
3. Tailscale 재연결 시도
4. 기존 세션 끊김, 새 IP 할당 가능
5. 클러스터 전체 통신 장애 발생!
```

**문제:**
- Pod는 ephemeral (일시적)
- Tailscale 터널은 persistent (지속적) 필요
- 근본적인 불일치

#### 3. **hostNetwork 사용의 보안 문제**

```yaml
hostNetwork: true  # Pod가 호스트 네트워크 직접 사용
```

**문제점:**
- Pod가 호스트의 모든 네트워크 접근 가능
- Kubernetes 네트워크 격리 무력화
- 보안 감사 실패

### 기술적 배경: 왜 작동하지 않는가?

#### 네트워크 네임스페이스의 한계

```mermaid
graph TB
    subgraph Host["🖥️ Host Network Namespace"]
        TS[Tailscale tailscale0]
        TSR[라우팅 테이블<br/>100.64.0.0/10]
        HostRoute[호스트 라우팅 테이블]

        TS --> TSR
        TSR --> HostRoute
    end

    subgraph PodNS["📦 Pod Network Namespace"]
        Veth[veth 인터페이스]
        PodRoute[독립 라우팅 테이블<br/>10.244.x.0/24]

        Veth --> PodRoute
    end

    Host -.->|❌ 접근 불가| PodNS
    PodNS -.->|❌ 접근 불가| Host

    style Host stroke:#00bfff,stroke-width:3px
    style PodNS stroke:#ff6b6b,stroke-width:3px
    style TS stroke:#4ecdc4,stroke-width:2px
    style Veth stroke:#ee5a6f,stroke-width:2px
```

**핵심 문제:**
- Tailscale은 호스트 라우팅 테이블 수정
- Pod는 독립 네트워크 네임스페이스
- 두 레이어가 서로 통신 불가

### 결론: 노드 레벨 systemd 서비스

**최종 선택:**
```bash
# 각 노드에서 직접 설치
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --login-server=...
```

**장점:**
- ✅ 안정적인 터널 (재부팅 시에도 유지)
- ✅ CNI와 충돌 없음
- ✅ 단순하고 명확한 관리
- ✅ Tailscale 공식 권장 방법

**단점:**
- ❌ 노드별 수동 설정 필요
- ❌ Kubernetes 방식 관리 불가
- ❌ GitOps에 포함 어려움

**트레이드오프:**
운영 안정성 > 관리 편의성

---

## 🚫 실패담 2: Cilium Native Routing 시도

### 시도한 방법

VXLAN 오버헤드를 제거하여 성능을 높이려고 시도:

```bash
# Native Routing 모드로 Cilium 설치
helm install cilium cilium/cilium \
  --set routingMode=native \          # ← VXLAN 대신 Native
  --set autoDirectNodeRoutes=true \   # ← 자동 라우팅 기대
  --set ipv4NativeRoutingCIDR=10.244.0.0/16
```

**왜 시도했나?**
- VXLAN 캡슐화 오버헤드 제거 (5-10% 성능 향상)
- "더 빠른 네트워크"에 대한 욕심
- 기술 블로그에서 "Native Routing이 최고"라는 글 읽음

### 겪은 문제

#### 1. **Tailscale이 Pod 라우팅을 자동 설정하지 않음**

```bash
# 기대했던 것:
tailscale up --advertise-routes=10.244.0.0/16
# → 모든 노드에 자동으로 라우팅 설정될 줄 알았음

# 현실:
ip route show
# 100.64.0.2 via tailscale0  ← 노드 IP만 있음
# 10.244.1.0/24는 없음!
```

**깨달음:**
- Tailscale은 **노드 간 터널만 제공**
- Pod 네트워크는 **별도로 라우팅 설정 필요**
- `advertise-routes`는 광고만, 자동 설정 아님

#### 2. **수동 라우팅 테이블 관리의 악몽**

**각 노드에서 수동 설정 필요:**

```bash
# Master 노드 (100.64.0.1)
sudo ip route add 10.244.1.0/24 via 100.64.0.2  # Worker 1
sudo ip route add 10.244.2.0/24 via 100.64.0.3  # Worker 2

# Worker 1 (100.64.0.2)
sudo ip route add 10.244.0.0/24 via 100.64.0.1  # Master
sudo ip route add 10.244.2.0/24 via 100.64.0.3  # Worker 2

# Worker 2 (100.64.0.3)
sudo ip route add 10.244.0.0/24 via 100.64.0.1  # Master
sudo ip route add 10.244.1.0/24 via 100.64.0.2  # Worker 1
```

```mermaid
graph TB
    subgraph Master["🖥️ Master (100.64.0.1)"]
        M1[Pod CIDR: 10.244.0.0/24]
        M2[라우팅 테이블]
        M3["route add 10.244.1.0/24 via .2"]
        M4["route add 10.244.2.0/24 via .3"]
    end

    subgraph Worker1["🖥️ Worker 1 (100.64.0.2)"]
        W1[Pod CIDR: 10.244.1.0/24]
        W2[라우팅 테이블]
        W3["route add 10.244.0.0/24 via .1"]
        W4["route add 10.244.2.0/24 via .3"]
    end

    subgraph Worker2["🖥️ Worker 2 (100.64.0.3)"]
        WW1[Pod CIDR: 10.244.2.0/24]
        WW2[라우팅 테이블]
        WW3["route add 10.244.0.0/24 via .1"]
        WW4["route add 10.244.1.0/24 via .2"]
    end

    Master <-->|Tailscale 터널| Worker1
    Worker1 <-->|Tailscale 터널| Worker2
    Master <-->|Tailscale 터널| Worker2

    Note1["😱 노드 추가 시<br/>모든 노드에서<br/>라우팅 업데이트 필요!"]

    style Note1 stroke:#ff6b6b,stroke-width:3px
    style Master stroke:#00bfff,stroke-width:2px
    style Worker1 stroke:#4ecdc4,stroke-width:2px
    style Worker2 stroke:#ffa500,stroke-width:2px
```

**문제점:**
1. 노드 추가 시마다 **모든 노드**에서 업데이트
2. IP 변경 시 **모든 라우팅 테이블** 수정
3. 재부팅 시 라우팅 유실 → 스크립트 작성 필요
4. 운영 복잡도 기하급수적 증가

#### 3. **자동화의 어려움**

시도한 자동화 방법들:

```bash
# 1. systemd 서비스로 라우팅 추가 (실패)
#    - 노드 추가/제거 시 동기화 문제

# 2. Kubernetes Operator 작성 (너무 복잡)
#    - 라우팅 테이블 관리 Operator 필요
#    - 오버엔지니어링

# 3. Ansible 플레이북 (관리 포인트 증가)
#    - Kubernetes 외부 도구 의존성
#    - GitOps와 불일치
```

**결론:** 모두 만족스럽지 않음

### 기술적 배경: Native Routing의 요구사항

#### Native Routing이 작동하는 환경

```mermaid
graph TB
    subgraph Ideal["✅ Native Routing 이상적 환경"]
        direction LR
        N1[Node 1] <--> Switch[L2 스위치 또는<br/>클라우드 라우팅]
        N2[Node 2] <--> Switch
        N3[Node 3] <--> Switch

        Switch --> Auto[자동 라우팅<br/>BGP / VPC Peering /<br/>물리 스위치]
    end

    subgraph Reality["❌ 우리 환경 (Tailscale)"]
        direction TB
        subgraph Acc1["Oracle Account 1"]
            Node1[Node 1<br/>10.244.0.0/24]
        end
        subgraph Acc2["Oracle Account 2"]
            Node2[Node 2<br/>10.244.1.0/24]
        end
        subgraph Acc3["Oracle Account 3"]
            Node3[Node 3<br/>10.244.2.0/24]
        end

        Node1 <-.->|WireGuard 터널| Node2
        Node2 <-.->|WireGuard 터널| Node3
        Node1 <-.->|WireGuard 터널| Node3

        Manual[수동 라우팅만 가능<br/>자동화 없음!]
    end

    style Ideal stroke:#4ecdc4,stroke-width:3px
    style Reality stroke:#ff6b6b,stroke-width:3px
    style Auto stroke:#95e1d3,stroke-width:2px
    style Manual stroke:#ee5a6f,stroke-width:2px
    style Acc1 stroke:#00bfff,stroke-width:1px
    style Acc2 stroke:#00bfff,stroke-width:1px
    style Acc3 stroke:#00bfff,stroke-width:1px
```

**근본적인 불일치:**
- Native Routing: Layer 2/3 라우팅 필요
- Tailscale: Layer 4 터널 (라우팅 제공 안 함)

### 결론: VXLAN 터널링으로 회귀

**최종 선택:**
```bash
helm install cilium cilium/cilium \
  --set routingMode=tunnel \
  --set tunnelProtocol=vxlan
```

**장점:**
- ✅ 라우팅 자동 관리 (Cilium이 전담)
- ✅ 노드 추가/제거 시 설정 불필요
- ✅ 운영 복잡도 낮음
- ✅ 안정적이고 예측 가능

**단점:**
- ❌ VXLAN 캡슐화 오버헤드 (~5-10%)
- ❌ MTU 감소 (1500 → 1200)

**성능 테스트 결과:**
```bash
# iperf3 테스트 (Pod to Pod)
Native Routing: ~9.2 Gbps
VXLAN:          ~8.7 Gbps
차이:           ~5%

# 실제 워크로드 영향: 거의 무시 가능
# CPU/메모리가 먼저 병목
```

**트레이드오프:**
미세한 성능 차이 < 압도적인 운영 편의성

---

## ✅ 최종 아키텍처 결정

### 선택한 구성

```mermaid
graph TB
    subgraph Layer1["🌐 Layer 1: 노드 간 연결 (Tailscale)"]
        TS[Tailscale systemd 서비스]
        TSF[WireGuard 터널<br/>100.64.0.0/10]
        TSM[관리: systemd]

        TS --> TSF
        TSF --> TSM
    end

    subgraph Layer2["📦 Layer 2: Pod 네트워킹 (Cilium CNI)"]
        Cilium[Cilium VXLAN 모드]
        PodNet[Pod 네트워크<br/>10.244.0.0/16]
        AutoRoute[자동 라우팅 관리]
        CiliumM[관리: Helm Chart]

        Cilium --> PodNet
        PodNet --> AutoRoute
        AutoRoute --> CiliumM
    end

    subgraph Layer3["⚡ Layer 3: 서비스 로드밸런싱 (eBPF)"]
        eBPF[Cilium eBPF]
        KubeProxy[kube-proxy 대체]
        SvcLB[Service Load Balancing]

        eBPF --> KubeProxy
        KubeProxy --> SvcLB
    end

    Layer1 ==>|노드 터널 제공| Layer2
    Layer2 ==>|Pod 네트워크 제공| Layer3

    Responsibility["책임 분리<br/>---<br/>Tailscale: 노드만 연결<br/>Cilium: 나머지 전부"]

    style Layer1 stroke:#00bfff,stroke-width:3px
    style Layer2 stroke:#4ecdc4,stroke-width:3px
    style Layer3 stroke:#ffa500,stroke-width:3px
    style Responsibility stroke:#ee5a6f,stroke-width:2px
```

### 각 계층의 책임

| 계층 | 역할 | 관리 방식 | CIDR 대역 |
|------|------|-----------|-----------|
| **Tailscale** | 노드 간 터널 | systemd 서비스 | 100.64.0.0/10 |
| **Cilium** | Pod 네트워킹 | Helm Chart | 10.244.0.0/16 |
| **eBPF** | Service LB | Cilium 내장 | - |

**명확한 책임 분리:**
- Tailscale: "노드만 연결" (Layer 1)
- Cilium: "나머지 전부" (Layer 2 + 3)

### 의사결정 플로우차트

```mermaid
graph TD
    Start[Kubernetes 클러스터 구축 필요] --> Q1{노드들이<br/>동일 네트워크?}

    Q1 -->|예<br/>VPC/L2 동일| VPC[Native Routing 가능]
    Q1 -->|아니오<br/>서로 다른 네트워크| Overlay[Overlay 네트워크 필요]

    VPC --> Q2{BGP 또는<br/>클라우드 라우팅<br/>사용 가능?}
    Q2 -->|예| NativeOK[✅ Cilium Native Routing<br/>최고 성능]
    Q2 -->|아니오| ManualRoute{수동 라우팅<br/>관리 가능?}

    ManualRoute -->|가능<br/>노드 수 고정| NativeManual[⚠️ Native Routing<br/>+ 수동 설정<br/>5-10% 성능 향상]
    ManualRoute -->|불가능<br/>노드 수 가변| UseVXLAN[→ VXLAN 사용]

    Overlay --> Q3{Tailscale/WireGuard<br/>터널 사용?}
    Q3 -->|예| TailscaleQ{Tailscale을<br/>어디서 실행?}
    Q3 -->|아니오| OtherOverlay[다른 Overlay 검토<br/>Flannel, Calico 등]

    TailscaleQ -->|K8s 내부<br/>DaemonSet| TSFail[❌ 네임스페이스 충돌<br/>권장 안 함]
    TailscaleQ -->|노드 레벨<br/>systemd| TSOK[✅ Tailscale systemd]

    TSOK --> UseVXLAN
    TSFail -.->|실패 후| TSOK

    UseVXLAN --> Final[✅ 최종 구성<br/>Tailscale systemd<br/>+ Cilium VXLAN<br/>+ eBPF]

    NativeOK --> CNI[CNI 선택:<br/>Cilium, Calico, Flannel]
    UseVXLAN --> CNI2[CNI 선택:<br/>Cilium 권장]

    style Start stroke:#00bfff,stroke-width:3px
    style Final stroke:#4ecdc4,stroke-width:4px
    style NativeOK stroke:#95e1d3,stroke-width:3px
    style TSFail stroke:#ff6b6b,stroke-width:3px
    style TSOK stroke:#4ecdc4,stroke-width:3px
```

### 트레이드오프 분석

#### 성능 vs 운영성

| 구성 | 성능 | 운영 난이도 | 안정성 | 선택 |
|------|------|------------|--------|------|
| Native + K8s Tailscale | ⭐⭐⭐⭐⭐ | ⭐ | ⭐ | ❌ |
| Native + systemd Tailscale | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ❌ |
| VXLAN + systemd Tailscale | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ✅ |

**우선순위:**
1. 안정성 (가장 중요)
2. 운영 편의성 (두 번째)
3. 성능 (세 번째)

#### 실무 관점

**"완벽한 설정"은 없습니다:**
- 모든 것을 만족하는 구성은 존재하지 않음
- 상황과 우선순위에 따라 선택
- 우리의 선택: **안정성과 단순함**

---

## 💡 배운 교훈

### 1. 단순함이 최고

```
복잡한 최적화 << 단순하고 안정적인 구성
```

**예시:**
- ❌ Native Routing + 자동화 스크립트 + Operator
- ✅ VXLAN + 기본 설정

**이유:**
- 복잡한 시스템은 디버깅 어려움
- 운영 중 문제 발생 시 원인 파악 지연
- 팀원 온보딩 시간 증가

### 2. 운영 안정성 > 미세한 성능

**5% 성능 향상 vs 50% 운영 부담 감소**
- 대부분의 경우 성능은 충분함
- CPU/메모리/디스크가 먼저 병목
- 네트워크 5% 차이는 체감 불가

**실제 경험:**
```bash
# 성능 병목 분석 결과
1. 데이터베이스 쿼리 최적화: 50% 개선
2. API 게이트웨이 코드 리팩토링: 30% 개선
3. 네트워크 최적화 (Native vs VXLAN): 5% 개선

→ 우선순위가 명확함
```

### 3. 공식 문서 권장사항에는 이유가 있다

**Tailscale 공식 문서:**
> "Run Tailscale as a system service on each node"

**Cilium 공식 문서:**
> "VXLAN is recommended for overlay networks"

**왜 처음부터 안 따랐나?**
- "내 상황은 다를 거야"
- "더 나은 방법이 있을 거야"
- "최신 기술을 써야 해"

**깨달음:**
- 공식 문서는 수많은 사례 기반
- 대부분의 엣지 케이스 고려됨
- 특별한 이유 없으면 권장사항 따르기

### 4. 완벽한 설정은 없다

**모든 아키텍처는 트레이드오프:**
- 성능 ↔ 안정성
- 복잡도 ↔ 유연성
- 자동화 ↔ 제어권

**우선순위 명확히:**
1. 무엇이 가장 중요한가?
2. 무엇을 포기할 수 있는가?
3. 팀이 관리할 수 있는가?

### 5. 삽질은 배움의 과정

**실패한 시도들이 가치 있는 이유:**
- 기술의 한계 이해
- 트레이드오프 체감
- 더 나은 결정의 근거

**이 문서의 목적:**
- 같은 실수 반복 방지
- 결정의 배경 공유
- 다음 구축자를 위한 가이드

---

## 🔮 향후 개선 가능성

### 1. Tailscale in Kubernetes 재검토

**언제 다시 시도할 가치가 있나?**
- Tailscale Kubernetes Operator 안정화
- CNI와의 통합 개선
- 공식 지원 시작

**현재 상태:**
- Tailscale Operator: 베타
- 프로덕션 사용: 권장 안 됨

### 2. Native Routing 재검토

**언제 시도할 가치가 있나?**
- BGP 라우팅 가능한 환경
- 클라우드 VPC Peering 사용
- 초고성능 필요 (HPC, ML)

**우리 환경에서는:**
- 서로 다른 Oracle 계정 = VPC Peering 불가
- Native Routing 불가능

### 3. 모니터링 강화

**추가할 만한 것:**
- Cilium Hubble (네트워크 가시성)
- Prometheus + Grafana
- 성능 메트릭 수집

---

## 📚 추가 자료

### 참고한 문서들

- [Tailscale Kubernetes Best Practices](https://tailscale.com/kb/1185/kubernetes/)
- [Cilium Network Policy](https://docs.cilium.io/en/stable/network/kubernetes/)
- [VXLAN RFC 7348](https://tools.ietf.org/html/rfc7348)

### 비슷한 경험담

- [Reddit: Tailscale + K8s Issues](https://reddit.com/r/kubernetes)
- [GitHub: Cilium Native Routing Discussions](https://github.com/cilium/cilium/discussions)

---

## ✍️ 추가 실패담 (향후 작성 예정)

이 문서는 계속 업데이트됩니다. 앞으로 추가할 내용:

- [ ] iptables vs eBPF 선택 과정
- [ ] MTU 설정 시행착오
- [ ] 블록볼륨 마운트 실수
- [ ] Headscale vs Tailscale 공식 서버
- [ ] Oracle Cloud 방화벽 설정 함정
- [ ] SELinux Permissive 모드의 이유
- [ ] 기타...

**기여 환영:**
비슷한 경험이 있다면 공유해주세요!

---

*"실패는 성공의 어머니" - 하지만 남의 실패로부터 배우면 더 빠릅니다.*