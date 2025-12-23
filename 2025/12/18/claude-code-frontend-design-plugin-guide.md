# Claude Code로 세련된 UI 만들기

> **작성일**: 2025년 12월 18일
> **카테고리**: Frontend, Claude Code, Design
> **키워드**: Claude Code, Plugin, Frontend Design, UI/UX, Anthropic

## 요약

[Antigravity](https://antigravity.google/)처럼 "AI가 만든 티가 나지 않는" UI를 Claude Code로도 만들 수 있다. Anthropic에서 제공하는 frontend-design 플러그인을 설치하거나, Frontend Aesthetics Cookbook의 프롬프트 기법을 적용하면 된다. 이 글에서는 두 가지 방법과 실제 적용 결과를 다룬다.

## 문제: AI UI는 왜 다 비슷한가

AI로 UI를 생성하면 대부분 비슷한 결과가 나온다.

| 패턴 | 예시 |
|------|------|
| 폰트 | Inter, Roboto, Arial |
| 색상 | 보라색/파란색 그라데이션 |
| 레이아웃 | 대칭, 균등 배치 |
| 인터랙션 | hover 시 opacity 변경 |

이유는 명확하다. AI는 학습 데이터에서 가장 빈번하게 등장하는 "안전한" 선택을 한다. 결과적으로 기능은 동작하지만, 브랜드 차별화가 어렵고 "AI가 만들었네"라는 인상을 준다.

## 해결책 1: Frontend-Design 플러그인

Anthropic에서 공식 제공하는 플러그인이다. 설치만 하면 Claude Code가 자동으로 디자인 원칙을 적용한다.

### 설치

```bash
# Claude Code 실행 후 플러그인 마켓플레이스에서 설치
/plugin marketplace add anthropics/claude-code
```

설치 화면에서 **frontend-design** 스킬을 Spacebar로 선택하고 Enter로 설치한다. 재시작 후 `/plugin` 명령어로 활성화 여부를 확인할 수 있다.

### 사용

```
frontend-design 스킬로 Dashboard 컴포넌트를 만들어줘
```

스킬 이름을 명시하면 해당 스킬의 디자인 철학이 적용된다.

### 플러그인이 변경하는 것

| 영역 | Before | After |
|------|--------|-------|
| 타이포그래피 | 시스템 폰트, 단조로운 위계 | 폰트 페어링, 명확한 시각적 위계 |
| 색상 | 하드코딩된 값 | CSS 변수 기반 팔레트 |
| 애니메이션 | 없거나 기본 transition | 마이크로인터랙션 |
| 레이아웃 | 대칭, 예측 가능 | 의도적 비대칭, 공간 활용 |

## 해결책 2: 프롬프트 기법

플러그인 없이도 프롬프트만으로 개선할 수 있다. Anthropic의 [Frontend Aesthetics Cookbook](https://github.com/anthropics/claude-cookbooks/blob/main/coding/prompting_for_frontend_aesthetics.ipynb)에서 소개하는 기법이다.

### 기본 원칙

Claude는 명시적으로 지시하지 않으면 보수적인 선택을 한다. 따라서 **피해야 할 것을 명시**하는 것이 핵심이다.

| 차원 | 피해야 할 것 | 지시 방법 |
|------|------------|----------|
| 타이포그래피 | Arial, Inter, Roboto | "디스플레이와 본문 간 극단적 대비" |
| 색상 | 흰 배경 + 보라색 그라데이션 | "IDE 다크 테마에서 영감, CSS 변수로 관리" |
| 모션 | 산발적인 hover 효과 | "페이지 로드 시 staggered reveal" |
| 배경 | 단색 | "미묘한 그라데이션이나 기하학적 패턴" |

### 프롬프트 비교

**일반적인 요청**:
```
랜딩 페이지를 만들어줘
```

**개선된 요청**:
```
SaaS 랜딩 페이지를 만들어줘.

- 타이포그래피: Inter/Roboto 사용 금지, 디스플레이와 본문 간 극단적 대비
- 색상: IDE 다크 테마에서 영감받은 팔레트, CSS 변수로 관리
- 모션: 페이지 로드 시 staggered reveal 애니메이션
- 배경: 단색 금지, 미묘한 그라데이션이나 기하학적 패턴
```

### 한 줄 프롬프트: "세련된 UI로 만들어"

상세 프롬프트를 작성할 시간이 없다면 이 한 줄만 추가한다:

```
세련된 UI로 만들어줘
```

또는:

```
이 컴포넌트를 세련되게 만들어줘
```

Claude가 "세련됨"을 해석하여 타이포그래피, 간격, 색상을 자동으로 조정한다. 단, 결과가 매번 다르므로 일관성이 필요하면 상세 프롬프트를 사용해야 한다.

| 방식 | 장점 | 단점 |
|------|------|------|
| "세련되게 만들어" | 빠름 | 결과 예측 불가 |
| 4가지 차원 명시 | 일관된 품질 | 프롬프트 작성 시간 필요 |
| 테마 고정 | 프로젝트 전체 일관성 | 초기 설정 필요 |

### 테마 고정

여러 컴포넌트에 일관된 스타일을 적용하려면 테마를 먼저 정의한다:

```
이 프로젝트의 모든 UI는 "Industrial Tech" 미학을 따른다:
- 색상: OKLCH 기반 cyan/teal 계열
- 폰트: Plus Jakarta Sans (UI) + JetBrains Mono (코드)
- 스타일: 기능 영역별 색상 코딩, 상태 표시 강조

이 미학으로 Dashboard 컴포넌트를 만들어줘.
```

## 결과 비교

**SaaS 랜딩 페이지**

| Before | After |
|--------|-------|
| ![Baseline SaaS](https://raw.githubusercontent.com/anthropics/claude-cookbooks/main/images/frontend_aesthetics/baseline_saas.png) | ![Distilled SaaS](https://raw.githubusercontent.com/anthropics/claude-cookbooks/main/images/frontend_aesthetics/distilled_saas.png) |

**Admin Dashboard**

| Before | After |
|--------|-------|
| ![Baseline Dashboard](https://raw.githubusercontent.com/anthropics/claude-cookbooks/main/images/frontend_aesthetics/baseline_dashboard.png) | ![Distilled Dashboard](https://raw.githubusercontent.com/anthropics/claude-cookbooks/main/images/frontend_aesthetics/distilled_dashboard.png) |

*출처: [Frontend Aesthetics Cookbook](https://github.com/anthropics/claude-cookbooks/blob/main/coding/prompting_for_frontend_aesthetics.ipynb)*

레이아웃 구조는 동일하지만 시각적 품질이 개선되었다.

## 주의점

### 기존 코드에는 보수적으로 적용된다

위의 Before/After 이미지는 **처음부터 새로 생성**한 결과다. 기존 코드에 스킬을 적용하면 이 정도의 변화가 나오지 않을 가능성이 높다. Claude는 일반적으로 기존 구조를 존중하며 점진적으로 개선하는 경향이 있기 때문이다.

기존 코드에서 극적인 변화를 원한다면 구체적인 변경 방향을 지시해야 한다:

```
다음 방향으로 대시보드를 재설계해줘:
- 위젯 기반 드래그앤드롭 레이아웃
- 서비스 목록을 칸반 보드 스타일로 (상태별 컬럼 뷰)
- 탑 네비게이션 + 커맨드 팔레트 중심 UX
- 차트, 그래프, 플로우 다이어그램 추가
```

### 기타 주의점

| 항목 | 설명 |
|------|------|
| 기존 디자인 시스템 | 이미 확립된 디자인 토큰이 있으면 충돌할 수 있다 |
| 접근성 | 생성된 색상 대비가 WCAG 기준을 충족하는지 검증이 필요하다 |
| 번들 크기 | 추가된 폰트와 애니메이션이 성능에 영향을 줄 수 있다 |

## 결론

| 방법 | 적합한 경우 |
|------|-----------|
| 플러그인 | 설정 없이 바로 사용하고 싶을 때 |
| 프롬프트 기법 | 세밀한 제어가 필요할 때 |
| "세련되게 만들어" | 빠른 프로토타이핑 |

디자이너 없이 개발하는 팀이나 빠른 UI 개선이 필요한 경우 유용하다.

## 참고 자료

- [Frontend-Design Plugin](https://github.com/anthropics/claude-code/tree/main/plugins/frontend-design)
- [Frontend Aesthetics Cookbook](https://github.com/anthropics/claude-cookbooks/blob/main/coding/prompting_for_frontend_aesthetics.ipynb)
- [Claude Code Plugins Documentation](https://docs.claude.com/en/docs/claude-code/plugins)
