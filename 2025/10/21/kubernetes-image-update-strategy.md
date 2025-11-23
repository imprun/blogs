# Kubernetes 이미지 업데이트 전략: ImagePullPolicy 완벽 가이드

> **작성일**: 2025-10-21
> **태그**: Kubernetes, Docker, DevOps, Helm
> **난이도**: 중급

## 들어가며

[**imprun.dev**](https://imprun.dev)는 Kubernetes 기반 서버리스 Cloud Function 플랫폼입니다. Runtime Exporter를 배포하는 과정에서 "Docker Hub에 새 이미지를 푸시했는데 왜 Pod가 이전 이미지를 계속 사용하지?"라는 문제에 직면했습니다.

**우리가 마주한 상황**:
- ✅ Docker Hub에 `junsik/imp-runtime-exporter:latest` 새 이미지 푸시 완료
- ✅ `helm upgrade` 실행하여 배포
- ❌ Pod가 **이전 이미지를 계속 사용**
- ❌ Pod 재시작해도 동일한 문제

**원인 파악**:
- Kubernetes는 **노드에 캐시된 이미지를 우선 사용**
- `ImagePullPolicy: IfNotPresent` 기본 설정 때문에 새 이미지를 받지 않음

이 글은 **imprun.dev 플랫폼 구축 경험**을 바탕으로, `ImagePullBackOff` 문제 해결 과정과 Kubernetes의 이미지 Pull 정책, 그리고 효과적인 업데이트 전략을 공유합니다.

## 문제 상황: ImagePullBackOff

### 증상

imprun.dev 플랫폼의 Runtime Exporter를 배포하는 중 다음과 같은 오류가 발생했습니다:

```bash
$ kubectl get pods -n imprun-system
NAME                                          READY   STATUS             RESTARTS   AGE
imprun-imp-runtime-exporter-7c888f666d-dr5sk  0/1     ImagePullBackOff   0          30m
```

### 원인 분석

Pod 상세 정보를 확인해보니:

```bash
$ kubectl describe pod imprun-imp-runtime-exporter-7c888f666d-dr5sk -n imprun-system

Events:
  Type     Reason     Age                Message
  ----     ------     ----               -------
  Warning  Failed     29m (x4 over 31m)  Failed to pull image "docker.io/junsik/imp-runtime-exporter:latest"
  Warning  Failed     29m (x4 over 31m)  Error: ErrImagePull
  Normal   BackOff    63s (x131)         Back-off pulling image
```

**문제 발견**:
1. ❌ 이미지명 오류: `junsik/imp-runtime-exporter` (잘못됨)
2. ✅ 올바른 이름: `junsik/imprun-runtime-exporter`
3. Helm values에서 `enabled: false`였지만 이전 배포가 활성화된 상태

## 해결 과정

### 1단계: Helm Values 수정

[k8s/imprun/values.yaml:260](../k8s/imprun/values.yaml#L260) 수정:

```yaml
imp-runtime-exporter:
  enabled: true  # false → true로 변경

  image:
    repository: junsik/imprun-runtime-exporter  # 이미지명 확인
    tag: latest
    pullPolicy: IfNotPresent
```

### 2단계: Helm 차트 업그레이드

```bash
# 의존성 업데이트
cd k8s/imprun
helm dependency update

# Dry-run으로 검증
helm upgrade imprun . --namespace imprun-system \
  --reuse-values \
  --set imp-runtime-exporter.enabled=true \
  --set imp-runtime-exporter.image.repository=junsik/imprun-runtime-exporter \
  --dry-run --debug

# 실제 적용
helm upgrade imprun . --namespace imprun-system \
  --reuse-values \
  --set imp-runtime-exporter.enabled=true \
  --set imp-runtime-exporter.image.repository=junsik/imprun-runtime-exporter
```

### 3단계: 검증

```bash
$ kubectl get pods -n imprun-system -l app=imp-runtime-exporter
NAME                                          READY   STATUS    RESTARTS   AGE
imprun-imp-runtime-exporter-b8b5b4cb5-whwtt   1/1     Running   0          18s

$ kubectl logs -n imprun-system -l app=imp-runtime-exporter --tail=5
[INFO] server - server 1 listened on 2342
```

✅ **성공!** Pod가 정상적으로 시작되었습니다.

---

## Kubernetes ImagePullPolicy 깊이 이해하기

### Pull Policy 종류

Kubernetes는 3가지 이미지 Pull 정책을 제공합니다:

| Policy | 동작 | 사용 사례 |
|--------|------|----------|
| **IfNotPresent** | 로컬에 이미지가 없을 때만 다운로드 | 프로덕션 (버전 태그 사용 시) |
| **Always** | Pod 시작 시마다 레지스트리 확인 | 개발 환경 (`latest` 태그 사용 시) |
| **Never** | 로컬 이미지만 사용 (없으면 실패) | 에어갭 환경 |

### 기본 동작 규칙

```yaml
# Case 1: latest 태그 → Always (자동)
image: nginx:latest
# imagePullPolicy: Always (기본값)

# Case 2: 명시적 버전 → IfNotPresent (자동)
image: nginx:1.21.0
# imagePullPolicy: IfNotPresent (기본값)

# Case 3: 명시적 지정 (우선순위 최상)
image: nginx:latest
imagePullPolicy: IfNotPresent  # 기본값 오버라이드
```

**주의**: Helm 등에서 명시적으로 `pullPolicy`를 지정하면 기본 규칙이 무시됩니다!

---

## 문제의 핵심: `latest` + `IfNotPresent` 조합

### 왜 이미지가 업데이트되지 않을까?

```yaml
# 현재 설정 (문제 상황)
image:
  repository: junsik/imprun-runtime-exporter
  tag: latest
  pullPolicy: IfNotPresent  # ⚠️ 여기가 문제!
```

**실행 시나리오**:

1. **첫 배포**:
   - 노드에 이미지 없음 → Docker Hub에서 `latest` 다운로드
   - 이미지 ID: `sha256:34d1478...`

2. **새 이미지 빌드 & 푸시**:
   ```bash
   docker build -t junsik/imprun-runtime-exporter:latest .
   docker push junsik/imprun-runtime-exporter:latest
   # Docker Hub의 latest → sha256:a7f3c21... (새 이미지)
   ```

3. **Pod 재시작**:
   - `IfNotPresent` 정책: "로컬에 `latest` 있네? 그거 써야지!"
   - ❌ 여전히 `sha256:34d1478...` (구 이미지) 사용

### 실제 확인 방법

```bash
$ kubectl describe pod <pod-name> -n <namespace> | grep "Image ID"
Image ID: docker.io/junsik/imprun-runtime-exporter@sha256:34d1478...
```

이 SHA 값이 Docker Hub의 최신 이미지와 다르다면 구버전을 사용하는 것입니다.

---

## 환경별 권장 전략

### 개발/스테이징 환경: 빠른 반복 우선

**전략**: `latest` 태그 + `Always` 정책

```yaml
# values-dev.yaml
imprun-server:
  image:
    repository: junsik/imprun-server
    tag: latest
    pullPolicy: Always  # 항상 최신 이미지

imprun-console:
  image:
    repository: junsik/imprun-console
    tag: latest
    pullPolicy: Always

imp-runtime-exporter:
  image:
    repository: junsik/imprun-runtime-exporter
    tag: latest
    pullPolicy: Always
```

**워크플로우**:
```bash
# 1. 코드 변경 후 이미지 빌드
docker build -t junsik/imprun-server:latest .
docker push junsik/imprun-server:latest

# 2. Deployment 재시작 (자동으로 최신 이미지 Pull)
kubectl rollout restart deployment/imprun-imprun-server -n imprun-system

# 3. 롤아웃 상태 확인
kubectl rollout status deployment/imprun-imprun-server -n imprun-system
```

**장점**:
- ✅ 빠른 개발 사이클
- ✅ 이미지 태그 관리 불필요
- ✅ 항상 최신 코드 반영

**단점**:
- ❌ 레지스트리 부하 증가
- ❌ Pod 시작 시간 증가 (이미지 다운로드)
- ❌ 네트워크 장애 시 실패 가능

### 프로덕션 환경: 안정성 & 추적성 우선

**전략**: 명시적 버전 태그 + `IfNotPresent` 정책

```yaml
# values-production.yaml
imprun-server:
  image:
    repository: junsik/imprun-server
    tag: v1.2.3  # 명시적 버전
    pullPolicy: IfNotPresent  # 성능 최적화

imprun-console:
  image:
    repository: junsik/imprun-console
    tag: v2.0.1
    pullPolicy: IfNotPresent

imp-runtime-exporter:
  image:
    repository: junsik/imprun-runtime-exporter
    tag: v1.0.5
    pullPolicy: IfNotPresent
```

**워크플로우**:
```bash
# 1. Git 태그 기반 버전 생성
VERSION=$(git describe --tags --abbrev=0)  # v1.2.4
docker build -t junsik/imprun-server:${VERSION} .
docker push junsik/imprun-server:${VERSION}

# 2. Helm으로 버전 업데이트
helm upgrade imprun . -n imprun-system \
  --reuse-values \
  --set imprun-server.image.tag=${VERSION}

# 3. 롤백 필요 시 (이전 버전으로)
helm rollback imprun -n imprun-system
# 또는 특정 버전 지정
helm upgrade imprun . -n imprun-system \
  --set imprun-server.image.tag=v1.2.3
```

**장점**:
- ✅ 명확한 버전 추적
- ✅ 쉬운 롤백
- ✅ 빠른 Pod 시작 (이미지 캐싱)
- ✅ 재현 가능한 배포 (Immutable Infrastructure)

**단점**:
- ❌ 배포마다 태그 변경 필요
- ❌ CI/CD 파이프라인 복잡도 증가

---

## 고급 전략: 하이브리드 접근

### Digest 기반 배포 (최고 수준의 불변성)

```yaml
image:
  repository: junsik/imprun-server
  # SHA256 digest 사용 (태그 무시)
  tag: latest@sha256:a7f3c21b8e9d4f1a0c3e5d7f9b1a3c5e7d9f1b3a5c7e9d1f3b5a7c9e1d3f5b7a
  pullPolicy: IfNotPresent
```

**장점**: 태그가 변경되어도 정확히 같은 이미지 사용 보장

### CI/CD 파이프라인 자동화

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    tags:
      - 'v*'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Extract version
        id: version
        run: echo "VERSION=${GITHUB_REF#refs/tags/}" >> $GITHUB_OUTPUT

      - name: Build and push
        run: |
          docker build -t junsik/imprun-server:${{ steps.version.outputs.VERSION }} .
          docker push junsik/imprun-server:${{ steps.version.outputs.VERSION }}

      - name: Update Helm
        run: |
          helm upgrade imprun ./k8s/imprun \
            -n imprun-system \
            --reuse-values \
            --set imprun-server.image.tag=${{ steps.version.outputs.VERSION }}
```

---

## 실무 팁 & 트러블슈팅

### Tip 1: 현재 사용 중인 이미지 확인

```bash
# 방법 1: kubectl describe
kubectl describe pod <pod-name> -n <namespace> | grep -E "Image:|Image ID:"

# 방법 2: jsonpath로 정확히 추출
kubectl get pod <pod-name> -n <namespace> \
  -o jsonpath='{.status.containerStatuses[0].imageID}'

# 방법 3: 모든 Deployment의 이미지 한눈에 보기
kubectl get deployment -n imprun-system \
  -o custom-columns=NAME:.metadata.name,IMAGE:.spec.template.spec.containers[0].image
```

### Tip 2: 강제로 이미지 재다운로드

```bash
# 방법 1: Deployment 재시작 (권장)
kubectl rollout restart deployment/<deployment-name> -n <namespace>

# 방법 2: Pod 직접 삭제 (Deployment가 자동 재생성)
kubectl delete pod -l app=<app-label> -n <namespace>

# 방법 3: 모든 Deployment 일괄 재시작
kubectl rollout restart deployment -n <namespace>
```

### Tip 3: Pull Policy 일괄 확인

```bash
# 네임스페이스의 모든 Deployment Pull Policy 확인
kubectl get deployment -n imprun-system -o json | \
  jq -r '.items[] | .metadata.name + ": " + .spec.template.spec.containers[0].imagePullPolicy'

# 출력 예시:
# imprun-imprun-server: IfNotPresent
# imprun-imprun-console: IfNotPresent
# imprun-imp-runtime-exporter: IfNotPresent
```

### Tip 4: ImagePullBackOff 디버깅

```bash
# 1. 이벤트 로그 확인
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | tail -20

# 2. Pod 상세 정보 (ImagePullBackOff 이유 확인)
kubectl describe pod <pod-name> -n <namespace>

# 3. 이미지 수동으로 Pull 테스트 (노드에서 직접)
docker pull <image-name>:<tag>

# 4. Image Pull Secret 확인 (Private 레지스트리)
kubectl get secret <pull-secret-name> -n <namespace> -o yaml
```

### Tip 5: 이미지 크기 최적화

이미지 업데이트 속도를 높이려면:

```dockerfile
# Bad: 큰 이미지 (1.2GB)
FROM node:18

# Good: Alpine 기반 (200MB)
FROM node:18-alpine

# Better: Multi-stage build (100MB)
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .
CMD ["node", "dist/index.js"]
```

---

## 체크리스트: 배포 전 확인사항

### 개발 환경
- [ ] `pullPolicy: Always` 설정 확인
- [ ] `tag: latest` 사용
- [ ] Docker Hub 푸시 후 `kubectl rollout restart` 실행
- [ ] 로그로 정상 시작 확인

### 프로덕션 환경
- [ ] 명시적 버전 태그 사용 (`v1.2.3` 형식)
- [ ] `pullPolicy: IfNotPresent` 설정
- [ ] Git 태그와 이미지 태그 일치 확인
- [ ] Helm values 파일에 버전 기록
- [ ] 롤백 계획 수립
- [ ] 모니터링/알림 설정 확인

---

## 결론

Kubernetes의 이미지 Pull 정책은 단순해 보이지만, 잘못 설정하면 "왜 업데이트가 안 되지?"라는 문제를 만들어냅니다. 핵심은:

1. **`latest` 태그 사용 시 → `pullPolicy: Always`**
2. **버전 태그 사용 시 → `pullPolicy: IfNotPresent`**
3. **환경에 맞는 전략 선택**: 개발은 속도, 프로덕션은 안정성

오늘 다룬 내용이 여러분의 Kubernetes 이미지 관리에 도움이 되길 바랍니다.

## 참고 자료

- [Kubernetes 공식 문서 - Images](https://kubernetes.io/docs/concepts/containers/images/)
- [Helm Best Practices - Values Files](https://helm.sh/docs/chart_best_practices/values/)
- [imprun.dev 프로젝트 구조](../CLAUDE.md)

---

**질문이나 피드백이 있다면 GitHub Issue로 남겨주세요!**

📧 Contact: [GitHub Issues](https://github.com/imprun/imprun.dev/issues)
