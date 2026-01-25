# uv sync: Python 패키지 개발 모드의 새로운 표준

> **작성일**: 2026년 1월 16일
> **카테고리**: Python, 개발 도구, 패키징
> **키워드**: uv, Python, editable install, 개발 모드, pip, 패키지 개발

[##_Image|kage@cCWt7J/dJMcaaRzNax/AAAAAAAAAAAAAAAAAAAAAO1pBU6m2k0NmlppSkwbIecMUjBpDH2m5jib0vOHwcSJ/img.jpg|alignCenter|width="100%"|_##]

## 요약

uv는 Python 패키지를 **기본적으로 editable mode로 설치**한다. `uv sync` 한 번으로 프로젝트와 의존성이 모두 개발 모드로 설치되어, 코드 수정 시 재설치 없이 즉시 반영된다. 기존 pip의 `pip install -e .`보다 10배 빠르며, 현대적인 Python 패키지 개발의 표준이 되고 있다.

## 왜 Editable Install이 필요한가?

### 반복 작업의 고통

Python 패키지를 개발할 때 다음 과정을 수백 번 반복한다:

```bash
# 1. 코드 수정
vim mypackage/utils.py

# 2. 패키지 재설치
pip install .

# 3. 테스트
python test.py

# 4. 또 수정
vim mypackage/utils.py

# 5. 또 재설치
pip install .

# 무한 반복...
```

**문제**: 매번 재설치는 시간 낭비다.

### 일반 설치 vs Editable 설치

**일반 설치** (`pip install .`):
```
프로젝트 디렉토리 → [복사] → site-packages/
```
- 파일을 site-packages로 **복사**
- 코드 수정 → 재설치 필요

**Editable 설치** (`pip install -e .` 또는 `uv sync`):
```
프로젝트 디렉토리 ← [링크] ← site-packages/
```
- 프로젝트 디렉토리를 **링크**만 연결
- 코드 수정 → **즉시 반영**

### 비유

**일반 설치**: 책을 복사해서 책장에 넣기
- 원본 수정해도 책장의 책은 그대로

**Editable 설치**: 책의 위치를 메모만 해두기
- 원본 수정하면 바로 확인 가능

## uv의 Editable Install: 기본이 개발 모드

### 핵심 철학

uv는 **"개발하는 동안은 항상 editable이어야 한다"**는 철학을 가진다.

```bash
# uv: editable이 기본
uv sync

# 이것만으로 프로젝트가 editable로 설치됨
# 별도 플래그 불필요
```

### 프로젝트 구조 예시

```
my-package/
├── pyproject.toml
├── src/
│   └── mypackage/
│       ├── __init__.py
│       └── utils.py
└── tests/
    └── test_utils.py
```

`pyproject.toml`:

```toml
[project]
name = "mypackage"
version = "0.1.0"
dependencies = []

[project.optional-dependencies]
dev = ["pytest", "black", "mypy"]
```

### 개발 워크플로우

```bash
# 1. 프로젝트 동기화 (editable 설치)
cd my-package
uv sync

# 2. 코드 수정
vim src/mypackage/utils.py

# 3. 즉시 테스트 (재설치 불필요!)
uv run pytest

# 4. 또 수정
vim src/mypackage/utils.py

# 5. 또 즉시 테스트
uv run pytest

# 재설치 없이 무한 반복 가능
```

### pip와의 비교

| 작업 | pip | uv |
|------|-----|-----|
| Editable 설치 | `pip install -e .` | `uv sync` |
| 개발 의존성 | `pip install -e .[dev]` | `uv sync --group dev` (자동) |
| 실행 | `python script.py` | `uv run python script.py` |
| 재설치 여부 | -e 플래그 있어야 editable | **기본이 editable** |
| 속도 | 느림 (Python 구현) | **매우 빠름** (Rust 구현) |

### 성능 비교

실측 예시:

```bash
# pip (전통적 방식)
$ time pip install -e .
real    0m8.543s

# uv (현대적 방식)
$ time uv sync
real    0m0.892s
```

uv가 **약 10배 빠르다**.

## uv 사용법

### 1. 기본 동기화

```bash
# 프로젝트를 editable로 설치
uv sync

# 이제 어디서든 import 가능
python
>>> import mypackage
>>> mypackage.__file__
'/home/user/projects/my-package/src/mypackage/__init__.py'
```

경로가 프로젝트 디렉토리를 가리키면 성공이다.

### 2. 개발 의존성 포함

```toml
[project.optional-dependencies]
dev = ["pytest", "black", "mypy"]
docs = ["sphinx"]
```

```bash
# 모든 optional dependencies 포함
uv sync --all-extras

# 특정 그룹만
uv sync --extra dev
```

### 3. 로컬 의존성도 Editable로

여러 패키지를 동시에 개발할 때:

```toml
# my-api/pyproject.toml
[project]
dependencies = ["mylib"]

[tool.uv.sources]
mylib = { path = "../my-lib", editable = true }
```

```bash
# my-api 프로젝트에서
uv sync

# mylib도 자동으로 editable 설치됨
```

이제 `my-lib`를 수정하면 `my-api`에서 즉시 반영된다.

### 4. Non-editable이 필요한 경우

프로덕션이나 CI 환경에서는:

```bash
# Non-editable 설치
uv sync --no-editable

# 또는 프로덕션 전용 명령
uv pip install .
```

### 5. 스크립트 실행

```bash
# uv run으로 실행 (격리된 환경)
uv run python script.py

# uv run으로 테스트
uv run pytest

# uv run으로 린터
uv run black src/
uv run mypy src/
```

`uv run`은 자동으로 가상환경을 활성화하고 실행한다.

## 실전 활용 사례

### 1. 라이브러리 개발

```bash
# 프로젝트 구조
mylibrary/
├── pyproject.toml
├── src/
│   └── mylibrary/
│       ├── __init__.py
│       └── core.py
└── tests/
    └── test_core.py

# 한 번만 동기화
cd mylibrary
uv sync

# 개발 루프
vim src/mylibrary/core.py  # 수정
uv run pytest              # 즉시 테스트
vim src/mylibrary/core.py  # 또 수정
uv run pytest              # 또 즉시 테스트
```

### 2. 모노레포: 여러 패키지 동시 개발

```bash
workspace/
├── packages/
│   ├── core/      # 핵심 라이브러리
│   ├── api/       # API (core 의존)
│   └── cli/       # CLI (api, core 의존)
```

각 패키지의 `pyproject.toml`:

```toml
# packages/api/pyproject.toml
[project]
dependencies = ["mycore"]

[tool.uv.sources]
mycore = { path = "../core", editable = true }
```

```toml
# packages/cli/pyproject.toml
[project]
dependencies = ["myapi", "mycore"]

[tool.uv.sources]
myapi = { path = "../api", editable = true }
mycore = { path = "../core", editable = true }
```

```bash
# CLI 패키지에서 동기화
cd packages/cli
uv sync

# 이제 core, api, cli 모두 editable
# 어느 패키지를 수정해도 즉시 반영
```

### 3. 오픈소스 라이브러리 디버깅

```bash
# 라이브러리 클론
git clone https://github.com/some/library.git
cd library

# Editable 설치
uv sync

# 소스 수정하며 디버깅
vim src/library/module.py

# 내 프로젝트에서 즉시 수정된 라이브러리 사용
cd ~/my-project
uv run python app.py  # library의 수정사항 반영됨
```

### 4. 플러그인 시스템 개발

```bash
plugins/
├── main-app/
├── plugin-auth/
├── plugin-payment/
└── plugin-analytics/
```

각 플러그인을 editable로:

```bash
# 메인 앱
cd main-app && uv sync

# 플러그인들
cd ../plugin-auth && uv sync
cd ../plugin-payment && uv sync
cd ../plugin-analytics && uv sync

# 모든 플러그인을 수정하며 실시간 테스트
cd main-app
uv run python main.py  # 모든 플러그인 변경사항 즉시 반영
```

## pip로 Editable Install 하기 (레거시)

기존 프로젝트나 팀에서 pip를 사용한다면:

### pip install -e

```bash
# 현재 디렉토리를 editable로
pip install -e .

# 특정 경로
pip install -e /path/to/package

# 의존성 포함
pip install -e .[dev]
```

### pip의 한계

```bash
# 1. 느림
$ time pip install -e .
real    0m8.543s

# 2. 명시적 플래그 필요
pip install -e .  # -e 빼먹으면 일반 설치

# 3. 의존성 해석 느림
pip install -e .[dev,docs,test]  # 수 초 대기

# 4. 가상환경 별도 관리
python -m venv .venv
source .venv/bin/activate
pip install -e .
```

### uv로 마이그레이션

기존 pip 명령어를 uv로 전환:

| pip | uv |
|-----|-----|
| `pip install -e .` | `uv sync` |
| `pip install -e .[dev]` | `uv sync --extra dev` |
| `python script.py` | `uv run python script.py` |
| `pytest` | `uv run pytest` |

추가 이점:
- **10배 빠른 속도**
- **자동 가상환경 관리**
- **lock 파일로 재현 가능성**

## 동작 원리

### Editable Install의 내부 구조

패키지를 editable로 설치하면 링크 파일이 생성된다:

```bash
$ ls .venv/lib/python3.11/site-packages/ | grep editable
__editable___mypackage-0.1.0-py3.11.pth
__editable___mypackage_0_1_0_finder.py
```

### .pth 파일

```python
# __editable___mypackage-0.1.0-py3.11.pth
import __editable___mypackage_0_1_0_finder; \
__editable___mypackage_0_1_0_finder.install()
```

Python 시작 시 이 파일이 자동 실행된다.

### Finder 파일

```python
# __editable___mypackage_0_1_0_finder.py
MAPPING = {
    'mypackage': '/home/user/projects/my-package/src/mypackage'
}

def install():
    # Import hook 등록
    # mypackage import 시 MAPPING 경로로 리다이렉트
```

### Import 과정

```mermaid
graph LR
    Import[import mypackage]
    Finder[Editable Finder]
    Mapping[경로 매핑]
    Source[프로젝트 소스]
    Module[모듈 로드]

    Import --> Finder
    Finder --> Mapping
    Mapping --> Source
    Source --> Module

    style Import stroke:#2563eb,stroke-width:3px
    style Finder stroke:#16a34a,stroke-width:2px
    style Source stroke:#ea580c,stroke-width:3px
    style Module stroke:#dc2626,stroke-width:2px
```

1. `import mypackage` 실행
2. Editable finder가 가로챔
3. 매핑 테이블에서 실제 경로 확인
4. 프로젝트 소스 디렉토리에서 로드

## uv의 고급 기능

### 1. Workspace 지원

`uv.toml` (워크스페이스 루트):

```toml
[workspace]
members = ["packages/*"]
```

```bash
# 한 번에 모든 패키지 동기화
uv sync --workspace

# 모든 패키지가 editable로 설치됨
```

### 2. Lock 파일로 재현성

```bash
# 처음 동기화 시 uv.lock 생성
uv sync

# 다른 환경에서 동일한 버전 설치
uv sync --frozen
```

CI/CD에서 정확히 같은 환경을 재현할 수 있다.

### 3. 빠른 의존성 추가

```bash
# 의존성 추가 + 동기화 (한 번에)
uv add requests

# 개발 의존성 추가
uv add --dev pytest

# Editable 상태 유지됨
```

### 4. 선택적 동기화

```bash
# 특정 패키지만 업데이트
uv sync --package mypackage

# 의존성 제외 (프로젝트만)
uv sync --no-deps
```

## 주의사항

### 1. C/C++ 확장 모듈

Python 파일만 editable이다. C 확장 모듈은 재빌드가 필요하다:

```bash
# C 코드 수정 후
uv sync  # 재빌드 트리거
```

### 2. IDE Import 해석 문제

VSCode의 Pylance 등은 PEP-660 import hook을 제대로 인식 못할 수 있다.

**증상**:
```python
import mypackage  # "Import could not be resolved" 경고
```

실제로는 실행되지만 IDE가 경고를 표시한다.

**해결책**:
1. VSCode 재시작
2. Python 인터프리터 재선택 (Cmd/Ctrl+Shift+P → "Python: Select Interpreter")
3. `.venv` 경로를 명시적으로 선택

### 3. 프로덕션 배포

Editable은 개발 전용이다. 프로덕션에서는:

```bash
# ❌ 프로덕션에서 이러면 안 됨
uv sync

# ✅ Non-editable 설치
uv sync --no-editable

# ✅ 빌드하여 배포
uv build
uv pip install dist/mypackage-0.1.0-py3-none-any.whl
```

### 4. 경로 이동

Editable 설치는 **절대 경로**를 사용한다:

```bash
# Editable 설치
cd /home/user/project
uv sync

# 프로젝트 이동
mv /home/user/project /opt/project

# ❌ Import 실패
python -c "import mypackage"  # ModuleNotFoundError
```

해결:

```bash
cd /opt/project
uv sync  # 새 경로로 재설치
```

### 5. 의존성 메타데이터 변경

코드 변경은 즉시 반영되지만, 메타데이터 변경은 재동기화 필요:

```bash
# pyproject.toml 수정
vim pyproject.toml

# 재동기화 필요
uv sync
```

## pip vs uv 완전 비교

### 설치

| 작업 | pip | uv |
|------|-----|-----|
| Editable 설치 | `pip install -e .` | `uv sync` |
| 개발 의존성 | `pip install -e .[dev]` | `uv sync --extra dev` |
| 속도 | 8초 | **0.9초** |

### 실행

| 작업 | pip | uv |
|------|-----|-----|
| 가상환경 활성화 | `source .venv/bin/activate` | 불필요 |
| 스크립트 실행 | `python script.py` | `uv run python script.py` |
| 테스트 | `pytest` | `uv run pytest` |

### 의존성 관리

| 작업 | pip | uv |
|------|-----|-----|
| 의존성 추가 | `vim pyproject.toml` + `pip install -e .` | `uv add package` |
| Lock 파일 | 별도 도구 필요 | `uv.lock` 자동 생성 |
| 재현성 | requirements.txt 수동 관리 | `uv sync --frozen` |

### 모노레포

| 작업 | pip | uv |
|------|-----|-----|
| 로컬 패키지 링크 | 각각 `pip install -e` | `tool.uv.sources` + `uv sync` |
| 워크스페이스 | 미지원 | `uv sync --workspace` |

## 마이그레이션 가이드

### Step 1: uv 설치

```bash
# macOS/Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"

# Homebrew
brew install uv
```

### Step 2: 기존 프로젝트 전환

```bash
# 기존 가상환경 제거 (선택)
rm -rf .venv

# uv로 동기화
uv sync

# 테스트
uv run pytest
```

### Step 3: CI/CD 업데이트

**Before (pip)**:
```yaml
- name: Install dependencies
  run: |
    python -m venv .venv
    source .venv/bin/activate
    pip install -e .[dev]

- name: Run tests
  run: |
    source .venv/bin/activate
    pytest
```

**After (uv)**:
```yaml
- name: Setup uv
  run: curl -LsSf https://astral.sh/uv/install.sh | sh

- name: Run tests
  run: uv run pytest
```

훨씬 간단하고 빠르다.

### Step 4: 팀원 교육

```bash
# 기존 명령어 → uv 명령어
pip install -e .        → uv sync
pip install requests    → uv add requests
python script.py        → uv run python script.py
pytest                  → uv run pytest
```

## 문제 해결

### "No pyproject.toml found"

**원인**: 프로젝트 루트가 아닌 디렉토리에서 실행

**해결**:
```bash
cd /path/to/project  # pyproject.toml이 있는 디렉토리로 이동
uv sync
```

### Import가 안 됨

**확인**:
```bash
$ uv run python -c "import sys; print(sys.path)"
```

프로젝트 경로가 포함되어 있는지 확인.

**해결**:
```bash
# 재동기화
uv sync

# 또는 가상환경 재생성
rm -rf .venv
uv sync
```

### 변경사항이 반영 안 됨

**원인**: Python 인터프리터가 모듈을 캐시

**해결**:
```python
# Python 재시작
exit()

# 새 세션에서
uv run python
>>> import mypackage  # 변경사항 반영됨
```

### uv가 느림

**원인**: 네트워크 문제 또는 대량 의존성

**해결**:
```bash
# 캐시 확인
uv cache clean

# 오프라인 모드 (캐시만 사용)
uv sync --offline
```

## 요약

### Editable Install이란?

| 항목 | 일반 설치 | Editable 설치 |
|------|----------|---------------|
| **파일 위치** | site-packages에 복사 | 프로젝트 디렉토리 링크 |
| **코드 수정 시** | 재설치 필요 | 즉시 반영 |
| **용도** | 프로덕션, 배포 | 개발, 디버깅 |
| **속도** | 느림 (복사) | 빠름 (링크만) |

### uv의 장점

1. **기본이 Editable**: 별도 플래그 불필요
2. **10배 빠른 속도**: Rust 구현
3. **자동 가상환경**: 활성화 불필요
4. **Lock 파일**: 재현 가능한 환경
5. **Workspace 지원**: 모노레포 최적화

### 권장 사항

**신규 프로젝트**: uv 사용 강력 권장
- 현대적이고 빠름
- 개발 경험 우수

**기존 프로젝트**: 점진적 마이그레이션
- `uv sync`로 전환 용이
- 기존 pip와 혼용 가능

Editable install은 Python 패키지 개발의 필수 기능이며, uv는 이를 가장 편리하고 빠르게 사용할 수 있는 도구다.

## 참고 자료

### uv 공식 문서
- [uv: Managing packages](https://docs.astral.sh/uv/pip/packages/)
- [uv: Configuring projects](https://docs.astral.sh/uv/concepts/projects/config/)
- [Using Python UV with Editable Source Code Dependencies](https://lonelyneuron.substack.com/p/using-python-uv-with-editable-source)

### Python 패키징 표준
- [PEP-660: Editable installs for pyproject.toml based builds](https://peps.python.org/pep-0660/)
- [setuptools: Development Mode](https://setuptools.pypa.io/en/latest/userguide/development_mode.html)

### pip (레거시)
- [pip install 문서](https://pip.pypa.io/en/stable/cli/pip_install/)
- [How to Use pip install -e](https://medium.com/@garybao/how-to-use-pip-install-e-for-python-package-development-506ac33235e7)
