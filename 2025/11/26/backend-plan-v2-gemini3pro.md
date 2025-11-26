# Backend Implementation Plan v2 (Gemini 3 Pro)

**Version**: 1.0
**Date**: 2025-11-27
**Author**: Gemini 3 Pro
**Based on**: [Architecture v2](architecture.md), [PRD v2](prd.md), [Backend Spec v2](backend-spec.md), [Data Model v2](data-model.md)

---

## 1. Overview

This document outlines the implementation plan for the **Imp-Gateway v2 Backend (Imprun Server)**. The backend serves as the Control Plane, managing API definitions, products, and orchestrating deployments to multiple clusters via Agents.

The project will be implemented in the `services/imprun-server/` directory (or similar), following **Clean Architecture** principles.

### 1.1 Tech Stack

- **Language**: Go 1.21+
- **Framework**: Gin (HTTP), gRPC (Agent Communication)
- **Database**: PostgreSQL 15+
- **ORM**: GORM
- **Authentication**: Keycloak (OIDC)
- **Documentation**: Swagger/OpenAPI (via swaggo/swag)
- **Testing**: Testify, Mockery

---

## 2. Architecture (Clean Architecture)

We will adopt a layered architecture to separate concerns and ensure testability.

### 2.1 Layer Structure

```
services/imprun-server/
├── cmd/
│   └── server/           # Main entrypoint
├── internal/
│   ├── api/              # Interface Layer (HTTP Handlers)
│   │   ├── handler/      # Gin Handlers
│   │   ├── middleware/   # Auth, CORS, Logging
│   │   └── router/       # Route definitions
│   │
│   ├── grpc/             # Interface Layer (gRPC Server)
│   │   └── server/       # Agent gRPC implementation
│   │
│   ├── core/             # Business Logic Layer
│   │   ├── domain/       # Domain Models (Structs)
│   │   ├── port/         # Interfaces (Input/Output Ports)
│   │   └── service/      # Use Cases / Business Logic
│   │
│   └── infra/            # Infrastructure Layer
│       ├── persistence/  # Database Repositories (GORM)
│       ├── keycloak/     # Auth Provider Client
│       └── logger/       # Logging implementation
│
├── pkg/                  # Shared packages (Utils, Errors)
└── config/               # Configuration loading
```

### 2.2 Dependency Injection
- We will use `google/wire` or manual dependency injection in `cmd/server/main.go` to wire up Repositories -> Services -> Handlers.

---

## 3. Database Schema & Migration

We need to update the database schema to support v2 requirements, specifically the separation of Logic and Physical layers.

### 3.1 Key Changes (v1 -> v2)
1.  **`api_services`**: Remove `gateway_id`.
2.  **`product_publishes`**: Add `cluster_id` (UUID, Not Null, FK to clusters).
3.  **`clusters`**: New table for physical clusters.
4.  **`agents`**: New table for agent registration.

### 3.2 Migration Strategy
- Use **GORM AutoMigrate** for development.
- For production, use a migration tool like `golang-migrate/migrate` or `gormigrate`.
- **Migration Script**:
    ```go
    // Migration Step 1: Add ClusterID to ProductPublish
    type ProductPublishV2 struct {
        ClusterID string `gorm:"type:uuid;default:null"`
    }
    // Migration Step 2: Backfill default cluster for existing records
    // Migration Step 3: Make ClusterID NOT NULL
    ```

---

## 4. API Implementation Plan

### 4.1 REST API (HTTP)

We will group APIs by user persona as defined in the spec.

#### 4.1.1 Provider APIs (`/v1/provider`)
-   **API Service**: `POST`, `GET`, `PUT`, `DELETE` (No `gateway_id` in payload).
-   **Gateway Template**: `POST`, `GET` (Just a template).
-   **Product**: `POST`, `GET` (Catalog).
-   **ProductPublish**: `POST` (Must include `cluster_id`), `GET` (Filter by `cluster_id`).

#### 4.1.2 Operator APIs (`/v1/operator`)
-   **Cluster**: `POST` (Register Cluster), `GET`.
-   **Agent**: `POST` (Generate Token), `GET` (List Agents).

#### 4.1.3 Customer APIs (`/v1/customer`)
-   **Subscription**: `POST` (Subscribe), `GET`.
-   **Credential**: `POST` (Generate API Key).

### 4.2 gRPC API (Agent Communication)

#### 4.2.1 Service Definition (`agent.proto`)
-   `ConnectAgent`: Bidirectional stream.
    -   **Agent -> Server**: `ConnectRequest` (Handshake), `Heartbeat`, `DeployResult`.
    -   **Server -> Agent**: `ConnectResponse`, `HeartbeatAck`, `DeployCommand`.

#### 4.2.2 Server Implementation
-   **Connection Management**: Maintain a map of active streams `map[AgentID]Stream`.
-   **Authentication**: Validate JWT in the first `ConnectRequest`.
-   **Routing**: When a `ProductPublish` is created/updated, look up the `ClusterID`, find the connected Agent for that cluster, and send `DeployCommand`.

---

## 5. Authentication & Authorization

### 5.1 Keycloak Integration
-   **Middleware**: `AuthMiddleware` extracts Bearer token and validates against Keycloak public key (JWKS).
-   **Context**: Inject `UserID`, `TenantIDs`, `Roles` into Gin Context.

### 5.2 RBAC (Role-Based Access Control)
-   Implement a helper `HasPermission(ctx, resource, action)` to check roles.
-   **Roles**: `system-admin`, `org-admin`, `api-developer`, `consumer`.

---

## 6. Implementation Roadmap

### Phase 1: Foundation & v2 Schema (Days 1-2)
- [ ] Initialize Go module and project structure.
- [ ] Setup GORM and define v2 Domain Models (`Cluster`, `Agent`, updated `ProductPublish`).
- [ ] Implement Database Migrations.
- [ ] Setup Keycloak Auth Middleware.

### Phase 2: Operator & Physical Layer (Days 3-4)
- [ ] Implement `Cluster` CRUD APIs.
- [ ] Implement `Agent` Registration APIs (Token generation).
- [ ] Implement gRPC Server foundation (`ConnectAgent`).

### Phase 3: Provider & Logical Layer (Days 5-7)
- [ ] Implement `APIService` CRUD (Remove `gateway_id` dependency).
- [ ] Implement `Gateway` Template CRUD.
- [ ] Implement `Product` & `ProductPublish` APIs.
- [ ] **Core Logic**: When `ProductPublish` is saved, trigger `AgentService.NotifyAgent(clusterID)`.

### Phase 4: Agent Coordination (Days 8-10)
- [ ] Implement `DeployCommand` dispatch logic.
- [ ] Implement `DeployResult` handling (Update status in DB).
- [ ] Implement Heartbeat & Disconnect handling.

---

## 7. Testing Strategy

### 7.1 Unit Tests
-   Use `testify` for assertions.
-   Mock Repositories using `mockery` to test Service logic in isolation.
-   Example: Test `ProductPublish` service ensures `ClusterID` is valid.

### 7.2 Integration Tests
-   Spin up a test DB (Docker).
-   Test API endpoints using `httptest`.
-   Verify DB persistence and constraints.

### 7.3 End-to-End (E2E) Tests
-   Simulate a full flow: Register Cluster -> Connect Mock Agent -> Create API -> Publish Product -> Verify Agent receives Command.

---

## 8. API Specification (Swagger)

We will use `swaggo/swag` annotations to generate OpenAPI documentation.

```go
// @Summary Create Product Publish
// @Description Publish a product to a specific cluster
// @Tags provider
// @Accept json
// @Produce json
// @Param body body CreateProductPublishRequest true "Publish Request"
// @Success 201 {object} ProductPublishResponse
// @Router /v1/provider/product-publishes [post]
func (h *ProductHandler) CreatePublish(c *gin.Context) { ... }
```
