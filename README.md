# imprun-blogs

imprun.dev 기술 블로그 원본 저장소

## 구조

```
imprun-blogs/
├── CLAUDE.md          # 블로그 작성 가이드
├── index.md           # 블로그 번호 ↔ 파일 매핑
├── draft/             # 작성 중인 초안
└── 2025/MM/DD/        # 발행된 블로그 (날짜별)
```

## 블로그 작성

1. `draft/` 폴더에 초안 작성
2. [CLAUDE.md](./CLAUDE.md) 가이드 준수
3. 완성 후 `2025/MM/DD/` 폴더로 이동
4. [README.md](./README.md)에 번호 추가

## 발행

- 블로그 URL: https://blog.imprun.dev/{번호}

# 블로그 entity mapping

> 블로그 번호는 +1 하면 된다.
>
> **블로그 작성 가이드는 [CLAUDE.md](./CLAUDE.md)를 참고하세요.**

---

|  실제 블로그 링크 | 원본 markdown link |
|-----|------|
| https://blog.imprun.dev/103 | [Claude Code 컨텍스트 최적화 실전 가이드](2025/12/29/claude-code-context-token-optimization.md) |
| https://blog.imprun.dev/102 | [Claude Code v2.0.76 LSP 트러블슈팅: 레이스 컨디션 버그와 해결책](2025/12/28/claude-code-lsp-troubleshooting.md) |
| https://blog.imprun.dev/101 | [Git Worktree로 AI 에이전트 동시 개발하기: 실전 튜토리얼](2025/12/28/git-worktree-multi-agent-tutorial.md) |
| https://blog.imprun.dev/100 | [Claude Code가 AGENTS.md를 무시할 때: 심볼릭 링크 해결법](2025/12/28/claude-code-agents-md-symlink-fix.md) |
| https://blog.imprun.dev/99 | [Google Antigravity 업데이트 주의: 같은 폴더에서 AI 도구 동시 사용 시 작업 손실 위험](2025/12/22/google-antigravity-update-git-checkout-warning.md) |
| https://blog.imprun.dev/98 | [HTTP Cookie Deep Dive: 웹 상태 관리의 핵심](2025/12/22/http-cookie-deep-dive.md) |
| https://blog.imprun.dev/97 | [Next.js + Tailwind CSS v4 + shadcn/ui 테마 시스템 구축 가이드](2025/12/22/nextjs-tailwind-v4-shadcn-theming-guide.md) |
| https://blog.imprun.dev/96 | [GORM 실무 트러블슈팅: 운영 환경에서 만난 함정들](2025/12/22/gorm-edge-cases-troubleshooting.md) |
| https://blog.imprun.dev/95 | [GORM 기반 엔터프라이즈 Go API Server 아키텍처](2025/12/22/gorm-enterprise-go-api-architecture.md) |
| https://blog.imprun.dev/94 | [GORM 소개: Go 개발자를 위한 ORM 완벽 가이드](2025/12/22/gorm-introduction-go-orm.md) |
| https://blog.imprun.dev/93 | [Claude Code로 세련된 UI 만들기](TODO) |
| https://blog.imprun.dev/92 | [Ory Kratos 로그인 플로우 Deep Dive](TODO) |
| https://blog.imprun.dev/91 | [Ory 스택 완전 가이드: 오픈소스 인증/인가 시스템](TODO) |
| https://blog.imprun.dev/90 | [Kubernetes Gateway API 설계 철학: Ingress를 넘어서](TODO) |
| https://blog.imprun.dev/89 | [Kubernetes Operator 패턴과 Reconciliation: 선언적 인프라의 핵심](TODO) |
| https://blog.imprun.dev/88 | [gRPC 완전 가이드: REST와의 비교부터 실전 활용까지](TODO) |
| https://blog.imprun.dev/87 | [API Gateway 입문 가이드: 마이크로서비스의 관문](TODO) |
| https://blog.imprun.dev/86 | [Airflow vs Prefect vs Dagster vs Temporal: 빌링 SaaS를 위한 워크플로우 오케스트레이션 비교](TODO) |
| https://blog.imprun.dev/85 | [Google Stitch: 피그마를 몰라도 AI로 UI 디자인하기](2025/12/11/google-stitch-ai-ui-design-tool-guide.md) |
| https://blog.imprun.dev/84 | [Claude Code Slack 연동: 팀 대화에서 바로 코딩 작업 위임하기](2025/12/11/claude-code-slack-integration-guide.md) |
| https://blog.imprun.dev/83 | [Claude Code 베스트 프랙티스: Anthropic 공식 가이드 정리](2025/12/11/claude-code-best-practices-official.md) |
| https://blog.imprun.dev/82 | [CLAUDE.md 공식 가이드 정리: 프로젝트 컨텍스트 자동화](2025/12/11/claude-md-guide-official.md) |
| https://blog.imprun.dev/81 | [Claude Code 사용 리포트: 27일간 12억 토큰의 기록](2025/12/11/claude-code-usage-report-nov-dec-2025.md) |
| https://blog.imprun.dev/80 | [ImpRun 인증/인가 아키텍처: Ory 스택 통합 구현 가이드](2025/12/06/ory-stack-integration-imprun-architecture.md) |
| https://blog.imprun.dev/79 | [Ory Oathkeeper를 활용한 Zero Trust IAP 구현 가이드](2025/12/06/ory-oathkeeper-zero-trust-iap-guide.md) |
| https://blog.imprun.dev/78 | [Ory Keto를 활용한 ReBAC 기반 권한 관리 시스템 구축](2025/12/06/ory-keto-rebac-authorization-guide.md) |
| https://blog.imprun.dev/77 | [Ory Kratos를 활용한 사용자 인증 시스템 구축: ImpRun 프로젝트 적용 사례](2025/12/06/ory-kratos-identity-management-practical-guide.md) |
| https://blog.imprun.dev/76 | [Ory Hydra OAuth2/OIDC 구현 가이드: Go 백엔드에서의 실제 활용](2025/12/06/ory-hydra-oauth2-implementation-guide.md) |
| https://blog.imprun.dev/75 | [Gemini 3.0 Pro + Antigravity 실사용 후기: Claude Code 사용자의 답답한 경험](2025/12/06/gemini-3-pro-antigravity-review.md) |
| https://blog.imprun.dev/74 | [Go 개발 생산성 향상을 위한 Air Live Reload 도입 가이드](2025/12/06/golang-air-live-reload-development-guide.md) |
| https://blog.imprun.dev/73 | [Claude Opus 4.5 vs Gemini 3.0 Pro vs Gemini 2.5 vs GPT-5.1: 백엔드 설계문서 비교](2025/11/26/ai-backend-plan-comparison.md) |
| https://blog.imprun.dev/72 | [Claude Opus 4.5 vs Gemini 3.0 Pro vs Gemini 2.5 Pro vs GPT-5.1 Codex Max: 프론트엔드 설계문서 비교](2025/11/26/ai-frontend-plan-comparison.md) |
| https://blog.imprun.dev/71 | [Claude Code 실무 개발 워크플로우: EPIC부터 일일 개발, 주간보고까지](2025/11/25/claude-code-power-user-workflow.md) |
| https://blog.imprun.dev/70 | [Feature-Sliced Design: 프론트엔드 아키텍처의 표준화된 접근법](2025/11/24/feature-sliced-design-frontend-architecture.md) |
| https://blog.imprun.dev/69 | [Claude, Gemini, Codex에서 AGENTS.md 설정하기: AI 에이전트 통합 가이드](2025/11/24/ai-agents-md-configuration-guide.md) |
| https://blog.imprun.dev/68 | [AI Agent를 위한 Frontend 개발 가이드: AGENTS.md로 Next.js + shadcn/ui 프로젝트 구조 설계](2025/11/24/frontend-ai-agent-development-guide.md) |
| https://blog.imprun.dev/67 | [Claude, Codex, Gemini가 본 API Gateway 콘솔 메뉴 구조: AI 모델별 UX 리뷰 비교](2025/11/24/ai-models-review-comparison-api-gateway-menu.md) |
| https://blog.imprun.dev/66 | [Kubernetes 환경에서 Keycloak 커스텀 로그인 테마 배포하기](2025/11/23/keycloak-custom-theme.md) |
| https://blog.imprun.dev/65 | [Kubernetes Ephemeral Storage 문제 해결 가이드](2025/11/18/kubernetes-ephemeral-storage-troubleshooting-guide.md) |
| https://blog.imprun.dev/64 | [Kubernetes Ephemeral Storage 부족으로 인한 MongoDB Pod Eviction 트러블슈팅](2025/11/18/mongodb-eviction-kubernetes-storage-crisis.md) |
| https://blog.imprun.dev/63 | [Kubernetes 운영 효율화: kubectl 별칭과 스크립트 활용법](2025/11/11/kubectl-aliases-and-scripts-for-kubernetes.md) |
| https://blog.imprun.dev/62 | [Frontend 컴포넌트 배치 완전 정복: Flexbox부터 Grid까지](2025/11/08/frontend-component-layout-mastery.md) |
| https://blog.imprun.dev/61 | [스크롤바로 인한 레이아웃 Shift 완벽 해결 가이드: scrollbar-gutter를 활용한 크로스 브라우저 대응](2025/11/08/scrollbar-layout-shift-prevention.md) |
| https://blog.imprun.dev/60 | [NestJS + React 표준 응답과 JWT 인증 완벽 가이드: ResponseUtil, Axios, Zustand](2025/11/05/nestjs-react-reusable-boilerplate.md) |
| https://blog.imprun.dev/59 | [Environment-Agnostic Architecture 구현 가이드: 실전 개발자를 위한 체크리스트](2025/11/03/environment-agnostic-implementation-guide.md) |
| https://blog.imprun.dev/58 | [Environment-Agnostic Architecture: Frontend와 Backend의 환경 분리 패턴](2025/11/03/environment-agnostic-architecture-pattern.md) |
| https://blog.imprun.dev/57 | [CLAUDE.md 최적화 여정: AI가 패턴을 무시하는 이유와 해결책](2025/11/03/claude-md-optimization-journey.md) |
| https://blog.imprun.dev/55 | [MongoDB Aggregation Pipeline로 N+1 문제 해결하기: $lookup과 $facet 활용](2025/11/02/mongodb-aggregation-pipeline-n-plus-1.md) |
| https://blog.imprun.dev/54 | [MongoDB 인덱스 생성 베스트 프랙티스: 수동 vs 자동, 그리고 Hybrid 접근](2025/11/02/mongodb-index-creation-best-practices.md) |
| https://blog.imprun.dev/53 | [MongoDB 연결 타임아웃 50% 해결기: Connection Pool 분리가 부른 나비효과](2025/11/02/mongodb-connection-pool-timeout-debugging.md) |
| https://blog.imprun.dev/52 | [Saga Pattern 소개: 언제 사용하고, 언제 피해야 하나?](2025/11/02/saga-pattern-when-not-to-use.md) |
| https://blog.imprun.dev/51 | [API Gateway 생성 실패 대응: Timeout과 Graceful Degradation 패턴](2025/11/02/timeout-graceful-degradation-pattern.md) |
| https://blog.imprun.dev/50 | [imprun Platform 아키텍처: API 개발부터 AI 통합까지](2025/11/02/imprun-platform-architecture-overview.md) |
| https://blog.imprun.dev/49 | [API Platform의 Consumer 인증 설계: Application-Grant 아키텍처](2025/11/01/application-grant-architecture.md) |
| https://blog.imprun.dev/48 | [API Gateway의 Consumer: 인증의 시작점](2025/11/01/api-gateway-consumer-explained.md) |
| https://blog.imprun.dev/47 | [분산 환경에서 Optimistic Lock으로 동시성 제어하기](2025/11/01/optimistic-lock-concurrency-control.md) |
| https://blog.imprun.dev/46 | [State Machine 패턴으로 Kubernetes 리소스 생명주기 관리하기](2025/11/01/state-machine-resource-management.md) |
| https://blog.imprun.dev/45 | [NestJS UnknownDependenciesException 완벽 해결 가이드](2025/11/01/nestjs-unknown-dependencies-exception-guide.md) |
| https://blog.imprun.dev/44 | [Kubernetes ImagePullPolicy 완벽 가이드: 개발과 운영 환경의 모범 사례](2025/10/31/kubernetes-imagepullpolicy-best-practices.md) |
| https://blog.imprun.dev/43 | [imprun의 진화: Serverless에서 API Gateway Platform으로](2025/10/30/from-serverless-to-api-gateway.md) |
| https://blog.imprun.dev/42 | [Claude AI와 함께하는 프론트엔드 개발: imprun.dev의 CLAUDE.md 가이드 공개](2025/10/27/frontend-architecture-guide-with-claude-ai.md) |
| https://blog.imprun.dev/41 | [Next.js를 버리고 순수 React로 돌아온 이유: 실무 관점의 프레임워크 선택 여정](2025/10/27/nextjs-to-react-migration-journey.md) |
| https://blog.imprun.dev/40 | [APISIX Ingress Controller 2.0: CRD 선택 가이드](2025/10/28/apisix-ingress-2.0-crd-guide.md) |
| https://blog.imprun.dev/39 | [Cilium 환경에서 API Gateway 배포 시 hostNetwork가 필요한 이유](2025/10/28/cilium-kubernetes-gateway-hostnetwork-architecture.md) |
| https://blog.imprun.dev/38 | [Kong에서 APISIX로의 험난한 여정: Cilium 환경에서의 시행착오](2025/10/28/kong-to-apisix-migration-journey.md) |
| https://blog.imprun.dev/37 | [Apache APISIX로 멀티 테넌트 서버리스 플랫폼 설계하기: 3계층 아키텍처 구현 노하우](2025/10/29/apisix-multi-tenant-serverless-architecture.md) |
| https://blog.imprun.dev/36 | [서버리스 플랫폼의 Stage 아키텍처 설계: dev → staging → prod 환경 분리 전략](2025/10/29/staging-environment-architecture-design.md) |
| https://blog.imprun.dev/35 | [Monaco Editor "TextModel got disposed" 에러 완벽 해결 가이드](2025/10/27/monaco-editor-textmodel-dispose-error-fix.md) |
| https://blog.imprun.dev/34 | [Kubernetes Gateway API 실전 가이드: Kong Ingress에서 표준 API로 전환하기](2025/10/27/kubernetes-gateway-api-migration-guide.md) |
| https://blog.imprun.dev/32 | [Claude AI와 함께하는 프론트엔드 개발 (원본)](2025/10/27/frontend-architecture-guide-with-claude-ai.md) |
| https://blog.imprun.dev/31 | [Next.js를 버리고 순수 React로 돌아온 이유 (원본)](2025/10/27/nextjs-to-react-migration-journey.md) |
| https://blog.imprun.dev/30 | [Sequential Thinking MCP: AI의 구조화된 사고 프로세스](2025/10/26/sequential-thinking-mcp-guide.md) |
| https://blog.imprun.dev/29 | [Serena MCP: AI 코딩 어시스턴트를 위한 시맨틱 코드 분석 도구](2025/10/26/serena-mcp-semantic-code-analysis-guide.md) |
| https://blog.imprun.dev/28 | [Cilium devices 파라미터 완벽 가이드: Tailscale 환경에서의 핵심](2025/10/24/cilium-devices-parameter-guide.md) |
| https://blog.imprun.dev/27 | [Kubernetes Pod Security Standards: nginx-unprivileged로 보안 강화하기](2025/10/26/nginx-unprivileged-pod-security-standards.md) |
| https://blog.imprun.dev/26 | [MongoDB ReplicaSet 완벽 가이드: 읽기/쓰기 라우팅부터 트랜잭션, 샤딩까지](2025/10/26/mongodb-replicaset-readPreference-sharding-guide.md) |
| https://blog.imprun.dev/25 | [5단계: 실전 경험과 아키텍처 결정 배경](2025/10/25/05-lessons-learned.md) |
| https://blog.imprun.dev/24 | [4단계: 네트워킹 심화 이해](2025/10/25/04-deep-dive-networking.md) |
| https://blog.imprun.dev/23 | [3단계: Kubernetes + Cilium 클러스터 구축](2025/10/25/03-setup-kubernetes-cilium.md) |
| https://blog.imprun.dev/22 | [2단계: Tailscale 메시 네트워크 구성](2025/10/25/02-setup-tailscale-network.md) |
| https://blog.imprun.dev/21 | [2단계 (선택): Headscale 자체 호스팅 서버 구축](2025/10/25/02a-setup-headscale-server.md) |
| https://blog.imprun.dev/20 | [1단계: Oracle Cloud 준비 및 설정](2025/10/25/01-preparation-oracle-cloud.md) |
| https://blog.imprun.dev/16 | [Kubernetes Namespace 삭제 전쟁: 18시간의 Terminating과의 싸움](2025/10/22/troubleshooting-kubernetes-namespace-deletion.md) |
| https://blog.imprun.dev/15 | [Kubernetes 민감정보 관리 완벽 가이드: Secret, 암호화, 그리고 실전 전략](2025/10/22/kubernetes-sensitive-data-management.md) |
| https://blog.imprun.dev/14 | [Next.js SSR 환경에서 API URL 환경변수 관리 전략](2025/10/22/nextjs-api-url-configuration-strategies.md) |
| https://blog.imprun.dev/13 | [CI/CD 파이프라인 구축: GitHub Actions로 완전 자동화하기](2025/10/21/cicd-pipeline-with-github-actions.md) |
| https://blog.imprun.dev/12 | [Kubernetes 리소스 최적화: ARM64 환경에서 효율적으로 운영하기](2025/10/21/kubernetes-resource-optimization.md) |
| https://blog.imprun.dev/11 | [Helm 차트 관리 Best Practices: Umbrella Chart부터 Secret 관리까지](2025/10/21/helm-chart-best-practices.md) |
| https://blog.imprun.dev/10 | [APISIX Consumer 인증 아키텍처 완벽 이해하기: X-Consumer-Username 헤더와 사용자 추적](2025/11/01/apisix-consumer-authentication-tracking.md) |
| https://blog.imprun.dev/9 | [CloudFunction 버전 관리와 릴리즈 전략: Hot Reload 환경에서의 안전한 배포](2025/10/31/cloudfunction-versioning-and-release-ko.md) |
| https://blog.imprun.dev/8 | [Environment-Agnostic Architecture: Frontend와 Backend의 환경 분리 패턴](2025/11/02/environment-agnostic-architecture-pattern.md) |
| https://blog.imprun.dev/7 | [Kubernetes 이미지 업데이트 전략: ImagePullPolicy 완벽 가이드](2025/10/21/kubernetes-image-update-strategy.md) |
| https://blog.imprun.dev/6 | [CloudFunction 릴리즈 노트 아키텍처: 배포 이력 추적과 관리](2025/10/20/cloudfunction-release-notes-architecture.md) |
| https://blog.imprun.dev/5 | [무료 Kubernetes 클러스터 완벽 가이드: Oracle Cloud + Tailscale로 글로벌 멀티 리전 구축하기](2025/10/25/oracle-cloud-kubernetes-guide-overview.md) |
| https://blog.imprun.dev/4 | [Cloudflare + Let's Encrypt 무료 SSL 인증서 자동화 가이드](2025/10/20/cloudflare-letsencrypt-ssl-automation.md) |
| https://blog.imprun.dev/3 | [Kubernetes Ingress TLS 설정 완벽 가이드: cert-manager + Cloudflare 연동](2025/10/20/kubernetes-ingress-tls-cert-manager-cloudflare.md) |
| https://blog.imprun.dev/2 | [Oracle Cloud + k3s로 시작하는 완전 무료 쿠버네티스 클러스터](2025/10/19/oracle-cloud-k3s-free-kubernetes.md) |

