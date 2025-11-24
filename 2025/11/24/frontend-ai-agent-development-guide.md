# AI Agent를 위한 Frontend 개발 가이드: AGENTS.md로 Next.js + shadcn/ui 프로젝트 구조 설계

> **작성일**: 2025년 11월 24일
> **카테고리**: Frontend, AI, Architecture
> **키워드**: Next.js, React, AI Agent, AGENTS.md, CLAUDE.md, shadcn/ui, TanStack Query, Zustand

## 요약

AI 코딩 어시스턴트(Claude, Copilot 등)와 협업할 때 프로젝트 구조를 명확히 문서화하면 AI가 일관된 코드를 생성합니다. 이 글에서는 Next.js 16 + shadcn/ui 기반 프로젝트에서 AI Agent가 따라야 할 아키텍처 패턴, 네이밍 컨벤션, 데이터 흐름을 정의하는 방법을 공유합니다. 실제 B2B API Gateway 콘솔 프로젝트에서 사용 중인 가이드 문서입니다.

## 문제 상황

### 배경

AI 코딩 어시스턴트와 함께 프론트엔드를 개발할 때 다음 문제들이 발생합니다:

- AI가 매번 다른 파일 구조와 네이밍 컨벤션을 사용
- API 호출 로직이 컴포넌트, 훅, 서비스 등 여기저기 흩어짐
- 상태 관리 방식이 일관되지 않음 (useState, Zustand, Context 혼용)
- 새 기능 추가 시 어디에 코드를 배치해야 할지 AI가 판단하지 못함

### 환경 구성

- **Framework**: Next.js 16.0.3 (App Router)
- **React**: 19.0.0
- **TypeScript**: 5.7.2
- **Styling**: Tailwind CSS 4.0.0
- **UI Components**: shadcn/ui (new-york style)
- **State Management**: Zustand 5.0.2
- **Server State**: TanStack Query 5.x
- **Forms**: React Hook Form + Zod
- **Auth**: Keycloak (OIDC PKCE)

## 해결 과정: AI Agent 가이드 문서 설계

### 1. 계층 아키텍처 정의

AI가 코드를 배치할 위치를 명확히 알 수 있도록 계층 구조를 정의합니다:

```
src/
├── types/              # TypeScript interfaces & types only
├── services/           # API Layer (object-based services)
├── hooks/              # React Query hooks (domain-specific)
├── store/              # Zustand stores (global UI state)
├── features/           # Feature-Sliced Design modules
├── components/         # Shared UI components
├── lib/                # Utilities (NOT API calls)
└── app/                # Pages (UI composition only)
```

### 2. 데이터 흐름 시각화

AI가 데이터 흐름을 이해할 수 있도록 다이어그램으로 표현합니다:

```
┌─────────────────────────────────────────────────────────────┐
│                         Page (app/)                         │
│  - UI composition only                                      │
│  - Imports from features/ and hooks/                        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Features (features/)                     │
│  - Domain-specific components                               │
│  - Uses hooks/ for data, components/ for UI primitives      │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      Hooks (hooks/)                         │
│  - React Query hooks (useQuery, useMutation)                │
│  - Calls services/ for API operations                       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Services (services/)                     │
│  - API layer (object-based)                                 │
│  - Uses lib/api/client.ts for HTTP requests                 │
└─────────────────────────────────────────────────────────────┘
```

### 3. 네이밍 컨벤션 명시

일관된 네이밍을 위해 규칙을 표로 정리합니다:

| Item | Convention | Example |
|------|------------|---------|
| Hooks | `use-*.ts` | `use-gateways.ts`, `use-products.ts` |
| Services | `*.ts` | `gateway.ts`, `product.ts` |
| Types | `*.ts` | `gateway.ts`, `product.ts` |
| Components | `*.tsx` | `gateway-card.tsx`, `product-form.tsx` |
| Utilities | `*.ts` | `format-date.ts`, `cn.ts` |
| Feature folders | `kebab-case` | `api-services/`, `client-apps/` |

**핵심 원칙**: 모든 파일은 `kebab-case`를 사용합니다.

### 4. Service 패턴 표준화

API 호출은 모두 service 객체를 통해 수행합니다:

```typescript
// services/gateway.ts
import { apiClient } from '@/lib/api/client';
import type { Gateway, CreateGatewayRequest } from '@/types/gateway';

const BASE_PATH = '/v1/provider/gateways';

export const gatewayService = {
  /**
   * GET /v1/provider/gateways - List all gateways
   */
  list: async (): Promise<Gateway[]> => {
    const { data } = await apiClient.get<{ gateways: Gateway[] }>(BASE_PATH);
    return data.gateways || [];
  },

  /**
   * GET /v1/provider/gateways/:id - Get single gateway
   */
  get: async (id: string): Promise<Gateway> => {
    const { data } = await apiClient.get<{ gateway: Gateway }>(`${BASE_PATH}/${id}`);
    return data.gateway;
  },

  /**
   * POST /v1/provider/gateways - Create gateway
   */
  create: async (request: CreateGatewayRequest): Promise<Gateway> => {
    const { data } = await apiClient.post<{ gateway: Gateway }>(BASE_PATH, request);
    return data.gateway;
  },

  /**
   * DELETE /v1/provider/gateways/:id - Delete gateway
   */
  delete: async (id: string): Promise<void> => {
    await apiClient.delete(`${BASE_PATH}/${id}`);
  },
};
```

### 5. React Query Hook 패턴 표준화

서비스를 호출하는 React Query 훅을 표준화합니다:

```typescript
// hooks/use-gateways.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { gatewayService } from '@/services';
import type { CreateGatewayRequest } from '@/types/gateway';

// Query keys for cache management
export const gatewayKeys = {
  all: ['gateways'] as const,
  list: () => [...gatewayKeys.all, 'list'] as const,
  detail: (id: string) => [...gatewayKeys.all, 'detail', id] as const,
};

/**
 * Hook to fetch all gateways
 */
export function useGateways() {
  return useQuery({
    queryKey: gatewayKeys.list(),
    queryFn: () => gatewayService.list(),
  });
}

/**
 * Hook to create gateway
 */
export function useCreateGateway() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (request: CreateGatewayRequest) => gatewayService.create(request),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: gatewayKeys.all });
    },
  });
}
```

### 6. Feature-Sliced Design 패턴

각 도메인별로 자체 컴포넌트를 가진 feature 모듈을 구성합니다:

```
features/<feature-name>/
├── components/
│   ├── <feature>-card.tsx           # Display component
│   ├── <feature>-form.tsx           # Form component
│   ├── <feature>-table.tsx          # Table component
│   ├── create-<feature>-dialog.tsx  # Create dialog
│   └── edit-<feature>-dialog.tsx    # Edit dialog
└── index.ts                         # Public exports only
```

```typescript
// features/gateways/index.ts
export { GatewayCard } from './components/gateway-card';
export { GatewayForm } from './components/gateway-form';
export { GatewayTable } from './components/gateway-table';
export { CreateGatewayDialog } from './components/create-gateway-dialog';
```

### 7. 상태 관리 가이드라인

상태 종류별로 어떤 도구를 사용할지 명확히 정의합니다:

| 상태 종류 | 도구 | 용도 |
|----------|------|------|
| Server State | TanStack Query | API 데이터 (자동 캐싱, 리페칭) |
| Global UI State | Zustand | 사이드바 상태 등 전역 UI 상태 |
| URL State | nuqs | 필터, 검색, 페이지네이션 |
| Local State | useState | 다이얼로그 open/close 등 |

```tsx
// Server State
const { data, isLoading } = useGateways();

// Global UI State (Zustand)
const { isOpen, toggle } = useSidebarStore();

// URL State (nuqs)
const [search, setSearch] = useQueryState('q');

// Local State
const [isDialogOpen, setIsDialogOpen] = useState(false);
```

### 8. 새 기능 추가 체크리스트

AI가 새 기능을 추가할 때 따라야 할 순서를 정의합니다:

1. **Create Types** (`types/my-feature.ts`)
2. **Create Service** (`services/my-feature.ts`)
3. **Create Hooks** (`hooks/use-my-feature.ts`)
4. **Create Feature Components** (`features/my-feature/`)
5. **Create Page** (`app/(admin)/my-feature/page.tsx`)
6. **Add to Sidebar** (`components/layout/app-sidebar.tsx`)

## 프로젝트 구조 전체 예시

```
frontend/
├── src/
│   ├── app/                          # Next.js App Router pages
│   │   ├── layout.tsx                # Root layout (providers, theme)
│   │   ├── (auth)/                   # Auth route group (no sidebar)
│   │   │   ├── login/page.tsx
│   │   │   └── callback/page.tsx
│   │   ├── (admin)/                  # Admin/Provider route group
│   │   │   ├── layout.tsx            # AppSidebar + Header + auth
│   │   │   ├── dashboard/
│   │   │   ├── gateways/
│   │   │   └── products/
│   │   └── (portal)/                 # Customer DevPortal route group
│   │       ├── layout.tsx            # PortalSidebar + Header + auth
│   │       ├── home/
│   │       └── market/
│   │
│   ├── features/                     # Feature-Sliced Design modules
│   │   ├── gateways/
│   │   │   ├── components/
│   │   │   └── index.ts
│   │   └── products/
│   │       ├── components/
│   │       └── index.ts
│   │
│   ├── hooks/                        # React Query hooks
│   │   ├── index.ts
│   │   ├── use-gateways.ts
│   │   └── use-products.ts
│   │
│   ├── services/                     # API services (object-based)
│   │   ├── index.ts
│   │   ├── gateway.ts
│   │   └── product.ts
│   │
│   ├── types/                        # TypeScript interfaces
│   │   ├── index.ts
│   │   ├── gateway.ts
│   │   └── product.ts
│   │
│   ├── components/                   # Shared UI components
│   │   ├── ui/                       # shadcn/ui primitives
│   │   ├── layout/                   # App layout components
│   │   └── forms/                    # Form field wrappers
│   │
│   ├── store/                        # Zustand stores
│   │   └── sidebar.ts
│   │
│   └── lib/                          # Utilities
│       ├── utils.ts
│       ├── api/client.ts
│       └── auth/keycloak.ts
│
├── components.json                   # shadcn/ui configuration
├── tailwind.config.js
└── next.config.ts
```

## 교훈

### 1. 명시적 규칙이 일관된 AI 출력을 만든다

"API 호출은 services/에서만", "상태 관리는 용도별로 분리" 등 명시적인 규칙을 문서화하면 AI가 일관된 코드를 생성합니다.

### 2. 코드 예시가 설명보다 효과적

"Service는 object-based 패턴을 사용합니다"라는 설명보다 실제 코드 예시가 AI에게 더 명확한 가이드가 됩니다.

### 3. 계층 구조 다이어그램이 데이터 흐름을 명확히 한다

텍스트 설명만으로는 복잡한 계층 구조를 전달하기 어렵습니다. ASCII 다이어그램으로 시각화하면 AI가 더 잘 이해합니다.

### 4. 새 기능 추가 체크리스트가 실수를 방지한다

파일 생성 순서와 필요한 파일 목록을 체크리스트로 제공하면 AI가 필요한 파일을 빠뜨리지 않습니다.

### 5. 네이밍 컨벤션 표가 혼란을 방지한다

`kebab-case` vs `camelCase` vs `PascalCase` 등 네이밍 규칙을 표로 정리하면 AI가 일관된 네이밍을 사용합니다.

## 참고 자료

### 관련 문서
- [Feature-Sliced Design](https://feature-sliced.design/) - 프론트엔드 아키텍처 방법론
- [TanStack Query](https://tanstack.com/query/latest) - 서버 상태 관리
- [Zustand](https://zustand-demo.pmnd.rs/) - 클라이언트 상태 관리

### 기술 스택
- [Next.js](https://nextjs.org/) - React 프레임워크
- [shadcn/ui](https://ui.shadcn.com/) - UI 컴포넌트 라이브러리
- [Tailwind CSS](https://tailwindcss.com/) - 유틸리티 CSS

## 부록 A: 에이전트별 AGENTS.md 연결 설정

세 가지 주요 AI 에이전트(Claude Code, Gemini CLI, OpenAI Codex CLI)에서 AGENTS.md를 인식하는 방법입니다. 모든 에이전트가 AGENTS.md를 자동으로 인식하므로, 하나의 파일로 통일된 개발 경험을 제공할 수 있습니다.

### Claude Code

Claude Code는 `CLAUDE.md`를 자동으로 인식합니다. 프로젝트 루트와 하위 디렉토리에서 계층적으로 로드됩니다.

**파일 우선순위**:
1. `~/.claude/CLAUDE.md` - 글로벌 설정
2. 프로젝트 루트 `CLAUDE.md` - 프로젝트 공통 규칙
3. 하위 디렉토리 `CLAUDE.md` - 도메인별 규칙

**CLAUDE.md에서 AGENTS.md 참조**:
```markdown
# CLAUDE.md (루트)

## 프로젝트 구조

이 모노레포는 다음 앱들로 구성됩니다:

- **Frontend**: `apps/web/` - [AGENTS.md 참조](./apps/web/AGENTS.md)
- **Backend**: `apps/api/` - [AGENTS.md 참조](./apps/api/AGENTS.md)

## 공통 규칙

- 커밋 메시지는 Conventional Commits 형식을 따릅니다
- PR 생성 시 관련 이슈를 링크합니다
- 작업할 디렉토리의 AGENTS.md를 먼저 읽고 해당 패턴을 따릅니다
```

**팁**: `#` 키를 누르고 지침을 입력하면 CLAUDE.md에 자동 추가됩니다.

### Google Gemini CLI

Gemini CLI는 `GEMINI.md`를 기본으로 인식하지만, 설정을 통해 `AGENTS.md`도 인식하도록 구성할 수 있습니다.

**기본 파일 로드 순서**:
1. `~/.gemini/GEMINI.md` - 글로벌 설정
2. 프로젝트 루트에서 현재 디렉토리까지 순회하며 `GEMINI.md` 로드
3. 하위 디렉토리의 `GEMINI.md`도 스캔

**AGENTS.md 인식 설정** (`.gemini/settings.json`):
```json
{
  "contextFileName": "AGENTS.md"
}
```

**GEMINI.md에서 AGENTS.md 임포트**:
```markdown
# GEMINI.md

## 프로젝트 컨텍스트

@./apps/web/AGENTS.md
@./apps/api/AGENTS.md

## 공통 규칙

- 위 AGENTS.md 파일들의 패턴을 따릅니다
- kebab-case 파일 네이밍을 사용합니다
```

**참고**: Android Studio에서는 AGENTS.md를 기본으로 인식하며, GEMINI.md와 동일 디렉토리에 있을 경우 GEMINI.md가 우선합니다.

### OpenAI Codex CLI

Codex CLI는 `AGENTS.md`를 기본으로 자동 인식합니다. 계층적 로드와 오버라이드를 지원합니다.

**파일 검색 순서** (디렉토리별):
1. `AGENTS.override.md` - 최우선 (있으면 사용)
2. `AGENTS.md` - 기본 파일
3. `project_doc_fallback_filenames`에 정의된 추가 파일

**글로벌 설정** (`~/.codex/AGENTS.md`):
```markdown
# 글로벌 Codex 설정

## 기본 규칙
- 모든 프로젝트에서 TypeScript strict 모드 사용
- ESLint와 Prettier 규칙 준수
- 테스트 코드는 .spec.ts 또는 .test.ts 확장자 사용
```

**프로젝트 설정** (`~/.codex/config.toml`):
```toml
# 추가 파일명 인식
project_doc_fallback_filenames = ["CLAUDE.md", "GEMINI.md", ".agents.md"]

# 최대 컨텍스트 크기 (기본 32KB)
project_doc_max_bytes = 65536
```

**AGENTS.md 자동 생성**:
```bash
# Codex CLI에서 /init 명령으로 AGENTS.md 생성
codex /init
```

### 에이전트별 특성 비교

| 특성 | Claude Code | Gemini CLI | Codex CLI |
|------|-------------|------------|-----------|
| 기본 파일명 | `CLAUDE.md` | `GEMINI.md` | `AGENTS.md` |
| AGENTS.md 인식 | 링크로 참조 | 설정 필요 | 기본 인식 |
| 계층적 로드 | 지원 | 지원 | 지원 |
| 오버라이드 파일 | 없음 | 없음 | `AGENTS.override.md` |
| 모듈 임포트 | 링크 | `@file.md` 문법 | 없음 |
| 글로벌 설정 위치 | `~/.claude/` | `~/.gemini/` | `~/.codex/` |
| 컨텍스트 한도 | 200K tokens | 1M+ tokens | 32KB (설정 가능) |
| 파일 시스템 접근 | 가능 | 가능 | 가능 |
| 터미널 명령 실행 | 가능 | 가능 | 가능 |

### 통합 전략: 단일 AGENTS.md 사용

세 에이전트 모두 지원하려면 다음 구조를 권장합니다:

```
project-root/
├── CLAUDE.md           # Claude Code용 (AGENTS.md 링크)
├── AGENTS.md           # 실제 개발 가이드 (Codex/Gemini 기본 인식)
└── .gemini/
    └── settings.json   # Gemini가 AGENTS.md를 인식하도록 설정
```

**CLAUDE.md** (AGENTS.md 참조):
```markdown
# Claude Code 설정

아래 AGENTS.md를 참조하세요:
[AGENTS.md](./AGENTS.md)
```

**.gemini/settings.json** (AGENTS.md 직접 인식):
```json
{
  "contextFileName": "AGENTS.md"
}
```

이 설정으로 GEMINI.md 파일 없이도 Gemini CLI가 AGENTS.md를 직접 인식합니다. Codex CLI는 AGENTS.md를 기본으로 인식하므로 별도 설정이 필요 없습니다.

결과적으로 `AGENTS.md` 하나만 관리하면서 세 에이전트 모두에서 동일한 개발 가이드를 적용할 수 있습니다.

## 부록 B: 전체 AGENTS.md 원본

아래는 실제 프로젝트에서 사용 중인 AI Agent 가이드 문서 전체입니다 (752줄):

```markdown
# Frontend - AI Agent Development Guide

This document provides codebase documentation for the imp-gateway frontend.

## Tech Stack

| Category | Technology | Version |
|----------|------------|---------|
| Framework | Next.js (App Router) | 16.0.3 |
| React | React | 19.0.0 |
| Language | TypeScript | 5.7.2 |
| Styling | Tailwind CSS | 4.0.0 |
| UI Components | shadcn/ui (new-york style) | - |
| State Management | Zustand | 5.0.2 |
| Server State | TanStack Query | 5.x |
| Forms | React Hook Form + Zod | 7.54.1 / 4.1.8 |
| Tables | TanStack React Table | 8.21.2 |
| Auth | Keycloak (OIDC PKCE) | - |
| Charts | Recharts | 2.15.1 |
| Drag & Drop | dnd-kit | 6.3.1 |
| URL State | nuqs | 2.4.1 |
| Bundler | Turbopack (via Next.js) | - |

---

## Architecture Overview

### Layer Architecture

src/
├── types/              # TypeScript interfaces & types only
├── services/           # API Layer (object-based services)
├── hooks/              # React Query hooks (domain-specific)
├── store/              # Zustand stores (global UI state)
├── features/           # Feature-Sliced Design modules
├── components/         # Shared UI components
├── lib/                # Utilities (NOT API calls)
└── app/                # Pages (UI composition only)

### Data Flow

┌─────────────────────────────────────────────────────────────┐
│                         Page (app/)                         │
│  - UI composition only                                      │
│  - Imports from features/ and hooks/                        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Features (features/)                     │
│  - Domain-specific components                               │
│  - Uses hooks/ for data, components/ for UI primitives      │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      Hooks (hooks/)                         │
│  - React Query hooks (useQuery, useMutation)                │
│  - Calls services/ for API operations                       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Services (services/)                     │
│  - API layer (object-based)                                 │
│  - Uses lib/api/client.ts for HTTP requests                 │
└─────────────────────────────────────────────────────────────┘

---

## Naming Convention

**All files use `kebab-case`** for consistency.

| Item | Convention | Example |
|------|------------|---------|
| Hooks | `use-*.ts` | `use-gateways.ts`, `use-products.ts` |
| Services | `*.ts` | `gateway.ts`, `product.ts` |
| Types | `*.ts` | `gateway.ts`, `product.ts` |
| Components | `*.tsx` | `gateway-card.tsx`, `product-form.tsx` |
| Utilities | `*.ts` | `format-date.ts`, `cn.ts` |
| Feature folders | `kebab-case` | `api-services/`, `client-apps/` |

---

## Project Structure

frontend/
├── src/
│   ├── app/                          # Next.js App Router pages
│   │   ├── layout.tsx                # Root layout (providers, theme)
│   │   ├── page.tsx                  # Landing page (redirects)
│   │   │
│   │   ├── (auth)/                   # Auth route group (no sidebar)
│   │   │   ├── login/page.tsx
│   │   │   └── callback/page.tsx
│   │   │
│   │   ├── (admin)/                  # Admin/Provider route group
│   │   │   ├── layout.tsx            # AppSidebar + Header + auth
│   │   │   ├── dashboard/
│   │   │   ├── gateways/
│   │   │   ├── api-services/
│   │   │   ├── products/
│   │   │   ├── deployments/
│   │   │   ├── subscriptions/
│   │   │   └── operator/
│   │   │
│   │   └── (portal)/                 # Customer DevPortal route group
│   │       ├── layout.tsx            # PortalSidebar + Header + auth
│   │       ├── home/
│   │       ├── market/
│   │       ├── my-subscriptions/
│   │       ├── client-apps/
│   │       └── consumers/
│   │
│   ├── features/                     # Feature-Sliced Design modules
│   │   ├── gateways/
│   │   │   ├── components/
│   │   │   │   ├── gateway-card.tsx
│   │   │   │   ├── gateway-form.tsx
│   │   │   │   ├── gateway-table.tsx
│   │   │   │   └── create-gateway-dialog.tsx
│   │   │   └── index.ts              # Public exports
│   │   │
│   │   ├── api-services/
│   │   │   ├── components/
│   │   │   └── index.ts
│   │   │
│   │   ├── products/
│   │   │   ├── components/
│   │   │   └── index.ts
│   │   │
│   │   ├── subscriptions/
│   │   │   ├── components/
│   │   │   └── index.ts
│   │   │
│   │   └── ... (other features)
│   │
│   ├── hooks/                        # React Query hooks
│   │   ├── index.ts                  # Re-exports all hooks
│   │   ├── use-gateways.ts
│   │   ├── use-api-services.ts
│   │   ├── use-products.ts
│   │   ├── use-subscriptions.ts
│   │   ├── use-consumers.ts
│   │   ├── use-api-query.ts          # Base query wrapper
│   │   ├── use-api-mutation.ts       # Base mutation wrapper
│   │   ├── use-tenant.ts
│   │   ├── use-rbac.ts
│   │   └── use-*.ts                  # Utility hooks (shadcn)
│   │
│   ├── services/                     # API services (object-based)
│   │   ├── index.ts                  # Re-exports all services
│   │   ├── gateway.ts
│   │   ├── api-service.ts
│   │   ├── product.ts
│   │   ├── subscription.ts
│   │   └── admin.ts
│   │
│   ├── types/                        # TypeScript interfaces
│   │   ├── index.ts
│   │   ├── gateway.ts
│   │   ├── api-service.ts
│   │   ├── product.ts
│   │   └── subscription.ts
│   │
│   ├── components/                   # Shared UI components
│   │   ├── ui/                       # shadcn/ui primitives
│   │   ├── layout/                   # App layout components
│   │   │   ├── app-sidebar.tsx
│   │   │   ├── portal-sidebar.tsx
│   │   │   ├── header.tsx
│   │   │   ├── page-container.tsx
│   │   │   └── providers.tsx
│   │   ├── forms/                    # Form field wrappers
│   │   ├── tables/                   # Table components
│   │   ├── charts/                   # Chart components
│   │   ├── modals/                   # Modal components
│   │   ├── empty-states/             # Empty state components
│   │   └── kbar/                     # Command palette
│   │
│   ├── store/                        # Zustand stores
│   │   ├── sidebar.ts
│   │   └── entity-cache.ts
│   │
│   ├── lib/                          # Utilities (NO API calls here)
│   │   ├── utils.ts                  # cn() and helpers
│   │   ├── api/
│   │   │   └── client.ts             # Fetch wrapper with auth
│   │   ├── auth/
│   │   │   ├── keycloak.ts           # OIDC PKCE implementation
│   │   │   └── rbac.ts               # Role capability mapping
│   │   └── tenant/
│   │       └── context.tsx           # TenantProvider context
│   │
│   └── constants/                    # Static data and configs
│       └── data.ts
│
├── public/                           # Static assets
├── .env.local
├── .env.example
├── components.json                   # shadcn/ui configuration
├── tailwind.config.js
├── tsconfig.json
└── next.config.ts

---

## Feature-Sliced Design Pattern

Each feature module is self-contained with its own components.

### Feature Structure

features/<feature-name>/
├── components/
│   ├── <feature>-card.tsx           # Display component
│   ├── <feature>-form.tsx           # Form component
│   ├── <feature>-table.tsx          # Table component
│   ├── create-<feature>-dialog.tsx  # Create dialog
│   └── edit-<feature>-dialog.tsx    # Edit dialog
└── index.ts                         # Public exports only

### Feature Index (Public API)

// features/gateways/index.ts
export { GatewayCard } from './components/gateway-card';
export { GatewayForm } from './components/gateway-form';
export { GatewayTable } from './components/gateway-table';
export { CreateGatewayDialog } from './components/create-gateway-dialog';

### Page Using Features

// app/(admin)/gateways/page.tsx
import { GatewayTable, CreateGatewayDialog } from '@/features/gateways';
import { useGateways, useCreateGateway } from '@/hooks';
import PageContainer from '@/components/layout/page-container';

export default function GatewaysPage() {
  const { data: gateways, isLoading } = useGateways();
  const createMutation = useCreateGateway();

  return (
    <PageContainer>
      <div className="flex justify-between">
        <h1>Gateways</h1>
        <CreateGatewayDialog onSubmit={createMutation.mutateAsync} />
      </div>
      <GatewayTable data={gateways} isLoading={isLoading} />
    </PageContainer>
  );
}

---

## Services Pattern

All API calls go through service objects in `services/`.

### Service Structure

// services/gateway.ts
import { apiClient } from '@/lib/api/client';
import type { Gateway, CreateGatewayRequest, UpdateGatewayRequest } from '@/types/gateway';

const BASE_PATH = '/v1/provider/gateways';

export const gatewayService = {
  /**
   * GET /v1/provider/gateways - List all gateways
   */
  list: async (): Promise<Gateway[]> => {
    const { data } = await apiClient.get<{ gateways: Gateway[] }>(BASE_PATH);
    return data.gateways || [];
  },

  /**
   * GET /v1/provider/gateways/:id - Get single gateway
   */
  get: async (id: string): Promise<Gateway> => {
    const { data } = await apiClient.get<{ gateway: Gateway }>(`${BASE_PATH}/${id}`);
    return data.gateway;
  },

  /**
   * POST /v1/provider/gateways - Create gateway
   */
  create: async (request: CreateGatewayRequest): Promise<Gateway> => {
    const { data } = await apiClient.post<{ gateway: Gateway }>(BASE_PATH, request);
    return data.gateway;
  },

  /**
   * PUT /v1/provider/gateways/:id - Update gateway
   */
  update: async (id: string, request: UpdateGatewayRequest): Promise<Gateway> => {
    const { data } = await apiClient.put<{ gateway: Gateway }>(`${BASE_PATH}/${id}`, request);
    return data.gateway;
  },

  /**
   * DELETE /v1/provider/gateways/:id - Delete gateway
   */
  delete: async (id: string): Promise<void> => {
    await apiClient.delete(`${BASE_PATH}/${id}`);
  },
};

### Service Index

// services/index.ts
export { gatewayService } from './gateway';
export { apiServiceService } from './api-service';
export { productService, productPublishService, planService } from './product';
export { subscriptionService, clientAppService, consumerService } from './subscription';
export { adminService } from './admin';

---

## React Query Hooks Pattern

All data fetching uses React Query hooks in `hooks/`.

### Hook Structure

// hooks/use-gateways.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { gatewayService } from '@/services';
import type { CreateGatewayRequest, UpdateGatewayRequest } from '@/types/gateway';

// Query keys for cache management
export const gatewayKeys = {
  all: ['gateways'] as const,
  list: () => [...gatewayKeys.all, 'list'] as const,
  detail: (id: string) => [...gatewayKeys.all, 'detail', id] as const,
};

/**
 * Hook to fetch all gateways
 */
export function useGateways() {
  return useQuery({
    queryKey: gatewayKeys.list(),
    queryFn: () => gatewayService.list(),
  });
}

/**
 * Hook to fetch single gateway
 */
export function useGateway(id: string) {
  return useQuery({
    queryKey: gatewayKeys.detail(id),
    queryFn: () => gatewayService.get(id),
    enabled: !!id,
  });
}

/**
 * Hook to create gateway
 */
export function useCreateGateway() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (request: CreateGatewayRequest) => gatewayService.create(request),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: gatewayKeys.all });
    },
  });
}

/**
 * Hook to update gateway
 */
export function useUpdateGateway(id: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (request: UpdateGatewayRequest) => gatewayService.update(id, request),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: gatewayKeys.all });
    },
  });
}

/**
 * Hook to delete gateway
 */
export function useDeleteGateway() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (id: string) => gatewayService.delete(id),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: gatewayKeys.all });
    },
  });
}

### Hook Index

// hooks/index.ts
export { useTenant } from './use-tenant';
export { useRBAC } from './use-rbac';

// Gateway hooks
export {
  useGateways,
  useGateway,
  useCreateGateway,
  useUpdateGateway,
  useDeleteGateway,
  gatewayKeys,
} from './use-gateways';

// Product hooks
export {
  useProducts,
  useProduct,
  useCreateProduct,
  useUpdateProduct,
  useDeleteProduct,
  productKeys,
} from './use-products';

// ... other domain hooks

---

## Route Groups

| Route Group | Purpose | Layout | Auth Required |
|-------------|---------|--------|---------------|
| `(auth)` | Login/Callback | Minimal | No |
| `(admin)` | Provider/Operator UI | AppSidebar | Yes |
| `(portal)` | Customer DevPortal | PortalSidebar | Yes |

**URL Mapping**:
- `(admin)/dashboard` → `/dashboard`
- `(portal)/home` → `/home`
- `(auth)/login` → `/login`

---

## Authentication

### Keycloak OIDC PKCE Flow

┌─────────┐     ┌─────────┐     ┌──────────┐
│  User   │────▶│ /login  │────▶│ Keycloak │
└─────────┘     └─────────┘     └──────────┘
                                      │
                                      ▼
┌───────────┐   ┌───────────┐   ┌──────────┐
│ /dashboard│◀──│ /callback │◀──│ Redirect │
└───────────┘   └───────────┘   └──────────┘

### Key Functions (`lib/auth/keycloak.ts`)

initiateLogin(): Promise<void>
initiateRegistration(): Promise<void>
handleCallback(code: string, state: string): Promise<boolean>
isAuthenticated(): boolean
getCurrentUser(): UserInfo | null
refreshAccessToken(): Promise<boolean>
logout(): void

---

## API Response Patterns

### Standard Response Envelope

// Success response
{
  "success": true,
  "data": { /* payload */ }
}

// Error response
{
  "success": false,
  "error": {
    "code": "ERR_VALIDATION",
    "message": "Human readable message",
    "reason": "Detailed error reason"
  }
}

The `apiClient` automatically unwraps the envelope:

const { data } = await apiClient.get<{ gateway: Gateway }>('/v1/gateways/123');
// data = { gateway: { id: '123', ... } }

### Pagination Response

// Request
GET /v1/admin/tenants?limit=20&offset=0

// Response
{
  "success": true,
  "data": {
    "tenants": [...],
    "total": 150
  }
}

---

## State Management

### Server State (TanStack Query)

Use for all API data - automatic caching, refetching, and synchronization.

const { data, isLoading, error, refetch } = useGateways();

### Global UI State (Zustand)

Use for UI-only state that needs to persist across components.

// store/sidebar.ts
import { create } from 'zustand';

interface SidebarStore {
  isOpen: boolean;
  toggle: () => void;
}

export const useSidebarStore = create<SidebarStore>((set) => ({
  isOpen: true,
  toggle: () => set((state) => ({ isOpen: !state.isOpen })),
}));

### URL State (nuqs)

Use for state that should be reflected in the URL (filters, search, pagination).

import { useQueryState } from 'nuqs';
const [search, setSearch] = useQueryState('q');

### Local Component State (useState)

Use for ephemeral UI state (form inputs, dialog open/close).

const [isDialogOpen, setIsDialogOpen] = useState(false);

---

## Form Pattern

Forms use React Hook Form with Zod validation:

import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import * as z from 'zod';

const schema = z.object({
  name: z.string().min(2, 'Name must be at least 2 characters'),
  email: z.string().email('Invalid email address'),
});

type FormData = z.infer<typeof schema>;

export function MyForm() {
  const form = useForm<FormData>({
    resolver: zodResolver(schema),
    defaultValues: { name: '', email: '' },
  });

  const onSubmit = (data: FormData) => {
    // Handle submission
  };

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)}>
        {/* Form fields */}
      </form>
    </Form>
  );
}

---

## Adding New Features

### 1. Create Types

// types/my-feature.ts
export interface MyFeature {
  id: string;
  name: string;
  // ...
}

export interface CreateMyFeatureRequest {
  name: string;
}

### 2. Create Service

// services/my-feature.ts
import { apiClient } from '@/lib/api/client';
import type { MyFeature, CreateMyFeatureRequest } from '@/types/my-feature';

export const myFeatureService = {
  list: async (): Promise<MyFeature[]> => { /* ... */ },
  get: async (id: string): Promise<MyFeature> => { /* ... */ },
  create: async (req: CreateMyFeatureRequest): Promise<MyFeature> => { /* ... */ },
  // ...
};

### 3. Create Hooks

// hooks/use-my-feature.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { myFeatureService } from '@/services';

export const myFeatureKeys = {
  all: ['my-features'] as const,
  list: () => [...myFeatureKeys.all, 'list'] as const,
  detail: (id: string) => [...myFeatureKeys.all, 'detail', id] as const,
};

export function useMyFeatures() { /* ... */ }
export function useCreateMyFeature() { /* ... */ }
// ...

### 4. Create Feature Components

// features/my-feature/components/my-feature-table.tsx
// features/my-feature/components/create-my-feature-dialog.tsx
// features/my-feature/index.ts

### 5. Create Page

// app/(admin)/my-feature/page.tsx
import { MyFeatureTable, CreateMyFeatureDialog } from '@/features/my-feature';
import { useMyFeatures } from '@/hooks';

### 6. Add to Sidebar

Update `components/layout/app-sidebar.tsx` or `portal-sidebar.tsx`.

---

## Development Commands

# Install dependencies
pnpm install

# Development server (with Turbopack)
pnpm dev

# Production build
pnpm build

# Type checking
pnpm typecheck

# Linting
pnpm lint
pnpm lint:fix

# Formatting
pnpm format
pnpm format:check

---

## Code Style Summary

| Category | Convention |
|----------|------------|
| File naming | `kebab-case` for all files |
| Component naming | `PascalCase` in code |
| Hook naming | `use-*.ts` files, `useMyHook` in code |
| Service naming | Object-based, `myService.method()` |
| TypeScript | Strict mode, explicit types |
| Exports | Named exports preferred |
| State | React Query for server, Zustand for UI |

---

## Important Files

| File | Purpose |
|------|---------|
| `src/app/layout.tsx` | Root layout with providers |
| `src/app/(admin)/layout.tsx` | Admin layout with AppSidebar |
| `src/app/(portal)/layout.tsx` | Portal layout with PortalSidebar |
| `src/lib/api/client.ts` | API client with auth |
| `src/lib/auth/keycloak.ts` | OIDC PKCE authentication |
| `src/components/layout/providers.tsx` | QueryClientProvider setup |
| `src/hooks/index.ts` | Hook exports |
| `src/services/index.ts` | Service exports |
```
