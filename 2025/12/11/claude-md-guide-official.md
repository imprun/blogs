# CLAUDE.md 공식 가이드 정리: 프로젝트 컨텍스트 자동화

> **작성일**: 2025년 12월 11일
> **카테고리**: AI, Developer Tools, Claude Code
> **키워드**: CLAUDE.md, Claude Code, Project Context, AI Configuration

## 요약

Claude 공식 블로그에 CLAUDE.md 파일 사용법 가이드가 올라왔다. CLAUDE.md는 프로젝트별 컨텍스트를 자동으로 Claude에게 전달하는 설정 파일이다. 매번 프로젝트 구조와 규칙을 반복 설명할 필요가 없어진다.

## CLAUDE.md란?

저장소에 위치하는 특수 마크다운 파일로, Claude와의 모든 대화에 프로젝트 특정 맥락을 자동으로 포함시킨다.

### 파일 배치 위치

| 위치 | 적용 범위 |
|---|---|
| 저장소 루트 | 해당 프로젝트, 팀과 공유 |
| 상위 디렉토리 | 모노레포 전체 설정 |
| 홈 폴더 (`~`) | 모든 프로젝트에 적용 |

### 파일 Import 문법

`@path/to/file` 문법으로 다른 마크다운 파일을 import할 수 있다.

```markdown
# 공통 규칙 import
@~/.claude/coding-standards.md

# 프로젝트 문서 참조
@docs/architecture.md
@docs/api-design.md
```

| 경로 유형 | 예시 |
|---|---|
| 상대 경로 | `@docs/setup.md` |
| 홈 디렉토리 | `@~/.claude/common-rules.md` |

주요 규칙:
- 재귀 import 지원 (최대 5단계)
- 코드 블록 내에서는 import로 처리되지 않음
- `/memory` 명령어로 로드된 파일 목록 확인 가능
- 여러 git worktree 환경에서 `CLAUDE.local.md`보다 유연함

## 권장 구조

### 1. 프로젝트 맵

디렉토리 구조를 명시하여 Claude가 코드 위치를 빠르게 파악하도록 한다.

```markdown
## 프로젝트 구조

- `app/models/` - 데이터베이스 모델
- `app/api/` - 라우트 핸들러
- `app/core/` - 설정 및 유틸리티
- `tests/` - pytest 테스트
```

### 2. 개발 표준

코딩 컨벤션과 규칙을 명시한다.

```markdown
## 코딩 규칙

- 모든 함수에 Type hints 필수
- PEP 8 준수, 100자 줄 길이 제한
- pytest로 테스트 작성
```

### 3. 자주 사용하는 명령어

실제 워크플로우에서 사용하는 명령어를 기록한다.

```markdown
## 명령어

- 개발 서버: `pnpm dev`
- 테스트: `pnpm test`
- 빌드: `pnpm build`
```

### 4. 도구 연결

MCP 서버, 외부 도구 등 커스텀 설정을 기록한다.

```markdown
## MCP 서버

- `@github` - GitHub 연동
- `@slack` - #dev-alerts 채널 알림
```

### 5. 표준 워크플로우

다음 네 가지 질문에 답하는 구조를 정의한다:

1. 현재 상태 조사가 필요한가?
2. 구현 전 상세 계획이 필요한가?
3. 누락된 정보가 있는가?
4. 효과성을 어떻게 검증하는가?

## 시작하기: /init 명령어

프로젝트에서 `/init` 실행 시 Claude가 자동으로:

1. 패키지 파일 분석 (`package.json`, `pyproject.toml` 등)
2. 기존 문서 검토 (`README.md` 등)
3. 설정 파일 조사
4. 코드 구조 파악

초기 CLAUDE.md를 생성한다. 생성 후 팀 관행에 맞게 수정하는 것이 중요하다.

## 활용 팁

### 컨텍스트 정리

`/clear` 명령어로 이전 작업 내용을 제거하되, CLAUDE.md는 유지된다. 새로운 작업에 집중할 때 유용하다.

### 서브에이전트 활용

복잡한 작업의 각 단계별로 isolated context를 유지하여 이전 정보의 간섭을 방지한다. 예를 들어 보안 검토는 별도 서브에이전트에서 수행한다.

### 커스텀 명령어

`.claude/commands/` 디렉토리에 마크다운 파일로 반복적 프롬프트를 저장하면 `/명령어명`으로 호출할 수 있다.

```
.claude/
└── commands/
    ├── review.md      # /review로 호출
    └── deploy.md      # /deploy로 호출
```

## 작성 원칙

### Start simple, expand deliberately

모든 정보를 한 번에 넣으려 하지 않는다. 실제 문제 해결 과정에서 필요한 내용을 점진적으로 추가한다.

### 간결하게 유지

CLAUDE.md는 매 대화마다 시스템 프롬프트에 포함되므로 토큰을 소비한다. 큰 정보는 별도 마크다운 파일로 분리하고 참조한다.

### 실제 워크플로우 반영

팀의 실제 개발 방식을 반영한다. 이론적 베스트 프랙티스보다 실제로 사용하는 방식을 기록한다.

### 민감 정보 제외

API 키, 자격증명 등 민감 정보는 절대 포함하지 않는다.

## 결론

CLAUDE.md는 프로젝트 컨텍스트를 자동화하여 반복적인 설명을 줄여준다. `/init`으로 시작해서 점진적으로 확장하는 것을 권장한다.

## 참고 자료

- [Using CLAUDE.md files](https://www.claude.com/blog/using-claude-md-files) - Claude 공식 블로그
- [Claude Code: Best practices](https://www.anthropic.com/engineering/claude-code-best-practices) - Anthropic 엔지니어링 블로그
- [Memory Documentation](https://code.claude.com/docs/en/memory) - Import 문법 공식 문서
- [Claude Code Documentation](https://docs.anthropic.com/en/docs/claude-code)
