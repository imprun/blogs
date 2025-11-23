# Kubernetes 환경에서 Keycloak 커스텀 로그인 테마 배포하기

> **작성일**: 2025년 11월 23일
> **카테고리**: Kubernetes, Authentication
> **키워드**: Keycloak, Theme, ConfigMap, FreeMarker, Kubernetes

## 요약

Kubernetes 환경에서 Keycloak 커스텀 로그인 테마를 ConfigMap으로 배포하는 과정에서 여러 문제에 직면했습니다. ConfigMap이 중첩 디렉토리 구조를 보존하지 않는 특성과 네임스페이스 불일치 문제가 주요 원인이었습니다. 이 글에서는 Keycloak 테마의 구조, Kubernetes에서의 배포 방법, 그리고 배포 과정에서 겪은 시행착오와 해결 방법을 공유합니다.

## 문제 상황

### 증상

프로덕션 환경에서 Keycloak의 기본 로그인 UI 대신 자체 브랜딩에 맞는 커스텀 테마가 필요했습니다. Kubernetes 환경에서 테마 파일을 ConfigMap으로 배포하려 했으나, 다음과 같은 문제들이 발생했습니다:

```
error: failed to create configmap: namespaces "keycloak" not found
```

```
Warning  FailedMount  MountVolume.SetUp failed for volume "theme-imprun":
         configmap "keycloak-theme-imprun" not found
```

테마 파일이 마운트되었음에도 Keycloak Admin Console의 Theme 드롭다운에 커스텀 테마가 나타나지 않는 현상도 발생했습니다.

### 환경 구성

- **Keycloak**: Kubernetes 클러스터 내 Deployment로 배포
- **네임스페이스**: `keycloak-system`
- **테마 배포 방식**: ConfigMap을 볼륨으로 마운트
- **테마 경로**: `/opt/keycloak/themes/imprun/`

### 초기 진단

ConfigMap 생성 시도 결과:

```bash
$ kubectl create configmap keycloak-theme-imprun --from-file=imprun/ -n keycloak
error: failed to create configmap: namespaces "keycloak" not found
```

실제 네임스페이스 확인:

```bash
$ kubectl get pods -A | grep -i keycloak
keycloak-system   keycloak-xxx   1/1   Running   0   5d
```

Keycloak이 `keycloak` 네임스페이스가 아닌 `keycloak-system` 네임스페이스에 배포되어 있었습니다.

## 근본 원인 분석

### ConfigMap의 플랫 구조 특성

Kubernetes ConfigMap은 중첩 디렉토리 구조를 보존하지 않습니다. `kubectl create configmap --from-file=<directory>` 명령은 모든 파일을 최상위 레벨에 플랫하게 저장합니다.

Keycloak 테마는 다음과 같은 디렉토리 구조를 요구합니다:

```
/opt/keycloak/themes/{theme-name}/
└── login/
    ├── theme.properties
    ├── template.ftl
    ├── login.ftl
    ├── resources/
    │   └── css/
    │       └── login.css
    └── messages/
        ├── messages_en.properties
        └── messages_ko.properties
```

### 설정 오류

당초 의도한 구성:
```
단일 ConfigMap (keycloak-theme-imprun)
└── 중첩 디렉토리 구조로 마운트
    ├── theme.properties
    ├── template.ftl
    ├── login.ftl
    ├── resources/css/login.css
    └── messages/messages_*.properties
```

실제 구성:
```
단일 ConfigMap (keycloak-theme-imprun)
└── 플랫한 key-value 저장
    ├── theme.properties (key)
    ├── template.ftl (key)
    ├── login.ftl (key)
    ├── resources/css/login.css (key) → 파일명이 아닌 문자열 키로 저장됨
    └── messages/messages_en.properties (key) → 하위 디렉토리로 마운트 안 됨
```

`resources/css/login.css` 파일은 `resources/css/login.css`라는 문자열 키로 저장되어, 실제 하위 디렉토리로 마운트되지 않습니다. Keycloak은 `resources/css/` 디렉토리 내에 CSS 파일이 있어야 테마를 로드할 수 있습니다.

### 네임스페이스 불일치

Keycloak이 `keycloak-system` 네임스페이스에 배포되어 있었으나, ConfigMap 생성 시 `keycloak` 네임스페이스를 사용하여 리소스를 찾을 수 없는 오류가 발생했습니다.

### ConfigMap 참조 불일치

ConfigMap 이름을 변경한 후 Deployment의 볼륨 참조를 업데이트하지 않아 Pod가 `ContainerCreating` 상태에서 멈추는 문제가 발생했습니다.

## 해결 과정

### 1. 네임스페이스 확인

```bash
# 실제 Keycloak 네임스페이스 확인
$ kubectl get pods -A | grep -i keycloak
keycloak-system   keycloak-xxx   1/1   Running   0   5d
```

### 2. 다중 ConfigMap 생성

디렉토리 레벨별로 별도의 ConfigMap을 생성합니다:

```yaml
# keycloak-theme-configmaps.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: keycloak-theme-imprun-login
  namespace: keycloak-system
data:
  theme.properties: |
    parent=keycloak
    import=common/keycloak
    styles=css/login.css

  template.ftl: |
    <#macro registrationLayout bodyClass="" displayInfo=false displayMessage=true>
    <!DOCTYPE html>
    <html>
    <head>
        <meta charset="utf-8">
        <title>${msg("loginTitle",(realm.displayName!''))}</title>
        <#if properties.styles?has_content>
            <#list properties.styles?split(' ') as style>
                <link href="${url.resourcesPath}/${style}" rel="stylesheet" />
            </#list>
        </#if>
        <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
    </head>
    <body class="login-pf ${bodyClass}">
        <div class="login-pf-page">
            <div id="kc-content-wrapper">
                <#nested "header">
                <#if displayMessage && message?has_content>
                    <div class="alert alert-${message.type}">
                        ${kcSanitize(message.summary)?no_esc}
                    </div>
                </#if>
                <#nested "form">
                <#if displayInfo><#nested "info"></#if>
            </div>
        </div>
    </body>
    </html>
    </#macro>

  login.ftl: |
    <#import "template.ftl" as layout>
    <@layout.registrationLayout displayInfo=realm.password && realm.registrationAllowed; section>
        <#if section = "header">
            <div class="imprun-logo">
                <div class="imprun-logo-icon">
                    <svg><!-- 로고 SVG --></svg>
                </div>
                <span class="imprun-logo-text">IMPRUN</span>
            </div>
        <#elseif section = "form">
            <form action="${url.loginAction}" method="post">
                <div class="form-group">
                    <label for="username">${msg("usernameOrEmail")}</label>
                    <input id="username" name="username" type="text" autofocus />
                </div>
                <div class="form-group">
                    <label for="password">${msg("password")}</label>
                    <input id="password" name="password" type="password" />
                </div>
                <input type="submit" id="kc-login" value="${msg("doLogIn")}" />
            </form>
        <#elseif section = "info">
            <#if realm.registrationAllowed>
                <a href="${url.registrationUrl}">${msg("doRegister")}</a>
            </#if>
        </#if>
    </@layout.registrationLayout>

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: keycloak-theme-imprun-css
  namespace: keycloak-system
data:
  login.css: |
    :root {
      --primary: #3b82f6;
      --background: #171717;
      --card: #262626;
      --foreground: #fafafa;
      --border: rgba(255, 255, 255, 0.1);
    }

    body.login-pf {
      background: linear-gradient(135deg, #0f0f0f, #1a1a2e, #16213e);
      min-height: 100vh;
      display: flex;
      align-items: center;
      justify-content: center;
      font-family: 'Inter', sans-serif;
    }

    #kc-form-wrapper {
      background: var(--card);
      border: 1px solid var(--border);
      border-radius: 0.625rem;
      padding: 2rem;
      max-width: 28rem;
    }

    #kc-login {
      width: 100%;
      background: var(--primary);
      color: white;
      border: none;
      padding: 0.75rem;
      border-radius: 0.5rem;
      cursor: pointer;
    }

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: keycloak-theme-imprun-messages
  namespace: keycloak-system
data:
  messages_en.properties: |
    loginTitleHtml=Sign in to your account
    doLogIn=Sign In
    doRegister=Create Account

  messages_ko.properties: |
    loginTitleHtml=계정에 로그인
    doLogIn=로그인
    doRegister=계정 만들기
```

```bash
$ kubectl apply -f keycloak-theme-configmaps.yaml
configmap/keycloak-theme-imprun-login created
configmap/keycloak-theme-imprun-css created
configmap/keycloak-theme-imprun-messages created
```

### 3. Deployment에 볼륨 마운트 구성

각 ConfigMap을 해당 디렉토리 경로에 마운트합니다:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: keycloak
  namespace: keycloak-system
spec:
  template:
    spec:
      containers:
        - name: keycloak
          volumeMounts:
            - name: theme-login
              mountPath: /opt/keycloak/themes/imprun/login
            - name: theme-css
              mountPath: /opt/keycloak/themes/imprun/login/resources/css
            - name: theme-messages
              mountPath: /opt/keycloak/themes/imprun/login/messages
      volumes:
        - name: theme-login
          configMap:
            name: keycloak-theme-imprun-login
        - name: theme-css
          configMap:
            name: keycloak-theme-imprun-css
        - name: theme-messages
          configMap:
            name: keycloak-theme-imprun-messages
```

```bash
$ kubectl apply -f keycloak-deployment.yaml
$ kubectl rollout status deployment keycloak -n keycloak-system
deployment "keycloak" successfully rolled out
```

### 4. 결과 검증

```bash
$ kubectl exec -it $(kubectl get pod -n keycloak-system -l app=keycloak \
  -o jsonpath='{.items[0].metadata.name}') -n keycloak-system \
  -- ls -la /opt/keycloak/themes/imprun/login/
total 16
drwxr-xr-x 4 root root 4096 Nov 23 10:00 .
drwxr-xr-x 3 root root 4096 Nov 23 10:00 ..
-rw-r--r-- 1 root root  156 Nov 23 10:00 login.ftl
drwxr-xr-x 3 root root 4096 Nov 23 10:00 messages
drwxr-xr-x 3 root root 4096 Nov 23 10:00 resources
-rw-r--r-- 1 root root 1024 Nov 23 10:00 template.ftl
-rw-r--r-- 1 root root   58 Nov 23 10:00 theme.properties
```

테마 파일이 올바른 디렉토리 구조로 마운트되었습니다.

### 5. Keycloak Admin Console에서 테마 적용

1. Keycloak Admin Console 접속
2. 좌측 메뉴에서 **Realm settings** 클릭
3. **Themes** 탭 선택
4. **Login theme** 드롭다운에서 `imprun` 선택
5. **Save** 클릭

## 재발 방지 방안

### 1. ConfigMap 분리 전략

Keycloak 테마 배포 시 디렉토리 레벨별로 ConfigMap을 분리합니다:

```
keycloak-theme-{name}-login    → /opt/keycloak/themes/{name}/login/
keycloak-theme-{name}-css      → /opt/keycloak/themes/{name}/login/resources/css/
keycloak-theme-{name}-messages → /opt/keycloak/themes/{name}/login/messages/
```

### 2. 검증 절차

테마 배포 후 다음 항목을 순서대로 확인합니다:

```bash
# 1. ConfigMap 존재 확인
$ kubectl get configmap -n keycloak-system | grep keycloak-theme

# 2. 볼륨 마운트 확인
$ kubectl exec -it <pod> -n keycloak-system -- ls -la /opt/keycloak/themes/imprun/login/

# 3. theme.properties 확인
$ kubectl exec -it <pod> -n keycloak-system -- cat /opt/keycloak/themes/imprun/login/theme.properties
```

### 3. 모니터링

테마 로드 오류를 감지하기 위해 Keycloak 로그를 모니터링합니다:

```bash
$ kubectl logs -f <pod> -n keycloak-system | grep -i "theme\|freemarker"
```

## 교훈

### 1. ConfigMap은 플랫한 key-value 구조

Kubernetes ConfigMap은 중첩 디렉토리 구조를 지원하지 않습니다. 복잡한 디렉토리 구조가 필요한 경우 여러 ConfigMap을 생성하고 각각 다른 경로에 마운트해야 합니다.

### 2. 네임스페이스 확인 우선

리소스를 생성하기 전에 항상 `kubectl get pods -A | grep <app-name>`으로 실제 네임스페이스를 확인해야 합니다.

### 3. ConfigMap 변경 시 의존 리소스 동시 업데이트

ConfigMap 이름을 변경하면 해당 ConfigMap을 참조하는 모든 Deployment, Pod 등의 볼륨 참조도 함께 업데이트해야 합니다.

### 4. 테마 변경 후 캐시 클리어

테마 파일 변경 후 반영되지 않으면 Keycloak을 재시작해야 합니다:

```bash
$ kubectl rollout restart deployment keycloak -n keycloak-system
```

## 부록: FreeMarker 템플릿 참조

### 주요 변수

| 변수 | 설명 |
|-----|------|
| `${url.loginAction}` | 로그인 폼 제출 URL |
| `${url.registrationUrl}` | 회원가입 페이지 URL |
| `${url.loginResetCredentialsUrl}` | 비밀번호 재설정 URL |
| `${url.resourcesPath}` | 테마 리소스 경로 |
| `${msg("key")}` | 메시지 번들에서 텍스트 가져오기 |
| `${realm.displayName}` | Realm 표시 이름 |
| `${login.username}` | 이전에 입력한 사용자명 |

### 조건문

```ftl
<#if realm.password && realm.registrationAllowed>
    <a href="${url.registrationUrl}">회원가입</a>
</#if>

<#if messagesPerField.existsError('username','password')>
    <span class="error">${kcSanitize(messagesPerField.getFirstError('username','password'))?no_esc}</span>
</#if>
```

### 소셜 로그인 반복문

```ftl
<#if social.providers??>
    <#list social.providers as p>
        <a href="${p.loginUrl}">${p.displayName}</a>
    </#list>
</#if>
```

### 추가 커스터마이징 가능 페이지

| 파일 | 페이지 |
|-----|-------|
| `login-reset-password.ftl` | 비밀번호 재설정 요청 |
| `login-update-password.ftl` | 비밀번호 변경 |
| `login-otp.ftl` | OTP 입력 |
| `login-verify-email.ftl` | 이메일 인증 |
| `error.ftl` | 에러 페이지 |

## 참고 자료

### 관련 문서
- Keycloak Theme 방식과 Direct Access Grant 방식 비교 시, 보안과 MFA 지원을 고려하여 Theme 방식을 선택

### 공식 문서
- [Keycloak Server Developer Guide - Themes](https://www.keycloak.org/docs/latest/server_development/#_themes)
- [FreeMarker Manual](https://freemarker.apache.org/docs/)
- [Keycloak Default Theme Source](https://github.com/keycloak/keycloak/tree/main/themes/src/main/resources/theme)
