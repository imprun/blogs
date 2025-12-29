# Google Antigravity 업데이트 주의: 같은 폴더에서 AI 도구 동시 사용 시 작업 손실 위험

> **작성일**: 2025년 12월 22일
> **카테고리**: Tools, AI, Warning
> **키워드**: Google Antigravity, Claude Code, Git, 작업 손실, AI 코딩 도구

## 요약

Google Antigravity와 Claude Code를 같은 작업 폴더에서 동시에 사용하던 중, Antigravity 업데이트 시 `git checkout`이 실행되어 Claude Code로 수정한 파일이 전부 초기화되는 사고가 발생했다. AI 코딩 도구를 동시에 사용할 때는 반드시 작업 폴더를 분리하거나, 수시로 커밋하는 습관이 필요하다.

## 사고 경위

### 상황

- 동일한 Git 저장소 폴더를 VS Code에서 열고 작업
- Claude Code로 프론트엔드 에픽을 멀티 에이전트로 진행 중
- **약 40개 파일**이 수정된 상태로 누적
- 동시에 Google Antigravity도 같은 폴더에서 다른 작업 진행

### 사고 발생

1. Google Antigravity 업데이트 알림 발생
2. 무심코 업데이트 진행
3. Antigravity가 내부적으로 `git checkout` 또는 유사한 명령 실행
4. Claude Code로 수정하던 **40개 파일의 uncommitted 변경사항이 전부 삭제됨**

### 손실된 작업

- 프론트엔드 에픽 전체 진행분 (40개 파일)
- 멀티 에이전트로 수 시간 동안 누적된 작업

## 원인 분석

### Antigravity의 업데이트 동작

Google Antigravity는 업데이트 시 작업 디렉토리의 Git 상태를 초기화하는 것으로 추정된다:

```
# 추정되는 내부 동작
git checkout .
# 또는
git reset --hard
```

이는 Antigravity 자체의 코드 동기화를 위한 것으로 보이나, 같은 폴더에서 다른 도구로 작업 중인 파일까지 영향을 받는다.

### 문제의 핵심

| 도구 | 역할 | 파일 접근 |
|-----|------|---------|
| Claude Code | 블로그 마크다운 편집 | 읽기/쓰기 |
| Google Antigravity | 별도 작업 | Git 상태 조작 |

두 도구 모두 같은 Git 저장소를 공유하지만, 서로의 작업을 인지하지 못한다.

## 재발 방지 방안

### 1. 작업 폴더 분리 (권장)

```
C:\Users\junsi\WORK\
├── blogs/           # Claude Code 전용
├── antigravity-work/ # Antigravity 전용
└── other-projects/
```

AI 도구별로 별도의 폴더에서 작업하면 충돌을 원천 차단할 수 있다.

### 2. 수시 커밋 습관화

```bash
# 의미 있는 변경이 있을 때마다
git add -A && git commit -m "WIP: 작업 중간 저장"
```

커밋된 변경사항은 `git checkout`이나 `git reset`으로 삭제되지 않는다.

### 3. Git Stash 활용

도구 전환 전에 임시 저장:

```bash
git stash push -m "Claude Code 작업 중"

# 나중에 복원
git stash pop
```

### 4. VS Code의 Timeline 기능 확인

VS Code는 파일 변경 이력을 자체적으로 보관한다:

1. 파일 우클릭 → "Open Timeline"
2. 이전 버전 선택 → 복원

단, 이 방법은 VS Code가 열려 있을 때만 유효하다.

### 5. Antigravity 업데이트 시 주의

- 업데이트 전 uncommitted 변경사항 확인
- 중요한 작업 중에는 업데이트 연기
- 가능하다면 별도 터미널에서 `git status` 확인 후 진행

## 교훈

### AI 코딩 도구의 공존 문제

현재 AI 코딩 도구들은 서로의 존재를 인지하지 못한다:

- Claude Code: 파일 시스템 직접 조작
- Cursor: VS Code 기반 편집
- Google Antigravity: Git 저장소 조작
- GitHub Copilot: 에디터 내 제안

같은 프로젝트에서 여러 도구를 사용하면 충돌 가능성이 있다.

### 결론

**AI 도구가 편리해도 Git 기본기는 여전히 중요하다.** 자주 커밋하고, 작업 폴더를 분리하는 습관이 사고를 방지한다.

## 참고

- 이 사고로 인해 프론트엔드 에픽 전체를 처음부터 다시 진행해야 했다
- Google Antigravity의 정확한 업데이트 동작은 문서화되어 있지 않아 추정에 의존함
- 멀티 에이전트 작업 시 변경 파일이 많아질수록 커밋 없이 누적하는 것은 위험하다

### 관련 블로그

- [Git Worktree로 AI 에이전트 동시 개발하기: 실전 튜토리얼](https://blog.imprun.dev/101) - 이 문제의 근본적 해결책
