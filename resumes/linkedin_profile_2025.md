# LinkedIn Profile - Junsik Park

## Headline
Founder at imprun | 25+ Years in High-Performance Systems | Ex-CDNetworks, Kiwoom InfoTech

---

## About

Building high-performance servers and platforms for over 25 years across CDN infrastructure, network acceleration, and large-scale data scraping systems.

Core expertise includes distributed system design, low-level network stack implementation, and platforms handling millions of daily transactions. 11 years of CDN platform development experience spanning media services, large-scale configuration distribution systems, and cache servers. Proficient in C, Go, Java, and Python with deep understanding of HTTP and TCP/IP internals.

Currently building imprun (https://imprun.dev) and documenting the technical journey at https://blog.imprun.dev

Specialties: Server Architecture, High-Performance Systems, Network Programming, Distributed Systems

---

## Experience

### imprun
**Founder**
Nov 2025 - Present | Seoul, Korea

Building imprun (https://imprun.dev), an API Lifecycle Management Platform. Enables API providers to manage gateway services and developers to discover and access APIs through a Developer Portal.

▶ Core Architecture
• Envoy Gateway-based Kubernetes-native API Gateway
• Ory stack for unified auth (Kratos, Hydra, Keto, Oathkeeper)
• ReBAC (Relationship-Based Access Control) for fine-grained permissions
• Multi-Region support (kr, us, eu)
• Multi-tenancy with Organization → Gateway → Product hierarchy

▶ Tech Stack
• Backend: Go (Gin), GORM, PostgreSQL, Redis
• Identity: Ory Kratos (users), Ory Hydra (OAuth2/OIDC)
• Authorization: Ory Keto (ReBAC), Ory Oathkeeper (Zero Trust IAP)
• Infrastructure: Kubernetes, Envoy Gateway
• Frontend: Next.js 15, Tailwind CSS v4, shadcn/ui

Documenting the technical journey at https://blog.imprun.dev

---

### Kiwoom InfoTech (기웅정보통신)
**Team Lead / Principal Engineer**
Nov 2019 - Oct 2025 · 6 yrs | Seoul, Korea

Led the development of scraping core engine, SDK, and development framework at Korea's leading fintech data service company. The development methodology I established is now the company-wide standard for scraping operations. Ranked in top 3 performers for 6 consecutive years.

▶ Core Scraping Engine (C)
• Designed and implemented scraping engine based on QuickJS + libcurl
• Developed XHR implementation and XML/HTML parser for JavaScript runtime
• Built encryption/decryption modules and PKI integration
• Implemented automatic PII (Personally Identifiable Information) redaction
• Cross-platform support for Android, iOS, Windows, Linux

▶ JavaScript Scraping SDK
• Designed JavaScript SDK on top of the C core engine
• Created developer-friendly API for rapid scraping script development

▶ Development Framework
• Established company-wide standard development framework
• Created training materials and conducted internal developer education
• Introduced and established code review processes

▶ DataHub Platform
• Operated data processing platform handling 5M+ daily scraping requests
• Developed high-performance scraping workers in Go
• Implemented job scheduling, load balancing, and fault tolerance mechanisms

▶ Anti-Scraping Bypass Infrastructure
• Developed session proxy servers with anti-detection techniques
• Implemented browser fingerprint management and rotation
• Built distributed session management for high-availability scraping

---

### Cafe24 (카페24)
**Sr. Software Engineer** | SRE Team
Apr 2019 - Oct 2019 · 7 mos | Seoul, Korea

Participated in designing and implementing Kubernetes-based Edge servers for e-commerce hosting services.

• Designed and implemented Edge server architecture on Kubernetes
• Developed NGINX dynamic virtual domain module (loading 100K domains in seconds)
• Cross-team collaboration for service-wide issue analysis and resolution

---

### Freelancer
**Sr. Software Engineer**
2018 - 2019 · 1 yr | Seoul, Korea

▶ K-Center Equipment Verification System
• System architecture design and server implementation
• REST API development using Django REST Framework
• System monitoring and diagnostic agent development

▶ NexG Firewall CLI
• CLI development for device control via XML communication with management daemon
• C implementation based on Quagga CLI engine

---

### Kiwoom InfoTech (기웅정보통신)
**Deputy General Manager / Team Lead**
2017 - 2018 · 1 yr | Seoul, Korea

Led the next-generation scraping engine development team for Android/iOS.

▶ SmartAIB SDK
• Ported duktape JavaScript engine to Android/iOS, reducing SDK size to 1/10
• Enabled parallel scraping, improving processing speed by 2-3x
• Complied with FSS mobile security guidelines (preventing sensitive data memory retention)
• Obtained security certification from Hana Bank
• Established development processes: SVN, Code Review, CI

▶ Patent: System and Method for Providing Human Resources Verification Service (KR101950769B1)

---

### WindForce
**Co-founder**
2017 · 1 yr | Seoul, Korea

Developed Transparent TCP Proxy based on Userland TCP/IP stack for WAN acceleration.

• FreeBSD 10.3-based Transparent TCP Proxy implementation
• IPv4/IPv6 and PROXY protocol support
• WAN-optimized congestion control algorithm
• PayPal-integrated license payment system
• Django-based customer portal

---

### FasterThan
**Co-founder & Sr. Software Engineer**
2014 - 2017 · 3 yrs | Seoul, Korea

Founding member of mobile network acceleration startup. Developed core engines across server, client, and SDK.

▶ TCP Acceleration Engine & SDK
• Ported FreeBSD 10.1 TCP/IP stack to Android, iOS, Windows, Linux, macOS
• Re-implemented kernel interfaces (malloc, locks, callout, pcpu, kthread, uma, TLS)
• Ported network debugging tools (netstat, arp, route, ifconfig, sysctl)
• Designed and implemented mobile SDK usable without rooting
• Built high-performance TCP/HTTP Proxy (epoll, kqueue, overlapped I/O)
• Applied connection pooling, lockless circular buffer, multi-threading
• Integrated with NGINX, netperf, curl

▶ Patent: Server and Method for Transmitting Acceleration Data (KR 10-2014-0177731)

▶ Android VPN App
• VPN-based network acceleration app
• Google Play payment and social login (Facebook, Google) integration

▶ Accelerated Web Browser
• Embedded Thunder SDK in Firefox for web browsing acceleration

---

### CDNetworks
**Senior Software Engineer**
2012 - 2014 · 2 yrs | Seoul, Korea

Joined the WebCache team developing high-performance HTTP cache server (pure Java), deployed on 3,000+ servers across 120 global POPs.

▶ HTTP Cache Server Optimization
• Reduced GeoIP memory usage from 300MB to 1MB (algorithm and data structure optimization)
• Replaced Java Pattern Matcher with JNI/PCRE for improved string search performance
• Added support for files larger than 2GB
• Converted blocking I/O to non-blocking NIO for faster monitoring and config distribution
• Developed efficient timeout handling module using Timing Wheel
• Regression testing and release note management

▶ Web Application Firewall
• Ported ModSecurity (C) to Java and integrated with cache server

---

### CDNetworks
**Senior Software Engineer**
2008 - 2012 · 4 yrs | San Jose, CA, USA

Designed and implemented large-scale distributed systems for deploying configurations to 10,000+ servers worldwide.

▶ Configuration Distribution System
• Designed architecture for large-scale distributed configuration deployment
• Currently operating on 3,000+ servers across 120 POPs

▶ Log Collector
• Developed log collection system with loss prevention and duplicate upload prevention
• Implemented async HTTP agent using multi-curl

▶ Configuration Management Portal
• Developed Django-based configuration deployment interface for operators

---

### CDNetworks
**Software Engineer**
2003 - 2008 · 5 yrs | Seoul, Korea

Developed full-range media platform from CDN's early days: player, transcoder, media server, DRM. Achieved S-grade (highest rating) for 4 consecutive evaluation periods over 2 years.

▶ Hermes Sync Server
• FTP-based real-time file synchronization server (Java)
• Immediate synchronization to all chained servers upon upload completion

▶ Aqua Player
• Media player used by e-learning sites including Megastudy
• DirectShow-based media engine and MFC-based skin engine
• FFmpeg-based transform filter development
• Recording prevention feature (Patent: KR 10-2007-0086902)

▶ Distributed Transcoder System
• Designed distributed transcoding platform (Uploader, Transcoder, Job Manager, Packager, Deployer)
• Deployed for mgoon.com service

▶ Flash Media Server
• HTTP Cache Plugin development

---

### CoreInfo System
**Software Engineer**
2001 - 2003 · 2 yrs | Seoul, Korea

Java web development using Struts. Participated in national project for historical manuscripts management system.

---

### Mediopia
**Software Engineer**
2000 - 2001 · 1 yr | Seoul, Korea

Developed e-learning application. Implemented replayable drawing tool and chat program on PPT using MFC.

---

## Education

### Dongguk University
**BS, Computer Engineering**
1998 - 2006

---

## Patents

**System and Method for Providing Human Resources Verification Service**
KR101950769B1 · Issued 2019

**Server and Method for Transmitting Acceleration Data**
KR 10-2014-0177731 · Issued 2016

**Method and Apparatus for Preventing Recording of Image Data by Using Event Detection**
KR 10-2007-0086902

**Method for Providing Contents to Client and Server Using the Same**
US 12/673,079

**Protection Against Unauthorized Copying of Digital Media Content**
US 12/201,920

---

## Skills

**Languages**: C, Go, Java, Python, JavaScript
**Systems**: Linux, FreeBSD, Windows, Android, iOS
**Networking**: TCP/IP Stack, HTTP, TLS/SSL, PKI
**Runtime/Libraries**: QuickJS, libcurl, Django, DirectShow, FFmpeg
**Infrastructure**: Kubernetes, Docker, NGINX
**Tools**: Git, CI/CD
