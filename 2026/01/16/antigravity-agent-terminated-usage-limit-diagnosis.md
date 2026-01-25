# Antigravity IDE "Agent terminated due to error" 해결하기: Gemini /stats로 사용 한도 진단

> **작성일**: 2026년 01월 16일
> **카테고리**: IDE, Troubleshooting, AI Tools
> **키워드**: Antigravity, Gemini, Agent Terminated, Usage Limit, AI IDE

## 요약

Antigravity IDE에서 반복적으로 발생하는 "Agent terminated due to error" 에러의 원인을 파악했습니다. Windows와 WSL 환경 모두에서 동일한 문제가 발생했으며, Gemini CLI의 `/stats` 명령어를 통해 사용 한도 도달이 근본 원인임을 확인했습니다. 다른 AI IDE들과 달리 Antigravity는 사용 한도 도달 시 명확한 안내 없이 일반적인 에러 메시지만 반복 표시하는 UX 문제가 있습니다.

## 문제 상황

### 증상

Antigravity IDE 사용 중 다음과 같은 에러가 지속적으로 발생:

```
Agent terminated due to error
You can prompt the model to try again or start a new conversation if the error persists.
```

- "Retry" 버튼 클릭 시 동일한 에러 반복
- 새로운 대화 시작 시에도 같은 증상
- 에러 로그나 상세 정보 없음

### 환경 구성

- **운영체제**: Windows 11 / WSL2 Ubuntu
- **IDE**: Antigravity IDE (Google)
- **AI 모델**: Gemini 2.5 Flash Lite
- **발생 시점**: Windows 환경에서 먼저 발생, WSL 환경으로 전환해도 동일

### 초기 진단

**1. 환경 문제 의심**

Windows 환경 특유의 문제로 판단하고 WSL2 Ubuntu로 전환했으나, 동일한 에러 발생.

**2. IDE 버그 의심**

Antigravity IDE의 버그로 추정했으나, 에러 메시지가 구체적이지 않아 원인 파악 어려움.

## 근본 원인 분석

### Gemini CLI /stats 명령어를 통한 진단

Antigravity 대신 Gemini CLI를 실행하고 `/stats` 명령어로 사용 현황을 확인:

```bash
$ gemini /stats

╭─────────────────────────────────────────────────────────────────────────────────────────╮
│  Session Stats                                                                          │
│                                                                                         │
│  Model Usage                 Reqs                  Usage left                          │
│  ────────────────────────────────────────────────────────────                          │
│  gemini-2.5-flash-lite          2  100.0% (Resets in 22h 23m)                          │
│  gemini-2.5-flash               -  100.0% (Resets in 22h 23m)                          │
│  gemini-2.5-pro                 -  100.0% (Resets in 22h 23m)                          │
│  gemini-3-flash-preview         -  100.0% (Resets in 22h 30m)                          │
│  gemini-3-pro-preview           -  100.0% (Resets in 22h 23m)                          │
│                                                                                         │
│  Usage limits span all sessions and reset daily.                                       │
│  /auth to upgrade or switch to API key.                                                │
╰─────────────────────────────────────────────────────────────────────────────────────────╯
```

**주요 발견:**
- `Usage left: 100.0% (Resets in 22h 23m)` - 사용 한도에 도달하여 약 22시간 후 리셋 예정
- `Usage limits span all sessions` - 모든 세션에서 한도를 공유

### Antigravity IDE의 에러 처리 문제

**다른 AI IDE들의 사용 한도 처리:**

| IDE | 사용 한도 도달 시 동작 |
|-----|---------------------|
| Claude Code | 명확한 한도 도달 메시지 표시, 업그레이드 안내 |
| Cursor | 사용량 표시, 구독 플랜 안내 |
| GitHub Copilot | 남은 요청 수 표시 |
| **Antigravity** | **일반적인 에러 메시지만 반복** |

**문제점:**
1. 사용자가 에러의 실제 원인을 알 수 없음
2. 환경 문제로 오인하여 불필요한 디버깅 시도
3. 리셋 시간 정보 없음

### 근본 원인

**사용 한도 도달 + 불명확한 에러 메시지**

Gemini API의 일일 사용 한도에 도달했지만, Antigravity IDE는 이를 사용자에게 명확히 전달하지 않고 일반적인 "Agent terminated" 에러만 표시했습니다.

## 해결 과정

### 1. 현재 사용량 확인

```bash
# Gemini CLI 설치 (npm)
$ npm install -g @google/generative-ai-cli

# 또는 직접 실행 (npx)
$ npx @google/generative-ai-cli

# stats 명령어 실행
> /stats
```

### 2. 리셋 시간 확인

`/stats` 출력에서 `Resets in` 정보 확인:
- `Resets in 22h 23m` - 약 22시간 후 한도 리셋

### 3. 대안

**즉시 사용이 필요한 경우:**

```bash
# Gemini CLI에서 인증 명령어 실행
> /auth

# 옵션:
# 1. API Key 직접 사용 (유료 플랜)
# 2. 다른 계정으로 전환
```

**기다리는 경우:**
- 일일 한도는 24시간 주기로 자동 리셋
- 추가 비용 없이 다음 날 사용 가능

## 교훈

### 1. 에러 메시지는 구체적이어야 한다

**나쁜 예시 (Antigravity):**
```
Agent terminated due to error
```

**좋은 예시:**
```
Usage limit reached. Your daily quota will reset in 22h 23m.
To continue now, upgrade your plan or use an API key.
```

**사용자 경험 개선 포인트:**
- 에러의 실제 원인 명시
- 해결 방법 제시 (대기 시간 또는 업그레이드 옵션)
- 불필요한 디버깅 시도 방지

### 2. 사용량 정보는 투명하게 공개되어야 한다

**권장 구현:**
- IDE 내에서 현재 사용량 표시
- 한도 도달 전 경고 메시지
- 리셋 시간 정보 제공

### 3. 크로스 플랫폼 문제와 API 한도는 별개

**디버깅 순서:**
1. 에러 메시지의 정확한 의미 파악
2. API 사용량/한도 확인
3. 환경 문제 검증

환경 문제로 오인하여 Windows → WSL로 전환하는 등 불필요한 시도를 했으나, 실제 원인은 API 사용 한도였습니다.

### 4. CLI 도구의 가치

Gemini CLI의 `/stats` 명령어가 문제 해결의 핵심이었습니다. IDE가 제공하지 않는 정보를 CLI 도구에서 확인할 수 있었습니다.

**개발 환경 구성 권장 사항:**
- 주 IDE 외에 CLI 도구도 함께 설치
- 디버깅 시 다각도 접근 가능

## 참고 자료

### 공식 문서
- [Gemini API Documentation](https://ai.google.dev/docs)
- [Gemini CLI GitHub](https://github.com/google/generative-ai-js/tree/main/packages/cli)

### 관련 도구
- **Antigravity IDE**: Google의 AI 기반 IDE
- **Gemini CLI**: Gemini API 사용량 모니터링 및 관리 도구
