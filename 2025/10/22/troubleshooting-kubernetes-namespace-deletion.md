# Kubernetes Namespace 삭제 전쟁: 18시간의 Terminating과의 싸움

## 들어가며

[**imprun.dev**](https://imprun.dev)는 Kubernetes 기반 서버리스 Cloud Function 플랫폼입니다. KubeBlocks 0.9에서 1.0.1로 업그레이드하는 과정에서 **네임스페이스가 18시간 동안 Terminating 상태에 멈추는** 악몽 같은 상황을 겪었습니다.

## TL;DR

**우리가 겪은 상황**:
- ❌ `kubectl delete namespace imprun-system` 실행
- ❌ 18시간 동안 Terminating 상태 지속
- ❌ **서버 재부팅해도 해결 안 됨**
- ✅ **Finalizer 제거**로 최종 해결

**핵심 교훈**: `kubectl delete`만으로는 부족합니다. **Finalizer를 이해하고 제거**해야 합니다.

이 글은 **imprun.dev 플랫폼 구축 경험**을 바탕으로, Namespace 삭제 문제의 원인과 해결 과정을 상세히 공유합니다.

---

## 1. 시작: 단순한 삭제가 18시간의 여정이 되다

```bash
# 2025-10-22 오전 10시
$ kubectl delete namespace imprun-system kb-system
namespace "imprun-system" deleted
namespace "kb-system" deleted

# 30분 후...
$ kubectl get ns
NAME            STATUS        AGE
imprun-system   Terminating   18h  # 😱
kb-system       Terminating   73d  # 😱😱😱
```

**예상 소요 시간**: 30초
**실제 소요 시간**: 18시간 이상 (그리고 계속 진행 중)

---

## 2. 첫 번째 시도: "시간이 해결해주겠지"

**오전 10시 30분**
```bash
$ kubectl get ns imprun-system
NAME            STATUS        AGE
imprun-system   Terminating   30m
```

> "Kubernetes가 알아서 정리하고 있을 거야. 조금만 기다려보자."

**오후 3시**
```bash
$ kubectl get ns imprun-system
NAME            STATUS        AGE
imprun-system   Terminating   5h
```

> "뭔가 잘못됐다..."

---

## 3. Finalizer 발견: 네임스페이스가 멈춘 진짜 이유

### Finalizer란 무엇인가?

**Kubernetes의 안전장치**: 리소스 삭제 전 반드시 완료해야 할 작업 목록

```yaml
metadata:
  finalizers:
    - kubernetes.io/pv-protection
    - instanceset.workloads.kubeblocks.io/finalizer
```

### Finalizer 동작 원리

```
정상 흐름:
1. kubectl delete 실행
2. deletionTimestamp 설정 → Terminating 상태
3. Controller가 finalizer 작업 처리
4. Controller가 finalizer 제거
5. 리소스 완전 삭제

우리의 상황:
1. kubectl delete 실행 ✅
2. deletionTimestamp 설정 ✅
3. Controller 작동 불가 ❌ ← 여기서 멈춤!
4. (영원히 대기...)
```

### 왜 Controller가 작동하지 않았나?

```bash
$ kubectl logs -n kb-system -l app.kubernetes.io/name=kubeblocks --tail=50 | grep ERROR

ERROR: deployments.apps "kubeblocks" is forbidden:
User "system:serviceaccount:kb-system:kubeblocks" cannot get resource "deployments"

ERROR: Failed to watch *v1.ComponentDefinition:
json: cannot unmarshal string into Go struct field SystemAccount
```

**근본 원인**:
- KubeBlocks 0.9 → 1.0.1 업그레이드 중 RBAC 권한 문제
- 0.9 버전 ComponentDefinition이 1.0.1 Controller와 호환 불가
- Controller가 작동하지 않아 Finalizer 처리 불가능

---

## 4. 두 번째 시도: "재부팅하면 되겠지"

**오후 4시**
```bash
$ sudo systemctl restart kubelet
```

**결과**: 변화 없음

**오후 5시**
```bash
# Swap 때문에 kubelet이 시작 실패했다는 것을 발견
$ sudo swapoff -a
$ sudo sed -i '/ swap / s/^/#/' /etc/fstab
$ sudo systemctl restart kubelet
```

**결과**: kubelet은 정상이지만 네임스페이스는 여전히 Terminating

**오후 6시** (절박한 마음으로)
```bash
$ sudo reboot
```

**재부팅 후**:
```bash
$ kubectl get ns | grep -E "imprun-system|kb-system"
imprun-system   Terminating   18h  # 여전히...
kb-system       Terminating   73d  # 여전히...
```

> **충격**: 재부팅해도 Terminating 상태는 그대로였다!

---

## 5. 세 번째 시도: Finalizer 수동 제거 작전

### 5-1. 네임스페이스 내 리소스 확인

```bash
$ kubectl get all,cm,secret,svc -n imprun-system
service/mongodb-mongodb-headless   ClusterIP   None   ...
configmap/mongodb-mongodb-env      2
secret/mongodb-conn-credential     Opaque      8
```

### 5-2. Finalizer 확인

```bash
$ kubectl get service mongodb-mongodb-headless -n imprun-system -o jsonpath='{.metadata.finalizers}'
["instanceset.workloads.kubeblocks.io/finalizer"]
```

**발견**: 모든 리소스에 KubeBlocks finalizer가 붙어있음!

### 5-3. Finalizer 제거 시도 #1

```bash
$ kubectl patch service mongodb-mongodb-headless -n imprun-system \
  -p '{"metadata":{"finalizers":null}}' --type=merge

service/mongodb-mongodb-headless patched ✅
```

**희망의 빛!** 하지만...

```bash
$ kubectl patch configuration.apps.kubeblocks.io mongodb-mongodb -n imprun-system \
  -p '{"metadata":{"finalizers":[]}}' --type=merge

Error: spec.systemAccounts[1].statement: Invalid value: "string":
spec.systemAccounts[1].statement in body must be of type object: "string"
```

**문제**: 0.9 버전 리소스는 1.0.1 validation을 통과하지 못함!

---

## 6. 네 번째 시도: CRD 자체를 제거하자

### 왜 CRD를 삭제하는가?

```
리소스 계층 구조:
CRD (ComponentDefinition 정의)
  ↓
ComponentDefinition 인스턴스들 (mongodb, postgresql 등)
  ↓
Cluster 인스턴스들
  ↓
Service, ConfigMap, Secret 등
```

**CRD를 삭제하면 모든 인스턴스가 자동 삭제됩니다!**

### 실행

```bash
# 1. CRD finalizer 제거
$ kubectl patch crd componentdefinitions.apps.kubeblocks.io \
  -p '{"metadata":{"finalizers":[]}}' --type=merge

customresourcedefinition.apiextensions.k8s.io/componentdefinitions.apps.kubeblocks.io patched

# 2. CRD 삭제
$ kubectl delete crd componentdefinitions.apps.kubeblocks.io

customresourcedefinition.apiextensions.k8s.io "componentdefinitions.apps.kubeblocks.io" deleted
```

### 결과 확인

```bash
$ kubectl get componentdefinition
Error from server (NotFound): Unable to list "apps.kubeblocks.io/v1,
Resource=componentdefinitions": the server could not find the requested resource
```

**성공!** CRD가 완전히 사라짐

---

## 7. 최종 돌파: 모든 리소스 Finalizer 제거

### 남은 리소스 정리

```bash
# KubeBlocks 관련 모든 리소스의 finalizer 제거
$ kubectl get configurations,backuppolicies,backupschedules -n imprun-system -o name | \
  xargs -I {} kubectl patch {} -n imprun-system -p '{"metadata":{"finalizers":[]}}' --type=merge

configuration.apps.kubeblocks.io/mongodb-mongodb patched
backuppolicy.dataprotection.kubeblocks.io/mongodb-mongodb-backup-policy patched
backupschedule.dataprotection.kubeblocks.io/mongodb-mongodb-backup-schedule patched
```

### 일반 리소스 Finalizer 제거

```bash
$ kubectl patch service mongodb-mongodb-headless -n imprun-system \
  -p '{"metadata":{"finalizers":null}}' --type=merge
$ kubectl patch configmap mongodb-mongodb-env -n imprun-system \
  -p '{"metadata":{"finalizers":null}}' --type=merge
$ kubectl patch secret mongodb-conn-credential -n imprun-system \
  -p '{"metadata":{"finalizers":null}}' --type=merge
```

### 네임스페이스 Finalizer 확인 및 제거

```bash
$ kubectl get namespace imprun-system -o jsonpath='{.spec.finalizers}'
["kubernetes"]

$ kubectl patch namespace imprun-system -p '{"spec":{"finalizers":[]}}' --type=merge
namespace/imprun-system patched
```

### 드디어...

```bash
$ sleep 5 && kubectl get ns | grep imprun-system
(출력 없음)

$ kubectl get ns | grep kb-system
(출력 없음)
```

🎉 **18시간 만에 드디어 삭제 성공!**

---

## 8. 핵심 교훈 및 실전 가이드

### 교훈 1: 재부팅은 만능이 아니다

> "Terminating 네임스페이스는 etcd에 메타데이터로 남아있습니다.
> 재부팅해도 etcd 데이터는 그대로이므로 문제가 해결되지 않습니다."

### 교훈 2: Finalizer를 이해해야 한다

```yaml
# Finalizer의 본질
metadata:
  finalizers: ["some-controller.io/cleanup"]

# 의미: "이 리소스를 삭제하기 전에 some-controller에게 물어봐야 해"
# Controller가 죽으면? → 영원히 Terminating
```

### 교훈 3: 버전 호환성 문제는 조기 발견해야 한다

**업그레이드 전 체크리스트**:
```bash
# 1. Controller 로그 확인
kubectl logs -n kb-system -l app.kubernetes.io/name=kubeblocks --tail=100

# 2. CRD 호환성 확인
kubectl get componentdefinition -o yaml | head -50

# 3. 문제 발견 시 즉시 롤백
```

---

## 9. 실전 해결 플레이북

### 상황 1: 네임스페이스가 Terminating에서 멈춤

```bash
# Step 1: 내부 리소스 확인
kubectl api-resources --verbs=list --namespaced -o name | \
  while read resource; do
    count=$(kubectl get $resource -n <namespace> 2>/dev/null | grep -v '^NAME' | wc -l)
    [ $count -gt 0 ] && echo "$resource: $count"
  done

# Step 2: Finalizer 있는 리소스 찾기
kubectl get all,cm,secret,svc -n <namespace> -o json | \
  jq '.items[] | select(.metadata.finalizers != null) |
  {kind: .kind, name: .metadata.name, finalizers: .metadata.finalizers}'

# Step 3: Finalizer 제거
kubectl patch <resource-type> <name> -n <namespace> \
  -p '{"metadata":{"finalizers":null}}' --type=merge

# Step 4: 네임스페이스 finalizer 제거
kubectl patch namespace <namespace> \
  -p '{"spec":{"finalizers":[]}}' --type=merge
```

### 상황 2: CRD 리소스가 삭제 안됨

```bash
# Validation 에러로 patch 실패 시 → CRD 자체를 삭제
kubectl patch crd <crd-name> -p '{"metadata":{"finalizers":[]}}' --type=merge
kubectl delete crd <crd-name>
```

### 상황 3: 모든 것이 실패했을 때 (최후의 수단)

```bash
# etcd에서 직접 삭제
MSYS_NO_PATHCONV=1 kubectl exec -n kube-system etcd-<node> -- \
  etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  del /registry/namespaces/<namespace-name>
```

⚠️ **주의**: etcd 직접 수정은 클러스터를 망가뜨릴 수 있습니다!

---

## 10. 예방이 최선

### KubeBlocks 업그레이드 시 올바른 순서

```bash
# ❌ 잘못된 방법 (우리가 했던 것)
helm upgrade kubeblocks kubeblocks/kubeblocks --version 1.0.1

# ✅ 올바른 방법
# 1. 백업
kubectl get clusters -A -o yaml > clusters-backup.yaml
kubectl get componentdefinitions -o yaml > compdef-backup.yaml

# 2. 0.9 ComponentDefinition 및 Addon 정리
kubectl delete addon --all
kubectl delete componentdefinition --all

# 3. 0.9 KubeBlocks 완전 제거
helm uninstall kubeblocks -n kb-system
kubectl delete crd -l app.kubernetes.io/name=kubeblocks

# 4. 1.0.1 CRD 설치
kubectl create -f https://github.com/apecloud/kubeblocks/releases/download/v1.0.1/kubeblocks_crds.yaml

# 5. 1.0.1 KubeBlocks 설치
helm install kubeblocks kubeblocks/kubeblocks \
  --namespace kb-system \
  --create-namespace \
  --version 1.0.1

# 6. Cluster 복구
kubectl apply -f clusters-backup.yaml
```

### Finalizer 관리 베스트 프랙티스

```yaml
# 1. 개발/테스트 환경에서는 terminationPolicy를 Delete로
spec:
  terminationPolicy: Delete  # 빠른 정리

# 2. 프로덕션에서는 DoNotTerminate로
spec:
  terminationPolicy: DoNotTerminate  # 실수 방지

# 3. 삭제 전 명시적 정리
kubectl delete clusters --all --wait=true -n <namespace>
kubectl delete namespace <namespace>
```

---

## 11. 마치며

### 이번 사건의 타임라인

- **10:00** - 네임스페이스 삭제 시작
- **10:30** - "조금만 기다려보자"
- **15:00** - "뭔가 잘못됐다"
- **16:00** - kubelet 재시작 (실패)
- **17:00** - Swap 문제 해결
- **18:00** - 서버 재부팅 (실패)
- **19:00** - Finalizer 발견
- **20:00** - 수동 제거 시작
- **21:00** - Validation 에러 발생
- **22:00** - CRD 삭제 시도
- **23:00** - 모든 리소스 정리
- **04:00 (다음날)** - 드디어 성공! 🎉

**총 소요 시간**: 18시간

### 배운 것

1. **Finalizer는 강력한 안전장치**지만, Controller가 죽으면 독이 된다
2. **재부팅은 만능이 아니다** - etcd는 영원하다
3. **버전 업그레이드는 신중하게** - 호환성 확인 필수
4. **CRD 계층 구조를 이해**하면 빠른 해결 가능
5. **문서화가 중요** - 다음에는 이 삽질을 안 하기 위해

### 당신이 지금 Terminating 지옥에 있다면

1. 숨을 고르세요 (재부팅은 소용없습니다)
2. 이 문서의 "실전 해결 플레이북"을 따라하세요
3. Finalizer를 찾고 제거하세요
4. CRD 계층을 이해하고 위에서부터 정리하세요
5. 그래도 안되면 etcd... (하지만 신중하게!)

---

**작성일**: 2025-10-22
**환경**: Kubernetes v1.28.15, KubeBlocks 0.9 → 1.0.1
**삽질 시간**: 18시간
**얻은 교훈**: 무가(無價)

**끝.**
