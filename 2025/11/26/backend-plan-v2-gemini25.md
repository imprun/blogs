# Imp-Gateway Backend Plan v2 (Gemini 2.5)

**버전**: 2.0
**대상**: `services/imprun-server` (Control Plane)
**기반 아키텍처**: `docs/architecture.md`, `docs/prd.md`
**참고 명세**: `docs/backend-spec.md`

---

## 1. 개요 및 목표

본 문서는 Imp-Gateway v2의 백엔드(Control Plane) 구현을 위한 상세 계획서입니다.
**Go/Gin** 프레임워크를 기반으로 하며, **논리/물리 계층 분리**와 **멀티 클러스터 지원**을 위한 아키텍처 변경을 핵심으로 합니다.

### 핵심 목표
1.  **논리적 API 정의 분리**: `APIService`에서 `GatewayID` 의존성을 제거하여 순수한 API 청사진으로 리팩토링합니다.
2.  **물리적 배포 타겟팅 구현**: `ProductPublish`에 `ClusterID`를 필수 필드로 추가하여 배포 위치를 명시합니다.
3.  **Agent 통신 고도화**: gRPC 스트리밍을 통해 Agent 연결 관리, 토큰 인증, 배포 명령 전달 시스템을 구축합니다.
4.  **페르소나별 API 격리**: Provider, Customer, Operator, Agent 전용 API 그룹을 명확히 분리하고 RBAC를 적용합니다.

---

## 2. 프로젝트 구조 (Clean Architecture)

`services/imprun-server` 내부에 도메인 중심의 Clean Architecture를 적용합니다.

```
services/imprun-server/
├── cmd/
│   └── server/             # main.go (진입점)
├── internal/
│   ├── config/             # 환경 설정 (env)
│   ├── db/                 # DB 연결 및 마이그레이션
│   ├── server/             # HTTP/gRPC 서버 설정
│   │
│   ├── domain/             # [Layer 1] 핵심 비즈니스 로직 (Model, Repository interface)
│   │   ├── cluster/        # Cluster, Agent 모델
│   │   ├── apiservice/     # APIService 모델
│   │   ├── product/        # Product, ProductPublish 모델
│   │   └── gateway/        # Gateway 템플릿 모델
│   │
│   ├── handler/            # [Layer 2] HTTP/gRPC 핸들러 (Controller)
│   │   ├── http/           # Gin 핸들러
│   │   │   ├── operator/
│   │   │   ├── provider/
│   │   │   └── customer/
│   │   └── grpc/           # gRPC 서버 구현
│   │
│   ├── service/            # [Layer 3] 비즈니스 로직 구현 (Usecase)
│   │   ├── cluster_service.go
│   │   └── ...
│   │
│   └── repository/         # [Layer 4] 데이터 접근 구현 (GORM)
│       ├── postgres/
│       └── redis/
│
└── pkg/                    # 공용 유틸리티
    ├── auth/               # JWT, Keycloak 미들웨어
    └── logger/             # 로깅 설정
```

---

## 3. 개발 단계 (Phases)

### Phase 1: 인프라 및 마이그레이션 (Week 1)

**목표**: v2 데이터 모델을 DB에 반영하고, 기본 서버 구조를 확립합니다.

1.  **DB 마이그레이션 스크립트 작성**
    *   `api_services`: `gateway_id` 컬럼 제거 (Nullable 처리 후 데이터 정리)
    *   `product_publishes`: `cluster_id` 컬럼 추가 (FK: `clusters.id`), `gateway_id`는 유지(템플릿 참조용)
    *   `clusters`: 신규 테이블 생성 (name, region, provider, capabilities)
    *   `agents`: 신규 테이블 생성 (cluster_id FK, status, token)
2.  **API 그룹 및 미들웨어 설정**
    *   Gin 라우터 그룹 분리 (`/v1/provider`, `/v1/operator` 등)
    *   Keycloak OIDC 미들웨어 및 RBAC 정책 적용
    *   에러 핸들링 및 로깅 미들웨어 표준화

### Phase 2: Operator 모듈 & Agent 관리 (Week 2)

**목표**: 물리적 인프라(클러스터/에이전트)를 관리하는 Operator 기능을 구현합니다.

1.  **Cluster CRUD**
    *   `POST /v1/operator/clusters`: 클러스터 등록
    *   `GET /v1/operator/clusters`: 클러스터 목록 (상태 포함)
2.  **Agent 등록 및 토큰 발급**
    *   `POST /v1/operator/clusters/{id}/registration-tokens`: Agent 설치용 토큰 생성
    *   `AgentService` (Internal): 토큰 검증 및 Agent 레코드 생성 로직
3.  **gRPC Agent Server 구현**
    *   `ConnectAgent` 스트림 RPC 구현
    *   Agent 인증 (JWT 발급/검증)
    *   Agent 연결 상태(Online/Offline)를 Redis 또는 메모리에서 관리

### Phase 3: Provider 모듈 - 논리 계층 (Week 3)

**목표**: 게이트웨이에 종속되지 않는 순수 API 정의 기능을 구현합니다.

1.  **APIService Refactoring**
    *   `gateway_id` 파라미터를 제거한 `CreateAPIService` 로직 구현
    *   `Route`, `Backend`, `Policy` 엔티티가 독립적으로 `APIService`에 종속되도록 조정
2.  **Gateway Template 관리**
    *   `Gateway` 엔티티를 "배포 타겟"에서 "설정 템플릿"으로 재정의
    *   `GatewayClass` (envoy, nginx 등) 및 `Listeners` 설정 저장

### Phase 4: Provider 모듈 - 배포 계층 (Week 4)

**목표**: 논리적 정의와 물리적 클러스터를 결합하여 실제 배포를 수행합니다.

1.  **ProductPublish 구현 (v2 핵심)**
    *   `POST /v1/provider/product-publishes`:
        *   Input: `product_id`, `cluster_id`(필수), `gateway_id`(템플릿), `api_services`
        *   Validation: 선택된 클러스터가 유효한지, API 서비스들이 존재하는지 확인
        *   Action: `ProductPublish` 레코드 생성 (Status: DRAFT)
2.  **배포 트리거 로직**
    *   Publish 생성 시, 해당 `cluster_id`에 연결된 Agent에게 gRPC 메시지(이벤트) 전송
    *   Agent가 없다면 큐잉(Queueing) 또는 Polling 대기 상태로 유지

### Phase 5: Consumer 모듈 & 마켓플레이스 (Week 5)

**목표**: API 소비자가 제품을 구독하고 사용할 수 있는 환경을 제공합니다.

1.  **Marketplace API**
    *   `GET /v1/public/products`: 공개된 제품 목록 조회
2.  **Subscription 관리**
    *   `POST /v1/customer/subscriptions`: 구독 신청
    *   자동/수동 승인 프로세스 구현
3.  **Credential 발급**
    *   `POST /v1/customer/credentials`: API Key 또는 OAuth Client 생성
    *   생성된 자격증명을 Agent에게 동기화하는 로직 필요 (보안 고려)

---

## 4. 데이터베이스 스키마 (PostgreSQL)

### 주요 테이블 변경 사항

```sql
-- v2: Clusters (New)
CREATE TABLE clusters (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    region VARCHAR(100) NOT NULL,
    cloud_provider VARCHAR(50),
    capabilities JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- v2: Agents (New)
CREATE TABLE agents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cluster_id UUID REFERENCES clusters(id),
    name VARCHAR(255),
    status VARCHAR(50) DEFAULT 'disconnected',
    last_heartbeat_at TIMESTAMPTZ,
    version VARCHAR(50)
);

-- v2: API Services (Alter)
ALTER TABLE api_services DROP COLUMN gateway_id;

-- v2: Product Publishes (Alter)
ALTER TABLE product_publishes ADD COLUMN cluster_id UUID REFERENCES clusters(id);
ALTER TABLE product_publishes ALTER COLUMN cluster_id SET NOT NULL; -- 마이그레이션 후 적용
```

---

## 5. Agent 통신 프로토콜 (gRPC)

### Proto 정의 (`services/imprun-server/proto/agent.proto`)

```protobuf
syntax = "proto3";

package agent.v1;

service AgentService {
  rpc Connect (stream AgentMessage) returns (stream ServerMessage);
}

message AgentMessage {
  oneof payload {
    ConnectRequest connect_request = 1;
    Heartbeat heartbeat = 2;
    DeployResult deploy_result = 3;
  }
}

message ServerMessage {
  oneof payload {
    ConnectResponse connect_response = 1;
    DeployCommand deploy_command = 2;
  }
}

message DeployCommand {
  string product_publish_id = 1;
  // Agent는 ID만 받고, 상세 정보는 REST API로 조회 (Payload 최소화)
}
```

---

## 6. 테스트 전략

1.  **Unit Test**: Service 레이어의 비즈니스 로직 검증 (Mock Repository 사용)
2.  **Integration Test**: HTTP 핸들러부터 DB까지의 통합 흐름 검증 (TestContainer 활용 권장)
3.  **E2E Test**: Agent Mock 서버를 띄워 실제 배포 명령 흐름 검증

---

## 7. 마이그레이션 가이드

1.  **백업**: DB 전체 백업 필수.
2.  **배포 순서**:
    1.  Backend v2 배포 (DB 마이그레이션 포함 - 하위 호환성 유지 주의)
    2.  Frontend v2 배포
    3.  Agent v2 배포 (클러스터별 순차 적용)
3.  **구버전 데이터 처리**: 기존 `gateway_id` 기반의 배포 정보는 `default` 클러스터를 생성하여 매핑하거나, 마이그레이션 툴을 제공하여 운영자가 직접 매핑하도록 유도.
