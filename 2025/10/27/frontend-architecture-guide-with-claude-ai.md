# Claude AI와 함께하는 프론트엔드 개발: imprun.dev의 CLAUDE.md 가이드 공개

> **작성일**: 2025년 10월 27일
> **프로젝트**: imprun.dev (Kubernetes 기반 서버리스 Cloud Function 플랫폼)
> **키워드**: Claude AI, 프론트엔드 아키텍처, 개발 가이드, Container/Presentational Pattern, Layered Architecture

---

## 들어가며

[**imprun.dev**](https://imprun.dev)는 Kubernetes 기반 서버리스 Cloud Function 플랫폼입니다. 프론트엔드를 개발하면서 Claude AI와 협업하는 과정에서, **새 세션마다 코딩 스타일이 달라지는** 문제에 직면했습니다.

**우리가 겪은 문제**:
- ❌ **새 세션마다 다른 패턴**: useQuery → useState, Service Layer 생략 등
- ❌ 프레임워크 변경 시 (Next.js → React Router) 일관성 상실
- ❌ 20개 페이지, 50개 컴포넌트 관리하며 매번 스타일 재설명 필요
- ✅ CLAUDE.md 작성 후 모든 문제 해결

**핵심 발견**:
- Container/Presentational + Layered Architecture만 명확하면 **프레임워크 변경 시 95% 코드 재사용**
- 997줄 CLAUDE.md로 **코드 리뷰 시간 75% 단축, 신입 온보딩 2주→3일**
- AI가 읽을 개발 가이드는 **프로젝트별 필수 문서**

이 글은 **imprun.dev 프론트엔드 구축 경험**을 바탕으로, Claude AI와의 협업 효율을 극대화하는 CLAUDE.md 작성법을 공유합니다.

---

## TL;DR

**imprun.dev 프론트엔드 개발 과정에서 터득한 노하우**를 Claude AI 전용 개발 가이드로 정리했습니다.

- ✅ 새 세션마다 일관된 코드를 작성하기 위한 **프론트엔드 개발 가이드** 공개
- ✅ **Container/Presentational + Layered Architecture**로 프레임워크 독립성 확보
- ✅ Claude AI가 이 가이드 하나로 **Next.js → React Router DOM 마이그레이션도 95%는 그대로 재사용**
- ✅ 997줄, 11개 섹션으로 구성된 **완전한 개발 지침서**

**📄 가이드 전문**: [frontend/CLAUDE.md](../../frontend/CLAUDE.md)

---

## 배경: imprun.dev 프론트엔드 개발 여정

### imprun.dev란?

**imprun.dev**는 Kubernetes 기반 서버리스 Cloud Function 플랫폼입니다:

```
기능:
- Cloud Functions 관리 (생성, 편집, 배포)
- 실시간 로그 모니터링
- Database 관리 (MongoDB)
- Trigger 설정 (HTTP, Cron)
- 앱별 독립 런타임 환경

스택:
- Frontend: React 19 + react-router-dom + TanStack Query
- Backend: NestJS + MongoDB
- Infra: Kubernetes (ARM64)
```

### 개발 과정의 시행착오

1. **React 19 + TanStack Router** → 파일 라우팅 불편
2. **Next.js App Router로 대규모 리팩토링** → Docker/K8s 배포 문제
3. **React Router DOM으로 재전환** → 아키텍처 덕분에 95% 코드 재사용

자세한 마이그레이션 스토리는 [이 글](./nextjs-to-react-migration-journey.md)을 참조하세요.

### 이 가이드가 탄생한 계기

이런 반복적인 마이그레이션 과정에서 **"프레임워크에 종속되지 않는 아키텍처"**의 중요성을 깨달았고, Claude AI와의 협업 과정에서 **일관성 유지의 어려움**을 경험했습니다.

> "Claude는 강력하지만, 세션이 바뀌면 이전 규칙을 잊어버린다."

그래서 만든 것이 **CLAUDE.md** - Claude AI가 읽고 따를 수 있는 개발 가이드입니다.

---

## 왜 이 가이드를 만들었나?

### 문제: Claude AI의 휘발성 메모리

Claude AI는 강력한 코딩 어시스턴트지만, **세션이 끝나면 컨텍스트를 잃어버립니다**.

```
세션 1: "Container/Presentational 패턴으로 컴포넌트 작성해줘"
  → ✅ 완벽하게 작성

세션 2 (다음 날): "비슷한 컴포넌트 하나 더 만들어줘"
  → ❌ 전혀 다른 패턴으로 작성
  → ❌ 이전 규칙 완전히 망각
```

**결과**: 코드베이스가 시간이 지나면서 **일관성을 잃고 혼란**스러워집니다.

### 해결책: CLAUDE.md

> **"AI에게 읽힐 개발 가이드를 만들자"**

Claude AI의 프로젝트 파일 읽기 기능을 활용해, 매 세션마다 동일한 규칙으로 개발하도록 하는 **AI 전용 개발 가이드**를 작성했습니다.

---

## CLAUDE.md의 핵심 철학

### 1. 프레임워크 독립적 아키텍처

> **"프레임워크는 UI 레이어에만 영향을 줘야 한다. 비즈니스 로직과 데이터 레이어는 독립적이어야 한다."**

이 철학 덕분에 **Next.js → React Router DOM 마이그레이션**이 놀라울 정도로 쉬웠습니다:

**프레임워크 변경 시 영향 범위**:
- 변경 필요 (5%): 라우팅, 페이지 구조
- 변경 불필요 (95%): Hooks, Services, Components, Store

자세한 마이그레이션 경험담은 [이 글](./nextjs-to-react-migration-journey.md)을 참조하세요.

### 2. Container/Presentational Pattern

**데이터 로직과 UI 렌더링을 완전히 분리**합니다.

```tsx
// ✅ Container Component (데이터 + 로직)
export function ApplicationListContainer() {
  const { data: applications, isLoading } = useApplications()
  const activeApps = applications?.filter(app => app.phase !== 'Deleted')

  return <ApplicationList applications={activeApps} isLoading={isLoading} />
}

// ✅ Presentational Component (렌더링만)
export function ApplicationList({ applications, isLoading }) {
  if (isLoading) return <Spinner />

  return (
    <div className="grid grid-cols-3 gap-4">
      {applications.map(app => (
        <ApplicationCard key={app.gatewayId} app={app} />
      ))}
    </div>
  )
}
```

**장점**:
- 테스트 용이: Presentational은 props만 검증하면 됨
- 재사용성: 동일한 UI를 다른 데이터 소스에 연결 가능
- 프레임워크 독립성: 렌더링 로직은 라우터 변경에 영향 없음

### 3. Layered Architecture (4계층)

```
┌──────────────────────────────────────┐
│  Component Layer                      │  ← UI 렌더링
│  - React 컴포넌트                      │
│  - 사용자 인터랙션                     │
├──────────────────────────────────────┤
│  Hook Layer                           │  ← 비즈니스 로직
│  - TanStack Query (서버 상태)         │
│  - 로직 Hook (재사용 가능한 로직)      │
├──────────────────────────────────────┤
│  Service Layer                        │  ← API 인터페이스
│  - *.service.ts                       │
│  - 도메인별 API 그룹화                 │
├──────────────────────────────────────┤
│  HTTP Client Layer                    │  ← 네트워크 통신
│  - axios                              │
│  - 인증 토큰, 공통 에러 처리           │
└──────────────────────────────────────┘
```

**실제 데이터 흐름**:

```tsx
// 1️⃣ Component Layer
export function ApplicationCard({ app }) {
  const { actions, state } = useApplicationControl(app)  // Hook 호출

  return (
    <Button onClick={actions.start} disabled={!state.canStart}>
      시작
    </Button>
  )
}

// 2️⃣ Hook Layer
export function useApplicationControl(app) {
  const queryClient = useQueryClient()

  const start = async () => {
    await applicationService.start(app.gatewayId)  // Service 호출
    queryClient.invalidateQueries({ queryKey: ['applications'] })
    toast.success("시작했습니다")
  }

  return { actions: { start }, state: { canStart: app.phase === 'Stopped' } }
}

// 3️⃣ Service Layer
export const applicationService = {
  async start(gatewayId: string): Promise<void> {
    await httpClient.post(`/v1/applications/${gatewayId}/start`)  // HTTP 호출
  }
}

// 4️⃣ HTTP Client Layer
const httpClient = axios.create({ baseURL: env.API_URL })
httpClient.interceptors.request.use((config) => {
  const token = useAuthStore.getState().token
  if (token) config.headers.Authorization = `Bearer ${token}`
  return config
})
```

**각 레이어의 책임이 명확**해서 어디에 코드를 작성해야 할지 고민할 필요가 없습니다.

---

## CLAUDE.md 구조

### 997줄, 11개 섹션

```markdown
1. 기술 스택
   - Core: React 19, react-router-dom, TypeScript, Vite
   - 상태 관리: Zustand, TanStack Query v5
   - UI: Tailwind CSS v4, shadcn/ui

2. 아키텍처 패턴 ⭐
   - Container/Presentational Pattern
   - Layered Architecture (4계층)
   - 프레임워크 독립성

3. 프로젝트 구조
   - pages/, routes/, components/, hooks/, services/, store/, types/

4. 페이지 작성 규칙
   - React Router DOM 기반
   - Hook으로 데이터 페칭
   - 컴포넌트 조합으로 UI 구성

5. 컴포넌트 작성 규칙
   - 분리 기준: 재사용, 복잡도, 독립 로직
   - 도메인별 배치
   - Container/Presentational 패턴 적용

6. Hooks 패턴
   - Data Hooks: TanStack Query
   - Logic Hooks: 재사용 가능한 비즈니스 로직
   - 명명 규칙

7. 타입 시스템
   - 도메인별 타입 분리
   - 중앙 re-export 필수
   - State vs Phase 구분

8. 스타일링 & 디자인 시스템 ⭐
   - Tailwind 패턴 (flex + gap)
   - Semantic Colors (다크모드 자동 지원)
   - DESIGN_GUIDELINES.md 링크

9. API 서비스 패턴
   - Service Layer 구조
   - HTTP Client 설정
   - 에러 처리

10. 새 기능 추가 가이드
    - 7단계 체크리스트
    - 라우트 → NavBar → 타입 → Service → Hook → 컴포넌트 → 페이지

11. 코드 리뷰 체크리스트
    - 아키텍처, 컴포넌트, Hooks, 타입, 스타일링, API
```

---

## 실제 사용 예시

### 새 기능 추가 시

Claude에게 이렇게 요청합니다:

```
"CLAUDE.md를 참고해서 Trigger 관리 페이지를 추가해줘"
```

Claude는 자동으로:

1. ✅ **라우트 추가** (`routes/index.tsx`)
2. ✅ **NavBar 업데이트** (`components/layout/AppNavBar.tsx`)
3. ✅ **타입 정의** (`types/trigger.ts` + `types/index.ts` re-export)
4. ✅ **Service 작성** (`services/trigger.service.ts`)
5. ✅ **Hook 작성** (`hooks/use-triggers.ts`)
6. ✅ **컴포넌트 작성** (`components/triggers/TriggerList.tsx`)
7. ✅ **페이지 작성** (`pages/TriggerPage.tsx`)

**모든 코드가 기존 패턴과 동일한 구조**로 생성됩니다!

### 코드 리뷰 시

```
"이 PR을 CLAUDE.md의 체크리스트로 검증해줘"
```

Claude가 자동으로 확인:
- [ ] Container/Presentational 패턴 준수?
- [ ] 4계층 구조 유지?
- [ ] 비즈니스 로직이 Hook Layer에?
- [ ] 타입이 `types/index.ts`에서 re-export?
- [ ] `flex + gap` 사용, `margin` 회피?
- [ ] Semantic colors 사용?

---

## 효과

### Before (CLAUDE.md 없이)

```tsx
// 세션 1: 이렇게 작성
const { data } = useQuery(['apps'], fetchApps)

// 세션 2: 저렇게 작성
const apps = await appService.getAll()

// 세션 3: 또 다르게 작성
const [apps, setApps] = useState([])
useEffect(() => { /* ... */ }, [])
```

**결과**: 3가지 패턴이 혼재 → 유지보수 악몽

### After (CLAUDE.md 사용)

```tsx
// 모든 세션에서 동일한 패턴
const { data: apps } = useApplications()  // TanStack Query Hook
```

**결과**: 일관된 패턴 → 유지보수 용이

### 정량적 효과

| 지표 | Before | After | 개선 |
|------|--------|-------|------|
| 새 기능 추가 시간 | 2-3시간 | 30분 | **75% 단축** |
| 코드 리뷰 시간 | 1시간 | 15분 | **75% 단축** |
| 패턴 일관성 | 60% | 95% | **35%p 향상** |
| 온보딩 시간 | 2주 | 3일 | **78% 단축** |

---

## CLAUDE.md의 숨겨진 가치

### 1. 살아있는 문서

일반적인 개발 문서는 **작성 후 곧 구식**이 됩니다. 하지만 CLAUDE.md는 다릅니다:

- ✅ **AI가 직접 사용**하므로 항상 최신 상태 유지 동기 강함
- ✅ **매 커밋마다 검증**되므로 실제와 괴리 없음
- ✅ **새 팀원이 Claude와 페어 프로그래밍**하며 빠르게 학습

**imprun.dev 사례**:
```
Function 에디터 개선 → CLAUDE.md 업데이트 → 다음 기능(Database 뷰)에 자동 반영
```

### 2. 온보딩 도구

신입 개발자에게:

```
"CLAUDE.md를 읽고, Claude에게 Trigger 목록 페이지 만들어달라고 해봐"
```

- 가이드를 읽으며 imprun.dev 아키텍처 이해
- Claude가 생성한 코드로 실제 패턴 학습
- 3일 만에 팀 코딩 스타일 숙지

**실제 경험**: 새 개발자가 Application 관리 기능을 보고 → Database 관리 기능을 동일한 패턴으로 작성

### 3. 의사결정 기록

CLAUDE.md는 단순 규칙이 아니라 **"왜 이렇게 하는가"**를 담고 있습니다:

```markdown
### State vs Phase

- **State**: 사용자가 설정한 목표 상태 (Running, Stopped)
- **Phase**: 실제 시스템 상태 (Starting, Started, Stopping, Stopped)

// ✅ Phase 기반 UI 제어
const canStart = app.phase === ApplicationPhase.Stopped
```

**왜 State가 아닌 Phase를 사용하는가?**
→ K8s Pod의 실제 상태를 반영해야 하므로

이런 **컨텍스트**가 있어야 나중에 규칙을 바꿀 때도 올바른 판단을 할 수 있습니다.

---

## 다른 프로젝트에 적용하기

### 1. 기본 구조 복사

```bash
# CLAUDE.md 기본 템플릿
cp frontend/CLAUDE.md your-project/

# 수정할 부분
- 기술 스택 섹션 (프레임워크, 라이브러리)
- 프로젝트 구조 (실제 디렉터리 구조로 수정)
- 코드 예시 (도메인 이름 변경)
```

### 2. 팀 규칙 추가

```markdown
## 우리 팀만의 규칙

### Git Commit Convention
- feat: 새 기능
- fix: 버그 수정
- refactor: 리팩토링

### 테스트 커버리지
- 모든 Hook은 테스트 필수
- E2E 테스트는 주요 플로우만
```

### 3. 점진적 개선

```markdown
## 변경 이력

### 2025-10-27
- Container/Presentational 패턴 도입
- 이유: 프레임워크 마이그레이션 대비

### 2025-11-01
- Service Layer에 재시도 로직 추가
- 이유: 네트워크 불안정 환경 대응
```

**살아있는 문서**로 만들려면 **변경 이력**을 남기세요.

---

## 한계와 주의사항

### 1. 과도한 규칙은 독

```markdown
❌ 나쁜 예:
"변수명은 반드시 camelCase로, 3단어 이상 15단어 이하로 작성"

✅ 좋은 예:
"변수명은 명확하고 간결하게 (예: userName, isLoading)"
```

**원칙**: 규칙은 최소한으로, 예시는 풍부하게

### 2. AI는 완벽하지 않음

Claude는 CLAUDE.md를 참고하지만:
- ❌ 100% 정확하게 따르진 않음 (가끔 실수)
- ❌ 복잡한 비즈니스 로직은 사람이 검증 필요
- ✅ 패턴 일관성 유지에는 매우 효과적

**결론**: AI 생성 코드도 코드 리뷰는 필수

### 3. 컨텍스트 길이 제한

CLAUDE.md가 너무 길면:
- ❌ AI가 전체를 읽지 못할 수 있음
- ❌ 중요한 부분이 묻힐 수 있음

**해결책**:
- 핵심 규칙만 CLAUDE.md에
- 상세 가이드는 별도 문서로 (예: DESIGN_GUIDELINES.md)
- 링크로 연결

---

## 마무리: "일관성이 완벽함보다 중요하다"

프론트엔드 개발에서 **가장 중요한 것은 일관성**입니다.

```
완벽한 아키텍처 (but 일관성 없음)
  <
좋은 아키텍처 (with 일관성)
```

CLAUDE.md는:
- ✅ 팀원들이 동일한 패턴으로 개발
- ✅ 새 세션의 Claude도 동일한 패턴으로 생성
- ✅ 6개월 후 코드를 봐도 이해 가능

**결과**: 유지보수 가능한 코드베이스

---

## 전체 가이드 공개

**📄 CLAUDE.md 전문**: [frontend/CLAUDE.md](../../frontend/CLAUDE.md)

**주요 내용**:
- 997줄 완전한 개발 가이드
- Container/Presentational + Layered Architecture
- 새 기능 추가 7단계 가이드
- 코드 리뷰 체크리스트
- 실제 코드 예시 다수

자유롭게 참고하고, 여러분의 프로젝트에 맞게 수정해서 사용하세요!

---

## 관련 글

- [Next.js를 버리고 순수 React로 돌아온 이유](./nextjs-to-react-migration-journey.md)
- [프레임워크 독립적 아키텍처가 마이그레이션을 쉽게 만든다](./nextjs-to-react-migration-journey.md#마이그레이션이-쉬웠던-이유)

---

## 피드백 환영

이 가이드는 계속 진화하고 있습니다. 여러분의 경험과 피드백을 공유해주세요:

- 📧 이메일: [your-email]
- 💬 Issue: [GitHub Issues]
- 🐦 트위터: [@your-handle]

**함께 더 나은 개발 문화를 만들어갑시다!** 🚀

---

**태그**: `#Claude` `#AI` `#프론트엔드` `#아키텍처` `#개발가이드` `#React` `#TypeScript` `#일관성`
