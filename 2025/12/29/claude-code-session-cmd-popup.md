# Claude Code 세션 시작 시 CMD 창이 뜨는 문제 해결하기

Claude Code를 사용하다 보면 세션을 시작할 때마다 CMD(명령 프롬프트) 창이 잠깐 나타났다 사라지는 현상을 경험할 수 있습니다. 이 글에서는 이 현상의 원인과 해결 방법을 알아보겠습니다.

## 문제 현상

Claude Code에서 새 세션을 시작하면 다음과 같은 메시지가 표시됩니다:

```
SessionStart:Callback hook succeeded: Success
```

동시에 Windows에서는 CMD 창이 잠깐 나타났다가 사라집니다. 작업에 방해가 될 수 있는 이 현상은 왜 발생하는 걸까요?

## 원인: Claude Code Hooks

이 현상의 원인은 **Claude Code의 Hook 시스템**입니다.

### Hook이란?

Hook은 특정 이벤트가 발생할 때 자동으로 실행되는 스크립트입니다. Claude Code는 다양한 이벤트에 대해 Hook을 지원합니다:

- `SessionStart`: 새 세션이 시작될 때
- `PreToolUse`: 도구가 실행되기 전
- `PostToolUse`: 도구가 실행된 후
- `Stop`: 세션이 종료될 때

### 플러그인과 Hook

Claude Code의 플러그인들은 자체적인 Hook을 등록할 수 있습니다. 예를 들어, `explanatory-output-style` 플러그인은 다음과 같은 Hook을 가지고 있습니다:

```json
// ~/.claude/plugins/cache/claude-code-plugins/explanatory-output-style/1.0.0/hooks/hooks.json
{
  "description": "Explanatory mode hook that adds educational insights instructions",
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/hooks-handlers/session-start.sh"
          }
        ]
      }
    ]
  }
}
```

이 Hook은 세션이 시작될 때 `session-start.sh` 스크립트를 실행합니다. Windows에서는 셸 스크립트가 실행될 때 CMD 창이 잠깐 나타나게 됩니다.

### 근본 원인: windowsHide 옵션 누락

이 문제는 Claude Code 내부에서 `child_process.spawn()`을 호출할 때 `windowsHide: true` 옵션을 사용하지 않기 때문입니다. 이 옵션이 없으면 Windows에서 자식 프로세스가 실행될 때 콘솔 창이 잠깐 나타납니다.

이 문제는 이미 GitHub에 보고되어 있습니다:
- [#15572 - Windows: Console windows flash when running Bash commands](https://github.com/anthropics/claude-code/issues/15572)
- [#14828 - Windows: Console window flashing when executing tools](https://github.com/anthropics/claude-code/issues/14828)

**제안된 수정 방법** (Claude Code 팀에게):

```javascript
spawn(command, args, {
    ...options,
    windowsHide: process.platform === 'win32'
});
```

하지만 이 수정은 Claude Code 내부 코드 변경이 필요하므로, 우리는 workaround를 사용해야 합니다.

## 해결 방법 (Workaround)

### 방법 1: 해당 플러그인 비활성화

가장 간단한 방법은 Hook을 등록한 플러그인을 비활성화하는 것입니다.

1. Claude의 전역 설정 파일을 엽니다:
   - 경로: `~/.claude/settings.json` (Windows: `C:\Users\{사용자명}\.claude\settings.json`)

2. `enabledPlugins` 섹션에서 해당 플러그인을 `false`로 변경합니다:

```json
{
  "enabledPlugins": {
    "explanatory-output-style@claude-code-plugins": false,
    // ... 다른 플러그인들
  }
}
```

3. Claude Code를 새로 시작합니다.

### 방법 2: Hook을 등록한 플러그인 찾기

어떤 플러그인이 Hook을 등록했는지 모르는 경우, 다음 명령어로 찾을 수 있습니다:

```bash
# Windows (Git Bash)
find ~/.claude -name "*hook*" -type f

# 또는 특정 Hook 타입 검색
grep -r "SessionStart" ~/.claude/plugins --include="*.json"
```

### 방법 3: 플러그인의 Hook만 수정

플러그인의 다른 기능은 유지하면서 Hook만 비활성화하고 싶다면, 해당 플러그인의 `hooks/hooks.json` 파일에서 특정 Hook을 제거하면 됩니다. 다만, 플러그인 업데이트 시 변경사항이 덮어씌워질 수 있습니다.

## 현재 활성화된 Hook 확인하기

현재 어떤 Hook들이 활성화되어 있는지 확인하려면:

```bash
# 모든 hooks.json 파일 찾기
find ~/.claude/plugins -name "hooks.json" -exec echo "=== {} ===" \; -exec cat {} \;
```

## 마무리

Claude Code의 Hook 시스템은 강력한 확장성을 제공하지만, Windows 환경에서는 셸 스크립트 실행 시 CMD 창이 나타나는 부작용이 있을 수 있습니다.

만약 이 현상이 거슬린다면, 해당 플러그인을 비활성화하거나 Hook을 등록하지 않는 대안 플러그인을 찾아보세요.

---

**참고**: 플러그인을 비활성화하면 해당 플러그인이 제공하는 다른 기능들(예: 특별한 출력 스타일, 추가 명령어 등)도 함께 비활성화됩니다. 비활성화 전에 해당 플러그인의 기능을 확인하세요.
