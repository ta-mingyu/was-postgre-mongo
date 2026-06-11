# Apache 성능 튜닝 가이드 V3.1 (요약)

> **환경**: 4 Core CPU / 4~32GB RAM | **기준**: AVG_RSS=8MB(Worker/Event), 12MB(Prefork), ThreadStackSize=1MB, KeepAliveTimeout=3

---

## 1. 핵심 원칙

- **RAM 90% 활용**: 가용_RAM = Total_RAM × 0.9 (10%는 OS/커널 예약)
- **ServerLimit ≤ 200**: 4 Core CPU 컨텍스트 스위칭 한계
- **TPC ≤ 128 (안전)**: 외부 모듈(DB드라이버, mod_security) 환경은 락 경합 방지
- **TPC > 128 (극대화)**: 순수 리버스 프록시/SSL 종단 환경만 허용
- **Event가 기본 선택**: KeepAlive 연결을 Listener Thread가 관리 → Worker 대비 2~5배 연결 수용
- **KeepAliveTimeout 3**: HTTP/1.1 평균 요청 간격(2~4초)의 하한선인 3초를 적용하여, 유휴 연결을 더 빠르게 정리하고 Worker/Thread를 즉시 회수함으로써 동시 접속 처리 효율을 극대화함. 공격적인 상향 설정(높은 MRW/TPC)과 궁합이 최적.

---

## 2. 핵심 공식

```
가용_RAM = Total_RAM × 0.9
MaxRequestWorkers = 가용_RAM / 프로세스당_비용
ServerLimit ≤ 200 (4 Core 제약)
MaxSpareThreads = min(MinSpare × 3, 2000) (4 Core 캡, MaxSpare ≥ MinSpare 보장 필수)
Thrashing 방지: (MaxSpare - MinSpare) > ThreadsPerChild 반드시 충족
Event ThreadLimit = TPC × (1 + AsyncRequestWorkerFactor), 깔끔한 수로 올림
Event 총 연결 한계 = (AsyncFactor + 1) × MRW
Scoreboard 검증: SL × TL ≥ 총 연결 한계
```

---

## 3. MPM Prefork (AVG_RSS=12MB)

> 1프로세스=1요청. RAM이 많아도 4 Core에서 200프로세스가 한계. **8GB 이상은 Worker/Event 전환 필수.**

| RAM | ServerLimit | MaxRequestWorkers | StartServers | MinSpareServers | MaxSpareServers | MaxConnPerChild | KeepAliveTimeout | ListenBacklog |
|-----|-------------|-------------------|-------------|----------------|----------------|----------------|-----------------|-------------|
| 4GB | 200 (cap) | 200 | 8 | 8 | 16 | 10000 | 3 | 1024 |
| 8GB | 200 (cap) | 200 | 8 | 8 | 16 | 10000 | 3 | 1024 |
| 16GB | 200 (cap) | 200 | 10 | 10 | 20 | 10000 | 3 | 1024 |
| 32GB | 200 (cap) | 200 | 10 | 10 | 20 | 10000 | 3 | 1024 |

**설정 예시 (8GB):**

```apache
ServerLimit            200
MaxRequestWorkers      200
StartServers           8
MinSpareServers        8
MaxSpareServers        16
MaxConnectionsPerChild 10000
KeepAliveTimeout       3
ListenBacklog          1024
```

---

## 4. MPM Worker (AVG_RSS=8MB, ThreadStackSize=1MB)

### 전략 A: 자원 극대화 (순수 리버스 프록시 / SSL 종단)

| RAM | ServerLimit | ThreadLimit | ThreadsPerChild | MaxRequestWorkers | StartServers | MinSpare | MaxSpare | MaxConnPerChild |
|-----|-------------|-------------|-----------------|-------------------|-------------|----------|----------|----------------|
| 4GB | 102 | 64 | 28 | 2,856 | 8 | 224 | 672 | 10000 |
| 8GB | 153 | 64 | 40 | 6,120 | 8 | 320 | 960 | 10000 |
| 16GB | 150 | 256 | 90 | 13,500 | 10 | 900 | 2,000 (cap) | 10000 |
| 32GB | 150 | 384 | 188 | 28,200 | 8 | 1,504 | 2,000 (cap) | 10000 |

### 전략 B: 안전 (DB 드라이버, mod_security, 외부 라이브러리 사용)

| RAM | ServerLimit | ThreadLimit | ThreadsPerChild | MaxRequestWorkers | StartServers | MinSpare | MaxSpare | MaxConnPerChild |
|-----|-------------|-------------|-----------------|-------------------|-------------|----------|----------|----------------|
| 4GB | 102 | 64 | 28 | 2,856 | 8 | 224 | 672 | 10000 |
| 8GB | 153 | 64 | 40 | 6,120 | 8 | 320 | 960 | 10000 |
| 16GB | 150 | 192 | 64 | 9,600 | 10 | 640 | 1,920 | 10000 |
| 32GB | 200 | 192 | 128 | 25,600 | 12 | 1,536 | 2,000 (cap) | 10000 |

**설정 예시 (8GB, 전략 B):**

```apache
ServerLimit            153
ThreadLimit            64
ThreadsPerChild        40
ThreadStackSize        1048576
MaxRequestWorkers      6120
StartServers           8
MinSpareThreads        320
MaxSpareThreads        960
MaxConnectionsPerChild 10000
KeepAliveTimeout       3
ListenBacklog          1024
```

---

## 5. MPM Event (AVG_RSS=8MB, ThreadStackSize=1MB, AsyncFactor=3)

> Worker + Listener Thread가 KeepAlive 연결을 대신 관리 → 동일 스레드로 2~5배 더 많은 연결 수용.
> MaxConnectionsPerChild=0 (Event는 프로세스 churning 최소화해야 Listener 안정).

### 전략 A: 자원 극대화 (순수 리버스 프록시 / SSL 종단)

| RAM | ServerLimit | ThreadLimit | ThreadsPerChild | MRW | AsyncFactor | 총 연결 한계 | Scoreboard(SL×TL) | StartServers | MinSpare | MaxSpare | MaxConnPerChild |
|-----|-------------|-------------|-----------------|-----|-------------|-------------|-------------------|-------------|----------|----------|----------------|
| 4GB | 102 | 128 | 28 | 2,856 | 3 | 11,424 | 13,056 ✓ | 8 | 224 | 672 | 0 |
| 8GB | 153 | 192 | 40 | 6,120 | 3 | 24,480 | 29,376 ✓ | 10 | 400 | 1,200 | 0 |
| 16GB | 150 | 384 | 90 | 13,500 | 3 | 54,000 | 57,600 ✓ | 10 | 900 | 2,000 (cap) | 0 |
| 32GB | 150 | 768 | 188 | 28,200 | 3 | 112,800 | 115,200 ✓ | 8 | 1,504 | 2,000 (cap) | 0 |

### 전략 B: 안전 (DB 드라이버, mod_security, 외부 라이브러리 사용)

| RAM | ServerLimit | ThreadLimit | ThreadsPerChild | MRW | AsyncFactor | 총 연결 한계 | Scoreboard(SL×TL) | StartServers | MinSpare | MaxSpare | MaxConnPerChild |
|-----|-------------|-------------|-----------------|-----|-------------|-------------|-------------------|-------------|----------|----------|----------------|
| 4GB | 102 | 128 | 28 | 2,856 | 3 | 11,424 | 13,056 ✓ | 8 | 224 | 672 | 0 |
| 8GB | 153 | 192 | 40 | 6,120 | 3 | 24,480 | 29,376 ✓ | 10 | 400 | 1,200 | 0 |
| 16GB | 150 | 256 | 64 | 9,600 | 3 | 38,400 | 38,400 ✓ | 10 | 640 | 1,920 | 0 |
| 32GB | 200 | 512 | 128 | 25,600 | 3 | 102,400 | 102,400 ✓ | 13 | 1,664 | 2,000 (cap) | 0 |

**설정 예시 (8GB, 전략 B):**

```apache
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

## 6. 공통 설정

| 파라미터 | Worker | Event | Prefork | 비고 |
|---------|--------|-------|---------|------|
| ThreadStackSize | 1048576 (1MB) | 1048576 (1MB) | 미사용 | 기본값 8MB는 낭비. **restart 필요** |
| KeepAliveTimeout | **3** | **3** | **3** | HTTP/1.1 평균 요청 간격(2~4초) 하한선. 유휴 연결 빠른 정리 → Worker/Thread 즉시 회수 → 동시 접속 효율 극대화. 공격적 상향 설정과 최적 궁합. **reload 가능** |
| ListenBacklog | 1024 | 1024 | 1024 | 기본값 511. 마이크로 버스트 방지. **restart 필요** |
| MaxConnectionsPerChild | **10000** | **0** | **10000** | Event는 리스너 안정성을 위해 프로세스 교체 최소화(0) 권장. Worker/Prefork은 미세 누수 방지용 10000. **reload 가능** |

---

## 7. OS 튜닝

```bash
sudo tee /etc/sysctl.d/99-apache-tuning.conf <<'EOF'
vm.swappiness = 10
fs.file-max = 100000
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_tw_reuse = 1
net.core.somaxconn = 2048
EOF

sudo sysctl -p /etc/sysctl.d/99-apache-tuning.conf
```

---

## 8. reload vs restart

| 분류 | 파라미터 | 적용 방법 |
|------|---------|----------|
| **restart 필요** | ServerLimit, ThreadLimit, ThreadStackSize, ListenBacklog | 프로세스/스레드 구조 변경 → 완전 재시작 |
| **reload 가능** | MaxRequestWorkers, ThreadsPerChild (TL 이내), StartServers, MinSpare, MaxSpare, MaxConnectionsPerChild, KeepAliveTimeout, AsyncRequestWorkerFactor | 기존 연결 유지하면서 새 child에 적용 |

```bash
# Ubuntu/Debian
sudo systemctl restart apache2
sudo systemctl reload  apache2
# RHEL/CentOS
sudo systemctl restart httpd
sudo systemctl reload  httpd
```

---

## 9. 다중 인스턴스 환경 설정 (4 Parent Processes)

> 1대의 물리 서버(4 Core / 32GB+ RAM)에서 4개의 독립된 Apache 인스턴스가 각각 다른 서비스를 담당.
> RAM은 풍족하므로 **TPC=128을 고정 할당**하여 메모리를 최대한 활용, CPU(SL≤50)만 엄격 제한.

### 자원 분할 원칙

```
가용_RAM_인스턴스 = (Total_RAM × 0.9) / 4
ServerLimit_인스턴스 ≤ 50 (4 Core 총합 ≤ 200 제약 유지)
ThreadsPerChild = 128 (RAM 풍족 → 안전 상한 고정 할당)
MaxSpareThreads ≤ 800 (4인스턴스 합산 유휴 스레드 ~3,200 수준 제한)
```

| Total RAM | 전체 가용 RAM | 인스턴스당 RAM | SL (RAM 한계) | SL (CPU cap) | 실제 SL |
|-----------|-------------|---------------|--------------|-------------|--------|
| 4 GB | 3,686 MB | 921 MB | 6 | 50 | **6** (RAM 한계) |
| 8 GB | 7,373 MB | 1,843 MB | 13 | 50 | **13** (RAM 한계) |
| 16 GB | 14,746 MB | 3,686 MB | 27 | 50 | **27** (RAM 한계) |
| 32 GB | 29,491 MB | 7,373 MB | 54 | 50 | **50** (CPU cap) |

> 16GB 이하에서는 RAM이 TPC=128×SL을 감당하지 못해 SL이 50 미만으로 제한됨.
> **32GB에서만 CPU cap(50)에 도달하며, 이때 인스턴스당 RAM 사용률 92%로 여유 확보.**

### MPM Prefork — 인스턴스당 설정 (AVG_RSS=12MB)

> 4개 인스턴스 × 50프로세스 = 총 200프로세스. **Prefork는 이 환경에서도 비효율적 → Worker/Event 권장.**

| RAM | ServerLimit | MaxRequestWorkers | StartServers | MinSpareServers | MaxSpareServers | MaxConnPerChild | KeepAliveTimeout | ListenBacklog |
|-----|-------------|-------------------|-------------|----------------|----------------|----------------|-----------------|-------------|
| 4 GB | 50 | 50 | 5 | 5 | 10 | 10000 | 3 | 1024 |
| 8 GB | 50 | 50 | 5 | 5 | 10 | 10000 | 3 | 1024 |
| 16 GB | 50 | 50 | 5 | 5 | 10 | 10000 | 3 | 1024 |
| 32 GB | 50 | 50 | 5 | 5 | 10 | 10000 | 3 | 1024 |

### MPM Worker — 인스턴스당 설정 (AVG_RSS=8MB, TPC=128 고정)

| RAM | ServerLimit | ThreadLimit | ThreadsPerChild | MaxRequestWorkers | StartServers | MinSpare | MaxSpare | MaxConnPerChild |
|-----|-------------|-------------|-----------------|-------------------|-------------|----------|----------|----------------|
| 4 GB | 6 | 192 | 128 | 768 | 4 | 512 | **800** (cap) | 10000 |
| 8 GB | 13 | 192 | 128 | 1,664 | 4 | 512 | **800** (cap) | 10000 |
| 16 GB | 27 | 192 | 128 | 3,456 | 4 | 512 | **800** (cap) | 10000 |
| 32 GB | 50 | 192 | 128 | 6,400 | 4 | 512 | **800** (cap) | 10000 |

> Thrashing Guard: buffer = 800 - 512 = 288 > TPC×2(256) ✓

### MPM Event — 인스턴스당 설정 (AVG_RSS=8MB, TPC=128, AsyncFactor=3, TL=TPC×4)

| RAM | SL | TL | TPC | MRW | AsyncFactor | 인스턴스당 총 연결 | Scoreboard(SL×TL) | SS | MinSpare | MaxSpare | MaxConnPerChild |
|-----|----|----|-----|-----|-------------|------------------|-------------------|-----|----------|----------|----------------|
| 4 GB | 6 | 512 | 128 | 768 | 3 | 3,072 | 3,072 ✓ | 4 | 512 | **800** (cap) | 0 |
| 8 GB | 13 | 512 | 128 | 1,664 | 3 | 6,656 | 6,656 ✓ | 4 | 512 | **800** (cap) | 0 |
| 16 GB | 27 | 512 | 128 | 3,456 | 3 | 13,824 | 13,824 ✓ | 4 | 512 | **800** (cap) | 0 |
| 32 GB | 50 | 512 | 128 | 6,400 | 3 | 25,600 | 25,600 ✓ | 4 | 512 | **800** (cap) | 0 |

### ⚠️ 네트워크 포트 고갈(Port Exhaustion) 경고

```
단일 IP의 최대 포트 수 = 65,535 (TCP 명세)
32GB 4인스턴스 총 연결 = 4 × 25,600 = 102,400 > 65,535 ❌
16GB 4인스턴스 총 연결 = 4 × 13,824 = 55,296 < 65,535 (여유 적음, 아웃바운드 고려 시 위험)
```

**해결책 (32GB 환경 필수):**

| 방안 | 내용 | 적용 |
|------|------|------|
| **IP Aliasing** | 인스턴스별 독립 IP 부여 → 각 IP마다 65,535 포트 확보 | `eth0:1 10.0.0.101`, `eth0:2 10.0.0.102`, ... |
| **tcp_tw_reuse** | TIME_WAIT 소켓 즉시 재사용 → 포트 회수 가속 | `sysctl -w net.ipv4.tcp_tw_reuse=1` (이미 OS 튜닝에 포함) |
| **ip_local_port_range 확장** | 임시 포트 범위 최대화 | `sysctl -w net.ipv4.ip_local_port_range="1024 65535"` |

**IP Aliasing 설정 예시:**

```bash
# 인스턴스별 Listen IP 분리
sudo ip addr add 10.0.0.101/24 dev eth0
sudo ip addr add 10.0.0.102/24 dev eth0
sudo ip addr add 10.0.0.103/24 dev eth0
sudo ip addr add 10.0.0.104/24 dev eth0

# 각 인스턴스 설정 파일에
# Listen 10.0.0.101:80
# Listen 10.0.0.102:80
# ...
```

### 단일 인스턴스 대비 처리량 비교 (Event, 전략 B)

| RAM | 단일 인스턴스 MRW | 단일 총 연결 | **4인스턴스 MRW 합** | **4인스턴스 총 연결** | **향상률** |
|-----|-----------------|-------------|-------------------|-------------------|----------|
| 16 GB | 9,600 | 38,400 | **13,824** | **55,296** | **+44%** |
| 32 GB | 25,600 | 102,400 | **25,600** | **102,400** | 동일 (분산만) |

> **16GB에서 비약적 향상**: 단일 인스턴스는 TPC=64(SL=150)였으나, 4인스턴스는 각각 TPC=128(SL=27)로 총합 SL=108 → 높은 TPC가 더 많은 MRW를 산출.
> **32GB에서는 동일**: 단일 SL=200, TPC=128 vs 4×(SL=50, TPC=128) = 총합 동일. 대신 장애 격리 이점.

**설정 예시 — 32GB, Event, 인스턴스당:**

```apache
ServerLimit              50
ThreadLimit              512
ThreadsPerChild          128
ThreadStackSize          1048576
MaxRequestWorkers        6400
AsyncRequestWorkerFactor 3
StartServers             4
MinSpareThreads          512
MaxSpareThreads          800
MaxConnectionsPerChild   0
KeepAliveTimeout         3
ListenBacklog            1024
```

### 인스턴스 간 충돌 방지 체크리스트

각 인스턴스는 독립 설정 파일과 포트를 사용해야 함:

```apache
# 인스턴스 1: httpd-instance1.conf
Listen 10.0.0.101:80
PidFile /var/run/httpd-instance1.pid
ErrorLog /var/log/httpd/instance1-error.log
CustomLog /var/log/httpd/instance1-access.log combined

# 인스턴스 2: httpd-instance2.conf
Listen 10.0.0.102:80
PidFile /var/run/httpd-instance2.pid
ErrorLog /var/log/httpd/instance2-error.log
CustomLog /var/log/httpd/instance2-access.log combined

# 인스턴스 3, 4: 동일 패턴 (10.0.0.103, 10.0.0.104)
```

```bash
# 개별 인스턴스 시작/중지
httpd -f /etc/httpd/conf/httpd-instance1.conf
httpd -f /etc/httpd/conf/httpd-instance2.conf

# Ubuntu (apache2ctl)
apache2ctl -f /etc/apache2/httpd-instance1.conf -k start
apache2ctl -f /etc/apache2/httpd-instance2.conf -k start
```

| 체크 항목 | 필수 | 비고 |
|----------|------|------|
| **Listen IP+Port 분리** | ✅ | IP Aliasing으로 인스턴스별 독립 IP 부여 (Port Exhaustion 방지) |
| **PidFile 분리** | ✅ | 시그널 충돌 방지. `/var/run/httpd-inst{N}.pid` |
| **ErrorLog / CustomLog 분리** | ✅ | 장애 추적 시 인스턴스 식별 가능 |
| **LockFile / Mutex 분리** | ✅ | `Mutex file:/var/run/httpd-inst{N}-lock default` |
| **ScoreBoardFile 분리** | 권장 | `ScoreBoardFile /var/run/httpd-inst{N}-scoreboard` |
| **systemd 서비스 유닛 분리** | 권장 | `httpd@instance1.service` 템플릿 활용 |
| **ip_local_port_range 확장** | 32GB 필수 | `net.ipv4.ip_local_port_range="1024 65535"` |
| **tcp_tw_reuse 활성화** | 권장 | OS 튜닝(Section 7)에 이미 포함 |
