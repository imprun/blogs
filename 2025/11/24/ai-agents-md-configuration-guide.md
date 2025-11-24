# Claude, Gemini, Codex에서 AGENTS.md 설정하기: AI 에이전트 통합 가이드

> **작성일**: 2025년 11월 24일
> **카테고리**: AI, DevTools, Configuration
> **키워드**: Claude Code, Gemini CLI, OpenAI Codex, AGENTS.md, CLAUDE.md, AI Agent

## 요약

Claude Code, Gemini CLI, OpenAI Codex CLI 세 가지 AI 에이전트에서 AGENTS.md를 인식하도록 설정하는 방법을 정리합니다. 각 에이전트의 기본 파일명과 설정 방법이 다르지만, 적절한 설정을 통해 단일 AGENTS.md 파일로 모든 에이전트를 통합 관리할 수 있습니다.

## 문제 상황

### 배경

모노레포 환경에서 여러 AI 코딩 어시스턴트를 사용할 때 다음 문제가 발생합니다:

- 각 에이전트가 인식하는 기본 파일명이 다름 (CLAUDE.md, GEMINI.md, AGENTS.md)
- 동일한 개발 가이드를 여러 파일로 중복 관리해야 함
- 에이전트별 설정 방법을 파악하기 어려움

### 목표

- 단일 AGENTS.md 파일로 세 에이전트 모두 지원
- 각 에이전트의 설정 방법 문서화
- 모노레포 환경에서의 통합 전략 제시

## 에이전트별 설정

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

**GEMINI.md에서 AGENTS.md 임포트** (선택적):
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

## 에이전트별 특성 비교

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

## 통합 전략: 단일 AGENTS.md 사용

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

## 교훈

### 1. 통합 파일명으로 AGENTS.md 사용

Codex CLI가 기본으로 인식하고, Gemini와 Claude도 설정이나 링크로 연결 가능하므로 AGENTS.md를 표준으로 사용하는 것이 효율적입니다.

### 2. 설정 파일 위치 숙지

각 에이전트의 글로벌 설정 위치(`~/.claude/`, `~/.gemini/`, `~/.codex/`)를 알아두면 프로젝트별 설정 없이도 기본 규칙을 적용할 수 있습니다.

### 3. 계층적 로드 활용

세 에이전트 모두 계층적 로드를 지원하므로, 모노레포에서 루트와 각 앱/패키지별로 AGENTS.md를 배치하면 자동으로 상속됩니다.

## 참고 자료

### 공식 문서
- [AGENTS.md 공식 사이트](https://agents.md) - 20,000+ 오픈소스 프로젝트에서 사용하는 AI 에이전트 지침 표준
- [Claude Code Documentation](https://docs.anthropic.com/claude-code) - CLAUDE.md 기반 지침 시스템
- [Gemini CLI Documentation](https://google-gemini.github.io/gemini-cli/)
- [OpenAI Codex CLI Documentation](https://github.com/openai/codex)

### 관련 블로그
- [AI Agent를 위한 Frontend 개발 가이드: AGENTS.md로 Next.js + shadcn/ui 프로젝트 구조 설계](https://blog.imprun.dev/68)
