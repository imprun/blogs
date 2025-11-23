# 2단계: Tailscale 메시 네트워크 구성

> **시리즈**: [Oracle Cloud + Tailscale + Kubernetes 완벽 가이드](README.md)
> ← **이전**: [1단계: Oracle Cloud 준비](01-preparation-oracle-cloud.md) | **다음**: [3단계: Kubernetes + Cilium 구축](03-setup-kubernetes-cilium.md) →

---

> 서로 다른 Oracle Cloud 계정의 인스턴스를 하나의 네트워크로 연결

## 📋 이 단계에서 할 일

1. Tailscale 설치 및 최적화
2. Tailscale 공용 서버 또는 Headscale 연결
3. 노드 간 연결 확인
4. MTU 최적화

## 🌐 Tailscale이란?

- **WireGuard 기반** 메시 VPN
- **NAT 통과**: 복잡한 네트워크 환경에서도 연결
- **자동 키 관리**: 수동 설정 불필요
- **100.64.0.0/10 대역** 자동 할당

## 🚀 빠른 시작 (Tailscale 공용 서버)

가장 간단한 방법은 Tailscale 공식 서버를 사용하는 것입니다.

### 1. Tailscale 설치

**모든 노드에서 실행**:

```bash
# Tailscale 설치
curl -fsSL https://tailscale.com/install.sh | sh

# 서비스 활성화
sudo systemctl enable --now tailscaled
```

### 2. Tailscale 연결

```bash
# Tailscale 연결 (인터랙티브 인증)
sudo tailscale up

# 브라우저에서 인증 URL 열어서 로그인
# Google, Microsoft, GitHub 계정 사용 가능
```

### 3. 연결 확인

```bash
# 자신의 Tailscale IP 확인
tailscale ip -4
# 100.64.x.x

# 연결된 노드 목록
tailscale status

# 다른 노드 ping
tailscale ping <다른_노드_IP>
```

## 🏠 자체 호스팅 (Headscale 서버)

완전한 통제권을 원한다면 Headscale을 사용하세요.
→ [**02a-setup-headscale-server.md**](02a-setup-headscale-server.md) 참고

### Headscale 사전 준비

Headscale 서버가 이미 구축되어 있다면:

1. **사전 인증 키 생성** (Headscale 서버에서)
   ```bash
   sudo headscale users create k8s
   sudo headscale --user k8s preauthkeys create --reusable --expiration 2160h
   ```

2. **연결 정보 확인**
   - Server URL: `https://headscale.yourdomain.com:20443`
   - Auth Key: 위에서 생성한 키

### Headscale 연결

**각 노드에서 실행**:

```bash
# 노드 이름 설정
NODENAME="master-01"  # 각 노드별로 변경

# Headscale 서버 연결
sudo tailscale up \
  --login-server=https://headscale.yourdomain.com:20443 \
  --authkey=<YOUR_PREAUTHKEY> \
  --hostname=$NODENAME \
  --accept-routes \
  --accept-dns=false

# 연결 확인
tailscale status
```

## ⚙️ 네트워크 최적화

> **참고**: IP 포워딩과 커널 파라미터는 [1단계](01-preparation-oracle-cloud.md)에서 이미 설정했습니다.

### 1. 네트워크 성능 최적화 (Tailscale 전용)

Oracle Cloud 환경에서 Tailscale 최적 성능을 위한 설정:

```bash
# 기본 네트워크 인터페이스 찾기
NETDEV=$(ip -o route get 8.8.8.8 | cut -f 5 -d " ")

# UDP 성능 최적화 (Tailscale WireGuard용)
sudo ethtool -K $NETDEV rx-udp-gro-forwarding on rx-gro-list off 2>/dev/null || true

echo "Network device $NETDEV optimized for Tailscale"
```

### 2. MTU 확인

Tailscale + VXLAN 환경을 위한 MTU 최적화:

```bash
# Tailscale 인터페이스 MTU 확인
ip link show tailscale0
# 기본값: 1280

# 필요시 MTU 조정 (권장하지 않음)
# sudo ip link set dev tailscale0 mtu 1280
```

## 🔍 연결 검증

### 1. 기본 연결 테스트

```bash
# 모든 노드 상태 확인
tailscale status

# 출력 예시:
# 100.64.0.1   master-01    k8s@    linux   active
# 100.64.0.2   worker-01    k8s@    linux   active
# 100.64.0.3   worker-02    k8s@    linux   active
```

### 2. 네트워크 품질 확인

```bash
# 네트워크 진단
tailscale netcheck

# 다른 노드로 ping (Tailscale 내장)
tailscale ping 100.64.0.1
# pong from master-01 (100.64.0.1) via DERP(sin) in 45ms

# 일반 ping 테스트
ping -c 3 100.64.0.1
```

### 3. 포트 연결 테스트

```bash
# nc (netcat) 테스트
nc -zv 100.64.0.1 22  # SSH
nc -zv 100.64.0.1 6443  # Kubernetes API (나중에)
```

## 🛡️ 방화벽 설정

### Oracle Linux (firewalld)

```bash
# Tailscale 인터페이스를 trusted zone에 추가
sudo firewall-cmd --permanent --zone=trusted --add-interface=tailscale0

# 필요한 포트 열기 (Kubernetes용)
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=8472/udp  # VXLAN

# 설정 적용
sudo firewall-cmd --reload
```

### Ubuntu (ufw)

```bash
# Tailscale 트래픽 허용
sudo ufw allow in on tailscale0

# 필요한 포트 열기
sudo ufw allow 6443/tcp
sudo ufw allow 10250/tcp
sudo ufw allow 8472/udp

# 적용
sudo ufw reload
```

## 📋 체크리스트

각 노드에서 확인:

- [ ] Tailscale 설치 완료
- [ ] Tailscale 연결 성공
- [ ] 100.64.x.x IP 할당 확인
- [ ] 다른 노드와 ping 통신 성공
- [ ] IP 포워딩 활성화
- [ ] 방화벽 규칙 설정

## 🎯 예상 결과

성공적으로 구성된 경우:

```bash
$ tailscale status
100.64.0.1   master-01    k8s@    linux   active
100.64.0.2   worker-01    k8s@    linux   active
100.64.0.3   worker-02    k8s@    linux   active

$ tailscale ping 100.64.0.2
pong from worker-01 (100.64.0.2) via DERP(sin) in 25ms
```

## ⚠️ 일반적인 문제

### "no matching peer" 에러
- 노드가 아직 연결되지 않음
- `tailscale status`로 상태 확인
- `sudo systemctl restart tailscaled` 시도

### 높은 지연시간
- DERP 릴레이 서버 경유 중
- 직접 연결 대기 (1-2분 소요)
- `tailscale netcheck`로 연결 상태 확인

### 방화벽 차단
- Oracle Cloud Security List 확인
- UDP 41641 포트 열려있는지 확인
- OS 레벨 방화벽 규칙 확인

## 🔄 다음 단계

Tailscale 네트워크가 구성되면:
- Headscale 자체 호스팅 (선택): [**02a-setup-headscale-server.md**](02a-setup-headscale-server.md)
- Kubernetes 설치 진행: [**03-setup-kubernetes-cilium.md**](03-setup-kubernetes-cilium.md)

## 💡 추가 팁

### Tailscale vs Headscale 선택 기준

| 항목 | Tailscale 공용 | Headscale 자체 호스팅 |
|------|---------------|---------------------|
| **설정 난이도** | 매우 쉬움 | 중간 |
| **무료 노드 수** | 20개 | 무제한 |
| **제어권** | 제한적 | 완전한 통제 |
| **인증 방법** | OAuth (Google 등) | 사전 인증 키 |
| **추천 대상** | 빠른 테스트, 소규모 | 프로덕션, 대규모 |

### 성능 최적화 팁

1. **직접 연결 확인**
   ```bash
   tailscale status | grep "direct"
   ```

2. **DERP 서버 우회**
   - 시간이 지나면 자동으로 직접 연결
   - 방화벽 규칙 재확인

3. **Exit Node 설정 (선택)**
   ```bash
   # 특정 노드를 Exit Node로 설정
   sudo tailscale up --advertise-exit-node
   ```

---

*다음 문서: [Kubernetes + Cilium 클러스터 구축](03-setup-kubernetes-cilium.md)*