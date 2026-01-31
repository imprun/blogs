# Windows에서 Claude Code가 갑자기 멍청해진 이유: Bash 도구의 빈 출력 버그

> **작성일**: 2026년 1월 31일
> **카테고리**: Claude Code, Windows, Troubleshooting
> **키워드**: Claude Code, Windows, MINGW64, Git Bash, Shell Script, pnpm, npm, docker

## 요약

Windows Git Bash(MINGW64) 환경에서 Claude Code가 `docker`, `pnpm`, `npm` 등의 명령어를 실행할 때 결과가 "(No content)"로 표시되는 버그가 발생했다. 이는 이들 명령어가 실제로는 셸 스크립트 래퍼이기 때문이다. 해결책은 `docker.exe`, `pnpm.cmd`처럼 확장자를 명시하거나, CLAUDE.md에 이를 미리 설정하는 것이다.

## 문제 상황

### 증상

최근 며칠간 Windows에서 Claude Code를 사용할 때 이상한 현상이 발생했다.

```bash
$ docker ps
# Claude Code 응답: (No content)

$ pnpm install
# Claude Code 응답: (No content)

$ npm run build
# Claude Code 응답: (No content)
```

분명히 명령어가 실행되었을 텐데, Claude Code는 아무런 출력도 받지 못한다. 결과를 모르니 다음 단계도 진행하지 못하고, 마치 갑자기 멍청해진 것처럼 동작했다.

### 환경 구성

- **OS**: Windows 11 (Build 26200)
- **Shell**: MINGW64 (Git Bash)
- **Claude Code**: 최신 버전

## 근본 원인 분석

### Windows의 명령어 래퍼 구조

Windows에서 `docker`, `pnpm`, `npm` 같은 명령어를 실행할 때, 실제로 실행되는 것은 `.exe` 파일이 아니다.

```
C:\Program Files\Docker\Docker\resources\bin\docker
├── docker        ← 셸 스크립트 래퍼 (Git Bash용)
├── docker.exe    ← 실제 Windows 실행 파일
└── docker.cmd    ← CMD/PowerShell용 래퍼
```

Git Bash에서 `docker`를 입력하면:
1. PATH에서 `docker`를 찾음
2. 확장자 없는 `docker` 파일이 먼저 매칭됨
3. 이것은 실제로 셸 스크립트 래퍼
4. 래퍼가 내부적으로 `docker.exe`를 호출

### Claude Code Bash 도구의 한계

문제는 Claude Code의 Bash 도구가 **셸 스크립트 파일 실행 시 stdout을 캡처하지 못한다**는 점이다.

| 실행 방식 | 결과 |
|----------|------|
| `docker` (셸 스크립트) | (No content) |
| `docker.exe` (직접 실행) | 정상 출력 |
| `pnpm` (셸 스크립트) | (No content) |
| `pnpm.cmd` (CMD 래퍼) | 정상 출력 |
| `sh -c 'echo hello'` (인라인) | 정상 출력 |

Claude Code가 새로운 서브프로세스를 생성해 스크립트 파일을 실행할 때, MINGW64 환경에서 stdout 캡처가 제대로 이루어지지 않는 것으로 추정된다.

## 해결 방법

### 방법 1: 확장자 명시 (즉시 적용)

명령어에 확장자를 명시하면 셸 스크립트 래퍼를 우회하여 직접 실행된다.

```bash
# Before (실패)
docker ps
pnpm install
npm run build

# After (성공)
docker.exe ps
pnpm.cmd install
npm.cmd run build
```

### 방법 2: CLAUDE.md에 설정 (권장)

매번 확장자를 붙이는 것은 번거롭다. [GitHub Issue #18748](https://github.com/anthropics/claude-code/issues/18748)에 소개된 방법으로, CLAUDE.md 맨 앞에 다음 내용을 추가하면 Claude Code가 자동으로 올바른 명령어를 사용한다.

```markdown
## No Content on Bash output
On Claude >2.1.7 you may be a victim of "Bash tool returns empty output when executing shell script files on Windows (MINGW64)" https://github.com/anthropics/claude-code/issues/18748.
The bug is in Claude Code's subprocess stdout capture mechanism. When a shell script wrapper spawns a child process on MINGW64:

Claude Code → bash → npm (shell script) → node.exe (subprocess)
                                          ↑
                              stdout lost here

The output from the nested subprocess doesn't propagate back through the shell script layer to Claude Code's capture mechanism.

### Workaround
Use .cmd or .exe suffixes (npm.cmd, pnpm.cmd, node.exe)
```

### 방법 3: Shell 함수 래퍼 자동화 (권장)

매번 확장자를 붙이는 것보다 더 나은 방법은 셸 함수로 자동화하는 것이다. `~/.claude/shell-fixes/mingw64-workarounds.sh` 파일을 생성하면 된다.

**Step 1: 디렉토리 생성**

```bash
mkdir -p ~/.claude/shell-fixes
```

**Step 2: 워커라운드 스크립트 작성**

`~/.claude/shell-fixes/mingw64-workarounds.sh`:

```bash
#!/bin/bash
# MINGW64 shell script wrapper workaround
# GitHub Issue: https://github.com/anthropics/claude-code/issues/18748

# Common tools that have .exe/.cmd equivalents
docker() { docker.exe "$@"; }
pnpm() { pnpm.cmd "$@"; }
npm() { npm.cmd "$@"; }
yarn() { yarn.cmd "$@"; }
node() { node.exe "$@"; }
git() { git.exe "$@"; }
```

**Step 3: BASH_ENV 설정**

Claude Code 설정에서 이 스크립트를 로드하도록 설정한다:

```bash
export BASH_ENV=~/.claude/shell-fixes/mingw64-workarounds.sh
```

또는 `~/.bashrc`에 추가:

```bash
if [ -f ~/.claude/shell-fixes/mingw64-workarounds.sh ]; then
    source ~/.claude/shell-fixes/mingw64-workarounds.sh
fi
```

이제 `docker`, `pnpm`, `npm` 등을 확장자 없이 입력하면 자동으로 `.exe` 또는 `.cmd` 버전이 실행된다.

### 방법 4: Claude Code 다운그레이드

버그가 Claude Code 2.1.7 이후 버전에서 발생하므로, 이전 버전으로 다운그레이드하면 문제가 해결된다.

```bash
npm install -g @anthropic-ai/claude-code@2.1.6
```

### 방법 5: macOS 또는 Linux 사용

이 버그는 Windows MINGW64 환경에서만 발생한다. macOS나 Linux에서는 정상 동작한다.

### 방법 6: 인라인 실행 (대안)

스크립트 파일 대신 인라인으로 실행하면 동작한다.

```bash
# 직접 실행 (실패)
./deploy.sh

# 인라인 실행 (성공)
sh -c "$(cat deploy.sh)"
```

## 영향도

이 버그는 Windows MINGW64 환경에서 Claude Code를 사용하는 모든 개발자에게 영향을 미친다.

### 영향 받는 명령어

셸 스크립트 래퍼를 사용하는 모든 도구:
- Docker Desktop
- Node.js 패키지 매니저 (npm, pnpm, yarn)
- 사용자 정의 셸 스크립트 (`.sh` 파일)

### 증상 패턴

Claude Code가 다음과 같이 동작한다면 이 버그를 의심할 수 있다:
- 명령어 실행 후 결과 없이 다음 단계로 넘어감
- 빌드/테스트 결과를 확인하지 못하고 재시도
- 에러가 발생했는지 성공했는지 판단하지 못함

## 교훈

### 1. 플랫폼별 동작 차이 인지

같은 명령어라도 OS와 셸 환경에 따라 실행 메커니즘이 다르다. Windows의 Git Bash는 Linux처럼 보이지만, 내부적으로는 Windows의 제약을 받는다.

### 2. CLAUDE.md의 환경 설정 중요성

플랫폼별 특수 상황을 CLAUDE.md에 명시하면, Claude Code가 맥락을 이해하고 올바르게 동작한다. 이는 단순히 코딩 컨벤션을 넘어 개발 환경의 제약사항도 포함해야 한다.

### 3. 디버깅 시 근본 원인 추적

"명령어가 안 된다"에서 멈추지 않고, "왜 안 되는지"를 파악해야 한다. 이 경우 셸 스크립트 래퍼 → 서브프로세스 생성 → stdout 캡처 실패로 이어지는 연결고리를 추적했다.

## 참고 자료

### 관련 이슈
- [GitHub Issue #18748: Bash tool returns empty output for shell script files on Windows MINGW64](https://github.com/anthropics/claude-code/issues/18748)

### 작동하는 대안 정리

| 실패 | 성공 |
|------|------|
| `docker` | `docker.exe` |
| `pnpm` | `pnpm.cmd` |
| `./script.sh` | `sh -c "$(cat script.sh)"` |
| `bash script.sh` | `. script.sh` (source) |
