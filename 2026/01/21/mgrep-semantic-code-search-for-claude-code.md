# mgrep: Claude Code를 위한 시맨틱 코드 검색 도구

> **작성일**: 2026년 01월 21일
> **카테고리**: Developer Tools, AI, Code Search
> **키워드**: mgrep, semantic search, Claude Code, grep, code navigation

## 요약

대규모 코드베이스에서 원하는 코드를 찾는 것은 개발자의 주요 병목 중 하나다. 기존 grep은 키워드 매칭에 의존하여 42개의 무관한 파일을 반환하거나, 정작 필요한 파일을 놓치기도 한다. mgrep은 시맨틱 검색을 통해 "무엇을 찾고 싶은지"를 자연어로 표현하면 관련성 높은 결과를 반환한다.

## 문제: grep의 한계

### 키워드 기반 검색의 딜레마

grep은 정규표현식 기반 텍스트 매칭 도구다. 강력하지만 근본적인 한계가 있다.

| 상황 | grep의 문제 |
|------|------------|
| 동의어 | "authentication"을 검색하면 "auth", "login", "signin"을 놓침 |
| 맥락 무시 | "connection"을 검색하면 DB connection, socket connection, HTTP connection이 모두 섞임 |
| 과다 결과 | 일반적인 단어로 검색 시 수백 개 파일 반환 |
| 누락 | 변수명이 약어면 검색 불가 |

### 실제 사례: 42개 파일 문제

"payment provider의 spec"을 찾으려 할 때 기존 방식의 결과:

```bash
# 패턴 검색 시도 1
Search(pattern: "**/*payment*")
→ Found 3 files (너무 적음 - .pth 파일과 migration.md만 발견)

# 패턴 검색 시도 2
Search(pattern: "payment.*provider|PaymentProvider")
→ Found 42 files (너무 많음 - 어떤 파일이 핵심인지 모름)
```

42개 파일 목록:
- `spec-format.md`
- `provider.yaml`
- `test_specs.py`
- `provider.md`
- `provider.py`
- `workflows/__init__.py`
- `connection_flow.py`
- `main.py`
- `uv.lock`
- `pyproject.toml`
- ... (32개 더)

개발자는 이 42개 파일을 하나씩 열어보며 "spec 정의"가 어디 있는지 찾아야 한다.

## 해결: mgrep의 시맨틱 검색

[mgrep](https://github.com/mixedbread-ai/mgrep)은 mixedbread-ai에서 개발한 시맨틱 검색 CLI 도구다. "grep for 2025"를 표방하며, 자연어 쿼리로 코드, 텍스트, PDF, 이미지까지 검색할 수 있다.

### 핵심 원리

```
사용자 쿼리 → 임베딩 벡터화 → 코드베이스 인덱스 검색 → 유사도 기반 정렬
```

1. **자연어 쿼리**: "payment provider spec definition and configuration"
2. **임베딩 변환**: 쿼리를 벡터로 변환
3. **유사도 검색**: 코드베이스의 각 청크와 코사인 유사도 계산
4. **결과 정렬**: 관련성 점수(match %)로 정렬하여 반환

### 동일 검색의 mgrep 결과

```bash
mgrep "payment provider spec definition and configuration"
```

결과:
```
.\src\apps\providers\workflow\src\lib\mock-data\providers.ts:33-74 (99.35% match)
.\providers\payment\specs\provider.yaml:1-34 (98.79% match)
.\providers\payment\src\payment_provider\provider.py:1-51 (98.46% match)
.\providers\payment\tests\test_specs.py:1-12 (98.36% match)
.\docs\guides\api-reference.md:91-168 (98.03% match)
.\providers\docs\spec-format.md:67-131 (96.04% match)
.\providers\docs\specs\provider.md:128-202 (95.79% match)
.\packages\provider-sdk\src\provider_sdk\spec.py:385-464 (86.70% match)
```

**42개 → 8개**, 그리고 각 파일이 **왜 관련 있는지** match percentage로 표시된다.

## grep vs mgrep 비교

| 항목 | grep | mgrep |
|------|------|-------|
| 검색 방식 | 정규표현식 매칭 | 시맨틱 유사도 |
| 쿼리 언어 | 정규표현식 | 자연어 |
| 결과 정렬 | 파일명/발견 순 | 관련성 점수 |
| 결과 범위 | 전체 파일 | 관련 코드 청크 (줄 범위) |
| 동의어 처리 | 불가 | 자동 인식 |
| 맥락 이해 | 없음 | 쿼리 의도 파악 |

### 사용 시나리오별 도구 선택

| 시나리오 | 추천 도구 |
|----------|----------|
| 정확한 함수명/변수명 검색 | grep |
| 에러 메시지 문자열 검색 | grep |
| "인증 관련 코드 찾기" | mgrep |
| "이 기능이 어디서 구현됐는지" | mgrep |
| 리팩토링 대상 코드 탐색 | mgrep |

## 설치 및 설정

### 1. 설치

```bash
npm install -g @mixedbread/mgrep
```

### 2. 로그인

```bash
mgrep login
```

또는 환경변수로 API 키 설정:

```bash
export MGREP_API_KEY=your_api_key
```

### 3. 인덱싱

```bash
# 현재 디렉토리 인덱싱 및 파일 변경 감시
mgrep watch
```

`mgrep watch`는 백그라운드에서 실행되며, 파일이 변경될 때마다 자동으로 인덱스를 업데이트한다.

## 사용법

### 기본 사용

```bash
mgrep "자연어로 찾고 싶은 내용 설명"
```

### 주요 옵션

| 옵션 | 설명 |
|------|------|
| `-m <N>` | 결과 개수 제한 (기본: 10) |
| `-c` | 매칭된 내용 표시 |
| `-a` | AI 답변 생성 |
| `--web` | 로컬 + 웹 동시 검색 |

```bash
# 상위 5개 결과만
mgrep -m 5 "authentication middleware"

# 내용까지 표시
mgrep -c "error handling patterns"

# 웹 검색 포함
mgrep --web "Next.js server actions best practices"
```

### 효과적인 쿼리 작성

```bash
# 좋은 쿼리 예시
mgrep "user authentication flow with JWT token validation"
mgrep "database connection pool configuration and timeout settings"
mgrep "error handling middleware for API responses"

# 개선이 필요한 쿼리
mgrep "auth"  # 너무 짧음
mgrep "code"  # 너무 일반적
```

### Claude Code에서의 통합

Claude Code skill로 등록하면 AI 어시스턴트가 자동으로 mgrep을 활용한다.

```yaml
# .claude/skills/mgrep.md
name: mgrep
description: 시맨틱 코드 검색. grep 대신 사용.
```

## 실제 사용 예시

다음은 Claude Code에서 mgrep을 활용한 실제 세션 기록이다.

---

### 상황

사용자가 "payment provider의 spec 설명"을 요청했다.

### 기존 방식 (Search/Grep)

```
❯ payment provider 의 spec 설명 좀 해줘.

● Search(pattern: "**/*payment*")
  ⎿  Found 3 files
     api\.venv\Lib\site-packages\_payment_provider.pth
     providers\.venv\Lib\site-packages\_payment_provider.pth
     providers\docs\payment-migration.md

● Search(pattern: "payment.*provider|PaymentProvider")
  ⎿  Found 42 files
     providers\docs\spec-format.md
     providers\payment\specs\provider.yaml
     providers\payment\tests\test_specs.py
     ...
  ⎿  Interrupted · What should Claude do instead?
```

42개 파일이 나와서 사용자가 중단했다.

### mgrep 사용

```
❯ mgrep 을 사용해

● Skill(mgrep)
  ⎿  Successfully loaded skill

● Bash(mgrep "payment provider spec definition and configuration")
  ⎿  .\src\apps\providers\...\providers.ts:33-74 (99.35% match)
     .\providers\payment\specs\provider.yaml:1-34 (98.79% match)
     .\providers\payment\src\...\provider.py:1-51 (98.46% match)
     .\providers\payment\tests\test_specs.py:1-12 (98.36% match)
     .\docs\guides\api-reference.md:91-168 (98.03% match)
     .\providers\docs\spec-format.md:67-131 (96.04% match)
     .\providers\docs\specs\provider.md:128-202 (95.79% match)
     .\packages\provider-sdk\src\...\spec.py:385-464 (86.70% match)
```

8개 파일로 좁혀졌고, `provider.yaml`(98.79%)이 핵심 파일임을 즉시 알 수 있다.

### 결과

Claude는 상위 2개 파일(`provider.yaml`, `provider.md`)을 읽고 정확한 답변을 제공했다:

```
● Payment Provider Spec 설명

  파일 위치
  providers/payment/specs/provider.yaml

  전체 구조
  apiVersion: platform.io/v1
  kind: Provider
  metadata:
    name: payment
    namespace: payment
    title: Payment Gateway
    ...
```

**43초** 만에 정확한 답변이 완성됐다.

---

## 결론

mgrep은 "무엇을 찾을지는 알지만 어디 있는지 모를 때" 사용하는 도구다. grep이 "정확한 문자열"을 찾는다면, mgrep은 "의미적으로 관련된 코드"를 찾는다.

대규모 코드베이스에서 AI 어시스턴트와 협업할 때, mgrep은 컨텍스트 윈도우를 효율적으로 활용하게 해준다. 42개 파일을 모두 읽는 대신 관련성 높은 8개 파일만 읽으면 되기 때문이다.

## 참고 자료

### 공식 문서
- [mgrep GitHub](https://github.com/mixedbread-ai/mgrep) - 공식 저장소

### 관련 기술
- [Semantic Code Search](https://en.wikipedia.org/wiki/Semantic_search) - 시맨틱 검색 개념
- [Vector Embeddings](https://www.pinecone.io/learn/vector-embeddings/) - 벡터 임베딩 기초

### 관련 블로그
- [Serena MCP: AI 코딩 어시스턴트를 위한 시맨틱 코드 분석 도구](https://blog.imprun.dev/29)
- [Claude Code 컨텍스트 최적화 실전 가이드](https://blog.imprun.dev/103)
