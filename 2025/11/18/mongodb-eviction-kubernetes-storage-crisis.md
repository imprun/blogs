# Kubernetes Ephemeral Storage 부족으로 인한 MongoDB Pod Eviction 트러블슈팅

> **작성일**: 2025년 11월 18일
> **카테고리**: Kubernetes, Storage, Troubleshooting
> **키워드**: Kubernetes, Ephemeral Storage, Pod Eviction, Oracle Cloud, Block Volume

## 요약

Oracle Cloud 환경에서 Kubernetes를 운영하던 중 MongoDB Pod가 `Evicted` 상태로 종료되는 장애가 발생했습니다. 근본 원인은 150GB 블록 볼륨을 연결했으나 실제로 마운트하지 않아 Kubernetes가 root 파티션만을 ephemeral-storage로 인식한 것이었습니다. 이 글에서는 문제의 원인과 해결 과정, 그리고 재발 방지를 위한 방안을 공유합니다.

## 문제 상황

### 증상

운영 중인 Kubernetes 클러스터에서 MongoDB StatefulSet Pod가 다음 메시지와 함께 종료되었습니다:

```
The node was low on resource: ephemeral-storage.
Threshold quantity: 4741241430, available: 4344280Ki.
```

Pod 상태는 `Evicted`로, Kubernetes가 노드의 ephemeral-storage 부족을 감지하고 자동으로 Pod를 제거한 것으로 확인되었습니다.

### 환경 구성

- **클라우드**: Oracle Cloud Infrastructure (OCI)
- **인스턴스**: ARM64 기반 무료 티어 (4 Core, 24GB RAM)
- **스토리지 구성**:
  - Boot Volume: 50GB
  - Block Volume: 150GB (컨테이너 데이터 전용)
- **Kubernetes 버전**: v1.28
- **컨테이너 런타임**: containerd

### 초기 진단

```bash
$ df -h /
Filesystem                  Size  Used Avail Use% Mounted on
/dev/mapper/ocivolume-root   30G   25G    5G  84% /

$ lsblk
sda                          50G
├─sda1                       50G
sdb                         150G
└─sdb1                      150G
```

디스크 사용률이 84%에 달했으며, 150GB 블록 볼륨(`/dev/sdb1`)이 연결되어 있으나 마운트되지 않은 상태였습니다.

## 근본 원인 분석

### Kubernetes Ephemeral Storage 계산 메커니즘

Kubernetes는 다음 경로들의 디스크 사용량을 ephemeral-storage로 계산합니다:

```
/var/lib/kubelet/pods/       # Pod emptyDir, 임시 데이터
/var/lib/containerd/         # Container writable layers
/var/log/pods/               # Pod 로그
/var/log/containers/         # Container 로그
```

중요한 점은 Kubernetes가 이 경로들이 위치한 **파일시스템**을 기준으로 용량을 계산한다는 것입니다. 별도의 볼륨이 마운트되지 않으면 root 파티션의 용량을 기준으로 판단합니다.

### 설정 오류

당초 의도한 구성:
```
Boot Volume (50GB)  → OS 및 시스템 파일
Block Volume (150GB) → 컨테이너 데이터 및 로그
```

실제 구성:
```
Boot Volume (30GB, root 파티션) → OS + 컨테이너 데이터 (84% 사용)
Block Volume (150GB) → 미사용 (마운트 안 됨)
```

Oracle Cloud에서 Block Volume을 인스턴스에 "연결(Attach)"하는 것과 실제로 파일시스템에 "마운트(Mount)"하는 것은 별개의 작업입니다. 블록 디바이스로 인식되더라도 마운트하지 않으면 사용할 수 없습니다.

## 해결 과정

### 1. 상태 확인

```bash
# 블록 디바이스 확인
$ lsblk
sdb                  8:16   0  150G  0 disk
└─sdb1               8:17   0  150G  0 part

# 현재 마운트 상태
$ df -h | grep containerd
(출력 없음 - 마운트되지 않음)
```

### 2. 볼륨 마운트

```bash
# Kubernetes 중지
systemctl stop kubelet

# 기존 데이터 백업
mv /var/lib/containerd /var/lib/containerd-backup

# 마운트 포인트 생성 및 볼륨 마운트
mkdir -p /var/lib/containerd
mount /dev/sdb1 /var/lib/containerd

# 기존 데이터 복사
rsync -av /var/lib/containerd-backup/ /var/lib/containerd/

# 영구 마운트 설정
echo "/dev/sdb1 /var/lib/containerd ext4 defaults,noatime 0 0" >> /etc/fstab
```

### 3. Kubernetes 재시작

```bash
systemctl start kubelet

# 상태 확인
kubectl get nodes
kubectl get pods -A
```

### 4. 결과 검증

```bash
$ df -h
Filesystem                  Size  Used Avail Use% Mounted on
/dev/mapper/ocivolume-root   30G   17G   13G  57% /
/dev/sdb1                   147G   11G  130G   8% /var/lib/containerd

$ kubectl get pods -n imp-system
NAME                  READY   STATUS    RESTARTS   AGE
imp-mongodb-mongodb-0 4/4     Running   0          5m
```

Root 파티션 사용률이 84%에서 57%로 감소했으며, containerd는 150GB 볼륨을 사용하도록 변경되었습니다. 모든 Pod가 정상 상태로 복구되었습니다.

## 추가 최적화

### local-path-provisioner 이동

분석 결과 `/opt/local-path-provisioner`가 5GB를 사용하며 root 파티션을 압박하고 있었습니다.

```bash
# 데이터 이동
mkdir -p /var/lib/containerd/local-path-provisioner
rsync -av /opt/local-path-provisioner/ /var/lib/containerd/local-path-provisioner/

# 심볼릭 링크 생성
mv /opt/local-path-provisioner /opt/local-path-provisioner.backup
ln -s /var/lib/containerd/local-path-provisioner /opt/local-path-provisioner
```

최종 디스크 사용률:
- Root 파티션: 57% (27% 개선)
- 150GB 볼륨: 11%

## 재발 방지 방안

### 1. 설치 전 구성 (권장)

Kubernetes 설치 **전**에 다음과 같이 구성하면 문제를 예방할 수 있습니다:

```bash
# Block Volume을 중앙 위치에 마운트
mount /dev/sdb1 /var/lib/k8s-data

# 심볼릭 링크 생성
ln -s /var/lib/k8s-data/containerd /var/lib/containerd
ln -s /var/lib/k8s-data/kubelet /var/lib/kubelet
ln -s /var/lib/k8s-data/pods-log /var/log/pods
ln -s /var/lib/k8s-data/containers-log /var/log/containers
```

### 2. 검증 절차

각 경로가 올바른 파일시스템을 사용하는지 확인합니다:

```bash
df /var/lib/containerd
df /var/lib/kubelet
df /var/log/pods
df /var/log/containers
```

모두 150GB 볼륨을 가리켜야 합니다.

### 3. 모니터링

```bash
# 디스크 사용량 모니터링
watch -n 300 'df -h | grep -E "Filesystem|containerd|ocivolume"'

# Eviction 이벤트 모니터링
kubectl get events --all-namespaces --field-selector reason=Evicted
```

## 교훈

### 1. 클라우드 블록 스토리지의 명시성

클라우드 환경에서 블록 스토리지를 "연결"하는 것과 "마운트"하는 것은 별개의 작업입니다. 연결만으로는 사용할 수 없으며, 명시적인 마운트 작업이 필요합니다.

### 2. Kubernetes의 Ephemeral Storage 계산

Kubernetes는 경로가 아닌 파일시스템을 기준으로 ephemeral-storage를 계산합니다. 심볼릭 링크를 사용하더라도 실제 데이터가 저장되는 파일시스템이 중요합니다.

### 3. 로그 디렉토리의 중요성

많은 가이드가 `/var/lib/containerd`와 `/var/lib/kubelet`만 다루지만, `/var/log/pods`와 `/var/log/containers`도 ephemeral-storage 계산에 포함됩니다. 특히 로그 레벨이 높거나 트래픽이 많은 환경에서는 로그가 빠르게 증가할 수 있습니다.

### 4. 검증의 중요성

설정 후에는 반드시 `df` 명령어로 실제 마운트 상태를 확인해야 합니다. 가정하지 말고 검증해야 합니다.

## 참고 자료

### 관련 문서
- [Kubernetes Ephemeral Storage 문제 해결 가이드](https://blog.imprun.dev/65) - 상세 기술 문서
- [Oracle Cloud 준비 및 설정](https://blog.imprun.dev/20) - Block Volume 설정 가이드

### 공식 문서
- [Kubernetes Node-pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)
- [Oracle Cloud Block Volumes](https://docs.oracle.com/en-us/iaas/Content/Block/home.htm)
- [Kubelet Configuration](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)

