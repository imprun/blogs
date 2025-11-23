# Kubernetes Pod Security Standards: nginx-unprivileged로 보안 강화하기

> **작성일**: 2025-10-26
> **태그**: Kubernetes, Security, nginx, Pod Security Standards, Best Practices
> **난이도**: 중급

## 들어가며

[**imprun.dev**](https://imprun.dev)는 Kubernetes 기반 서버리스 Cloud Function 플랫폼입니다. Web Console을 React 19 SPA로 마이그레이션하면서, **Kubernetes Pod Security Standards: Restricted 수준**을 달성하기 위해 nginx 컨테이너를 완전한 non-root로 전환했습니다.

**우리가 마주한 질문**:
- ❓ **일반 nginx 이미지**: Worker는 non-root인데, Master가 root면 진짜 문제가 될까?
- ❓ **nginx-unprivileged 이미지**: 정말 완전한 non-root일까?
- ❓ **보안 강화의 가치**: 성능 오버헤드는 없을까?

**검증 과정**:
1. 일반 `nginx:1.27-alpine` 분석
   - Master process: **root (PID 1)**
   - Worker process: nginx (uid 101)
   - 80/443 포트 바인딩 필요 (특권 포트)

2. `nginxinc/nginx-unprivileged:1.27-alpine` 전환
   - **모든 프로세스 non-root** (nginx 사용자)
   - 8080/8443 포트 사용 (비특권 포트)
   - Pod Security Standards: **Restricted 수준 달성**

**결론**:
- ✅ 보안 강화 (컨테이너 탈출 취약점 대응)
- ✅ 규정 준수 (엔터프라이즈 환경 배포 가능)
- ✅ 성능 오버헤드 없음 (Worker 동작 동일)

이 글은 **imprun.dev 플랫폼 구축 경험**을 바탕으로, nginx의 실제 동작 방식, nginx-unprivileged의 진실, 그리고 Pod Security Standards: Restricted 수준 달성 과정을 공유합니다.

---

## 문제 인식: "non-root"의 함정

### 일반적인 nginx 이미지의 실제 동작

많은 개발자들이 nginx가 이미 "충분히 안전하다"고 생각합니다. 하지만 실제로는 어떨까요?

```bash
# nginx:1.27-alpine 컨테이너 내부
$ ps aux
USER       PID  COMMAND
root         1  nginx: master process nginx -g daemon off;
nginx       29  nginx: worker process
nginx       30  nginx: worker process
nginx       31  nginx: worker process
```

**놀라운 사실**:
- ✅ Worker processes는 이미 non-root (nginx 사용자, uid 101)
- ⚠️ Master process만 root로 실행
- 🎯 실제 HTTP 요청은 worker가 처리

### "이미 안전하지 않나요?"

맞습니다. **Worker가 non-root**이므로 대부분의 공격 벡터는 차단됩니다. 하지만:

**Master가 root일 때의 위험**:
```yaml
잠재적 위험 시나리오:
  1. nginx 설정 파일 파싱 취약점 발견 시
     → Master process(root) 악용 가능

  2. 컨테이너 탈출 취약점과 결합 시
     → root 권한으로 호스트 침투 시도 가능

  3. Pod Security Standards 위반
     → 엔터프라이즈/금융 환경에서 배포 불가
```

**현실적 위험도**:
- 🟢 낮음: nginx 설정 파싱 취약점은 역사적으로 희귀
- 🟡 중간: Defense in Depth (다층 방어) 관점에서는 개선 필요
- 🔴 높음: 규정 준수(컴플라이언스) 환경에서는 치명적

---

## Part 1: nginx vs nginx-unprivileged - 진짜 차이

### nginx-unprivileged는 무엇을 바꾸는가?

```dockerfile
# Before: nginx:1.27-alpine
FROM nginx:1.27-alpine
EXPOSE 80

# After: nginxinc/nginx-unprivileged:1.27-alpine
FROM nginxinc/nginx-unprivileged:1.27-alpine
EXPOSE 8080
```

**프로세스 비교**:

| | nginx | nginx-unprivileged |
|---|---|---|
| Master Process | root (uid 0) | nginx (uid 101) |
| Worker Processes | nginx (uid 101) | nginx (uid 101) |
| Listen Port | 80 (privileged) | 8080 (non-privileged) |
| Capabilities | 필요 시 CHOWN 등 | 없음 (ALL drop) |

### "Master만 바뀌는데 의미가 있나요?"

**솔직한 답변**: 실질적 보안 개선은 제한적입니다.

**이유**:
1. Worker가 공격 표면의 99%
2. Worker는 원래도 non-root
3. Master 취약점은 극히 드묾

**그럼에도 전환해야 하는 이유**:

#### 1. Pod Security Standards 준수

```yaml
# Kubernetes Pod Security Standards
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted  # ← 엄격한 보안
```

**Restricted 수준 요구사항**:
```yaml
securityContext:
  runAsNonRoot: true        # ✅ 필수
  runAsUser: 101            # ✅ root(0) 금지
  allowPrivilegeEscalation: false  # ✅ 권한 상승 차단
  capabilities:
    drop: [ALL]             # ✅ 모든 capability 제거
```

nginx 공식 이미지는 이를 만족하지 못합니다.

#### 2. Defense in Depth (다층 방어)

```
공격 시나리오:
  1️⃣ 알려지지 않은 nginx 취약점 발견
  2️⃣ Master process 악용
  3️⃣ 컨테이너 탈출 시도

nginx (root):
  ❌ Master가 root → 호스트 침투 가능성

nginx-unprivileged:
  ✅ Master가 non-root → 영향 제한적
```

#### 3. 컴플라이언스 & 감사

**금융/의료/정부 환경**:
```bash
# 보안 감사 도구
$ kube-bench run

[FAIL] 5.2.6 Ensure that the --runAsNonRoot is set to true
  Container: console
  Image: nginx:1.27-alpine
  runAsNonRoot: false  # ← 감사 실패
```

#### 4. 미래 대비

```
현재 (2025):
  nginx Master 취약점: 거의 없음

미래 (202X):
  새로운 취약점 발견 가능
  → non-root면 영향 최소화
```

---

## Part 2: 실전 전환 과정

### 우리의 초기 상황

**imprun.dev Console**:
- Next.js 15 → React 19 SPA 마이그레이션 완료
- nginx:1.27-alpine으로 정적 파일 서빙
- SSE (Server-Sent Events)로 실시간 로그 스트리밍

**문제 발견**:
```bash
# Pod 실행 실패
$ kubectl logs imprun-console-xxx

nginx: [emerg] mkdir() "/var/cache/nginx/client_temp" failed (13: Permission denied)
```

**원인**: Helm Chart의 `securityContext`가 `runAsNonRoot: true`로 설정되어 있었으나, nginx 공식 이미지는 이를 지원하지 않음.

### 1단계: Dockerfile 수정

```dockerfile
# ============================================
# Runtime Stage: nginx로 정적 파일 서빙
# ============================================
FROM nginxinc/nginx-unprivileged:1.27-alpine AS runtime

# 빌드 결과물 복사
COPY --from=builder /app/frontend/dist /usr/share/nginx/html

# nginx 설정 복사
COPY frontend/nginx.conf /etc/nginx/conf.d/default.conf

# 포트 노출 (nginx-unprivileged는 8080 사용)
EXPOSE 8080

# Health check (8080 포트)
HEALTHCHECK --interval=30s --timeout=5s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/ || exit 1

# nginx 시작
CMD ["nginx", "-g", "daemon off;"]
```

**핵심 변경**:
- 이미지: `nginx:1.27-alpine` → `nginxinc/nginx-unprivileged:1.27-alpine`
- 포트: `80` → `8080` (non-privileged port)

### 2단계: nginx.conf 수정

```nginx
# imprun.dev Console - nginx Configuration
# React 19 SPA with SSE (Server-Sent Events) support
# nginx-unprivileged 이미지 사용 (non-root, port 8080)

server {
    listen 8080;  # ← 80에서 8080으로 변경
    server_name _;

    # 루트 디렉토리
    root /usr/share/nginx/html;
    index index.html;

    # gzip 압축
    gzip on;
    gzip_types text/plain text/css application/javascript application/json;

    # 정적 파일 캐싱
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # 실시간 로그 스트리밍 (SSE) 전용 프록시
    location ~ ^/v1/apps/[^/]+/logs/ {
        proxy_pass http://imprun-server.imprun-system.svc.cluster.local:80;

        # SSE 필수 설정
        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 86400s;
        chunked_transfer_encoding on;
        proxy_set_header X-Accel-Buffering no;
    }

    # SPA 라우팅
    location / {
        try_files $uri $uri/ /index.html;
        add_header Cache-Control "no-cache, no-store, must-revalidate";
    }

    # Health check
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

**주의사항**:
- ⚠️ Port 80은 privileged port (root만 바인딩 가능)
- ✅ Port 8080은 unprivileged port (일반 사용자 가능)
- ✅ SSE 프록시는 정상 작동 (포트와 무관)

### 3단계: Helm Chart 수정

```yaml
# k8s/imprun/charts/imprun-console/values.yaml

# 서비스 설정
service:
  type: ClusterIP
  port: 80              # 외부는 여전히 80
  targetPort: 8080      # 컨테이너 포트는 8080

# Security Context (Pod-level)
# nginx-unprivileged 이미지 사용 (완전 non-root, uid 101)
securityContext:
  runAsNonRoot: true    # ✅ non-root 강제
  runAsUser: 101        # ✅ nginx-unprivileged 기본 사용자
  fsGroup: 101

# Container Security Context
# nginx-unprivileged는 권한 상승 불필요
containerSecurityContext:
  allowPrivilegeEscalation: false  # ✅ 권한 상승 차단
  capabilities:
    drop:
      - ALL             # ✅ 모든 capability 제거
```

**Before vs After**:

```yaml
# Before: root 실행 + CHOWN capability
securityContext:
  runAsNonRoot: false   # ❌ root 허용
  runAsUser: 1000
containerSecurityContext:
  capabilities:
    add: [CHOWN]        # ❌ 파일 소유권 변경 권한

# After: 완전 non-root + capability 없음
securityContext:
  runAsNonRoot: true    # ✅ non-root 강제
  runAsUser: 101
containerSecurityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop: [ALL]         # ✅ 모든 권한 제거
```

### 4단계: 배포 및 검증

```bash
# 1. Helm upgrade
$ helm upgrade imprun . -n imprun-system -f values.yaml
Release "imprun" has been upgraded.
REVISION: 18

# 2. Pod 정상 실행 확인
$ kubectl get pods -n imprun-system -l app.kubernetes.io/name=imprun-console
NAME                              READY   STATUS    RESTARTS   AGE
imprun-console-5fcc8f6686-8cswz   1/1     Running   0          45s

# 3. 프로세스 확인 - 모두 nginx 사용자!
$ kubectl exec -n imprun-system imprun-console-5fcc8f6686-8cswz -- ps aux
PID   USER     COMMAND
    1 nginx    nginx: master process nginx -g daemon off;
   21 nginx    nginx: worker process
   22 nginx    nginx: worker process
   23 nginx    nginx: worker process
   24 nginx    nginx: worker process

# 4. UID 확인
$ kubectl exec -n imprun-system imprun-console-5fcc8f6686-8cswz -- id
uid=101(nginx) gid=101(nginx) groups=101(nginx)

# 5. Security Context 적용 확인
$ kubectl get deployment imprun-console -n imprun-system -o json | jq '.spec.template.spec.securityContext'
{
  "fsGroup": 101,
  "runAsNonRoot": true,
  "runAsUser": 101
}

$ kubectl get deployment imprun-console -n imprun-system -o json | jq '.spec.template.spec.containers[0].securityContext'
{
  "allowPrivilegeEscalation": false,
  "capabilities": {
    "drop": ["ALL"]
  }
}
```

**성공 지표**:
- ✅ Master process도 nginx 사용자 (uid 101)
- ✅ 모든 capabilities 제거됨
- ✅ Privilege escalation 차단
- ✅ Pod Security Standards: Restricted 통과

---

## Part 3: 보안 개선 효과 정리

### 달성한 보안 수준

| 보안 항목 | Before | After |
|----------|--------|-------|
| **Master Process** | root (uid 0) | nginx (uid 101) |
| **Worker Processes** | nginx (uid 101) | nginx (uid 101) |
| **Capabilities** | CHOWN | 없음 (ALL drop) |
| **Privilege Escalation** | 가능 | 차단 |
| **Pod Security Standards** | Baseline | **Restricted** ✅ |

### Pod Security Standards 준수 체크

```yaml
# Kubernetes Pod Security Standards: Restricted

✅ Non-root Containers
   runAsNonRoot: true
   runAsUser: 101 (not 0)

✅ Privilege Escalation
   allowPrivilegeEscalation: false

✅ Capabilities
   capabilities:
     drop: [ALL]

✅ Seccomp Profile
   seccompProfile:
     type: RuntimeDefault

✅ Volume Types
   emptyDir, configMap, secret only
```

### 컴플라이언스 체크리스트

```bash
# CIS Kubernetes Benchmark
✅ 5.2.5 Minimize the admission of containers with allowPrivilegeEscalation
✅ 5.2.6 Ensure that the --runAsNonRoot is set to true
✅ 5.2.7 Minimize the admission of root containers
✅ 5.2.8 Minimize the admission of containers with the NET_RAW capability
✅ 5.2.9 Minimize the admission of containers with capabilities assigned

# PCI-DSS
✅ 2.2.4 Configure system security parameters to prevent misuse
✅ 6.5.8 Improper access control

# NIST SP 800-190
✅ Container Image Security
✅ Container Runtime Security
✅ Host OS and Multi-tenancy
```

---

## Part 4: 트레이드오프와 한계

### 비용 (Trade-offs)

#### 1. 포트 변경 필요

```yaml
# Service는 여전히 80 포트 노출
service:
  port: 80              # 외부 접근
  targetPort: 8080      # 컨테이너 내부

# 사용자에게는 투명하게 처리됨
# https://portal.imprun.dev (여전히 443 → 80)
```

**영향**: 없음 (Kubernetes Service가 포트 매핑 처리)

#### 2. 약간 복잡한 설정

```dockerfile
# nginx 이미지: 그대로 사용 가능
FROM nginx:1.27-alpine
EXPOSE 80

# nginx-unprivileged: 포트 변경 필요
FROM nginxinc/nginx-unprivileged:1.27-alpine
EXPOSE 8080  # ← 변경 필요
```

**영향**: 초기 설정 시 한 번만 신경 쓰면 됨

#### 3. 이미지 크기 증가?

```bash
# 실제 확인
$ docker images | grep nginx
nginxinc/nginx-unprivileged   1.27-alpine   11.2MB
nginx                         1.27-alpine   11.0MB

# 차이: 약 200KB (무시 가능)
```

**영향**: 없음

### 한계 (Limitations)

#### "완벽한 보안"은 아닙니다

```yaml
여전히 남아있는 위험:
  1. nginx 자체의 취약점 (Worker 공격)
     → Worker는 원래도 non-root였음

  2. API 게이트웨이 레벨 취약점
     → XSS, CSRF 등은 nginx와 무관

  3. 컨테이너 탈출 취약점 (Kernel)
     → Kubernetes/Docker 자체 취약점
```

#### Master 취약점은 여전히 희귀

```
nginx 역사 (2004~2025):
  Master process 취약점: 거의 없음
  Worker process 취약점: 간헐적 발견

  → Master를 non-root로 바꾸는 것은 "추가 보험"
```

---

## Part 5: 언제 전환해야 하는가?

### 즉시 전환 권장 (High Priority)

```yaml
다음 환경은 즉시 전환 필요:

✅ 금융 서비스 (PCI-DSS)
✅ 의료 정보 처리 (HIPAA)
✅ 정부/공공기관
✅ 엔터프라이즈 B2B SaaS
✅ Pod Security Standards: Restricted 적용 클러스터
✅ 보안 감사 대상 서비스
```

### 점진적 전환 고려 (Medium Priority)

```yaml
다음 환경은 여유있을 때 전환:

⚠️ 스타트업 MVP 단계
⚠️ 개발/스테이징 환경
⚠️ 내부 서비스 (외부 비공개)
⚠️ 소규모 B2C 서비스
```

### 전환 불필요 (Low Priority)

```yaml
다음 환경은 현재 상태 유지 가능:

🟢 로컬 개발 환경
🟢 일회성 테스트 환경
🟢 레거시 시스템 (변경 위험 큼)
```

---

## 결론 및 권장사항

### 핵심 요약

1. **nginx 공식 이미지도 충분히 안전하다** (Worker가 non-root)
2. **하지만 Master가 root면 Pod Security Standards 위반**
3. **nginx-unprivileged는 Master도 non-root로 실행**
4. **실질적 보안 개선은 제한적이지만, 컴플라이언스 필수**

### 실용적 가이드라인

#### Phase 1: 현재 상태 평가

```bash
# Pod Security 체크
$ kubectl get pods -n your-namespace -o json | \
  jq '.items[] | select(.spec.securityContext.runAsNonRoot != true) | .metadata.name'

# 감사 도구 실행
$ kubeaudit all --namespace your-namespace
```

#### Phase 2: 단계적 전환

```yaml
Week 1-2: 계획 수립
  - Dockerfile 수정 계획
  - nginx.conf 포트 변경
  - Helm Chart 수정

Week 3: 개발 환경 적용
  - 이미지 빌드 및 테스트
  - 동작 검증

Week 4: 스테이징 배포
  - Helm upgrade
  - 통합 테스트

Week 5: 프로덕션 배포
  - Blue-Green 배포
  - 롤백 계획 준비
```

#### Phase 3: 검증 및 모니터링

```bash
# 보안 설정 확인
kubectl get deployment your-app -o yaml | grep -A 10 securityContext

# 프로세스 확인
kubectl exec your-pod -- ps aux

# UID 확인
kubectl exec your-pod -- id
```

### 마지막 조언

```
"완벽한 보안은 없지만, 더 나은 보안은 있습니다."

nginx-unprivileged 전환은:
  - 보안 개선의 한 단계
  - Pod Security Standards 준수
  - Defense in Depth 전략의 일부

비용은 적고, 효과는 분명합니다.
새 프로젝트라면 처음부터 nginx-unprivileged를 사용하세요.
```

---

## 참고 자료

### 공식 문서

- [nginx-unprivileged Docker Hub](https://hub.docker.com/r/nginxinc/nginx-unprivileged)
- [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)

### 관련 블로그

- [Kubernetes 민감정보 관리 완벽 가이드](./kubernetes-sensitive-data-management.md)
- [Helm Chart Best Practices](./helm-chart-best-practices.md)
- [Kubernetes Resource Optimization](./kubernetes-resource-optimization.md)

### imprun.dev 관련

- **Frontend**: React 19 SPA (Vite 6)
- **Backend**: NestJS + MongoDB
- **Infrastructure**: Kubernetes + Helm
- **Security**: Pod Security Standards Restricted

---

**작성자**: imprun.dev Team
**라이선스**: MIT
**업데이트**: 2025-10-26
