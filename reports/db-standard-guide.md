# DB 서버 표준 설정 가이드라인

---

## 바쁜 엔지니어를 위한 스펙별 핵심 설정 요약 (Quick Cheatsheet)

> **[참고사항]** 아래 표는 DB 전용 서버 기준의 핵심 값만 추출한 요약입니다. 상세 산정 근거는 각 절을 참조하세요.

| DB 서버 RAM | PostgreSQL `shared_buffers` | PostgreSQL `max_wal_size` | MongoDB `cacheSizeGB` | 비고 |
| :---: | :---: | :---: | :---: | :--- |
| **8 GB** | 2 GB | 2 GB | 3.5 GB | 소규모 |
| **16 GB** | 4 GB | 4 GB | 7.5 GB | 표준 |
| **32 GB** | 8 GB | 16 GB | 15.5 GB | 고성능 |
| **64 GB** | 16 GB | 32 GB | 31.5 GB | 대규모 |

---

## 1. 계층별 핵심 파라미터 산정 가이드

> **[참고사항]** **PostgreSQL 고가용성(HA) 표준 아키텍처**: 내장 스트리밍 복제(Streaming Replication) 기술과 PgPool-II를 연계하여 구성한다.

### 1.1 PostgreSQL 핵심 파라미터

| 우선순위 | 파라미터 | 산정 공식 및 기준 | 비고 |
| :---: | :--- | :--- | :--- |
| 높음 | **shared_buffers** | `RAM * 0.25` | * PostgreSQL 공식 권장<br>* OS 페이지 캐시와 공유 |
| 낮음 | **effective_cache_size** | `RAM * 0.75` | * 쿼리 플래너 참고값 (실제 메모리 할당 아님) |
| 높음 | **work_mem** | `(RAM - shared_buffers) / (max_connections * 3)` 상한 | * 정렬/해시 연산용 메모리<br>* 과다 설정 시 OOM 위험 |
| 중간 | **maintenance_work_mem** | `RAM * 0.03 ~ 0.05` | * VACUUM, CREATE INDEX, REINDEX 시 사용 |
| 중간 | **wal_buffers** | `16MB` 고정 (또는 기본값 `-1`) | * PostgreSQL이 shared_buffers / 32를 자동 계산 (최대 16MB 캡)<br>* 별도 대용량 설정 불필요 |
| 높음 | **max_connections** | `Sum(WAS maxPoolSize) * 1.5` 이상 | * 70% Ceiling Rule 역산 기준 |
| 높음 | **superuser_reserved_connections** | `3` 고정 | * 관리자 긴급 접속 예약 |

### 1.2 PgPool-II 핵심 파라미터

| 우선순위 | 파라미터 | 산정 공식 및 기준 | 비고 |
| :---: | :--- | :--- | :--- |
| 높음 | **num_init_children** | `Sum(WAS maxPoolSize) + 여유량 (최소 120 이상)` | * PgPool이 수용할 동시 클라이언트(WAS) 연결 수<br>* Fixed-size 풀 환경에서는 풀 합산보다 반드시 커야 함 |
| 높음 | **max_pool** | 단일 DB/계정: `1`, 복수 DB/계정: `조합 수` | * 단일 DB 및 단일 계정 환경에서는 1로 설정 원칙<br>* 복수 DB/계정 환경에서만 조합 수만큼 상향 |
| 중간 | **child_life_time** | `1,800` (30min) | * DB idle_session_timeout(35min)보다 짧게 설정 |
| 중간 | **connection_life_time** | `1,800` (30min) | * PgPool -> DB 연결 수명 |
| 중간 | **client_idle_limit** | `600` (10min) | * 클라이언트 유휴 타임아웃 |
| 높음 | **reserved_connections** | `1 ~ 2` | * PgPool 관리용 예약 슬롯 |

> **[참고사항]** **[4GB RAM 독립 서버 가이드]** PgPool-II 전용 서버가 4GB RAM 사양이므로, `num_init_children = 120` 구동 시 프로세스 메모리 점유율(약 1GB 내외)은 안정 범위이나, 다중 자식 프로세스의 안정적인 포크(Fork) 및 세션 유지를 위해 OS 커널 세마포어 상한선 설정이 필수이다. Linux `sysctl.conf`에 `kernel.sem = 250 32000 250 128` 설정을 표준화한다.

### 1.3 MongoDB 핵심 파라미터

| 우선순위 | 파라미터 | 산정 공식 및 기준 | 비고 |
| :---: | :--- | :--- | :--- |
| 높음 | **wiredTigerEngineCacheSizeGB** | `0.5 * (RAM - 1GB)` | * MongoDB 기본 공식<br>* DB 전용 서버에서는 RAM의 50% 수준 할당 |
| 중간 | **writeConcern** | `w: 1` (기본) / `w: majority` | * RTO/RPO 요구사항에 따라 선택 |
| 중간 | **readPreference** | 서비스 특성에 따라 선택 | * 정산/결제 등 정합성 필수 서비스는 `primary`(기본값) 유지<br>* Replication Lag로 인한 과거 데이터 조회가 허용되는 조회성 서비스에 한해 `secondaryPreferred` 제한 적용 |
| 높음 | **Profiling Level** | `1 (slowms: 100)` | * COLLSCAN 감지 필수 |

---

## 2. 인프라 자원 스펙별 표준 설정값

### 2.1 PostgreSQL 메모리 설정 (DB 전용 서버 기준)

| DB 서버 RAM | shared_buffers | effective_cache_size | work_mem | maintenance_work_mem | wal_buffers | 비고 |
| :---: | :---: | :---: | :---: | :---: | :---: | :--- |
| **8 GB** | **2 GB** | 6 GB | 10 MB | 384 MB | 16 MB | 소규모 |
| **16 GB** | **4 GB** | 12 GB | 32 MB | 1 GB | 16 MB | 표준 |
| **32 GB** | **8 GB** | 24 GB | 64 MB | 2 GB | 16 MB | 고성능 |
| **64 GB** | **16 GB** | 48 GB | 128 MB | 4 GB | 16 MB | 대규모 |

> **주의**: DB 서버는 WAS/Web과 물리적으로 분리된 전용 서버(또는 VM)를 권장.
> 컨설팅사 경험에 따르면 Master DB는 Physical Box 할당 사례 존재.

### 2.2 PostgreSQL WAL 및 체크포인트 설정

| 우선순위 | 파라미터 | DB 서버 RAM | 표준값 | 비고 |
| :---: | :--- | :---: | :---: | :--- |
| 높음 | **max_wal_size** | ~16 GB | **2 GB** | 체크포인트 간 WAL 누적 상한 |
| 높음 | **max_wal_size** | 16 ~ 32 GB | **4 GB** | 16GB RAM 기준값 |
| 높음 | **max_wal_size** | 32 GB 이상 | **16 ~ 32 GB** | 과소 설정 시 잦은 강제 체크포인트로 디스크 I/O 스파이크 유발 |
| 중간 | **min_wal_size** | 공통 | **1 GB** | WAL 최소 유지 크기 |
| 중간 | **checkpoint_completion_target** | 공통 | **0.9** | 체크포인트를 시간에 걸쳐 분산 |
| 높음 | **max_wal_senders** | 공통 | **3 ~ 5** | Replica(또는 Standby) 서버 수 + 여유 |

### 2.3 PostgreSQL autovacuum 설정

| 우선순위 | 파라미터 | 표준값 | 비고 |
| :---: | :--- | :--- | :--- |
| 높음 | **autovacuum** | on | * 반드시 활성화 (비활성화 금지) |
| 중간 | **autovacuum_max_workers** | `3 ~ 5` | * 워커 수를 무작정 늘리면 디스크 I/O를 잠식<br>* 5 이하 유지 |
| 중간 | **autovacuum_naptime** | 1 min | * 점검 주기 |
| 중간 | **autovacuum_vacuum_scale_factor** | 0.1 | * Dead Tuple 10% 도달 시 VACUUM |
| 중간 | **autovacuum_vacuum_cost_limit** | `1000 ~ 2000` | * 기본값(200) 대비 상향<br>* 초당 처리 한도를 높여 워커 증설 없이 VACUUM 성능 확보 |

### 2.4 PostgreSQL 커넥션 및 타임아웃 설정

| 우선순위 | 파라미터 | 표준값 | 비고 |
| :---: | :--- | :--- | :--- |
| 높음 | **max_connections** | 200 ~ 500 | * WAS 풀 합산의 1.5배 이상 확보 |
| 높음 | **superuser_reserved_connections** | 3 | * 관리자 예약 |
| 중간 | **statement_timeout** | 30,000 ms (30s) | * 장기 실행 쿼리 방지<br>* 애플리케이션 레벨 설정 권장 |
| 높음 | **lock_timeout** | 10,000 ms (10s) | * Lock 대기 시간 제한 |
| 높음 | **idle_in_transaction_session_timeout** | 60,000 ms (60s) | * 트랜잭션 시작 후 유휴 상태 강제 종료 (교착 방지) |

### 2.5 MongoDB 메모리 및 Replica Set 설정

| DB 서버 RAM | wiredTiger cacheSizeGB | 비고 |
| :---: | :---: | :--- |
| **8 GB** | **3.5 GB** | `0.5 * (8 - 1) = 3.5` |
| **16 GB** | **7.5 GB** | `0.5 * (16 - 1) = 7.5` |
| **32 GB** | **15.5 GB** | `0.5 * (32 - 1) = 15.5` |
| **64 GB** | **31.5 GB** | `0.5 * (64 - 1) = 31.5` |

> **[참고사항]** MongoDB는 DB 전용 서버 기준으로 `0.5 * (RAM - 1GB)` 공식에 따라 WiredTiger 캐시를 할당.
> 공유 환경(WAS/DB 혼합 배포)에서는 25% 수준으로 명시적 제한 권장.
>
> **Replica Set 멤버 구성 (Quorum) 표준**:
> * 스플릿 브레인 방지 및 정상적인 투표(Quorum) 성립을 위해 **최소 3노드(PSS: Primary-1, Secondary-2) 구성을 표준**으로 한다.
> * 하드웨어 자원이 극도로 제한된 경우에 한해 중재자(Arbiter)를 포함한 3노드(PSA: Primary-1, Secondary-1, Arbiter-1) 구성을 허용한다.

---

## 3. 타임아웃 캐스케이드 (Timeout Cascade) 표준값

상위 계층이 하위 계층보다 먼저 연결을 끊도록 설정하여 무효 커넥션 예외 및 교착 상태를 차단합니다.

### 3.1 WAS (HikariCP) -> PgPool-II -> PostgreSQL

```
WAS HikariCP maxLifetime (1,740,000ms = 29min)
     |
     |  maxLifetime < child_life_time < idle_session_timeout
     v
PgPool-II child_life_time (1,800s = 30min)
     |
     |  child_life_time < idle_session_timeout
     v
PostgreSQL idle_session_timeout (2,100,000ms = 35min)
```

> **[높음] 핵심 원칙**: `WAS maxLifetime (29min) < PgPool child_life_time (30min) < DB idle_session_timeout (35min)`
>
> 상위 계층과 하위 계층이 동시에 커넥션을 끊는 레이스 컨디션을 방지하기 위해, 등호(`<=`)가 아닌 **엄격한 부등호(`<`)**로 계층 간 타임아웃을 격리합니다.

### 3.2 WAS (HikariCP) -> MongoDB

```
WAS HikariCP maxLifetime (1,740,000ms = 29min)
     |
     v
MongoDB connectionPool maxIdleTimeMS (1,800,000ms = 30min)
     |
     v
MongoDB driver socketTimeoutMS (0 = 무제한, 애플리케이션 레벨 제어)
```

### 3.3 PostgreSQL 내부 타임아웃 계층

```
statement_timeout (30s)
     |
     +-- idle_in_transaction_session_timeout (60s)
     |      -- 트랜잭션 시작 후 쿼리 없이 60초 대기 시 강제 종료
     |
     +-- lock_timeout (10s)
            -- Lock 대기 10초 초과 시 자동 취소 (교착 방지)
```

---

## 4. DBMS 벤더별 실무 설정 스크립트 & 프로퍼티

### 4.1 PostgreSQL (`postgresql.conf`)

```conf
# -------------------------------------------------------
# Memory (8GB DB 전용 서버 기준) - OOM 방지 보정 완료
# -------------------------------------------------------
shared_buffers = 2GB                # [높음] RAM * 0.25 (공식 준수)
effective_cache_size = 6GB          # [낮음] 쿼리 플래너 참고값 (RAM * 0.75)
work_mem = 10MB                     # [높음] 정렬/해시 연산용 상한선 (8GB 장비 안전마진 확보)
maintenance_work_mem = 384MB        # [중간] VACUUM, CREATE INDEX용 (RAM * 0.05 이내 제한)
wal_buffers = 16MB                  # [중간] 자동 계산 캡(16MB) 고정

# -------------------------------------------------------
# WAL & Checkpoint
# -------------------------------------------------------
wal_level = replica                 # [높음] 스트리밍 복제(Streaming Replication) 구성을 위한 필수 로그 레벨
max_wal_size = 2GB                  # [높음] ~16GB RAM 기준
min_wal_size = 1GB                  # [중간] WAL 최소 유지 크기
checkpoint_completion_target = 0.9  # [중간] 체크포인트 분산
max_wal_senders = 5                 # [높음] Replica 구성원 수 + 여유

# -------------------------------------------------------
# Connections
# -------------------------------------------------------
max_connections = 200               # [높음] WAS 풀 합산 * 1.5 이상
superuser_reserved_connections = 3  # [높음] 관리자 긴급 접속 예약
hot_standby = on                    # [높음] Replica 노드에서 읽기 쿼리 수용 (PgPool-II 읽기 분산 필수)
listen_addresses = '*'              # [높음] 원격 접속 허용

# -------------------------------------------------------
# Timeouts
# -------------------------------------------------------
statement_timeout = 30000                       # [중간] 장기 실행 쿼리 방지 (30초)
lock_timeout = 10000                            # [높음] Lock 대기 시간 제한 (10초)
idle_in_transaction_session_timeout = 60000     # [높음] 트랜잭션 유휴 강제 종료 (60초, 교착 방지)
idle_session_timeout = 2100000                  # [높음] 유휴 세션 강제 종료 (35분, 타임아웃 캐스케이드 최하위)

# -------------------------------------------------------
# Autovacuum
# -------------------------------------------------------
autovacuum = on                             # [높음] 비활성화 금지
autovacuum_max_workers = 3                  # [중간] 3~5 유지, 무작정 증설 금지
autovacuum_naptime = 1min                   # [중간] 점검 주기
autovacuum_vacuum_scale_factor = 0.1        # [중간] Dead Tuple 10% 도달 시 VACUUM
autovacuum_vacuum_cost_limit = 2000         # [중간] 기본값(200) 대비 상향, VACUUM 성능 확보

# -------------------------------------------------------
# Query Planner
# -------------------------------------------------------
random_page_cost = 1.1              # [높음] SSD 환경 (HDD: 4.0)
effective_io_concurrency = 200      # [높음] SSD 환경 (HDD: 2)
```

### 4.2 PgPool-II (`pgpool.conf`)

```conf
# -------------------------------------------------------
# PgPool-II 전용 서버 (4GB RAM 독립 서버 기준)
# Connection Pooling
# -------------------------------------------------------
num_init_children = 120          # [높음] WAS 풀 합산 수용을 위해 상향
max_pool = 1                     # [높음] 단일 DB/단일 계정 환경 기준. 복수 DB/계정 시 조합 수만큼 상향
child_life_time = 1800           # [높음] 30min (DB idle_session_timeout 35min보다 짧게 설정)
connection_life_time = 1800      # [중간] 30min
client_idle_limit = 600          # [중간] 10min
reserved_connections = 1         # [높음] 관리 접속 보장

# -------------------------------------------------------
# Load Balancing
# -------------------------------------------------------
load_balance_mode = on           # [높음] 읽기 분산 활성화
master_slave_mode = on           # [높음] 스트리밍 복제 모드
master_slave_sub_mode = stream   # [높음] 스트림 복제 서브모드

backend_weight0 = 1              # [중간] Primary 가중치
backend_weight1 = 1              # [중간] Replica 가중치 (R:W 7:3 환경에서 읽기 분산)

# -------------------------------------------------------
# Health Check
# -------------------------------------------------------
health_check_period = 30         # [중간] 헬스체크 주기 (초)
health_check_timeout = 10        # [중간] 헬스체크 타임아웃 (초)
health_check_max_retries = 3     # [중간] 최대 재시도 횟수

# -------------------------------------------------------
# Watchdog (SPOF 방지 - VIP 기반 고가용성)
# -------------------------------------------------------
use_watchdog = on                # [높음] SPOF 방지를 위한 Watchdog 활성화
wd_hostname = 'pgpool-node1'    # [높음] Watchdog 호스트명
wd_vip = '10.0.0.100'           # [높음] 서비스 가상 IP (VIP)

# -------------------------------------------------------
# Auto Failover (Primary 노드 다운 시 복제 노드 승격)
# -------------------------------------------------------
failover_command = '/etc/pgpool-II/failover.sh'  # [높음] 페일오버 트리거 스크립트
```

### 4.3 MongoDB (`mongod.conf`)

```yaml
# -------------------------------------------------------
# Storage (8GB DB 전용 서버 기준)
# -------------------------------------------------------
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 3.5            # [높음] 0.5 * (8 - 1) = 3.5GB

# -------------------------------------------------------
# Replica Set
# -------------------------------------------------------
replication:
  replSetName: rs0                # [높음] Replica Set 명

# -------------------------------------------------------
# Profiling (COLLSCAN 감지 필수)
# -------------------------------------------------------
operationProfiling:
  mode: slowOp                    # [높음] Slow Query 감지 활성화
  slowOpThresholdMs: 100          # [높음] 100ms 이상 쿼리 기록

# -------------------------------------------------------
# Network
# -------------------------------------------------------
net:
  maxIncomingConnections: 65536   # [중간] 최대 수신 커넥션
```

```javascript
// Replica Set 초기화 후 설정 (mongosh)

// [높음] Profiling Level 설정 (Slow Query 감지)
db.setProfilingLevel(1, { slowms: 100 })
```

---

## 5. 공유 DB 환경 커넥션 풀 가용 가이드 (70% Ceiling Rule)

단일 DB를 여러 서비스가 공유할 때 전사 장애 전파를 막기 위한 인스턴스별 배정 수치입니다.

**대원칙**: 모든 애플리케션의 maxPoolSize 총합 <= DB max_connections * 0.7

### 5.1 PostgreSQL max_connections 산정 기준

| 전사 WAS 인스턴스 수 | WAS 풀 합산 (maxPoolSize) | 권장 max_connections | 비고 |
| :---: | :---: | :---: | :--- |
| 10개 이하 | ~200 | **200 ~ 300** | 소규모 |
| 10 ~ 20개 | 200 ~ 400 | **300 ~ 500** | 중규모 |
| 20개 이상 | 400+ | **500+** (PgPool-II 필수) | 대규모 |

### 5.2 팀별 DB 설정 점검 및 보정 내역

> **현재 즉시 적용 대상** (플랫폼개발팀) | **미래 선행 표준 준비** (DB2 운영팀)

| 팀 / 서비스 | 사용 DBMS | 현행 주요 설정 | 표준 적용 방향 | 사유 및 보정 방향 |
| :--- | :--- | :--- | :--- | :--- |
| **플랫폼개발 (나이스M)** | PostgreSQL (via PgPool-II) + MongoDB | PgPool 경유, R:W=7:3 | 인스턴스당 **maxPoolSize=20** | * PostgreSQL 및 MongoDB 둘 다 사용<br>* 읽기 비중 높으므로 Replica 읽기 분산 필수 |
| **플랫폼개발 (나이스차저)** | MongoDB | M1/S2/A0, R:W=6:4 | 인스턴스당 **maxPoolSize=20~30**<br>**Profiling Level 1 즉시 활성화** | * MongoDB만 사용 중<br>* COLLSCAN 무감지 상태 운영 위험<br>* 정산/결제 서비스는 `primary` 유지 필수 |
| |||||
| **CL플랫폼** | DB2 | 현행 50 | 인스턴스당 **maxPoolSize=15** | * 현금정보계와 같은 서버 사용, 인스턴스당 15로 제한<br>* 향후 신규 서비스 도입 시 공유 환경 합류 대비 선행 표준 적용 |
| **주차서비스** | DB2 | 현행 100 | 인스턴스당 **maxPoolSize=20~30** | * 독립 DB2 환경으로 타 서비스와 커넥션 경합 없음<br>* 과대 설정 축소 보정 |
| **현금정보계** | DB2 | 7개 컨테이너, maxPoolSize=50 | 인스턴스당 **maxPoolSize=15** (총 105) | * 7개 컨테이너 다중화 환경 감안 인스턴스당 15로 제한<br>* 향후 신규 서비스 도입 시 공유 환경 합류 대비 선행 표준 적용 |

> **[참고사항]** CL플랫폼 및 현금정보계의 `maxPoolSize`를 50에서 15로 축소 제안한 사항에 대하여, 풀 크기 축소 적용 전 APM 모니터링을 통해 실제 피크 타임의 **Active Connection Peak 수치를 반드시 검증**해야 하며, 커넥션 고갈 우려 시 **WAS 인스턴스의 스케일 아웃(Scale-out)을 병행**해야 합니다.

### 5.3 PgPool-II 커넥션 풀 산출 예시 (플랫폼개발팀 나이스M 기준)

| 항목 | 플랫폼개발팀 산출 수치 | 비고 |
| :--- | :--- | :--- |
| **대상 서비스** | 나이스M (Nice M) | PostgreSQL(via PgPool-II) + MongoDB 운영 |
| **총 WAS 인스턴스 수** | 4 개 (이중화 아키텍처) | WAS 표준 가이드라인 기준 |
| **전체 WAS 풀 합산** | **80 ~ 100 개** | WAS V2 고정 풀 기준 (인스턴스당 20~25) |
| [높음] **PgPool num_init_children** | **120** | WAS 풀 합산(100) 대비 안전 마진 확보 |
| [높음] **PgPool max_pool** | **1** | 단일 DB/단일 계정 환경 기준. 복수 DB/계정 시 조합 수만큼 상향 |
| **PgPool -> PG 이론상 최대 연결** | 120 개 | `num_init_children(120) * max_pool(1)` = 120 |
| [높음] **PG 권장 max_connections** | **200** | PgPool 백엔드 수용량(120)의 1.5배 이상 확보, superuser 접속 여유 포함 |

---

## 6. DB 모니터링 최소 체계

| 우선순위 | 모니터링 항목 | PostgreSQL | MongoDB | 임계치 | 조치 |
| :---: | :--- | :--- | :--- | :--- | :--- |
| 높음 | **Active Sessions** | `pg_stat_activity` | `db.serverStatus().connections` | max_connections 70% 경고 / 85% 위험 | 커넥션 풀 설정 재검토 |
| 높음 | **Slow Query** | `pg_stat_statements` (>= 1s) | Profiling Level 1 (>= 100ms) | 발생 시 즉시 분석 | 인덱스 추가 / 쿼리 튜닝 |
| 높음 | **COLLSCAN** | `seq_scan / idx_scan` 비율 | `system.profile` stage: COLLSCAN | 발생 시 즉시 조치 | 인덱스 설계 |
| 중간 | **Lock Wait** | `pg_locks` | `db.currentOp()` | 대기 시간 > 1s | 트랜잭션 분석 |
| 높음 | **Replication Lag** | `pg_stat_replication` | `rs.printSecondaryReplicationInfo()` | > 5s 경고 / > 30s 위험 | 네트워크/부하 점검 |
| 중간 | **Dead Tuples** | `pg_stat_user_tables` n_dead_tup | -- | 테이블 크기 10% 초과 | autovacuum 강제 |
| 중간 | **Cache Hit Ratio** | `pg_stat_database` (blks_hit/blks_read) | WiredTiger cache percent | < 95% 경고 | shared_buffers / cacheSizeGB 증설 검토 |

---

## 7. 검증 체크리스트

| 우선순위 | 검증 항목 | 조건 | 위반 시 영향 |
| :---: | :--- | :--- | :--- |
| 높음 | shared_buffers <= RAM * 0.25 | PostgreSQL 공식 권장 | OOM, 커널 페이지 캐시 부족 |
| 높음 | max_connections >= Sum(WAS maxPoolSize) * 1.5 | 70% Ceiling Rule 역산 | 커넥션 거부, 서비스 장애 |
| 높음 | WAS maxLifetime < PgPool child_life_time < DB idle_session_timeout | 타임아웃 캐스케이드 정합 (엄격 부등호) | 레이스 컨디션, 무효 커넥션, 간헐적 에러 |
| 높음 | autovacuum = on | 필수 | Dead Tuple 누적, 성능 점진 저하 |
| 중간 | autovacuum_vacuum_cost_limit >= 1000 | 기본값(200) 대비 상향 | VACUUM 처리 지연, Dead Tuple 누적 |
| 높음 | MongoDB Profiling Level >= 1 | COLLSCAN 감지 필수 | 인덱스 누락 무감지, 선형 성능 저하 |
| 중간 | Cache Hit Ratio >= 95% | PostgreSQL / MongoDB 공통 | 디스크 I/O 증가, 응답 지연 |
| 높음 | idle_in_transaction_session_timeout 설정 | 교착 방지 | 트랜잭션 유휴로 인한 Lock 점유 |
| 중간 | random_page_cost = 1.1 (SSD) | SSD 환경 필수 | 쿼리 플래너 비효율적 실행 계획 |
| 높음 | PgPool reserved_connections >= 1 | 관리 접속 보장 | 장애 시 DBA 접속 불가 |
| 높음 | PgPool max_pool = 1 (단일 DB/계정) | 불필요한 커넥션 폭증 방지 | 백엔드 연결 수 기하급수적 증가 |
| 높음 | Sum(maxPoolSize) <= DB max_conn * 0.7 | 공유 DB 70% Ceiling | 타 서비스 커넥션 고갈, 전파 장애 |
