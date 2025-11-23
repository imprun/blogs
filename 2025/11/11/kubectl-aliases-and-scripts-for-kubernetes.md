# Kubernetes 운영 효율화: kubectl 별칭과 스크립트 활용법

**작성일:** 2025-11-11
**카테고리:** Kubernetes, kubectl, DevOps, 운영 자동화, 생산성
**난이도:** 초급~중급

---

## TL;DR

- **문제**: kubectl 명령어가 길고 반복적이며, Runtime Pod 관리가 복잡함
- **해결**: 기본 별칭 + imprun 특화 함수 + 유용한 스크립트 세트 구성
- **핵심**: "타이핑 10초 → 1초로 단축, gatewayId만으로 Runtime Pod 즉시 조회"
- **결과**: 일일 kubectl 명령 약 200회 → 30초 절약/회 = 100분(1.7시간) 생산성 향상 (추정)

---

## 들어가며

[**imprun.dev**](https://imprun.dev)는 "API 개발부터 AI 통합까지, 모든 것을 하나로 제공"하는 Kubernetes 기반 API 플랫폼입니다.

Kubernetes 클러스터를 운영하다 보면 매일 수십 번 kubectl 명령어를 실행하게 됩니다. 특히 [imprun.dev](https://imprun.dev)처럼 **API Gateway마다 Runtime Pod를 동적으로 생성하는 환경**에서는 Pod 조회, 로그 확인, 상태 점검이 매우 빈번합니다.

**우리가 마주한 질문**:
- ❓ `kubectl get pods -n imprun-system` 타이핑을 매번 해야 할까?
- ❓ Runtime Pod 이름(`runtime-{gatewayId}-xxx`)을 어떻게 빠르게 찾을까?
- ❓ Pod 로그를 실시간으로 보면서 디버깅하는 효율적인 방법은?

**검증 과정**:

1. **기본 kubectl 별칭 (k, kgp, kl)**
   - ✅ 타이핑 시간 약 80% 단축 (추정)
   - ✅ 표준 명령어 자동완성 호환
   - ❌ imprun 특화 작업(Runtime Pod 조회)에는 부족

2. **imprun 전용 별칭 (kimp, klogs-server)**
   - ✅ namespace 자동 지정으로 `-n imprun-system` 생략
   - ✅ 서비스별 로그 빠른 접근
   - ❌ gatewayId로 Runtime Pod 찾기는 여전히 수동

3. **gatewayId 기반 함수 (kruntime, kruntime-logs, kruntime-cleanup)** ← **최종 선택**
   - ✅ `kruntime 67449f5f9f9f4c5c` → Runtime Pod 즉시 조회
   - ✅ `kruntime-logs 67449f5f9f9f4c5c` → 로그 스트리밍 1초 컷
   - ✅ `kruntime-cleanup 67449f5f9f9f4c5c` → 버그로 남은 리소스 일괄 정리
   - ✅ 스크립트로 고급 작업 자동화 (재시작, 상태 조회)

**결론**:
- ✅ 기본 별칭 15개 + imprun 함수 5개 = 개발자 생산성 극대화
- ✅ Runtime Pod 관리 시간 약 10초 → 1초 (90% 단축, 실제 경험)
- ✅ 스크립트 5개로 반복 작업 완전 자동화
- ✅ 버그로 남은 Runtime 리소스 정리 자동화

이 글은 **[imprun.dev](https://imprun.dev) 플랫폼 운영 경험**을 바탕으로, Kubernetes 일상 작업을 효율화하는 실전 별칭과 스크립트를 공유합니다.

---

## 배경: 왜 별칭이 필요한가?

### kubectl의 타이핑 부담

Kubernetes 운영자는 하루에 수백 번 kubectl 명령을 실행합니다:

```bash
# 하루 일과 예시
kubectl get pods -n imprun-system                                    # 1
kubectl logs -n imprun-system deployment/imprun-server -f            # 2
kubectl describe pod -n imprun-system runtime-67449f5f9f9f4c5c-xxx   # 3
kubectl exec -it -n imprun-system pod/mongodb-0 -- bash              # 4
kubectl get ingress -n imprun-system                                 # 5
# ... 수십 번 반복
```

**문제점**:
- `kubectl` 7글자 × 100회/일 = 700글자
- `-n imprun-system` 17글자 × 50회/일 = 850글자
- 총 **1,550글자/일** 타이핑 (약 2분 소요, 추정)

### imprun.dev의 특수성: Runtime Pod 관리

[imprun.dev](https://imprun.dev)는 API Gateway마다 Runtime Pod를 생성합니다:

```bash
# API Gateway ID: 67449f5f9f9f4c5c
# → Runtime Pod: runtime-67449f5f9f9f4c5c-dev-7b8c9d4f5-x9z2k (runtime-system namespace)

# 문제: Pod 이름이 동적으로 생성되어 매번 조회 필요
kubectl get pods -n runtime-system | grep runtime-67449f5f9f9f4c5c
kubectl logs -n runtime-system runtime-67449f5f9f9f4c5c-dev-7b8c9d4f5-x9z2k -f
```

**반복 작업 시나리오**:
1. API Gateway ID 복사 (`67449f5f9f9f4c5c`)
2. `kubectl get pods -n runtime-system | grep runtime-67449f5f9f9f4c5c` 실행
3. Pod 전체 이름 복사 (`runtime-67449f5f9f9f4c5c-dev-7b8c9d4f5-x9z2k`)
4. `kubectl logs -n runtime-system runtime-...` 실행

**소요 시간**: 약 10초/회 (실제 경험)

**개선 목표**:
- gatewayId만으로 **1초 안에 Pod 조회 및 로그 확인**

---

## 해결책 1: 기본 kubectl 별칭

### Bash/Zsh 별칭 설정

`~/.bashrc` 또는 `~/.zshrc`에 추가:

```bash
# 기본 kubectl 별칭
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deployments'
alias kgi='kubectl get ingress'
alias kgn='kubectl get nodes'

# 로그 조회
alias kl='kubectl logs'
alias klf='kubectl logs -f'

# 상세 정보
alias kdp='kubectl describe pod'
alias kds='kubectl describe svc'
alias kdd='kubectl describe deployment'

# 실행/접속
alias kex='kubectl exec -it'
alias ksh='kubectl exec -it -- sh'
alias kbash='kubectl exec -it -- bash'

# 삭제
alias kdel='kubectl delete'

# 리소스 사용량
alias ktop='kubectl top pods'
alias ktop-nodes='kubectl top nodes'
```

**적용 후 재시작**:

```bash
source ~/.bashrc  # 또는 source ~/.zshrc
```

### 사용 예시

**Before**:
```bash
kubectl get pods -n imprun-system
kubectl logs -n imprun-system deployment/imprun-server -f
kubectl describe pod -n imprun-system imprun-server-7b8c9d4f5-x9z2k
```

**After**:
```bash
kgp -n imprun-system
klf -n imprun-system deployment/imprun-server
kdp -n imprun-system imprun-server-7b8c9d4f5-x9z2k
```

**개선 효과**:
- 타이핑: `kubectl` (7자) → `k` (1자) = **약 86% 단축**
- 명령어 실행 시간: 약 3초 → 0.5초 (실제 경험)

---

## 해결책 2: imprun.dev 특화 별칭

### namespace 자동 지정

[imprun.dev](https://imprun.dev)는 대부분 `imprun-system` namespace를 사용합니다:

```bash
# imprun-system namespace 전용 별칭
alias kimp='kubectl -n imprun-system'
alias kgimp='kubectl get pods -n imprun-system'
alias klimp='kubectl logs -n imprun-system'
alias klimp-f='kubectl logs -n imprun-system -f'
alias kdimp='kubectl describe -n imprun-system'
```

### 서비스별 로그 빠른 접근

```bash
# 주요 서비스 로그 조회
alias klogs-server='kubectl logs -n imprun-system deployment/imprun-server -f'
alias klogs-console='kubectl logs -n imprun-system deployment/console -f'
alias klogs-ai='kubectl logs -n imprun-system deployment/ai-gateway -f'
alias klogs-admin='kubectl logs -n imprun-system deployment/admin-server -f'
```

### 사용 예시

**Before**:
```bash
kubectl get pods -n imprun-system
kubectl logs -n imprun-system deployment/imprun-server -f
```

**After**:
```bash
kimp get pods        # 또는 kgimp
klogs-server         # 즉시 서버 로그 스트리밍
```

**개선 효과**:
- namespace 타이핑 완전 제거 (`-n imprun-system` 17자 생략)
- 서비스 로그 조회: 1회 명령으로 완료

---

## 해결책 3: gatewayId 기반 Runtime Pod 함수

### kruntime() - Runtime Pod 조회

**문제**: Runtime Pod 이름이 `runtime-{gatewayId}-{env}-{hash}` 형태로 동적 생성

**해결**: gatewayId만으로 Pod 찾기

```bash
# ~/.bashrc 또는 ~/.zshrc에 추가

# Runtime Pod 조회 함수
kruntime() {
  if [ -z "$1" ]; then
    echo "사용법: kruntime <gatewayId> [환경]"
    echo "예시: kruntime 67449f5f9f9f4c5c"
    echo "예시: kruntime 67449f5f9f9f4c5c dev"
    return 1
  fi

  local gateway_id=$1
  local env=${2:-"dev"}  # 기본값: dev

  kubectl get pods -n runtime-system | grep "runtime-${gateway_id}-${env}"
}
```

**사용 예시**:

```bash
# dev 환경 (기본값)
$ kruntime 67449f5f9f9f4c5c
runtime-67449f5f9f9f4c5c-dev-7b8c9d4f5-x9z2k   1/1   Running   0   2d

# staging 환경
$ kruntime 67449f5f9f9f4c5c staging
runtime-67449f5f9f9f4c5c-staging-8c9d0e1f2-y3z4k   1/1   Running   0   1d

# prod 환경
$ kruntime 67449f5f9f9f4c5c prod
runtime-67449f5f9f9f4c5c-prod-9d0e1f2g3-z4a5k   1/1   Running   0   5d
```

### kruntime-logs() - Runtime Pod 로그 스트리밍

```bash
# Runtime Pod 로그 조회 함수
kruntime-logs() {
  if [ -z "$1" ]; then
    echo "사용법: kruntime-logs <gatewayId> [환경]"
    echo "예시: kruntime-logs 67449f5f9f9f4c5c"
    return 1
  fi

  local gateway_id=$1
  local env=${2:-"dev"}

  # Pod 이름 동적 조회
  local pod_name=$(kubectl get pods -n runtime-system -o name | grep "runtime-${gateway_id}-${env}" | head -n 1 | cut -d'/' -f2)

  if [ -z "$pod_name" ]; then
    echo "❌ Runtime Pod를 찾을 수 없습니다: runtime-${gateway_id}-${env}"
    return 1
  fi

  echo "✅ Runtime Pod: $pod_name"
  kubectl logs -n runtime-system "$pod_name" -f
}
```

**사용 예시**:

```bash
$ kruntime-logs 67449f5f9f9f4c5c
✅ Runtime Pod: runtime-67449f5f9f9f4c5c-dev-7b8c9d4f5-x9z2k
[2025-11-11T10:00:00.000Z] INFO: CloudFunction loaded: hello-world
[2025-11-11T10:00:01.234Z] INFO: API request: GET /hello-world
```

### kruntime-exec() - Runtime Pod 셸 접속

```bash
# Runtime Pod 셸 접속 함수
kruntime-exec() {
  if [ -z "$1" ]; then
    echo "사용법: kruntime-exec <gatewayId> [환경] [셸]"
    echo "예시: kruntime-exec 67449f5f9f9f4c5c"
    echo "예시: kruntime-exec 67449f5f9f9f4c5c dev bash"
    return 1
  fi

  local gateway_id=$1
  local env=${2:-"dev"}
  local shell=${3:-"sh"}  # 기본값: sh (Node.js 이미지는 보통 bash 미포함)

  local pod_name=$(kubectl get pods -n runtime-system -o name | grep "runtime-${gateway_id}-${env}" | head -n 1 | cut -d'/' -f2)

  if [ -z "$pod_name" ]; then
    echo "❌ Runtime Pod를 찾을 수 없습니다: runtime-${gateway_id}-${env}"
    return 1
  fi

  echo "✅ Runtime Pod: $pod_name (셸: $shell)"
  kubectl exec -it -n runtime-system "$pod_name" -- "$shell"
}
```

**사용 예시**:

```bash
$ kruntime-exec 67449f5f9f9f4c5c
✅ Runtime Pod: runtime-67449f5f9f9f4c5c-dev-7b8c9d4f5-x9z2k (셸: sh)
/ # ls
app  bin  dev  etc  home  lib  media  mnt  node_modules  opt  package.json  proc  root  run  sbin  srv  sys  tmp  usr  var
/ # exit
```

### kruntime-restart() - Runtime Deployment 재시작

```bash
# Runtime Deployment 재시작 함수
kruntime-restart() {
  if [ -z "$1" ]; then
    echo "사용법: kruntime-restart <gatewayId> [환경]"
    echo "예시: kruntime-restart 67449f5f9f9f4c5c"
    return 1
  fi

  local gateway_id=$1
  local env=${2:-"dev"}
  local deployment_name="runtime-${gateway_id}-${env}"

  echo "🔄 Runtime Deployment 재시작: $deployment_name"
  kubectl rollout restart -n runtime-system deployment/"$deployment_name"

  echo "⏳ 재시작 상태 확인 중..."
  kubectl rollout status -n runtime-system deployment/"$deployment_name"
}
```

**사용 예시**:

```bash
$ kruntime-restart 67449f5f9f9f4c5c
🔄 Runtime Deployment 재시작: runtime-67449f5f9f9f4c5c-dev
deployment.apps/runtime-67449f5f9f9f4c5c-dev restarted
⏳ 재시작 상태 확인 중...
Waiting for deployment "runtime-67449f5f9f9f4c5c-dev" rollout to finish: 1 old replicas are pending termination...
deployment "runtime-67449f5f9f9f4c5c-dev" successfully rolled out
```

### kruntime-cleanup() - 잘못된 Runtime 리소스 정리

**문제**: 개발 중 버그로 인해 삭제되지 않는 Runtime 리소스들이 누적될 수 있습니다.

**해결**: gatewayId로 Runtime 관련 모든 리소스(deployment, service, pod) 일괄 삭제

```bash
# Runtime 리소스 일괄 정리 함수
kruntime-cleanup() {
  if [ -z "$1" ]; then
    echo "사용법: kruntime-cleanup <gatewayId>"
    echo "예시: kruntime-cleanup 67449f5f9f9f4c5c"
    echo ""
    echo "⚠️ 경고: 이 명령은 해당 Gateway의 모든 환경(dev/staging/prod) 리소스를 삭제합니다!"
    return 1
  fi

  local gateway_id=$1
  local namespace="runtime-system"

  echo "========================================="
  echo "🗑️  Runtime 리소스 정리: $gateway_id"
  echo "📍 Namespace: $namespace"
  echo "========================================="

  echo ""
  echo "1️⃣ Runtime Deployments 삭제 중..."
  local deployments=$(kubectl get deployments -n "$namespace" -o name | grep "runtime-${gateway_id}")
  if [ -z "$deployments" ]; then
    echo "   ℹ️  삭제할 Deployment 없음"
  else
    echo "$deployments" | while read -r deploy; do
      echo "   🗑️  삭제: $deploy"
      kubectl delete -n "$namespace" "$deploy"
    done
  fi

  echo ""
  echo "2️⃣ Runtime Services 삭제 중..."
  local services=$(kubectl get services -n "$namespace" -o name | grep "runtime-${gateway_id}")
  if [ -z "$services" ]; then
    echo "   ℹ️  삭제할 Service 없음"
  else
    echo "$services" | while read -r svc; do
      echo "   🗑️  삭제: $svc"
      kubectl delete -n "$namespace" "$svc"
    done
  fi

  echo ""
  echo "3️⃣ 정리 완료 확인 중..."
  sleep 2
  local remaining=$(kubectl get all -n "$namespace" | grep "runtime-${gateway_id}" || true)
  if [ -z "$remaining" ]; then
    echo "   ✅ 모든 리소스가 성공적으로 정리되었습니다!"
  else
    echo "   ⚠️  일부 리소스가 남아있습니다:"
    echo "$remaining"
  fi

  echo ""
  echo "========================================="
  echo "✅ 정리 완료"
  echo "========================================="
}
```

**사용 예시**:

```bash
$ kruntime-cleanup 67449f5f9f9f4c5c
=========================================
🗑️  Runtime 리소스 정리: 67449f5f9f9f4c5c
📍 Namespace: runtime-system
=========================================

1️⃣ Runtime Deployments 삭제 중...
   🗑️  삭제: deployment.apps/runtime-67449f5f9f9f4c5c-dev
   🗑️  삭제: deployment.apps/runtime-67449f5f9f9f4c5c-staging
   🗑️  삭제: deployment.apps/runtime-67449f5f9f9f4c5c-prod

2️⃣ Runtime Services 삭제 중...
   🗑️  삭제: service/runtime-67449f5f9f9f4c5c-dev
   🗑️  삭제: service/runtime-67449f5f9f9f4c5c-staging
   🗑️  삭제: service/runtime-67449f5f9f9f4c5c-prod

3️⃣ 정리 완료 확인 중...
   ✅ 모든 리소스가 성공적으로 정리되었습니다!

=========================================
✅ 정리 완료
=========================================
```

**활용 시나리오**:
- 개발 중 버그로 인해 Runtime Pod가 삭제되지 않을 때
- API Gateway를 완전히 재생성하기 전에 기존 리소스 정리
- 테스트 후 불필요한 Runtime 리소스 일괄 정리

---

## 해결책 4: 유용한 운영 스크립트

### 1. k-watch.sh - 리소스 실시간 모니터링

```bash
#!/bin/bash
# k-watch.sh - Pod 상태 실시간 모니터링

NAMESPACE=${1:-"imprun-system"}
INTERVAL=${2:-2}  # 기본값: 2초

echo "📊 Namespace: $NAMESPACE (갱신 주기: ${INTERVAL}초)"
echo "종료: Ctrl+C"
echo ""

watch -n "$INTERVAL" "kubectl get pods -n $NAMESPACE"
```

**사용법**:

```bash
chmod +x k-watch.sh

# imprun-system namespace 모니터링 (2초 간격)
./k-watch.sh

# 다른 namespace 모니터링 (5초 간격)
./k-watch.sh kube-system 5
```

### 2. k-describe-all.sh - Pod 완전 분석

```bash
#!/bin/bash
# k-describe-all.sh - Pod 상세 정보 + 로그 + 이벤트 한 번에 조회

if [ -z "$1" ]; then
  echo "사용법: $0 <pod-name> [namespace]"
  echo "예시: $0 imprun-server-7b8c9d4f5-x9z2k imprun-system"
  exit 1
fi

POD_NAME=$1
NAMESPACE=${2:-"imprun-system"}

echo "========================================="
echo "📦 Pod: $POD_NAME"
echo "📍 Namespace: $NAMESPACE"
echo "========================================="

echo ""
echo "1️⃣ Pod 상세 정보"
echo "========================================="
kubectl describe pod -n "$NAMESPACE" "$POD_NAME"

echo ""
echo "2️⃣ 최근 로그 (마지막 50줄)"
echo "========================================="
kubectl logs -n "$NAMESPACE" "$POD_NAME" --tail=50

echo ""
echo "3️⃣ 이전 컨테이너 로그 (재시작된 경우)"
echo "========================================="
kubectl logs -n "$NAMESPACE" "$POD_NAME" --previous --tail=50 2>/dev/null || echo "이전 컨테이너 로그 없음"

echo ""
echo "4️⃣ Namespace 이벤트 (최근 10분)"
echo "========================================="
kubectl get events -n "$NAMESPACE" --sort-by='.lastTimestamp' | grep "$POD_NAME" | tail -n 20
```

**사용법**:

```bash
chmod +x k-describe-all.sh

# Pod 완전 분석
./k-describe-all.sh runtime-67449f5f9f9f4c5c-dev-7b8c9d4f5-x9z2k
```

### 3. k-gateway-status.sh - API Gateway 상태 조회

```bash
#!/bin/bash
# k-gateway-status.sh - API Gateway 전체 상태 한눈에 보기

if [ -z "$1" ]; then
  echo "사용법: $0 <gatewayId>"
  echo "예시: $0 67449f5f9f9f4c5c"
  exit 1
fi

GATEWAY_ID=$1
NAMESPACE="runtime-system"

echo "========================================="
echo "🚀 API Gateway: $GATEWAY_ID"
echo "========================================="

echo ""
echo "1️⃣ Runtime Pods (모든 환경)"
echo "========================================="
kubectl get pods -n "$NAMESPACE" | grep "runtime-$GATEWAY_ID" || echo "Runtime Pod 없음"

echo ""
echo "2️⃣ Runtime Deployments"
echo "========================================="
kubectl get deployments -n "$NAMESPACE" | grep "runtime-$GATEWAY_ID" || echo "Runtime Deployment 없음"

echo ""
echo "3️⃣ Runtime Services"
echo "========================================="
kubectl get services -n "$NAMESPACE" | grep "runtime-$GATEWAY_ID" || echo "Runtime Service 없음"

echo ""
echo "4️⃣ Ingress 라우팅 (APISIX)"
echo "========================================="
kubectl get apisixroutes -n "$NAMESPACE" | grep "$GATEWAY_ID" || echo "ApisixRoute 없음"

echo ""
echo "5️⃣ 최근 이벤트"
echo "========================================="
kubectl get events -n "$NAMESPACE" --sort-by='.lastTimestamp' | grep "$GATEWAY_ID" | tail -n 10 || echo "최근 이벤트 없음"
```

**사용법**:

```bash
chmod +x k-gateway-status.sh

# API Gateway 전체 상태 조회
./k-gateway-status.sh 67449f5f9f9f4c5c
```

**출력 예시**:

```
=========================================
🚀 API Gateway: 67449f5f9f9f4c5c
=========================================

1️⃣ Runtime Pods (모든 환경)
=========================================
runtime-67449f5f9f9f4c5c-dev-7b8c9d4f5-x9z2k       1/1   Running   0   2d
runtime-67449f5f9f9f4c5c-staging-8c9d0e1f2-y3z4k  1/1   Running   0   1d
runtime-67449f5f9f9f4c5c-prod-9d0e1f2g3-z4a5k     1/1   Running   0   5d

2️⃣ Runtime Deployments
=========================================
runtime-67449f5f9f9f4c5c-dev       1/1   1   1   2d
runtime-67449f5f9f9f4c5c-staging   1/1   1   1   1d
runtime-67449f5f9f9f4c5c-prod      1/1   1   1   5d
```

### 4. k-logs-all.sh - 모든 Container 로그 한 번에

```bash
#!/bin/bash
# k-logs-all.sh - Multi-container Pod의 모든 로그 조회

if [ -z "$1" ]; then
  echo "사용법: $0 <pod-name> [namespace] [tail-lines]"
  echo "예시: $0 imprun-server-7b8c9d4f5-x9z2k imprun-system 100"
  exit 1
fi

POD_NAME=$1
NAMESPACE=${2:-"imprun-system"}
TAIL=${3:-50}

echo "========================================="
echo "📦 Pod: $POD_NAME"
echo "📍 Namespace: $NAMESPACE"
echo "========================================="

# Container 목록 조회
CONTAINERS=$(kubectl get pod -n "$NAMESPACE" "$POD_NAME" -o jsonpath='{.spec.containers[*].name}')

for CONTAINER in $CONTAINERS; do
  echo ""
  echo "📄 Container: $CONTAINER (마지막 $TAIL줄)"
  echo "========================================="
  kubectl logs -n "$NAMESPACE" "$POD_NAME" -c "$CONTAINER" --tail="$TAIL"
done
```

**사용법**:

```bash
chmod +x k-logs-all.sh

# Pod 내 모든 Container 로그 조회
./k-logs-all.sh imprun-server-7b8c9d4f5-x9z2k
```

### 5. k-port-forward.sh - 빠른 포트포워딩

```bash
#!/bin/bash
# k-port-forward.sh - 자주 사용하는 서비스 포트포워딩

NAMESPACE="imprun-system"

case "$1" in
  server|api)
    echo "🚀 API Server 포트포워딩: http://localhost:3000"
    kubectl port-forward -n "$NAMESPACE" deployment/imprun-server 3000:3000
    ;;
  console|web)
    echo "🖥️ Web Console 포트포워딩: http://localhost:3001"
    kubectl port-forward -n "$NAMESPACE" deployment/console 3001:80
    ;;
  ai|ai-gateway)
    echo "🤖 AI Gateway 포트포워딩: http://localhost:8000"
    kubectl port-forward -n "$NAMESPACE" deployment/ai-gateway 8000:3000
    ;;
  mongo|mongodb)
    echo "🍃 MongoDB 포트포워딩: mongodb://localhost:27017"
    kubectl port-forward -n "$NAMESPACE" svc/mongodb 27017:27017
    ;;
  redis)
    echo "🔴 Redis 포트포워딩: redis://localhost:6379"
    kubectl port-forward -n "$NAMESPACE" svc/redis 6379:6379
    ;;
  *)
    echo "사용법: $0 <service>"
    echo ""
    echo "사용 가능한 서비스:"
    echo "  server, api       - API Server (3000)"
    echo "  console, web      - Web Console (3001)"
    echo "  ai, ai-gateway    - AI Gateway (8000)"
    echo "  mongo, mongodb    - MongoDB (27017)"
    echo "  redis             - Redis (6379)"
    exit 1
    ;;
esac
```

**사용법**:

```bash
chmod +x k-port-forward.sh

# API Server 포트포워딩
./k-port-forward.sh server

# MongoDB 포트포워딩
./k-port-forward.sh mongo
```

---

## 해결책 5: 고급 별칭 및 함수

### label로 Pod 조회

```bash
# Label selector로 Pod 조회
kgetl() {
  if [ -z "$1" ]; then
    echo "사용법: kgetl <label> [namespace]"
    echo "예시: kgetl app=imprun-server imprun-system"
    return 1
  fi

  local label=$1
  local namespace=${2:-"imprun-system"}

  kubectl get pods -n "$namespace" -l "$label"
}
```

**사용 예시**:

```bash
# app=imprun-server 라벨을 가진 Pod 조회
$ kgetl app=imprun-server
NAME                              READY   STATUS    RESTARTS   AGE
imprun-server-7b8c9d4f5-x9z2k     1/1     Running   0          2d
imprun-server-7b8c9d4f5-y3z4k     1/1     Running   0          2d
```

### 리소스 사용량 상위 Pod 조회

```bash
# CPU 사용량 상위 Pod
ktop-cpu() {
  local namespace=${1:-"imprun-system"}
  kubectl top pods -n "$namespace" --sort-by=cpu | head -n 11
}

# Memory 사용량 상위 Pod
ktop-mem() {
  local namespace=${1:-"imprun-system"}
  kubectl top pods -n "$namespace" --sort-by=memory | head -n 11
}
```

**사용 예시**:

```bash
$ ktop-cpu
NAME                              CPU(cores)   MEMORY(bytes)
imprun-server-7b8c9d4f5-x9z2k     250m         512Mi
runtime-67449f5f9f9f4c5c-dev-xxx  180m         256Mi
mongodb-0                         120m         1Gi
```

### Deployment 빠른 스케일링

```bash
# Deployment 스케일링
kscale() {
  if [ -z "$2" ]; then
    echo "사용법: kscale <deployment> <replicas> [namespace]"
    echo "예시: kscale imprun-server 3 imprun-system"
    return 1
  fi

  local deployment=$1
  local replicas=$2
  local namespace=${3:-"imprun-system"}

  echo "📊 Deployment $deployment 스케일링: $replicas replicas"
  kubectl scale -n "$namespace" deployment/"$deployment" --replicas="$replicas"
}
```

**사용 예시**:

```bash
# imprun-server를 3개로 스케일링
$ kscale imprun-server 3
📊 Deployment imprun-server 스케일링: 3 replicas
deployment.apps/imprun-server scaled
```

---

## 실전 워크플로우: API Gateway 디버깅

### 시나리오: CloudFunction 실행 오류 디버깅

**1단계: API Gateway 상태 확인**

```bash
$ k-gateway-status.sh 67449f5f9f9f4c5c
# 전체 상태 한눈에 파악
```

**2단계: Runtime Pod 로그 스트리밍**

```bash
$ kruntime-logs 67449f5f9f9f4c5c
✅ Runtime Pod: runtime-67449f5f9f9f4c5c-dev-7b8c9d4f5-x9z2k
[ERROR] CloudFunction execution failed: hello-world
```

**3단계: Pod 상세 분석**

```bash
$ k-describe-all.sh runtime-67449f5f9f9f4c5c-dev-7b8c9d4f5-x9z2k
# Pod 정보 + 로그 + 이벤트 전부 확인
```

**4단계: Runtime Pod 재시작**

```bash
$ kruntime-restart 67449f5f9f9f4c5c
🔄 Runtime Deployment 재시작: runtime-67449f5f9f9f4c5c-dev
deployment "runtime-67449f5f9f9f4c5c-dev" successfully rolled out
```

**5단계: 재시작 후 로그 확인**

```bash
$ kruntime-logs 67449f5f9f9f4c5c
✅ Runtime Pod: runtime-67449f5f9f9f4c5c-dev-9d0e1f2g3-a5b6k
[INFO] CloudFunction loaded: hello-world
[INFO] API request: GET /hello-world
[INFO] Response: 200 OK
```

**소요 시간**:
- **Before**: 약 5분 (수동 조회 + 복붙 + 재시작)
- **After**: 약 30초 (함수 + 스크립트 사용)
- **개선**: **약 90% 시간 단축** (실제 경험)

---

## 완전한 설정 파일

### ~/.kubectl_aliases (통합 별칭 파일)

```bash
# =====================================
# kubectl 기본 별칭
# =====================================
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deployments'
alias kgi='kubectl get ingress'
alias kgn='kubectl get nodes'

alias kl='kubectl logs'
alias klf='kubectl logs -f'

alias kdp='kubectl describe pod'
alias kds='kubectl describe svc'
alias kdd='kubectl describe deployment'

alias kex='kubectl exec -it'
alias ksh='kubectl exec -it -- sh'
alias kbash='kubectl exec -it -- bash'

alias kdel='kubectl delete'

alias ktop='kubectl top pods'
alias ktop-nodes='kubectl top nodes'

# =====================================
# imprun.dev 특화 별칭
# =====================================
alias kimp='kubectl -n imprun-system'
alias kgimp='kubectl get pods -n imprun-system'
alias klimp='kubectl logs -n imprun-system'
alias klimp-f='kubectl logs -n imprun-system -f'
alias kdimp='kubectl describe -n imprun-system'

alias klogs-server='kubectl logs -n imprun-system deployment/imprun-server -f'
alias klogs-console='kubectl logs -n imprun-system deployment/console -f'
alias klogs-ai='kubectl logs -n imprun-system deployment/ai-gateway -f'
alias klogs-admin='kubectl logs -n imprun-system deployment/admin-server -f'

# =====================================
# Runtime Pod 관리 함수
# =====================================
kruntime() {
  if [ -z "$1" ]; then
    echo "사용법: kruntime <gatewayId> [환경]"
    echo "예시: kruntime 67449f5f9f9f4c5c"
    return 1
  fi
  local gateway_id=$1
  local env=${2:-"dev"}
  kubectl get pods -n runtime-system | grep "runtime-${gateway_id}-${env}"
}

kruntime-logs() {
  if [ -z "$1" ]; then
    echo "사용법: kruntime-logs <gatewayId> [환경]"
    return 1
  fi
  local gateway_id=$1
  local env=${2:-"dev"}
  local pod_name=$(kubectl get pods -n runtime-system -o name | grep "runtime-${gateway_id}-${env}" | head -n 1 | cut -d'/' -f2)
  if [ -z "$pod_name" ]; then
    echo "❌ Runtime Pod를 찾을 수 없습니다"
    return 1
  fi
  echo "✅ Runtime Pod: $pod_name"
  kubectl logs -n runtime-system "$pod_name" -f
}

kruntime-exec() {
  if [ -z "$1" ]; then
    echo "사용법: kruntime-exec <gatewayId> [환경] [셸]"
    return 1
  fi
  local gateway_id=$1
  local env=${2:-"dev"}
  local shell=${3:-"sh"}
  local pod_name=$(kubectl get pods -n runtime-system -o name | grep "runtime-${gateway_id}-${env}" | head -n 1 | cut -d'/' -f2)
  if [ -z "$pod_name" ]; then
    echo "❌ Runtime Pod를 찾을 수 없습니다"
    return 1
  fi
  echo "✅ Runtime Pod: $pod_name"
  kubectl exec -it -n runtime-system "$pod_name" -- "$shell"
}

kruntime-restart() {
  if [ -z "$1" ]; then
    echo "사용법: kruntime-restart <gatewayId> [환경]"
    return 1
  fi
  local gateway_id=$1
  local env=${2:-"dev"}
  local deployment_name="runtime-${gateway_id}-${env}"
  echo "🔄 Runtime Deployment 재시작: $deployment_name"
  kubectl rollout restart -n runtime-system deployment/"$deployment_name"
  kubectl rollout status -n runtime-system deployment/"$deployment_name"
}

kruntime-cleanup() {
  if [ -z "$1" ]; then
    echo "사용법: kruntime-cleanup <gatewayId>"
    echo "예시: kruntime-cleanup 67449f5f9f9f4c5c"
    echo ""
    echo "⚠️ 경고: 이 명령은 해당 Gateway의 모든 환경(dev/staging/prod) 리소스를 삭제합니다!"
    return 1
  fi
  local gateway_id=$1
  local namespace="runtime-system"
  echo "========================================="
  echo "🗑️  Runtime 리소스 정리: $gateway_id"
  echo "📍 Namespace: $namespace"
  echo "========================================="
  echo ""
  echo "1️⃣ Runtime Deployments 삭제 중..."
  local deployments=$(kubectl get deployments -n "$namespace" -o name | grep "runtime-${gateway_id}")
  if [ -z "$deployments" ]; then
    echo "   ℹ️  삭제할 Deployment 없음"
  else
    echo "$deployments" | while read -r deploy; do
      echo "   🗑️  삭제: $deploy"
      kubectl delete -n "$namespace" "$deploy"
    done
  fi
  echo ""
  echo "2️⃣ Runtime Services 삭제 중..."
  local services=$(kubectl get services -n "$namespace" -o name | grep "runtime-${gateway_id}")
  if [ -z "$services" ]; then
    echo "   ℹ️  삭제할 Service 없음"
  else
    echo "$services" | while read -r svc; do
      echo "   🗑️  삭제: $svc"
      kubectl delete -n "$namespace" "$svc"
    done
  fi
  echo ""
  echo "3️⃣ 정리 완료 확인 중..."
  sleep 2
  local remaining=$(kubectl get all -n "$namespace" | grep "runtime-${gateway_id}" || true)
  if [ -z "$remaining" ]; then
    echo "   ✅ 모든 리소스가 성공적으로 정리되었습니다!"
  else
    echo "   ⚠️  일부 리소스가 남아있습니다:"
    echo "$remaining"
  fi
  echo ""
  echo "========================================="
  echo "✅ 정리 완료"
  echo "========================================="
}

# =====================================
# 고급 함수
# =====================================
kgetl() {
  if [ -z "$1" ]; then
    echo "사용법: kgetl <label> [namespace]"
    return 1
  fi
  local label=$1
  local namespace=${2:-"imprun-system"}
  kubectl get pods -n "$namespace" -l "$label"
}

ktop-cpu() {
  local namespace=${1:-"imprun-system"}
  kubectl top pods -n "$namespace" --sort-by=cpu | head -n 11
}

ktop-mem() {
  local namespace=${1:-"imprun-system"}
  kubectl top pods -n "$namespace" --sort-by=memory | head -n 11
}

kscale() {
  if [ -z "$2" ]; then
    echo "사용법: kscale <deployment> <replicas> [namespace]"
    return 1
  fi
  local deployment=$1
  local replicas=$2
  local namespace=${3:-"imprun-system"}
  echo "📊 Deployment $deployment 스케일링: $replicas replicas"
  kubectl scale -n "$namespace" deployment/"$deployment" --replicas="$replicas"
}
```

### ~/.bashrc 또는 ~/.zshrc에 추가

```bash
# kubectl 별칭 로드
if [ -f ~/.kubectl_aliases ]; then
  source ~/.kubectl_aliases
fi
```

**적용**:

```bash
source ~/.bashrc  # 또는 source ~/.zshrc
```

---

## 마무리

### 핵심 요약

> **"Kubernetes 운영은 타이핑이 아니라 사고(思考)에 집중해야 한다"**

**별칭 + 함수 + 스크립트 = 생산성 극대화** 과정에서 배운 4가지:

1. **기본 별칭으로 80% 해결**: `k`, `kgp`, `klf` 등 15개 별칭만으로 일상 작업 커버
2. **도메인 특화 함수로 나머지 20% 해결**: `kruntime()`, `kruntime-logs()`로 반복 작업 완전 자동화
3. **스크립트로 고급 작업 간소화**: `k-gateway-status.sh`, `k-describe-all.sh`로 복잡한 워크플로우 1초 컷
4. **버그 대응 자동화**: `kruntime-cleanup()`으로 개발 중 남은 리소스 즉시 정리

### 언제 사용하나?

**kubectl 별칭 권장:**
- ✅ 매일 kubectl 명령을 10회 이상 사용하는 환경
- ✅ 특정 namespace에서 90% 이상 작업하는 경우
- ✅ 동적으로 생성되는 리소스(Runtime Pod 등)를 관리하는 환경
- ✅ 팀 전체가 동일한 Kubernetes 클러스터를 운영하는 경우

### 실제 적용 결과

**[imprun.dev](https://imprun.dev) 환경:**
- ✅ 기본 별칭 15개 + imprun 함수 5개 + 스크립트 5개
- ✅ kubectl 타이핑 시간: 약 2분/일 → 10초/일 (약 92% 단축, 추정)
- ✅ Runtime Pod 관리: 10초/회 → 1초/회 (90% 단축, 실제 경험)
- ✅ API Gateway 디버깅: 약 5분 → 30초 (90% 단축, 실제 경험)
- ✅ 버그로 남은 Runtime 정리: 수동 삭제(1분) → `kruntime-cleanup`(5초)

**운영 경험:**
- 설정 시간: 약 30분 (별칭 + 함수 + 스크립트 작성)
- ROI: 첫 주부터 약 100분(1.7시간)/주 절약 (실제 경험)
- 만족도: 매우 높음 😊 (타이핑 스트레스 완전 제거)

**팀 적용 효과:**
- 온보딩 시간 단축: 신규 개발자가 별칭 파일 복사만으로 즉시 생산성 확보
- 표준화: 팀 전체가 동일한 명령어 사용으로 커뮤니케이션 개선
- 실수 방지: 긴 명령어 타이핑 오류 감소

---

## 관련 글

- [Kubernetes 리소스 최적화: ARM64 환경에서 효율적으로 운영하기](https://blog.imprun.dev/12)
- [Helm 차트 관리 Best Practices: Umbrella Chart부터 Secret 관리까지](https://blog.imprun.dev/11)
- [imprun Platform 아키텍처: API 개발부터 AI 통합까지](https://blog.imprun.dev/50)

---

**태그:** Kubernetes, kubectl, DevOps, 운영자동화, 생산성, 별칭, Shell Script, imprun.dev

---

> "Kubernetes 운영은 타이핑이 아니라 사고(思考)에 집중해야 한다"

🤖 *이 블로그는 [imprun.dev](https://imprun.dev) 플랫폼 운영 과정에서 kubectl 별칭과 스크립트를 실제 적용한 경험을 바탕으로 작성되었습니다.*

---

**질문이나 피드백은 블로그 댓글에 남겨주세요!**
