# Web 서버(Apache HTTP Server) 설정 가이드

> **적용 범위**: Apache HTTP Server 2.4.x (Event / Worker / Prefork MPM)
> **기준 인프라**: 4 Core CPU / 4~32 GB RAM
> **원천**: `source/apache-tuning-guide.md` V3.1 + 도메인 불변량. 본 파일이 정본. 원본과 충돌 시 본 파일 우선.

---

## 0. 적용 전제

- **방화벽 TCP 30~60min(최단 30min=1,800초 기준)**: 모든 타임아웃의 최상위. 역방향 프록시 커넥션 포함 30min 미만 설계.
- **70% Ceiling**: Apache 자체는 해당 없음. 단, `mod_dbd`/`mod_php` 등 DB 연결 모듈 내장 시 WAS와 합산하여 `Sum(커넥션) ≤ DB max_connections × 0.7`.
- **타임아웃 캐스케이드(역방향 프록시)**: `ProxyPass ttl < 1800(방화벽 30min)`. 동일 호스트 loopback은 방화벽 경유 안 함(§8.2).

---

## 1. OS 커널 설정

### 1.1 공통 sysctl (WAS/DB 정본과 정렬)

```ini
# /etc/sysctl.d/99-infra-common.conf
fs.file-max = 2097152
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 5
```

### 1.2 Web 서버 전용 sysctl

```ini
# /etc/sysctl.d/99-web-tuning.conf
vm.swappiness = 1
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_tw_reuse = 1
# 기본 32768 65535 (WAS/DB 정본 정렬, 혼합 용도 안전).
# Web 단독 다중 인스턴스/대형 역방향 프록시 포트 고갈 시에만 1024 65535 확장 (§8.2)
net.ipv4.ip_local_port_range = 32768 65535
```

| 파라미터 | 값 | 비고 |
|:---|:---|:---|
| fs.file-max | 2,097,152 | 시스템 전체 FD 상한 |
| net.core.somaxconn | 4,096 | `min(somaxconn, ListenBacklog)`가 실제 accept 큐 |
| tcp_keepalive_time | 300 (5min) | 기본 7,200s 단축 |
| vm.swappiness | 1 | page cache 우선. RAM 헤드룸 확보 전제 |
| tcp_fin_timeout | 15 | TIME_WAIT 단축 |
| tcp_tw_reuse | 1 | 아웃바운드 TIME_WAIT 재사용. NAT/L4 환경 주의(§8.2) |
| ip_local_port_range | 기본 32768~65535 / 확장 1024~65535 | 기본값이 혼합 용도 안전. 확장은 Web 단독 다중 인스턴스만 |

### 1.3 systemd Limits (필수)

```ini
# /etc/systemd/system/httpd.service.d/override.conf
# Ubuntu/Debian: apache2.service.d/override.conf
[Service]
LimitNOFILE=1048576
LimitNPROC=65536
```

> `limits.conf`는 PAM 세션만 적용. systemd 데몬은 위 drop-in 없으면 기본 1024로 동작.

### 1.4 적용

```bash
sysctl --load /etc/sysctl.d/99-infra-common.conf
sysctl --load /etc/sysctl.d/99-web-tuning.conf
systemctl daemon-reload
systemctl restart httpd   # Ubuntu: apache2
```

---

## 2. MPM 선택

| MPM | 적용 조건 | 권장 |
|:---|:---|:---|
| **Event** | 기본. Apache 2.4 정식 | **표준** |
| Worker | Event 호환성 이슈 시 | 레거시 |
| Prefork | `mod_php` 등 스레드-unsafe 모듈 필수 시만 | **비권장**(8 GB+ 비효율) |

```apache
# /etc/httpd/conf.modules.d/00-mpm.conf  (Ubuntu: /etc/apache2/mods-enabled/mpm_event.load)
LoadModule mpm_event_module modules/mod_mpm_event.so
```

### 핵심 공식

```
가용_RAM            = Total_RAM × 0.9
MaxRequestWorkers   = 가용_RAM / 프로세스당_비용
ServerLimit         ≤ 200 (4 Core 제약)
ThreadsPerChild     ≤ 128 (안전) / > 128 (극대화, 순수 프록시 한정)
MaxSpareThreads     = min(MinSpare × 3, 2000)
Thrashing 방지      : (MaxSpare − MinSpare) > ThreadsPerChild   [필수]
Event 총 연결 한계  = (AsyncRequestWorkerFactor + 1) × MaxRequestWorkers
Scoreboard 검증     : ServerLimit × ThreadLimit ≥ 총 연결 한계  [필수]
```

- **ThreadsPerChild ≤ 128**: 안전(외부 모듈/DB 드라이버 환경). **> 128**: 순수 리버스 프록시/SSL 종단만.
- **AsyncRequestWorkerFactor = 3**: Event 기본값.
- **KeepAliveTimeout = 3**: HTTP/1.1 평균 요청 간격 하한선.

---

## 3. MPM Event 표준 설정 (AVG_RSS=8MB, ThreadStackSize=1MB, AsyncFactor=3)

> `MaxConnectionsPerChild = 0` (Event는 Listener 안정성 위해 프로세스 교체 최소화). 누수 모니터링 필수(§10).

### 3.1 전략 A: 자원 극대화 (순수 리버스 프록시 / SSL 종단)

| RAM | ServerLimit | ThreadLimit | ThreadsPerChild | MaxRequestWorkers | 총 연결 한계 | Scoreboard | StartServers | MinSpare | MaxSpare |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| 4 GB | 102 | 128 | 28 | 2,856 | 11,424 | 13,056 | 8 | 224 | 672 |
| 8 GB | 153 | 192 | 40 | 6,120 | 24,480 | 29,376 | 10 | 400 | 1,200 |
| 16 GB | 150 | 384 | 90 | 13,500 | 54,000 | 57,600 | 10 | 900 | 2,000 (cap) |
| 32 GB | 150 | 768 | 188 | 28,200 | 112,800 | 115,200 | 8 | 1,504 | 2,000 (cap) |

### 3.2 전략 B: 안전 (DB 드라이버 / mod_security / 외부 라이브러리 사용) — 권장 기본

| RAM | ServerLimit | ThreadLimit | ThreadsPerChild | MaxRequestWorkers | 총 연결 한계 | Scoreboard | StartServers | MinSpare | MaxSpare |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| 4 GB | 102 | 128 | 28 | 2,856 | 11,424 | 13,056 | 8 | 224 | 672 |
| 8 GB | 153 | 192 | 40 | 6,120 | 24,480 | 29,376 | 10 | 400 | 1,200 |
| 16 GB | 150 | 256 | 64 | 9,600 | 38,400 | 38,400 | 10 | 640 | 1,920 |
| 32 GB | 200 | 512 | 128 | 25,600 | 102,400 | 102,400 | 13 | 1,664 | 2,000 (cap) |

### 3.3 설정 예시 (8 GB, 전략 B — 표준 권장)

```apache
# /etc/httpd/conf.d/mpm.conf
ServerLimit              153
ThreadLimit              192
ThreadsPerChild          40
ThreadStackSize          1048576
MaxRequestWorkers        6120
AsyncRequestWorkerFactor 3
StartServers             10
MinSpareThreads          400
MaxSpareThreads          1200
MaxConnectionsPerChild   0
KeepAliveTimeout         3
ListenBacklog            1024
```

---

## 4. MPM Worker / Prefork (참고)

### 4.1 MPM Worker — 전략 B (레거시 환경)

> `MaxConnectionsPerChild = 10000` (미세 누수 방지). 8 GB+에서 Event 전환 권장.

| RAM | ServerLimit | ThreadLimit | ThreadsPerChild | MaxRequestWorkers | StartServers | MinSpare | MaxSpare |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| 4 GB | 102 | 64 | 28 | 2,856 | 8 | 224 | 672 |
| 8 GB | 153 | 64 | 40 | 6,120 | 8 | 320 | 960 |
| 16 GB | 150 | 192 | 64 | 9,600 | 10 | 640 | 1,920 |
| 32 GB | 200 | 192 | 128 | 25,600 | 12 | 1,536 | 2,000 (cap) |

### 4.2 MPM Prefork

> **비권장**. 4 Core에서 ServerLimit=200 cap, RAM 무관. `mod_php` 등 스레드-unsafe 모듈 필수 환경만 허용.

| RAM | ServerLimit | MaxRequestWorkers | StartServers | MinSpare | MaxSpare | MaxConnPerChild | KeepAliveTimeout | ListenBacklog |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| 공통 | 200 (cap) | 200 | 8 | 8 | 16 | 10000 | 3 | 1024 |

---

## 5. 공통 지시어 (MPM 무관)

| 파라미터 | 값 | 비고 |
|:---|:---|:---|
| ThreadStackSize | 1048576 (1MB) | Worker/Event. 플랫폼 기본(보통 8MB) 과다. **restart 필요** |
| KeepAliveTimeout | 3 | HTTP/1.1 평균 요청 간격 하한선. **reload 가능** |
| ListenBacklog | 1024 | `min(somaxconn, ListenBacklog)`가 실제 큐. somaxconn(4096)과 정렬. **restart 필요** |
| MaxConnectionsPerChild | Event=0 / Worker·Prefork=10000 | Event는 Listener 안정성 위해 0. 누수 모니터링 필수 |
| KeepAlive | On | HTTP/1.1 기본 |
| MaxKeepAliveRequests | 100 | 0(무제한) 금지 |

---

## 6. 다중 인스턴스 환경 (4 Parent Processes)

> 1대 물리 서버(4 Core / 32 GB+)에서 4개 Apache 인스턴스. TPC=128 고정, ServerLimit ≤ 50 (CPU cap).

### 6.1 자원 분할

| Total RAM | 인스턴스당 RAM | 실제 ServerLimit | 비고 |
|:---:|:---:|:---:|:---|
| 4 GB | 921 MB | 6 | RAM 한계 |
| 8 GB | 1,843 MB | 13 | RAM 한계 |
| 16 GB | 3,686 MB | 27 | RAM 한계 |
| 32 GB | 7,373 MB | 50 | CPU cap 도달 |

### 6.2 MPM Event — 인스턴스당 설정 (TPC=128, AsyncFactor=3, TL=512, MaxSpare=800 cap)

| RAM | SL | MRW | 인스턴스당 총 연결 | Scoreboard | 4인스턴스 합산 |
|:---:|:---:|:---:|:---:|:---:|:---|
| 4 GB | 6 | 768 | 3,072 | 3,072 | 12,288 |
| 8 GB | 13 | 1,664 | 6,656 | 6,656 | 26,624 |
| 16 GB | 27 | 3,456 | 13,824 | 13,824 | 55,296 (Port 고갈 임박) |
| 32 GB | 50 | 6,400 | 25,600 | 25,600 | **102,400 (Port 고갈, IP Aliasing 필수)** |

> 단일 IP 최대 포트 = 65,535. 32 GB 4인스턴스 합산 102,400 초과 → **IP Aliasing 필수**(`eth0:1` ~ `eth0:4`).

### 6.3 인스턴스별 충돌 방지 설정

```apache
# /etc/httpd/conf/httpd-instance1.conf (인스턴스 1)
Listen 10.0.0.101:80
PidFile /var/run/httpd/httpd-instance1.pid
ErrorLog /var/log/httpd/instance1-error.log
CustomLog /var/log/httpd/instance1-access.log combined
Mutex file:/var/run/httpd/instance1-lock default
ScoreBoardFile /var/run/httpd/instance1-scoreboard
```

| 체크 항목 | 필수 |
|:---|:---:|
| Listen IP+Port 분리 (IP Aliasing) | 필수 |
| PidFile / ErrorLog / CustomLog / Mutex 분리 | 필수 |
| ScoreBoardFile 분리 | 권장 |
| systemd 유닛 분리 (`httpd@instance1.service`) | 권장 |
| ip_local_port_range = 1024 65535 (32 GB 필수) | 조건부 |

---

## 7. reload vs restart

| 적용 | 파라미터 |
|:---|:---|
| **restart 필요** | ServerLimit, ThreadLimit, ThreadStackSize, ListenBacklog, MPM 전환 |
| **reload 가능** | MaxRequestWorkers(TL 이내), ThreadsPerChild(TL 이내), StartServers, Min/MaxSpare, MaxConnectionsPerChild, KeepAliveTimeout, AsyncRequestWorkerFactor |

```bash
sudo systemctl reload  httpd   # Ubuntu: apache2
sudo systemctl restart httpd   # 구조 변경 시만
```

---

## 8. 주의사항

### 8.1 원본 V3.1 대비 정정값 (본 규정 적용값)

| 항목 | 원본 V3.1 | 본 규정 | 사유 |
|:---|:---|:---|:---|
| fs.file-max | 100000 | **2,097,152** | WAS/DB 정본 정렬. 과소 |
| somaxconn | 2048 | **4,096** | tcp_max_syn_backlog와 세트 |
| Thrashing Guard | `> TPC×2` | **`> TPC`** | 원본 공식과 다중인스턴스 절 모순 |
| systemd LimitNOFILE/NPROC | 누락 | **필수 추가** | drop-in 없으면 데몬 기본 1024 |
| ListenBacklog ↔ somaxconn | 별도 언급 없음 | **정렬 의무** | `min()`으로 커널이 자름 |
| ThreadStackSize | "기본 8MB 낭비" 단정 | **플랫폼 의존(보통 8MB)** | 빌드별 상이, `httpd -V`/`ulimit -s` 실측 |
| MaxConnectionsPerChild=0 (Event) | 권장만 | **+ 누수 모니터링 의무** | 프로세스 재활용 없음 |

### 8.2 혼합 용도 서버 설정

#### Web + WAS 동일 호스트

| 항목 | 설정 |
|:---|:---|
| Apache 가용 RAM | `Total_RAM × 0.9 − WAS_JVM_Heap − WAS_Metaspace − WAS_Native` (WAS Heap 산정은 `reports/final/was.md` §2.2) |
| ServerLimit | 전용 서버 기준 **× 0.7** (CPU 경합 완화) |
| 역방향 프록시 | `ProxyPass http://127.0.0.1:8080/` (loopback, 방화벽 미경유) |
| ip_local_port_range | **32768~65535 유지** (WAS 8080/9080 포트 충돌 방지) |
| tcp_tw_reuse | 1 (loopback 포트 고갈 방지) |
| systemd Limits | httpd.service / tomcat.service 각각 독립 LimitNOFILE 설정 |

#### Web + DB 동일 호스트 (비권장)

| 항목 | 설정 |
|:---|:---|
| DB 커넥션 합산 | `Sum(Apache 커넥션 + WAS maxPoolSize) ≤ DB max_connections × 0.7` (`mod_dbd`/임베디드 드라이버 포함) |
| DB 메모리 | WiredTiger `cacheSizeGB` / PG `shared_buffers` 명시 할당 후 남은 RAM에서 Apache MRW 산정 |
| I/O 스케줄러 | `mq-deadline` 고정 (DB 우선) |
| 모니터링 | 메트릭 라벨로 역할 분리 (장애 원인 구분) |

#### Web + 캐시(Redis/Memcached) 동일 호스트

- 캐시 RSS를 Apache 가용 RAM에서 **선차감**. Redis `maxmemory` 명시 필수.
- Redis `tcp-backlog 511` ↔ Apache `ListenBacklog 1024` 정렬 권장.

#### L4 / NAT 환경

- `tcp_tw_reuse=1`: 아웃바운드(역방향 프록시) 안전. NAT 모드 L4에서는 5-tuple 충돌 가능 → L4 health check 주기 + `MaxConnectionsPerChild`로 보조.
- 헬스체크 전용 엔드포인트(`/healthz`) + `CustomLog` 제외: `SetEnvIf Request_URI "^/healthz$" dontlog`
- 클라이언트 IP 신뢰: `RemoteIPHeader X-Forwarded-For` + `RemoteIPInternalProxy <L4_IP>`

### 8.3 역방향 프록시(ProxyPass) 타임아웃

```apache
ProxyPass        /api/  http://was-backend:8080/api/  ttl=1500 acquire=3000 timeout=60 keepalive=On retry=5
ProxyPassReverse /api/  http://was-backend:8080/api/
```

| 파라미터 | 권장값 | 비고 |
|:---|:---|:---|
| ttl | 1500 (25min) | 방화벽 30min 미만 |
| acquire | 3000 (3s) | 커넥션 획득 대기. Fail-Fast |
| timeout | 60 (60s) | WAS `connectionTimeout(20s)` + 여유 |
| keepalive | On | 풀 재사용. Off 시 포트 고갈 |
| retry | 5 (5s) | 기본 60s 너무 김. 백엔드 장애 시 빠른 복구 |

### 8.4 보안 기본 설정

```apache
# /etc/httpd/conf.d/security.conf
ServerTokens      Prod
ServerSignature   Off
TraceEnable       Off

<Directory "/">
    Options -Indexes
    AllowOverride None
    Require all denied
</Directory>

<Directory "/var/www/html">
    Require all granted
</Directory>

<FilesMatch "^\.ht">
    Require all denied
</FilesMatch>
```

> **사전 확인**: `TraceEnable Off` 적용 전 L4/모니터링 헬스체크 방식 확인(TRACE 사용 시 OPTIONS/GET으로 변경 선행). SSL은 `Protocols all -SSLv3 -TLSv1 -TLSv1.1`, 레거시 클라이언트 사전 파악 필수.

---

## 9. 검증 체크리스트

- [ ] MPM = Event (Worker/Prefork 잔존 시 전환 사유 문서화)
- [ ] `ServerLimit × ThreadLimit ≥ Event 총 연결 한계` (Scoreboard 검증)
- [ ] `(MaxSpare − MinSpare) > ThreadsPerChild` (Thrashing 방지)
- [ ] `MaxSpare ≤ 2,000` (4 Core cap)
- [ ] `ServerLimit ≤ 200` (단일) / `≤ 50 × N` (다중)
- [ ] `ListenBacklog ≤ net.core.somaxconn` (accept 큐 정렬)
- [ ] `ThreadStackSize = 1048576` (1MB)
- [ ] `MaxConnectionsPerChild`: Event=0, Worker/Prefork=10000
- [ ] `KeepAliveTimeout = 3`
- [ ] systemd `LimitNOFILE=1048576` / `LimitNPROC=65536` drop-in 설정
- [ ] `fs.file-max = 2097152`, `somaxconn = 4096`
- [ ] `tcp_tw_reuse = 1` + `ip_local_port_range` 기본 32768~65535 (Web 단독만 1024~65535)
- [ ] 다중 인스턴스: Listen IP / PidFile / ErrorLog / Mutex 분리
- [ ] 다중 인스턴스 32 GB: IP Aliasing 적용
- [ ] 역방향 프록시: `ProxyPass ttl < 1800` (방화벽 30min 미만)
- [ ] 혼합(Web+WAS): WAS Heap 선차감 후 MRW 산정, ip_local_port_range 32768~65535 유지
- [ ] 혼합(Web+DB): 70% Ceiling 합산 검증
- [ ] 보안: `ServerTokens Prod` / `ServerSignature Off` / `TraceEnable Off` / `Options -Indexes`

---

## 10. 모니터링 체크리스트

| 항목 | 조회 | 경고 | 위험 | 조치 |
|:---|:---|:---|:---|:---|
| Worker 사용률 | `mod_status` | BusyWorkers > 70% MRW | > 90% MRW | MRW/TPC 상향 검토 |
| accept 큐 적체 | `ss -ltn` (Recv-Q) | > 100 지속 | > 1000 | somaxconn/ListenBacklog 점검 |
| TIME_WAIT 누적 | `ss -tan \| grep TIME-WAIT \| wc -l` | > 10,000 | > 30,000 | tcp_tw_reuse, MaxConnPerChild 점검 |
| 포트 고갈 | ip_local_port_range 대비 ESTABLISHED | 가용 < 5,000 | < 1,000 | IP Aliasing, `keepalive=On` |
| 프로세스 RSS | `ps -o pid,rss,cmd -C httpd` | > AVG_RSS × 1.5 | > AVG_RSS × 2 | 누수, MaxConnPerChild 축소(Event 0 → 10000 전환) |
| 백엔드 장애 | `mod_status` proxy worker | ERR 발생 | ERR 지속 | WAS 건강도, retry=5 적용 |
| 역방향 프록시 5xx | access log | > 1% | > 5% | ProxyPass timeout/ttl 점검 |

> `MaxConnectionsPerChild=0` (Event) 환경에서는 **프로세스 RSS 추적 필수**. 프로세스 재활용이 없으므로 누수 감지 유일 수단.
