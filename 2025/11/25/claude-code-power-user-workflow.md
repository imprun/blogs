# Claude Code 파워 유저 워크플로우: /catchup부터 컨텍스트 관리까지

> **작성일**: 2025년 11월 25일
> **카테고리**: Developer Tools, AI, Productivity
> **키워드**: Claude Code, AI Coding, Workflow, Context Management, Custom Commands

## 요약

Shrivu Shankar의 블로그 "[How I Use Every Claude Code Feature](https://blog.sshh.io/p/how-i-use-every-claude-code-feature)"에서 소개된 Claude Code 활용 전략을 분석합니다. 특히 `/catchup` 커스텀 명령어를 통한 세션 온보딩, 컨텍스트 관리 전략, 그리고 Fire and Forget 워크플로우가 실무에서 어떻게 적용되는지 살펴봅니다.

## 배경: 매일 반복되는 온보딩 문제

Claude Code를 실무에서 사용하다 보면 매번 새 세션을 시작할 때마다 동일한 패턴이 반복됩니다:

1. 현재 브랜치에서 작업 중인 파일들을 파악
2. 최근 변경사항 확인
3. 작업 맥락 설명
4. 실제 작업 시작

이 온보딩 과정이 매번 5-10분씩 소요되고, 토큰도 상당량 소비됩니다. Shrivu는 이 문제를 커스텀 슬래시 명령어로 해결했습니다.

## /catchup: 세션 온보딩 자동화

### 개념

`/catchup`은 새 세션에서 현재 브랜치의 변경 파일들을 자동으로 읽어 컨텍스트를 구축하는 커스텀 명령어입니다.

### 구현

`.claude/commands/catchup.md` 파일을 생성합니다:

```markdown
현재 git 브랜치에서 main/master 대비 변경된 모든 파일을 읽고,
작업 맥락을 파악하세요.

1. `git diff --name-only main...HEAD`로 변경된 파일 목록 확인
2. 각 파일을 Read 도구로 읽기
3. 변경 내용을 요약하고 현재 작업 상태 보고
4. 다음 작업을 제안

파일이 많으면 주요 파일 위주로 읽고, 나머지는 목록만 보고하세요.
```

### 사용 워크플로우

```bash
# 1. 새 세션 시작
claude

# 2. 이전 컨텍스트 정리
/clear

# 3. 브랜치 변경사항 기반 온보딩
/catchup

# 4. 실제 작업 시작
"이어서 API 엔드포인트 구현해줘"
```

### 장점

- 매번 수동으로 파일을 설명할 필요 없음
- git 기반이므로 정확한 작업 범위 파악
- 토큰 효율적 (필요한 파일만 읽음)

## 컨텍스트 관리 전략

Claude Code는 200K 토큰 컨텍스트 윈도우를 사용합니다. Shrivu는 이를 효율적으로 관리하는 세 가지 전략을 제시합니다.

### 1. /compact - 피해야 할 방법

```bash
/compact
```

자동 압축은 불투명하고 중요한 컨텍스트가 손실될 수 있습니다. 프로덕션 환경에서는 권장하지 않습니다.

### 2. /clear + /catchup - 표준 방법

```bash
/clear      # 상태 완전 초기화
/catchup    # git 변경사항 기반 재구축
```

가장 예측 가능하고 안정적인 방법입니다. 컨텍스트가 70% 이상 차면 이 방법을 사용합니다.

### 3. 문서화 & 초기화 - 복잡한 작업용

복잡한 멀티스텝 작업에서는 진행 상황을 마크다운으로 저장한 후 초기화합니다:

```bash
# 진행 상황 저장
"현재까지의 작업 내용을 PROGRESS.md에 정리해줘"

# 초기화
/clear

# 새 세션에서 재개
"PROGRESS.md 읽고 이어서 작업해줘"
```

### 컨텍스트 사용량 확인

```bash
/context
```

현재 토큰 사용량을 확인할 수 있습니다. 새 세션은 약 20K 토큰(10%)을 기본으로 소비합니다.

### VSCode 확장에서의 컨텍스트 모니터링

VSCode 확장에서는 상태 표시줄에 남은 컨텍스트 퍼센티지가 표시됩니다:

| 표시 | 의미 | 권장 조치 |
|------|------|-----------|
| 70%+ 남음 | 여유 있음 | 계속 작업 |
| 40-70% 남음 | 주의 필요 | 복잡한 작업 전 정리 고려 |
| 30-40% 남음 | 정리 권장 | `/clear + /catchup` 실행 |
| 30% 미만 | 즉시 정리 | 자동 compact 전에 수동 정리 |

### 자동 Compact vs 수동 정리

0%에 도달하면 시스템이 자동으로 `/compact`를 실행합니다. 하지만 앞서 언급했듯이 자동 압축은 예측 불가능하고 중요한 컨텍스트가 손실될 수 있습니다.

**권장 워크플로우**:

```bash
# 30-40% 남았을 때 (60-70% 사용 시점)
/clear
/catchup

# 이렇게 하면:
# 1. 예측 가능한 상태로 초기화
# 2. git 변경사항 기반으로 필요한 컨텍스트만 재구축
# 3. 자동 compact로 인한 정보 손실 방지
```

컨텍스트가 부족해지기 전에 미리 정리하는 습관이 생산성을 높입니다.

## CLAUDE.md 작성 원칙

Shrivu는 CLAUDE.md를 "에이전트의 헌법"이라고 표현합니다.

### 가드레일 우선, 매뉴얼 아님

```markdown
## 하지 말 것
- console.log 디버깅 사용 금지

## 대신
- 구조화된 로깅 라이브러리 사용 (pino, winston)
```

단순히 "하지 마"만 있으면 에이전트가 대안을 찾지 못합니다. 항상 대안을 함께 제시합니다.

### 파일 임베딩 피하기

```markdown
## 잘못된 예
@src/config/database.ts 참고해서 작업하세요

## 올바른 예
데이터베이스 설정은 src/config/database.ts를 참고하세요.
필요시 Read 도구로 직접 읽으세요.
```

`@파일` 임베딩은 매 세션마다 컨텍스트를 소비합니다. 필요할 때만 읽도록 안내합니다.

### 복잡한 CLI 단순화

```markdown
## 내부 배포 도구
복잡한 옵션은 `./scripts/deploy.sh --help` 참조

기본 사용:
- 개발: `./scripts/deploy.sh dev`
- 스테이징: `./scripts/deploy.sh staging`
```

복잡한 CLI는 래퍼 스크립트로 단순화하고, CLAUDE.md에는 간단한 예시만 포함합니다.

## 커스텀 명령어 실전 예시

### /pr - PR 준비 자동화

`.claude/commands/pr.md`:

```markdown
PR을 준비합니다:

1. 코드 정리
   - 불필요한 console.log 제거
   - import 정리
   - 포맷팅 확인

2. 테스트 실행
   - `pnpm test` 실행
   - 실패하면 수정

3. 변경사항 스테이징
   - `git add -A`

4. PR 제목과 본문 작성
   - 변경사항 요약
   - Breaking changes 명시

5. 결과 보고
```

### /review - 코드 리뷰

`.claude/commands/review.md`:

```markdown
현재 브랜치의 변경사항을 리뷰합니다:

1. `git diff main...HEAD` 확인
2. 다음 관점으로 검토:
   - 보안 이슈
   - 성능 문제
   - 코드 스타일
   - 테스트 커버리지
3. 발견된 이슈를 우선순위별로 정리
4. 개선 제안
```

## Hooks: 테스트 통과 강제

Shrivu는 hooks를 사용해 테스트 통과 전 커밋을 방지합니다.

### 설정

`.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "if echo \"$TOOL_INPUT\" | grep -q 'git commit'; then if [ ! -f /tmp/agent-pre-commit-pass ]; then echo 'BLOCK: 테스트를 먼저 실행하세요'; exit 1; fi; fi"
          }
        ]
      }
    ]
  }
}
```

### 워크플로우

1. 에이전트가 코드 작성
2. `git commit` 시도
3. `/tmp/agent-pre-commit-pass` 파일 확인
4. 없으면 테스트 실행 강제
5. 테스트 통과 시 파일 생성
6. 커밋 허용

## Fire and Forget 워크플로우

Shrivu의 핵심 철학입니다:

1. **컨텍스트 설정**: CLAUDE.md + 커스텀 명령어
2. **작업 위임**: 명확한 목표 전달
3. **결과 확인**: PR로 최종 판단

중간 과정에 개입하지 않고, 에이전트가 자율적으로 판단하도록 합니다. 이를 위해:

- 복잡한 커스텀 에이전트보다 단순한 CLI 선호
- 강제된 워크플로우보다 자율적 위임 선호
- 컨텍스트 게이트키핑 피하기

## 실무 적용 팁

### 1. 점진적 도입

처음부터 모든 기능을 사용하지 않습니다. `/catchup`부터 시작해서 점진적으로 확장합니다.

### 2. 팀 공유

`.claude/` 디렉토리를 git에 커밋하여 팀 전체가 동일한 워크플로우를 사용합니다.

### 3. 반복 패턴 발견

자주 하는 작업을 발견하면 커스텀 명령어로 자동화합니다.

### 4. 토큰 비용 모니터링

```bash
/context
```

정기적으로 토큰 사용량을 확인하고, 70% 이상이면 `/clear + /catchup`을 실행합니다.

## 응용: 회사 개발자의 출퇴근 워크플로우

Shrivu의 블로그를 읽으면서 동료 개발자가 사용하는 방식이 떠올랐습니다. 동료는 회사에서 출퇴근 시 Claude Code의 컨텍스트를 관리하기 위해 **문서 기반 접근**을 사용하고 있었습니다. Shrivu의 Git 기반 `/catchup`과 동료의 문서 기반 방식을 조합하면 더 효과적일 것 같아 정리해봤습니다.

### /daily-start: 출근 시 컨텍스트 복구

`.claude/commands/daily-start.md`:

```markdown
어제 작업을 이어받아 오늘 개발을 시작합니다.

1. 진행상황 파악
   - `outputs/daily-progress/` 에서 가장 최근 파일 읽기
   - `outputs/progress/checklist-status.md` 확인
   - 어제 완료한 작업과 오늘 P0 목표 파악

2. 코드 상태 확인
   - `git status` 로 uncommitted 변경사항 확인
   - `git log --oneline -5` 로 최근 커밋 확인

3. 오늘 계획 제안
   - 어제 미완료 작업 우선
   - 새로운 P0 목표 제시
   - 예상 이슈 언급

간결하게 보고하고, 바로 작업을 시작할 수 있도록 준비하세요.
```

### /daily-end: 퇴근 시 상태 저장

`.claude/commands/daily-end.md`:

```markdown
오늘 작업을 정리하고 내일을 위해 상태를 저장합니다.

1. 오늘 진행사항 파일 생성
   - 파일: `outputs/daily-progress/$(date +%Y-%m-%d).md`
   - 완료한 작업 목록
   - 작성한 테스트 케이스
   - 발견한 이슈와 해결 방법

2. 체크리스트 동기화
   - `outputs/progress/checklist-status.md` 업데이트
   - 완료 항목 체크, 진행률 계산

3. 내일 준비
   - P0 우선순위 작업 명시
   - 예상되는 어려움이나 필요한 조사 항목
   - 다른 팀원에게 문의할 사항

4. 커밋
   - 작업 중인 코드 커밋 (WIP 포함)
   - 진행상황 문서 커밋
```

### 출퇴근 워크플로우 비교

| 시점 | `/catchup` 방식 | 문서 기반 방식 |
|------|----------------|---------------|
| **출근** | `/clear` → `/catchup` | `/daily-start` |
| **컨텍스트 소스** | Git diff | 진행상황 문서 + Git |
| **퇴근** | (별도 작업 없음) | `/daily-end` |
| **장점** | 빠른 코드 컨텍스트 | 의사결정 맥락 보존 |
| **적합 상황** | 단기 집중 작업 | 장기 프로젝트, 팀 협업 |

### 진행상황 문서 구조

`outputs/` 폴더를 다음과 같이 구성합니다:

```
outputs/
├── daily-progress/          # 일일 개발 로그
│   ├── 2025-01-15.md
│   ├── 2025-01-16.md
│   └── 2025-01-17.md
├── progress/                # 전체 진행상황
│   ├── checklist-status.md  # 체크리스트 현황
│   └── test-progress.md     # 테스트 진행률
└── artifacts/               # 구현 산출물
    ├── types.md             # 구현된 타입 정의
    ├── functions.md         # 구현된 함수 목록
    └── architecture.md      # 현재 아키텍처
```

### 일일 진행상황 템플릿

`outputs/daily-progress/YYYY-MM-DD.md`:

```markdown
# 일일 진행사항 - YYYY-MM-DD

## 오늘 목표
- [ ] **P0**: [가장 중요한 작업]
- [ ] **P0**: [두 번째 중요 작업]
- [ ] **P1**: [시간 여유시 진행]

## 완료한 작업
- [x] Story 1.2: API 엔드포인트 구현
- [x] 테스트 케이스 3개 추가

## 발견한 이슈
### 이슈 1: [제목]
- **증상**: [문제 상황]
- **원인**: [원인 분석]
- **해결**: [해결 방법]

## 내일 우선순위
1. Story 1.3 시작
2. 코드 리뷰 반영

## 메모
- [팀원에게 문의할 사항]
- [추가 조사 필요한 항목]
```

### 정리: 하루 워크플로우

```bash
# 출근해서 Claude Code 실행
claude
/daily-start          # 어제 진행상황 파악
/catchup              # 코드 변경사항 확인

# 작업 중 (컨텍스트 70% 이상 사용 시)
/clear
/catchup              # 빠른 컨텍스트 복구

# 퇴근 전
/daily-end            # 오늘 진행상황 저장
```

## 참고 자료

### 원문
- [How I Use Every Claude Code Feature](https://blog.sshh.io/p/how-i-use-every-claude-code-feature) - Shrivu Shankar

### 관련 문서
- [Claude Code Documentation](https://docs.anthropic.com/claude-code)
- [Claude Code Custom Commands](https://docs.anthropic.com/claude-code/commands)

### 관련 블로그
- [CLAUDE.md 최적화 여정: AI가 패턴을 무시하는 이유와 해결책](https://blog.imprun.dev/57) - CLAUDE.md 작성 실전 경험
- [AI Agent를 위한 Frontend 개발 가이드](https://blog.imprun.dev/68) - AGENTS.md 작성 가이드
- [Claude, Gemini, Codex에서 AGENTS.md 설정하기](https://blog.imprun.dev/69) - AI 에이전트 통합
