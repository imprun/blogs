# 2단계 (선택): Headscale 자체 호스팅 서버 구축

> **시리즈**: [Oracle Cloud + Tailscale + Kubernetes 완벽 가이드](README.md)
> ← **이전**: [2단계: Tailscale 메시 네트워크 구성](02-setup-tailscale-network.md) | **다음**: [3단계: Kubernetes + Cilium 구축](03-setup-kubernetes-cilium.md) →
> ⚠️ **이 단계는 선택사항입니다**. Tailscale 공용 서버를 사용하면 건너뛰어도 됩니다.

---

> Tailscale 컨트롤 서버를 직접 운영하여 완전한 통제권 확보

## 📋 이 단계에서 할 일

1. Traefik 리버스 프록시 설치 (HTTPS 필수)
2. Headscale 바이너리 설치
3. Let's Encrypt 인증서 자동 발급
4. 노드 등록 및 관리

## ❓ Headscale이 필요한가요?

### 필요한 경우
- **무제한 노드**: 20개 이상의 노드 관리
- **완전한 통제**: 모든 데이터를 직접 관리
- **프로덕션 환경**: 외부 의존성 제거
- **커스터마이징**: 특별한 네트워크 정책 적용

### 필요 없는 경우
- **빠른 테스트**: Tailscale 공식 서버로 충분
- **소규모 클러스터**: 20개 미만 노드
- **간단한 설정** 선호

## 🏗️ 아키텍처

```
Internet
    ↓
[Traefik :20443] → Let's Encrypt 인증서
    ↓
[Headscale :25896] → localhost only
    ↓
Tailscale Nodes
```

## 📦 Phase 1: Traefik 설치

Headscale은 **반드시 HTTPS**로 서비스되어야 합니다.

### 1. Traefik 바이너리 설치

```bash
# Traefik 다운로드
TRAEFIK_VERSION="3.1.0"
cd /tmp
wget https://github.com/traefik/traefik/releases/download/v${TRAEFIK_VERSION}/traefik_v${TRAEFIK_VERSION}_linux_arm64.tar.gz

# 설치
tar -zxvf traefik_v${TRAEFIK_VERSION}_linux_arm64.tar.gz
sudo mv traefik /usr/local/bin/
sudo chmod +x /usr/local/bin/traefik

# 확인
traefik version
```

### 2. Traefik 사용자 및 디렉토리 생성

```bash
# 사용자 생성
sudo useradd -r -d /var/lib/traefik -m -s /sbin/nologin traefik

# 디렉토리 생성
sudo mkdir -p /etc/traefik/dynamic
sudo mkdir -p /var/lib/traefik
sudo mkdir -p /var/log/traefik

# 권한 설정
sudo chown -R traefik:traefik /etc/traefik
sudo chown -R traefik:traefik /var/lib/traefik
sudo chown -R traefik:traefik /var/log/traefik
```

### 3. Traefik 메인 설정

```bash
cat <<'EOF' | sudo tee /etc/traefik/traefik.yml
global:
  checkNewVersion: false
  sendAnonymousUsage: false

serversTransport:
  insecureSkipVerify: true

api:
  dashboard: true

entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https

  websecure:
    address: ":20443"

providers:
  file:
    directory: /etc/traefik/dynamic
    watch: true

certificatesResolvers:
  cloudflare:
    acme:
      email: your-email@example.com
      storage: /var/lib/traefik/acme.json
      caServer: https://acme-v02.api.letsencrypt.org/directory
      keyType: EC256
      dnsChallenge:
        provider: cloudflare
        resolvers:
          - "1.1.1.1:53"
          - "8.8.8.8:53"

log:
  level: INFO
  filePath: /var/log/traefik/traefik.log

accessLog:
  filePath: /var/log/traefik/access.log
EOF

sudo chown traefik:traefik /etc/traefik/traefik.yml
```

### 4. Headscale 라우팅 설정

```bash
cat <<'EOF' | sudo tee /etc/traefik/dynamic/headscale.yml
http:
  routers:
    headscale:
      rule: "Host(`headscale.yourdomain.com`)"
      service: headscale-service
      entryPoints:
        - websecure
      tls:
        certResolver: cloudflare

  services:
    headscale-service:
      loadBalancer:
        servers:
          - url: "http://127.0.0.1:25896"
EOF

sudo chown traefik:traefik /etc/traefik/dynamic/headscale.yml
```

### 5. Cloudflare API 토큰 설정

```bash
# API 토큰 생성 (Cloudflare 대시보드)
# Zone:DNS:Edit 권한 필요

# 환경 변수 파일 생성
cat <<'EOF' | sudo tee /etc/traefik/traefik.env
CF_DNS_API_TOKEN=YOUR_CLOUDFLARE_API_TOKEN
CF_ZONE_API_TOKEN=YOUR_CLOUDFLARE_API_TOKEN
EOF

sudo chown traefik:traefik /etc/traefik/traefik.env
sudo chmod 600 /etc/traefik/traefik.env
```

### 6. Traefik Systemd 서비스

```bash
cat <<'EOF' | sudo tee /etc/systemd/system/traefik.service
[Unit]
Description=Traefik
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=traefik
Group=traefik
EnvironmentFile=/etc/traefik/traefik.env
ExecStart=/usr/local/bin/traefik --configfile=/etc/traefik/traefik.yml
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal
SyslogIdentifier=traefik
PrivateTmp=true
PrivateDevices=true
ProtectHome=true
ProtectSystem=strict
ReadWritePaths=/var/lib/traefik /var/log/traefik
NoNewPrivileges=true
AmbientCapabilities=CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
EOF

# ACME 저장소 초기화
sudo touch /var/lib/traefik/acme.json
sudo chown traefik:traefik /var/lib/traefik/acme.json
sudo chmod 600 /var/lib/traefik/acme.json

# 서비스 시작
sudo systemctl daemon-reload
sudo systemctl enable --now traefik
```

## 🎮 Phase 2: Headscale 설치

### 1. Headscale 바이너리 설치

```bash
# 최신 버전 다운로드
HEADSCALE_VERSION="0.27.0-beta.2"
wget https://github.com/juanfont/headscale/releases/download/v${HEADSCALE_VERSION}/headscale_${HEADSCALE_VERSION}_linux_arm64

# 설치
chmod +x headscale_${HEADSCALE_VERSION}_linux_arm64
sudo mv headscale_${HEADSCALE_VERSION}_linux_arm64 /usr/local/bin/headscale

# 확인
headscale version
```

### 2. Headscale 설정

```bash
# 디렉토리 생성
sudo mkdir -p /etc/headscale
sudo mkdir -p /var/lib/headscale

# 사용자 생성
sudo useradd -r -d /var/lib/headscale -m -s /sbin/nologin headscale

# 설정 파일 생성
cat <<'EOF' | sudo tee /etc/headscale/config.yaml
server_url: https://headscale.yourdomain.com:20443
listen_addr: 127.0.0.1:25896
metrics_listen_addr: 127.0.0.1:9090

grpc_listen_addr: 127.0.0.1:50443
grpc_allow_insecure: false

private_key_path: /var/lib/headscale/private.key
noise:
  private_key_path: /var/lib/headscale/noise_private.key

prefixes:
  v4: 100.64.0.0/10
  v6: fd7a:115c:a1e0::/48
  allocation: sequential

database:
  type: sqlite
  sqlite:
    path: /var/lib/headscale/db.sqlite

ephemeral_node_inactivity_timeout: 30m
node_update_check_interval: 10s

derp:
  server:
    enabled: false
  urls:
    - https://controlplane.tailscale.com/derpmap/default
  auto_update_enabled: true
  update_frequency: 24h

disable_check_updates: false
log:
  level: info

policy:
  path: ""
  mode: file

dns:
  magic_dns: true
  base_domain: headscale.local
  nameservers:
    global:
      - 1.1.1.1
      - 8.8.8.8
  search_domains: []
  extra_records: []
EOF

sudo chown -R headscale:headscale /etc/headscale
sudo chown -R headscale:headscale /var/lib/headscale
```

### 3. Headscale Systemd 서비스

```bash
cat <<'EOF' | sudo tee /etc/systemd/system/headscale.service
[Unit]
Description=Headscale
After=network.target

[Service]
Type=simple
User=headscale
WorkingDirectory=/var/lib/headscale
ExecStart=/usr/local/bin/headscale serve
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# 서비스 시작
sudo systemctl daemon-reload
sudo systemctl enable --now headscale
```

## 🔑 Phase 3: 노드 등록 준비

### 1. 사용자(네임스페이스) 생성

```bash
# Kubernetes용 네임스페이스 생성
sudo headscale users create k8s

# 확인
sudo headscale users list
```

### 2. 사전 인증 키 생성

```bash
# 재사용 가능한 키 생성 (90일 유효)
sudo headscale --user k8s preauthkeys create --reusable --expiration 2160h

# 생성된 키 확인
sudo headscale preauthkeys list --user k8s

# 출력 예:
# ID | Key               | Reusable | Ephemeral | Used  | Expiration | Created
# 1  | 3fd7c2...8f3d1a2 | true     | false     | false | 90 days    | Now
```

> **중요**: 이 키를 안전하게 보관하세요. 노드 연결 시 필요합니다.

### 3. 연결 URL 확인

```bash
echo "Server URL: https://headscale.yourdomain.com:20443"
echo "Auth Key: <위에서 생성한 키>"
```

## 🌐 DNS 설정

Cloudflare 또는 사용 중인 DNS 제공자에서:

```
Type: A
Name: headscale
Value: <서버_공인_IP>
Proxy: Disabled (DNS Only)
```

## ✅ 검증

### 1. Traefik 상태 확인

```bash
# 서비스 상태
sudo systemctl status traefik

# 인증서 발급 확인
sudo ls -la /var/lib/traefik/acme.json

# 로그 확인
sudo tail -f /var/log/traefik/traefik.log
```

### 2. Headscale 상태 확인

```bash
# 서비스 상태
sudo systemctl status headscale

# 로그 확인
sudo journalctl -u headscale -f

# 버전 확인
curl http://127.0.0.1:25896/health
```

### 3. HTTPS 접속 테스트

```bash
# 외부에서 테스트
curl -I https://headscale.yourdomain.com:20443/health

# 응답 확인
# HTTP/2 200
```

## 🔗 노드 연결

이제 각 노드에서 Headscale에 연결할 수 있습니다:

```bash
# 각 노드에서 실행
sudo tailscale up \
  --login-server=https://headscale.yourdomain.com:20443 \
  --authkey=<YOUR_PREAUTHKEY> \
  --hostname=$(hostname) \
  --accept-routes
```

## 🛠️ 관리 명령어

### 노드 관리

```bash
# 연결된 노드 목록
sudo headscale nodes list

# 노드 이름 변경
sudo headscale nodes rename --identifier <OLD_NAME> --new-name <NEW_NAME>

# 노드 삭제
sudo headscale nodes delete --identifier <NODE_NAME>
```

### 라우트 관리

```bash
# 라우트 목록 (노드별로 확인)
sudo headscale routes list

# 라우트 승인 (새로운 방식)
sudo headscale nodes approve-routes --identifier <NODE_ID> --routes "10.0.0.0/8,192.168.0.0/24"

# 모든 라우트 승인
sudo headscale nodes approve-routes --identifier <NODE_ID> --routes ""

# 특정 노드의 승인된 라우트 확인
sudo headscale nodes list
```

### 사용자 관리

```bash
# 사용자 추가
sudo headscale users create <USERNAME>

# 사용자 삭제
sudo headscale users delete <USERNAME>
```

## ⚠️ 트러블슈팅

### 인증서 발급 실패
- Cloudflare API 토큰 권한 확인
- DNS 레코드 전파 대기 (5-10분)
- Let's Encrypt 속도 제한 확인

### Headscale 연결 실패
- 포트 20443이 열려있는지 확인
- HTTPS가 정상 작동하는지 확인
- 사전 인증 키가 유효한지 확인

### "TLS handshake error"
- Traefik이 정상 실행 중인지 확인
- 인증서가 발급되었는지 확인
- `/var/lib/traefik/acme.json` 권한 확인

## 📋 체크리스트

- [ ] Traefik 설치 및 실행
- [ ] Let's Encrypt 인증서 발급
- [ ] Headscale 설치 및 실행
- [ ] DNS 레코드 설정
- [ ] HTTPS 접속 테스트 성공
- [ ] 사전 인증 키 생성
- [ ] 첫 번째 노드 연결 성공

## 🔄 다음 단계

Headscale 서버가 준비되면:
→ [**03-setup-kubernetes-cilium.md**](03-setup-kubernetes-cilium.md) - Kubernetes 클러스터 구축

---

*참고: Headscale은 선택사항입니다. Tailscale 공식 서버를 사용해도 충분합니다.*