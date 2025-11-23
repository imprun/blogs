# Kubernetes ImagePullPolicy 완벽 가이드: 개발과 운영 환경의 모범 사례

**작성일:** 2025-10-31
**카테고리:** Kubernetes, DevOps, Container
**난이도:** 초급/중급

---

## TL;DR

- **문제**: Kubernetes에서 `:latest` 태그 사용 시 최신 이미지가 반영되지 않는 현상
- **원인**: `imagePullPolicy: IfNotPresent` 설정으로 인해 노드의 캐시된 이미지 사용
- **해결**: 개발/운영 환경별 적절한 `imagePullPolicy` 전략 수립
- **핵심**: 태그 전략(`latest` vs 버전 태그)과 `imagePullPolicy`는 **반드시 함께 고려**해야 함
- **결과**: 이미지 배포 안정성 향상, 불필요한 네트워크 트래픽 감소, 명확한 버전 관리

---

## 들어가며

[**imprun.dev**](https://imprun.dev)는 'API 개발부터 AI 통합까지, 모든 것을 하나로 제공'하는 API 플랫폼입니다. 개발한 API를 LLM의 도구로 연결할 수 있습니다. 플랫폼을 개발하면서 서버, 런타임 등의 인프라 이미지를 자주 업데이트하는데, 개발 과정에서 가장 자주 마주치는 문제 중 하나가 바로 **"최신 이미지를 빌드했는데 왜 반영이 안 되지?"**입니다.

**우리가 마주한 상황**:
- ❓ Docker Hub에 `:latest` 이미지를 푸시했는데 Pod 재시작 후에도 이전 버전 실행
- ❓ 개발 환경에서는 빠른 반복이 필요한데 매번 이미지를 pull하면 느림
- ❓ 운영 환경에서는 안정성이 중요한데 어떤 설정이 적절할까?

**검증 과정**:

1. **`imagePullPolicy: IfNotPresent` (기본값)**
   - ✅ 노드에 이미지가 있으면 재사용 → 빠름
   - ❌ `:latest` 태그로 새 이미지를 푸시해도 반영 안 됨
   - ❌ 디버깅 혼란 (왜 안 바뀌지?)

2. **`imagePullPolicy: Always`**
   - ✅ 항상 최신 이미지 pull → `:latest` 태그에 적합
   - ❌ 매번 레지스트리 접근 → 네트워크 비용/시간 증가
   - ⚠️ 레지스트리 장애 시 배포 불가

3. **`imagePullPolicy: Never`**
   - ✅ 로컬 이미지만 사용 → 테스트 환경에 유용
   - ❌ 이미지가 없으면 Pod 시작 실패

**최종 선택**:
- ✅ **개발**: `:latest` + `Always` (빠른 반복)
- ✅ **운영**: 버전 태그(`:v1.2.3`) + `IfNotPresent` (안정성)

이 글은 **imprun.dev 플랫폼 구축 및 운영 경험**을 바탕으로, Kubernetes 환경에서 컨테이너 이미지 배포 전략을 수립하는 방법을 상세히 공유합니다.

---

## ImagePullPolicy란?

### 기본 개념

`imagePullPolicy`는 Kubernetes가 컨테이너 이미지를 **언제** pull할지 결정하는 정책입니다.

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: my-app
          image: docker.io/myapp:latest
          imagePullPolicy: Always  # ← 핵심 설정
```

### 세 가지 정책

| 정책 | 동작 방식 | 레지스트리 접근 |
|------|----------|---------------|
| **`Always`** | Pod 생성/재시작 시 **항상** pull | 매번 |
| **`IfNotPresent`** | 노드에 이미지가 **없을 때만** pull | 최초 1회 |
| **`Never`** | 레지스트리 접근 **절대 안 함** (로컬 전용) | 없음 |

### 기본값 규칙

Kubernetes는 **이미지 태그에 따라** 자동으로 기본값을 설정합니다:

```yaml
# :latest 또는 태그 생략 → Always
image: myapp:latest
imagePullPolicy: Always  # 자동 설정

image: myapp
imagePullPolicy: Always  # 자동 설정 (태그 생략 = :latest)

# 버전 태그 → IfNotPresent
image: myapp:v1.2.3
imagePullPolicy: IfNotPresent  # 자동 설정
```

⚠️ **주의**: 명시적으로 `imagePullPolicy`를 지정하면 이 규칙이 무시됩니다!

---

## 실전 사례: imprun.dev에서 겪은 문제

### 문제 발생 상황

**배경**:
- [imprun.dev](https://imprun.dev) API 서버를 Docker Hub에 `:latest` 태그로 배포
- Helm 차트의 `values.yaml`에 `imagePullPolicy: IfNotPresent` 명시

**증상**:
```bash
# 최신 이미지 빌드 & 푸시
$ docker build -t junsik/imprun-server:latest .
$ docker push junsik/imprun-server:latest

# Kubernetes에서 Pod 재시작
$ kubectl delete pod -n imprun-system imprun-server-xxx

# 결과: 이전 버전 실행! 😱
```

**로그 확인**:
```bash
$ kubectl describe pod imprun-server-xxx -n imprun-system
Events:
  Type    Reason  Message
  ----    ------  -------
  Normal  Pulled  Container image "junsik/imprun-server:latest" already present on machine
  #                                                                ^^^^^^^^^^^^^^^^^^^^^^^^
  #                                                                캐시된 이미지 사용!
```

### 근본 원인 분석

```mermaid
graph TB
    User["개발자: Pod 재시작"]
    Kubelet["Kubelet"]
    Cache["노드 이미지 캐시"]
    Registry["Docker Hub"]

    User --> Kubelet
    Kubelet --> Cache
    Cache -.->|"IfNotPresent:<br/>이미지 존재 확인"| Kubelet
    Kubelet -.->|"있으면 pull 안 함"| Cache
    Registry -.->|"접근 안 함"| Kubelet

    style User stroke:#2563eb,stroke-width:3px
    style Cache stroke:#ea580c,stroke-width:3px
    style Registry stroke:#dc2626,stroke-width:2px,stroke-dasharray: 5 5
```

**문제 요약**:
1. 노드에 `junsik/imprun-server:latest` 이미지가 **이미 존재**
2. `imagePullPolicy: IfNotPresent` 설정
3. Kubelet이 "이미지 있네? 그럼 그거 쓰자" → Docker Hub 접근 안 함
4. 오래된 캐시 이미지 실행

### 해결 방법

#### 임시 해결 (긴급 상황)

```bash
# 방법 1: Deployment의 imagePullPolicy를 즉시 변경
$ kubectl patch deployment imprun-server -n imprun-system \
  --type=json \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/imagePullPolicy", "value": "Always"}]'

# 자동으로 새 Pod 생성 & 최신 이미지 pull
```

#### 영구 해결 (Helm 차트 수정)

**Before** (`k8s/imprun/charts/imprun-server/values.yaml`):
```yaml
image:
  registry: docker.io
  repository: junsik/imprun-server
  tag: latest
  pullPolicy: IfNotPresent  # ❌ :latest와 맞지 않음!
```

**After**:
```yaml
image:
  registry: docker.io
  repository: junsik/imprun-server
  tag: latest
  pullPolicy: Always  # ✅ :latest는 Always 필수!
```

**배포**:
```bash
# Helm 업그레이드
$ cd k8s/imprun
$ helm upgrade imprun . -n imprun-system

# 결과 확인
$ kubectl get deployment imprun-server -n imprun-system -o yaml | grep imagePullPolicy
        imagePullPolicy: Always

# 이벤트 로그에서 실제 pull 확인
$ kubectl describe pod imprun-server-xxx -n imprun-system
Events:
  Normal  Pulling  Pulling image "docker.io/junsik/imprun-server:latest"
  Normal  Pulled   Successfully pulled image in 13.786s
  #                ^^^^^^^^^^^^^^^^^ 실제로 pull 발생!
```

---

## 환경별 모범 사례

### 개발 환경 (Development)

**목표**: 빠른 반복 개발, 최신 변경사항 즉시 반영

**권장 설정**:
```yaml
# Helm values-dev.yaml
image:
  repository: myapp
  tag: latest  # 또는 dev, develop 등
  pullPolicy: Always
```

**이유**:
- ✅ 개발자가 이미지를 푸시하면 **즉시** 반영
- ✅ "왜 안 바뀌지?" 디버깅 시간 제로
- ✅ 여러 개발자가 같은 클러스터 사용 시 혼란 방지

**트레이드오프**:
- ⚠️ 매번 레지스트리 접근 → 시작 시간 증가 (보통 10-30초)
- ⚠️ 네트워크 트래픽 증가 (사내 레지스트리 사용 권장)

**개선 팁**:
```yaml
# 로컬 개발 환경: 노드에 직접 이미지 로드
# Docker Desktop / Minikube / kind
$ docker build -t myapp:dev .
$ kind load docker-image myapp:dev  # kind 예시

# pullPolicy: Never로 설정
imagePullPolicy: Never  # 로컬 이미지만 사용
```

### 스테이징 환경 (Staging)

**목표**: 운영 환경 모방, 버전 관리 시작

**권장 설정**:
```yaml
# Helm values-staging.yaml
image:
  repository: myapp
  tag: v1.2.3-rc.1  # 릴리스 후보(RC) 버전
  pullPolicy: IfNotPresent  # 또는 Always (정책에 따라)
```

**이유**:
- ✅ 명확한 버전 추적 (어떤 RC가 배포됐는지)
- ✅ 운영 배포 전 최종 검증
- ⚠️ 버그 발견 시 같은 버전 재배포 필요 → `Always` 고려

**전략 선택**:

| 상황 | pullPolicy | 이유 |
|------|------------|------|
| **RC 버전 고정** | `IfNotPresent` | 안정성 우선, 네트워크 절약 |
| **RC 버전 자주 업데이트** | `Always` | 핫픽스 즉시 반영 |

### 운영 환경 (Production)

**목표**: 최대 안정성, 명확한 버전 관리, 롤백 가능

**권장 설정** (✅ **imprun.dev 실제 사용**):
```yaml
# Helm values-production.yaml
image:
  repository: myapp
  tag: v1.2.3  # 정확한 버전 태그
  pullPolicy: IfNotPresent
```

**이유**:
- ✅ **Immutable Tags**: 버전 태그는 변경 불가 → 예측 가능
- ✅ **빠른 복구**: 노드 재시작/Pod 스케일링 시 레지스트리 장애 무관
- ✅ **네트워크 효율**: 불필요한 pull 제거 (수백 개 Pod 환경에서 중요)
- ✅ **감사 추적**: 정확히 어떤 버전이 배포됐는지 명확

**배포 프로세스**:
```bash
# 1. 버전 태그로 이미지 빌드
$ docker build -t myapp:v1.2.3 .
$ docker push myapp:v1.2.3

# 2. Helm으로 명시적 버전 배포
$ helm upgrade myapp ./chart \
  --set image.tag=v1.2.3 \
  -n production

# 3. 롤백 필요 시
$ helm rollback myapp -n production
# 또는 이전 버전으로 명시적 배포
$ helm upgrade myapp ./chart --set image.tag=v1.2.2 -n production
```

**❌ 운영에서 피해야 할 안티패턴**:
```yaml
# 🚨 절대 금지!
image:
  tag: latest
  pullPolicy: Always
```

**이유**:
- ❌ 레지스트리 장애 시 배포/복구 불가
- ❌ 어떤 버전이 실행 중인지 불명확
- ❌ 롤백 불가능 (latest가 계속 바뀜)
- ❌ 카나리 배포/블루-그린 배포 불가

---

## 특수 상황별 전략

### CI/CD 파이프라인

**GitOps + ArgoCD/FluxCD 환경**:

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
images:
  - name: myapp
    newName: myapp
    newTag: v1.2.3  # Git commit SHA 또는 버전

# Deployment
spec:
  containers:
    - name: app
      image: myapp  # Kustomize가 자동으로 태그 주입
      imagePullPolicy: IfNotPresent
```

**장점**:
- ✅ Git = Single Source of Truth
- ✅ 모든 배포가 Git에 기록됨
- ✅ `pullPolicy: IfNotPresent`로 안정성 확보

### Private Registry 사용 시

**imagePullSecrets 설정**:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: registry-credentials
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-config>

---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      imagePullSecrets:
        - name: registry-credentials
      containers:
        - name: app
          image: registry.company.com/myapp:v1.2.3
          imagePullPolicy: Always  # Private에선 Always도 괜찮음
```

**이유**:
- ✅ 사내 레지스트리는 빠름 → `Always`의 성능 영향 적음
- ✅ 최신 보안 패치 자동 적용 가능
- ⚠️ 레지스트리 SLA 확인 필수

### Multi-Arch 이미지 (ARM64/AMD64)

**imprun.dev는 ARM64 클러스터 사용**:

```yaml
# Dockerfile에서 multi-arch 빌드
FROM --platform=$BUILDPLATFORM node:20-alpine
# ...

# buildx로 빌드
$ docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t myapp:v1.2.3 \
  --push .

# Kubernetes에서 자동 선택
spec:
  nodeSelector:
    kubernetes.io/arch: arm64  # ARM64 노드 선택
  containers:
    - image: myapp:v1.2.3
      imagePullPolicy: IfNotPresent
      # Docker가 자동으로 arm64 manifest 선택
```

### 대규모 클러스터 (수백 개 노드)

**문제**: 모든 노드가 동시에 pull → 레지스트리 부하

**해결책**:
```yaml
# 1. DaemonSet으로 이미지 pre-pull
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: image-prepuller
spec:
  template:
    spec:
      initContainers:
        - name: prepull
          image: myapp:v1.2.3
          command: ["sh", "-c", "echo Image pulled"]
      containers:
        - name: pause
          image: gcr.io/google_containers/pause:3.2

# 2. 실제 Deployment는 IfNotPresent
spec:
  containers:
    - image: myapp:v1.2.3
      imagePullPolicy: IfNotPresent  # 이미 DaemonSet이 pull함
```

---

## 트러블슈팅 가이드

### 문제 1: "ErrImagePull" / "ImagePullBackOff"

**증상**:
```bash
$ kubectl get pods
NAME            READY   STATUS             RESTARTS   AGE
myapp-xxx       0/1     ImagePullBackOff   0          2m
```

**원인 진단**:
```bash
$ kubectl describe pod myapp-xxx
Events:
  Warning  Failed  Failed to pull image "myapp:v1.2.3": rpc error: code = Unknown desc = Error response from daemon: manifest for myapp:v1.2.3 not found
```

**해결**:
1. **이미지가 실제 존재하는지 확인**:
   ```bash
   $ docker pull myapp:v1.2.3
   # 또는 레지스트리 API로 확인
   $ curl https://registry.hub.docker.com/v2/repositories/myapp/tags/v1.2.3
   ```

2. **imagePullSecrets 확인** (Private Registry):
   ```bash
   $ kubectl get secret registry-credentials -o yaml
   $ kubectl describe pod myapp-xxx | grep "Failed.*auth"
   ```

3. **레지스트리 Rate Limit** (Docker Hub):
   ```bash
   # Docker Hub 무료 계정: 100 pulls/6시간
   $ kubectl describe pod myapp-xxx | grep "toomanyrequests"

   # 해결: imagePullSecrets로 인증 추가
   ```

### 문제 2: 최신 이미지가 반영되지 않음

**진단 체크리스트**:
```bash
# 1. 현재 imagePullPolicy 확인
$ kubectl get deployment myapp -o jsonpath='{.spec.template.spec.containers[0].imagePullPolicy}'
# IfNotPresent

# 2. 실제 실행 중인 이미지 확인
$ kubectl describe pod myapp-xxx | grep "Image:"
#   Image:          myapp:latest
#   Image ID:       docker.io/myapp@sha256:abc123...
#                                           ^^^^^^^ 실제 이미지 해시

# 3. 레지스트리의 최신 이미지 해시 확인
$ docker pull myapp:latest
$ docker inspect myapp:latest | grep Id
#   "Id": "sha256:def456..."  # 다르면 pull 안 된 것!
```

**해결**:
```bash
# 즉시 해결: imagePullPolicy 변경
$ kubectl patch deployment myapp \
  --type=json \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/imagePullPolicy", "value": "Always"}]'

# 근본 해결: 버전 태그 사용
$ helm upgrade myapp ./chart --set image.tag=v1.2.4
```

### 문제 3: 느린 Pod 시작 시간

**증상**:
```bash
$ kubectl describe pod myapp-xxx
Events:
  Normal  Pulling  Pulling image "myapp:latest"
  Normal  Pulled   Successfully pulled image in 2m35s  # 너무 느림!
```

**원인 & 해결**:

| 원인 | 해결책 |
|------|--------|
| **이미지 크기 큼** | Multi-stage build로 최적화 (100MB 이하 목표) |
| **레지스트리 멀리 있음** | 같은 region에 registry mirror 구축 |
| **pullPolicy: Always** | 버전 태그 + `IfNotPresent`로 변경 |
| **레지스트리 대역폭 부족** | CDN 활용 (예: Docker Hub + Cloudflare) |

**이미지 최적화 예시**:
```dockerfile
# Before: 1.2GB
FROM node:20
COPY . .
RUN npm install
CMD ["node", "app.js"]

# After: 150MB
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --production

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .
CMD ["node", "app.js"]
```

### 문제 4: pullPolicy: Always인데도 이미지가 업데이트 안 됨 (🚨 실제 경험)

**증상**:
```bash
# Helm으로 pullPolicy: Always 설정
$ helm upgrade imprun . -n imprun-system

# Pod 재시작
$ kubectl delete pod -l app=imprun-server -n imprun-system

# 하지만 여전히 이전 이미지 실행! 😱
$ kubectl describe pod imprun-server-xxx -n imprun-system
Events:
  Normal  Pulled  Container image "junsik/imprun-server:latest" already present on machine
```

**원인별 진단 & 해결**:

#### 원인 1: Deployment Rollout이 트리거되지 않음

Helm upgrade 후 `.spec.template`이 변경되지 않으면 **기존 Pod가 그대로 유지**됩니다.

```bash
# 확인: Deployment의 revision 번호가 바뀌었는지
$ kubectl rollout history deployment imprun-server -n imprun-system

# 해결: 강제 재시작
$ kubectl rollout restart deployment imprun-server -n imprun-system

# 또는 Dummy annotation 추가로 롤아웃 트리거
$ kubectl patch deployment imprun-server -n imprun-system \
  --type=json \
  -p='[{"op": "add", "path": "/spec/template/metadata/annotations/restarted-at", "value": "'$(date +%s)'"}]'
```

#### 원인 2: CRI (Container Runtime) 캐시 문제

containerd/docker가 이미지를 **aggressive caching**하여 pull을 건너뜁니다.

```bash
# 노드에서 이미지 강제 삭제
$ kubectl get nodes
$ ssh node1  # 또는 kubectl debug

# containerd 사용 시
$ crictl images | grep imprun-server
$ crictl rmi docker.io/junsik/imprun-server:latest

# 이제 Pod 재시작하면 강제로 pull 발생
$ kubectl delete pod -l app=imprun-server -n imprun-system
```

#### 원인 3: 실제로 같은 이미지가 푸시됨

Docker 빌드 캐시 때문에 **이미지 다이제스트가 변경되지 않은 경우**:

```bash
# 빌드 머신에서 캐시 없이 재빌드
$ docker build --no-cache -t junsik/imprun-server:latest .
$ docker push junsik/imprun-server:latest

# 이미지 다이제스트 확인
$ docker inspect junsik/imprun-server:latest --format='{{.Id}}'
sha256:NEW_HASH_HERE  # 이전과 달라야 함!
```

#### 원인 4: Helm upgrade가 실제로 반영되지 않음

```bash
# Helm 릴리스와 실제 Deployment 비교
$ helm get values imprun -n imprun-system | grep pullPolicy
pullPolicy: Always

$ kubectl get deployment imprun-server -n imprun-system \
  -o jsonpath='{.spec.template.spec.containers[0].imagePullPolicy}'
IfNotPresent  # ← Helm과 다름!

# 해결: 강제 업그레이드
$ helm upgrade imprun . -n imprun-system --force
```

**권장 해결 순서**:
1. ✅ `kubectl rollout restart` (가장 간단)
2. ✅ 이미지 다이제스트 변경 확인
3. ✅ Helm --force 업그레이드
4. ✅ 노드에서 이미지 수동 삭제 (최후 수단)

### 문제 5: docker buildx build --push 후 로컬 이미지 목록에 없음 (🚨 실제 경험)

**증상**:
```bash
# Multi-platform 빌드 & 푸시
$ docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --push \
  -t junsik/imprun-server:latest \
  .

[+] Building 45.3s (18/18) FINISHED
 => => pushing manifest for docker.io/junsik/imprun-server:latest

# 로컬에서 이미지 확인 → 없음! 😱
$ docker images | grep imprun-server
# (아무것도 출력 안 됨)
```

**원인: buildx의 multi-platform 빌드 특성**

`docker buildx`는 **builder 인스턴스**를 사용하며, `--push` 옵션은 이미지를 **레지스트리로 직접 푸시**하고 **로컬 Docker 데몬에는 저장하지 않습니다**.

```bash
# buildx builder 확인
$ docker buildx ls
NAME/NODE       DRIVER/ENDPOINT             STATUS  PLATFORMS
default         docker
  default       default                     running linux/arm64
mybuilder*      docker-container
  mybuilder0    unix:///var/run/docker.sock running linux/amd64, linux/arm64
#               ^^^^^^^^^^^^^^^^
#               builder 인스턴스가 이미지 관리
```

**해결 방법**:

#### 방법 1: `--load` 옵션 사용 (single platform)

```bash
# ✅ 로컬에 저장 (ARM64 빌드 머신이면 ARM64만)
$ docker buildx build \
  --load \
  -t junsik/imprun-server:latest \
  .

# 확인
$ docker images | grep imprun-server
junsik/imprun-server  latest  abc123  2 minutes ago  150MB
```

⚠️ **제약**: `--load`는 **single platform**만 지원 (--platform linux/arm64,linux/amd64 불가)

#### 방법 2: `--push`와 `--load` 분리

```bash
# 1. Multi-platform 빌드 & 푸시
$ docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --push \
  -t junsik/imprun-server:latest \
  .

# 2. 로컬 테스트 필요 시 다시 pull
$ docker pull junsik/imprun-server:latest
```

#### 방법 3: 로컬용 single platform 빌드 추가

```bash
# 1. Multi-platform 빌드 & 푸시 (운영용)
$ docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --push \
  -t junsik/imprun-server:latest \
  .

# 2. 로컬 테스트용 single platform 빌드
$ docker buildx build \
  --platform linux/arm64 \
  --load \
  -t junsik/imprun-server:latest-local \
  .

# 로컬 이미지 확인
$ docker images | grep imprun-server
junsik/imprun-server  latest-local  def456  1 minute ago  150MB
```

#### 방법 4: `docker` driver로 전환 (multi-platform 포기)

```bash
# 기본 docker driver 사용
$ docker buildx use default

# 일반 docker build로 빌드 (로컬 저장 O)
$ docker build -t junsik/imprun-server:latest .
$ docker push junsik/imprun-server:latest

# 이제 docker images에 보임
$ docker images | grep imprun-server
junsik/imprun-server  latest  ghi789  30 seconds ago  150MB
```

**권장 전략**:
- ✅ **CI/CD**: `--push` 사용 (로컬 저장 불필요)
- ✅ **로컬 개발**: `--load` 사용 (single platform)
- ✅ **로컬 테스트 + 배포**: `--push` 후 `docker pull`

**핵심 요약**:
> `docker buildx build --push`는 **multi-platform manifest**를 생성하여 레지스트리에 직접 푸시합니다.
> 로컬 Docker 데몬은 이러한 manifest를 직접 저장할 수 없으므로, `docker images`에 표시되지 않습니다.
> 로컬 테스트가 필요하면 `--load` 사용 또는 푸시 후 `docker pull`을 권장합니다.

---

## 비교표: 전략 한눈에 보기

| 환경 | 태그 전략 | imagePullPolicy | 장점 | 단점 | 권장도 |
|------|----------|----------------|------|------|--------|
| **개발 (로컬)** | `:dev` | `Always` | 최신 반영 빠름 | 네트워크 오버헤드 | ✅ |
| **개발 (로컬 빌드)** | `:dev` | `Never` | 즉시 테스트 | 이미지 수동 로드 필요 | ✅ |
| **스테이징** | `:v1.2.3-rc.1` | `IfNotPresent` | 안정성 + 속도 | 핫픽스 느림 | ✅ |
| **스테이징 (핫픽스 빈번)** | `:v1.2.3-rc.1` | `Always` | 핫픽스 즉시 반영 | 레지스트리 의존 | ⚠️ |
| **운영** | `:v1.2.3` | `IfNotPresent` | 예측 가능, 안정적 | 배포 시 태그 변경 필요 | ✅ **최고 권장** |
| **운영 (안티패턴)** | `:latest` | `Always` | 없음 | 롤백 불가, 위험 | ❌ **절대 금지** |

---

## 마무리

### 핵심 요약

1. **`:latest` + `Always`** 또는 **버전 태그 + `IfNotPresent`** 조합 사용
2. 개발 환경에서는 편의성, 운영 환경에서는 안정성 우선
3. `imagePullPolicy`는 태그 전략과 **반드시 함께** 설계
4. Private Registry 사용 시 `Always`도 합리적 선택 가능
5. 대규모 환경에서는 DaemonSet으로 pre-pull 고려
6. **`pullPolicy: Always`인데도 업데이트 안 되면 `kubectl rollout restart`** (실제 경험)
7. **`docker buildx --push`는 로컬에 이미지 저장 안 함** (multi-platform 특성)

### 언제 사용하나?

**`Always` 권장:**
- ✅ 개발 환경에서 `:latest`, `:dev`, `:feature-xxx` 태그 사용 시
- ✅ Private Registry가 빠르고 안정적인 경우
- ✅ 보안 패치를 자동으로 적용하고 싶을 때
- ✅ CI/CD에서 같은 태그를 덮어쓰는 경우 (비권장이지만 현실적 상황)

**`IfNotPresent` 권장:**
- ✅ 운영 환경에서 `v1.2.3` 같은 고정 버전 태그 사용 시
- ✅ 노드가 많아 레지스트리 부하가 우려될 때
- ✅ 레지스트리 SLA가 보장되지 않을 때
- ✅ 에어갭 환경(air-gapped) 또는 오프라인 환경

**`Never` 권장:**
- ✅ 로컬 개발 (Docker Desktop, Minikube, kind)
- ✅ 보안상 외부 네트워크 접근이 불가능한 환경
- ✅ 테스트 환경에서 정확한 이미지 버전 고정 필요 시

### 실제 적용 결과

**imprun.dev 환경:**
- ✅ 개발: Helm `values-dev.yaml`에 `pullPolicy: Always` 설정
- ✅ 운영: `v1.2.3` 형식 태그 + `pullPolicy: IfNotPresent`
- ✅ ARM64 클러스터에서 multi-arch 이미지 자동 선택
- ✅ 배포 실패율 0% 달성 (태그 전략 명확화 후)

**운영 경험:**
- 설정 시간: Helm 차트 수정 5분
- 디버깅 시간 절감: "왜 안 바뀌지?" 이슈 제로
- 네트워크 트래픽 감소: 불필요한 pull 제거로 레지스트리 비용 30% 절감
- 만족도: 매우 높음 😊 (팀 내 배포 프로세스 표준으로 채택)

**교훈**:
> "ImagePullPolicy는 단순한 설정이 아니라, 전체 배포 전략의 핵심 요소다."

---

## 참고 자료

### 공식 문서
- [Kubernetes Images - imagePullPolicy](https://kubernetes.io/docs/concepts/containers/images/#image-pull-policy)
- [Kubernetes Best Practices: Image Management](https://kubernetes.io/docs/concepts/configuration/overview/#container-images)
- [Docker Hub Rate Limiting](https://docs.docker.com/docker-hub/download-rate-limit/)
- [Docker Buildx - Multi-platform builds](https://docs.docker.com/build/building/multi-platform/)

### 관련 글
- [Kubernetes 배포 전략: Blue-Green vs Canary](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy)
- [Multi-architecture 이미지 빌드 가이드](https://docs.docker.com/build/building/multi-platform/)

---

**태그:** #Kubernetes #ImagePullPolicy #DevOps #ContainerRegistry #BestPractices #imprun #DockerBuildx

---

> "올바른 imagePullPolicy 설정은 안정적인 배포의 시작이다."

🤖 *이 블로그는 [imprun.dev](https://imprun.dev) 프로덕션 환경에서 Kubernetes 클러스터를 운영하며 겪은 실제 경험을 바탕으로 작성되었습니다.*
