# Windows에서 uvicorn 종료 후 포트가 풀리지 않는 문제: 유령 프로세스 추적기

> **작성일**: 2026년 01월 28일
> **카테고리**: Python, Troubleshooting, Windows
> **키워드**: uvicorn, FastAPI, Windows, zombie process, port binding, CTRL_C_EVENT

## 요약

Windows 환경에서 uvicorn을 `--host 0.0.0.0`으로 실행한 뒤 프로세스를 종료하면, 자식 프로세스가 제대로 정리되지 않아 포트가 계속 점유된 상태로 남는다. 소스 코드를 수정해도 이전 프로세스가 응답하기 때문에 변경사항이 반영되지 않는 것처럼 보인다. 근본 원인은 Windows의 `CTRL_C_EVENT` 시그널이 특정 프로세스가 아닌 콘솔 전체에 브로드캐스트되는 구조적 한계에 있다.

## 문제 상황

### 증상

FastAPI + uvicorn으로 개발 중, 다음과 같은 현상이 반복되었다:

1. `uvicorn main:app --host 0.0.0.0 --port 8000 --reload`로 서버 실행
2. `Ctrl+C`로 종료
3. 다시 실행하면 `[WinError 10048] Only one usage of each socket address is normally permitted` 에러 발생
4. 또는 에러 없이 실행되지만, 코드 변경이 반영되지 않음

특히 4번이 치명적이다. 서버가 정상 동작하는 것처럼 보이지만, **이전 프로세스가 여전히 포트를 점유하고 응답**하고 있기 때문에 새로 시작한 서버는 사실상 바인딩에 실패하거나 다른 포트에서 동작한다. 코드를 아무리 수정해도 변경사항이 적용되지 않아 원인을 찾는 데 상당한 시간을 소비했다.

### 환경

- **OS**: Windows 10/11
- **Python**: 3.11+
- **uvicorn**: 0.29.x
- **실행 방식**: `uvicorn main:app --host 0.0.0.0 --port 8000 --reload`

## 근본 원인 분석

### Windows의 프로세스 시그널 구조

Unix 계열 OS에서는 `SIGTERM`, `SIGINT` 등을 특정 프로세스(PID)에 정확히 전달할 수 있다. 반면 Windows는 근본적으로 다른 방식을 사용한다.

**핵심 차이: 시그널 전달 방식**

| 항목 | Unix/Linux | Windows |
|------|-----------|---------|
| 시그널 전달 대상 | 특정 PID | 콘솔 그룹 전체 |
| 종료 시그널 | `SIGTERM` (프로세스 단위) | `CTRL_C_EVENT` (브로드캐스트) |
| 자식 프로세스 정리 | 프로세스 그룹 단위 제어 가능 | 콘솔 공유 프로세스 전체에 전파 |

택배 배송에 비유하면, Unix는 "아파트 302호에 배달"이 가능하고, Windows는 "아파트 전체에 방송"만 가능한 셈이다.

### uvicorn의 reload 메커니즘과 Windows 충돌

uvicorn은 `--reload` 옵션 사용 시 다음과 같이 동작한다:

```
[부모 프로세스: Reloader]
    └── [자식 프로세스: 실제 서버]
```

파일 변경이 감지되면:
1. 부모 프로세스가 자식 프로세스에 종료 시그널 전송
2. 자식 프로세스 종료 후 새 자식 프로세스 생성

문제는 1단계에서 발생한다. uvicorn은 Windows에서 자식 프로세스를 종료할 때 `os.kill(pid, signal.CTRL_C_EVENT)`를 호출한다. 그런데 Windows의 `CTRL_C_EVENT`는 **특정 PID를 대상으로 보낼 수 없다**. 내부적으로 `GenerateConsoleCtrlEvent` API를 호출하며, 이 API는 같은 콘솔에 연결된 **모든 프로세스**에 시그널을 브로드캐스트한다.

```
[터미널/콘솔]
    ├── [부모: Reloader] ← CTRL_C_EVENT 수신
    ├── [자식: 서버]     ← CTRL_C_EVENT 수신
    └── [손자: 워커]     ← CTRL_C_EVENT 수신... 또는 수신 못 함
```

이 과정에서 발생하는 시나리오:

1. **부모가 먼저 죽는 경우**: 자식 프로세스가 고아(orphan)가 되어 포트를 계속 점유
2. **자식이 시그널을 무시하는 경우**: 시그널 핸들러가 제대로 설정되지 않은 자식 프로세스가 살아남음
3. **`Ctrl+C` 두 번 연타**: 첫 번째로 graceful shutdown 시작, 두 번째로 부모만 강제 종료되고 자식은 남음

### workers > 1일 때 더 심각한 문제

`--workers 2` 이상으로 실행하면 문제가 더 심각해진다. uvicorn Issue #1872에 따르면, SIGINT가 부모 프로세스에서 `self.should_exit.wait()`에 의해 블로킹되어 워커 프로세스를 정상적으로 종료하지 못한다.

## 진단 방법

### 1. 포트 점유 프로세스 확인 (PowerShell)

```
# 포트 8000을 점유하고 있는 프로세스 PID 확인
Get-NetTCPConnection -LocalPort 8000 | Select-Object OwningProcess

# 해당 PID의 프로세스 정보 확인
Get-Process -Id <PID> | Select-Object Id, ProcessName, Path
```

### 2. 자식 프로세스 추적 (PowerShell)

```
# 특정 PID의 자식 프로세스 확인
Get-CimInstance Win32_Process | Where-Object { $_.ParentProcessId -eq <PID> } |
  Select-Object ProcessId, Name, CommandLine
```

### 3. Python 프로세스 전체 확인 (CMD)

```cmd
# 실행 중인 모든 python 프로세스와 커맨드라인 확인
wmic process where "name='python.exe'" get ProcessId,CommandLine
```

### 4. 강제 종료

```
# 특정 PID 강제 종료
taskkill /F /PID <PID>

# python.exe 전체 종료 (주의: 다른 Python 프로세스도 종료됨)
taskkill /F /IM python.exe
```

## 해결 방안

### 방안 1: `127.0.0.1` 사용 (가장 간단)

외부 접근이 필요 없는 로컬 개발 환경이라면:

```bash
# 변경 전
uvicorn main:app --host 0.0.0.0 --port 8000 --reload

# 변경 후
uvicorn main:app --host 127.0.0.1 --port 8000 --reload
```

`0.0.0.0`은 모든 네트워크 인터페이스에 바인딩하므로, WSL 네트워크 브릿지와 Windows 네트워크 스택 사이에서 포트 충돌이 더 쉽게 발생한다. `127.0.0.1`로 제한하면 문제 빈도가 줄어든다.

### 방안 2: `if __name__ == '__main__'` 가드 (필수)

프로그래밍 방식으로 uvicorn을 실행할 때, 이 가드가 없으면 reload 시 모듈이 재임포트되면서 서버가 중복 실행된다:

```python
# main.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def root():
    return {"message": "hello"}

# 이 가드가 없으면 reload 시 "Address already in use" 발생
if __name__ == "__main__":
    import uvicorn
    uvicorn.run("main:app", host="0.0.0.0", port=8000, reload=True)
```

### 방안 3: 종료 스크립트 작성

개발 중 반복되는 수동 종료를 자동화한다:

```
# kill-uvicorn.ps1
param(
    [int]$Port = 8000
)

$connections = Get-NetTCPConnection -LocalPort $Port -ErrorAction SilentlyContinue
if ($connections) {
    $pids = $connections | Select-Object -ExpandProperty OwningProcess -Unique
    foreach ($pid in $pids) {
        $proc = Get-Process -Id $pid -ErrorAction SilentlyContinue
        if ($proc) {
            Write-Host "Killing process $pid ($($proc.ProcessName))"
            Stop-Process -Id $pid -Force
        }
    }
    Write-Host "Port $Port cleared."
} else {
    Write-Host "Port $Port is not in use."
}
```

사용법:

```
.\kill-uvicorn.ps1
.\kill-uvicorn.ps1 -Port 3000
```

### 방안 4: WSL 환경 사용

WSL은 Linux 커널을 사용하므로 Unix 방식의 프로세스 시그널링이 정상 동작한다:

```bash
# WSL 내부에서 실행
uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

다만 WSL 자체에도 좀비 프로세스 이슈가 있다(WSL Issue #13591). WSL을 사용할 때는 `wsl --shutdown`으로 주기적으로 정리하는 것을 권장한다.

### 방안 5: uvicorn 버전 다운그레이드

uvicorn 0.29.0부터 Windows에서의 자식 프로세스 관리 방식이 변경되면서 문제가 심화되었다. 0.27.1 이하 버전에서는 `Ctrl+C` 종료가 비교적 정상 동작한다:

```bash
pip install "uvicorn<0.28"
```

단, 이는 임시 해결책이다. 보안 패치와 새로운 기능을 포기하게 된다.

## 재발 방지 방안

### 1. 개발 환경 래퍼 스크립트

서버 시작 전 잔여 프로세스를 자동으로 정리하는 래퍼를 사용한다:

```python
# dev.py
import subprocess
import sys

def kill_port(port: int):
    """지정된 포트를 점유하고 있는 프로세스 종료"""
    try:
        result = subprocess.run(
            ["powershell", "-Command",
             f"Get-NetTCPConnection -LocalPort {port} -ErrorAction SilentlyContinue | "
             f"Select-Object -ExpandProperty OwningProcess -Unique"],
            capture_output=True, text=True
        )
        for pid in result.stdout.strip().split('\n'):
            pid = pid.strip()
            if pid and pid.isdigit():
                subprocess.run(["taskkill", "/F", "/PID", pid],
                             capture_output=True)
                print(f"Killed stale process: PID {pid}")
    except Exception:
        pass

if __name__ == "__main__":
    port = 8000
    kill_port(port)
    subprocess.run([
        sys.executable, "-m", "uvicorn",
        "main:app", "--host", "127.0.0.1",
        "--port", str(port), "--reload"
    ])
```

### 2. VS Code Task 연동

`.vscode/tasks.json`에 정리 작업을 등록한다:

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Kill Port 8000",
      "type": "shell",
      "command": "powershell",
      "args": [
        "-Command",
        "Get-NetTCPConnection -LocalPort 8000 -ErrorAction SilentlyContinue | ForEach-Object { Stop-Process -Id $_.OwningProcess -Force -ErrorAction SilentlyContinue }; Write-Host 'Port 8000 cleared'"
      ],
      "problemMatcher": []
    }
  ]
}
```

## 교훈

### 1. "코드 변경이 안 먹힌다"의 진짜 원인을 의심하라

코드를 수정했는데 동작이 바뀌지 않을 때, 대부분은 캐시나 빌드 문제를 먼저 의심한다. 하지만 이 경우처럼 **이전 프로세스가 살아서 응답하고 있는 경우**도 있다. 특히 포트 기반 서비스에서는 `netstat`이나 `Get-NetTCPConnection`으로 먼저 확인하는 습관이 필요하다.

### 2. Windows에서의 프로세스 시그널링은 Unix와 다르다

많은 Python 라이브러리가 Unix 환경을 기준으로 설계되어 있다. `os.kill()`이 Windows에서 어떻게 동작하는지, `CTRL_C_EVENT`가 브로드캐스트 방식이라는 점을 이해하면, 이런 류의 문제를 빠르게 진단할 수 있다.

### 3. 개발 환경에서도 방어적 스크립팅이 필요하다

"로컬이니까 대충 해도 된다"는 생각이 디버깅 시간을 늘린다. 서버 시작 전 포트 정리 스크립트를 두는 것만으로 반복적인 삽질을 방지할 수 있다.

## 참고 자료

### uvicorn GitHub Issues
- [Ctrl+C broken on Windows (Issue #1972)](https://github.com/Kludex/uvicorn/issues/1972)
- [Ctrl+C not close uvicorn with workers > 1 (Issue #1872)](https://github.com/Kludex/uvicorn/issues/1872)
- [uvicorn.run() makes program not killable by Ctrl+C (Issue #1010)](https://github.com/Kludex/uvicorn/issues/1010)
- [Child Processes Not Terminating with 0.29.0 (Discussion #2281)](https://github.com/Kludex/uvicorn/discussions/2281)
- [Windows kills subprocesses on reload (Discussion #2292)](https://github.com/Kludex/uvicorn/discussions/2292)

### Windows / WSL
- [WSL Zombie Process Issue (Issue #13591)](https://github.com/microsoft/WSL/issues/13591)
- [Python os.kill on Windows (Python Bug Tracker)](https://bugs.python.org/issue42962)
- [Python signal module documentation](https://docs.python.org/3/library/signal.html)
