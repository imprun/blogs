# 1단계: Oracle Cloud 준비 및 설정

> **시리즈**: [Oracle Cloud + Tailscale + Kubernetes 완벽 가이드](README.md)
> **다음**: [2단계: Tailscale 메시 네트워크 구성](02-setup-tailscale-network.md) →

---

> Oracle Cloud 무료 계정 생성부터 인스턴스 준비까지

## 📋 이 단계에서 할 일

1. Oracle Cloud 무료 계정 생성 (필요한 만큼)
2. ARM64 인스턴스 생성
3. VCN 및 보안 규칙 설정
4. SSH 접속 및 초기 설정

## 🆓 Oracle Cloud 무료 계정

### 무료 제공 사양
- **ARM64 (Ampere A1)**: 4 OCPU + 24GB RAM
- **Boot Volume**: 최대 200GB
- **네트워크**: 10TB/월 아웃바운드
- **기간**: 영구 무료 (Free Tier)

### 멀티 노드 구성 팁
Oracle Cloud는 계정당 ARM 인스턴스를 1개만 허용합니다.
멀티 노드 클러스터를 구성하려면:
- 가족/친구 계정 활용
- 각 계정에서 인스턴스 생성
- Tailscale로 네트워크 연결

## 🖥️ 인스턴스 생성

### 1. Compute 인스턴스 생성

**Compute → Instances → Create Instance**

#### Shape 선택
- **Shape**: `VM.Standard.A1.Flex` (ARM64)
- **Operating System**: Oracle Linux 8 (최신)

#### 리소스 할당

| 노드 역할 | OCPU | Memory | Boot Volume | Block Volume | 용도 |
|-----------|------|--------|-------------|--------------|------|
| **Master** | 2 | 12GB | 50GB | 150GB | Control Plane, etcd |
| **Worker** | 4 | 24GB | 50GB | 150GB | 워크로드 실행 |

#### 스토리지 설계 철학

**왜 Boot Volume 50GB + Block Volume 150GB로 분리하나요?**

| 구분 | Boot Volume (50GB) | Block Volume (150GB) |
|------|-------------------|---------------------|
| **용도** | OS, 시스템 패키지 | 컨테이너 이미지, 데이터, 로그 |
| **경로** | `/` (루트 파일시스템) | `/var/lib/containerd`, `/var/log` |
| **변경 빈도** | 낮음 (OS 업데이트만) | 높음 (Pod 생성/삭제) |
| **백업 필요성** | 낮음 (재생성 가능) | 높음 (워크로드 데이터) |
| **확장성** | 어려움 (재부팅 필요) | 쉬움 (온라인 확장) |

**실무 이점:**
1. **디스크 풀 방지**: 컨테이너 이미지가 쌓여도 OS 영역은 안전
2. **백업 효율**: Block Volume만 스냅샷 생성
3. **성능 분리**: OS I/O와 Container I/O 분리
4. **확장 용이**: Block Volume만 확장 가능 (Boot Volume은 복잡)

**용량 근거:**
- Boot Volume 50GB
  - OS 베이스: ~8GB
  - 시스템 패키지: ~5GB
  - 여유 공간: ~37GB

- Block Volume 150GB
  - 컨테이너 이미지: ~30-50GB (캐시 포함)
  - containerd 작업 공간: ~20GB
  - 로그 파일: ~10GB
  - PersistentVolume 데이터: ~70GB

#### 네트워킹
- **Public IP**: 반드시 할당 ✅
- **Private Subnet**: 기본값 사용

#### SSH 키
```bash
# 로컬에서 SSH 키 생성 (없는 경우)
ssh-keygen -t rsa -b 4096 -C "your-email@example.com"

# 공개 키 내용 복사
cat ~/.ssh/id_rsa.pub
```

Oracle Cloud 콘솔에 공개 키 붙여넣기

### 2. 인스턴스 생성 완료 후

생성된 인스턴스 정보 메모:
- **Public IP**: X.X.X.X
- **Private IP**: 10.0.0.X (VCN 내부)
- **Hostname**: instance-YYYYMMDD-HHMM

## 💾 Block Volume 연결 및 마운트

### 1. Block Volume 생성 (Oracle Cloud 콘솔)

**Storage → Block Volumes → Create Block Volume**

설정:
- **Name**: `k8s-data-volume` (또는 원하는 이름)
- **Size**: 150GB
- **Availability Domain**: 인스턴스와 동일한 AD 선택 (중요!)
- **Volume Performance**: Balanced (기본값)

### 2. 인스턴스에 연결

**Compute → Instances → 해당 인스턴스 → Attached Block Volumes → Attach Block Volume**

설정:
- **Volume**: 위에서 생성한 볼륨 선택
- **Device Path**: `/dev/oracleoci/oraclevdb` (기본값)
- **Access Type**: Read/Write
- **Attachment Type**: Paravirtualized (권장)

**중요**: 연결 후 `iSCSI Commands & Information` 버튼을 클릭하여 연결 명령어를 확인하세요.

### 3. 볼륨 포맷 및 Kubernetes용 마운트 (SSH 접속 후)

```bash
# 연결된 디스크 확인
lsblk
# 출력 예:
# sdb           8:16   0  150G  0 disk

# 디스크 이름 확인 (보통 sdb)
DISK="/dev/sdb"

# 파티션 생성
sudo parted $DISK --script mklabel gpt
sudo parted $DISK --script mkpart primary ext4 0% 100%

# 포맷 (ext4)
sudo mkfs.ext4 ${DISK}1

# Kubernetes 데이터 디렉토리 생성
sudo mkdir -p /var/lib/k8s-data

# 마운트
sudo mount ${DISK}1 /var/lib/k8s-data

# UUID 확인 및 fstab 등록
UUID=$(sudo blkid -s UUID -o value ${DISK}1)
echo "UUID=$UUID /var/lib/k8s-data ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab

# 마운트 확인
df -h /var/lib/k8s-data
```

### 4. Kubernetes 디렉토리 구조 생성 (⚠️ **K8s 설치 전 필수**)

> **중요**: 이 단계는 반드시 **Kubernetes 설치 전**에 완료해야 합니다!

#### 방법 A: 심볼릭 링크 (권장 - 설치 전)

```bash
# 블록볼륨에 Kubernetes 전용 디렉토리 생성
sudo mkdir -p /var/lib/k8s-data/containerd
sudo mkdir -p /var/lib/k8s-data/kubelet
sudo mkdir -p /var/lib/k8s-data/pods-log
sudo mkdir -p /var/lib/k8s-data/containers-log

# 표준 경로에서 블록볼륨으로 심볼릭 링크
sudo ln -s /var/lib/k8s-data/containerd /var/lib/containerd
sudo ln -s /var/lib/k8s-data/kubelet /var/lib/kubelet
sudo ln -s /var/lib/k8s-data/pods-log /var/log/pods
sudo ln -s /var/lib/k8s-data/containers-log /var/log/containers

# 권한 설정
sudo chown -R root:root /var/lib/k8s-data
sudo chmod -R 755 /var/lib/k8s-data

# 확인
ls -la /var/lib/ | grep -E "containerd|kubelet"
# lrwxrwxrwx  1 root root   30 ... containerd -> /var/lib/k8s-data/containerd
# lrwxrwxrwx  1 root root   28 ... kubelet -> /var/lib/k8s-data/kubelet

ls -la /var/log/ | grep -E "pods|containers"
# lrwxrwxrwx  1 root root   30 ... pods -> /var/lib/k8s-data/pods-log
# lrwxrwxrwx  1 root root   35 ... containers -> /var/lib/k8s-data/containers-log
```

#### 검증

```bash
# 각 경로가 150GB 볼륨을 가리키는지 확인
df /var/lib/containerd
df /var/lib/kubelet
df /var/log/pods
df /var/log/containers

# 모두 다음과 같이 출력되어야 함:
# Filesystem     1K-blocks   Used Available Use% Mounted on
# /dev/sdb1      154989760  61440 146928320   1% /var/lib/k8s-data

# 전체 마운트 상태 확인
df -h /var/lib/k8s-data
# /dev/sdb1       148G   61M  140G   1% /var/lib/k8s-data
```

#### 방법 B: 직접 마운트 (설치 후 문제 발생 시)

> **이미 Kubernetes가 설치되어 있고 ephemeral-storage 부족 문제가 발생한 경우**
>
> 자세한 내용은 [Kubernetes Ephemeral Storage 문제 해결 가이드](../../../kubernetes-ephemeral-storage-fix.md) 참조

```bash
# 1. Kubelet 중지
sudo systemctl stop kubelet

# 2. 기존 데이터 백업
sudo mv /var/lib/containerd /var/lib/containerd-backup

# 3. 마운트 포인트 생성
sudo mkdir -p /var/lib/containerd

# 4. 150GB 볼륨을 /var/lib/containerd에 직접 마운트
sudo umount /var/lib/k8s-data  # 기존 마운트 해제
sudo mount /dev/sdb1 /var/lib/containerd

# 5. 데이터 복사
sudo rsync -av /var/lib/containerd-backup/ /var/lib/containerd/

# 6. fstab 업데이트
sudo sed -i 's|/var/lib/k8s-data|/var/lib/containerd|g' /etc/fstab

# 7. Kubelet 시작
sudo systemctl start kubelet

# 8. 확인
df -h /var/lib/containerd
kubectl get nodes
```

### ⚠️ 주의사항

#### 필수 확인 사항

1. **Availability Domain 일치**: Block Volume과 인스턴스는 반드시 같은 AD에 있어야 함
2. **nofail 옵션**: fstab에 필수 (디스크 문제 시 부팅 실패 방지)
3. **설치 시점**: 심볼릭 링크는 **반드시 Kubernetes 설치 전**에 생성
4. **로그 디렉토리**: `/var/log/pods`와 `/var/log/containers`도 150GB 볼륨에 위치해야 함

#### Kubernetes Ephemeral Storage 계산 방식

Kubernetes는 다음 경로들을 기준으로 ephemeral-storage를 계산합니다:

```
체크 대상 경로:
├── /var/lib/kubelet/pods/              # Pod emptyDir, logs
├── /var/lib/containerd/.../cri/        # Container writable layers
├── /var/log/pods/                      # Pod logs ⚠️ 누락하기 쉬움!
└── /var/log/containers/                # Container logs ⚠️ 누락하기 쉬움!
```

**로그 디렉토리를 누락하면:**
- 로그가 Boot Volume(50GB)에 쌓임
- Boot Volume이 80% 이상 차면 Pod Eviction 발생
- MongoDB 등 중요한 Pod가 강제 종료될 수 있음

#### 문제 발생 시 대처

1. **Pod Eviction 발생**: [Kubernetes Ephemeral Storage 문제 해결 가이드](../../../kubernetes-ephemeral-storage-fix.md) 참조
2. **마운트 실패**: `lsblk`, `df -h`, `cat /etc/fstab`로 상태 확인
3. **백업**: 중요 데이터는 정기적으로 스냅샷 생성

## 🔐 VCN 보안 설정

### Security List 편집

**Networking → Virtual Cloud Networks → VCN 선택 → Security Lists → Default Security List**

### Ingress Rules 추가

모든 규칙에서 **Stateless: No** 설정

#### 공통 (모든 노드)

| Source | Protocol | Port | 설명 |
|--------|----------|------|------|
| 0.0.0.0/0 | TCP | 22 | SSH 접속 |
| 0.0.0.0/0 | UDP | 41641 | Tailscale WireGuard |
| 100.64.0.0/10 | TCP | 10250 | Kubelet API |
| 100.64.0.0/10 | UDP | 8472 | Cilium VXLAN |
| 100.64.0.0/10 | TCP | 4240 | Cilium Health |
| 100.64.0.0/10 | TCP | 4244 | Cilium Hubble |

#### 마스터 노드 전용

| Source | Protocol | Port | 설명 |
|--------|----------|------|------|
| 0.0.0.0/0 | TCP | 6443 | Kubernetes API Server |
| 100.64.0.0/10 | TCP | 2379-2380 | etcd |
| 100.64.0.0/10 | TCP | 10251 | kube-scheduler |
| 100.64.0.0/10 | TCP | 10252 | kube-controller-manager |
| 100.64.0.0/10 | TCP | 10257 | kube-controller-manager (secure) |
| 100.64.0.0/10 | TCP | 10259 | kube-scheduler (secure) |

#### API 게이트웨이 노출 (마스터 또는 지정된 노드)

| Source | Protocol | Port | 설명 | 대상 |
|--------|----------|------|------|------|
| 0.0.0.0/0 | TCP | 80 | HTTP (리다이렉트) | 모든 노드 |
| 0.0.0.0/0 | TCP | 443 | HTTPS | 모든 노드 |
| 0.0.0.0/0 | TCP | 20443 | Headscale HTTPS | 마스터 (Headscale 서버) |
| 0.0.0.0/0 | TCP | 30000-32767 | NodePort Services | 모든 노드 |

> **참고**: `100.64.0.0/10`은 Tailscale 네트워크 대역입니다

### Egress Rules
- **Destination**: 0.0.0.0/0
- **Protocol**: All Protocols
- **설명**: 모든 아웃바운드 트래픽 허용

## 🛡️ OS 방화벽 설정

### firewalld 비활성화 (권장)

Oracle Cloud Security List로 이미 방화벽이 제어되므로, OS 레벨 방화벽은 비활성화합니다.

**비활성화하는 이유:**
1. **중복 제어**: Oracle Cloud Security List로 충분
2. **CNI 충돌 방지**: Cilium/iptables와 firewalld 충돌 가능
3. **트러블슈팅 단순화**: 네트워크 문제 발생 시 원인 파악 용이
4. **Kubernetes 권장사항**: 대부분의 설치 가이드에서 OS 방화벽 비활성화

```bash
# SSH 접속
ssh -i ~/.ssh/id_rsa opc@<PUBLIC_IP>

# firewalld 중지 및 비활성화
sudo systemctl stop firewalld
sudo systemctl disable firewalld

# 상태 확인 (inactive 확인)
sudo systemctl status firewalld
```

## 📝 초기 시스템 설정

### 1. 시스템 업데이트

```bash
# 패키지 업데이트
sudo dnf update -y

# 재부팅 (커널 업데이트 시)
sudo reboot
```

### 2. 필수 패키지 설치

```bash
# 기본 도구
sudo dnf install -y \
  vim \
  git \
  curl \
  wget \
  net-tools \
  nc \
  telnet \
  tree \
  htop

# 시간 동기화
sudo systemctl enable --now chronyd
timedatectl status
```

### 3. SELinux 설정 (Oracle Linux)

```bash
# SELinux를 Permissive 모드로 설정
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# 확인
sestatus
```

### 4. 스왑 비활성화

Kubernetes는 스왑을 사용하지 않습니다:

```bash
# 스왑 비활성화
sudo swapoff -a

# 영구 비활성화
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 확인
free -h
```

### 5. 커널 파라미터 설정 (한 번에 모두)

```bash
# 커널 모듈 자동 로드 설정
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 모든 sysctl 파라미터 한 번에 설정
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes.conf
# Kubernetes 기본 요구사항
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1

# Tailscale 최적화
net.ipv4.tcp_keepalive_time = 60
net.ipv4.tcp_keepalive_intvl = 10
net.ipv4.tcp_keepalive_probes = 6

# 파일 디스크립터 증가 (많은 Pod/Container 지원)
fs.inotify.max_user_watches = 1048576
fs.inotify.max_user_instances = 1024

# Cilium CNI 기본 요구사항
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0

# Cilium eBPF 최적화
kernel.unprivileged_bpf_disabled = 1
net.core.bpf_jit_enable = 1
net.core.bpf_jit_harden = 0
net.core.bpf_jit_kallsyms = 1
kernel.bpf_stats_enabled = 1

# 연결 추적 테이블 크기 (eBPF LB용)
net.netfilter.nf_conntrack_max = 1000000
net.netfilter.nf_conntrack_buckets = 250000
net.netfilter.nf_conntrack_tcp_timeout_established = 86400
net.netfilter.nf_conntrack_tcp_timeout_close_wait = 3600

# 네트워크 성능 최적화
net.core.somaxconn = 32768
net.ipv4.tcp_max_syn_backlog = 8096
net.core.netdev_max_backlog = 5000
net.ipv4.neigh.default.gc_thresh1 = 8000
net.ipv4.neigh.default.gc_thresh2 = 12000
net.ipv4.neigh.default.gc_thresh3 = 16000
EOF

# 설정 적용
sudo sysctl --system

# 확인
lsmod | grep br_netfilter
lsmod | grep overlay
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.ipv4.ip_forward
```

## 📋 체크리스트

각 노드에서 확인:

**인프라 설정**
- [ ] Oracle Cloud 인스턴스 생성 완료
- [ ] Public IP 할당 확인
- [ ] Security List 규칙 추가
- [ ] SSH 접속 가능
- [ ] firewalld 비활성화 완료

**Block Volume 설정** (중요!)
- [ ] 150GB Block Volume 생성 및 연결
- [ ] `/var/lib/k8s-data`에 마운트 완료
- [ ] fstab 영구 설정 완료
- [ ] `/var/lib/containerd` 심볼릭 링크 생성
- [ ] `/var/lib/kubelet` 심볼릭 링크 생성
- [ ] `/var/log/pods` 심볼릭 링크 생성 ⚠️
- [ ] `/var/log/containers` 심볼릭 링크 생성 ⚠️
- [ ] `df /var/lib/containerd`로 150GB 볼륨 확인
- [ ] `df /var/log/pods`로 150GB 볼륨 확인

**시스템 설정**
- [ ] 시스템 업데이트 완료
- [ ] SELinux Permissive 모드
- [ ] 스왑 비활성화
- [ ] 커널 모듈 로드 (overlay, br_netfilter)
- [ ] sysctl 파라미터 적용

## 🔄 다음 단계

모든 노드가 준비되면:
→ [**02-setup-tailscale-network.md**](02-setup-tailscale-network.md) - Tailscale 메시 네트워크 구성

## 💡 팁

### ARM vs x86 선택
- **ARM64 추천**: 무료 리소스가 더 많음 (4 OCPU vs 2 OCPU)
- 대부분의 컨테이너 이미지가 ARM64 지원

### 인스턴스가 생성되지 않을 때
- **용량 부족**: 다른 가용 도메인(AD) 시도
- **시간대 변경**: 새벽 시간대 시도
- **Shape 변경**: AMD (x86) 시도

### SSH 접속 문제
```bash
# 권한 문제 해결
chmod 600 ~/.ssh/id_rsa

# 상세 로그 확인
ssh -vvv -i ~/.ssh/id_rsa opc@<PUBLIC_IP>
```

---

*다음 문서: [Tailscale 메시 네트워크 구성](02-setup-tailscale-network.md)*