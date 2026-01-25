# 이력서

## 기본 정보

| 항목 | 내용 |
|------|------|
| **이름** | 박준식 (Junsik Park) |
| **생년월일** | 1977.04.01 |
| **연락처** | 010-5696-9317 |
| **이메일** | junsik.park@gmail.com |
| **주소** | 서울시 |

---

## 학력

| 기간 | 학교 | 학과 | 구분 |
|------|------|------|------|
| 1998~2006 | 동국대학교 | 컴퓨터공학 | 학사 |

---

## 자격사항

| 취득일자 | 자격종류 | 발행처 |
|----------|----------|--------|
| 1997 | 정보처리기능사 | 국가기술자격검정원 |

---

## 경력 요약

| 근무기간 | 회사명 | 직무 | 직급/직책 |
|----------|--------|------|-----------|
| 2025.11 - 현재 | imprun | 플랫폼 개발 | Founder |
| 2019.11 - 2025.10 | 기웅정보통신 | 스크래핑 엔진/플랫폼 개발 | 팀장 |
| 2019.04 - 2019.10 | 카페24 | Edge 서버 개발 | Sr. Engineer |
| 2018 - 2019 | 프리랜서 | Backend 개발 | Sr. Engineer |
| 2017 - 2018 | 기웅정보통신 | 스크래핑 엔진 개발 | 차장/팀장 |
| 2017 | WindForce | 네트워크 가속 플랫폼 | Co-founder |
| 2014 - 2017 | FasterThan | 네트워크 가속 플랫폼 | Co-founder |
| 2012 - 2014 | 씨디네트웍스 | HTTP Cache 서버 개발 | 책임연구원 |
| 2008 - 2012 | CDNetworks US | Backend 플랫폼 개발 | 선임연구원 |
| 2003 - 2008 | 씨디네트웍스 | 미디어 플랫폼 개발 | 선임연구원 |
| 2001 - 2003 | 코아정보통신 | 웹 시스템 개발 | 연구원 |
| 2000 - 2001 | 메디오피아 | e-Learning 개발 | 연구원 |

---

## 자기소개

25년간 CDN 인프라, 네트워크 가속, 대용량 스크래핑 시스템 등 다양한 분야에서 고성능 서버와 플랫폼을 설계하고 구현해왔습니다.

분산 시스템 설계, 저수준 네트워크 스택 구현, 일 수백만 건 트랜잭션 처리 플랫폼 구축이 핵심 역량입니다. C, Go, Java, Python에 능숙하며, HTTP와 TCP/IP 내부 동작에 대한 깊은 이해를 바탕으로 개발합니다.

여러 프로젝트에서 훌륭한 멘토와 동료들을 만나 코드 리뷰와 기술 토론을 통해 문제 해결 능력을 키워왔습니다. **"개발자는 코드로 말한다"**는 신념으로 순수 개발자의 길을 걷고 있습니다.

---

## 핵심 역량

- 25년 크로스 플랫폼 애플리케이션 개발
- 11년 CDN 플랫폼 개발 경험 (미디어 서비스, 대규모 설정 배포 시스템, Cache 서버)
- 고성능 서버 아키텍처 설계 및 구현
- 대규모 분산 시스템 설계 (설정 배포, 모니터링, 데이터 처리)
- 저수준 네트워크 프로그래밍 (TCP/IP stack, userland networking)
- 스크래핑 엔진 및 안티스크래핑 우회 기술
- 주요 언어: C, Go, Java, Python, JavaScript

---

## 경력 기술서

### imprun (2025.11 - 현재)

**Founder**

API 라이프사이클 관리 플랫폼인 [imprun](https://imprun.dev)을 개발하고 있습니다. API 제공자가 Gateway를 통해 서비스를 관리하고, 개발자가 API를 발견하고 접근할 수 있는 Developer Portal을 제공합니다.

#### 핵심 아키텍처
- Envoy Gateway 기반 Kubernetes 네이티브 API Gateway
- Ory 스택 통합 인증/인가 (Kratos, Hydra, Keto, Oathkeeper)
- ReBAC(Relationship-Based Access Control) 기반 세분화된 권한 관리
- Multi-Region 지원 (kr, us, eu)
- Organization → Gateway → Product 계층 구조의 멀티테넌시

#### 기술 스택
- Backend: Go (Gin), GORM, PostgreSQL, Redis
- Identity: Ory Kratos (사용자), Ory Hydra (OAuth2/OIDC)
- Authorization: Ory Keto (ReBAC), Ory Oathkeeper (Zero Trust IAP)
- Infrastructure: Kubernetes, Envoy Gateway
- Frontend: Next.js 15, Tailwind CSS v4, shadcn/ui

개발 과정에서 경험한 기술적 여정을 [기술 블로그](https://blog.imprun.dev)에 기록하고 있습니다.

---

### 기웅정보통신 (2019.11 - 2025.10)

**팀장 / Principal Engineer** | 6년 연속 성과 Top 3

국내 핀테크 데이터 서비스 선두 기업에서 스크래핑 코어 엔진, SDK, 개발 프레임워크를 설계하고 구현했습니다. 현재 회사 내 대부분의 스크래핑이 이 개발 방법론을 기반으로 운영되고 있습니다.

#### 스크래핑 코어 엔진 (C)
- QuickJS + libcurl 기반 스크래핑 엔진 설계 및 구현
- JavaScript 런타임용 XHR, XML/HTML 파서 개발
- 암복호화 모듈 및 PKI 연동 모듈 개발
- 민감정보(PII) 자동 삭제 기능 구현
- Android, iOS, Windows, Linux 크로스 플랫폼 지원

#### JavaScript 스크래핑 SDK
- C 코어 엔진 기반 JavaScript SDK 설계
- 스크래핑 스크립트 개발용 API 설계 및 구현

#### 개발 프레임워크 구축 및 전파
- 전사 표준 개발 프레임워크 설계 및 구축
- 개발자 교육 자료 작성 및 사내 교육 진행
- 코드 리뷰 프로세스 도입 및 정착
- 현재 회사 내 스크래핑 개발 표준으로 정착

#### 데이터허브 플랫폼 운영
- 일 500만 건 스크래핑 요청 처리 플랫폼 운영
- Go 기반 고성능 스크래핑 워커 개발
- 작업 스케줄링, 로드 밸런싱, 장애 복구 메커니즘 구현

#### 안티스크래핑 우회 인프라
- 세션 프록시 서버 개발 (안티스크래핑 우회 기술 적용)
- 브라우저 핑거프린트 관리 및 로테이션 시스템 구현
- 고가용성 분산 세션 관리 시스템 구축

---

### 카페24 (2019.04 - 2019.10)

**Sr. Software Engineer** | SRE팀

쇼핑몰 호스팅 서비스를 위한 Kubernetes 기반 Edge 서버 개발에 참여했습니다.

#### Edge 서버 개발
- Kubernetes 기반 Edge 서버 아키텍처 설계 및 구현
- NGINX 동적 가상 도메인 모듈 개발 (10만 개 도메인 수 초 내 적재)
- 서비스 전반의 이슈 분석 및 크로스팀 협업을 통한 문제 해결

---

### 프리랜서 (2018 - 2019)

**Sr. Software Engineer**

#### K-Center 장비 검증 시스템
- 시스템 아키텍처 설계 및 서버 파트 구현
- Django REST Framework 기반 API 개발
- 시스템 모니터링 및 진단 결과 수집 에이전트 개발

#### NexG Firewall CLI
- 콘솔 명령어를 XML로 변환하여 관리 데몬과 통신하는 CLI 개발
- Quagga CLI 엔진 기반 C 언어 구현

---

### 기웅정보통신 (2017 - 2018)

**차장 / 팀장**

Android/iOS 기반 차세대 스크래핑 엔진 개발팀을 이끌었습니다.

#### SmartAIB SDK
- duktape JavaScript 엔진 Android/iOS 포팅으로 SDK 용량 1/10 축소
- 병렬 스크래핑 지원으로 처리 속도 2~3배 향상
- 금감원 모바일 보안 가이드 준수 (민감정보 메모리 잔류 방지)
- 하나은행 보안 인증 획득
- SVN, Code Review, CI 등 개발 프로세스 체계화

#### 특허
- 인력정보 검증서비스 제공 시스템 및 방법 (KR101950769B1)

---

### WindForce (2017)

**Co-founder**

WAN 가속을 위한 Userland TCP/IP 기반 Transparent TCP Proxy를 개발했습니다.

#### Windforce TCP Proxy
- FreeBSD 10.3 기반 Transparent TCP Proxy 구현
- IPv4/IPv6 및 PROXY 프로토콜 지원
- WAN 최적화 혼잡 제어 알고리즘 적용
- PayPal 연동 라이선스 결제 시스템 구현
- Django 기반 고객 포털 개발

---

### FasterThan (2014 - 2017)

**Co-founder & Sr. Software Engineer**

모바일 네트워크 가속 스타트업의 창립 멤버로, 서버/클라이언트/SDK 전반의 코어 엔진을 개발했습니다.

#### TCP 가속 엔진 및 SDK
- FreeBSD 10.1 TCP/IP 스택을 Android, iOS, Windows, Linux, macOS용으로 포팅
- 커널 인터페이스 재구현 (malloc, locks, callout, pcpu, kthread, uma, TLS)
- 네트워크 디버깅 도구 포팅 (netstat, arp, route, ifconfig, sysctl)
- 루팅 없이 사용 가능한 모바일 SDK 설계 및 구현
- 고성능 TCP/HTTP Proxy 구현 (epoll, kqueue, overlapped I/O)
- Connection pooling, lockless circular buffer, multi-threading 적용
- NGINX, netperf, curl 통합
- **특허**: 가속 데이터 전송 서버 및 방법 (KR 10-2014-0177731)

#### Android VPN App
- VPN 기반 네트워크 가속 앱 개발
- Google Play 결제 및 소셜 로그인(Facebook, Google) 연동

#### 가속 웹브라우저
- Firefox에 Thunder SDK 내장하여 웹 브라우징 가속 지원

---

### 씨디네트웍스 (2012 - 2014)

**책임연구원**

전 세계 120개 POP, 3,000대 이상 서버에서 운영되는 고성능 HTTP Cache 서버 개발팀에서 성능 최적화와 모니터링 시스템을 담당했습니다.

#### HTTP Cache Server 최적화
- GeoIP 메모리 사용량 300MB → 1MB 감소 (알고리즘 및 자료구조 개선)
- Java Pattern Matcher를 JNI/PCRE로 대체하여 문자열 검색 성능 향상
- 2GB 초과 대용량 파일 서비스 지원
- Blocking I/O를 Non-blocking NIO로 전환하여 모니터링/설정 배포 성능 개선
- Timing Wheel 기반 효율적인 타임아웃 처리 모듈 개발
- 회귀 테스트 작성 및 릴리즈 노트 관리

#### Web Application Firewall
- ModSecurity(C)를 Java로 포팅하여 Cache 서버에 통합

---

### CDNetworks US (2008 - 2012)

**선임연구원** | San Jose, CA

전 세계 10,000대 이상 서버에 설정을 배포하는 대규모 분산 시스템을 설계하고 구현했습니다.

#### Configuration Distribution System
- 대규모 분산 설정 배포 시스템 아키텍처 설계
- 현재 120개 POP, 3,000대 이상 서버에서 운영 중

#### Log Collector
- 로그 유실 방지 및 중복 업로드 방지 로직이 적용된 로그 수집 시스템 개발
- multi-curl 기반 비동기 HTTP 에이전트 구현

#### 설정 관리 포털
- Django 기반 운영자용 설정 배포 웹 인터페이스 개발

---

### 씨디네트웍스 (2003 - 2008)

**선임연구원** | 2년간 4회 연속 S등급(최고 성과) 달성

CDN 초창기부터 미디어 플랫폼 전반(플레이어, 트랜스코더, 미디어 서버, DRM)을 개발했습니다.

#### Hermes Sync Server
- FTP 기반 실시간 파일 동기화 서버 개발 (Java)
- 업로드 완료 즉시 체인으로 연결된 모든 서버에 동기화

#### Aqua Player
- 메가스터디 등 e-Learning 사이트에서 사용된 미디어 플레이어 개발
- DirectShow 기반 미디어 엔진 및 MFC 기반 스킨 엔진 구현
- FFmpeg 기반 Transform Filter 개발
- 녹화 방지 기능 구현 (**특허**: KR 10-2007-0086902)

#### 분산 트랜스코더 시스템
- Uploader, Transcoder, Job Manager, Packager, Deployer로 구성된 분산 트랜스코딩 플랫폼 설계 및 구현
- mgoon.com 서비스에 적용

#### Flash Media Server
- HTTP Cache Plugin 개발

---

### 코아정보통신 (2001 - 2003)

**연구원**

Struts 기반 Java 웹 개발. 국가 프로젝트인 고문서 수집 관리 시스템 개발에 참여했습니다.

---

### 메디오피아 (2000 - 2001)

**연구원**

e-Learning 애플리케이션 개발. PPT 화면 위에서 동작하는 재생 가능한 드로잉 툴과 채팅 프로그램을 MFC로 구현했습니다.

---

## 특허

| 특허명 | 특허번호 | 국가 |
|--------|----------|------|
| 인력정보 검증서비스 제공 시스템 및 방법 | KR101950769B1 | 한국 |
| 가속 데이터 전송 서버 및 방법 | KR 10-2014-0177731 | 한국 |
| 이벤트 감지를 이용한 영상 데이터 녹화 방지 방법 및 장치 | KR 10-2007-0086902 | 한국 |
| 콘텐츠 제공 방법 및 이를 이용한 서버 | US 12/673,079 | 미국 |
| 디지털 미디어 콘텐츠 무단 복제 방지 | US 12/201,920 | 미국 |

---

## 기술 스택

| 분류 | 기술 |
|------|------|
| **언어** | C, Go, Java, Python, JavaScript |
| **시스템** | Linux, FreeBSD, Windows, Android, iOS |
| **네트워크** | TCP/IP Stack, HTTP, TLS/SSL, PKI |
| **런타임/라이브러리** | QuickJS, libcurl, Django, DirectShow, FFmpeg |
| **인프라** | Kubernetes, Docker, NGINX |
| **도구** | Git, CI/CD |

---

**작성일**: 2025.12.25
**작성자**: 박준식
