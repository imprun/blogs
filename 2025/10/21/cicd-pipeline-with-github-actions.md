# CI/CD 파이프라인 구축: GitHub Actions로 완전 자동화하기

> **작성일**: 2025-10-21
> **태그**: CI/CD, GitHub Actions, Docker, Helm, GitOps
> **난이도**: 중급~고급

## 들어가며

[**imprun.dev**](https://imprun.dev)는 Kubernetes 기반 서버리스 Cloud Function 플랫폼입니다. 초기에는 로컬에서 Docker 이미지를 빌드하고 수동으로 Helm 배포를 실행했습니다. 하지만 **서비스가 복잡해지면서 수동 배포의 한계**에 부딪혔습니다.

**수동 배포의 문제점**:
- ❌ **사람의 실수**: 잘못된 이미지 태그, 환경 설정 누락
- ❌ **배포 시간**: 빌드 → 푸시 → 배포에 10분 이상 소요
- ❌ **일관성 부족**: 개발자마다 다른 배포 절차
- ❌ **롤백 어려움**: 문제 발생 시 이전 버전 찾기 힘듦

**목표**:
- ✅ `git push` 한 번으로 테스트 → 빌드 → 배포까지 완전 자동화
- ✅ 실패 시 자동 롤백
- ✅ 모든 배포 이력 추적 가능

이 글은 **imprun.dev 플랫폼 구축 경험**을 바탕으로, GitHub Actions를 활용한 완전 자동화 CI/CD 파이프라인 구축 과정을 단계별로 공유합니다.

## 목표 아키텍처

```
┌──────────────┐
│ Developer    │
│ git push     │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────────────────────┐
│ GitHub Actions Workflow                              │
├──────────────────────────────────────────────────────┤
│ 1. Lint & Test       (타입 체크, 단위 테스트)         │
│ 2. Build Docker      (멀티 플랫폼: amd64 + arm64)    │
│ 3. Push to Registry  (Docker Hub / ECR)             │
│ 4. Update Helm       (values 파일 버전 업데이트)     │
│ 5. Deploy to K8s     (kubectl / ArgoCD)             │
│ 6. Smoke Test        (Health Check)                 │
│ 7. Notify            (Slack / Discord)              │
└──────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────┐
│ Kubernetes       │
│ Cluster          │
│ (Production)     │
└──────────────────┘
```

---

## Part 1: GitHub Actions 기초

### Workflow 파일 구조

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production  # Workflow 이름

on:  # 트리거 조건
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:  # 실행할 작업들
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: npm test

  build:
    needs: test  # test 성공 후 실행
    runs-on: ubuntu-latest
    steps:
      - name: Build Docker image
        run: docker build -t myapp .
```

### Secrets 관리

**GitHub Repository Settings → Secrets and Variables → Actions**에서 등록:

```yaml
# Workflow에서 사용
env:
  DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
  DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
  KUBECONFIG_DATA: ${{ secrets.KUBECONFIG_DATA }}
```

**필요한 Secrets**:
- `DOCKER_USERNAME`, `DOCKER_PASSWORD`: Docker Hub 인증
- `KUBECONFIG_DATA`: Kubernetes 클러스터 접근
- `SLACK_WEBHOOK_URL`: 알림 (선택)

---

## Part 2: 단계별 파이프라인 구축

### Step 1: Lint & Test

**목적**: 코드 품질 검증, 버그 조기 발견

```yaml
# .github/workflows/test.yml
name: Test & Lint

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test-server:
    name: Test API Server
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '18'
        cache: 'pnpm'

    - name: Install pnpm
      uses: pnpm/action-setup@v2
      with:
        version: 8

    - name: Install dependencies
      run: pnpm install --frozen-lockfile

    - name: TypeScript type check
      working-directory: ./server
      run: pnpm typecheck

    - name: ESLint
      working-directory: ./server
      run: pnpm lint

    - name: Run unit tests
      working-directory: ./server
      run: pnpm test
      env:
        NODE_ENV: test
        DATABASE_URL: mongodb://localhost:27017/test_db

    # MongoDB 서비스 컨테이너
    services:
      mongodb:
        image: mongo:6.0
        ports:
          - 27017:27017
        options: >-
          --health-cmd "mongosh --eval 'db.adminCommand({ping: 1})'"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

  test-console:
    name: Test Web Console
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: '18'
        cache: 'pnpm'

    - uses: pnpm/action-setup@v2
      with:
        version: 8

    - run: pnpm install --frozen-lockfile

    - name: Next.js build test
      working-directory: ./frontend
      run: pnpm build

    - name: TypeScript check
      working-directory: ./frontend
      run: pnpm typecheck
```

**핵심 포인트**:
- `services`로 MongoDB 테스트 환경 구성
- `working-directory`로 Monorepo 대응
- `cache: 'pnpm'`으로 의존성 캐싱 (빌드 시간 단축)

### Step 2: Docker 빌드 & 푸시 (멀티 플랫폼)

**목적**: ARM64 + AMD64 이미지 동시 빌드

```yaml
# .github/workflows/build-and-push.yml
name: Build & Push Docker Images

on:
  push:
    tags:
      - 'v*'  # v1.0.0, v1.0.1 등

env:
  REGISTRY: docker.io
  IMAGE_PREFIX: junsik

jobs:
  build-server:
    name: Build imprun-server
    runs-on: ubuntu-latest

    permissions:
      contents: read
      packages: write

    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Set up QEMU
      uses: docker/setup-qemu-action@v3
      with:
        platforms: linux/amd64,linux/arm64

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Extract version from tag
      id: version
      run: echo "VERSION=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT

    - name: Login to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: ./server
        file: ./server/Dockerfile
        platforms: linux/amd64,linux/arm64
        push: true
        tags: |
          ${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}/imprun-server:${{ steps.version.outputs.VERSION }}
          ${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}/imprun-server:latest
        cache-from: type=gha
        cache-to: type=gha,mode=max

    outputs:
      version: ${{ steps.version.outputs.VERSION }}

  build-console:
    name: Build imprun-console
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4

    - uses: docker/setup-qemu-action@v3
      with:
        platforms: linux/amd64,linux/arm64

    - uses: docker/setup-buildx-action@v3

    - id: version
      run: echo "VERSION=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT

    - uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - uses: docker/build-push-action@v5
      with:
        context: ./frontend
        file: ./frontend/Dockerfile
        platforms: linux/amd64,linux/arm64
        push: true
        tags: |
          ${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}/imprun-console:${{ steps.version.outputs.VERSION }}
          ${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}/imprun-console:latest
        cache-from: type=gha
        cache-to: type=gha,mode=max

    outputs:
      version: ${{ steps.version.outputs.VERSION }}
```

**핵심 포인트**:
- `docker/setup-qemu-action`: 멀티 플랫폼 빌드 활성화
- `cache-from/cache-to: type=gha`: GitHub Actions 캐시 활용 (빌드 시간 70% 단축)
- `tags`: 버전 태그 + latest 동시 푸시

### Step 3: Helm Values 업데이트 (GitOps)

**목적**: 배포 버전 추적, 롤백 가능성

```yaml
# .github/workflows/update-helm-values.yml
name: Update Helm Values

on:
  workflow_run:
    workflows: ["Build & Push Docker Images"]
    types:
      - completed

jobs:
  update-values:
    name: Update Helm Values
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}

    steps:
    - name: Checkout
      uses: actions/checkout@v4
      with:
        token: ${{ secrets.PAT_TOKEN }}  # Personal Access Token (write 권한)

    - name: Extract version
      id: version
      run: echo "VERSION=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT

    - name: Update values-production.yaml
      run: |
        # yq 설치 (YAML 파서)
        sudo wget -qO /usr/local/bin/yq https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64
        sudo chmod +x /usr/local/bin/yq

        # imprun-server 이미지 태그 업데이트
        yq eval -i '.imprun-server.image.tag = "${{ steps.version.outputs.VERSION }}"' k8s/imprun/values-production.yaml

        # imprun-console 이미지 태그 업데이트
        yq eval -i '.imprun-console.image.tag = "${{ steps.version.outputs.VERSION }}"' k8s/imprun/values-production.yaml

    - name: Commit and push
      run: |
        git config user.name "github-actions[bot]"
        git config user.email "github-actions[bot]@users.noreply.github.com"

        git add k8s/imprun/values-production.yaml
        git commit -m "chore: update image tags to ${{ steps.version.outputs.VERSION }}"
        git push
```

**결과**:
```yaml
# k8s/imprun/values-production.yaml (자동 업데이트됨)
imprun-server:
  image:
    tag: 1.2.3  # v1.2.3 태그에서 자동 추출
```

### Step 4: Kubernetes 배포

**방법 A: kubectl 직접 사용 (간단)**

```yaml
# .github/workflows/deploy-k8s.yml
name: Deploy to Kubernetes

on:
  push:
    branches: [main]
    paths:
      - 'k8s/imprun/values-production.yaml'

jobs:
  deploy:
    name: Deploy to Production
    runs-on: ubuntu-latest

    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Setup kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: 'v1.28.0'

    - name: Configure kubeconfig
      run: |
        mkdir -p ~/.kube
        echo "${{ secrets.KUBECONFIG_DATA }}" | base64 -d > ~/.kube/config
        chmod 600 ~/.kube/config

    - name: Install Helm
      uses: azure/setup-helm@v3
      with:
        version: 'v3.13.0'

    - name: Deploy with Helm
      run: |
        cd k8s/imprun
        helm dependency update
        helm upgrade imprun . \
          --namespace imprun-system \
          --values values-production.yaml \
          --wait \
          --timeout 10m

    - name: Verify deployment
      run: |
        kubectl rollout status deployment/imprun-imprun-server -n imprun-system
        kubectl rollout status deployment/imprun-imprun-console -n imprun-system

    - name: Run smoke test
      run: |
        # Health check 엔드포인트 확인
        kubectl run curl-test --rm -i --restart=Never --image=curlimages/curl -- \
          curl -f http://imprun-imprun-server.imprun-system.svc/v1/health/liveness

        echo "✅ Deployment successful!"
```

**방법 B: ArgoCD (GitOps, 권장)**

```yaml
# .github/workflows/trigger-argocd.yml
name: Trigger ArgoCD Sync

on:
  push:
    branches: [main]
    paths:
      - 'k8s/imprun/values-production.yaml'

jobs:
  sync:
    name: Sync ArgoCD Application
    runs-on: ubuntu-latest

    steps:
    - name: Trigger ArgoCD sync
      run: |
        # ArgoCD CLI 설치
        curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
        chmod +x argocd
        sudo mv argocd /usr/local/bin/

        # ArgoCD 로그인
        argocd login ${{ secrets.ARGOCD_SERVER }} \
          --username ${{ secrets.ARGOCD_USERNAME }} \
          --password ${{ secrets.ARGOCD_PASSWORD }} \
          --insecure

        # Application 동기화
        argocd app sync imprun-production --prune

        # 동기화 완료 대기
        argocd app wait imprun-production --health
```

**ArgoCD Application 정의**:
```yaml
# argocd/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: imprun-production
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/imprun/imprun.dev
    targetRevision: main
    path: k8s/imprun
    helm:
      valueFiles:
        - values-production.yaml

  destination:
    server: https://kubernetes.default.svc
    namespace: imprun-system

  syncPolicy:
    automated:
      prune: true      # 삭제된 리소스 자동 제거
      selfHeal: true   # Drift 자동 복구
      allowEmpty: false

    syncOptions:
    - CreateNamespace=true

  revisionHistoryLimit: 10  # 롤백 가능한 히스토리 수
```

---

## Part 3: 버전 태깅 자동화

### 시맨틱 버저닝 자동화

**Conventional Commits 사용**:

```
feat: 새로운 기능 추가 → MINOR 버전 증가 (1.2.0 → 1.3.0)
fix: 버그 수정 → PATCH 버전 증가 (1.2.0 → 1.2.1)
BREAKING CHANGE: 호환성 깨짐 → MAJOR 버전 증가 (1.2.0 → 2.0.0)
```

**semantic-release 설정**:

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    branches: [main]

jobs:
  release:
    name: Create Release
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0  # 전체 히스토리 필요

    - uses: actions/setup-node@v4
      with:
        node-version: '18'

    - name: Install semantic-release
      run: npm install -g semantic-release @semantic-release/git @semantic-release/changelog

    - name: Run semantic-release
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      run: semantic-release
```

**설정 파일** (`.releaserc.json`):

```json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    "@semantic-release/changelog",
    [
      "@semantic-release/git",
      {
        "assets": ["CHANGELOG.md", "package.json"],
        "message": "chore(release): ${nextRelease.version} [skip ci]\n\n${nextRelease.notes}"
      }
    ],
    "@semantic-release/github"
  ]
}
```

**결과**:
1. 커밋 분석 → 버전 결정
2. CHANGELOG.md 자동 생성
3. Git 태그 생성 (`v1.3.0`)
4. GitHub Release 생성
5. Docker 빌드 파이프라인 트리거!

---

## Part 4: 자동 롤백 전략

### Health Check 실패 시 자동 롤백

```yaml
# .github/workflows/deploy-with-rollback.yml
name: Deploy with Auto Rollback

on:
  push:
    branches: [main]

jobs:
  deploy:
    name: Deploy to Production
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4

    # ... (kubectl 설정, Helm 배포)

    - name: Wait for rollout
      id: rollout
      run: |
        kubectl rollout status deployment/imprun-imprun-server -n imprun-system --timeout=5m
      continue-on-error: true

    - name: Health check
      id: health
      if: steps.rollout.outcome == 'success'
      run: |
        sleep 30  # Pod 안정화 대기

        # 5번 재시도
        for i in {1..5}; do
          if kubectl run curl-test --rm -i --restart=Never --image=curlimages/curl -- \
            curl -f http://imprun-imprun-server.imprun-system.svc/v1/health/liveness; then
            echo "Health check passed!"
            exit 0
          fi
          echo "Retry $i/5..."
          sleep 10
        done

        echo "Health check failed!"
        exit 1
      continue-on-error: true

    - name: Rollback on failure
      if: steps.health.outcome == 'failure' || steps.rollout.outcome == 'failure'
      run: |
        echo "🔴 Deployment failed! Rolling back..."

        # Helm rollback
        helm rollback imprun -n imprun-system

        # 롤백 성공 확인
        kubectl rollout status deployment/imprun-imprun-server -n imprun-system

        # Slack 알림
        curl -X POST ${{ secrets.SLACK_WEBHOOK_URL }} \
          -H 'Content-Type: application/json' \
          -d '{
            "text": "❌ Production deployment failed and rolled back!",
            "attachments": [{
              "color": "danger",
              "fields": [
                {"title": "Commit", "value": "${{ github.sha }}", "short": true},
                {"title": "Author", "value": "${{ github.actor }}", "short": true}
              ]
            }]
          }'

        exit 1

    - name: Success notification
      if: steps.health.outcome == 'success'
      run: |
        curl -X POST ${{ secrets.SLACK_WEBHOOK_URL }} \
          -H 'Content-Type: application/json' \
          -d '{
            "text": "✅ Production deployment successful!",
            "attachments": [{
              "color": "good",
              "fields": [
                {"title": "Version", "value": "${{ steps.version.outputs.VERSION }}", "short": true},
                {"title": "Author", "value": "${{ github.actor }}", "short": true}
              ]
            }]
          }'
```

### Canary Deployment (점진적 배포)

```yaml
# .github/workflows/canary-deploy.yml
name: Canary Deployment

on:
  push:
    tags:
      - 'v*'

jobs:
  canary:
    name: Canary Deployment
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4

    # ... (kubectl 설정)

    - name: Deploy canary (10%)
      run: |
        # Canary Deployment 생성
        kubectl set image deployment/imprun-imprun-server-canary \
          api-server=${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}/imprun-server:${{ steps.version.outputs.VERSION }} \
          -n imprun-system

        kubectl scale deployment/imprun-imprun-server-canary --replicas=1 -n imprun-system

    - name: Monitor canary
      run: |
        sleep 300  # 5분 모니터링

        # 에러율 확인 (Prometheus)
        ERROR_RATE=$(curl -s 'http://vmsingle.monitoring.svc:8428/api/v1/query' \
          --data-urlencode 'query=rate(http_requests_total{status=~"5..",deployment="canary"}[5m])')

        if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
          echo "Canary error rate too high! Aborting..."
          kubectl scale deployment/imprun-imprun-server-canary --replicas=0 -n imprun-system
          exit 1
        fi

    - name: Promote canary to production
      run: |
        # Production Deployment 업데이트
        kubectl set image deployment/imprun-imprun-server \
          api-server=${{ env.REGISTRY }}/${{ env.IMAGE_PREFIX }}/imprun-server:${{ steps.version.outputs.VERSION }} \
          -n imprun-system

        kubectl rollout status deployment/imprun-imprun-server -n imprun-system

        # Canary 제거
        kubectl scale deployment/imprun-imprun-server-canary --replicas=0 -n imprun-system
```

---

## Part 5: 고급 패턴

### 1. Matrix Strategy (병렬 빌드)

**여러 서비스 동시 빌드**:

```yaml
jobs:
  build:
    name: Build ${{ matrix.service }}
    runs-on: ubuntu-latest

    strategy:
      matrix:
        service:
          - imprun-server
          - imprun-console
          - imp-runtime-exporter
        include:
          - service: imprun-server
            context: ./server
            dockerfile: ./server/Dockerfile
          - service: imprun-console
            context: ./frontend
            dockerfile: ./frontend/Dockerfile
          - service: imp-runtime-exporter
            context: ./runtimes/imp-runtime-exporter
            dockerfile: ./runtimes/imp-runtime-exporter/Dockerfile

    steps:
    - uses: actions/checkout@v4

    - uses: docker/build-push-action@v5
      with:
        context: ${{ matrix.context }}
        file: ${{ matrix.dockerfile }}
        platforms: linux/amd64,linux/arm64
        push: true
        tags: junsik/${{ matrix.service }}:${{ github.sha }}
```

### 2. Reusable Workflows

**공통 워크플로우 재사용**:

```yaml
# .github/workflows/reusable-build.yml
name: Reusable Build

on:
  workflow_call:
    inputs:
      service-name:
        required: true
        type: string
      context:
        required: true
        type: string
    secrets:
      docker-username:
        required: true
      docker-password:
        required: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/build-push-action@v5
        with:
          context: ${{ inputs.context }}
          push: true
          tags: junsik/${{ inputs.service-name }}:latest
```

**호출**:

```yaml
# .github/workflows/build-server.yml
name: Build Server

on: [push]

jobs:
  build-server:
    uses: ./.github/workflows/reusable-build.yml
    with:
      service-name: imprun-server
      context: ./server
    secrets:
      docker-username: ${{ secrets.DOCKER_USERNAME }}
      docker-password: ${{ secrets.DOCKER_PASSWORD }}
```

### 3. Environment Protection Rules

**GitHub Settings → Environments → production**:

```yaml
# .github/workflows/deploy.yml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://api.imprun.dev

    steps:
      # ... 배포 스텝
```

**설정**:
- ✅ Required reviewers: 1명 이상 승인 필요
- ✅ Wait timer: 5분 대기 (실수 방지)
- ✅ Deployment branches: main만 허용

---

## Part 6: 모니터링 & 알림

### Slack 알림 통합

```yaml
- name: Notify deployment start
  uses: slackapi/slack-github-action@v1
  with:
    payload: |
      {
        "text": "🚀 Deployment started",
        "blocks": [
          {
            "type": "section",
            "text": {
              "type": "mrkdwn",
              "text": "*Deployment Started*\n*Commit:* <${{ github.event.head_commit.url }}|${{ github.sha }}>\n*Author:* ${{ github.actor }}"
            }
          }
        ]
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### Discord Webhook

```yaml
- name: Discord notification
  run: |
    curl -X POST ${{ secrets.DISCORD_WEBHOOK_URL }} \
      -H 'Content-Type: application/json' \
      -d '{
        "username": "Deploy Bot",
        "avatar_url": "https://github.githubassets.com/images/modules/logos_page/GitHub-Mark.png",
        "embeds": [{
          "title": "✅ Deployment Successful",
          "description": "Version: v${{ steps.version.outputs.VERSION }}",
          "color": 5763719,
          "fields": [
            {"name": "Commit", "value": "${{ github.sha }}", "inline": true},
            {"name": "Author", "value": "${{ github.actor }}", "inline": true}
          ],
          "timestamp": "${{ github.event.head_commit.timestamp }}"
        }]
      }'
```

---

## 실전 시나리오

### 시나리오 1: 핫픽스 긴급 배포

```bash
# 1. hotfix 브랜치 생성
git checkout -b hotfix/critical-bug main

# 2. 버그 수정 후 커밋
git commit -m "fix: critical security vulnerability (CVE-2024-1234)"

# 3. 푸시 (자동 테스트 실행)
git push origin hotfix/critical-bug

# 4. PR 생성 & 승인 (Production environment protection)

# 5. main 병합 → 자동 배포!
git checkout main
git merge hotfix/critical-bug
git push

# 6. GitHub Actions가 자동으로:
#    - 테스트
#    - 빌드 (v1.2.4-hotfix.1 태그)
#    - 배포
#    - Slack 알림
```

**소요 시간**: 5~10분 (수동 승인 포함)

### 시나리오 2: 배포 실패 후 롤백

```bash
# Workflow 로그:
# ✅ Build successful
# ✅ Push to Docker Hub
# ✅ Helm upgrade
# ❌ Health check failed! (5/5 retries)
# 🔄 Rolling back...
# ✅ Rollback successful

# Slack 알림:
# ❌ Production deployment failed and rolled back!
# Commit: abc123
# Reason: Health check timeout

# 조치:
# 1. 로컬에서 디버깅
# 2. 수정 후 재배포
```

---

## 체크리스트

### CI/CD 파이프라인 설계
- [ ] 테스트 → 빌드 → 배포 단계 명확히 분리
- [ ] 각 단계마다 실패 시 중단
- [ ] 병렬 실행 가능한 작업은 Matrix Strategy 사용
- [ ] Reusable Workflows로 중복 제거

### 보안
- [ ] Secrets는 GitHub Secrets에 저장 (코드에 하드코딩 금지)
- [ ] KUBECONFIG는 base64 인코딩 후 저장
- [ ] Docker Hub Token 사용 (패스워드 대신)
- [ ] Environment Protection Rules 설정 (Production)

### 배포 안정성
- [ ] Health Check 구현 (Liveness + Readiness)
- [ ] Rollback 전략 수립
- [ ] Canary Deployment (선택)
- [ ] Deployment 타임아웃 설정 (10분)

### 모니터링
- [ ] Slack/Discord 알림 설정
- [ ] 배포 성공/실패 알림
- [ ] Rollout 상태 모니터링
- [ ] Workflow 실행 시간 추적

---

## 결론

완전 자동화된 CI/CD 파이프라인의 장점:

1. **빠른 배포**: 코드 푸시 → 프로덕션 배포 5분
2. **안정성**: 자동 테스트 + Health Check + Rollback
3. **추적성**: Git 태그 = Docker 태그 = Helm 버전
4. **협업**: PR 리뷰 → 승인 → 자동 배포

imprun.dev는 이 파이프라인을 통해 하루 10회 이상 안전하게 배포하며, 장애 발생 시 1분 내 롤백이 가능합니다.

## 참고 자료

- [GitHub Actions 공식 문서](https://docs.github.com/en/actions)
- [Docker Build Push Action](https://github.com/docker/build-push-action)
- [Helm 공식 문서](https://helm.sh/docs/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Semantic Release](https://github.com/semantic-release/semantic-release)

---

**질문이나 피드백은 GitHub Issues로!**
📧 [GitHub Issues](https://github.com/imprun/imprun.dev/issues)

## 시리즈 전체 보기

1. [Kubernetes 이미지 업데이트 전략](./kubernetes-image-update-strategy.md)
2. [Helm 차트 관리 Best Practices](./helm-chart-best-practices.md)
3. [Kubernetes 리소스 최적화](./kubernetes-resource-optimization.md)
4. **CI/CD 파이프라인 구축** (현재 글)
