# Claude Code Status Line: 프롬프트 한 줄로 터미널 정보 바 만들기

> **작성일**: 2026년 1월 20일
> **카테고리**: AI Development, Claude Code, Developer Tools
> **키워드**: Status Line, Claude Code, PowerShell, Terminal Customization, Developer Experience

## 요약

Claude Code에서 `/statusline` 명령과 자연어 프롬프트 한 줄이면 터미널 하단에 **모델명, 컨텍스트 사용량, Git 브랜치, 세션 비용**을 표시하는 Status Line이 완성된다. 스크립트 작성도, 설정 파일 편집도 Claude가 알아서 해준다.

## Status Line이란?

Claude Code 터미널 하단에 지속적으로 표시되는 정보 바다. Oh My Zsh의 프롬프트처럼, 작업에 필요한 정보를 한눈에 보여준다.

```
Opus 4.5 | [████████░░░░░░░░░░░░] 40% (80.3K) | imprun @ main | $2.96
```

## 만드는 방법: 프롬프트 한 줄

Claude Code에서 다음과 같이 입력한다:

```
/statusline 윈도우 환경에 글로벌 세팅으로 하고, 스크립트는 분리된 ps1으로 해라.
상태라인에는 모델이름, 진행바, 컨텍스트 사용 퍼센트, 토큰, Git 브랜치, 프로젝트 이름.
그리고 상태라인에 항목에 색상을 넣어보면 좋겠다.
특히 프로그래스바는 70% 이상이면 색에 변화를 주고 경고하고 싶다.
```

**끝이다.**

Claude Code가 자동으로:

1. `~/.claude/statusline.ps1` 스크립트 생성
2. `~/.claude/settings.json`에 statusLine 설정 추가
3. ANSI 색상 코드 적용
4. 70%/90% 임계값에 따른 색상 변화 로직 구현

Claude Code를 재시작하면 바로 적용된다.

## 프롬프트 예시

원하는 정보와 스타일에 따라 프롬프트를 조정하면 된다.

### 기본형

```
/statusline 모델명, 컨텍스트 사용량, 프로젝트명, Git 브랜치 표시해줘
```

### 비용 추적 추가

```
/statusline 모델명, 컨텍스트 퍼센트, 프로젝트, 브랜치, 세션 비용까지 표시.
비용은 $5 넘으면 노란색, $10 넘으면 빨간색으로.
```

### macOS/Linux 환경

```
/statusline bash 스크립트로 만들어줘. jq 사용해도 돼.
모델, 컨텍스트, 브랜치, 비용 표시.
```

### 미니멀 스타일

```
/statusline 최소한으로. 모델명이랑 컨텍스트 퍼센트만.
```

## 결과물 확인

프롬프트 실행 후 생성되는 파일들:

**settings.json:**
```json
{
  "statusLine": {
    "type": "command",
    "command": "powershell.exe -NoProfile -ExecutionPolicy Bypass -File C:\\Users\\username\\.claude\\statusline.ps1"
  }
}
```

**statusline.ps1:** Claude가 자동 생성한 PowerShell 스크립트

## 내부 동작 원리

Status Line 스크립트는 **stdin으로 세션 정보를 JSON으로 받는다**. Claude가 생성한 스크립트는 이 JSON을 파싱해서 원하는 형태로 출력한다.

```json
{
  "model": {
    "id": "claude-opus-4-5-20251101",
    "display_name": "Opus 4.5"
  },
  "cost": {
    "total_cost_usd": 2.96
  },
  "context_window": {
    "context_window_size": 200000,
    "current_usage": {
      "input_tokens": 7,
      "cache_creation_input_tokens": 44542
    }
  },
  "workspace": {
    "project_dir": "C:\\Users\\junsik\\WORK\\imprun"
  }
}
```

### 주의: 주간 쿼터 정보는 없다

`/status`에서 볼 수 있는 **주간 쿼터 사용량(예: 72%)**은 이 JSON에 포함되지 않는다. 쿼터 확인은 `/status` 명령을 사용하자.

## 수정이 필요할 때

Status Line을 변경하고 싶으면 다시 `/statusline` 명령을 사용한다.

```
/statusline 세션 비용 대신 세션 시간을 표시해줘
```

```
/statusline 프로그레스바 너비를 30칸으로 늘려줘
```

Claude가 기존 스크립트를 수정하거나 새로 생성해준다.

## 트러블슈팅

### 색상이 안 나올 때

```
/statusline 색상이 안 나와. NO_COLOR 환경변수 문제인지 확인하고 수정해줘
```

### 스크립트 오류로 표시 안 될 때

```
/statusline 동작 안 해. 디버깅용으로 JSON을 파일로 저장하게 수정해줘
```

## 교훈

### 1. AI 시대의 설정은 대화다

예전에는 스크립트를 직접 작성하고 설정 파일을 편집했다. 이제는 원하는 것을 말하면 된다. `/statusline`과 자연어 프롬프트가 수십 줄의 PowerShell 코드를 대체한다.

### 2. 컨텍스트 모니터링은 필수다

Claude Code에서 가장 중요한 리소스는 **컨텍스트 윈도우**다. 200K 토큰이 가득 차면 대화를 새로 시작해야 한다. Status Line으로 사용량을 상시 확인하면 적절한 타이밍에 `/compact`나 대화 정리를 할 수 있다.

### 3. stdin JSON의 한계를 알자

주간 쿼터 정보는 stdin JSON에 없다. 자체 추적 로직을 구현하면 실제 값과 맞지 않을 수 있다. 처음에 Opus 쿼터를 직접 추적하려다 `-4/5`라는 황당한 값이 나왔던 경험이 있다.

## 참고 자료

- [Claude Code Status Line 공식 문서](https://code.claude.com/docs/en/statusline)
- [ccstatusline - Powerline 스타일](https://github.com/sirmalloc/ccstatusline)
- [claude-powerline - Vim 스타일](https://github.com/Owloops/claude-powerline)
