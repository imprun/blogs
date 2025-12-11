# Claude Code 베스트 프랙티스: Anthropic 공식 가이드 정리

> **작성일**: 2025년 12월 11일
> **카테고리**: AI, Developer Tools, Claude Code
> **키워드**: Claude Code, CLAUDE.md, Best Practices, AI Coding Assistant

## 요약

Anthropic 엔지니어링 블로그에 게시된 Claude Code 베스트 프랙티스 가이드를 정리했다. CLAUDE.md 설정, 도구 확장, 워크플로우 최적화, 다중 인스턴스 활용까지 실전 팁을 다룬다.

## 1. CLAUDE.md 설정

### 개념

CLAUDE.md는 Claude가 대화 시작 시 자동으로 로드하는 프로젝트 설정 파일이다.

### 포함할 내용

- 프로젝트별 bash 명령어, 핵심 파일, 유틸리티 함수
- 코드 스타일 가이드라인
- 테스트 방법
- 예상치 못한 동작 관련 주의사항

### 배치 위치

| 위치 | 용도 |
|---|---|
| 저장소 루트 | 팀 공유 (체크인 권장) |
| 부모/자식 디렉토리 | 모노레포 설정 |
| `~/.claude/CLAUDE.md` | 모든 프로젝트 공통 |

### 팁: 프롬프트 개선

Anthropic은 CLAUDE.md 파일을 prompt improver에 돌려서 모델 지시 준수도를 높인다고 한다.

## 2. 권한 설정

도구 권한을 설정하는 네 가지 방법:

1. 세션 중 "Always allow" 선택
2. `/permissions` 명령어
3. `.claude/settings.json` 또는 `~/.claude.json` 수정
4. `--allowedTools` CLI 플래그

## 3. 도구 확장

### Bash 도구

Claude Code는 쉘 환경을 상속받는다.

- 도구명과 사용 예시를 설명
- `--help` 실행 유도
- 자주 사용하는 도구는 CLAUDE.md에 문서화

### MCP 서버 연결

| 방식 | 범위 |
|---|---|
| 프로젝트 설정 | 해당 디렉토리만 |
| 글로벌 설정 | 모든 프로젝트 |
| `.mcp.json` 체크인 | 팀 전체 공유 |

### 커스텀 슬래시 명령어

`.claude/commands/` 폴더에 마크다운 파일로 프롬프트 템플릿 저장:

```
.claude/commands/
├── review.md    # /review로 호출
└── deploy.md    # /deploy로 호출
```

`$ARGUMENTS` 키워드로 파라미터 전달 가능.

## 4. 워크플로우

### a) 탐색 → 계획 → 코딩 → 커밋

Anthropic 권장 순서:

1. **탐색**: 관련 파일 읽기 요청 (작성 금지 명시)
2. **계획**: "think" 키워드로 확장 사고 모드 활성화
3. **코딩**: 해결책 구현
4. **커밋**: 결과물 커밋 및 PR 생성

> "Steps #1-#2 are crucial—without them, Claude tends to jump straight to coding"

### b) 테스트 주도 개발 (TDD)

1. 입출력 기반 테스트 작성 (실제 구현 없음 강조)
2. 테스트 실행 및 실패 확인
3. 테스트 커밋
4. 테스트 통과 코드 작성
5. 최종 커밋

### c) 시각적 설계 반복

1. 스크린샷 또는 디자인 목업 제공
2. 구현 요청
3. 결과 스크린샷 확인
4. 일치할 때까지 반복

### d) 코드베이스 Q&A

새로운 프로젝트 학습 시 자연스러운 질문으로 탐색.

> Anthropic: "this way has become our core onboarding workflow"

### e) Git/GitHub 작업

Claude가 처리 가능한 작업:

- 히스토리 검색으로 설계 결정 추적
- 컨텍스트 기반 커밋 메시지 작성
- 복잡한 리베이스/병합 충돌 해결
- PR 생성 및 관리
- 코드 리뷰 피드백 자동 적용

## 5. 최적화 팁

### 구체적 지시

> "Claude can infer intent, but it can't read minds"

| 나쁜 예 | 좋은 예 |
|---|---|
| "foo.py 테스트 추가" | "foo.py를 위한 테스트 케이스 작성, 사용자 미로그인 엣지 케이스 포함. 모킹 회피" |

### 이미지 활용

- Mac: `Cmd+Ctrl+Shift+4` 스크린샷 후 `Ctrl+V`
- 드래그 드롭으로 이미지 직접 업로드
- 파일 경로 지정

### 조기에 자주 방향 수정

- `Escape`: 실행 중인 작업 중단 (컨텍스트 유지)
- `Escape` 더블탭: 이전 프롬프트로 돌아가기
- `/undo`: 변경사항 되돌리기

### /clear 명령어

장시간 세션에서 관련 없는 컨텍스트 정리.

### 체크리스트 활용

대규모 작업(마이그레이션 등)에 마크다운 체크리스트:

1. 오류를 체크리스트로 출력
2. 각 항목을 순서대로 해결

## 6. 다중 Claude 워크플로우

### 작성 + 리뷰 분리

1. Claude가 코드 작성
2. `/clear` 또는 새 터미널에서 두 번째 Claude
3. 두 번째가 첫 번째 결과물 리뷰
4. 세 번째 Claude가 피드백 기반 수정

### Git Worktree

```bash
git worktree add ../project-feature-a feature-a
```

각 worktree에 독립적 Claude 인스턴스 할당.

### 헤드리스 자동화

```bash
# 비대화형 모드
claude -p "<프롬프트>"

# JSON 출력
claude -p "<프롬프트>" --output-format stream-json

# 파이프라인 통합
claude -p "<프롬프트>" --json | your_command
```

## 7. 핵심 철학

> Claude Code는 "intentionally low-level and unopinionated" 설계로 안전하면서도 유연하다.

각 팀이 자신의 사용 패턴을 반복 실험하여 최적화하는 것이 권장된다.

## 참고 자료

- [Claude Code: Best practices for agentic coding](https://www.anthropic.com/engineering/claude-code-best-practices) - Anthropic 엔지니어링 블로그
- [Best Practices (Official Docs)](https://code.claude.com/docs/en/claude-code-on-the-web#best-practices) - 공식 문서
- [Claude Code Documentation](https://code.claude.com/docs/en/overview)
