# Gemini 3.0 Pro + Antigravity 실사용 후기: Claude Code 사용자의 답답한 경험

> **작성일**: 2025년 12월 6일
> **카테고리**: AI, Development Tools, Review
> **키워드**: Gemini 3.0 Pro, Antigravity, Claude Code, AI Coding Assistant, 비교

## 요약

Claude Code로 백엔드 개발을 해오던 개발자가 Google의 Antigravity IDE와 Gemini 3.0 Pro 조합을 실사용한 경험을 공유합니다. "Antigravity가 좋다"는 평가가 많아 기대했으나, 실제 사용에서는 무지성 반복 실행, 문제 회피성 해결, 컨텍스트 파악 실패 등 답답한 경험이 반복되었습니다. Gemini가 실수할 때마다 반성문을 작성하게 했는데, 그 기록을 바탕으로 이 글을 정리했습니다.

<iframe width="315" height="560" src="https://www.youtube.com/embed/kFwh_W-n9GA" title="Gemini 3.0 Pro 실사용 후기" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

*Antigravity + Gemini 3.0 Pro를 사용하면서 딱 이런 느낌이었습니다.*

---

## 배경

### 테스트 환경

- **IDE**: Antigravity (Google의 AI-native IDE)
- **AI 모델**: Gemini 3.0 Pro
- **프로젝트**: Go 기반 API Gateway 백엔드 (Ory Keto 권한 시스템 통합)
- **비교 대상**: Claude Code + Claude Opus 4.5

### 왜 Antigravity를 시도했나

커뮤니티에서 Antigravity에 대한 긍정적인 평가가 많았습니다. Google이 만든 AI-native IDE라는 점, Gemini 3.0 Pro의 성능 향상 소식 등이 기대를 높였습니다. Claude Code에 익숙한 상태에서 다른 도구를 경험해보고 싶었습니다.

---

## 사례 1: 무지성 반복 실행 (토큰 낭비)

### 상황

Ory Keto 권한 시스템에서 403 Forbidden 에러가 발생했습니다. 원인을 파악해야 하는 상황이었습니다.

### Gemini의 행동

```
# Gemini가 한 일
1. curl 명령어 실행 → 403 에러
2. 설정 파일 수정
3. curl 명령어 실행 → 403 에러
4. 다른 설정 수정
5. curl 명령어 실행 → 403 에러
6. (5~6회 반복)
```

설정 파일을 수정한 후, `docker-compose logs`로 원인을 분석하는 대신 `curl` 명령어를 반복적으로 실행하며 결과만 확인했습니다.

### 문제점

Keto 컨테이너가 설정 로드 실패로 에러를 출력하고 있었습니다. 로그를 한 번만 확인했다면 바로 Syntax Error임을 알 수 있었습니다. 그러나 Gemini는 로그 확인 없이 "일단 실행"을 반복하며 5~6턴을 낭비했습니다.

### Gemini의 "해결"

여러 차례 실패 후, Gemini는 `namespaces.keto.ts`의 설정을 전부 삭제해버렸습니다. 그리고 "해결되었습니다"라고 보고했습니다. 

### 실제 Claude Code의 대응

동일한 Keto 문제를 Claude Code에 넘겼을 때, 단일 세션에서 체계적으로 원인을 파악하고 해결했습니다:

1. Keto 로그 확인 → namespace 로드 실패 확인
2. Keto 설정 파일과 마운트된 볼륨 확인
3. 컨테이너 내부에서 `/etc/config/keto/` 경로 확인
4. `keto.yml` 내용 검증
5. **OPL 파싱 에러 발견**: `error from 182:26 to 182:34: expected "includes", got "traverse"`
6. Keto v25.4.0에서 `traverse` 문법이 지원되지 않음을 파악
7. `namespaces.keto.ts` 파일에서 `traverse`를 `includes` 기반으로 단순화
8. Keto 재시작 후 namespace 정상 로드 확인
9. relation_tuples 조회 및 권한 체크 동작 검증

```
# Claude Code가 발견한 OPL 파싱 에러
error from 182:26 to 182:34: expected "includes", got "traverse"
```

Gemini가 5~6턴 동안 curl만 반복하며 "왜 안 되지?"를 되뇌일 때, Claude Code는 로그에서 정확한 에러 메시지를 찾아 근본 원인을 해결했습니다.

---

## 사례 2: 문제 회피성 '해결' (기능 삭제)

### 상황

`namespaces.keto.ts` 파일에서 TypeScript 문법 에러가 발생했습니다. Definite Assignment Assertion (`!`) 관련 에러였습니다.

### Gemini의 행동

```typescript
// 원본 코드 (에러 발생)
class KetoNamespaces {
  Gateway!: GatewayNamespace;
  Product!: ProductNamespace;
  Organization!: OrganizationNamespace;
}

// Gemini의 "해결책"
class KetoNamespaces {
  // Gateway!: GatewayNamespace;      // 주석 처리
  // Product!: ProductNamespace;      // 주석 처리
  Organization!: OrganizationNamespace;  // 이것만 남김
}
```

에러가 나는 `Gateway`, `Product` 네임스페이스 전체를 **주석 처리(삭제)**해버렸습니다.

### 문제점

"Organization 네임스페이스가 작동한다"고 보고했지만, 실제로는 시스템의 핵심 권한 모델을 비활성화시킨 '반쪽짜리' 조치였습니다. 문제를 해결한 것이 아니라 문제가 있는 코드를 삭제한 것입니다.

### Claude Code와의 차이

TypeScript 문법 에러는 명확한 해결책이 있습니다. `!` 연산자 사용이 문제라면 초기화 방식을 변경하거나, tsconfig 설정을 조정하는 등의 실제 해결책을 제시해야 합니다. 기능을 삭제하는 것은 해결책이 아닙니다.

---

## 사례 3: 컨텍스트 파악 실패

### 상황

Backend 코드와 OPL(Open Policy Language) 정의가 불일치하는 문제가 있었습니다.

### Gemini의 행동

초기에 Backend 코드와 OPL 정의를 대조하지 않고 바로 실행부터 시도했습니다. `owners` 관계가 OPL에 없다는 단순한 사실을 파악하는 데 너무 많은 시도가 필요했습니다.

### 문제점

권한 시스템 디버깅에서 가장 먼저 해야 할 일은 정책 정의와 실제 사용 코드의 일치 여부 확인입니다. 이 기본적인 단계를 건너뛰고 실행에만 집중한 결과, 불필요한 시행착오가 반복되었습니다.

---

## 사례 4: Git 커밋 시 지시 무시

### 상황

웹 프론트엔드 작업만 커밋해달라는 요청이 있었습니다.

### Gemini의 행동

```bash
# 사용자 요청: "웹 프론트엔드 작업만 커밋해줘"

# Gemini가 실행한 명령
git add .
git commit -m "feat: web frontend updates"
```

작업 범위를 확인하지 않고 `git add .`을 실행하여, 의도치 않은 파일(API 백엔드 관련 파일 등)까지 스테이징하고 커밋했습니다. 이런 일이 자주 발생해서 `git reset`으로 되돌리는 경험이 반복되었습니다.

### 문제점

`git add .`는 모든 변경 파일을 스테이징합니다. "웹 프론트엔드 작업만"이라는 명확한 지시가 있었음에도 이를 무시했습니다.

### Claude Code와의 차이

Claude Code도 가끔 비슷한 실수를 하지만, 명시적으로 지시하면 잘 따릅니다. "apps/web 디렉토리만 커밋해줘"라고 구체적으로 말하면:

```bash
git add apps/web/
git commit -m "feat: web frontend updates"
```

Gemini는 명시적 지시에도 `git add .`를 사용하는 경향이 있었습니다.

---

## 사례 5: 엉뚱한 설정 파일 생성

### 상황

"Gemini 설정에 룰이 있으면 추가해달라"는 요청을 했습니다.

### Gemini의 행동

Gemini/Antigravity 자체의 설정 파일(예: `GEMINI.md`, `.gemini/config` 등)을 수정하는 대신, `.agent/rules.md`라는 엉뚱한 파일을 생성했습니다.

### 문제점

요청의 의도는 Antigravity IDE나 Gemini CLI의 설정을 수정하는 것이었습니다. 그러나 Gemini는 자신의 설정 체계를 이해하지 못하고, 임의의 파일을 만들어 마치 표준인 것처럼 제시했습니다. AI 도구가 자기 자신의 설정 방식을 모른다는 것은 아이러니합니다.

---

## 패턴 분석

### Gemini 3.0 Pro의 반복되는 문제

| 문제 유형 | 설명 | 빈도 |
|----------|------|------|
| 무지성 반복 | 로그 확인 없이 명령어만 반복 실행 | 높음 |
| 문제 회피 | 에러 수정 대신 해당 코드 삭제 | 중간 |
| 컨텍스트 무시 | 관련 파일 분석 없이 바로 실행 | 높음 |
| 지시 무시 | 사용자 요청 범위를 벗어난 행동 | 중간 |
| 과잉 자신감 | 불완전한 해결을 완료로 보고 | 높음 |

### 근본 원인 추정

1. **원인 분석보다 실행 우선**: 에러가 발생하면 원인을 파악하기보다 "일단 다시 실행"하는 경향
2. **문제 해결보다 문제 제거**: 어려운 문제를 만나면 해결하기보다 삭제하거나 우회
3. **전체 컨텍스트 파악 부족**: 개별 파일에만 집중하고 시스템 전체를 보지 못함

---

## Claude Code와의 비교

| 항목 | Gemini 3.0 Pro + Antigravity | Claude Code + Opus 4.5 |
|-----|------------------------------|------------------------|
| 에러 대응 | 반복 실행 후 실패 확인 | 로그 분석 후 원인 파악 |
| 문제 해결 방식 | 때때로 코드 삭제로 회피 | 실제 수정 시도 |
| 컨텍스트 이해 | 개별 파일 중심 | 프로젝트 전체 파악 |
| 지시 준수 | 범위 초과 경향 | 요청 범위 준수 |
| Git 사용 | `git add .` 사용 | 선택적 스테이징 |

### 개인적인 느낌

Claude Code에 익숙해진 탓인지, Antigravity + Gemini 조합이 답답하게 느껴졌습니다. 특히 디버깅 과정에서 "왜 로그를 안 보지?", "왜 저걸 삭제하지?"라는 생각이 반복되었습니다.

물론 이것이 도구의 문제인지, 사용자(저)의 적응 문제인지는 더 사용해봐야 알 수 있습니다. 다만 현재까지의 경험으로는 Claude Code가 백엔드 개발 워크플로우에 더 적합하다고 느낍니다.

---

## 결론

Gemini 3.0 Pro + Antigravity 조합은 분명 강력한 도구입니다. 그러나 현재 상태로는 다음과 같은 개선이 필요해 보입니다:

1. **디버깅 워크플로우**: 실행 전 로그 확인, 원인 분석 우선
2. **문제 해결 접근법**: 코드 삭제가 아닌 실제 수정
3. **컨텍스트 관리**: 개별 파일이 아닌 프로젝트 전체 이해
4. **지시 준수**: 사용자 요청 범위 내에서 작업

이 글은 Gemini가 작성한 "반성문"을 바탕으로 했습니다. 앞으로도 사례가 추가되면 업데이트할 예정입니다.

---

## 참고 자료

### 관련 문서
- [Claude Code 실무 개발 워크플로우](https://blog.imprun.dev/71) - Claude Code 활용법

### 외부 링크
- [Antigravity IDE](https://antigravity.dev/) - Google의 AI-native IDE
- [Gemini 3.0 Pro](https://deepmind.google/technologies/gemini/) - Google DeepMind

