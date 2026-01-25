# watchmedo: Python 파일 시스템 변경 감지 도구

> **작성일**: 2026년 1월 16일
> **카테고리**: Python, 개발 도구, 자동화
> **키워드**: Python, watchdog, 파일 모니터링, 자동화, inotify

## 요약

Watchdog는 파일 시스템의 변경 사항(생성, 수정, 삭제)을 실시간으로 감지하는 Python 라이브러리다. Linux의 inotify, macOS의 FSEvents 등 OS별 네이티브 API를 사용하며, API와 명령줄 도구(watchmedo)를 모두 제공한다. 자동 빌드, 테스트 실행, 로그 수집 등 파일 변경 기반 자동화에 활용된다.

## 왜 파일 모니터링이 필요한가?

### 반복 작업의 자동화

개발하다 보면 파일이 변경될 때마다 특정 작업을 반복한다:

```bash
# 코드 수정
$ vim app.py

# 테스트 실행
$ pytest tests/

# 다시 코드 수정
$ vim app.py

# 또 테스트 실행
$ pytest tests/
```

이 과정을 수백 번 반복하면 시간 낭비다. **파일이 저장될 때 자동으로 테스트가 실행**되면 좋지 않을까?

### 폴링의 한계

파일 변경을 감지하는 가장 간단한 방법은 주기적으로 확인하는 것이다(폴링):

```python
import time
import os

last_mtime = os.path.getmtime("config.yaml")

while True:
    current_mtime = os.path.getmtime("config.yaml")
    if current_mtime != last_mtime:
        print("파일이 변경되었습니다!")
        last_mtime = current_mtime
    time.sleep(1)
```

문제점:
- **지연**: 1초마다 확인하면 최대 1초 지연 발생
- **리소스 낭비**: 변경이 없어도 계속 확인
- **확장성 부족**: 수백 개 파일을 감시하면 CPU 사용량 급증

### OS 네이티브 이벤트

Linux의 `inotify`, macOS의 `FSEvents`, Windows의 `ReadDirectoryChangesW`는 커널 레벨에서 파일 변경을 알려준다. 폴링 없이 이벤트 기반으로 작동하므로 효율적이다.

그러나 각 OS의 API가 다르고, 저수준 C API를 직접 사용하기 번거롭다. Watchdog는 이를 **통합된 Python API**로 제공한다.

## Watchdog 핵심 개념

### 1. Observer: 감시자

파일 시스템을 감시하는 백그라운드 스레드다. 호텔의 CCTV 시스템을 생각하면 된다.

```python
from watchdog.observers import Observer

observer = Observer()  # CCTV 시스템 설치
observer.start()       # 감시 시작
```

### 2. EventHandler: 이벤트 처리자

변경 사항이 감지되면 어떻게 할지 정의한다. CCTV에서 이상 징후가 보이면 경비원에게 알리는 것과 같다.

```python
from watchdog.events import FileSystemEventHandler

class MyHandler(FileSystemEventHandler):
    def on_modified(self, event):
        print(f"{event.src_path}가 수정되었습니다")
```

### 3. Schedule: 감시 대상 등록

Observer에게 "어느 경로를 감시할지, 어떤 Handler를 사용할지" 알려준다.

```python
observer.schedule(event_handler, path="./src", recursive=True)
```

### 동작 흐름

```mermaid
graph LR
    FS[파일 시스템]
    OS[OS 커널 이벤트]
    Observer[Observer 스레드]
    Handler[EventHandler]
    Action[작업 실행]

    FS -->|파일 생성/수정/삭제| OS
    OS -->|inotify/FSEvents| Observer
    Observer -->|이벤트 전달| Handler
    Handler -->|콜백 호출| Action

    style FS stroke:#4b5563,stroke-width:2px
    style OS stroke:#2563eb,stroke-width:2px
    style Observer stroke:#16a34a,stroke-width:2px
    style Handler stroke:#ea580c,stroke-width:2px
    style Action stroke:#dc2626,stroke-width:2px
```

## 설치

```bash
# 기본 설치
pip install watchdog

# watchmedo 명령줄 도구 포함
pip install 'watchdog[watchmedo]'
```

## 사용 예시 1: 코드 변경 시 자동 테스트

### 문제: 코드 수정할 때마다 수동으로 pytest 실행

Python 파일을 수정할 때마다 테스트를 실행하고 싶다.

### 해결: Watchdog로 자동화

```python
import time
import subprocess
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

class TestRunner(FileSystemEventHandler):
    def on_modified(self, event):
        # .py 파일이 수정되면
        if event.src_path.endswith('.py'):
            print(f"\n{event.src_path} 수정 감지")
            print("테스트 실행 중...")
            subprocess.run(['pytest', 'tests/'])

if __name__ == "__main__":
    event_handler = TestRunner()
    observer = Observer()
    observer.schedule(event_handler, path='./src', recursive=True)
    observer.start()

    print("코드 변경 감시 중... (Ctrl+C로 종료)")

    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        observer.stop()

    observer.join()
```

실행 결과:

```bash
$ python watch_tests.py
코드 변경 감시 중... (Ctrl+C로 종료)

./src/app.py 수정 감지
테스트 실행 중...
============================= test session starts ==============================
collected 12 items

tests/test_app.py ............                                           [100%]

============================== 12 passed in 0.45s ===============================
```

### Context Manager 패턴

매번 `start()`와 `stop()`을 호출하는 대신, `with` 문을 사용할 수 있다:

```python
from watchdog.observers import Observer

with Observer() as observer:
    observer.schedule(event_handler, path='./src', recursive=True)
    observer.join()  # 무한 대기 (Ctrl+C로 종료)
```

## 사용 예시 2: 로그 파일 실시간 백업

### 문제: 로그 파일이 1MB를 초과하면 자동 압축

로그 디렉토리를 감시하다가 `.log` 파일이 1MB를 넘으면 자동으로 gzip 압축한다.

### 해결

```python
import os
import gzip
import shutil
from watchdog.events import FileSystemEventHandler
from watchdog.observers import Observer

class LogCompressor(FileSystemEventHandler):
    MAX_SIZE = 1024 * 1024  # 1MB

    def on_modified(self, event):
        if event.src_path.endswith('.log'):
            size = os.path.getsize(event.src_path)
            if size > self.MAX_SIZE:
                self.compress_log(event.src_path)

    def compress_log(self, filepath):
        compressed = filepath + '.gz'
        print(f"{filepath} 압축 중... ({os.path.getsize(filepath)} bytes)")

        with open(filepath, 'rb') as f_in:
            with gzip.open(compressed, 'wb') as f_out:
                shutil.copyfileobj(f_in, f_out)

        # 원본 로그 파일 비우기 (삭제하지 않음)
        open(filepath, 'w').close()
        print(f"{compressed} 생성 완료")

if __name__ == "__main__":
    observer = Observer()
    observer.schedule(LogCompressor(), path='./logs', recursive=False)
    observer.start()

    try:
        observer.join()
    except KeyboardInterrupt:
        observer.stop()
```

## watchmedo: 명령줄 도구

Python 코드를 작성하지 않고도 간단한 모니터링을 수행할 수 있다.

### 1. 로그 모드: 이벤트 기록

```bash
watchmedo log \
    --patterns='*.py;*.yaml' \
    --ignore-directories \
    --recursive \
    /path/to/watch
```

출력:

```
on_modified: <FileModifiedEvent: event_type=modified, src_path='./config.yaml', is_directory=False>
on_modified: <FileModifiedEvent: event_type=modified, src_path='./app.py', is_directory=False>
```

### 2. shell-command: 명령 자동 실행

파일이 변경되면 지정한 셸 명령어를 실행한다.

```bash
watchmedo shell-command \
    --patterns='*.py' \
    --recursive \
    --command='pytest tests/' \
    ./src
```

이제 `src/` 디렉토리 내 `.py` 파일이 수정되면 자동으로 `pytest tests/`가 실행된다.

### 3. auto-restart: 프로세스 자동 재시작

개발 서버를 실행 중인데, 코드가 변경되면 자동으로 재시작한다:

```bash
watchmedo auto-restart \
    --patterns='*.py' \
    --recursive \
    -- python app.py
```

Flask나 Django의 auto-reload 기능과 유사하지만, 모든 Python 스크립트에 적용할 수 있다.

### 4. tricks: YAML 설정 파일

복잡한 워크플로우는 `tricks.yaml`로 정의한다:

```yaml
tricks:
- watchdog.tricks.ShellCommandTrick:
    patterns: ["*.py"]
    ignore_patterns: ["*/test_*"]
    shell_command: "pytest tests/"
    wait_for_process: true

- watchdog.tricks.ShellCommandTrick:
    patterns: ["*.rst"]
    shell_command: "sphinx-build -b html docs/ build/"
```

실행:

```bash
watchmedo tricks tricks.yaml
```

## 이벤트 종류

Watchdog는 다음 이벤트를 구분한다:

| 이벤트 | 메서드 | 발생 시점 |
|--------|--------|----------|
| 생성 | `on_created(event)` | 파일/디렉토리 생성 |
| 수정 | `on_modified(event)` | 파일 내용 변경 |
| 삭제 | `on_deleted(event)` | 파일/디렉토리 삭제 |
| 이동 | `on_moved(event)` | 파일/디렉토리 이름 변경 또는 이동 |
| 모든 이벤트 | `on_any_event(event)` | 위 모든 경우 |

### 이벤트 필터링

특정 확장자만 감시하려면 `PatternMatchingEventHandler`를 사용한다:

```python
from watchdog.events import PatternMatchingEventHandler

class MyHandler(PatternMatchingEventHandler):
    patterns = ["*.py", "*.yaml"]
    ignore_patterns = ["*/test_*"]
    ignore_directories = True

    def on_modified(self, event):
        print(f"{event.src_path} 수정됨")
```

## 실전 활용 사례

### 1. 문서 자동 빌드

Sphinx나 MkDocs로 문서를 작성할 때, `.rst` 파일이 수정되면 자동으로 HTML을 재생성한다:

```python
from watchdog.events import PatternMatchingEventHandler
import subprocess

class DocBuilder(PatternMatchingEventHandler):
    patterns = ["*.rst", "*.md"]

    def on_any_event(self, event):
        print("문서 변경 감지, 빌드 중...")
        subprocess.run(['make', 'html'])
```

### 2. 설정 파일 자동 리로드

`config.yaml`이 변경되면 애플리케이션을 재시작하지 않고 설정을 다시 읽는다:

```python
class ConfigWatcher(FileSystemEventHandler):
    def __init__(self, app):
        self.app = app

    def on_modified(self, event):
        if event.src_path.endswith('config.yaml'):
            print("설정 파일 리로드 중...")
            self.app.reload_config()
```

### 3. 파일 동기화

로컬 디렉토리의 변경 사항을 원격 서버로 자동 업로드한다:

```python
class AutoUploader(FileSystemEventHandler):
    def on_modified(self, event):
        if not event.is_directory:
            print(f"{event.src_path} 업로드 중...")
            subprocess.run(['scp', event.src_path, 'user@server:/path/'])
```

### 4. 이미지 자동 압축

`uploads/` 디렉토리에 이미지가 업로드되면 자동으로 리사이즈한다:

```python
from PIL import Image

class ImageOptimizer(PatternMatchingEventHandler):
    patterns = ["*.jpg", "*.png"]

    def on_created(self, event):
        print(f"{event.src_path} 최적화 중...")
        img = Image.open(event.src_path)
        img.thumbnail((800, 800))
        img.save(event.src_path, optimize=True, quality=85)
```

## 주의사항

### 1. 이벤트 중복

텍스트 에디터가 파일을 저장할 때, `on_modified`가 여러 번 호출될 수 있다. 에디터가 임시 파일을 쓰고, 원본을 백업하고, 새 파일로 교체하는 과정에서 여러 이벤트가 발생하기 때문이다.

**해결책**: 디바운싱(일정 시간 내 중복 이벤트 무시)

```python
import time

class DebouncedHandler(FileSystemEventHandler):
    def __init__(self, delay=1.0):
        self.delay = delay
        self.last_modified = {}

    def on_modified(self, event):
        now = time.time()
        path = event.src_path

        # 마지막 처리 후 1초 이내면 무시
        if path in self.last_modified:
            if now - self.last_modified[path] < self.delay:
                return

        self.last_modified[path] = now
        self.process_file(path)

    def process_file(self, path):
        print(f"처리: {path}")
```

### 2. 재귀 감시의 성능

`recursive=True`로 설정하면 하위 디렉토리까지 모두 감시한다. `node_modules/`나 `.git/`처럼 파일이 수천 개인 디렉토리가 있으면 성능 문제가 발생할 수 있다.

**해결책**: 불필요한 디렉토리 제외

```python
import os

class SmartHandler(FileSystemEventHandler):
    IGNORE_DIRS = {'.git', 'node_modules', '__pycache__', '.venv'}

    def dispatch(self, event):
        # 무시할 디렉토리 필터링
        parts = event.src_path.split(os.sep)
        if any(part in self.IGNORE_DIRS for part in parts):
            return
        super().dispatch(event)
```

### 3. 파일 잠금

파일이 아직 쓰기 중일 때 읽으려고 하면 에러가 발생할 수 있다. 특히 큰 파일을 복사할 때 `on_created`가 먼저 발생하고, 파일 쓰기는 나중에 끝난다.

**해결책**: 파일이 완전히 닫힐 때까지 대기

```python
import time

def wait_for_file(filepath, timeout=10):
    """파일 쓰기가 완료될 때까지 대기"""
    start = time.time()
    while time.time() - start < timeout:
        try:
            with open(filepath, 'rb') as f:
                return True
        except IOError:
            time.sleep(0.1)
    return False
```

## 대안 비교

### watchdog vs inotify-tools (Linux)

| 항목 | watchdog | inotify-tools |
|------|----------|---------------|
| 언어 | Python | Shell |
| 크로스 플랫폼 | 지원 | Linux 전용 |
| 사용 편의성 | 높음 | 낮음 (Shell 스크립트) |
| 성능 | 약간 낮음 | 매우 높음 |

**언제 inotify-tools를 쓸까?**
- 순수 Shell 스크립트 환경
- Python 설치 불가
- 최대 성능이 중요

### watchdog vs nodemon (Node.js)

nodemon은 Node.js 프로세스를 자동 재시작하는 도구다.

| 항목 | watchdog | nodemon |
|------|----------|---------|
| 언어 | Python | Node.js |
| 목적 | 범용 파일 모니터링 | 서버 재시작 특화 |
| 유연성 | 높음 (API 제공) | 낮음 (재시작만) |

**언제 nodemon을 쓸까?**
- Node.js 개발 환경
- 서버 재시작만 필요
- 추가 로직 불필요

### watchdog vs fswatch

fswatch는 C로 작성된 크로스 플랫폼 파일 모니터링 도구다.

| 항목 | watchdog | fswatch |
|------|----------|---------|
| 언어 | Python | C |
| 확장성 | Python 코드로 확장 | Shell 명령어만 |
| 성능 | 중간 | 매우 높음 |

## 한계와 고려사항

### 네트워크 파일 시스템

NFS, SMB 같은 네트워크 드라이브는 OS 이벤트를 제대로 발생시키지 않을 수 있다. 이 경우 폴링 방식으로 전환해야 한다:

```python
from watchdog.observers.polling import PollingObserver

observer = PollingObserver()  # inotify 대신 폴링 사용
```

성능은 떨어지지만, 네트워크 드라이브에서도 작동한다.

### 시스템 리소스 한계

Linux의 inotify는 감시 가능한 파일 수에 제한이 있다:

```bash
# 현재 한도 확인
$ cat /proc/sys/fs/inotify/max_user_watches
8192

# 한도 증가 (임시)
$ sudo sysctl fs.inotify.max_user_watches=524288

# 영구 적용
$ echo "fs.inotify.max_user_watches=524288" | sudo tee -a /etc/sysctl.conf
```

## 실무 팁

### 1. 로깅 추가

디버깅을 위해 어떤 이벤트가 발생하는지 로그를 남긴다:

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class LoggingHandler(FileSystemEventHandler):
    def on_any_event(self, event):
        logger.info(f"{event.event_type}: {event.src_path}")
```

### 2. 예외 처리

Handler 내부에서 예외가 발생하면 Observer가 멈춘다. 반드시 try-except로 보호한다:

```python
class SafeHandler(FileSystemEventHandler):
    def on_modified(self, event):
        try:
            self.process_file(event.src_path)
        except Exception as e:
            logger.error(f"파일 처리 실패: {event.src_path}", exc_info=True)
```

### 3. 여러 디렉토리 동시 감시

```python
observer = Observer()
observer.schedule(handler1, path='./src', recursive=True)
observer.schedule(handler2, path='./docs', recursive=True)
observer.start()
```

## 요약

Watchdog는 파일 시스템 변경을 효율적으로 감지하는 Python 라이브러리다.

| 장점 | 설명 |
|------|------|
| **크로스 플랫폼** | Linux, macOS, Windows 모두 지원 |
| **이벤트 기반** | 폴링 없이 OS 네이티브 API 사용 |
| **유연성** | Python API로 복잡한 로직 구현 가능 |
| **CLI 도구** | watchmedo로 코드 없이 바로 사용 |

파일 변경 감지가 필요한 자동화 작업에 Watchdog를 활용하면 개발 생산성을 크게 높일 수 있다.

## 참고 자료

### 공식 리소스
- [GitHub: gorakhargosh/watchdog](https://github.com/gorakhargosh/watchdog)
- [Watchdog 문서](https://python-watchdog.readthedocs.io/)
- [PyPI: watchdog](https://pypi.org/project/watchdog/)

### 관련 기술
- [Linux inotify 문서](https://man7.org/linux/man-pages/man7/inotify.7.html)
- [macOS FSEvents](https://developer.apple.com/documentation/coreservices/file_system_events)
