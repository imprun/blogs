# Helm 차트 관리 Best Practices: Umbrella Chart부터 Secret 관리까지

> **작성일**: 2025-10-21
> **태그**: Helm, Kubernetes, DevOps, GitOps
> **난이도**: 중급~고급

## 들어가며

[**imprun.dev**](https://imprun.dev)는 Kubernetes 기반 서버리스 Cloud Function 플랫폼입니다. API 서버, Web Console, MongoDB, Redis, Runtime Exporter 등 **5개 이상의 컴포넌트를 하나의 클러스터에서 관리**하면서, Helm 차트 구조에 대한 고민이 깊어졌습니다.

**우리가 마주한 문제**:
- ❌ **개별 차트 관리의 복잡성**: 5번의 `helm install` 명령 필요
- ❌ **환경별 설정 중복**: dev/staging/production마다 반복되는 설정
- ❌ **Secret 관리 혼란**: 어떤 Secret을 어디서 정의해야 할지 모호
- ❌ **버전 불일치**: 컴포넌트 간 의존성 관리 어려움

**해결 과정**:
- ✅ **Umbrella Chart 패턴** 도입 → 단일 명령으로 전체 스택 배포
- ✅ **중앙 집중식 Secret 관리** → 일관성 확보
- ✅ **환경별 values 파일 분리** → 코드 재사용성 향상

이 글은 **imprun.dev 플랫폼 구축 경험**을 바탕으로, 실제 프로덕션 환경에서 검증된 Helm 차트 관리 전략을 공유합니다.

## 프로젝트 구조 소개

imprun.dev의 Helm 차트 구조:

```
k8s/imprun/
├── Chart.yaml                    # Umbrella Chart 정의
├── values.yaml                   # 공통 기본값
├── values-production.yaml        # 프로덕션 오버라이드
├── values-staging.yaml           # 스테이징 오버라이드
├── values-local.yaml             # 로컬 개발용
├── templates/
│   ├── secrets.yaml              # 중앙 집중식 Secret 관리
│   └── certificates.yaml         # TLS 인증서
└── charts/                       # 서브차트
    ├── imprun-server/
    ├── imprun-console/
    ├── imp-runtime-exporter/
    ├── mongodb/
    └── redis/
```

---

## Part 1: Umbrella Chart 패턴

### Umbrella Chart란?

**여러 독립적인 차트를 하나의 상위 차트로 묶어 관리하는 패턴**입니다.

### 왜 Umbrella Chart를 사용하는가?

**Before (개별 차트)**:
```bash
# 😰 5번의 배포 필요
helm install mongodb ./mongodb -n imprun-system
helm install redis ./redis -n imprun-system
helm install api ./imprun-server -n imprun-system
helm install console ./imprun-console -n imprun-system
helm install exporter ./imp-runtime-exporter -n imprun-system

# 업데이트도 5번...
# Secret 공유도 복잡...
# 의존성 관리도 어려움...
```

**After (Umbrella Chart)**:
```bash
# 🎉 한 번에 전체 스택 배포!
helm install imprun ./k8s/imprun -n imprun-system

# 업데이트도 한 번!
helm upgrade imprun ./k8s/imprun -n imprun-system
```

### Chart.yaml 구성

[k8s/imprun/Chart.yaml](../../k8s/imprun/Chart.yaml):

```yaml
apiVersion: v2
name: imprun
description: imprun.dev Serverless Platform - Umbrella Chart
type: application
version: 0.1.0
appVersion: "1.0.0"

# 서브차트 의존성 정의
dependencies:
  # 1. 인프라 계층 (먼저 시작)
  - name: mongodb
    version: "0.1.0"
    repository: "file://./charts/mongodb"
    condition: mongodb.enabled

  - name: redis
    version: "0.1.0"
    repository: "file://./charts/redis"
    condition: redis.enabled

  # 2. API 게이트웨이 계층
  - name: imprun-server
    version: "0.1.0"
    repository: "file://./charts/imprun-server"
    condition: imprun-server.enabled

  - name: imprun-console
    version: "0.1.0"
    repository: "file://./charts/imprun-console"
    condition: imprun-console.enabled

  # 3. 모니터링/유틸리티 계층
  - name: imp-runtime-exporter
    version: "0.1.0"
    repository: "file://./charts/imp-runtime-exporter"
    condition: imp-runtime-exporter.enabled
```

**핵심 포인트**:
- `condition`: 서브차트 활성화/비활성화 제어
- `repository: file://`: 로컬 차트 참조 (단일 저장소 관리)
- 계층별 구분: 인프라 → API 게이트웨이 → 유틸리티

### 의존성 업데이트

```bash
# 서브차트 변경 후 반드시 실행!
cd k8s/imprun
helm dependency update

# 생성되는 파일들:
# - Chart.lock: 의존성 잠금 파일
# - charts/*.tgz: 패키징된 서브차트
```

### 선택적 배포

**로컬 개발: 인프라만**
```yaml
# values-local.yaml
mongodb:
  enabled: true
redis:
  enabled: true

imprun-server:
  enabled: false  # 로컬에서 직접 실행
imprun-console:
  enabled: false  # 로컬에서 직접 실행
imp-runtime-exporter:
  enabled: false
```

```bash
helm install imprun . -n imprun-local --values values-local.yaml
# → MongoDB, Redis만 k8s에 배포
```

---

## Part 2: Values 파일 전략

### 계층적 Values 구조

```
values.yaml (기본값)
    ↓ 오버라이드
values-dev.yaml (개발 환경)
values-staging.yaml (스테이징)
values-production.yaml (프로덕션)
```

### 1. values.yaml: 공통 기본값

**원칙**: 환경에 관계없이 공통된 설정만 포함

```yaml
# values.yaml
global:
  imageRegistry: docker.io
  imagePullSecrets: []

imprun-server:
  enabled: true
  replicaCount: 1  # 기본값

  image:
    repository: junsik/imprun-server
    tag: latest
    pullPolicy: IfNotPresent  # 기본값

  resources:
    requests:
      cpu: 200m
      memory: 512Mi
    limits:
      cpu: 1000m
      memory: 2Gi

  env:
    JWT_EXPIRES_IN: "7d"  # 공통 설정
```

### 2. values-local.yaml: 로컬 개발

**특징**:
- 리소스 제한 완화
- 외부 서비스 비활성화
- NodePort 사용

```yaml
# values-local.yaml
global:
  domain: localhost

# 로컬 실행하므로 비활성화
imprun-server:
  enabled: false
imprun-console:
  enabled: false

# 인프라만 활성화
mongodb:
  enabled: true
  replicas: 1  # 단일 인스턴스
  resources:
    requests:
      cpu: 100m      # 낮은 리소스
      memory: 256Mi
  auth:
    rootPassword: "local_dev_password"  # 약한 비밀번호 OK

redis:
  enabled: true
  auth:
    enabled: false  # 로컬에서는 인증 불필요

# NodePort로 접근
services:
  type: NodePort
  ports:
    api: 30000
    console: 30001
```

**사용**:
```bash
helm install imprun . -n imprun-local -f values-local.yaml
kubectl port-forward svc/mongodb 27017:27017
kubectl port-forward svc/redis 6379:6379
```

### 3. values-staging.yaml: 스테이징

**특징**:
- 프로덕션 유사 환경
- 낮은 리소스 할당
- 테스트용 도메인

```yaml
# values-staging.yaml
global:
  domain: staging.imprun.dev

imprun-server:
  replicaCount: 1  # 프로덕션보다 적음

  image:
    tag: staging  # 명시적 태그
    pullPolicy: Always  # 최신 staging 이미지

  env:
    SERVICE_DOMAIN: "api.staging.imprun.dev"
    CONSOLE_URL: "https://console.staging.imprun.dev"

  resources:
    requests:
      cpu: 100m      # 절반 리소스
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 1Gi

mongodb:
  replicas: 1  # Single instance
  storage:
    size: 10Gi  # 작은 스토리지

certificates:
  enabled: true
  issuer: letsencrypt-staging  # 스테이징 인증서
```

### 4. values-production.yaml: 프로덕션

**특징**:
- 고가용성 (HA)
- 강력한 보안
- 프로덕션 인증서

```yaml
# values-production.yaml
global:
  domain: imprun.dev

imprun-server:
  replicaCount: 3  # HA 구성

  image:
    tag: v1.2.3  # 명시적 버전 (시맨틱 버저닝)
    pullPolicy: IfNotPresent  # 성능 최적화

  env:
    SERVICE_DOMAIN: "api.imprun.dev"
    CONSOLE_URL: "https://portal.imprun.dev"

  resources:
    requests:
      cpu: 500m      # 넉넉한 리소스
      memory: 1Gi
    limits:
      cpu: 2000m
      memory: 4Gi

  # HPA (Horizontal Pod Autoscaler)
  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 10
    targetCPUUtilizationPercentage: 70

mongodb:
  replicas: 3  # Replica Set
  storage:
    size: 100Gi
  auth:
    rootPassword: ""  # Secret으로 주입 (아래 참조)

redis:
  auth:
    enabled: true
    password: ""  # Secret으로 주입

certificates:
  enabled: true
  issuer: letsencrypt-prod  # 프로덕션 인증서

# Secret은 별도 관리 (아래 Part 3 참조)
```

**사용**:
```bash
helm install imprun . -n imprun-system \
  -f values.yaml \
  -f values-production.yaml \
  --set mongodb.auth.rootPassword=$MONGO_PASSWORD \
  --set redis.auth.password=$REDIS_PASSWORD
```

### Values 오버라이드 우선순위

```
기본값 (values.yaml)
    ↓ 오버라이드
환경별 Values (-f values-production.yaml)
    ↓ 오버라이드
커맨드 라인 (--set key=value)
```

**예시**:
```bash
# 최종값: pullPolicy=Always (--set이 최우선)
helm upgrade imprun . \
  -f values.yaml \                    # pullPolicy: IfNotPresent
  -f values-production.yaml \         # pullPolicy: IfNotPresent
  --set imprun-server.image.pullPolicy=Always  # 최종 승리!
```

---

## Part 3: Secret 관리

### Secret 관리의 어려움

**문제점**:
- 🔴 Git에 평문 저장 금지
- 🔴 환경마다 다른 비밀번호
- 🔴 여러 서비스 간 Secret 공유
- 🔴 비밀번호 로테이션

### 전략 1: Helm Values + 외부 주입 (기본)

**values.yaml에서 비밀번호 placeholder 정의**:

```yaml
# values.yaml
secrets:
  jwt:
    secret: ""  # 비어있음 → 외부에서 주입

  mongodb:
    rootPassword: ""

  github:
    clientId: ""
    clientSecret: ""

  google:
    clientId: ""
    clientSecret: ""
```

**배포 시 주입**:
```bash
# 환경 변수에서 읽기
export JWT_SECRET=$(openssl rand -base64 32)
export MONGO_PASSWORD=$(openssl rand -base64 32)
export GITHUB_CLIENT_SECRET="your_github_secret"

helm install imprun . -n imprun-system \
  -f values-production.yaml \
  --set secrets.jwt.secret=$JWT_SECRET \
  --set secrets.mongodb.rootPassword=$MONGO_PASSWORD \
  --set secrets.github.clientSecret=$GITHUB_CLIENT_SECRET
```

**템플릿에서 Secret 생성** ([k8s/imprun/templates/secrets.yaml](../../k8s/imprun/templates/secrets.yaml)):

```yaml
# templates/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: imprun-jwt-secret
  namespace: {{ .Release.Namespace }}
type: Opaque
data:
  JWT_SECRET: {{ .Values.secrets.jwt.secret | b64enc | quote }}

---
apiVersion: v1
kind: Secret
metadata:
  name: imprun-github-oauth
  namespace: {{ .Release.Namespace }}
type: Opaque
data:
  CLIENT_ID: {{ .Values.secrets.github.clientId | b64enc | quote }}
  CLIENT_SECRET: {{ .Values.secrets.github.clientSecret | b64enc | quote }}
```

**서브차트에서 참조**:

```yaml
# charts/imprun-server/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: api-server
        env:
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: imprun-jwt-secret  # Umbrella Chart가 생성한 Secret
              key: JWT_SECRET

        - name: GITHUB_CLIENT_SECRET
          valueFrom:
            secretKeyRef:
              name: imprun-github-oauth
              key: CLIENT_SECRET
```

### 전략 2: Sealed Secrets (GitOps 친화적)

**설치**:
```bash
# Sealed Secrets Controller 설치
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# kubeseal CLI 설치 (macOS)
brew install kubeseal
```

**워크플로우**:

1. **일반 Secret 생성 (로컬)**:
```bash
kubectl create secret generic imprun-jwt-secret \
  --from-literal=JWT_SECRET=$(openssl rand -base64 32) \
  --dry-run=client -o yaml > jwt-secret.yaml
```

2. **Sealed Secret으로 암호화**:
```bash
kubeseal -f jwt-secret.yaml -w jwt-sealed-secret.yaml

# jwt-sealed-secret.yaml → Git에 커밋 가능! (암호화됨)
rm jwt-secret.yaml  # 평문 삭제
```

3. **Git에 커밋**:
```yaml
# k8s/imprun/templates/jwt-sealed-secret.yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: imprun-jwt-secret
  namespace: imprun-system
spec:
  encryptedData:
    JWT_SECRET: AgBq7Kx3... (암호화된 값)
```

4. **Sealed Secrets Controller가 자동 복호화**:
```bash
kubectl apply -f k8s/imprun/templates/jwt-sealed-secret.yaml
# → Controller가 자동으로 Secret 생성
```

**장점**:
- ✅ Git에 안전하게 저장
- ✅ GitOps 워크플로우 호환
- ✅ PR 리뷰 가능

**단점**:
- ❌ 클러스터마다 다른 암호화 키
- ❌ Secret 로테이션 복잡

### 전략 3: External Secrets Operator (권장 - 엔터프라이즈)

**AWS Secrets Manager, Vault, GCP Secret Manager 연동**

```yaml
# external-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: imprun-jwt-secret
spec:
  refreshInterval: 1h  # 1시간마다 동기화
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore

  target:
    name: imprun-jwt-secret
    creationPolicy: Owner

  data:
  - secretKey: JWT_SECRET
    remoteRef:
      key: imprun/production/jwt-secret  # AWS Secrets Manager 경로
```

**장점**:
- ✅ 중앙 집중식 Secret 관리
- ✅ 자동 로테이션
- ✅ 감사 로그
- ✅ 세밀한 접근 제어

**단점**:
- ❌ 외부 서비스 의존성
- ❌ 추가 비용

### Secret 관리 Best Practices

#### 1. 환경별 분리

```
aws-secrets-manager/
├── imprun/
│   ├── dev/
│   │   ├── jwt-secret
│   │   ├── mongodb-password
│   │   └── github-oauth
│   ├── staging/
│   │   └── ...
│   └── production/
│       └── ...
```

#### 2. Secret 로테이션 전략

```bash
# 1. 새 Secret 생성 (AWS Secrets Manager)
aws secretsmanager update-secret \
  --secret-id imprun/production/jwt-secret \
  --secret-string $(openssl rand -base64 32)

# 2. External Secrets Operator가 자동 동기화 (1시간 이내)

# 3. Pod 재시작 (새 Secret 적용)
kubectl rollout restart deployment/imprun-imprun-server -n imprun-system
```

#### 3. Secret 최소 권한 원칙

```yaml
# imprun-server만 JWT Secret 접근 가능
apiVersion: v1
kind: ServiceAccount
metadata:
  name: imprun-server

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["imprun-jwt-secret"]  # 특정 Secret만
  verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: imprun-server-secret-reader
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: secret-reader
subjects:
- kind: ServiceAccount
  name: imprun-server
```

---

## Part 4: 고급 패턴

### 1. 동적 Secret 생성

**문제**: 배포 시마다 랜덤 비밀번호 생성

```yaml
# templates/_helpers.tpl
{{- define "imprun.jwt.secret" -}}
{{- if .Values.secrets.jwt.secret -}}
  {{- .Values.secrets.jwt.secret -}}
{{- else -}}
  {{- randAlphaNum 32 | b64enc -}}
{{- end -}}
{{- end -}}

# templates/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: imprun-jwt-secret
  annotations:
    "helm.sh/resource-policy": keep  # helm delete 시에도 유지
data:
  JWT_SECRET: {{ include "imprun.jwt.secret" . | quote }}
```

**주의**: `helm upgrade` 시마다 새 값 생성 방지 위해 `helm.sh/resource-policy: keep` 필수!

### 2. 서브차트 간 값 공유

**Umbrella Chart에서 공통 값 정의**:

```yaml
# values.yaml
global:
  domain: imprun.dev
  mongodb:
    host: mongodb-mongodb
    port: 27017
    database: sys_db
  redis:
    host: redis-master
    port: 6379
```

**서브차트에서 참조**:

```yaml
# charts/imprun-server/templates/deployment.yaml
env:
- name: MONGODB_HOST
  value: {{ .Values.global.mongodb.host }}
- name: REDIS_HOST
  value: {{ .Values.global.redis.host }}
```

### 3. 조건부 리소스 생성

```yaml
# templates/certificates.yaml
{{- if .Values.certificates.enabled }}
{{- range .Values.certificates.certs }}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: {{ .name }}
spec:
  secretName: {{ .secretName }}
  dnsNames:
  {{- range .dnsNames }}
  - {{ . }}
  {{- end }}
  issuerRef:
    name: {{ $.Values.certificates.issuer }}
    kind: ClusterIssuer
{{- end }}
{{- end }}
```

**사용**:
```yaml
# values-production.yaml
certificates:
  enabled: true
  issuer: letsencrypt-prod
  certs:
    - name: app-imprun-dev
      secretName: app-imprun-dev-tls
      dnsNames:
        - api.imprun.dev
    - name: wildcard-app-imprun-dev
      secretName: wildcard-app-imprun-dev-tls
      dnsNames:
        - "*.api.imprun.dev"
```

---

## 실전 시나리오

### 시나리오 1: 새 환경 추가 (QA 환경)

```bash
# 1. values-qa.yaml 생성
cp values-staging.yaml values-qa.yaml

# 2. QA 전용 설정 수정
vi values-qa.yaml
# domain: qa.imprun.dev
# replicaCount: 1
# resources: (staging과 동일)

# 3. 배포
helm install imprun-qa . -n imprun-qa \
  --create-namespace \
  -f values-qa.yaml \
  --set secrets.jwt.secret=$QA_JWT_SECRET

# 4. DNS 설정
# api.qa.imprun.dev → LoadBalancer IP
```

### 시나리오 2: Staging → Production 승격

```bash
# 1. Staging 이미지 태그 확인
kubectl get deployment imprun-imprun-server -n imprun-staging \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
# 출력: junsik/imprun-server:staging-abc123

# 2. Production 태그로 재태깅
docker pull junsik/imprun-server:staging-abc123
docker tag junsik/imprun-server:staging-abc123 junsik/imprun-server:v1.3.0
docker push junsik/imprun-server:v1.3.0

# 3. Production 배포
helm upgrade imprun . -n imprun-system \
  -f values-production.yaml \
  --set imprun-server.image.tag=v1.3.0
```

### 시나리오 3: Secret 긴급 로테이션

```bash
# 1. 새 Secret 생성
NEW_SECRET=$(openssl rand -base64 32)

# 2. Secret 업데이트
kubectl create secret generic imprun-jwt-secret \
  --from-literal=JWT_SECRET=$NEW_SECRET \
  --dry-run=client -o yaml | kubectl apply -f -

# 3. 모든 Pod 재시작 (새 Secret 로드)
kubectl rollout restart deployment -n imprun-system

# 4. 롤아웃 모니터링
kubectl rollout status deployment/imprun-imprun-server -n imprun-system
```

---

## 체크리스트

### Helm 차트 설계
- [ ] Umbrella Chart로 관련 서비스 그룹화
- [ ] `condition`으로 서브차트 선택적 활성화
- [ ] `global` 값으로 공통 설정 공유
- [ ] 서브차트 간 의존성 명확히 정의

### Values 파일 관리
- [ ] 환경별 Values 파일 분리 (local/staging/prod)
- [ ] 기본값은 `values.yaml`에만
- [ ] 민감 정보는 절대 Git에 커밋 금지
- [ ] `--set` 오버라이드 최소화 (복잡도 증가)

### Secret 관리
- [ ] 프로덕션은 외부 Secret 관리자 사용 (Sealed Secrets / External Secrets)
- [ ] Secret 로테이션 계획 수립
- [ ] ServiceAccount + RBAC로 최소 권한 부여
- [ ] `helm.sh/resource-policy: keep`로 중요 Secret 보호

### 배포 자동화
- [ ] CI/CD에서 환경별 Values 파일 자동 선택
- [ ] Helm diff로 변경 사항 사전 검토
- [ ] 롤백 계획 수립 (`helm rollback`)

---

## 결론

Helm 차트 관리는 단순한 패키징 이상의 의미를 가집니다:

1. **Umbrella Chart**: 복잡한 마이크로서비스를 단일 배포 단위로 통합
2. **Values 전략**: 환경별 차이를 명확히 분리하여 실수 방지
3. **Secret 관리**: 보안을 최우선으로, GitOps 워크플로우와 조화

imprun.dev의 실전 경험을 바탕으로 작성된 이 가이드가 여러분의 Kubernetes 여정에 도움이 되길 바랍니다.

## 다음 글 예고

- **Kubernetes 리소스 최적화**: ARM64 환경 제약사항과 HPA 설정
- **CI/CD 파이프라인 구축**: GitHub Actions로 완전 자동화하기

## 참고 자료

- [Helm 공식 문서 - Chart Best Practices](https://helm.sh/docs/chart_best_practices/)
- [Sealed Secrets GitHub](https://github.com/bitnami-labs/sealed-secrets)
- [External Secrets Operator](https://external-secrets.io/)
- [imprun.dev Helm Charts](../../k8s/imprun/)

---

**피드백은 언제나 환영합니다!**
📧 [GitHub Issues](https://github.com/imprun/imprun.dev/issues)
