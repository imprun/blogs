# Kubernetes 민감정보 관리 완벽 가이드: Secret, 암호화, 그리고 실전 전략

> **작성일**: 2025-10-22
> **태그**: Kubernetes, Security, Secret Management, DevOps, GitOps
> **난이도**: 중급~고급

## 들어가며

[**imprun.dev**](https://imprun.dev)는 Kubernetes 기반 서버리스 Cloud Function 플랫폼입니다. 초기에는 개발 편의성을 위해 **민감정보를 Helm values 파일에 평문으로 저장**했습니다. 하지만 곧 심각한 보안 위험을 인지하게 되었습니다.

**우리가 마주한 보안 위험**:
- ❌ **Git 저장소 노출**: values 파일에 평문 비밀번호 포함
- ❌ **관리 분산**: Secret이 여러 차트에 중복 정의됨
- ❌ **암호화 부재**: etcd에 평문으로 저장
- ❌ **접근 제어 미흡**: 누구나 Secret 조회 가능

**관리해야 하는 민감정보**:
- JWT Secret, OAuth 클라이언트 시크릿
- MongoDB 비밀번호, Redis 비밀번호
- Docker Registry 인증 정보
- TLS 인증서 private key

이 글은 **imprun.dev 플랫폼 구축 경험**을 바탕으로, Kubernetes 환경에서 **프로덕션 수준의 민감정보 관리 전략**을 단계별로 공유합니다.

## 우리가 관리해야 하는 민감정보

imprun.dev에서 관리하는 민감정보 목록:

```yaml
민감정보 카테고리:
  인증/보안:
    - JWT Secret (64자 랜덤 문자열)
    - GitHub OAuth (Client ID + Secret)
    - Google OAuth (Client ID + Secret)
    - Admin 계정 초기 비밀번호

  데이터베이스:
    - MongoDB Root 비밀번호
    - MongoDB 연결 URI
    - Redis 비밀번호

  인프라:
    - TLS/SSL 인증서 (cert-manager)
    - Cloudflare API 토큰 (DNS Challenge)
    - Container Registry 인증정보

  런타임:
    - 사용자 앱 전용 DB 접근 토큰
    - Runtime Exporter 인증 토큰
```

**문제**: 이 모든 정보를 안전하게 저장하면서도, 운영 편의성을 유지해야 합니다.

---

## Part 1: Kubernetes Secret 기초와 한계

### Kubernetes Secret이란?

**민감정보를 Base64로 인코딩하여 저장하는 Kubernetes 리소스**입니다.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-auth
  namespace: imprun-system
type: Opaque
data:
  username: cm9vdA==           # base64("root")
  password: bXlwYXNzd29yZA==   # base64("mypassword")
```

### Secret의 장점

✅ **환경 변수 주입 간편**:
```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: mongodb-auth
        key: password
```

✅ **RBAC 연동**: 네임스페이스/역할별 접근 제어
✅ **자동 마운트**: Pod에 파일로 자동 마운트 가능

### Secret의 한계와 위험

❌ **Base64는 암호화가 아님** (디코딩만으로 원본 노출)
❌ **etcd에 평문 저장** (etcd 암호화 미활성화 시)
❌ **Git에 커밋 위험** (실수로 values-production.yaml 커밋 시 유출)
❌ **Secret Sprawl**: Secret이 여러 곳에 분산되면 관리 복잡도 폭발

---

## Part 2: imprun.dev의 Secret 관리 전략

### 전략 1: 중앙 집중식 Secret 생성

**문제**: 5개의 서브차트가 각각 Secret을 생성하면 관리 불가능

**해결**: Umbrella Chart의 `templates/secrets.yaml`에서 모든 Secret을 생성

[k8s/imprun/templates/secrets.yaml](../../k8s/imprun/templates/secrets.yaml):

```yaml
{{- if .Values.secrets.jwt.secret }}
---
apiVersion: v1
kind: Secret
metadata:
  name: imprun-jwt-secret
  namespace: {{ .Release.Namespace }}
  annotations:
    helm.sh/resource-policy: keep  # ⭐ 핵심: Helm 업그레이드 시 유지
type: Opaque
data:
  JWT_SECRET: {{ .Values.secrets.jwt.secret | b64enc }}
{{- end }}

{{- if .Values.secrets.github.clientSecret }}
---
apiVersion: v1
kind: Secret
metadata:
  name: imprun-github-oauth
  namespace: {{ .Release.Namespace }}
  annotations:
    helm.sh/resource-policy: keep
type: Opaque
data:
  GITHUB_CLIENT_ID: {{ .Values.secrets.github.clientId | b64enc }}
  GITHUB_CLIENT_SECRET: {{ .Values.secrets.github.clientSecret | b64enc }}
{{- end }}
```

**핵심 포인트**:
1. **`helm.sh/resource-policy: keep`**: 업그레이드 시 Secret 재생성 방지
2. **조건부 생성**: 값이 제공된 경우만 생성 (`{{- if }}`)
3. **단일 진실 공급원**: 모든 Secret이 한 파일에 정의됨

### 전략 2: values-production.yaml 분리

**절대 Git에 커밋하지 않는 파일**: `values-production.yaml`

```yaml
# k8s/imprun/values-production.yaml (⚠️ .gitignore 필수)

secrets:
  jwt:
    secret: "abcd1234...64자 랜덤 문자열"  # openssl rand -base64 48

  github:
    clientId: "Ov23liOCvWeKTz6TO6Pn"
    clientSecret: "ghp_xxxxxxxxxxxxxxxxxxxx"

  google:
    clientId: "123456789-xxxxxxx.apps.googleusercontent.com"
    clientSecret: "GOCSPX-xxxxxxxxxxxxxxxx"

  adminSeed:
    username: "admin@imprun.dev"
    password: "초기_관리자_비밀번호"

mongodb:
  auth:
    rootPassword: "몽고_루트_비밀번호"

redis:
  auth:
    password: "레디스_비밀번호"
```

**보안 원칙**:
```bash
# .gitignore
values-production.yaml
values-*.secret.yaml
*.pem
*.key
```

### 전략 3: 자동 생성 vs 명시적 제공

**선택지 제공**:

| 방법 | 장점 | 단점 | 사용 시기 |
|------|------|------|----------|
| **자동 생성** | 편리함, 충돌 없음 | 값 추적 어려움 | 개발 환경 |
| **명시적 제공** | 예측 가능, 백업 용이 | 초기 설정 번거로움 | 프로덕션 |

**Helm Template 예시** (자동 생성):

```yaml
{{- if not .Values.secrets.jwt.secret }}
---
apiVersion: v1
kind: Secret
metadata:
  name: imprun-jwt-secret
  annotations:
    helm.sh/resource-policy: keep
data:
  JWT_SECRET: {{ randAlphaNum 64 | b64enc }}  # 64자 랜덤 생성
{{- end }}
```

**주의**: 자동 생성 시 `helm.sh/resource-policy: keep` 필수 (재설치 시 값 변경 방지)

---

## Part 3: KubeBlocks와 자동 생성 Secret

### 문제: 외부 Operator가 생성하는 Secret

MongoDB를 KubeBlocks로 관리하면, **KubeBlocks Operator가 Secret을 자동 생성**합니다.

```yaml
# k8s/imprun/templates/mongodb-cluster.yaml
apiVersion: apps.kubeblocks.io/v1alpha1
kind: Cluster
metadata:
  name: mongodb
spec:
  clusterDefinitionRef: mongodb
  terminationPolicy: DoNotTerminate
  componentSpecs:
    - name: mongodb
      componentDefRef: mongodb
```

KubeBlocks가 생성하는 Secret:
```bash
$ kubectl get secret mongodb-conn-credential -n imprun-system

NAME                       TYPE     DATA   AGE
mongodb-conn-credential    Opaque   4      2m
```

**포함 내용**:
```yaml
username: root
password: values에서 지정한 rootPassword
endpoint: mongodb-mongodb:27017
port: "27017"
```

### 해결: Secret 이름 표준화

**문제**: 우리가 `mongodb-auth`를 생성했지만, KubeBlocks는 `mongodb-conn-credential` 생성

**해결**: KubeBlocks의 표준 이름을 사용하도록 모든 참조 변경

```yaml
# imprun-server Deployment
env:
  - name: MONGODB_ROOT_USER
    valueFrom:
      secretKeyRef:
        name: mongodb-conn-credential  # ✅ KubeBlocks 표준 이름
        key: username
  - name: MONGODB_ROOT_PASSWORD
    valueFrom:
      secretKeyRef:
        name: mongodb-conn-credential
        key: password
```

**교훈**: 외부 Operator 사용 시 해당 Operator의 Secret 네이밍 규칙 따르기

---

## Part 4: Secret 라이프사이클 관리

### helm install: 초기 생성

```bash
# 1차: Secret이 존재하지 않으므로 생성
helm install imprun ./k8s/imprun -n imprun-system \
  -f values-production.yaml
```

### helm upgrade: 기존 유지

```yaml
annotations:
  helm.sh/resource-policy: keep  # ⭐ 이 annotation 덕분에 유지됨
```

```bash
# 2차: Secret은 그대로 유지, Deployment만 업데이트
helm upgrade imprun ./k8s/imprun -n imprun-system \
  -f values-production.yaml
```

**장점**:
- 비밀번호가 변경되지 않음 (서비스 안정성)
- 롤링 업데이트 중에도 인증 문제 없음

### helm uninstall: 삭제하지 않음

```bash
helm uninstall imprun -n imprun-system

# ✅ Secret은 여전히 존재
kubectl get secrets -n imprun-system
# NAME                      TYPE     DATA   AGE
# imprun-jwt-secret         Opaque   1      10d
# imprun-github-oauth       Opaque   2      10d
```

**재설치 시**:
```bash
# Secret이 이미 존재하므로 재생성하지 않음 ({{- if }} 조건)
helm install imprun ./k8s/imprun -n imprun-system
```

### Secret 완전 삭제 (신중히!)

```bash
# ⚠️ 프로덕션에서는 절대 실행 금지
kubectl delete secret imprun-jwt-secret -n imprun-system

# 재설치 시 새 값으로 생성됨
helm upgrade imprun ./k8s/imprun -n imprun-system
```

---

## Part 5: 민감정보 노출 방지 전략

### 1. .gitignore 철저히 관리

```bash
# k8s/imprun/.gitignore
values-production.yaml
values-*.secret.yaml
*.pem
*.key
*.crt
secrets/

# k8s/victoriametrics/.gitignore
values-production.yaml
```

**검증**:
```bash
# Git에 커밋되지 않았는지 확인
git status --ignored | grep values-production
```

### 2. Pre-commit Hook으로 자동 검증

```bash
# .git/hooks/pre-commit
#!/bin/bash

# 민감정보 패턴 검색
if git diff --cached | grep -E "(password|secret|token|apiKey).*[:=].*['\"]" ; then
  echo "❌ 민감정보가 커밋에 포함되어 있습니다!"
  exit 1
fi

# values-production.yaml 커밋 차단
if git diff --cached --name-only | grep -E "values-production\.yaml" ; then
  echo "❌ values-production.yaml은 커밋할 수 없습니다!"
  exit 1
fi
```

### 3. Secret 백업 전략

**안전한 백업**:
```bash
# 1. Secret을 YAML로 추출
kubectl get secrets -n imprun-system -o yaml > secrets-backup.yaml

# 2. GPG로 암호화
gpg --encrypt --recipient admin@imprun.dev secrets-backup.yaml

# 3. 암호화된 파일만 백업 스토리지에 저장
aws s3 cp secrets-backup.yaml.gpg s3://imprun-backups/secrets/

# 4. 원본 삭제
rm secrets-backup.yaml
```

**복구**:
```bash
# 1. 다운로드 및 복호화
aws s3 cp s3://imprun-backups/secrets/secrets-backup.yaml.gpg .
gpg --decrypt secrets-backup.yaml.gpg > secrets-backup.yaml

# 2. 복원
kubectl apply -f secrets-backup.yaml

# 3. 복호화된 파일 삭제
rm secrets-backup.yaml*
```

### 4. RBAC으로 접근 제한

```yaml
# k8s/imprun/templates/rbac.yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: imprun-system
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames:
      - imprun-jwt-secret
      - imprun-github-oauth
    verbs: ["get", "list"]  # create, update, delete 제외

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: imprun-server-secret-access
  namespace: imprun-system
subjects:
  - kind: ServiceAccount
    name: imprun-server
    namespace: imprun-system
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

**원칙**:
- Pod는 **필요한 Secret만** 접근 가능
- `get`, `list`만 허용 (`update`, `delete` 금지)
- ServiceAccount별로 세밀한 권한 분리

---

## Part 6: 고급 전략 - External Secret Operator

### 현재의 한계

현재 방식 (values-production.yaml):
- ❌ 여전히 평문 파일로 존재
- ❌ 개발자 로컬에 민감정보 저장
- ❌ Secret Rotation 수동 작업

### 해결: External Secrets Operator

**AWS Secrets Manager / HashiCorp Vault 연동**:

```yaml
# k8s/imprun/templates/external-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: imprun-secrets
  namespace: imprun-system
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: SecretStore
  target:
    name: imprun-jwt-secret
    creationPolicy: Owner
  data:
    - secretKey: JWT_SECRET
      remoteRef:
        key: imprun/production/jwt-secret
```

**장점**:
- ✅ 중앙 집중식 Secret 관리 (AWS Secrets Manager)
- ✅ 자동 Rotation 지원
- ✅ 감사 로그 (누가 언제 접근했는지 추적)
- ✅ 다중 클러스터 Secret 동기화

**단점**:
- 추가 복잡도
- 외부 서비스 의존성
- 비용 발생 (AWS Secrets Manager: $0.40/secret/month)

---

## Part 7: 실전 체크리스트

### 배포 전 검증

```bash
# ✅ 1. .gitignore 확인
git check-ignore -v k8s/imprun/values-production.yaml
# .gitignore:10:values-production.yaml    k8s/imprun/values-production.yaml

# ✅ 2. Secret annotation 확인
kubectl get secret imprun-jwt-secret -n imprun-system -o yaml | grep "resource-policy"
# helm.sh/resource-policy: keep

# ✅ 3. Secret 값 검증 (Base64 디코딩)
kubectl get secret imprun-jwt-secret -n imprun-system \
  -o jsonpath='{.data.JWT_SECRET}' | base64 -d
# (64자 랜덤 문자열 출력 확인)

# ✅ 4. RBAC 권한 확인
kubectl auth can-i get secrets --as=system:serviceaccount:imprun-system:imprun-server -n imprun-system
# yes

kubectl auth can-i delete secrets --as=system:serviceaccount:imprun-system:imprun-server -n imprun-system
# no
```

### 보안 감사

```bash
# 1. 모든 Secret 목록
kubectl get secrets --all-namespaces

# 2. 특정 Secret 사용처 추적
kubectl get pods -n imprun-system -o json | \
  jq -r '.items[] | select(.spec.containers[].env[]?.valueFrom.secretKeyRef.name == "imprun-jwt-secret") | .metadata.name'

# 3. Secret 생성 일자 확인 (오래된 Secret은 Rotation 고려)
kubectl get secrets -n imprun-system -o json | \
  jq -r '.items[] | [.metadata.name, .metadata.creationTimestamp] | @tsv'
```

---

## Part 8: MongoDB Secret 특수 사례 - terminationPolicy

### 문제: MongoDB 데이터 보존

```yaml
# KubeBlocks Cluster
spec:
  terminationPolicy: DoNotTerminate  # ⭐ 핵심 설정
```

**시나리오**:
```bash
# 1. Helm Chart 삭제
helm uninstall imprun -n imprun-system

# 2. MongoDB Cluster는?
kubectl get cluster -n imprun-system
# NAME      STATUS    AGE
# mongodb   Running   10d  ← 여전히 존재!

# 3. Secret도 유지됨
kubectl get secret mongodb-conn-credential -n imprun-system
# NAME                       TYPE     DATA   AGE
# mongodb-conn-credential    Opaque   4      10d
```

**장점**:
- ✅ 실수로 helm uninstall 해도 데이터 보존
- ✅ Secret도 함께 유지되어 재설치 시 즉시 연결 가능

**완전 삭제 방법**:
```bash
# 방법 1: Cluster만 삭제 (권장)
kubectl delete cluster mongodb -n imprun-system

# 방법 2: 네임스페이스 전체 삭제
kubectl delete namespace imprun-system
```

---

## 결론: 민감정보 관리의 핵심 원칙

### 1. **분리 (Separation)**
- 코드와 민감정보 분리 (values.yaml vs values-production.yaml)
- 환경별 Secret 분리 (dev/staging/production)

### 2. **최소 권한 (Least Privilege)**
- RBAC으로 Pod별 필요한 Secret만 접근
- `get`, `list`만 허용 (`update`, `delete` 금지)

### 3. **암호화 (Encryption)**
- etcd 암호화 활성화 (--encryption-provider-config)
- 백업 시 GPG 암호화
- 전송 중 TLS 필수

### 4. **감사 (Auditing)**
- Secret 접근 로그 기록
- 정기적인 Secret Rotation
- 사용하지 않는 Secret 정리

### 5. **자동화 (Automation)**
- Helm으로 Secret 생성 자동화
- `helm.sh/resource-policy: keep`로 안전한 업그레이드
- External Secrets Operator로 고급 관리

---

## 다음 단계

### 단계별 개선 로드맵

**Level 1 (현재)**: Helm + values-production.yaml
- ✅ 구현 간단
- ✅ 대부분의 스타트업/중소기업에 적합

**Level 2 (6개월 내)**: External Secrets Operator
- 중앙 집중식 관리
- 자동 Rotation

**Level 3 (1년 내)**: Vault + Dynamic Secrets
- 동적 Secret 생성 (DB 계정 자동 생성/삭제)
- 단기 유효 토큰
- 완벽한 감사 로그

---

## 참고 자료

### 공식 문서
- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Helm Resource Policy](https://helm.sh/docs/howto/charts_tips_and_tricks/#tell-helm-not-to-uninstall-a-resource)
- [External Secrets Operator](https://external-secrets.io/)
- [KubeBlocks Documentation](https://kubeblocks.io/docs)

### imprun.dev 관련 문서
- [k8s/imprun/SECRETS.md](../../k8s/imprun/SECRETS.md) - Secret 관리 상세 가이드
- [k8s/FAQ.md](../../k8s/FAQ.md) - MongoDB Secret 관련 FAQ
- [Helm Chart Best Practices](./helm-chart-best-practices.md)

### 추가 학습
- [CNCF Secret Management Best Practices](https://www.cncf.io/blog/2021/03/25/secrets-management-best-practices/)
- [OWASP Kubernetes Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Kubernetes_Security_Cheat_Sheet.html)

---

**글쓴이**: imprun.dev Team
**피드백**: GitHub Issues 또는 이메일로 문의해주세요

---

## 변경 이력

- **2025-10-22**: 초안 작성
  - Helm 기반 Secret 관리 전략
  - KubeBlocks Secret 처리 방법
  - terminationPolicy: DoNotTerminate 패턴
  - 보안 체크리스트 및 RBAC 설정
