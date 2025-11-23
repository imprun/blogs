# CLAUDE.md 최적화 여정: AI가 패턴을 무시하는 이유와 해결책

**작성일:** 2025-11-03
**카테고리:** Claude AI, 개발 가이드, 최적화, Anthropic Best Practices
**난이도:** 중급

---

## TL;DR

- **문제**: 997줄 CLAUDE.md를 작성했는데도 AI가 패턴을 60%만 준수
- **해결**: 파일 참조 전략 + MCP 강제 활성화 + 우선순위 재구조화
- **핵심**: "AI에게 코드를 주지 말고, 파일 경로를 알려주라"
- **결과**: 997줄 → 593줄 (43% 감축), 패턴 준수율 95%

---

## 들어가며

[**imprun.dev**](https://imprun.dev)는 "API 개발부터 AI 통합까지, 모든 것을 하나로 제공"하는 [Kubernetes 기반 API 플랫폼](https://blog.imprun.dev/50)입니다. [이전 글](https://blog.imprun.dev/42)에서 997줄짜리 **CLAUDE.md**를 공개했습니다. Container/Presentational 패턴과 Layered Architecture로 일관성 있는 코드를 만들어낸 성공 스토리였죠.

**하지만 실제 개발에서는 문제가 있었습니다.**

**우리가 마주한 질문**:
- ❓ 997줄 가이드를 작성했는데 왜 Claude가 패턴을 60%만 따를까?
- ❓ MCP 도구(Serena, Sequential-thinking)가 있는데 왜 직접 검색을 시도할까?
- ❓ 가이드에 코드가 있는데 왜 다른 스타일로 생성할까?

**검증 과정**:

1. **Anthropic 공식 Best Practices 분석**
   - ✅ 공식 문서 발견: [claude-code-best-practices](https://www.anthropic.com/engineering/claude-code-best-practices)
   - ✅ 현재 가이드 자체 평가: 약 81/100 (추정)
   - ❌ 코드 임베딩 과다, 우선순위 불명확

2. **파일 참조 전략 도입**
   - ✅ 25줄 코드 샘플 → 10줄 패턴 + 파일 경로
   - ✅ 997줄 → 537줄 (46% 감축)
   - ❌ 하지만 여전히 MCP 미사용 문제

3. **MCP 강제 활성화** ← **최종 선택**
   - ✅ Serena + Sequential-thinking 필수 사용
   - ✅ 비활성화 시 즉시 경고
   - ✅ 우선순위 재구조화 (DESIGN_GUIDELINES 최상단)

**결론**:
- ✅ 997줄 → 593줄 (43% 감축)
- ✅ Anthropic Best Practices 자체 평가 약 95/100 (추정)
- ✅ 패턴 준수율 약 60% → 95% (실제 개발 경험)

이 글은 **[imprun.dev](https://imprun.dev) 플랫폼 구축 경험**을 바탕으로, CLAUDE.md 최적화 과정과 Anthropic 공식 Best Practices 적용 방법을 공유합니다.

---

## 배경: 997줄인데 왜 안 따를까?

### 문제 발견

Part 1 블로그 포스팅 후 실제 개발을 진행하면서 이상한 점을 발견했습니다:

```
나: "Grant 관리 페이지를 Container/Presentational 패턴으로 만들어줘"

Claude: (997줄 CLAUDE.md를 읽음)
  → ❌ useState + useEffect 사용 (TanStack Query 무시)
  → ❌ 컴포넌트에 비즈니스 로직 혼재 (Hook 분리 안 함)
  → ❌ margin 사용 (flex + gap 규칙 무시)

나: "CLAUDE.md를 읽었잖아? 왜 안 따라?"

Claude: "아, 죄송합니다. 다시 작성하겠습니다."
  → ✅ 이번엔 제대로 작성
```

**패턴을 알고 있는데도 가끔 무시**합니다. 왜일까요?

### 가설 1: 가이드가 너무 길다

```markdown
frontend/CLAUDE.md: 997줄
DESIGN_GUIDELINES.md: 553줄

합계: 1,550줄
```

Claude의 컨텍스트 윈도우는 크지만, **997줄을 전부 읽고 기억하기는 어렵습니다**.

특히 중요한 규칙이 중간에 묻혀있으면:
- 읽어도 **우선순위를 파악하기 어려움**
- 가이드를 읽은 후 **사용자 요청을 처리하면서 일부 규칙을 잊어버림**

### 가설 2: 코드 샘플이 너무 많다

CLAUDE.md의 코드 샘플 예시:

```tsx
// ❌ Before: 25줄짜리 완전한 구현
export function useGrantFilters(data: Grant[] | undefined) {
  const [envFilter, setEnvFilter] = useState<string>('all')
  const [statusFilter, setStatusFilter] = useState<string>('all')
  const [syncFilter, setSyncFilter] = useState<string>('all')
  const [currentPage, setCurrentPage] = useState(1)
  const [pageSize, setPageSize] = useState(10)

  const filtered = useMemo(() => {
    if (!data) return []

    let result = data

    if (envFilter !== 'all') {
      result = result.filter(g => g.environment === envFilter)
    }

    if (statusFilter !== 'all') {
      result = result.filter(g => g.status === statusFilter)
    }

    // ... 더 많은 코드
  }, [data, envFilter, statusFilter, syncFilter])

  return {
    filters: { envFilter, statusFilter, syncFilter },
    setEnvFilter, setStatusFilter, setSyncFilter,
    data: { filtered, stats, paginatedGrants },
    pagination: { currentPage, totalPages, pageSize },
    setCurrentPage, setPageSize
  }
}
```

**문제점**:
- 완전한 구현을 보여주려다 보니 **25줄이나 차지**
- Claude가 코드를 **그대로 복사**할 가능성 높음
- **패턴의 본질**보다 **구현 세부사항**에 집중

### 가설 3: 우선순위가 불명확하다

997줄 중 **가장 중요한 것**은?

1. **DESIGN_GUIDELINES.md 참조** (디자인 일관성)
2. **Container/Presentational 패턴** (아키텍처 핵심)
3. **MCP 서버 활용** (코드 품질)

이 3가지가 가장 중요한데, 가이드 중간에 묻혀있었습니다.

**결과**: Claude는 모든 규칙을 평등하게 취급 → 핵심을 놓침

---

## 해결책 1: Anthropic 공식 Best Practices 분석

### Anthropic 공식 문서 발견

[claude-code-best-practices](https://www.anthropic.com/engineering/claude-code-best-practices)

공식 문서에서 제시하는 핵심 원칙:

```markdown
1. 파일 경로 참조 > 코드 임베딩
   - ✅ "hooks/use-grants.ts 참고"
   - ❌ 25줄짜리 코드 전체 복사

2. 우선순위 명확화
   - 최상단에 가장 중요한 규칙 배치
   - 섹션별 중요도 표시

3. 실제 파일 구조 반영
   - 프로젝트의 실제 디렉터리 구조 사용
   - 예시 파일은 실제로 존재하는 파일

4. 외부 도구 활용 명시
   - MCP 서버, 커스텀 도구 사용 가이드
   - 도구 사용이 필수인 경우 명확히 표시

5. 간결함 유지
   - 핵심만 담기
   - 상세 내용은 별도 문서로 분리
```

### 현재 CLAUDE.md 평가

Anthropic Best Practices 대조표:

| 항목 | Before | 점수 | 문제점 |
|------|--------|------|--------|
| **파일 참조 전략** | 코드 샘플 중심 | 60/100 | 25줄짜리 코드 다수 |
| **우선순위 명확화** | 중간에 묻힘 | 70/100 | DESIGN_GUIDELINES 언급이 8번째 섹션 |
| **실제 파일 구조** | 잘 반영됨 | 95/100 | ✅ |
| **외부 도구 활용** | 언급 없음 | 0/100 | MCP 서버 가이드 누락 |
| **간결함** | 997줄 | 80/100 | 너무 길지만 구조는 좋음 |

**총점: 81/100 (B)**

**목표: 95/100 (A+)**

---

## 해결책 2: 파일 참조 전략

### Before: 코드 임베딩 (25줄)

```markdown
### useGrantFilters Hook 예시

```tsx
export function useGrantFilters(data: Grant[] | undefined) {
  const [envFilter, setEnvFilter] = useState<string>('all')
  const [statusFilter, setStatusFilter] = useState<string>('all')
  const [syncFilter, setSyncFilter] = useState<string>('all')
  const [currentPage, setCurrentPage] = useState(1)
  const [pageSize, setPageSize] = useState(10)

  const filtered = useMemo(() => {
    if (!data) return []

    let result = data

    if (envFilter !== 'all') {
      result = result.filter(g => g.environment === envFilter)
    }

    if (statusFilter !== 'all') {
      result = result.filter(g => g.status === statusFilter)
    }

    // ... 더 많은 필터링 로직
  }, [data, envFilter, statusFilter, syncFilter])

  const stats = useMemo(() => ({
    total: data?.length || 0,
    approved: filtered.filter(g => g.status === 'approved').length,
    pending: filtered.filter(g => g.status === 'pending').length,
  }), [data, filtered])

  const paginatedGrants = useMemo(() => {
    const start = (currentPage - 1) * pageSize
    return filtered.slice(start, start + pageSize)
  }, [filtered, currentPage, pageSize])

  return {
    filters: { envFilter, statusFilter, syncFilter },
    setEnvFilter,
    setStatusFilter,
    setSyncFilter,
    data: { filtered, stats, paginatedGrants, totalFiltered: filtered.length },
    pagination: { currentPage, totalPages: Math.ceil(filtered.length / pageSize), pageSize },
    setCurrentPage,
    setPageSize
  }
}
```
\`\`\`
```

**문제점**:
- 총 40줄 차지
- 패턴보다 **구현 세부사항**에 집중
- Claude가 코드를 **그대로 복사**할 가능성 높음

### After: 파일 참조 (10줄)

```markdown
### useGrantFilters Hook 패턴

**핵심 개념**: 필터링 + 페이지네이션 + 통계 계산

```tsx
export function useGrantFilters(data: Grant[] | undefined) {
  const [envFilter, setEnvFilter] = useState('all')

  const { filtered, stats } = useMemo(() => {
    // 필터링 + 정렬 로직
  }, [data, envFilter])

  return {
    filters: { envFilter },
    setEnvFilter,
    data: { filtered, stats }
  }
}
```

**실제 구현 참고**: `hooks/use-grant-filters.ts`
**사용 예시**: `pages/api-gateways/GrantsPage.tsx`
```

**장점**:
- 총 15줄 (60% 감축)
- **패턴의 본질**만 전달
- 실제 구현은 **파일을 직접 읽도록** 유도
- Claude가 패턴을 **이해**하고 **응용**

### 효과

```tsx
// Before: Claude가 CLAUDE.md의 코드를 그대로 복사
export function useApplicationFilters(data: Application[] | undefined) {
  // ... 997줄 가이드의 코드를 복사-붙여넣기
}

// After: Claude가 패턴을 이해하고 응용
export function useApplicationFilters(data: Application[] | undefined) {
  const [phaseFilter, setPhaseFilter] = useState<ApplicationPhase | 'all'>('all')

  const { filtered, stats } = useMemo(() => {
    if (!data) return { filtered: [], stats: defaultStats }

    const filtered = phaseFilter === 'all'
      ? data
      : data.filter(app => app.phase === phaseFilter)

    return {
      filtered: sortBy(filtered, 'createdAt', 'desc'),
      stats: calculateApplicationStats(data)
    }
  }, [data, phaseFilter])

  return { filters: { phaseFilter }, setPhaseFilter, data: { filtered, stats } }
}
```

**차이점**:
- Before: 코드 복사 → 기계적
- After: 패턴 이해 → 응용 → 도메인에 맞게 수정

---

## 해결책 3: MCP 서버 강제 활성화

### 문제: Claude가 MCP 도구를 안 쓴다

**[imprun.dev](https://imprun.dev)**는 3개의 MCP 서버를 사용합니다:

```json
// .mcp.json
{
  "mcpServers": {
    "context7": {
      "type": "http",
      "url": "https://mcp.context7.com/mcp",
      "headers": {
        "CONTEXT7_API_KEY": "ctx7sk-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
      }
    },
    "sequential-thinking": {
      "type": "stdio",
      "command": "cmd",
      "args": [
        "/c",
        "npx",
        "-y",
        "@modelcontextprotocol/server-sequential-thinking"
      ],
      "env": {}
    },
    "serena": {
      "type": "stdio",
      "command": "uvx",
      "args": [
        "--from",
        "git+https://github.com/oraios/serena",
        "serena",
        "start-mcp-server",
        "--context",
        "ide-assistant"
      ],
      "env": {}
    }
  }
}
```

**역할**:
- **[Context7](https://context7.com/)**: 최신 라이브러리 공식 문서 검색 (MongoDB, React, Next.js 등)
- **[Serena](https://github.com/oraios/serena)**: 코드베이스 탐색 (파일 찾기, 심볼 검색, 의존성 분석) → [Serena MCP 가이드](https://blog.imprun.dev/29)
- **[Sequential-thinking](https://github.com/modelcontextprotocol/servers/tree/main/src/sequentialthinking)**: 복잡한 설계 계획 (단계별 사고, 문제 해결) → [Sequential Thinking MCP 가이드](https://blog.imprun.dev/30)

**문제**:
```
나: "GrantsPage에서 environment 필터링을 추가해줘"

Claude: (직접 파일을 열고 수정 시도)
  → ❌ 컴포넌트 구조 파악 실패
  → ❌ useGrantFilters Hook 존재 발견 못 함
  → ❌ 엉뚱한 위치에 코드 추가

이유: Serena로 탐색하지 않고 직접 파일을 읽으려 함
```

### 해결책: MCP 강제 활성화 가이드

CLAUDE.md 최상단에 추가:

```markdown
## ⚠️ MCP 서버 활성화 필수

**Serena + Sequential-thinking MCP가 비활성화된 경우**:

```
🚨 사용자에게 즉시 알려주세요:

"Serena MCP와 Sequential-thinking MCP가 필요합니다.
.mcp.json을 확인하고 두 MCP 서버를 활성화해주세요.
활성화 전까지 개발을 진행할 수 없습니다."
```

**필수 사용 시점**:

1. **탐색 단계** (Serena 필수)
   - ✅ 파일 구조 파악
   - ✅ 심볼 검색 (함수, 타입, 컴포넌트)
   - ✅ 의존성 분석

2. **계획 단계** (Sequential-thinking 필수)
   - ✅ 복잡한 기능 설계
   - ✅ 리팩토링 계획
   - ✅ 아키텍처 결정

**MCP 없이 개발 금지**: 탐색/계획 단계 없이 바로 코딩 시작 불가
```

### 효과

```
나: "GrantsPage에서 environment 필터링을 추가해줘"

Claude:
  1. ✅ Serena로 GrantsPage.tsx 탐색
  2. ✅ useGrantFilters Hook 발견
  3. ✅ Sequential-thinking으로 계획:
     - Hook에 envFilter 추가
     - GrantFilters 컴포넌트에 UI 추가
     - 필터링 로직 구현
  4. ✅ 계획대로 정확하게 구현

결과: 1회 시도로 정확한 구현
```

---

## 해결책 4: 우선순위 재구조화

### Before: 중요한 것이 중간에 묻힘

```markdown
1. 기술 스택 (100줄)
2. 아키텍처 패턴 (150줄)
3. 프로젝트 구조 (80줄)
...
8. 스타일링 규칙 (120줄)  ← DESIGN_GUIDELINES.md 여기서 처음 언급
...
```

**문제점**:
- DESIGN_GUIDELINES.md(553줄)가 8번째 섹션에서야 등장
- Container/Presentational 패턴 설명이 너무 길어서 핵심이 묻힘

### After: 최상단에 필수 문서 배치

```markdown
## 📖 필수 문서 (먼저 읽으세요)

### 1. **DESIGN_GUIDELINES.md** (553줄) - UI/UX 디자인 시스템

> **모든 컴포넌트 작성 전에 반드시 참고**

- 타이포그래피 (페이지 헤더, 섹션 제목, 본문)
- 컬러 시스템 (Semantic Colors, 상태 색상)
- 레이아웃 패턴 (페이지 구조, 그리드, 간격)
- 카드 디자인 (기본/호버/선택 상태)
- 버튼 & 인터랙션
- 테이블 디자인
- Empty State, 애니메이션

**디자인 일관성이 최우선입니다.**

### 2. **Container/Presentational Pattern** - 핵심 아키텍처

> **컴포넌트 = Container (로직) + Presentational (UI) + Hooks (비즈니스)**

```
Container (100줄 이하)
  ↓ 데이터 페칭
Hooks (150줄 이하)
  ↓ 비즈니스 로직
Presentational (200줄 이하)
  ↓ UI 렌더링
```

**이 패턴을 따르지 않으면 즉시 리팩토링됩니다.**
```

### 효과

```
나: "Database 관리 페이지를 만들어줘"

Claude: (CLAUDE.md를 읽음)
  → ✅ 첫 줄부터 DESIGN_GUIDELINES.md 참조
  → ✅ Container/Presentational 패턴 즉시 적용
  → ✅ Serena로 비슷한 페이지 (GrantsPage) 탐색
  → ✅ 동일한 디자인 패턴으로 구현

결과: 첫 시도부터 일관성 있는 코드
```

---

## 최종 결과: 997줄 → 593줄 (43% 감축)

### 최적화 타임라인

```
2025-10-27: CLAUDE.md 초안 작성 (997줄)
  → Part 1 블로그 포스팅

2025-10-30: 문제 발견 (패턴 무시)
  → Anthropic Best Practices 분석

2025-11-01: 1차 최적화 (997 → 741줄)
  → 파일 참조 전략 도입

2025-11-02: 2차 최적화 (741 → 537줄)
  → 코드 샘플 간소화

2025-11-02: MCP 추가 (537 → 561줄)
  → MCP 서버 강제 활성화 가이드 추가

2025-11-03: 우선순위 재구조화 (561 → 593줄)
  → DESIGN_GUIDELINES 최상단 배치
  → 필수 문서 섹션 추가
```

### Anthropic Best Practices 재평가 (자체 평가)

> **참고**: 아래 점수는 Anthropic 공식 Best Practices 문서를 참고하여 자체적으로 평가한 추정치입니다.

| 항목 | Before | After | 개선 |
|------|--------|-------|------|
| **파일 참조 전략** | ~60/100 | ~95/100 | +35 |
| **우선순위 명확화** | ~70/100 | ~100/100 | +30 |
| **실제 파일 구조** | ~95/100 | ~95/100 | - |
| **외부 도구 활용** | ~0/100 | ~95/100 | +95 |
| **간결함** | ~80/100 | ~90/100 | +10 |

**총점 (추정): ~81/100 → ~95/100 (B → A+)**

### 정량적 효과

**개발 시 패턴 준수율** (실제 개발 경험):
```
Before (997줄, 코드 샘플 중심):
  첫 시도 패턴 준수: 약 60%
  재작성 후 준수: 약 90%

After (593줄, 파일 참조 + 우선순위):
  첫 시도 패턴 준수: 약 95%
  재작성 필요: 약 5% (거의 없음)
```

**디자인 일관성** (실제 개발 경험):
```
Before (DESIGN_GUIDELINES 중간에 언급):
  디자인 가이드 참조율: 약 40%

After (DESIGN_GUIDELINES 최상단):
  디자인 가이드 참조율: 약 95%
```

**MCP 도구 사용률** (실제 개발 경험):
```
Before (MCP 가이드 없음):
  Serena 사용: 약 20%
  Sequential-thinking 사용: 약 5%

After (MCP 강제 활성화):
  Serena 사용: 약 90%
  Sequential-thinking 사용: 약 70%
```

---

## 핵심 교훈: "AI에게 코드를 주지 말고, 파일 경로를 알려주라"

### 1. 코드 샘플은 독이 될 수 있다

```markdown
❌ 나쁜 예:
"이렇게 작성하세요"
```tsx
export function useGrantFilters(data: Grant[] | undefined) {
  // ... 25줄짜리 완전한 구현
}
```

→ Claude가 복사-붙여넣기만 함

✅ 좋은 예:
"이런 패턴으로 작성하세요"
```tsx
export function useGrantFilters(data: Grant[] | undefined) {
  const [filter, setFilter] = useState('all')
  const { filtered, stats } = useMemo(() => { /* ... */ }, [data, filter])
  return { filters: { filter }, setFilter, data: { filtered, stats } }
}
```

**실제 구현**: `hooks/use-grant-filters.ts`

→ Claude가 패턴을 이해하고 응용
```

### 2. 우선순위를 최상단에

```markdown
❌ Before: 8번째 섹션에서 DESIGN_GUIDELINES.md 언급
  → Claude가 디자인 가이드를 읽지 않음

✅ After: 첫 번째 섹션에 "필수 문서" 배치
  → Claude가 항상 디자인 가이드를 먼저 확인
```

**원칙**: 가장 중요한 것부터 보여주기

### 3. 외부 도구는 강제해야 한다

```markdown
❌ Before: "Serena를 사용하면 좋습니다"
  → Claude가 가끔만 사용

✅ After: "Serena 없이는 개발 불가, 비활성화 시 즉시 사용자에게 알림"
  → Claude가 항상 사용
```

**원칙**: "권장"이 아닌 "필수"로 명시

### 4. 간결함 > 완벽함

```markdown
997줄 완벽한 가이드 (패턴 준수율 60%)
  <
593줄 간결한 가이드 (패턴 준수율 95%)
```

**원칙**: AI가 읽고 기억할 수 있는 분량으로

---

## DESIGN_GUIDELINES.md 통합 논쟁

### 논쟁: 553줄 디자인 가이드를 CLAUDE.md에 통합?

**찬성 의견**:
- ✅ 하나의 문서에 모든 규칙
- ✅ Claude가 참조하기 쉬움

**반대 의견**:
- ❌ 1,146줄 (593 + 553) → 너무 길어짐
- ❌ 아키텍처와 디자인은 다른 관심사
- ❌ AI의 컨텍스트 한계 초과 가능

### 결론: 별도 유지 + 최상단 참조

```markdown
## 📖 필수 문서 (먼저 읽으세요)

### 1. **DESIGN_GUIDELINES.md** (553줄) - UI/UX 디자인 시스템

> **모든 컴포넌트 작성 전에 반드시 참고**

[링크와 핵심 요약]
```

**이유**:
- CLAUDE.md는 **아키텍처 + 개발 워크플로우**에 집중
- DESIGN_GUIDELINES.md는 **UI/UX 디자인 시스템**에 집중
- 최상단 참조로 **Claude가 항상 디자인 가이드를 먼저 확인**
- 관심사 분리 (Separation of Concerns) 유지

**효과**:
```
나: "Database 관리 페이지를 만들어줘"

Claude:
  1. ✅ CLAUDE.md 읽음 → "필수 문서" 섹션 확인
  2. ✅ DESIGN_GUIDELINES.md 읽음 → 디자인 패턴 파악
  3. ✅ Container/Presentational 패턴 적용
  4. ✅ 디자인 가이드 준수하며 구현

결과: 아키텍처 + 디자인 모두 일관성 유지
```

---

## 실전 사례: Environment 기반 배포 관리 UI

### 요구사항

```
"Environment(dev/staging/prod) 기반 배포 관리 UI를 추가해줘.
각 환경별로 배포 상태를 보여주고, 환경 간 코드 비교(Diff)와
배포 승격(Promote) 기능도 필요해."
```

### Before (997줄 가이드): 3번 재작성

```
시도 1:
  ❌ useState로 환경 목록 관리 (TanStack Query 무시)
  ❌ 컴포넌트에 비즈니스 로직 혼재

나: "CLAUDE.md 패턴으로 다시 작성해줘"

시도 2:
  ✅ TanStack Query 사용
  ❌ 하지만 Hook 분리 안 함 (useStages에 모든 로직)
  ❌ Presentational 컴포넌트 없음

나: "Container/Presentational 패턴으로 분리해줘"

시도 3:
  ✅ 패턴 준수
  ✅ Hook 분리
  ❌ 하지만 디자인이 다른 페이지와 다름 (margin 사용)

나: "DESIGN_GUIDELINES.md 참고해서 스타일 수정"

시도 4:
  ✅ 최종 완성

소요 시간: 2시간
```

### After (593줄 가이드): 1번에 완성

```
나: "Environment 기반 배포 관리 UI를 추가해줘"

Claude:
  1. ✅ CLAUDE.md 읽음 → MCP 서버 사용 확인
  2. ✅ Serena로 비슷한 페이지 탐색 (GrantsPage 발견)
  3. ✅ DESIGN_GUIDELINES.md 참고
  4. ✅ Sequential-thinking으로 계획:
     - DeploymentStatusDto 타입 추가
     - function.service.ts에 API 추가
     - useStages Hook에 deployment 로직 추가
     - DeploymentStepperCard 컴포넌트 생성 (Presentational)
     - FunctionDeployments 컴포넌트 생성 (Container)
  5. ✅ 계획대로 구현

결과:
  ✅ 첫 시도부터 패턴 준수
  ✅ 디자인 일관성 유지
  ✅ GrantsPage와 동일한 구조

소요 시간: 30분
```

**생성된 파일**:
```
frontend/src/types/function.ts (TDeploymentStatus 타입)
frontend/src/services/function.service.ts (getDeploymentStatus API)
frontend/src/hooks/use-stages.ts (deployment 로직 추가)
frontend/src/components/function-detail/DeploymentStepperCard.tsx (Presentational)
frontend/src/components/function-detail/FunctionDeployments.tsx (Container)
```

**개선 효과**:
- 개발 시간: 2시간 → 30분 (75% 단축)
- 재작성: 3회 → 0회
- 첫 시도 완성도: 40% → 95%

---

## 다른 프로젝트에 적용하기

### 1. Anthropic Best Practices 체크리스트

```markdown
[ ] 파일 참조 전략
  - 코드 샘플: 10줄 이하 (패턴만)
  - 실제 구현: "파일 경로 참고" 방식

[ ] 우선순위 명확화
  - 최상단에 "필수 문서/패턴" 섹션
  - 중요도에 따라 ⭐ 표시

[ ] 실제 파일 구조 반영
  - 예시 파일 = 실제 존재하는 파일
  - 디렉터리 구조 정확히 표시

[ ] 외부 도구 활용
  - MCP 서버, 린터, 포매터 명시
  - "필수" vs "권장" 구분

[ ] 간결함 유지
  - 목표: 600줄 이하
  - 상세 내용은 별도 문서
```

### 2. 최적화 과정

```
1단계: 코드 샘플 간소화
  - 25줄 → 10줄 (패턴 본질만)
  - 실제 구현은 파일 참조

2단계: 우선순위 재구조화
  - 최상단에 핵심 패턴 배치
  - "필수 문서" 섹션 추가

3단계: MCP/도구 가이드 추가
  - 필수 도구 명시
  - 비활성화 시 경고 문구

4단계: 간결성 검토
  - 중복 제거
  - 링크로 대체 가능한 것은 링크로
```

### 3. 측정 지표

**최적화 전후 비교**:
```
첫 시도 패턴 준수율:
  Before: 60%
  After: 95%

재작성 빈도:
  Before: 3회/기능
  After: 0.2회/기능

개발 시간:
  Before: 2시간/기능
  After: 30분/기능
```

---

## 한계와 향후 개선 방향

### 현재 한계

1. **AI는 여전히 완벽하지 않음**
   - 95% 정확도 = 5%는 여전히 실수
   - 복잡한 비즈니스 로직은 사람이 검증 필요

2. **프로젝트 성숙도에 따라 효과 다름**
   - 초기: 가이드 작성 비용 > 효과
   - 중기: 가이드 효과 극대화
   - 후기: 가이드 유지보수 비용 발생

3. **팀 규모에 따라 다름**
   - 1인 프로젝트: 오버엔지니어링 가능성
   - 5인 이상 팀: 필수

### 향후 개선 방향

**1. 자동화된 패턴 검증**
```typescript
// .claude/hooks/pre-commit.ts
export function validatePattern(code: string): ValidationResult {
  // Container/Presentational 패턴 자동 검증
  // TanStack Query 사용 여부 확인
  // DESIGN_GUIDELINES 준수 여부 확인
}
```

**2. 컴포넌트 템플릿 생성기**
```bash
$ pnpm generate:page Database
  → pages/DatabasePage.tsx (Container)
  → components/database/DatabaseList.tsx (Presentational)
  → hooks/use-databases.ts (Data Hook)
  → services/database.service.ts (API Service)
  → types/database.ts (TypeScript Types)
```

**3. AI 기반 코드 리뷰 자동화**
```
PR 생성 시:
  → Claude가 CLAUDE.md 체크리스트로 자동 리뷰
  → 패턴 위반 항목 코멘트
  → 수정 제안 코드 생성
```

---

## 마무리

### 핵심 요약

> **"AI에게 완벽한 코드를 보여주지 말고, 완벽한 패턴을 알려주라"**

**997줄 → 593줄 (43% 감축)** 과정에서 배운 4가지:
1. **코드보다 파일 경로**: 25줄 코드 < 10줄 패턴 + 파일 참조
2. **우선순위가 전부**: 최상단에 핵심 배치
3. **도구는 강제해야**: "권장" < "필수"
4. **간결함 > 완벽함**: 600줄 간결한 가이드 > 1,000줄 완벽한 가이드

### 언제 사용하나?

**CLAUDE.md 최적화 권장:**
- ✅ AI가 패턴을 60% 이하로 따를 때
- ✅ 가이드가 800줄 이상일 때
- ✅ MCP 도구를 제공했는데 안 쓸 때
- ✅ 매번 재작성을 요청하는 경우

**Anthropic Best Practices 적용 권장:**
- ✅ 새 프로젝트에서 CLAUDE.md를 작성할 때
- ✅ 기존 가이드의 효과가 떨어질 때
- ✅ 팀 규모가 5인 이상일 때

### 실제 적용 결과

**[imprun.dev](https://imprun.dev) 환경:**
- ✅ CLAUDE.md: 997줄 → 593줄 (43% 감축)
- ✅ 패턴 준수율: 약 60% → 95% (실제 경험)
- ✅ 개발 시간: 약 2시간 → 30분 (약 75% 단축, 실제 경험)
- ✅ Anthropic Best Practices: 자체 평가 약 81/100 → 95/100 (추정)

**운영 경험:**
- 최적화 시간: 약 4시간 (3단계 최적화)
- ROI: 첫 주부터 개발 시간 단축 체감
- 만족도: 매우 높음 😊 (재작성 거의 없음)

---

## 전체 가이드 공개

**📄 최적화된 CLAUDE.md**: [frontend/CLAUDE.md](../../frontend/CLAUDE.md)

**주요 변경사항**:
- 997줄 → 593줄 (43% 감축)
- 파일 참조 전략 도입
- MCP 서버 강제 활성화 가이드 추가
- DESIGN_GUIDELINES 최상단 배치
- Anthropic Best Practices 95/100 달성

**📖 DESIGN_GUIDELINES.md**: [frontend/DESIGN_GUIDELINES.md](../../frontend/DESIGN_GUIDELINES.md) (553줄)

**📝 Part 1**: [Claude AI와 함께하는 프론트엔드 개발](https://blog.imprun.dev/42)

---

## 관련 글

### AI 개발 가이드
- [Claude AI와 함께하는 프론트엔드 개발 (Part 1)](https://blog.imprun.dev/42)
- [Serena MCP: AI 코딩 어시스턴트를 위한 시맨틱 코드 분석 도구](https://blog.imprun.dev/29)
- [Sequential Thinking MCP: AI의 구조화된 사고 프로세스](https://blog.imprun.dev/30)

### Frontend 아키텍처
- [Next.js를 버리고 순수 React로 돌아온 이유](https://blog.imprun.dev/41)
- [Monaco Editor "TextModel got disposed" 에러 완벽 해결 가이드](https://blog.imprun.dev/35)

### imprun 플랫폼
- [imprun Platform 아키텍처: API 개발부터 AI 통합까지](https://blog.imprun.dev/50)
- [API Platform의 Consumer 인증 설계: Application-Grant 아키텍처](https://blog.imprun.dev/49)

---

---

**태그:** Claude, AI, CLAUDE.md, 최적화, Anthropic Best Practices, MCP, 파일참조전략, 개발가이드


---

> "일관성은 공짜가 아니다. AI에게 코드를 주지 말고, 패턴을 알려주라."

🤖 *이 블로그는 [imprun.dev](https://imprun.dev) 플랫폼 구축 과정에서 CLAUDE.md 최적화를 진행한 실제 경험을 바탕으로 작성되었습니다.*

---

**질문이나 피드백은 블로그 댓글에 남겨주세요!**
