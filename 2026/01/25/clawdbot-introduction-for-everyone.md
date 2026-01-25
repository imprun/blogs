# Clawdbot: 24시간 나를 도와주는 AI 비서 만들기

**작성일**: 2026-01-25  
**카테고리**: AI, Automation, Productivity  
**키워드**: Clawdbot, AI Assistant, Claude, Automation, Discord Bot

[##_Image|kage@c64n7q/dJMcabXiRkV/AAAAAAAAAAAAAAAAAAAAAHMsRCFZnvu2ZkTSz3I3hjSdz-bxPRXZYBck5XFpMPnF/img.png|alignCenter|width="100%"|_##]

## 요약

Clawdbot은 Claude AI를 24시간 작동하는 개인 비서로 만들어주는 오픈소스 도구다. Discord, Telegram 등 원하는 메신저로 대화하고, 뉴스 체크부터 코드 작성까지 자동화할 수 있다. 이 글은 기술 배경 없는 일반인도 따라할 수 있도록 설치부터 실전 활용까지 안내한다.

---

## 왜 Clawdbot인가?

### ChatGPT와 무엇이 다른가?

ChatGPT는 호텔 리셉션 데스크와 같다. 필요할 때 직접 찾아가서 물어봐야 한다. 대화가 끝나면 기억이 리셋된다.

Clawdbot은 집에 있는 개인 비서다. 당신이 누군지 기억하고, 선호하는 것을 알고, 당신이 자는 동안에도 일한다.

| 구분 | ChatGPT (웹) | Clawdbot |
|------|-------------|----------|
| **접근** | 브라우저 열어서 방문 | Discord/Telegram 메시지 |
| **기억** | 대화 세션 종료 시 리셋 | 영구 저장 (파일 기반) |
| **자동화** | 불가능 | 가능 (스케줄링, 알림) |
| **파일 접근** | 업로드만 가능 | 컴퓨터 전체 접근 |
| **비용** | 월 $20 (Plus) | 월 $20 (Claude Pro) |

---

## 5분 만에 시작해보기

설치는 뒤에서 자세히 다룬다. 먼저 Clawdbot을 사용하면 어떤 느낌인지 보자.

### 기본 대화

**Discord DM에서:**
```
너: 오늘 서울 날씨 알려줘

봇: 서울 현재 날씨: 맑음, 기온 5°C
    오늘 최고 8°C, 최저 -2°C
    미세먼지: 보통
```

ChatGPT와 똑같다. 하지만 여기서부터 달라진다.

### 자동화 설정 (한 번만 하면 됨)

```
너: 매일 아침 7시에 오늘 날씨 알려줘

봇:  자동화 설정 완료!
    매일 오전 7시에 서울 날씨를 전송할게요.
```

**다음날 아침 7시:**
```
봇: [자동 전송]
    서울 오늘 날씨: 흐림, 기온 3°C
    오늘 최고 6°C, 최저 0°C
    미세먼지: 나쁨 - 마스크 착용 권장
```

→ 알람 앱 없이, 알람 설정 없이, 자동으로.

### 파일 작업

```
너: workspace에 있는 report.md 파일 요약해줘

봇: [파일 읽고 요약 생성]

너: 요약 내용을 summary.txt로 저장해줘

봇:  summary.txt 파일 저장 완료
```

→ 파일 업로드 없이, 컴퓨터에 있는 파일 바로 작업.

### 지속적인 기억

```
너: 내일 오후 3시 회의 있어. 30분 전에 알려줘.

봇:  알겠습니다. 내일 오후 2시 30분에 알림 보낼게요.

[다음날 14:30]
봇: 🔔 30분 후 회의가 있습니다.
    준비 사항 확인하세요.
```

→ 세션 종료해도 기억. 컴퓨터 재시작해도 기억.

---

## 주요 활용 사례

### 📰 정보 자동 수집
- 관심 뉴스 매일 아침 요약
- 주가, 환율, 날씨 정기 체크
- RSS 피드 모니터링

###  콘텐츠 작성
- 블로그 초안 작성
- 문서 요약/번역
- Git commit 자동화

### 📋 프로젝트 관리
- GitHub 이슈/PR 알림
- 작업 진행률 리포트
- 일정 리마인더

### 🤖 반복 작업 자동화
- 정기 백업
- 로그 분석
- 데이터 변환

**이런 사람에게 추천:**
- 매일 같은 정보 체크하는 사람 (뉴스, 주가, 날씨)
- 블로그/문서 작성하는 사람
- GitHub 프로젝트 관리하는 개발자
- 반복 업무가 많은 1인 기업가

---

## 설치 방법

### 공식 가이드 따라하기

Clawdbot은 Windows, macOS, Linux 모두 지원한다.

**📖 공식 설치 가이드:**
- 시작하기: https://docs.clawd.bot/getting-started
- 홈페이지: https://clawd.bot/

**플랫폼별 설치:**
- **Windows**: 인스톨러 또는 npm
- **macOS**: Homebrew 또는 npm
- **Linux**: 패키지 매니저 또는 npm

**가장 빠른 설치 (npm):**
```bash
npm install -g clawdbot
clawdbot onboard
```

### 인증 방법: Claude Pro 구독 활용

Clawdbot은 **Claude Pro 구독**($20/월)으로 사용할 수 있다.

**설정 방법:**
1. https://claude.ai 에서 Claude Pro 구독
2. `clawdbot onboard` 실행
3. Claude 계정으로 로그인 (OAuth)
4. 완료!

**장점:**
- API 키 발급 불필요
- 월 $20 고정 비용
- Claude.ai와 동일한 계정 사용

### 설치 확인

```bash
clawdbot status
```

`Gateway: reachable` 표시되면 성공!

---

## 첫 번째 사용: Discord 연동

### 왜 Discord인가?

- 모바일 앱으로 어디서나 접근
- 파일 주고받기 편함
- 알림 기능 강력
- 무료

### Discord 봇 만들기

#### 1단계: Discord 개발자 포털

1. https://discord.com/developers/applications 접속
2. **New Application** 클릭
3. 봇 이름 입력 (예: "내 비서")

#### 2단계: 봇 생성 및 토큰 복사

1. 왼쪽 **Bot** 메뉴
2. **Add Bot** 클릭
3. **Bot Token** 복사 (나중에 사용)

**중요 설정:**
- **Privileged Gateway Intents**
  -  Message Content Intent
  -  Server Members Intent

#### 3단계: 봇 초대 URL 생성

1. **OAuth2** → **URL Generator**
2. **Scopes**: `bot`, `applications.commands`
3. **Bot Permissions**:
   -  View Channels
   -  Send Messages
   -  Read Message History
   -  Attach Files

4. 생성된 URL 복사 → 브라우저에서 열기
5. 서버 선택 → **승인**

#### 4단계: Clawdbot 설정

설정 파일을 직접 편집해야 한다.

**설정 파일 위치:**
- Windows: `C:\Users\{사용자}\.clawdbot\clawdbot.json`
- macOS/Linux: `~/.clawdbot/clawdbot.json`

**편집:**
```bash
# VSCode로 열기
code ~/.clawdbot/clawdbot.json

# 또는 메모장 (Windows)
notepad %USERPROFILE%\.clawdbot\clawdbot.json
```

**Discord 설정 추가:**
```json
{
  "channels": {
    "discord": {
      "enabled": true,
      "token": "여기에_봇_토큰_붙여넣기",
      "groupPolicy": "allowlist",
      "dm": {
        "policy": "allowlist",
        "allowFrom": ["당신의_Discord_사용자_ID"]
      },
      "guilds": {
        "서버_ID_1": {
          "requireMention": false
        }
      }
    }
  }
}
```

**설정 항목 설명:**

| 항목 | 설명 | 값 |
|------|------|-----|
| `token` | Discord 봇 토큰 | 개발자 포털에서 복사 |
| `groupPolicy` | 서버 메시지 처리 방식 | `"open"` / `"disabled"` / `"allowlist"` |
| `dm.policy` | DM 허용 정책 | `"pairing"` (기본) / `"allowlist"` / `"open"` / `"disabled"` |
| `dm.allowFrom` | DM 허용 사용자 목록 | Discord 사용자 ID 배열 |
| `guilds` | 서버별 세부 설정 | 서버 ID → 설정 맵 |
| `requireMention` | 멘션 필수 여부 | `true` / `false` |

** 보안 권장 설정:**
- `dm.policy: "pairing"` (기본값) - 첫 DM 시 pairing code 승인 필요
- `dm.policy: "allowlist"` - 특정 사용자만 DM 허용
- `groupPolicy: "allowlist"` - 명시한 서버만 허용

**Discord 사용자 ID 확인:**
1. Discord 설정 → 고급 → **개발자 모드** 켜기
2. 자신의 프로필 우클릭 → **사용자 ID 복사**

**서버 ID 확인:**
1. 서버 아이콘 우클릭 → **서버 ID 복사**

**저장 후 재시작:**
```bash
clawdbot gateway restart
```

**설정 확인:**
```bash
clawdbot status
```

출력 예시:
```
Channels
┌──────────┬─────────┬────────┬──────────────────────┐
│ Channel  │ Enabled │ State  │ Detail               │
├──────────┼─────────┼────────┼──────────────────────┤
│ Discord  │ ON      │ OK     │ token config · 1/1   │
└──────────┴─────────┴────────┴──────────────────────┘
```

`State: OK`면 성공!

**트러블슈팅:**

**응답이 없는 경우:**
1. `clawdbot status`로 상태 확인
2. `clawdbot logs --follow`로 로그 확인
3. Discord Developer Portal에서 **Privileged Gateway Intents** 활성화 확인
4. `dm.allowFrom`에 자신의 Discord ID 추가 확인

**상세한 트러블슈팅:**
- [Discord 설정 가이드](https://docs.clawd.bot/channels/discord)
- [진단 도구](https://docs.clawd.bot/tools/diagnostics)

---

## Clawdbot 사용하는 3가지 방법

설치가 완료되면 3가지 방법으로 Clawdbot과 대화할 수 있다.

### 방법 1: TUI 채팅 (터미널)

**가장 빠른 방법:**

```bash
clawdbot
```

→ 터미널에서 바로 대화 시작

**장점:**
- 설치 직후 바로 사용 가능
- 추가 설정 불필요
- 빠른 응답

**단점:**
- 터미널 열어야 함
- 모바일 사용 불가

### 방법 2: CLI 명령어

**일회성 작업에 적합:**

```bash
clawdbot ask "오늘 날씨 알려줘"
```

또는 파이프로 파일 전달:

```bash
cat file.md | clawdbot ask "이 파일 요약해줘"
```

**장점:**
- 스크립트에 포함 가능
- 자동화 쉬움

**단점:**
- 대화 맥락 유지 어려움

### 방법 3: Discord DM (추천!)

Discord 봇 연동 후:

1. Discord에서 봇 찾기 (멤버 목록)
2. 우클릭 → **메시지**
3. "안녕?" 입력

→ 봇이 응답하면 성공! 🎉

**장점:**
- 📱 모바일에서 사용 가능
- 🔔 알림 받기
- 📎 파일 주고받기 편함
- 💬 자연스러운 대화

**단점:**
- 초기 Discord 봇 설정 필요 (한 번만)

---

## 실전 활용 예시

### 예시 1: 매일 아침 뉴스 받기

**목표:** 매일 오전 8시, 관심 분야 뉴스 요약 받기

**Discord DM에서:**
```
매일 오전 8시에 AI 뉴스 top 5개 요약해서 보내줘
```

**Clawdbot:**
```
 자동화 설정 완료!
매일 오전 8시에 AI 뉴스 top 5개 요약해서 전송할게요.
```

**결과:**
- 매일 아침 8시, 자동으로 Discord DM 도착
- 잠에서 깨면 이미 정리된 뉴스 대기 중
- 출근길에 읽기 딱 좋음

### 예시 2: GitHub 이슈 알림

**목표:** 내 프로젝트에 새 이슈 생기면 즉시 알림

**Discord DM에서:**
```
매 30분마다 내 GitHub 프로젝트 새 이슈 체크해줘.
새 이슈 있으면 알려줘.
```

Clawdbot이 자동으로 cron job 생성:
- 30분마다 GitHub 체크
- 새 이슈 발견 시 Discord 알림

### 예시 3: 블로그 초안 → 발행

**Discord DM 대화:**

```
너: 여행 블로그 초안 써줘. 제목: "혼자 떠난 제주 3박4일"

봇: [초안 생성]

너: 2일차에 성산일출봉 추가해줘

봇: [수정본 생성]

너: 좋아. Git commit하고 파일 보내줘

봇:  Commit 완료
    📎 [파일 첨부]
```

파일 다운로드 → Tistory 복붙 → 발행

### 예시 4: 주간 리포트 자동 생성

**목표:** 매주 금요일 17시, 한 주 작업 요약

**Discord DM에서:**
```
매주 금요일 오후 5시에 이번 주 Git commit 분석해서 
주간 리포트 작성해줘. 주요 성과, 개선점, 다음 주 계획 포함해서.
```

**결과:**
- 매주 금요일 17시, 자동으로 리포트 생성
- 주말 전에 한 주 돌아보기 완료
- 월요일 회의 자료로 활용

---

## 개인화: 나만의 비서 만들기

### SOUL.md: AI의 성격 정의

**파일 위치:** `C:\Users\{사용자}\clawd\SOUL.md`

**예시:**
```markdown
# SOUL.md

## 핵심 원칙
- 간결하게 답변 (불필요한 인사 생략)
- 전문 용어 사용 시 설명 추가
- 실행 가능한 제안 우선

## 말투
- 존댓말 사용
- 이모지 최소화
- 직설적이지만 친절하게
```

### USER.md: 나에 대한 정보

```markdown
# USER.md

- **이름:** 준식
- **타임존:** Asia/Seoul
- **관심사:**
  - 기술 뉴스 (AI, 개발)
  - 축구 (전북현대)
  - 블로그 운영

## 프로젝트
- impakt: Wehago 세무 데이터 수집
- imprun.dev: 기술 블로그
```

### MEMORY.md: 중요한 기억

```markdown
# MEMORY.md

## 배운 교훈
- 2026-01-25: PowerShell -replace 사용 금지 (인코딩 문제)

## 선호 사항
- 블로그: "하니스" → "하네스" (Agent Harness)
- Git commit: Conventional Commits 형식
```

---

## 비용은 얼마나 나올까?

### Claude Pro 구독으로 사용

**가장 간단한 방법:**

- **Claude Pro**: 월 $20 (고정)
- Claude Code CLI 사용 권한 포함
- Clawdbot에서 바로 사용 가능

**비교:**

| 서비스 | 월 비용 | 특징 |
|--------|---------|------|
| ChatGPT Plus | $20 | 웹 전용 |
| Claude Pro | $20 | **Clawdbot 사용 가능**  |
| Claude API (종량제) | $5~100+ | 사용량 따라 변동 |

**추천:**
- 일반 사용자: **Claude Pro** (예측 가능한 비용)
- 개발자/헤비 유저: 사용량 확인 후 API 종량제 고려

**Claude Pro 구독 방법:**
1. https://claude.ai 접속
2. 우측 상단 **Upgrade** 클릭
3. 월 $20 구독

---

## Windows 사용자 참고사항

### PowerShell 기본 사용

Windows에서 Clawdbot의 `exec` 도구는 **PowerShell을 기본으로 사용**합니다.

**bash 스타일 명령어 주의:**
```powershell
#  오류 발생
mkdir dir 2>nul & echo Done
command1 && command2

#  PowerShell 네이티브
New-Item -ItemType Directory -Force -Path "dir"
command1; if ($?) { command2 }
```

**복잡한 bash 명령어가 필요한 경우:**

Git Bash가 설치되어 있다면 직접 호출:
```powershell
C:\git\bin\bash.exe -c "command1 && command2"
```

**권장:**
- 간단한 작업: PowerShell 네이티브 명령어 사용
- 복잡한 bash 스크립트: `bash -c` 사용
- 자세한 사항은 [공식 문서](https://docs.clawd.bot/tools/exec) 참고

### Windows 한계와 macOS 권장

**Windows의 제약:**
- PowerShell 고정 (shell 변경 불가)
- bash 명령어 체이닝 제한 (`&&`, `||`)
- 일부 자동화 스크립트 호환성 이슈

**풀 기능 활용:**

Clawdbot의 모든 기능을 활용하려면 **macOS 또는 Linux** 권장:
- bash/zsh shell 네이티브 지원
- 모든 Unix 도구 기본 제공
- 스크립트 호환성 100%

**Windows에서도 충분한 경우:**
- Discord/메신저 연동
- 뉴스 체크, 일정 알림
- 간단한 파일 작업
- 블로그 작성 보조

**개발자/파워 유저:**
- macOS 또는 Linux 추천
- 또는 Windows에서 WSL2 사용

---

## 자주 묻는 질문

### Q1. 기술 배경 없어도 사용 가능한가?

**A:** 가능하다. 

- 설치: 가이드 따라 클릭 몇 번
- 사용: Discord 메시지 보내듯이
- 고급 기능: 필요할 때 천천히 배우면 됨

### Q2. 보안은 안전한가?

**A:** 설정만 제대로 하면 매우 안전하다.

**안전한 부분:**
- 모든 데이터: 내 컴퓨터에 저장
- API 통신: 암호화
- 외부 유출: 없음 (설정하지 않는 한)

** 중요한 보안 설정:**

**1. DM 정책 - 기본값은 안전함**

Clawdbot 기본 설정:
```json
"dm": {
  "policy": "pairing"  // 기본값: 첫 DM 시 승인 필요
}
```

**DM 정책 종류:**

| 정책 | 설명 | 보안 |
|------|------|------|
| `"pairing"` | 첫 DM 시 pairing code 승인 필요 (기본) |  안전 |
| `"allowlist"` | 특정 사용자만 DM 허용 |  안전 |
| `"disabled"` | 모든 DM 거부 |  안전 |
| `"open"` + `allowFrom: ["*"]` | 누구나 DM 가능 |  위험 |

**왜 `"open"` + `["*"]`가 위험한가?**
- 모르는 사람이 DM을 보내면 당신의 `USER.md`, `MEMORY.md`를 읽을 수 있음
- 개인 정보, 프로젝트 정보 노출 위험
- workspace 파일 접근 가능

**권장: 기본값(`"pairing"`) 유지 또는 `"allowlist"` 사용**

**2. 서버 설정 - 멘션 필수로**
```json
"guilds": {
  "공개_서버_ID": {
    "requireMention": true  // @봇 멘션해야만 응답
  }
}
```

**3. groupPolicy 이해하기**
```json
"groupPolicy": "allowlist"  // guilds에 명시된 서버만 허용
```

| groupPolicy | 의미 | 권장 |
|-------------|------|------|
| `"disabled"` | 모든 서버 메시지 무시 |  개인용 |
| `"allowlist"` | `guilds`에 명시된 서버만 |  팀/공유용 |
| `"open"` | 초대된 모든 서버 허용 |  주의 필요 |

**주의사항:**
- Discord 봇 토큰 절대 공유 금지
- `.clawdbot/clawdbot.json` 파일 보안 (chmod 600)
- 공개 서버에서는 개인 정보 요청 금지

### Q3. 컴퓨터 꺼지면 작동 안 하나?

**A:** 맞다.

**해결책:**
- 24시간 켜두기
- 또는 클라우드 서버 사용 (AWS, GCP)
- Raspberry Pi 같은 저전력 기기 사용

### Q4. 다른 메신저도 가능한가?

**A:** 가능하다.

지원 메신저:
- Discord 
- Telegram 
- Slack 
- Signal 
- WhatsApp 
- iMessage  (Mac만)

### Q5. 여러 명이 같이 쓸 수 있나?

**A:** 가능하다. 상황에 따라 3가지 방법이 있다.

**방법 1: 공유 서버/채널 (가장 간단)**
```json
"guilds": {
  "가족_서버_ID": {
    "requireMention": true  // @봇 멘션 필수
  }
}
```
- 가족/팀 Discord 서버에 봇 초대
- 멘션하면 응답
-  **workspace 공유됨** - 개인 정보 저장 금지

**방법 2: 완전 분리 (Multi-Agent) - 추천!**

각자 독립된 AI 비서를 원한다면:

```json
{
  "agents": {
    "list": [
      {
        "id": "alex",
        "workspace": "~/clawd-alex"
      },
      {
        "id": "mia",
        "workspace": "~/clawd-mia"
      }
    ]
  },
  "bindings": [
    {
      "agentId": "alex",
      "match": {
        "channel": "discord",
        "peer": { "kind": "dm", "id": "alex_사용자_ID" }
      }
    },
    {
      "agentId": "mia",
      "match": {
        "channel": "discord",
        "peer": { "kind": "dm", "id": "mia_사용자_ID" }
      }
    }
  ]
}
```

**장점:**
- 완전히 분리된 workspace
- 각자의 `MEMORY.md`, `USER.md`
- 독립된 개인 비서

**방법 3: 같은 봇, 여러 사람 DM**
```json
"dm": {
  "policy": "allowlist",
  "allowFrom": ["ID1", "ID2", "ID3"]
}
```
- 간단하지만 workspace 공유됨
- 개인 정보 노출 위험

**📖 상세 가이드:**
- Discord 설정: https://docs.clawd.bot/channels/discord
- Multi-Agent: https://docs.clawd.bot/concepts/multi-agent

---

## 커뮤니티 및 리소스

**공식 자료:**
- 📚 공식 문서: https://docs.clawd.bot
- 🔧 Discord 설정 가이드: https://docs.clawd.bot/channels/discord
- 👥 Multi-Agent 가이드: https://docs.clawd.bot/concepts/multi-agent
- 💻 GitHub: https://github.com/clawdbot/clawdbot
- 💬 Discord 커뮤니티: https://discord.gg/clawd

**학습 순서 추천:**
1. [Getting Started](https://docs.clawd.bot/getting-started) - 설치 및 첫 실행
2. [Discord Channel](https://docs.clawd.bot/channels/discord) - Discord 연동 상세
3. [Slash Commands](https://docs.clawd.bot/tools/slash-commands) - 네이티브 명령어
4. [Multi-Agent](https://docs.clawd.bot/concepts/multi-agent) - 여러 사람/봇 운영

---

## 마무리

Clawdbot은 단순한 챗봇이 아니다. 당신의 업무 방식, 생활 패턴, 선호도를 학습하고 기억하는 개인 비서다.

처음엔 간단한 뉴스 체크부터 시작해서, 점차 블로그 작성, 프로젝트 관리, 업무 자동화까지 확장할 수 있다.

**핵심은 "시작의 장벽을 낮추는 것"이다.**

- Discord 메시지 하나면 시작
- 복잡한 명령어 불필요
- 필요할 때 조금씩 배우면 됨

**이 글에서 다루지 못한 것:**
- 고급 자동화 (GitHub Actions 연동 등)
- 커스텀 스킬 작성
- Docker/클라우드 배포
- 음성/영상 처리

더 자세한 내용은 [공식 문서](https://docs.clawd.bot)를 참고하자.

당신만의 AI 비서, 오늘 시작해보자.

---

## 참고 자료

- Clawdbot 공식 문서: https://docs.clawd.bot
- Anthropic Claude API: https://console.anthropic.com
- Discord 개발자 포털: https://discord.com/developers
- Node.js 다운로드: https://nodejs.org
