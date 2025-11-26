# Imp-Gateway PRD v2

**제품명**: Imp-Gateway
**버전**: 2.0
**최종 수정**: 2025-11-26
**이전 버전**: [archived/v1/prd.md](archived/v1/prd.md)

---

## 1. 제품 비전

### 1.1 한 줄 요약

> **B2B API Gateway 플랫폼**: API를 정의하고, 제품화하고, 멀티 클러스터에 배포하여 고객에게 판매한다.

### 1.2 핵심 가치

| 가치 | 설명 |
|------|------|
| **논리/물리 분리** | API 정의(논리)와 배포 위치(물리)를 분리하여 재사용성 극대화 |
| **멀티 클러스터** | 같은 API를 여러 클러스터/리전에 동시 배포 |
| **Agent 기반 배포** | 중앙 서버가 설정을 관리하고, Agent가 실제 CRD를 주입 |
| **B2B 제품화** | API를 상품화하여 구독/과금 체계로 판매 |

### 1.3 데이터플레인

- **주력**: Envoy Gateway (Kubernetes Gateway API)
- **확장 예정**: Nginx Ingress, Kong, Traefik, APISIX

---

## 2. 대상 사용자 (Personas)

### 2.1 Provider (API 제공자)

| 역할 | 책임 |
|------|------|
| **API Developer** | API Service 정의 (Route, Backend, Policy) |
| **Business Manager** | Product 생성, Plan 설정, 가격 책정 |
| **Gateway Owner** | Gateway 템플릿 관리, 배포 승인 |

### 2.2 Customer (API 소비자)

| 역할 | 책임 |
|------|------|
| **Customer Admin** | 구독 관리, 팀원 초대 |
| **API Consumer** | Credential 발급, API 호출 |

### 2.3 Operator (SRE/운영자)

| 역할 | 책임 |
|------|------|
| **SRE Operator** | Cluster/Agent 관리, Fleet 모니터링 |
| **System Admin** | Tenant 관리, 전역 정책 설정 |

---

## 3. 핵심 개념 (Domain Model)

### 3.1 계층 구조

```
┌─────────────────────────────────────────────────────────────────┐
│                      논리 계층 (Logical)                         │
│            재사용 가능, 인프라 독립, 청사진                        │
├─────────────────────────────────────────────────────────────────┤
│ API Service    - API의 청사진 (Route, Backend, Policy 포함)     │
│ Product        - 카탈로그 메타데이터 (이름, 버전, 설명)           │
│ Gateway        - 게이트웨이 설정 템플릿 (Class, Listeners)       │
│ Plan           - 요금/쿼터/기능 묶음                             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      배포 계층 (Deployment)                      │
│            논리 + 물리 연결점, "무엇을 어디에 어떻게"              │
├─────────────────────────────────────────────────────────────────┤
│ ProductPublish - 배포 명세                                       │
│   ├── ProductID     (어떤 제품)                                  │
│   ├── APIServices[] (어떤 서비스들)                              │
│   ├── GatewayID     (어떤 설정)                                  │
│   ├── ClusterID     (어디에)                                     │
│   ├── Environment   (어떤 환경)                                  │
│   └── AuthMode      (어떤 인증)                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      물리 계층 (Physical)                        │
│            실제 인프라, 위치 종속                                 │
├─────────────────────────────────────────────────────────────────┤
│ Cluster        - K8s 클러스터 (리전, 프로바이더)                 │
│ Agent          - 클러스터에 설치된 CRD 주입기                    │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 엔티티 정의

#### 논리 계층

| 엔티티 | 설명 | 핵심 필드 |
|--------|------|----------|
| **Tenant** | 조직 (최상위 격리 단위) | id, name, slug |
| **API Service** | API의 청사진 | tenant_id, name, version, status |
| **Route** | 라우팅 규칙 | api_service_id, path, methods, backend_id |
| **Backend** | 업스트림 정의 | api_service_id, endpoints, lb_policy |
| **Policy** | 정책 | api_service_id, type, config |
| **Product** | 카탈로그 | tenant_id, name, version, categories |
| **Gateway** | 게이트웨이 템플릿 | tenant_id, name, gateway_class, listeners |
| **Plan** | 요금/쿼터 | product_id, name, rate_limit, quota |

#### 배포 계층

| 엔티티 | 설명 | 핵심 필드 |
|--------|------|----------|
| **ProductPublish** | 배포 명세 | product_id, api_services[], gateway_id, **cluster_id**, environment, auth_mode, status |

#### 물리 계층

| 엔티티 | 설명 | 핵심 필드 |
|--------|------|----------|
| **Cluster** | K8s 클러스터 | name, region, cloud_provider |
| **Agent** | CRD 주입기 | cluster_id, status, version |

#### 구독/소비 계층

| 엔티티 | 설명 | 핵심 필드 |
|--------|------|----------|
| **Customer** | B2B 고객사 | tenant_id, name |
| **Subscription** | 구독 | customer_id, product_publish_id, plan_id, status |
| **ClientApp** | 고객 애플리케이션 | customer_id, name |
| **Consumer** | 소비 주체 | subscription_id, client_app_id |
| **Credential** | 자격증명 | consumer_id, type (api_key/oauth), secret |

---

## 4. 핵심 사용자 흐름

### 4.1 Provider: API 정의 → 제품화 → 배포

```
1. API Service 생성
   ├── Route: /payments, /refunds
   ├── Backend: payment-svc.default.svc:8080
   └── Policy: RateLimit 1000/min

2. Gateway 생성 (템플릿)
   ├── GatewayClass: envoy-gateway
   └── Listeners: HTTPS:443

3. Product 생성 (카탈로그)
   ├── Name: "Payment API"
   ├── Version: "v1.0"
   └── Categories: ["payments", "fintech"]

4. ProductPublish 생성 (배포)
   ├── ProductID: product-123
   ├── APIServices: [payment-service]
   ├── GatewayID: gateway-456
   ├── ClusterID: cluster-kr-seoul  ← 핵심: 어디에 배포할지 결정
   ├── Environment: prod
   └── AuthMode: oauth2

5. Agent가 배포
   ├── ProductPublish 폴링 (ClusterID 매칭)
   ├── Gateway 템플릿 참조
   ├── API Service 정의 참조
   └── CRD 생성/적용
```

### 4.2 Customer: 구독 → Credential 발급 → API 호출

```
1. Marketplace에서 Product 탐색
2. Plan 선택 후 구독 신청
3. Provider 승인 (auto_approve 설정에 따라)
4. Client App 생성
5. Consumer 생성 (구독×앱×환경 조합)
6. Credential 발급 (API Key 또는 OAuth Client)
7. API 호출
```

### 4.3 Operator: 클러스터/Agent 관리

```
1. Cluster 등록
   └── region, cloud_provider 정보

2. Agent 등록
   ├── Registration Token 발급
   └── Agent 설치 (Helm Chart)

3. Agent 연결
   ├── JWT 토큰으로 인증
   └── gRPC 스트림 연결

4. 배포 모니터링
   ├── Agent 상태 확인
   ├── 배포 성공/실패 추적
   └── Drift 감지/교정
```

---

## 5. 기능 요구사항

### 5.1 Epic A: 테넌트/조직 관리 (P0)

| ID | 기능 | 설명 |
|----|------|------|
| A1 | Tenant CRUD | 조직 생성/조회/수정/삭제 |
| A2 | User 초대 | 조직에 사용자 초대 |
| A3 | Role 관리 | 역할 할당 (org-admin, api-developer 등) |

### 5.2 Epic B: API Service 관리 (P0)

| ID | 기능 | 설명 |
|----|------|------|
| B1 | API Service CRUD | API 서비스 생성/조회/수정/삭제 |
| B2 | Route 관리 | 라우팅 규칙 설정 |
| B3 | Backend 관리 | 업스트림 서버 설정 |
| B4 | Policy 관리 | RateLimit, CORS, Auth 정책 |

**중요**: API Service는 **Gateway에 종속되지 않음** (독립적 청사진)

### 5.3 Epic C: Gateway 관리 (P0)

| ID | 기능 | 설명 |
|----|------|------|
| C1 | Gateway CRUD | 게이트웨이 템플릿 생성/조회/수정/삭제 |
| C2 | GatewayClass 선택 | envoy-gateway, nginx, kong 등 |
| C3 | Listener 설정 | 포트, 프로토콜, TLS 설정 |

**중요**: Gateway는 **설정 템플릿**이며, 실제 배포는 ProductPublish가 결정

### 5.4 Epic D: Product/Plan 관리 (P0)

| ID | 기능 | 설명 |
|----|------|------|
| D1 | Product CRUD | 카탈로그 생성/조회/수정/삭제 |
| D2 | Plan CRUD | 요금/쿼터 설정 |
| D3 | ProductPublish | 배포 명세 생성 |
| D4 | 환경 관리 | dev/staging/prod 환경별 배포 |

**ProductPublish 필수 필드**:
- `product_id` - 어떤 제품
- `api_services[]` - 어떤 서비스들
- `gateway_id` - 어떤 설정 (템플릿)
- `cluster_id` - **어디에 배포** (핵심!)
- `environment` - 어떤 환경
- `auth_mode` - 인증 방식

### 5.5 Epic E: Cluster/Agent 관리 (P0)

| ID | 기능 | 설명 |
|----|------|------|
| E1 | Cluster CRUD | 클러스터 등록/관리 |
| E2 | Agent 등록 | Registration Token 발급 |
| E3 | Agent 연결 | gRPC 스트림, JWT 인증 |
| E4 | 배포 실행 | ProductPublish → CRD 변환/적용 |
| E5 | 상태 리포트 | 배포 성공/실패, Drift 감지 |

### 5.6 Epic F: 구독/Credential 관리 (P1)

| ID | 기능 | 설명 |
|----|------|------|
| F1 | Customer CRUD | B2B 고객사 관리 |
| F2 | Subscription | 구독 신청/승인/관리 |
| F3 | ClientApp | 고객 애플리케이션 관리 |
| F4 | Consumer | 소비 주체 관리 |
| F5 | Credential | API Key/OAuth Client 발급/회전/폐기 |

### 5.7 Epic G: Marketplace (P1)

| ID | 기능 | 설명 |
|----|------|------|
| G1 | Product 목록 | 공개된 Product 탐색 |
| G2 | Product 상세 | 문서, Plan, API 스펙 확인 |
| G3 | 구독 신청 | Plan 선택 후 구독 요청 |

---

## 6. 비기능 요구사항

### 6.1 성능

| 항목 | 목표 |
|------|------|
| API 응답 시간 | p99 < 200ms |
| Agent 동기화 | < 30초 (변경 감지 후) |
| CRD 적용 | < 10초 |

### 6.2 확장성

| 항목 | 목표 (Phase 2) |
|------|----------------|
| Tenant 수 | 1,000+ |
| Cluster 수 | 100+ |
| API Service 수 | 10,000+ |
| ProductPublish 수 | 50,000+ |

### 6.3 가용성

| 항목 | 목표 |
|------|------|
| Control Plane | 99.9% |
| Agent 연결 | 자동 재연결, 백오프 |
| CRD Drift | 자동 교정 |

---

## 7. 기술 스택

### 7.1 Backend

```
언어: Go 1.21+
프레임워크: Gin
ORM: GORM
데이터베이스: PostgreSQL 15+
인증: Keycloak (OIDC)
캐시: Redis (선택)
```

### 7.2 Frontend

```
프레임워크: Next.js 15 (App Router)
UI: shadcn/ui, Tailwind CSS
상태관리: TanStack Query
인증: NextAuth.js
```

### 7.3 Agent

```
언어: Go 1.21+
통신: gRPC (bidirectional stream)
K8s Client: client-go
CRD 적용: Server-Side Apply (SSA)
```

### 7.4 Infrastructure

```
데이터플레인: Envoy Gateway (주력)
인증 서버: Keycloak
관측성: VictoriaMetrics, Grafana
컨테이너: Docker, Kubernetes
```

---

## 8. 로드맵

### Phase 1: MVP (현재)

- [x] Tenant/User 관리
- [x] API Service CRUD (독립)
- [x] Gateway CRUD (템플릿)
- [x] Product/ProductPublish
- [x] Cluster/Agent 등록
- [x] Agent 연결 (gRPC + JWT)
- [ ] ProductPublish에 ClusterID 추가
- [ ] Agent 배포 로직 구현

### Phase 2: 멀티 클러스터 Fleet

- [ ] 멀티 클러스터 배포
- [ ] Cluster 선택 UI
- [ ] 리전별 배포 현황 대시보드
- [ ] Fleet 모니터링

### Phase 3: 구독/결제

- [ ] Customer/Subscription
- [ ] Credential 발급
- [ ] Rate Limiting (Consumer별)
- [ ] Usage Metering

### Phase 4: Marketplace

- [ ] Product 탐색
- [ ] 구독 신청 플로우
- [ ] API 문서 자동 생성
- [ ] 개발자 포털

---

## 9. 마이그레이션 (v1 → v2)

### 9.1 Breaking Changes

| 변경 | v1 | v2 |
|------|----|----|
| APIService.GatewayID | 필수 | **제거** |
| ProductPublish.ClusterID | 없음 | **필수** |
| Gateway 역할 | 배포 타겟 | 설정 템플릿 |

### 9.2 마이그레이션 가이드

1. **DB 스키마 변경**
   ```sql
   -- api_services 테이블에서 gateway_id 제거
   ALTER TABLE api_services DROP COLUMN gateway_id;

   -- product_publishes 테이블에 cluster_id 추가
   ALTER TABLE product_publishes ADD COLUMN cluster_id UUID REFERENCES clusters(id);
   ```

2. **기존 데이터 처리**
   - 기존 APIService의 gateway_id 값은 무시
   - 기존 ProductPublish에 기본 ClusterID 할당 필요

3. **API 변경**
   - `POST /v1/api-services`: gateway_id 파라미터 제거
   - `POST /v1/product-publishes`: cluster_id 파라미터 추가 (필수)

---

## 10. 참고 문서

- [Architecture v2](architecture.md)
- [Data Model v2](data-model.md)
- [Agent Spec v2](agent-spec.md)
- [Backend Spec v2](backend-spec.md)
- [Frontend Spec v2](frontend-spec.md)
- [이전 버전](archived/v1/)
