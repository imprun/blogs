# Kubernetes 리소스 최적화: ARM64 환경에서 효율적으로 운영하기

> **작성일**: 2025-10-21
> **태그**: Kubernetes, ARM64, Performance, HPA
> **난이도**: 중급~고급

## 들어가며

[**imprun.dev**](https://imprun.dev)는 Kubernetes 기반 서버리스 Cloud Function 플랫폼입니다. Oracle Cloud의 무료 ARM64 인스턴스에서 운영하면서, **제한된 리소스로 안정적인 서비스를 제공**하는 것이 가장 큰 과제였습니다.

**우리의 제약 조건**:
- **하드웨어**: 각 노드 4 cores, 24GB RAM (ARM64 Ampere)
- **워크로드**: MongoDB, Redis, API 서버, Web Console, Runtime Exporter
- **목표**: OOMKilled 없이 안정적인 서비스 제공

**겪었던 문제들**:
- ❌ MongoDB OOMKilled (메모리 부족으로 재시작 반복)
- ❌ API 서버 CPU 스로틀링 (트래픽 급증 시 응답 지연)
- ❌ Runtime Exporter 리소스 낭비 (과도한 메모리 예약)

**해결 과정**:
- ✅ **정밀한 리소스 튜닝**: requests/limits 최적화
- ✅ **HPA 자동 스케일링**: 트래픽 변동 대응
- ✅ **Pod Priority**: 중요도 기반 리소스 할당

이 글은 **imprun.dev 플랫폼 구축 경험**을 바탕으로, ARM64 제한된 리소스 환경에서 안정적인 서비스를 운영하기 위한 실전 최적화 기법을 공유합니다.

## 환경 소개

### 하드웨어 스펙

```yaml
클러스터 구성:
  노드 수: 3대
  CPU: 4 cores (ARM64 Ampere Altra)
  메모리: 24GB RAM
  스토리지: 200GB Block Storage
  네트워크: 10Gbps

총 가용 리소스:
  CPU: 12 cores
  메모리: 72GB
```

### API 게이트웨이 스택

```
┌─────────────────────────────────────────────┐
│ imprun-system Namespace                     │
├─────────────────────────────────────────────┤
│ • MongoDB (Replica Set x1)                  │
│ • Redis (Single Instance)                   │
│ • imprun-server (API Server)                │
│ • imprun-console (Next.js Web Console)      │
│ • imp-runtime-exporter (Metrics Exporter)   │
│                                             │
│ + 사용자 Runtime Pods (동적 생성/삭제)      │
└─────────────────────────────────────────────┘
```

---

## Part 1: ARM64 아키텍처 이해

### ARM64 vs x86_64 차이점

| 특성 | ARM64 | x86_64 |
|------|-------|--------|
| **전력 효율** | 🟢 우수 (같은 성능에 전력 50% 절감) | 🟡 보통 |
| **가격** | 🟢 저렴 (Oracle Cloud Free Tier) | 🔴 비쌈 |
| **Docker 이미지 호환성** | 🟡 제한적 (명시적 빌드 필요) | 🟢 완벽 |
| **성능** | 🟢 정수 연산 우수 | 🟢 부동소수점 우수 |
| **생태계** | 🟡 성장 중 | 🟢 성숙 |

### ARM64 전용 이미지 빌드

**Multi-platform 빌드 필수!**

```dockerfile
# Dockerfile (ARM64 호환 확인)
FROM node:18-alpine  # ✅ ARM64 지원

# ❌ 나쁜 예: x86_64 전용 바이너리
# COPY ./bin/x86_64/app /app

# ✅ 좋은 예: 소스 빌드
COPY package*.json ./
RUN npm ci --only=production
```

**Docker Buildx로 멀티 플랫폼 빌드**:

```bash
# Buildx 활성화 (한 번만)
docker buildx create --name multiarch --use
docker buildx inspect --bootstrap

# ARM64 + AMD64 동시 빌드 & 푸시
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t junsik/imprun-server:latest \
  --push \
  .
```

**Manifest 확인**:
```bash
docker manifest inspect junsik/imprun-server:latest | jq '.manifests[].platform'

# 출력:
# {
#   "architecture": "amd64",
#   "os": "linux"
# }
# {
#   "architecture": "arm64",
#   "os": "linux",
#   "variant": "v8"
# }
```

### Node Selector 설정

**ARM64 노드에만 배포**:

```yaml
# charts/imprun-server/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/arch: arm64  # ARM64 노드 선택

      # 또는 특정 노드 타입 지정
      # cloud.google.com/gke-nodepool: arm-pool
```

**혼합 클러스터 (ARM + x86)**:

```yaml
# values-production.yaml
imprun-server:
  nodeSelector:
    kubernetes.io/arch: arm64  # ARM64 선호

  tolerations:  # x86도 허용 (필요시)
  - key: kubernetes.io/arch
    operator: Equal
    value: amd64
    effect: NoSchedule
```

---

## Part 2: Resource Requests/Limits 튜닝

### 리소스 설정의 중요성

**Requests**: "이만큼은 보장해주세요" (스케줄링 기준)
**Limits**: "이 이상은 절대 안 돼요" (제한)

```yaml
resources:
  requests:
    cpu: 200m      # 0.2 core 보장
    memory: 512Mi  # 512MB 보장
  limits:
    cpu: 1000m     # 최대 1 core
    memory: 2Gi    # 최대 2GB (초과 시 OOMKilled)
```

### 컴포넌트별 리소스 전략

#### 1. MongoDB (Stateful, 메모리 집약적)

**초기 설정 (실패 사례)**:
```yaml
# ❌ 너무 높은 설정
mongodb:
  resources:
    requests:
      cpu: 500m
      memory: 2Gi  # 노드당 24GB인데 2Gi x 3 = 6GB
    limits:
      cpu: 2000m
      memory: 4Gi

# 문제: 다른 Pod 스케줄링 실패!
```

**최적화 후**:
```yaml
# ✅ 실제 사용량 기반 조정
mongodb:
  replicas: 1  # ARM64 환경에서는 단일 인스턴스

  resources:
    requests:
      cpu: 200m     # 실제 사용량: 평균 150m
      memory: 512Mi # 실제 사용량: 평균 400Mi
    limits:
      cpu: 500m     # 버스트 허용
      memory: 1Gi   # OOM 방지

  # WiredTiger 캐시 크기 제한
  extraEnvVars:
  - name: MONGO_INITDB_ARGS
    value: "--wiredTigerCacheSizeGB 0.5"  # 512MB 캐시
```

**모니터링으로 검증**:
```bash
# 실제 사용량 확인
kubectl top pod mongodb-mongodb-0 -n imprun-system

# 출력:
# NAME                CPU    MEMORY
# mongodb-mongodb-0   120m   380Mi  ← requests보다 훨씬 낮음!
```

#### 2. Redis (메모리 제한 중요)

```yaml
redis:
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi  # ⚠️ 중요: Redis는 메모리 제한 필수!

  # Redis 자체 메모리 제한
  config:
    maxmemory: "400mb"  # limits보다 20% 낮게
    maxmemory-policy: "allkeys-lru"  # 메모리 부족 시 LRU 제거
```

**왜 낮게 설정?**
- Redis는 메모리 제한 초과 시 OOMKilled → 데이터 손실
- 여유분 확보로 안전 마진 제공

#### 3. API Server (CPU 집약적)

```yaml
imprun-server:
  resources:
    requests:
      cpu: 200m     # Node.js 단일 스레드 기준
      memory: 512Mi
    limits:
      cpu: 1000m    # 버스트 허용 (NestJS 부팅 시)
      memory: 2Gi   # 메모리 릭 방지

  # Node.js 최적화
  env:
  - name: NODE_OPTIONS
    value: "--max-old-space-size=1536"  # 1.5GB (limit의 75%)
```

**부팅 시 CPU 버스트 패턴**:
```
부팅 시:   800m ~ 1000m (TypeScript 컴파일, 초기화)
정상 운영: 150m ~ 300m (요청 처리)
```

#### 4. Web Console (Next.js)

```yaml
imprun-console:
  resources:
    requests:
      cpu: 100m     # SSR이 아닌 정적 서빙
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi

  # Next.js Standalone 빌드 (메모리 절약)
  env:
  - name: NEXT_TELEMETRY_DISABLED
    value: "1"
```

#### 5. Runtime Exporter (경량)

```yaml
imp-runtime-exporter:
  resources:
    requests:
      cpu: 50m      # 메트릭 수집만
      memory: 128Mi
    limits:
      cpu: 200m
      memory: 256Mi
```

### 리소스 튜닝 프로세스

**1단계: 보수적 시작**
```yaml
resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

**2단계: 모니터링 (1주일)**
```bash
# VictoriaMetrics 쿼리 (또는 Prometheus)
# CPU 사용률 (P95)
histogram_quantile(0.95,
  rate(container_cpu_usage_seconds_total{pod=~"imprun-server.*"}[5m])
)

# 메모리 사용률 (MAX)
max_over_time(
  container_memory_working_set_bytes{pod=~"imprun-server.*"}[7d]
)
```

**3단계: 조정**
```yaml
# CPU P95 = 180m → requests: 200m (10% 여유)
# 메모리 MAX = 450Mi → requests: 512Mi (15% 여유)

resources:
  requests:
    cpu: 200m
    memory: 512Mi
  limits:
    cpu: 1000m     # 버스트 허용 (5배)
    memory: 1Gi    # OOM 방지 (2배)
```

**4단계: 검증**
```bash
# OOMKilled 이벤트 확인
kubectl get events -n imprun-system \
  --field-selector reason=OOMKilled

# CPU Throttling 확인
kubectl top pod -n imprun-system --containers
```

---

## Part 3: Horizontal Pod Autoscaler (HPA)

### HPA 기본 개념

**목표**: 부하에 따라 Pod 수 자동 조절

```
트래픽 증가 → CPU 사용률 상승 → HPA가 Pod 추가
트래픽 감소 → CPU 사용률 하락 → HPA가 Pod 제거
```

### Metrics Server 설치 (필수)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# ARM64 노드에서 TLS 오류 시
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'

# 동작 확인
kubectl top nodes
kubectl top pods -n imprun-system
```

### API Server HPA 설정

```yaml
# charts/imprun-server/templates/hpa.yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "imprun-server.fullname" . }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "imprun-server.fullname" . }}

  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}

  metrics:
  # CPU 기반 스케일링
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}

  # 메모리 기반 스케일링 (선택)
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}

  # 스케일링 동작 제어
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60  # 1분간 안정화 후 스케일 업
      policies:
      - type: Percent
        value: 50   # 한 번에 50% 증가
        periodSeconds: 60
      - type: Pods
        value: 2    # 또는 최대 2개 추가
        periodSeconds: 60
      selectPolicy: Min  # 둘 중 작은 값 선택

    scaleDown:
      stabilizationWindowSeconds: 300  # 5분간 안정화 후 스케일 다운
      policies:
      - type: Percent
        value: 25   # 한 번에 25%만 감소
        periodSeconds: 60
{{- end }}
```

**Values 설정**:

```yaml
# values-production.yaml
imprun-server:
  autoscaling:
    enabled: true
    minReplicas: 2      # 최소 2개 (HA)
    maxReplicas: 6      # 최대 6개 (4 cores x 3 nodes = 12 cores 고려)
    targetCPUUtilizationPercentage: 70     # CPU 70% 시 스케일 업
    targetMemoryUtilizationPercentage: 80  # 메모리 80% 시 스케일 업
```

### HPA 동작 시뮬레이션

**부하 테스트**:
```bash
# Apache Bench로 부하 생성
ab -n 10000 -c 100 https://api.imprun.dev/v1/health/liveness

# HPA 상태 모니터링
kubectl get hpa -n imprun-system -w
```

**예상 동작**:
```
NAME                REFERENCE                      TARGETS   MINPODS   MAXPODS   REPLICAS
imprun-server-hpa   Deployment/imprun-server       45%/70%   2         6         2

# 부하 증가 (1분 후)
imprun-server-hpa   Deployment/imprun-server       85%/70%   2         6         3  ← +1 Pod

# 부하 지속 (2분 후)
imprun-server-hpa   Deployment/imprun-server       78%/70%   2         6         4  ← +1 Pod

# 부하 감소 (5분 안정화 후)
imprun-server-hpa   Deployment/imprun-server       40%/70%   2         6         3  ← -1 Pod
```

### 커스텀 메트릭 기반 HPA (고급)

**사용 사례**: API 요청 수(RPS) 기반 스케일링

**1. Prometheus Adapter 설치**:
```bash
helm install prometheus-adapter prometheus-community/prometheus-adapter \
  -n monitoring \
  --set prometheus.url=http://vmsingle-vm.monitoring.svc:8428
```

**2. 커스텀 메트릭 정의**:
```yaml
# prometheus-adapter-values.yaml
rules:
- seriesQuery: 'http_requests_total{namespace="imprun-system",pod=~"imprun-server.*"}'
  resources:
    overrides:
      namespace: {resource: "namespace"}
      pod: {resource: "pod"}
  name:
    matches: "^(.*)_total$"
    as: "${1}_per_second"
  metricsQuery: 'rate(<<.Series>>{<<.LabelMatchers>>}[2m])'
```

**3. HPA에서 사용**:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"  # Pod당 100 RPS 초과 시 스케일 업
```

---

## Part 4: 리소스 최적화 실전 팁

### 1. QoS (Quality of Service) 클래스 이해

Kubernetes는 Pod를 3가지 QoS 클래스로 분류:

```yaml
# Guaranteed (최고 우선순위)
# requests == limits
resources:
  requests:
    cpu: 500m
    memory: 1Gi
  limits:
    cpu: 500m
    memory: 1Gi

# Burstable (중간 우선순위) ← 권장!
# requests < limits
resources:
  requests:
    cpu: 200m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 2Gi

# BestEffort (최저 우선순위)
# requests/limits 없음
resources: {}
```

**리소스 부족 시 제거 우선순위**:
```
BestEffort (먼저 죽음) > Burstable > Guaranteed (마지막까지 살아남음)
```

**권장**: Stateful 워크로드(MongoDB)는 Guaranteed, Stateless(API)는 Burstable

### 2. PodDisruptionBudget (PDB)

**목적**: 자발적 중단 시 최소 가용성 보장

```yaml
# charts/imprun-server/templates/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: imprun-server-pdb
spec:
  minAvailable: 1  # 최소 1개 Pod는 항상 Running
  selector:
    matchLabels:
      app.kubernetes.io/name: imprun-server

  # 또는 maxUnavailable 사용
  # maxUnavailable: 1  # 최대 1개만 동시 중단 가능
```

**효과**:
- `kubectl drain` 실행 시 1개 Pod는 보존
- 클러스터 업그레이드 시 무중단 배포

### 3. Vertical Pod Autoscaler (VPA)

**HPA vs VPA**:
- HPA: Pod **개수** 조절 (수평 확장)
- VPA: Pod **크기** 조절 (수직 확장)

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: mongodb-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: mongodb-mongodb

  updatePolicy:
    updateMode: "Auto"  # 자동으로 Pod 재시작 & 리소스 조정

  resourcePolicy:
    containerPolicies:
    - containerName: mongodb
      minAllowed:
        cpu: 200m
        memory: 512Mi
      maxAllowed:
        cpu: 2000m
        memory: 4Gi
```

**주의**: VPA + HPA 동시 사용 시 충돌 가능! (동일 메트릭 사용 금지)

### 4. Node Affinity로 워크로드 분산

**시나리오**: Stateful(MongoDB)과 Stateless(API) 분리

```yaml
# MongoDB: 특정 노드에 고정
mongodb:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-role
            operator: In
            values:
            - database  # database 역할 노드에만 배포

# API Server: MongoDB와 같은 노드 피하기 (선호)
imprun-server:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: mongodb
          topologyKey: kubernetes.io/hostname
```

### 5. 리소스 쿼터 설정

**네임스페이스 전체 리소스 제한**:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: imprun-system-quota
  namespace: imprun-system
spec:
  hard:
    requests.cpu: "8"        # 총 8 cores 요청 가능
    requests.memory: "16Gi"  # 총 16GB 메모리 요청 가능
    limits.cpu: "20"
    limits.memory: "48Gi"
    persistentvolumeclaims: "10"  # PVC 최대 10개
    pods: "50"  # Pod 최대 50개
```

**효과**: 실수로 과도한 리소스 할당 방지

---

## 실전 시나리오

### 시나리오 1: OOMKilled 디버깅

**증상**:
```bash
kubectl get pods -n imprun-system
# NAME                        READY   STATUS      RESTARTS
# imprun-server-7b78c-6zzzd   0/1     OOMKilled   3
```

**원인 파악**:
```bash
# 1. 이벤트 확인
kubectl describe pod imprun-server-7b78c-6zzzd -n imprun-system | grep -A5 "Last State"

# 출력:
# Last State:     Terminated
#   Reason:       OOMKilled
#   Exit Code:    137

# 2. 메모리 사용량 히스토리 (VictoriaMetrics)
# container_memory_working_set_bytes{pod="imprun-server-7b78c-6zzzd"}
```

**해결**:
```yaml
# limits 증가
resources:
  limits:
    memory: 2Gi  # 1Gi → 2Gi

# Node.js 힙 크기 조정
env:
- name: NODE_OPTIONS
  value: "--max-old-space-size=1536"  # 1.5GB
```

### 시나리오 2: CPU Throttling 최소화

**증상**: API 응답 속도 느림, CPU throttled 높음

**확인**:
```bash
# Throttling 비율 확인 (Prometheus)
rate(container_cpu_cfs_throttled_seconds_total{pod=~"imprun-server.*"}[5m])
/ rate(container_cpu_cfs_periods_total{pod=~"imprun-server.*"}[5m])

# 30% 이상이면 문제!
```

**해결**:
```yaml
# CPU limits 증가 또는 제거
resources:
  limits:
    cpu: 2000m  # 1000m → 2000m

# 또는 limits 제거 (Burstable QoS 유지)
# limits:
#   cpu: null  # CPU throttling 없음 (주의: 노이지 네이버 가능)
```

### 시나리오 3: HPA가 작동하지 않음

**증상**:
```bash
kubectl get hpa -n imprun-system
# NAME        REFERENCE             TARGETS         MINPODS   MAXPODS   REPLICAS
# api-hpa     Deployment/api-server <unknown>/70%   2         6         2
```

**원인 & 해결**:

```bash
# 1. Metrics Server 확인
kubectl top nodes
# Error: Metrics API not available

# → Metrics Server 설치 필요

# 2. Pod에 resources.requests 없음
kubectl get pod api-server-xxx -o yaml | grep -A5 resources
# resources: {}  ← 문제!

# → requests 추가 필수

# 3. HPA 버전 불일치
kubectl api-versions | grep autoscaling
# autoscaling/v1  ← v2 필요!

# → Kubernetes 버전 업그레이드
```

---

## 모니터링 대시보드

### Grafana 대시보드 (VictoriaMetrics)

**CPU 사용률 패널**:
```promql
# 컨테이너별 CPU 사용률
sum(rate(container_cpu_usage_seconds_total{namespace="imprun-system"}[5m])) by (pod)
/
sum(container_spec_cpu_quota{namespace="imprun-system"}/container_spec_cpu_period{namespace="imprun-system"}) by (pod)
* 100
```

**메모리 사용률 패널**:
```promql
# 컨테이너별 메모리 사용률
container_memory_working_set_bytes{namespace="imprun-system"}
/
container_spec_memory_limit_bytes{namespace="imprun-system"}
* 100
```

**HPA 스케일링 이벤트**:
```promql
# Replica 변경 히스토리
kube_horizontalpodautoscaler_status_current_replicas{namespace="imprun-system"}
```

---

## 체크리스트

### 리소스 설정
- [ ] 모든 컨테이너에 requests/limits 설정
- [ ] requests는 실제 사용량의 110~120%
- [ ] limits는 requests의 2~5배 (버스트 허용)
- [ ] Stateful 워크로드는 Guaranteed QoS 고려

### ARM64 최적화
- [ ] 멀티 플랫폼 이미지 빌드 (amd64 + arm64)
- [ ] Node Selector로 아키텍처 지정
- [ ] ARM64 전용 바이너리 없는지 확인

### HPA 설정
- [ ] Metrics Server 설치 및 동작 확인
- [ ] CPU 기반 HPA 우선 적용
- [ ] minReplicas ≥ 2 (고가용성)
- [ ] scaleDown 안정화 시간 충분히 확보 (5분+)

### 모니터링
- [ ] VictoriaMetrics/Prometheus 연동
- [ ] Grafana 대시보드 구성
- [ ] OOMKilled/CPU Throttling 알림 설정
- [ ] 주간 리소스 사용량 리포트

---

## 결론

제한된 리소스 환경에서 안정적인 서비스 운영의 핵심:

1. **정확한 측정**: 추측이 아닌 메트릭 기반 의사결정
2. **점진적 조정**: 작게 시작해서 실제 사용량 기반 확장
3. **자동화**: HPA로 부하 대응, VPA로 리소스 최적화
4. **방어적 설계**: QoS, PDB로 장애 영향 최소화

ARM64 환경의 비용 효율성과 성능을 최대한 활용하여 imprun.dev는 월 $0의 인프라 비용으로 운영되고 있습니다.

## 다음 글 예고

- **CI/CD 파이프라인 구축**: GitHub Actions로 빌드부터 배포까지 완전 자동화

## 참고 자료

- [Kubernetes Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [HPA Walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
- [ARM64 Best Practices](https://www.docker.com/blog/multi-arch-build-and-images-the-simple-way/)
- [imprun.dev Architecture](../../CLAUDE.md)

---

**질문이나 피드백은 GitHub Issues로!**
📧 [GitHub Issues](https://github.com/imprun/imprun.dev/issues)
