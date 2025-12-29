# Claude Code 컨텍스트 최적화 실전 가이드

> **작성일**: 2025년 12월 29일
> **카테고리**: Claude Code, Developer Tools, Performance
> **키워드**: Claude Code, 토큰 최적화, MCP, 플러그인, 컨텍스트 관리

## 요약

Claude Code에서 `/compact` 실행 후에도 컨텍스트가 33% 사용 중이었다. `/context` 명령어로 분석한 결과, 불필요한 MCP 도구(22.6k 토큰)와 중복 플러그인이 원인이었다. 정리 후 10%로 줄였고, Plan 모드와 Phase 분리 패턴으로 효율적인 작업 방식을 정립했다.

## 문제 상황

### 왜 컨텍스트 관리가 중요한가

Claude Code의 컨텍스트는 **책상 위 공간**과 같다. 책상이 좁으면 필요한 자료를 펼칠 수 없다. 컨텍스트가 부족하면 Claude가 코드를 분석하고 수정하는 데 필요한 정보를 담을 수 없다.

| 컨텍스트 사용률 | 상태 |
|----------------|------|
| 10% | 책상이 넓다. 대규모 리팩토링 가능 |
| 50% | 공간이 부족해지기 시작 |
| 80% | 새 파일을 열기 어려움 |
| 100% | 작업 불가. `Prompt is too long` 에러 |

### 증상

`/compact`로 대화를 압축했는데도 33%가 사용 중이었다.

```
/compact 실행 후
Tokens: 65.2k / 200.0k (33%)
```

대화 내용은 압축했는데 왜 여전히 높은가?

### 초기 진단

`/context` 명령어로 사용량을 확인했다.

```bash
/context
```

| Category | Tokens | Percentage |
|----------|--------|------------|
| System prompt | 3.4k | 1.7% |
| System tools | 20.4k | 10.2% |
| **MCP tools** | **22.6k** | **11.3%** |
| Custom agents | 2.3k | 1.1% |
| Messages | 16.3k | 8.1% |

MCP tools가 22.6k 토큰(11.3%)을 차지하고 있었다. 대화 내용(Messages)보다 MCP 도구 정의가 더 많은 공간을 점유하고 있었다.

## 근본 원인 분석

### MCP 도구가 토큰을 소비하는 이유

MCP 서버를 등록하면, Claude에게 "이런 도구를 사용할 수 있다"는 설명서가 전달된다. 도구가 많을수록 설명서가 길어지고, 그만큼 컨텍스트를 소비한다.

```
Playwright MCP 등록 시:
├── browser_close: 573 tokens (도구 설명)
├── browser_resize: 622 tokens
├── browser_fill_form: 776 tokens
└── ... (18개 도구, 총 ~11k tokens)
```

Playwright MCP 하나가 18개 도구를 등록하며 11k 토큰을 소비한다. 사용하지 않아도 등록만으로 공간을 차지한다.

### 중복 플러그인 문제

`/context`의 Skills 섹션에서 같은 플러그인이 두 번씩 나타났다.

| Skill | Tokens |
|-------|--------|
| code-review:code-review (1) | 1.7k |
| code-review:code-review (2) | 1.8k |
| frontend-design (1) | 989 |
| frontend-design (2) | 989 |

`claude-plugins-official`과 `claude-code-plugins` 두 저장소에서 같은 플러그인을 중복 설치한 것이 원인이었다.

### 프로젝트 전용 Skills

CLAUDE.md에 정의한 Skills도 토큰을 소비한다.

```
create-epic              3.7k tokens
generate-release-notes   2.1k tokens
start-epic               1.5k tokens
```

이 Skills는 특정 프로젝트에서만 필요하지만, 모든 세션에서 로드되고 있었다.

## 해결 과정

### 1. MCP 서버 정리

```bash
# 등록된 MCP 서버 확인
claude mcp list

# 사용하지 않는 서버 제거
claude mcp remove playwright
claude mcp remove shadcn
```

**판단 기준**: 최근 1주일 내 사용하지 않았으면 제거.

### 2. 중복 플러그인 제거

```bash
# 설치된 플러그인 확인
claude plugin list

# 중복 제거 (하나만 남기고)
claude plugin remove code-review@claude-code-plugins
```

### 3. 프로젝트별 설정 분리

```bash
# 전역 설정 (모든 프로젝트에서 필요한 것만)
~/.claude/settings.json

# 프로젝트별 설정 (해당 프로젝트에서만 필요한 것)
project/.claude/settings.local.json
```

### 4. 검증

```bash
/context
```

```
Context Usage
claude-opus-4-5-20251101 · 20k/200k tokens (10%)

⛁ System prompt: 3.4k tokens (1.7%)
⛁ System tools: 16.0k tokens (8.0%)
⛁ Custom agents: 152 tokens (0.1%)
⛁ Memory files: 107 tokens (0.1%)
⛶ Free space: 180k (90.2%)
```

33% → 10%로 감소. 45k 토큰을 확보했다.

| 항목 | Before | After | 절감 |
|------|--------|-------|------|
| **Total** | **65.2k (33%)** | **20k (10%)** | **45.2k (69%)** |
| MCP tools | 22.6k | 0 | 22.6k |
| Custom agents | 2.3k | 152 | 2.1k |
| Skills | ~15k | ~2.8k | ~12k |

## 교훈

### Plan 모드 + Phase 분리 패턴

컨텍스트를 효율적으로 사용하려면 작업 단위를 작게 나눠야 한다.

**문제**: 긴 작업을 한 번에 처리하면 컨텍스트가 빠르게 소진된다.

**해결**: Plan 모드로 작업을 Phase로 나누고, 각 Phase 완료 시 `/compact`를 실행한다.

```
Phase 1: 데이터 모델 설계
  → 작업 완료 → /compact

Phase 2: API 엔드포인트 구현
  → 작업 완료 → /compact

Phase 3: 테스트 작성
  → 작업 완료 → /compact
```

### 자동 /compact 비활성화

```bash
/config
# → auto-compact: off
```

자동 압축은 Claude가 임의 시점에 실행한다. 작업 중간에 압축되면 맥락이 손실될 수 있다. Phase 경계에서 수동으로 압축하면 논리적 단위가 유지된다.

### 컨텍스트 초과는 설계 실패

작업 도중 다음 에러가 나타나면, Phase 설계가 잘못되었다는 신호다.

```
Prompt is too long
```

이 상황이 발생하면 새 세션을 시작하고, 다음에는 Phase를 더 작게 나눠야 한다.

### 모니터링 습관

| 시점 | 명령어 | 목적 |
|------|--------|------|
| 세션 시작 | `/context` | 초기 사용량 확인 |
| Phase 완료 | `/compact` | 압축 |
| 50% 도달 | `/compact` | 조기 압축 |

## 참고 자료

- [Claude Code 공식 문서](https://docs.anthropic.com/en/docs/claude-code)
- [MCP 서버 관리](https://modelcontextprotocol.io)
