# Frontend 컴포넌트 배치 완전 정복: Flexbox부터 Grid까지

**작성일:** 2025-11-08
**카테고리:** Frontend, React, Tailwind CSS, Layout, UI/UX
**난이도:** 초급/중급

---

## TL;DR

- **문제**: 컴포넌트 배치에서 margin 남용, 불필요한 중첩, 반응형 미흡으로 유지보수 어려움
- **해결**: Flexbox `flex + gap` 패턴과 Grid를 활용한 체계적 레이아웃 시스템
- **핵심**: "margin을 버리고 gap을 택하라" - 간격은 부모가 관리
- **결과**: 코드 약 40% 감소, 다크모드 자동 지원, 반응형 대응 용이

---

## 들어가며

[**imprun.dev**](https://imprun.dev)는 "API 개발부터 AI 통합까지, 모든 것을 하나로 제공"하는 Kubernetes 기반 API 플랫폼입니다.

프론트엔드 개발에서 **컴포넌트 배치는 단순해 보이지만 가장 어려운 과제** 중 하나입니다. 초보 개발자는 물론 경험 많은 개발자도 다음과 같은 문제에 자주 직면합니다:

**우리가 마주한 질문**:
- ❓ `margin`을 써야 할까, `padding`을 써야 할까?
- ❓ Flexbox와 Grid 중 언제 무엇을 써야 할까?
- ❓ 반응형 디자인은 어떻게 구현할까?
- ❓ 다크모드에서도 잘 보이려면?

**검증 과정**:

1. **margin 기반 레이아웃**
   - ✅ 직관적이고 익숙함
   - ❌ 간격 관리가 분산됨 (`mb-4`, `mt-2` 등 중복)
   - ❌ 부모-자식 간 책임이 모호함

2. **space-y/space-x 유틸리티**
   - ✅ Tailwind 기본 제공
   - ❌ 첫/마지막 요소 예외 처리 필요
   - ❌ 조건부 렌더링 시 간격 깨짐

3. **flex + gap 패턴** ← **최종 선택**
   - ✅ 간격이 부모에서 중앙 집중 관리
   - ✅ 조건부 렌더링에도 안전
   - ✅ 코드 가독성 향상

**결론**:
- ✅ `flex + gap` 패턴으로 통일
- ✅ Grid는 카드 리스트 등 격자 배치에만 사용
- ✅ margin은 완전히 배제

이 글은 **[imprun.dev](https://imprun.dev) 프론트엔드 구축 경험**을 바탕으로, React + Tailwind CSS v4 환경에서 컴포넌트 배치를 마스터하는 방법을 공유합니다.

---

## 1. 왜 컴포넌트 배치가 중요한가?

### 배치가 결정하는 3가지

**1. 유지보수성**
- 나쁜 배치: 100줄 컴포넌트가 200줄로 비대해짐
- 좋은 배치: 간결하고 예측 가능한 코드

**2. 사용자 경험**
- 나쁜 배치: 모바일에서 깨지고 다크모드에서 안 보임
- 좋은 배치: 모든 환경에서 일관된 경험

**3. 팀 협업**
- 나쁜 배치: 매번 다른 개발자가 다른 패턴 사용
- 좋은 배치: 디자인 시스템으로 통일된 코드

---

## 2. Flexbox 완전 정복

### 2.1 Flexbox 기본 개념

Flexbox는 **1차원 레이아웃 시스템**입니다. 행(row) 또는 열(column) 방향으로 요소를 배치합니다.

```tsx
// 기본 구조
<div className="flex">
  {/* 자식 요소들이 행(row) 방향으로 배치됨 */}
  <div>Item 1</div>
  <div>Item 2</div>
  <div>Item 3</div>
</div>
```

### 2.2 flex-direction: 방향 설정

```tsx
// 행(row) - 기본값 (좌→우)
<div className="flex flex-row">
  <div>1</div>
  <div>2</div>
  <div>3</div>
</div>
// 결과: [1] [2] [3]

// 열(column) - 위→아래
<div className="flex flex-col">
  <div>1</div>
  <div>2</div>
  <div>3</div>
</div>
// 결과:
// [1]
// [2]
// [3]
```

### 2.3 justify-content: 주축 정렬

주축(main axis)은 `flex-direction`이 결정합니다.

```tsx
// 시작점 정렬 (기본값)
<div className="flex justify-start">
  <div>A</div>
  <div>B</div>
</div>
// 결과: [A][B]________

// 끝점 정렬
<div className="flex justify-end">
  <div>A</div>
  <div>B</div>
</div>
// 결과: ________[A][B]

// 중앙 정렬
<div className="flex justify-center">
  <div>A</div>
  <div>B</div>
</div>
// 결과: ____[A][B]____

// 양끝 정렬 (사이 공간 균등 분배)
<div className="flex justify-between">
  <div>A</div>
  <div>B</div>
  <div>C</div>
</div>
// 결과: [A]____[B]____[C]
```

### 2.4 align-items: 교차축 정렬

교차축(cross axis)은 주축에 수직입니다.

```tsx
// 상단 정렬
<div className="flex items-start h-32">
  <div className="h-8">Short</div>
  <div className="h-16">Tall</div>
</div>

// 중앙 정렬 (가장 많이 사용)
<div className="flex items-center h-32">
  <div className="h-8">Short</div>
  <div className="h-16">Tall</div>
</div>

// 하단 정렬
<div className="flex items-end h-32">
  <div className="h-8">Short</div>
  <div className="h-16">Tall</div>
</div>

// 늘려서 채우기
<div className="flex items-stretch h-32">
  <div>Stretched to parent height</div>
</div>
```

### 2.5 gap: imprun.dev의 핵심 패턴

**핵심 원칙**: margin 대신 gap 사용

```tsx
// ❌ BAD: margin 사용
<div className="flex">
  <div className="mr-2">Item 1</div>
  <div className="mr-2">Item 2</div>
  <div>Item 3</div> {/* 마지막은 mr 없음 */}
</div>

// ✅ GOOD: gap 사용
<div className="flex gap-2">
  <div>Item 1</div>
  <div>Item 2</div>
  <div>Item 3</div>
</div>
```

**gap의 장점**:
1. **간격 중앙 관리**: 부모에서 한 번만 설정
2. **조건부 렌더링 안전**: 요소가 없으면 gap도 자동으로 제거
3. **코드 간결성**: 자식마다 margin 설정 불필요

### 2.6 imprun.dev 표준 간격 시스템

```tsx
// gap-2 (0.5rem = 8px) - 매우 좁은 간격
<div className="flex gap-2">
  <Icon className="h-4 w-4" />
  <span>텍스트</span>
</div>

// gap-4 (1rem = 16px) - 카드 내부 요소
<CardContent className="flex flex-col gap-4">
  <div>Section 1</div>
  <div>Section 2</div>
</CardContent>

// gap-6 (1.5rem = 24px) - 그리드 카드 간격
<div className="grid grid-cols-3 gap-6">
  <Card />
  <Card />
  <Card />
</div>

// gap-8 (2rem = 32px) - 페이지 주요 섹션
<div className="flex flex-col gap-8">
  <Header />
  <Content />
  <Footer />
</div>
```

---

## 3. imprun.dev 실전 패턴

### 3.1 페이지 레이아웃 패턴

**GrantsPage.tsx의 완벽한 구조** (89줄):

```tsx
export default function GrantsPage() {
  const { gatewayId } = useParams()
  const { data: grants, isLoading } = useGrants(gatewayId!)
  const filters = useGrantFilters(grants)
  const actions = useGrantActions(gatewayId!)

  if (isLoading) return <LoadingSpinner />

  return (
    // 1. 최상위: 전체 높이 + 스크롤
    <div className="p-6 h-full overflow-auto">
      {/* 2. 컨테이너: 최대 너비 + 중앙 정렬 + 주요 섹션 간격 */}
      <div className="max-w-7xl mx-auto flex flex-col gap-6">
        {/* 3. Header: 제목 + 액션 버튼 */}
        <GrantsPageHeader />

        {/* 4. Filters: 필터 UI */}
        <GrantFilters {...filters.filters} />

        {/* 5. Stats: 통계 정보 */}
        <GrantStats stats={filters.data.stats} />

        {/* 6. Table: 메인 컨텐츠 */}
        <GrantTable grants={filters.data.paginatedGrants} />

        {/* 7. Pagination: 페이지네이션 */}
        <Pagination {...filters.pagination} />
      </div>
    </div>
  )
}
```

**구조 분석**:
- `p-6`: 페이지 전체 패딩 (모바일: `p-4`, 데스크탑: `p-6` 또는 `p-8`)
- `h-full overflow-auto`: 전체 높이 사용 + 내용 많으면 스크롤
- `max-w-7xl mx-auto`: 최대 1280px 너비 + 중앙 정렬
- `flex flex-col gap-6`: 세로 배치 + 섹션 간 24px 간격

### 3.2 페이지 헤더 패턴

```tsx
// 제목 + 설명 + 액션 버튼
<div className="flex items-center justify-between">
  {/* 좌측: 제목 + 설명 */}
  <div className="flex flex-col gap-2">
    <h1 className="text-4xl font-bold tracking-tight bg-gradient-to-r from-foreground to-foreground/70 bg-clip-text text-transparent">
      접근 관리
    </h1>
    <p className="text-sm text-muted-foreground">
      Function-level 접근 권한을 관리합니다
    </p>
  </div>

  {/* 우측: 액션 버튼 */}
  <Button size="lg" className="gap-2">
    <Plus className="h-4 w-4" />
    새로 만들기
  </Button>
</div>
```

**핵심 테크닉**:
- `justify-between`: 제목은 왼쪽, 버튼은 오른쪽
- `items-center`: 수직 중앙 정렬
- `flex flex-col gap-2`: 제목과 설명 사이 8px 간격
- 그라디언트 텍스트: 시각적 강조 효과

### 3.3 필터 레이아웃 패턴

**GrantFilters.tsx의 3컬럼 필터**:

```tsx
<Card>
  <CardHeader className="cursor-pointer" onClick={onToggleCollapse}>
    <div className="flex items-center justify-between">
      <div className="flex items-center gap-2">
        <Filter className="h-4 w-4" />
        <CardTitle className="text-base">필터</CardTitle>
      </div>
      <Button variant="ghost" size="sm">
        {collapsed ? '펼치기' : '접기'}
      </Button>
    </div>
  </CardHeader>

  {!collapsed && (
    <CardContent>
      {/* 3컬럼 그리드 */}
      <div className="grid grid-cols-3 gap-4">
        {/* 환경 필터 */}
        <div className="space-y-2">
          <label className="text-sm font-medium">환경</label>
          <Select value={envFilter} onValueChange={onEnvChange}>
            <SelectTrigger>
              <SelectValue />
            </SelectTrigger>
            <SelectContent>
              <SelectItem value="all">전체 환경</SelectItem>
              <SelectItem value="dev">Development</SelectItem>
              <SelectItem value="staging">Staging</SelectItem>
              <SelectItem value="prod">Production</SelectItem>
            </SelectContent>
          </Select>
        </div>

        {/* 승인 상태 필터 */}
        <div className="space-y-2">
          <label className="text-sm font-medium">승인 상태</label>
          <Select value={statusFilter} onValueChange={onStatusChange}>
            {/* ... */}
          </Select>
        </div>

        {/* 동기화 상태 필터 */}
        <div className="space-y-2">
          <label className="text-sm font-medium">동기화 상태</label>
          <Select value={syncFilter} onValueChange={onSyncChange}>
            {/* ... */}
          </Select>
        </div>
      </div>
    </CardContent>
  )}
</Card>
```

**패턴 분석**:
- `grid grid-cols-3 gap-4`: 3컬럼 그리드 + 16px 간격
- `space-y-2`: 라벨과 입력 필드 사이 8px 간격 (flex-col + gap-2와 동일)
- 조건부 렌더링 (`{!collapsed && ...}`): 접기/펼치기 구현

### 3.4 카드 리스트 그리드 패턴

```tsx
// 반응형 3컬럼 그리드
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
  {gateways.map(gateway => (
    <Link key={gateway.id} to={`/gateways/${gateway.id}`} className="group">
      <Card className="h-full transition-all duration-300 hover:shadow-2xl hover:-translate-y-1 hover:border-primary/50 border-2">
        <CardHeader className="space-y-4">
          {/* 아이콘 + 제목 */}
          <div className="flex items-start gap-4">
            {/* 그라디언트 아이콘 박스 */}
            <div className="flex-shrink-0 p-3 rounded-xl bg-gradient-to-br from-blue-500 to-cyan-500 shadow-lg group-hover:scale-110 transition-transform duration-300">
              <Code2 className="h-6 w-6 text-white" />
            </div>

            {/* 제목 + 설명 */}
            <div className="flex-1 min-w-0 space-y-1">
              <CardTitle className="text-lg truncate group-hover:text-primary transition-colors">
                {gateway.name}
              </CardTitle>
              <CardDescription className="text-xs truncate">
                {gateway.id}
              </CardDescription>
            </div>

            {/* 화살표 아이콘 */}
            <ArrowRight className="h-4 w-4 text-muted-foreground group-hover:text-primary group-hover:translate-x-1 transition-all duration-300" />
          </div>
        </CardHeader>
      </Card>
    </Link>
  ))}
</div>
```

**핵심 테크닉**:
- `grid-cols-1 md:grid-cols-2 lg:grid-cols-3`: 모바일 1열 → 태블릿 2열 → 데스크탑 3열
- `h-full`: 카드 높이를 그리드 행 높이에 맞춤 (같은 높이 유지)
- `flex-shrink-0`: 아이콘 박스가 줄어들지 않도록 고정
- `flex-1 min-w-0`: 제목이 남은 공간을 차지하고 truncate 가능하게
- `group-hover:`: 부모 호버 시 자식 요소 스타일 변경

---

## 4. Grid 완전 정복

### 4.1 Grid vs Flexbox: 언제 무엇을 쓸까?

```tsx
// ✅ Flexbox 사용 (1차원 레이아웃)
// - 행 또는 열 하나만 필요할 때
// - 유동적인 크기의 요소들
<div className="flex flex-col gap-4">
  <Header />
  <Content />
  <Footer />
</div>

// ✅ Grid 사용 (2차원 레이아웃)
// - 행과 열 모두 필요할 때
// - 균등한 크기의 카드 리스트
<div className="grid grid-cols-3 gap-6">
  <Card />
  <Card />
  <Card />
</div>
```

### 4.2 Grid 기본 패턴

```tsx
// 1. 고정 3컬럼
<div className="grid grid-cols-3 gap-4">
  <div>1</div>
  <div>2</div>
  <div>3</div>
  <div>4</div>
  <div>5</div>
  <div>6</div>
</div>
// 결과:
// [1] [2] [3]
// [4] [5] [6]

// 2. 반응형 컬럼 (imprun.dev 표준)
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
  {/* 모바일: 1열, 태블릿: 2열, 데스크탑: 3열 */}
</div>

// 3. 자동 채우기 (auto-fit)
<div className="grid grid-cols-[repeat(auto-fit,minmax(300px,1fr))] gap-6">
  {/* 최소 300px, 공간이 허락하면 자동으로 열 추가 */}
</div>

// 4. 비율 조정 (2:1:1)
<div className="grid grid-cols-[2fr,1fr,1fr] gap-4">
  <div>Wide (2배)</div>
  <div>Normal</div>
  <div>Normal</div>
</div>
```

### 4.3 Grid 실전 패턴

```tsx
// Dashboard 레이아웃 (2x2 그리드)
<div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
  {/* 큰 차트 (2열 차지) */}
  <Card className="lg:col-span-2">
    <CardHeader>
      <CardTitle>주요 지표</CardTitle>
    </CardHeader>
    <CardContent>
      {/* 차트 */}
    </CardContent>
  </Card>

  {/* 작은 위젯들 */}
  <Card>
    <CardHeader>
      <CardTitle>API 호출</CardTitle>
    </CardHeader>
  </Card>

  <Card>
    <CardHeader>
      <CardTitle>오류율</CardTitle>
    </CardHeader>
  </Card>
</div>
```

---

## 5. 일반적인 함정과 해결책

### 5.1 margin 남용

```tsx
// ❌ BAD: margin으로 간격 관리
<div>
  <div className="mb-4">Item 1</div>
  <div className="mb-4">Item 2</div>
  <div className="mb-4">Item 3</div>
  {showExtra && <div className="mb-4">Extra</div>}
  <div>Item 4</div> {/* 마지막은 mb 없음! */}
</div>

// ✅ GOOD: flex + gap으로 간격 관리
<div className="flex flex-col gap-4">
  <div>Item 1</div>
  <div>Item 2</div>
  <div>Item 3</div>
  {showExtra && <div>Extra</div>} {/* 조건부 렌더링 안전 */}
  <div>Item 4</div>
</div>
```

### 5.2 space-y의 함정

```tsx
// ❌ RISKY: space-y는 첫 번째 요소에 영향 없음
<div className="space-y-4">
  {items.length === 0 && <EmptyState />}
  {items.map(item => <Item key={item.id} />)}
</div>
// 문제: items가 비어있으면 EmptyState와 첫 Item 사이 간격 없음

// ✅ SAFE: flex + gap 사용
<div className="flex flex-col gap-4">
  {items.length === 0 && <EmptyState />}
  {items.map(item => <Item key={item.id} />)}
</div>
```

### 5.3 아이콘과 텍스트 정렬

```tsx
// ❌ BAD: 세로 정렬 안 됨
<div className="flex">
  <Icon className="h-4 w-4" />
  <span>텍스트</span>
</div>

// ✅ GOOD: items-center로 중앙 정렬 + gap으로 간격
<div className="flex items-center gap-2">
  <Icon className="h-4 w-4" />
  <span>텍스트</span>
</div>
```

### 5.4 카드 높이 불일치

```tsx
// ❌ BAD: 카드 높이가 제각각
<div className="grid grid-cols-3 gap-6">
  <Card>
    <CardContent>짧은 내용</CardContent>
  </Card>
  <Card>
    <CardContent>
      긴 내용<br />
      여러 줄<br />
      매우 긴 내용
    </CardContent>
  </Card>
  <Card>
    <CardContent>중간 내용</CardContent>
  </Card>
</div>

// ✅ GOOD: h-full로 높이 통일
<div className="grid grid-cols-3 gap-6">
  <Card className="h-full">
    <CardContent>짧은 내용</CardContent>
  </Card>
  <Card className="h-full">
    <CardContent>
      긴 내용<br />
      여러 줄<br />
      매우 긴 내용
    </CardContent>
  </Card>
  <Card className="h-full">
    <CardContent>중간 내용</CardContent>
  </Card>
</div>
```

### 5.5 반응형 미흡

```tsx
// ❌ BAD: 모바일에서 깨짐
<div className="flex gap-4">
  <div className="w-1/3">좁은 사이드바</div>
  <div className="w-2/3">넓은 메인</div>
</div>

// ✅ GOOD: 모바일은 세로 배치
<div className="flex flex-col lg:flex-row gap-4">
  <div className="lg:w-1/3">사이드바</div>
  <div className="lg:w-2/3">메인</div>
</div>
```

---

## 6. 실전 예제: 단계별 구현

### 예제 1: API Gateway 카드 리스트

**Step 1: 기본 구조 (페이지 컨테이너)**

```tsx
export default function GatewaysPage() {
  return (
    <div className="p-6 h-full overflow-auto">
      <div className="max-w-7xl mx-auto flex flex-col gap-8">
        {/* 컨텐츠가 여기 들어갈 예정 */}
      </div>
    </div>
  )
}
```

**Step 2: 헤더 추가**

```tsx
export default function GatewaysPage() {
  return (
    <div className="p-6 h-full overflow-auto">
      <div className="max-w-7xl mx-auto flex flex-col gap-8">
        {/* 헤더 */}
        <div className="flex items-center justify-between">
          <div className="flex flex-col gap-2">
            <h1 className="text-4xl font-bold">API Gateways</h1>
            <p className="text-sm text-muted-foreground">
              API Gateway를 생성하고 관리합니다
            </p>
          </div>
          <Button size="lg" className="gap-2" onClick={onCreate}>
            <Plus className="h-4 w-4" />
            새 Gateway 만들기
          </Button>
        </div>
      </div>
    </div>
  )
}
```

**Step 3: 카드 그리드 추가**

```tsx
export default function GatewaysPage() {
  const { data: gateways } = useGateways()

  return (
    <div className="p-6 h-full overflow-auto">
      <div className="max-w-7xl mx-auto flex flex-col gap-8">
        {/* 헤더 */}
        <div className="flex items-center justify-between">
          <div className="flex flex-col gap-2">
            <h1 className="text-4xl font-bold">API Gateways</h1>
            <p className="text-sm text-muted-foreground">
              API Gateway를 생성하고 관리합니다
            </p>
          </div>
          <Button size="lg" className="gap-2" onClick={onCreate}>
            <Plus className="h-4 w-4" />
            새 Gateway 만들기
          </Button>
        </div>

        {/* 카드 그리드 */}
        <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
          {gateways?.map(gateway => (
            <GatewayCard key={gateway.id} gateway={gateway} />
          ))}
        </div>
      </div>
    </div>
  )
}
```

**Step 4: 카드 컴포넌트 구현**

```tsx
interface GatewayCardProps {
  gateway: Gateway
}

function GatewayCard({ gateway }: GatewayCardProps) {
  return (
    <Link to={`/gateways/${gateway.id}`} className="group">
      <Card className="h-full border-2 hover:shadow-2xl hover:-translate-y-1 hover:border-primary/50 transition-all duration-300">
        <CardHeader className="space-y-4">
          {/* 아이콘 + 제목 + 화살표 */}
          <div className="flex items-start gap-4">
            {/* 그라디언트 아이콘 */}
            <div className="flex-shrink-0 p-3 rounded-xl bg-gradient-to-br from-blue-500 to-cyan-500 shadow-lg group-hover:scale-110 transition-transform duration-300">
              <Code2 className="h-6 w-6 text-white" />
            </div>

            {/* 제목 + ID */}
            <div className="flex-1 min-w-0 space-y-1">
              <CardTitle className="text-lg truncate group-hover:text-primary transition-colors">
                {gateway.name}
              </CardTitle>
              <CardDescription className="text-xs truncate">
                {gateway.id}
              </CardDescription>
            </div>

            {/* 화살표 */}
            <ArrowRight className="h-4 w-4 text-muted-foreground group-hover:text-primary group-hover:translate-x-1 transition-all duration-300" />
          </div>

          {/* 설명 */}
          <p className="text-sm text-muted-foreground leading-relaxed">
            {gateway.description}
          </p>

          {/* 환경 배지 */}
          <div className="flex items-center gap-2">
            <span className="px-2 py-1 text-xs font-medium rounded-full bg-green-100 text-green-800 dark:bg-green-900/30 dark:text-green-400">
              Dev
            </span>
            <span className="px-2 py-1 text-xs font-medium rounded-full bg-yellow-100 text-yellow-800 dark:bg-yellow-900/30 dark:text-yellow-400">
              Staging
            </span>
            <span className="px-2 py-1 text-xs font-medium rounded-full bg-blue-100 text-blue-800 dark:bg-blue-900/30 dark:text-blue-400">
              Prod
            </span>
          </div>
        </CardHeader>
      </Card>
    </Link>
  )
}
```

---

### 예제 2: 필터 + 테이블 레이아웃

```tsx
export default function FunctionsPage() {
  const { gatewayId } = useParams()
  const { data: functions } = useFunctions(gatewayId!)
  const [envFilter, setEnvFilter] = useState('all')

  return (
    <div className="p-6 h-full overflow-auto">
      <div className="max-w-7xl mx-auto flex flex-col gap-6">
        {/* 헤더 */}
        <div className="flex items-center justify-between">
          <h1 className="text-2xl font-bold">Functions</h1>
          <Button onClick={onCreate}>
            <Plus className="h-4 w-4" />
            새 Function
          </Button>
        </div>

        {/* 필터 */}
        <Card>
          <CardContent className="pt-6">
            <div className="flex items-center gap-4">
              <label className="text-sm font-medium">환경</label>
              <Select value={envFilter} onValueChange={setEnvFilter}>
                <SelectTrigger className="w-48">
                  <SelectValue />
                </SelectTrigger>
                <SelectContent>
                  <SelectItem value="all">전체 환경</SelectItem>
                  <SelectItem value="dev">Dev</SelectItem>
                  <SelectItem value="staging">Staging</SelectItem>
                  <SelectItem value="prod">Prod</SelectItem>
                </SelectContent>
              </Select>
            </div>
          </CardContent>
        </Card>

        {/* 테이블 */}
        <div className="bg-card border-2 rounded-xl overflow-hidden">
          <table className="w-full">
            <thead className="bg-muted/30 border-b-2">
              <tr>
                <th className="px-6 py-4 text-left text-xs font-semibold uppercase">
                  Name
                </th>
                <th className="px-6 py-4 text-left text-xs font-semibold uppercase">
                  Environment
                </th>
                <th className="px-6 py-4 text-left text-xs font-semibold uppercase">
                  Status
                </th>
                <th className="px-6 py-4 text-right text-xs font-semibold uppercase">
                  Actions
                </th>
              </tr>
            </thead>
            <tbody className="divide-y divide-border">
              {functions?.map(fn => (
                <tr key={fn.id} className="hover:bg-muted/50 transition-colors">
                  <td className="px-6 py-4">
                    <span className="font-medium">{fn.name}</span>
                  </td>
                  <td className="px-6 py-4">
                    <span className="text-sm text-muted-foreground">
                      {fn.environment}
                    </span>
                  </td>
                  <td className="px-6 py-4">
                    <span className="px-2 py-1 text-xs font-medium rounded-full bg-green-100 text-green-800">
                      Active
                    </span>
                  </td>
                  <td className="px-6 py-4 text-right">
                    <div className="flex items-center justify-end gap-2">
                      <Button variant="ghost" size="sm">
                        Edit
                      </Button>
                      <Button variant="ghost" size="sm">
                        Delete
                      </Button>
                    </div>
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      </div>
    </div>
  )
}
```

---

## 7. 반응형 디자인 베스트 프랙티스

### 7.1 Tailwind 브레이크포인트

```tsx
// sm: 640px  - 모바일 가로
// md: 768px  - 태블릿
// lg: 1024px - 데스크탑
// xl: 1280px - 대형 데스크탑

// 모바일 우선 (Mobile First) 접근
<div className="p-4 md:p-6 lg:p-8">
  {/* 모바일: 16px, 태블릿: 24px, 데스크탑: 32px */}
</div>
```

### 7.2 반응형 그리드

```tsx
// 패턴 1: 1열 → 2열 → 3열
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
  {/* 모바일: 1열, 태블릿: 2열, 데스크탑: 3열 */}
</div>

// 패턴 2: 1열 → 2열 → 4열
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
  {/* 작은 카드에 적합 */}
</div>

// 패턴 3: 자동 채우기 (최소 너비 기반)
<div className="grid grid-cols-[repeat(auto-fit,minmax(280px,1fr))] gap-6">
  {/* 280px 이상 공간이 있으면 자동으로 열 추가 */}
</div>
```

### 7.3 반응형 Flexbox

```tsx
// 패턴 1: 세로 → 가로
<div className="flex flex-col md:flex-row gap-4">
  {/* 모바일: 세로 배치, 태블릿 이상: 가로 배치 */}
</div>

// 패턴 2: 헤더 (모바일: 세로, 데스크탑: 가로)
<div className="flex flex-col md:flex-row md:items-center md:justify-between gap-4">
  <div className="flex flex-col gap-2">
    <h1 className="text-2xl md:text-3xl lg:text-4xl font-bold">
      제목
    </h1>
    <p className="text-sm text-muted-foreground">설명</p>
  </div>
  <Button className="w-full md:w-auto">
    액션
  </Button>
</div>
```

### 7.4 반응형 간격

```tsx
// 작은 화면: 작은 간격, 큰 화면: 큰 간격
<div className="flex flex-col gap-4 md:gap-6 lg:gap-8">
  {/* 모바일: 16px, 태블릿: 24px, 데스크탑: 32px */}
</div>

// 페이지 패딩
<div className="p-4 md:p-6 lg:p-8 h-full overflow-auto">
  {/* 반응형 패딩 */}
</div>
```

---

## 8. 다크모드 지원

### 8.1 Semantic Colors (권장)

```tsx
// ✅ GOOD: 자동으로 다크모드 대응
<div className="bg-background text-foreground border-border">
  <h1 className="text-foreground">제목</h1>
  <p className="text-muted-foreground">설명</p>
</div>

// ❌ BAD: 다크모드에서 안 보임
<div className="bg-white text-black border-gray-300">
  <h1 className="text-gray-900">제목</h1>
  <p className="text-gray-500">설명</p>
</div>
```

### 8.2 상태 색상 (다크모드 대응)

```tsx
// 성공 (녹색)
<div className="bg-green-100 text-green-800 dark:bg-green-900/30 dark:text-green-400">
  성공 메시지
</div>

// 경고 (노란색)
<div className="bg-yellow-100 text-yellow-800 dark:bg-yellow-900/30 dark:text-yellow-400">
  경고 메시지
</div>

// 에러 (빨간색)
<div className="bg-red-100 text-red-800 dark:bg-red-900/30 dark:text-red-400">
  에러 메시지
</div>

// 정보 (파란색)
<div className="bg-blue-100 text-blue-800 dark:bg-blue-900/30 dark:text-blue-400">
  정보 메시지
</div>
```

### 8.3 그라디언트 (다크모드 무관)

```tsx
// 그라디언트는 라이트/다크 모두 잘 보임
<div className="p-3 rounded-xl bg-gradient-to-br from-blue-500 to-cyan-500">
  <Icon className="h-6 w-6 text-white" />
</div>

// 텍스트 그라디언트
<h1 className="text-4xl font-bold bg-gradient-to-r from-foreground to-foreground/70 bg-clip-text text-transparent">
  제목
</h1>
```

---

## 9. 최종 체크리스트

### 페이지 레이아웃
- [ ] `p-6 h-full overflow-auto` 최상위 컨테이너
- [ ] `max-w-7xl mx-auto` 중앙 정렬 + 최대 너비
- [ ] `flex flex-col gap-8` 주요 섹션 간격
- [ ] 반응형 패딩 (`p-4 md:p-6 lg:p-8`)

### Flexbox
- [ ] `flex + gap` 사용 (margin 금지)
- [ ] `items-center` 아이콘과 텍스트 정렬
- [ ] `justify-between` 헤더 좌우 배치
- [ ] 반응형 방향 전환 (`flex-col md:flex-row`)

### Grid
- [ ] 카드 리스트에만 사용
- [ ] `grid-cols-1 md:grid-cols-2 lg:grid-cols-3` 반응형 컬럼
- [ ] `h-full` 카드 높이 통일
- [ ] `gap-6` 적절한 간격

### 스타일링
- [ ] Semantic colors 사용 (`foreground`, `muted-foreground`)
- [ ] 상태 색상 다크모드 대응 (`dark:` prefix)
- [ ] 아이콘 크기 통일 (`h-4 w-4`)
- [ ] 애니메이션 `duration-300` 통일

### 컴포넌트
- [ ] Container/Presentational 패턴 (100줄 이하)
- [ ] Custom Hooks로 로직 분리
- [ ] Props 명확히 정의

---

## 마무리

### 핵심 요약

> **"margin을 버리고 gap을 택하라. 간격은 부모가 관리한다."**

**컴포넌트 배치 4대 원칙**:
1. **flex + gap 패턴**: margin 완전 배제, 부모에서 간격 중앙 관리
2. **Semantic colors**: 다크모드 자동 대응, 하드코딩 색상 금지
3. **반응형 우선**: 모바일 → 태블릿 → 데스크탑 단계적 최적화
4. **Grid는 선택적**: 카드 리스트 등 2차원 격자 배치에만 사용

### 언제 사용하나?

**flex + gap 패턴 권장:**
- ✅ 페이지 주요 섹션 배치 (`flex-col gap-8`)
- ✅ 헤더 좌우 배치 (`justify-between`)
- ✅ 아이콘 + 텍스트 조합 (`items-center gap-2`)
- ✅ 폼 입력 필드 세로 배치 (`flex-col gap-4`)

**Grid 권장:**
- ✅ 카드 리스트 (API Gateway, Function 등)
- ✅ Dashboard 위젯 배치 (2x2, 3x3 등)
- ✅ 갤러리/썸네일 그리드

### 실제 적용 결과

**[imprun.dev](https://imprun.dev) 환경:**
- ✅ 페이지 컴포넌트: 평균 100줄 이하 유지
- ✅ 코드 감소: margin 제거로 약 40% 감축 (실제 경험)
- ✅ 다크모드: Semantic colors로 자동 대응 (추가 작업 0시간)
- ✅ 반응형: 브레이크포인트로 모바일/태블릿/데스크탑 최적화

**운영 경험:**
- 최초 학습 시간: 약 2-3시간 (Flexbox + Gap 패턴 숙지)
- 리팩토링 시간: 기존 페이지 1개당 약 30분
- 유지보수 시간: 레이아웃 수정 약 75% 단축 (실제 경험)
- 만족도: 매우 높음 😊 (일관된 디자인, 빠른 개발)

---

## 관련 글

- [Claude AI와 함께하는 프론트엔드 개발: imprun.dev의 CLAUDE.md 가이드 공개](https://blog.imprun.dev/42)
- [Next.js를 버리고 순수 React로 돌아온 이유: 실무 관점의 프레임워크 선택 여정](https://blog.imprun.dev/41)
- [CLAUDE.md 최적화 여정: AI가 패턴을 무시하는 이유와 해결책](https://blog.imprun.dev/57)

---

**태그:** React, Tailwind CSS, Flexbox, Grid, Layout, Frontend, UI/UX, imprun.dev

---

> "margin을 버리고 gap을 택하라. 간격은 부모가 관리한다."

🤖 *이 블로그는 [imprun.dev](https://imprun.dev) 플랫폼 프론트엔드 구축 과정에서 React + Tailwind CSS v4를 활용한 실제 경험을 바탕으로 작성되었습니다.*

---

**질문이나 피드백은 블로그 댓글에 남겨주세요!**
