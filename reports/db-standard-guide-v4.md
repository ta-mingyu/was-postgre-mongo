# DB 서버 표준 설정 가이드 (PostgreSQL + PgPool-II + MongoDB)

> **기준 문서**: `reports/final-standard-guide.md`
> **통합 본**. 각 서버별 상세 정본은 `reports/final/{postgresql,pgpool-ii,mongodb}.md` 참조.
> **버전**: v4 (2026-07-03 갱신 — 방화벽 30~60min 정정, MongoDB glibc rseq/maxIdleTimeMS 보완, idle_session PG14+ 명시, somaxconn↔acceptCount 관계)

---

# PostgreSQL 서버 설정 가이드

> **기준 문서**: `reports/final-standard-guide.md`
> **적용 범위**: PostgreSQL (Primary / Replica)
> **프로덕션 표준**: PgPool-II + Streaming Replication
> **개발/테스트**: Standalone

---

## 0. 적용 전제

프로덕션 표준 아키텍처는 PgPool-II + Streaming Replication. 아래 토폴로지와 전제를 반드시 함께 확인한다.

```mermaid
graph LR
    subgraph WAS Layer
        W1[WAS-1<br/>HikariCP]
        W2[WAS-2<br/>HikariCP]
        W3[WAS-N<br/>HikariCP]
    end

    subgraph PgPool Layer
        PP1[PgPool-II<br/>Active]
        PP2[PgPool-II<br/>Standby]
    end

    subgraph PostgreSQL Layer
        PG_M[(PostgreSQL<br/>Primary)]
        PG_R[(PostgreSQL<br/>Replica)]
    end

    W1 --> VIP[Virtual IP]
    W2 --> VIP
    W3 --> VIP
    VIP --> PP1
    PP1 -->|Write / Read| PG_M
    PP1 -->|Read Only<br/>Load Balance| PG_R
    PP1 -.->|Watchdog<br/>SPOF 방지| PP2
    PG_M -->|WAL Streaming| PG_R
```

- **max_connections=100 고정은 PgPool-II 연결 풀링 전제**. WAS 직접 연결 시에는 70% Ceiling 산정 공식(`max_connections >= Sum(WAS maxPoolSize) * 1.5`)을 따를 것
- **방화벽 TCP 30~60min (범위, 최단 30min 기준 설계)**: 모든 타임아웃의 최상위. WAS maxLifetime(27min) < PgPool child_life_time(28min) < PG idle_session_timeout(30min) <= 방화벽 최단(30min). keepaliveTime(60s)이 주기적 ping으로 잔여 레이스 방어
- **PostgreSQL과 MongoDB는 동일 호스트 병설 금지**: `vm.overcommit_memory` 설정이 PostgreSQL(2)과 MongoDB 8.0(1)에서 서로 충돌. 반드시 물리적으로 분리된 서버에서 운영

---

## 1. OS 커널 설정

### 1.1 공통 파라미터 (모든 서버 적용)

```ini
# /etc/sysctl.d/99-infra-common.conf -- 모든 서버 공통
fs.file-max = 2097152
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 5
```

```bash
# /etc/security/limits.d/99-infra-common.conf -- 모든 서버 공통 (PAM 기반 적용)
*  soft  nofile  1048576
*  hard  nofile  1048576
*  soft  nproc   65536
*  hard  nproc   65536
```

> **systemd 서비스 필수 추가 설정**: 위 limits.conf는 PAM 기반 세션 접속에만 적용되며, **systemd가 관리하는 서비스 데몬에는 적용되지 않음**. 아래 systemd drop-in override를 반드시 추가 적용해야 함.

```ini
# /etc/systemd/system/postgresql.service.d/override.conf
# 서비스명은 설치 방법에 따라 postgresql-<version>.service 일 수 있음
[Service]
LimitNOFILE=1048576
LimitNPROC=65536
```

```bash
# systemd 적용
systemctl daemon-reload
systemctl restart postgresql
```

| 파라미터 | 값 | 역할 |
|:---|:---|:---|
| fs.file-max | 2,097,152 | 시스템 전체 FD 상한. 대규모 동시 접속 시 Too many open files 방지 |
| net.core.somaxconn | 4,096 | OS 커널 TCP Listen Backlog. DB는 min(backlog, somaxconn)으로 실제 Backlog 결정 |
| net.ipv4.tcp_max_syn_backlog | 4,096 | SYN Queue 상한. somaxconn과 세트로 설정 |
| net.ipv4.tcp_keepalive_time | 300 (5분) | TCP Keepalive 최초 대기 시간. 기본 7,200초 단축. WAS→DB 죽은 커넥션 조기 감지 |
| net.ipv4.tcp_keepalive_intvl | 30 | Keepalive 프로브 재전송 간격 |
| net.ipv4.tcp_keepalive_probes | 5 | 연속 실패 시 dead 판정. 300+30x5=450초 내 확정 |
| ulimit -n (nofile) | 1,048,576 | 프로세스당 FD 한도. infinity 시 커널 ~8.6GB 할당 (Bug 2394600) |
| ulimit -u (nproc) | 65,536 | 프로세스/스레드 수 상한. Fork Bomb 방지 |

### 1.2 PostgreSQL 서버 전용 파라미터

```ini
# /etc/sysctl.d/99-postgresql-tuning.conf -- PostgreSQL 서버 전용
vm.swappiness = 1
vm.overcommit_memory = 2
vm.overcommit_ratio = 90
vm.dirty_background_ratio = 5
vm.dirty_ratio = 10
vm.min_free_kbytes = 102400
vm.zone_reclaim_mode = 0
```

```bash
# THP (Transparent Huge Pages) 비활성화 -- OS 리부팅 시 초기화되는 1회성 명령임.
# 영구 설정은 root 권한이 필요하므로 IT ONE을 통해 IT 운영실에 변경 요청할 것.
#
# [참고: 영구 설정 방법 -- IT 운영실 적용용]
# 방법 1 (권장): GRUB 커널 파라미터 (리부팅 필요)
#   grubby --update-kernel=ALL --args="transparent_hugepage=never"
#
# 방법 2: TuneD 프로파일
#   /etc/tuned/<profile>/tuned.conf 에 [vm] transparent_hugepages=never 설정
```

| 파라미터 | 값 | 역할 |
|:---|:---|:---|
| vm.swappiness | 1 | shared_buffers가 디스크로 내려가면 쿼리 성능 급감. 스왑 거의 허용 안 함 |
| vm.overcommit_memory | 2 | 엄격 모드. fork 기반 백엔드 생성 시 OOM Killer가 postmaster 종료 → 전체 장애 방지 |
| vm.overcommit_ratio | 90 | overcommit_memory=2 전용. 커밋 한도 = Swap + (RAM x 90%). 기본 50%는 너무 낮음 |
| Transparent Huge Pages | disabled (never) | OLTP 희소 접근 패턴에서 THP compaction이 수백 ms 지연 스파이크 유발 |
| vm.dirty_background_ratio | 5 | 더티 페이지 5% 도달 시 백그라운드 플러시. WAL/체크포인트 I/O 버스트 분산 |
| vm.dirty_ratio | 10 | 더티 페이지 10% 도달 시 동기 플러시 강제. 기본(20~30)은 DB 쓰기 부하에서 I/O 마비 |
| vm.min_free_kbytes | 102400 (100MB) | 커널 최소 여유 RAM. 대량 정렬/해시조인 시 Direct Reclaim(전체 일시정지) 방지 |
| vm.zone_reclaim_mode | 0 | NUMA 환경에서 다른 노드 메모리 회수 금지. shared_buffers 공유 접근 시 성능 급감 방지 |

> **PostgreSQL과 MongoDB는 동일 호스트 병설 금지**: vm.overcommit_memory 설정이 PostgreSQL(2)과 MongoDB 8.0(1)에서 서로 충돌. 반드시 물리적으로 분리된 서버에서 운영.

### 1.3 적용 명령어

```bash
sysctl --load /etc/sysctl.d/99-infra-common.conf
sysctl --load /etc/sysctl.d/99-postgresql-tuning.conf
# THP 영구 설정은 IT ONE을 통해 IT 운영실에 변경 요청 (상세는 1.2절 참조)
systemctl daemon-reload
systemctl restart postgresql
```

---

## 2. PostgreSQL 설정

### 2.1 Primary / Replica 핵심 파라미터

| 파라미터 | Primary | Replica | 역할 |
|:---|:---:|:---:|:---|
| wal_level | replica | (상속) | WAL에 복제에 필요한 충분한 정보 포함. minimal 시 복제 불가 |
| max_wal_senders | 5 | 5 | WAL 스트리밍 연결 최대 개수. 장애 시 승격 후 백업 연결 즉시 허용 |
| hot_standby | (해당 없음) | on | Replica 읽기 쿼리 수용. PgPool-II 읽기 분산 동작 필수 |
| hot_standby_feedback | (해당 없음) | on | Replica 읽기 분산 시 Primary Vacuum에 의한 쿼리 취소(Conflict) 방지 |
| archive_mode | always | always | WAL 아카이빙. Primary+Replica 모두 수행. Replica에서 직접 백업/계층형 복제 가능 |
| max_connections | 100 | 100 | 최대 동시 클라이언트 연결. OOM 예방을 위해 100 고정 (PgPool 연결 풀링 전제) |

### 2.2 RAM별 파라미터 매트릭스 (프로덕션 PgPool+SR)

**Memory 그룹**

| DB 서버 RAM | shared_buffers | effective_cache_size | work_mem | maintenance_work_mem | wal_buffers |
|:---:|:---:|:---:|:---:|:---:|:---:|
| 8 GB | 2 GB | 6 GB | 8 MB | 384 MB | 16 MB |
| 16 GB | 4 GB | 12 GB | 16 MB | 1 GB | 16 MB |
| 32 GB | 8 GB | 24 GB | 32 MB | 2 GB | 16 MB |
| 64 GB | 16 GB | 48 GB | 64 MB | 4 GB | 16 MB |

**WAL & Connections 그룹**

| DB 서버 RAM | max_wal_size | min_wal_size | max_wal_senders | max_connections | superuser_reserved |
|:---:|:---:|:---:|:---:|:---:|:---:|
| 8 GB | 2 GB | 1 GB | 5 | 100 | 3 |
| 16 GB | 4 GB | 1 GB | 5 | 100 | 3 |
| 32 GB | 16 GB | 1 GB | 5 | 100 | 3 |
| 64 GB | 32 GB | 1 GB | 5 | 100 | 3 |

> work_mem 산정 상한(이론 참고치): `(RAM - shared_buffers) / (max_connections * 3)` (kofemann/pgtune). PostgreSQL 공식은 명시 공식이 없으나 "complex query = multiple concurrent operations" 경고에 근거. **위 매트릭스(8/16/32/64MB)는 운영 적용 표준값**으로, 이론 상한보다 보수적(OLTP/PgPool 환경 최적화). (2026-07-02 TA 결정, *8 폐기)
> effective_cache_size는 실제 할당 아님(플래너 참고값, OS page cache 포함 추정치).

### 2.3 postgresql.conf 전문 (프로덕션, 8GB 기준)

```conf
# -------------------------------------------------------
# Memory (8GB DB 전용 서버 기준)
# -------------------------------------------------------
shared_buffers = 2GB                # RAM * 0.25
effective_cache_size = 6GB          # RAM * 0.75
work_mem = 8MB                     # 운영 표준값 (이론 상한: (RAM-shared_buffers)/(max_conn*3), kofemann/pgtune)
maintenance_work_mem = 384MB        # RAM * 0.047~0.0625 (PGTune 기준)
wal_buffers = 16MB                  # 고정

# -------------------------------------------------------
# WAL & Checkpoint
# -------------------------------------------------------
wal_level = replica
max_wal_size = 2GB
min_wal_size = 1GB
checkpoint_completion_target = 0.9
max_wal_senders = 5                 # Replica + 여유

# -------------------------------------------------------
# Connections
# -------------------------------------------------------
max_connections = 100               # OOM 예방 100 고정, PgPool 연결 풀링으로 백엔드 연결 제어
superuser_reserved_connections = 3
hot_standby = on
hot_standby_feedback = on           # Replica: Vacuum 충돌 방지
archive_mode = always               # WAL 아카이빙 (승격 대비)
listen_addresses = '*'

# -------------------------------------------------------
# Timeouts
# -------------------------------------------------------
statement_timeout = 30000                       # 30s
lock_timeout = 10000                            # 10s
idle_in_transaction_session_timeout = 60000     # 60s
idle_session_timeout = 1800000                  # 30min (캐스케이드 최하위)

# -------------------------------------------------------
# Autovacuum
# -------------------------------------------------------
autovacuum = on
autovacuum_max_workers = 3
autovacuum_naptime = 1min
autovacuum_vacuum_scale_factor = 0.1
autovacuum_vacuum_cost_limit = 2000

# -------------------------------------------------------
# Query Planner (SSD)
# -------------------------------------------------------
random_page_cost = 1.1
effective_io_concurrency = 200
```

> 각 RAM 스펙별로 Memory 섹션의 값만 2.2절 매트릭스에 맞게 변경. 나머지 설정은 모든 RAM 공통.

### 2.4 개발/테스트: Standalone

> 본 구성은 개발 및 테스트 환경에 한해서만 허용. 프로덕션에서는 PgPool-II + Streaming Replication 구성(0절 아키텍처) 적용 필수.

#### 핵심 파라미터 차이 (PgPool+SR 대비)

| 파라미터 | Standalone | PgPool+SR 대비 | 역할 |
|:---|:---:|:---:|:---|
| wal_level | replica | 동일 | 기본 표준 replica (PITR/아카이브 백업 허용). minimal은 순수 개발계만 |
| max_wal_senders | 0 | PgPool+SR: 5 | 복제 연결 불필요 |
| hot_standby | off | PgPool+SR: on (Replica) | Standby 노드 없음 |
| archive_mode | off | PgPool+SR: always | 아카이빙 불필요한 개발 환경 |
| max_connections | 100 | 동일 | 단일/소수 WAS 연결이므로 100 충분 |

#### RAM별 파라미터 매트릭스 (Standalone)

| DB 서버 RAM | shared_buffers | effective_cache_size | work_mem | maintenance_work_mem | wal_buffers | max_connections |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| 8 GB | 2 GB | 6 GB | 8 MB | 384 MB | 16 MB | 100 |
| 16 GB | 4 GB | 12 GB | 16 MB | 768 MB | 16 MB | 100 |
| 32 GB | 8 GB | 24 GB | 32 MB | 1.5 GB | 16 MB | 100 |
| 64 GB | 16 GB | 48 GB | 64 MB | 3 GB | 16 MB | 100 |

> max_wal_size는 기본값(1GB) 사용 가능, 쓰기 빈도에 따라 2GB까지 상향 허용.

---

## 3. 타임아웃 & 커넥션 캐스케이드

### 3.1 PostgreSQL 내부 타임아웃 Guardrails

각 파라미터는 서로 다른 시점에 동작하는 독립적인 가드레일.

| 파라미터 | 표준값 | 역할 |
|:---|:---|:---|
| statement_timeout | 30,000ms (30s) | 실행 중인 쿼리(Active Query) 최대 지속 시간. 장기 실행 쿼리 리소스 독점 방지 |
| lock_timeout | 10,000ms (10s) | Lock 획득 대기 최대 시간. 10초 초과 시 쿼리 자동 취소. lock_timeout < statement_timeout 유지 필수 |
| idle_in_transaction_session_timeout | 60,000ms (60s) | BEGIN 후 쿼리 완료 후 유휴 상태 최대 시간. COMMIT/ROLLBACK 없이 방치 시 Lock 점유로 전파 장애 |
| idle_session_timeout | 1,800,000ms (30min) | 클라이언트 유휴 세션 강제 종료. 연결 누수 및 좀비 세션으로부터 DB 자원 보호. **PostgreSQL 14+(2021) 전용 파라미터** — PG 13 이하 환경에서는 미지원(미인식), TCP keepalive로 대체 방어 |

```
PostgreSQL Session Timeout Guardrails
  |
  |-- statement_timeout (30s)
  |     쿼리 실행 중 상태의 최대 지속 시간 제어.
  |     |
  |     +-- lock_timeout (10s)
  |            statement_timeout 도중 Lock 대기 시간에만 관여.
  |            Lock 획득 대기 10초 초과 시 자동 취소 (교착 방지).
  |
  |-- idle_in_transaction_session_timeout (60s)
        트랜잭션 시작 후 쿼리 완료 후 유휴 상태 최대 시간.
        BEGIN 이후 COMMIT/ROLLBACK 없이 방치되는 세션 방지.
```

### 3.2 타임아웃 캐스케이드 (WAS -> PgPool -> PostgreSQL)

```
WAS HikariCP maxLifetime (1,620,000ms = 27min)
     |
     |  maxLifetime < child_life_time < idle_session_timeout
     v
PgPool-II child_life_time (1,680s = 28min)
     |
     |  child_life_time < idle_session_timeout
     v
PostgreSQL idle_session_timeout (1,800,000ms = 30min)
```

> WAS 직접 연결 시: HikariCP maxLifetime(27min) < PG idle_session_timeout(30min) < 방화벽(30min)

### 3.3 공유 DB 커넥션 풀 가이드 (직접 연결 시)

PgPool-II 없이 WAS가 DB에 직접 연결하는 환경:

```
DB max_connections = 100 (DB 서버 설정)
      |
      +-- 30 예약: 관리자 세션(superuser_reserved=3), 모니터링, 긴급 접속
      |
      +-- 70 가용: 애플리케이션 커넥션 풀 전체 합산 상한 (max_connections * 0.7)
```

> 절대 제약 (직접 연결 시): 모든 애플리케이션의 maxPoolSize 합산값은 max_connections * 0.7 (=70)을 초과 불가.

---

## 4. 검증 체크리스트

- [ ] shared_buffers <= RAM * 0.25 -- PostgreSQL 공식 권장 (위반 시: OOM, 커널 페이지 캐시 부족)
- [ ] max_connections = 100 고정 -- OOM 예방 (위반 시: OOM 발생, DB 서버 다운)
- [ ] autovacuum = on -- 필수 (위반 시: Dead Tuple 누적, 성능 점진 저하)
- [ ] autovacuum_vacuum_cost_limit >= 1000 -- 기본값(200) 대비 상향 (위반 시: VACUUM 처리 지연)
- [ ] idle_in_transaction_session_timeout 설정 -- 교착 방지 (위반 시: Lock 점유로 인한 전파 장애)
- [ ] WAS maxLifetime < PgPool child_life_time < DB idle_session_timeout -- 엄격 부등호 (위반 시: 레이스 컨디션)
- [ ] random_page_cost = 1.1 (SSD 환경) -- SSD 환경 필수 (위반 시: 비효율적 실행 계획)
- [ ] Replication Slot 구성 확인 -- pg_replication_slots (위반 시: Slot 누적 시 디스크 Full 위험)
- [ ] vm.swappiness = 1 (PostgreSQL 서버) -- DB 서버 안정성 (위반 시: 스와핑 시 쿼리 성능 급감)
- [ ] vm.overcommit_memory = 2 (PostgreSQL 서버) -- OOM Killer 방지 (위반 시: postmaster 강제 종료 → 전체 장애)
- [ ] vm.overcommit_ratio = 90 (PostgreSQL 서버) -- overcommit_memory=2 시 커밋 한도 보장 (위반 시: "Cannot allocate memory" 장애)
- [ ] THP = disabled (PostgreSQL 서버) -- OLTP 지연 스파이크 방지 (위반 시: 간헐적 수백 ms 쿼리 지연)
- [ ] THP 영구 설정 적용 -- IT ONE 변경 요청 완료 여부 확인 (미적용 시: 리부팅 후 THP 활성화로 성능 저하)
- [ ] systemd 서비스 LimitNOFILE/LimitNPROC override 설정 -- 서비스 데몬 ulimit 적용 (미적용 시: limits.conf 무시되어 기본값 1024로 동작)
- [ ] Sum(maxPoolSize) <= DB max_conn * 0.7 (직접 연결 시) -- 70% Ceiling Rule (위반 시: 타 서비스 커넥션 고갈)

---

## 5. 모니터링 체크리스트

| 모니터링 항목 | 조회 방법 | 경고(Warning) | 위험(Critical) | 조치 |
|:---|:---|:---|:---|:---|
| Active Sessions | `SELECT count(*) FROM pg_stat_activity WHERE state = 'active'` | max_connections 70% | max_connections 85% | 커넥션 풀 설정 재검토 |
| Slow Query | `pg_stat_statements` (>= 1s) | 발생 시 | 빈발 시 | 인덱스 추가 / 쿼리 튜닝 |
| Replication Lag | `SELECT now() - pg_last_xact_replay_timestamp() FROM pg_stat_replication` | > 5s | > 30s | 네트워크/부하 점검, Replica 증설 검토 |
| Dead Tuples | `SELECT n_dead_tup FROM pg_stat_user_tables` | 테이블 크기 10% | 테이블 크기 20% | VACUUM 강제 실행, autovacuum 조정 |
| Cache Hit Ratio | `SELECT sum(blks_hit)::float / NULLIF(sum(blks_hit + blks_read), 0) FROM pg_stat_database` | < 99% | < 95% | shared_buffers 증설 검토 |
| Lock Wait | `SELECT * FROM pg_locks WHERE NOT granted` | 대기 > 1s | 대기 > 5s | 트랜잭션 분석, lock_timeout 확인 |
| Autovacuum 진행 상태 | `SELECT * FROM pg_stat_progress_vacuum` | 장시간 미실행 | Dead Tuple 누적 | autovacuum_vacuum_cost_limit 상향 |

---

# PgPool-II 서버 설정 가이드

> **기준 문서**: `reports/final-standard-guide.md`
> **적용 범위**: PgPool-II (커넥션 풀링 + 로드 밸런싱 + 자동 페일오버)
> **연동**: PostgreSQL Streaming Replication (Primary / Replica)

---

## 0. 적용 전제

PgPool-II는 WAS의 비싼 DB 커넥션을 다중화하고, 읽기 분산·HA를 담당한다. 아래 토폴로지와 전제를 반드시 함께 확인한다.

```mermaid
graph LR
    subgraph WAS Layer
        W1[WAS-1<br/>HikariCP]
        W2[WAS-2<br/>HikariCP]
        W3[WAS-N<br/>HikariCP]
    end

    subgraph PgPool Layer
        PP1[PgPool-II<br/>Active]
        PP2[PgPool-II<br/>Standby]
    end

    subgraph PostgreSQL Layer
        PG_M[(PostgreSQL<br/>Primary)]
        PG_R[(PostgreSQL<br/>Replica)]
    end

    W1 --> VIP[Virtual IP]
    W2 --> VIP
    W3 --> VIP
    VIP --> PP1
    PP1 -->|Write / Read| PG_M
    PP1 -->|Read Only<br/>Load Balance| PG_R
    PP1 -.->|Watchdog<br/>SPOF 방지| PP2
    PG_M -->|WAL Streaming| PG_R
```

- **커넥션 풀링으로 70% Ceiling 대체**: PgPool 환경에서는 직접 연결의 70% Ceiling Rule이 PgPool-II의 연결 풀링으로 대체됨. WAS 풀 합산이 num_init_children을 초과해도 PgPool Listen Queue가 초과분을 안전하게 흡수
- **타임아웃 캐스케이드**: `WAS maxLifetime(27min) < PgPool child_life_time(28min) < PG idle_session_timeout(30min) <= 방화벽 최단(30min)` (방화벽은 30~60min 범위, 최단 30min 기준 설계. keepalive 60s로 잔여 레이스 방어)
- **kernel.sem 선행 필수**: `num_init_children=120` 구동 시 세마포어 상한 필수. 미설정 시 구동 실패

---

## 1. OS 커널 설정

### 1.1 공통 파라미터 (모든 서버 적용)

```ini
# /etc/sysctl.d/99-infra-common.conf -- 모든 서버 공통
fs.file-max = 2097152
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 5
```

```bash
# /etc/security/limits.d/99-infra-common.conf -- 모든 서버 공통 (PAM 기반 적용)
*  soft  nofile  1048576
*  hard  nofile  1048576
*  soft  nproc   65536
*  hard  nproc   65536
```

> **systemd 서비스 필수 추가 설정**: 위 limits.conf는 PAM 기반 세션 접속에만 적용되며, **systemd가 관리하는 서비스 데몬에는 적용되지 않음**. 아래 systemd drop-in override를 반드시 추가 적용해야 함.

```ini
# /etc/systemd/system/pgpool.service.d/override.conf
# 서비스명은 설치 방법에 따라 pgpool-II.service 일 수 있음
[Service]
LimitNOFILE=1048576
LimitNPROC=65536
```

```bash
# systemd 적용
systemctl daemon-reload
systemctl restart pgpool
```

| 파라미터 | 값 | 역할 |
|:---|:---|:---|
| fs.file-max | 2,097,152 | 시스템 전체 FD 상한. 대규모 동시 접속 시 Too many open files 방지 |
| net.core.somaxconn | 4,096 | OS 커널 TCP Listen Backlog. 트래픽 스파이크 시 패킷 Drop 방지 |
| net.ipv4.tcp_max_syn_backlog | 4,096 | SYN Queue 상한. somaxconn과 세트로 설정 |
| net.ipv4.tcp_keepalive_time | 300 (5분) | TCP Keepalive 최초 대기 시간. 기본 7,200초 단축 |
| net.ipv4.tcp_keepalive_intvl | 30 | Keepalive 프로브 재전송 간격 |
| net.ipv4.tcp_keepalive_probes | 5 | 연속 실패 시 dead 판정. 300+30x5=450초 내 확정 |
| ulimit -n (nofile) | 1,048,576 | 프로세스당 FD 한도. infinity 시 커널 ~8.6GB 할당 (Bug 2394600) |
| ulimit -u (nproc) | 65,536 | 프로세스/스레드 수 상한. Fork Bomb 방지 |

### 1.2 PgPool-II 서버 전용 파라미터

```ini
# /etc/sysctl.d/99-pgpool-tuning.conf -- PgPool-II 서버 전용
vm.swappiness = 10
kernel.sem = 250 32000 250 128
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_tw_reuse = 1
net.ipv4.ip_local_port_range = 32768 65535
```

| 파라미터 | 값 | 역할 |
|:---|:---|:---|
| kernel.sem | 250 32000 250 128 | child 프로세스당 System V 세마포어. SEMOPM(3번째) 32->250 상향. num_init_children=120 구동 시 세마포어 병목 해소. 형식: SEMMSL SEMMNS SEMOPM SEMMNI |
| vm.swappiness | 10 | PgPool은 WAS와 유사한 네트워크 프록시 역할. JVM과 동일한 수준 적용 |
| tcp_fin_timeout | 15 | PgPool->PostgreSQL 연결 종료 후 TIME_WAIT 소켓 신속 정리 |
| tcp_tw_reuse | 1 | TIME_WAIT 소켓 재사용으로 포트 고갈 방지 |
| ip_local_port_range | 32768~65535 | PgPool이 WAS로부터 커넥션 받아 PostgreSQL로 아웃바운드 연결. WAS와 동일 적용 |

> **PgPool-II + PostgreSQL 병설 서버**: 양쪽 파라미터 모두 적용 필요. 단, vm.swappiness는 충돌하므로 DB 서버 기준인 1을 우선 적용.
>
> **4GB RAM 독립 서버**: num_init_children=120 구동 시 프로세스 메모리 점유율(약 1GB)은 안정 범위이나, kernel.sem 설정이 선행되어야 함.

### 1.3 적용 명령어

```bash
sysctl --load /etc/sysctl.d/99-infra-common.conf
sysctl --load /etc/sysctl.d/99-pgpool-tuning.conf
systemctl daemon-reload
systemctl restart pgpool
```

---

## 2. PgPool-II 설정

### 2.1 PgPool-II 전용 파라미터

| 파라미터 | 표준값 | 역할 |
|:---|:---|:---|
| num_init_children | 120 | 동시 클라이언트 연결(프로세스) 수. PgPool 연결 풀링으로 실제 백엔드 동시 연결은 100 이하 유지. 초과 클라이언트는 Listen Queue에서 대기 |
| max_pool | 1 (단일 DB/계정) | child 프로세스당 DB 연결 수. 불필요한 상향 시 백엔드 연결 기하급수적 증가 |
| child_life_time | 1,680 (28min) | child 프로세스 최대 생존 시간. DB idle_session_timeout(30min)보다 짧게 설정 |
| connection_life_time | 1,680 (28min) | PgPool->PostgreSQL 백엔드 연결 최대 수명. DB 세션 타임아웃(30min)보다 짧게 |
| client_idle_limit | 600 (10min) | 클라이언트(WAS) 유휴 최대 시간. 좀비 커넥션이 PgPool 프로세스 점유 방지 |
| reserved_connections | 1 | PgPool 관리자 접속 예약 슬롯. 장애 시 DBA 접속 불가 상황 방지 |
| load_balance_mode | on | 읽기 쿼리(SELECT)를 Replica로 자동 분산. INSERT/UPDATE/DELETE는 항상 Primary |
| backend_clustering_mode | 'streaming_replication' | v4.x+ 스트리밍 복제 모드. 기존 master_slave_mode는 폐지 |
| backend_weight0 / weight1 | Primary 1 / Replica 3 | 읽기 쿼리 분산 비율. Primary는 쓰기 전담 + 최소 읽기, Replica에 읽기 부하 집중 (1:3 = 25%:75%) |

> num_init_children=120은 PgPool 공식 권고 공식(max_pool x num_init_children <= max_connections - superuser_reserved)을 초과(1x120 > 97). 단, 연결 풀링으로 실제 동시 백엔드 연결은 100 이하 유지됨.
>
> **쿼리 취소(cancellation) 시 연결 2배 소모 주의**: PgPool 공식은 쿼리 취소 발생 시 max_pool x num_init_children x 2 <= max_connections - superuser_reserved 를 요구. 피크 + 쿼리 취소 겹칠 시 PostgreSQL "too many clients already" → failover 트리거 위험.
>
> **운영 가드 (2026-07-02 TA 확정, 120 유지 조건)**: (1) 피크 타임 `SHOW POOL_PROCESSES` 모니터링 필수 (2) 쿼리 취소 빈도 추적 — statement_timeout/lock_timeout 가드레일로 취소 최소화 (3) "too many clients already" 발생 시 failover 트리거될 수 있음을 운영 Runbook에 명시.

### 2.2 pgpool.conf 전문

```conf
# -------------------------------------------------------
# Connection Pooling
# -------------------------------------------------------
num_init_children = 120           # DBA 운영 권장값 (PgPool 연결 풀링으로 백엔드 100 이하 유지)
max_pool = 1                     # 단일 DB/단일 계정
child_life_time = 1680           # 28min
connection_life_time = 1680      # 28min
client_idle_limit = 600          # 10min
reserved_connections = 1         # 관리 접속 보장

# -------------------------------------------------------
# Load Balancing
# -------------------------------------------------------
load_balance_mode = on
backend_clustering_mode = 'streaming_replication'
backend_weight0 = 1              # Primary (쓰기 전담 + 최소 읽기)
backend_weight1 = 3              # Replica (읽기 부하 집중)

# -------------------------------------------------------
# Health Check
# -------------------------------------------------------
health_check_period = 30
health_check_timeout = 10
health_check_max_retries = 3

# -------------------------------------------------------
# Watchdog (SPOF 방지)
# -------------------------------------------------------
use_watchdog = on
wd_hostname = 'pgpool-node1'
wd_vip = '10.0.0.100'

# -------------------------------------------------------
# Auto Failover
# -------------------------------------------------------
failover_command = '/etc/pgpool-II/failover.sh'
```

---

## 3. 타임아웃 & 커넥션 캐스케이드

### 3.1 타임아웃 캐스케이드 (WAS -> PgPool -> PostgreSQL)

```
WAS HikariCP maxLifetime (1,620,000ms = 27min)
     |
     |  maxLifetime < child_life_time < idle_session_timeout
     v
PgPool-II child_life_time (1,680s = 28min)
     |
     |  child_life_time < idle_session_timeout
     v
PostgreSQL idle_session_timeout (1,800,000ms = 30min)
```

> 엄격한 부등호(<)로 계층 간 타임아웃 격리. HikariCP가 PgPool보다 1분, PgPool이 DB보다 2분 먼저 커넥션 폐기.

### 3.2 PgPool-II 경유 커넥션 풀 가이드

```
PgPool-II 공식 권고 (참고):
  max_pool x num_init_children <= (max_connections - superuser_reserved_connections)
  -> 공식 준수 시: 1 x 97 <= (100 - 3) = 97

DBA 운영 권장 (적용값):
  num_init_children = 120
  -> 공식 초과(1 x 120 > 97)이나, PgPool-II 연결 풀링(커넥션 캐싱 및 재사용)으로
     실제 동시 백엔드 연결은 100 이하로 유지됨

WAS -> PgPool 계층:
  WAS 전체 풀 합산(80~100) <= num_init_children(120)
  -> 초과 클라이언트는 PgPool Listen Queue에서 대기 후 순차 처리
```

> PgPool-II 환경에서는 직접 연결의 70% Ceiling Rule이 PgPool-II의 연결 풀링으로 대체됨. WAS 풀 합산이 num_init_children을 초과해도 PgPool Listen Queue가 초과분을 안전하게 흡수.

---

## 4. 검증 체크리스트

- [ ] SHOW POOL_PROCESSES 피크 활성 연결 < 100 -- 최악 시나리오 방지 (위반 시: "too many clients already" + failover 트리거)
- [ ] reserved_connections >= 1 -- 관리 접속 보장 (위반 시: 장애 시 DBA 접속 불가)
- [ ] max_pool = 1 (단일 DB/계정) -- 불필요한 커넥션 폭증 방지 (위반 시: 백엔드 연결 기하급수적 증가)
- [ ] Watchdog 활성화 (use_watchdog = on) -- SPOF 방지 (위반 시: PgPool 단일 구성 시 전체 장애)
- [ ] WAS maxLifetime < PgPool child_life_time < DB idle_session_timeout -- 엄격 부등호 (위반 시: 레이스 컨디션)
- [ ] kernel.sem = "250 32000 250 128" -- 세마포어 병목 해소 (위반 시: PgPool 기동 시 "could not create semaphore set" 에러)
- [ ] load_balance_mode = on -- 읽기 분산 활성화 (위반 시: 모든 읽기가 Primary로 집중되어 과부하)
- [ ] backend_clustering_mode = 'streaming_replication' -- v4.x+ 표준 모드 (위반 시: 기존 master_slave_mode는 폐지됨)
- [ ] failover_command 지정 -- 자동 페일오버 스크립트 (위반 시: Primary 다운 시 수동 개입 필요)
- [ ] child_life_time < DB idle_session_timeout -- 타임아웃 캐스케이드 준수 (위반 시: DB 강제 차단 전 PgPool이 먼저 회수 불가)
- [ ] systemd 서비스 LimitNOFILE/LimitNPROC override 설정 -- 서비스 데몬 ulimit 적용 (미적용 시: limits.conf 무시되어 기본값 1024로 동작)

---

## 5. 모니터링 체크리스트

| 모니터링 항목 | 조회 방법 | 경고(Warning) | 위험(Critical) | 조치 |
|:---|:---|:---|:---|:---|
| PgPool 커넥션 사용률 | `SHOW POOL_NODES` + `SHOW POOL_PROCESSES` | 사용률 > 80% | 사용률 > 95% | num_init_children 증설 검토 |
| 쿼리 취소 빈도 | `SHOW POOL_PROCESSES` + PgPool 로그(cancellation) | 빈발 시 | 피크 + 취소 겹침 | statement_timeout/lock_timeout 가드레일 점검. "too many clients already" → failover 위험 |
| 피크 활성 백엔드 연결 수 | `SHOW POOL_PROCESSES` | 80+ 동시 활성 | 100 도달 | "too many clients already" 위험. 트래픽 분산 또는 num_init_children 조정 |

> 피크 타임 백엔드 연결 수 주기적 모니터링 필수. 120개 child 프로세스가 동시에 백엔드 연결을 요구하는 극단적 피크 시 PostgreSQL이 "too many clients already" 반환 + failover 트리거 가능.

---

## 부록. PgPool 산출 예시 (플랫폼개발팀 나이스M 기준)

> 특정 팀 사례로, 산정 방법론의 참고용. 전사 표준값은 본문 §2 참조.

| 항목 | 산출 수치 | 비고 |
|:---|:---|:---|
| 대상 서비스 | 나이스M (Nice M) | PostgreSQL(via PgPool-II) + MongoDB 운영 |
| 총 WAS 인스턴스 수 | 4개 (이중화) | WAS 표준 설정 기준 |
| 전체 WAS 풀 합산 | 80~100개 | 인스턴스당 20~25 |
| PgPool num_init_children | 120 | DBA 운영 권장값. 연결 풀링으로 백엔드 동시 연결 100 이하 유지 |
| PgPool max_pool | 1 | 단일 DB/단일 계정 |
| PgPool -> PG 최대 백엔드 연결 | 100개 (이론적 상한) | 연결 풀링으로 실제 동시 점유는 100 이하 |
| PG max_connections | 100 | OOM 예방 100 고정 |
| WAS 풀(80~100) <= num_init_children(120) | 정상 | 초과 클라이언트는 Listen Queue 대기 후 순차 처리 |

---

# MongoDB 서버 설정 가이드

> **기준 문서**: `reports/final-standard-guide.md`
> **적용 범위**: MongoDB 8.0+ (WiredTiger 스토리지 엔진)
> **프로덕션 표준**: Replica Set PSS (Primary 1 + Secondary 2)
> **개발/테스트**: Standalone

---

## 0. 적용 전제

프로덕션 표준 아키텍처는 Replica Set PSS. 아래 토폴로지와 전제를 반드시 함께 확인한다.

```mermaid
graph LR
    APP[WAS<br/>MongoDB Driver] --> MG_P[(mongod<br/>Primary)]
    MG_P -->|Oplog<br/>Replication| MG_S1[(mongod<br/>Secondary 1)]
    MG_P -->|Oplog<br/>Replication| MG_S2[(mongod<br/>Secondary 2)]
    MG_S1 -.Read<br/>Optional.-> APP
    MG_S2 -.Read<br/>Optional.-> APP
```

#### PSS vs PSA 비교

| 기준 | PSS (표준) | PSA (금지) |
|:---|:---|:---|
| 구성 | Primary 1 + Secondary 2 | Primary 1 + Secondary 1 + Arbiter 1 |
| 데이터 복제 | 3중 복제 | 2중 복제 (Arbiter는 데이터 미보관) |
| 데이터 안전성 | 1노드 장애까지 정상 서비스 유지 | Secondary 1 노드 장애 시 과반수 미달로 w:majority 쓰기 불가 (stall 장애) |

> **PSA 구조 치명적 제약**: 정산, 결제 등 트랜잭션 정합성이 필수인 도메인에는 PSA 구성 절대 금지. 반드시 PSS 구성 준수.

- **70% Ceiling (WAS 직접 연결 시)**: `Sum(모든 WAS 인스턴스 maxPoolSize) <= maxIncomingConnections * 0.7`. 단 MongoDB는 기본 65,536으로, 사내 표준은 RAM별 차등(1,000~10,000) 적용
- **방화벽 TCP 30~60min (범위, 최단 30min 기준 설계)**: 모든 타임아웃의 최상위. WAS maxLifetime(27min) < MongoDB connectionPool maxIdleTimeMS(30min) <= 방화벽 최단(30min). keepaliveTime(60s)이 주기적 ping으로 잔여 레이스 방어
- **PostgreSQL과 MongoDB는 동일 호스트 병설 금지**: `vm.overcommit_memory` 설정이 PostgreSQL(2)과 MongoDB 8.0(1)에서 서로 충돌. 반드시 물리적으로 분리된 서버에서 운영

---

## 1. OS 커널 설정

### 1.1 공통 파라미터 (모든 서버 적용)

```ini
# /etc/sysctl.d/99-infra-common.conf -- 모든 서버 공통
fs.file-max = 2097152
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 5
```

```bash
# /etc/security/limits.d/99-infra-common.conf -- 모든 서버 공통 (PAM 기반 적용)
*  soft  nofile  1048576
*  hard  nofile  1048576
*  soft  nproc   65536
*  hard  nproc   65536
```

> **systemd 서비스 필수 추가 설정**: 위 limits.conf는 PAM 기반 세션 접속에만 적용되며, **systemd가 관리하는 서비스 데몬에는 적용되지 않음**. 1.2절의 mongod.service drop-in override를 반드시 추가 적용해야 함.

| 파라미터 | 값 | 역할 |
|:---|:---|:---|
| fs.file-max | 2,097,152 | 시스템 전체 FD 상한. 대규모 동시 접속 시 Too many open files 방지 |
| net.core.somaxconn | 4,096 | OS 커널 TCP Listen Backlog. 트래픽 스파이크 시 패킷 Drop 방지 |
| net.ipv4.tcp_max_syn_backlog | 4,096 | SYN Queue 상한. somaxconn과 세트로 설정 |
| net.ipv4.tcp_keepalive_time | 300 (5분) | TCP Keepalive 최초 대기 시간. 기본 7,200초 단축 |
| net.ipv4.tcp_keepalive_intvl | 30 | Keepalive 프로브 재전송 간격 |
| net.ipv4.tcp_keepalive_probes | 5 | 연속 실패 시 dead 판정. 300+30x5=450초 내 확정 |
| ulimit -n (nofile) | 1,048,576 | 프로세스당 FD 한도. infinity 시 커널 ~8.6GB 할당 (Bug 2394600) |
| ulimit -u (nproc) | 65,536 | 프로세스/스레드 수 상한. Fork Bomb 방지 |

### 1.2 MongoDB 서버 전용 파라미터 (8.0+ 기준)

> **Kernel 6.19 주의**: MongoDB 8.0.4 미만 버전에서 Linux Kernel 6.19 구동 시 알려진 오류가 있음. **MongoDB 8.0.4 이상 사용 권장**(공식 문서 확인). 커널 6.19 환경에서 8.0.4 미만 사용 금지.

```ini
# /etc/sysctl.d/99-mongodb-tuning.conf -- MongoDB 서버 전용
vm.swappiness = 1
vm.overcommit_memory = 1
vm.dirty_background_ratio = 5
vm.dirty_ratio = 15
```

```bash
# THP (Transparent Huge Pages) 활성화 -- OS 리부팅 시 초기화되는 1회성 명령임.
# 영구 설정은 root 권한이 필요하므로 IT ONE을 통해 IT 운영실에 변경 요청할 것.
#
# [참고: 영구 설정 방법 -- IT 운영실 적용용]
# 방법 1 (권장): GRUB 커널 파라미터 (리부팅 필요)
#   grubby --update-kernel=ALL --args="transparent_hugepage=always"
#
# 방법 2: TuneD 프로파일
#   /etc/tuned/<profile>/tuned.conf 에 [vm] transparent_hugepages=always 설정
```

```bash
# /etc/security/limits.d/99-mongodb.conf -- MongoDB 추가 ulimit (PAM 기반)
# (nofile/nproc는 1.1 공통 limits와 동일하므로 생략, MongoDB 특수 항목만)
mongod  soft  fsize    unlimited
mongod  hard  fsize    unlimited
mongod  soft  cpu      unlimited
mongod  hard  cpu      unlimited
```

```ini
# /etc/systemd/system/mongod.service.d/override.conf -- MongoDB 서비스 데몬 ulimit
[Service]
LimitNOFILE=1048576
LimitNPROC=65536
LimitFSIZE=infinity
LimitCPU=infinity
Environment=GLIBC_TUNABLES=glibc.pthread.rseq=0
```

> **TCMalloc per-CPU cache 정상 동작 조건 (MongoDB 8.0+)**: THP always 외에 (1) **커널 4.18+**, (2) **glibc rseq 비활성**(`GLIBC_TUNABLES=glibc.pthread.rseq=0`, 위 systemd 환경변수)이 함께 충족되어야 per-CPU cache가 정상 작동. rseq를 끄지 않으면 glibc가 rseq를 선점해 TCMalloc per-CPU cache가 비활성화되어 THP 활성화의 성능 이점이 반감됨.

```bash
# systemd 적용
systemctl daemon-reload
systemctl restart mongod
```

| 파라미터 | 값 | 역할 |
|:---|:---|:---|
| vm.swappiness | 1 | WiredTiger 캐시(cacheSizeGB)가 디스크로 내려가면 성능 급감. 스왑 거의 허용 안 함 |
| vm.overcommit_memory | 1 | MongoDB 8.0의 TCMalloc per-CPU 캐시가 정상 동작하려면 오버커밋 항상 허용 필요. PostgreSQL(2)과 충돌 |
| Transparent Huge Pages | enabled (always) | MongoDB 8.0부터 TCMalloc per-CPU 캐시가 THP 활용하여 성능 향상. 7.0 이하와 방향 전환됨 |
| vm.dirty_background_ratio | 5 | 더티 페이지 5% 도달 시 백그라운드 플러시. WiredTiger 체크포인트와 커널 플러시 충돌 완화 |
| vm.dirty_ratio | 15 | 동기 플러시 임계치. PostgreSQL(10)보다 높게 설정 (WiredTiger 자체 쓰기 스케줄링 존재) |
| ulimit -n (nofile) | 1,048,576 | 모든 서버 공통값 (MongoDB 공식 최소 64,000 이상 충족) |
| ulimit -f / -t | unlimited | 파일 크기 및 CPU 시간 제한 해제. 대용량 데이터 처리 중 파일 크기 제한 도달 시 데이터 손상 위험 |

> **PostgreSQL과 MongoDB는 동일 호스트 병설 금지**: vm.overcommit_memory 설정이 PostgreSQL(2)과 MongoDB 8.0(1)에서 서로 충돌. 반드시 물리적으로 분리된 서버에서 운영.

### 1.3 적용 명령어

```bash
sysctl --load /etc/sysctl.d/99-infra-common.conf
sysctl --load /etc/sysctl.d/99-mongodb-tuning.conf
# THP 영구 설정은 IT ONE을 통해 IT 운영실에 변경 요청 (상세는 1.2절 참조)
systemctl daemon-reload
systemctl restart mongod
```

---

## 2. MongoDB 설정

### 2.1 핵심 파라미터

| 파라미터 | 표준값 | 역할 |
|:---|:---|:---|
| cacheSizeGB | `0.5 * (RAM - 1)`, 32GB+는 하향 조정 | WiredTiger 내부 캐시 크기(GB). DB 전용 서버: RAM에서 1GB 제외한 50%. 과다 설정 시 OS 메모리 부족으로 스와핑 |
| replSetName | rs0 (환경에 맞게 명명) | Replica Set 식별자. 클러스터 내 모든 노드가 동일한 이름 사용 |
| Profiling Level | 1 (slowms: 100) | 슬로우 쿼리 및 COLLSCAN 감지. 100ms 이상 연산만 기록 |
| electionTimeoutMillis | 10000 (10s, 기본값) | Primary 하트비트 수신 불가 시 선거(Election) 시작 대기 시간 |
| defaultMaxTimeMS | 60000 (권장) | MongoDB 8.0 신규. 개별 읽기 연산 기본 시간 제한(ms). 장기 실행 쿼리 자원 독점 방지 |
| maxIncomingConnections | RAM별 차등 (1,000~10,000) | 최대 동시 클라이언트 연결. 커넥션당 1MB 스레드 스택. 기본(65536)은 소형 서버에서 OOM 위험 |
| internalQueryExecBlockingSortBytes | RAM별 차등 (32~256 MB) | 인덱스 없는 블로킹 정렬 시 세션당 최대 메모리. 악성 쿼리 1개 시스템 전체 고갈 방지 |

> **defaultMaxTimeMS는 cluster parameter**(`setClusterParameter`로 설정, **mongod.conf 항목 아님**). 적용 예시:
> ```javascript
> db.adminCommand({ setClusterParameter: { defaultMaxTimeMS: { readOperations: 60000 } } })
> ```

### 2.2 Write Concern / Read Preference 의사결정표

| 서비스 유형 | writeConcern | readPreference | 사유 |
|:---|:---|:---|:---|
| 정산/결제 (정합성 필수) | w: majority | primary | 데이터 유실 허용 불가, 항상 최신 데이터 보장 |
| 일반 상용 (HA 필요) | w: 1 (명시 설정 시) | primary | 기본 안정성 확보 |
| 조회성 (Replication Lag 허용) | w: 1 | secondaryPreferred | 읽기 부하 분산 |
| 대시보드/통계 (실시간성 낮음) | w: 1 | secondary | Primary 읽기 부하 제로화 |

> 핵심 제약: 정산/결제 서비스는 반드시 primary readPreference 유지.
>
> MongoDB 5.0+ 기본 Write Concern: PSS 구성에서 클라이언트가 명시하지 않으면 implicit default = `w: majority`. (arbiter 포함/데이터 보유 투표 멤버 부족 시에만 `w: 1` fallback.)
>
> MongoDB 8.0 Write Concern 동작 변경: w:majority 설정 시 8.0부터 majority 노드가 oplog 엔트리를 write한 시점에 ack 반환 (기존: 적용 완료 후 ack). 성능 향상.

### 2.3 RAM별 파라미터 매트릭스 (노드당)

| DB 서버 RAM | cacheSizeGB | maxIncomingConnections | internalQueryExecMaxBlockingSortBytes | 비고 |
|:---:|:---:|:---:|:---:|:---|
| 8 GB | 3.5 GB | 1,000 | 32 MB | PSS 3노드 각각 동일 적용 |
| 16 GB | 7.5 GB | 2,000 | 64 MB | 표준 프로덕션 |
| 32 GB | 12.0 GB | 5,000 | 128 MB | 고성능. cacheSizeGB 하향 (OS page cache 마진) |
| 64 GB | 24.0 GB | 10,000 | 256 MB | 대규모. cacheSizeGB 하향 (대량 커넥션 + page cache 마진) |

### 2.4 mongod.conf 전문 (프로덕션, 8GB 기준)

```yaml
# -------------------------------------------------------
# Storage (8GB DB 전용 서버 기준)
# -------------------------------------------------------
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 3.5            # 0.5 * (8 - 1) = 3.5GB

# -------------------------------------------------------
# Query Settings
# -------------------------------------------------------
setParameter:
  internalQueryExecMaxBlockingSortBytes: 33554432  # 32MB (8GB RAM 기준)

# -------------------------------------------------------
# Replica Set
# -------------------------------------------------------
replication:
  replSetName: rs0                # Replica Set 명

# -------------------------------------------------------
# Profiling (COLLSCAN 감지 필수)
# -------------------------------------------------------
operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100

# -------------------------------------------------------
# Network
# -------------------------------------------------------
net:
  port: 27017
  bindIp: 0.0.0.0
  maxIncomingConnections: 1000    # 8GB RAM 기준 (커넥션당 1MB 스레드 스택)

# -------------------------------------------------------
# Security (프로덕션 필수)
# -------------------------------------------------------
security:
  keyFile: /etc/mongodb/keyfile   # 멤버 간 인증
  authorization: enabled          # 클라이언트 인증

# -------------------------------------------------------
# Logging
# -------------------------------------------------------
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
```

> 각 RAM 스펙별로 cacheSizeGB, maxIncomingConnections, internalQueryExecMaxBlockingSortBytes 값을 2.3절 매트릭스에 맞게 변경. 나머지 설정은 모든 RAM 공통.
> defaultMaxTimeMS는 mongod.conf 항목이 아님(cluster parameter). 2.1절 적용 예시 참조.

### 2.5 Replica Set 초기화 (mongosh)

```javascript
// Primary에서 실행
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo-primary:27017", priority: 2 },
    { _id: 1, host: "mongo-secondary1:27017", priority: 1 },
    { _id: 2, host: "mongo-secondary2:27017", priority: 1 }
  ]
})

// 초기화 완료 후 Profiling 설정
db.setProfilingLevel(1, { slowms: 100 })

// defaultMaxTimeMS cluster parameter 설정 (8.0+)
db.adminCommand({ setClusterParameter: { defaultMaxTimeMS: { readOperations: 60000 } } })

// Write Concern / Read Preference 설정 (연결 문자열 예시)
// mongodb://user:pass@mongo-primary:27017,mongo-secondary1:27017,mongo-secondary2:27017/?replicaSet=rs0&w=majority&readPreference=primary
```

### 2.6 개발/테스트: Standalone

> 본 구성은 개발 및 테스트 환경에 한해서만 허용. 프로덕션에서는 Replica Set PSS 구성(0절 아키텍처) 적용 필수. MongoDB Standalone은 멀티 도큐먼트 트랜잭션을 지원하지 않음.

#### RAM별 파라미터 매트릭스 (Standalone)

| DB 서버 RAM | cacheSizeGB | maxIncomingConnections | internalQueryExecMaxBlockingSortBytes | 비고 |
|:---:|:---:|:---:|:---:|:---|
| 8 GB | 3.5 GB | 1,000 | 32 MB | 개발/테스트 |
| 16 GB | 7.5 GB | 2,000 | 64 MB | 개발/테스트 |
| 32 GB | 12.0 GB | 5,000 | 128 MB | 프로토타입 (RS 전환 계획 필수) |
| 64 GB | 24.0 GB | 10,000 | 256 MB | 프로토타입 (RS 전환 계획 필수) |

#### mongod.conf (개발/테스트용, 8GB 기준)

```yaml
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 3.5            # 0.5 * (8 - 1)

setParameter:
  internalQueryExecMaxBlockingSortBytes: 33554432  # 32MB (8GB RAM 기준)

net:
  port: 27017
  bindIp: 0.0.0.0
  maxIncomingConnections: 1000    # 8GB RAM 기준

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true

operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100
```

> Standalone은 replication 섹션을 설정하지 않음. 사용자 증가가 예상되는 프로토타입은 Replica Set 전환 계획 수립 필수.

---

## 3. 타임아웃 & 커넥션 캐스케이드

### 3.1 타임아웃 캐스케이드 (WAS -> MongoDB)

```
WAS HikariCP / MongoDB Driver maxLifetime (1,620,000ms = 27min)
     |
     v
MongoDB connectionPool maxIdleTimeMS (1,800,000ms = 30min)
     |
     v
MongoDB driver socketTimeoutMS (0 = 무제한, 애플리케이션 레벨 제어)
```

> MongoDB는 논리 세션(localLogicalSessionTimeoutMinutes, 30min)과 물리 커넥션(Driver maxIdleTimeMS)의 이중 구조. HikariCP maxLifetime(27min)은 두 계층 모두보다 짧게 유지.
>
> **maxIdleTimeMS 설정 위치**: mongod.conf 항목이 **아님**. MongoDB **드라이버(WAS 측) 연결 풀 설정**이다.
> - HikariCP(JDBC) 환경: maxIdleTimeMS 대신 **HikariCP maxLifetime(27min)** 이 실제 컨트롤 (was.md §3.1 참조)
> - MongoDB Java Driver 직접 연동 시:
>   ```java
>   MongoClientSettings.builder()
>     .applyToConnectionPoolSettings(b -> b.maxIdleTime(30, TimeUnit.MINUTES))
>     .build()
>   ```
>   또는 연결 문자열: `mongodb://.../?maxIdleTimeMS=1800000`

### 3.2 유휴 세션 제한

| 파라미터 | 표준값 | 역할 |
|:---|:---|:---|
| localLogicalSessionTimeoutMinutes | 30 (30min) | 서버 내 논리 세션(Logical Session) 유휴 만료. 세션 누적으로 인한 서버 자원 고갈 방지 |

> localLogicalSessionTimeoutMinutes는 논리 세션 만료 규격일 뿐, 물리적 TCP 커넥션 유휴 상태를 직접 관리하지 않음. 물리적 유휴 커넥션은 드라이버단 maxIdleTimeMS로 제어. HikariCP maxLifetime 및 keepaliveTime과 연동하여 커넥션 풀 수준에서 관리 필요.

### 3.3 공유 DB 커넥션 풀 가이드 (직접 연결 시)

```
maxIncomingConnections = RAM별 차등 (1,000~10,000)
      |
      +-- 30% 예약 (관리자, 모니터링, 긴급 접속)
      |
      +-- 70% 가용: 애플리케이션 커넥션 풀 전체 합산 상한 (maxIncomingConnections * 0.7)
```

> 절대 제약 (직접 연결 시): 모든 애플리케이션의 maxPoolSize 합산값은 maxIncomingConnections * 0.7을 초과 불가.

---

## 4. 검증 체크리스트

- [ ] Replica Set >= 3노드 (PSS 표준) -- Quorum 보장 (위반 시: 2노드 시 Primary 선출 불가)
- [ ] 정산/결제 서비스 readPreference = primary -- 정합성 보장 (위반 시: Secondary 읽기 시 과거 데이터 반환)
- [ ] Profiling Level >= 1 (slowms: 100) -- COLLSCAN 감지 필수 (위반 시: 인덱스 누락 무감지)
- [ ] defaultMaxTimeMS 설정 (8.0+, cluster parameter) -- 장기 실행 쿼리 방어 (미설정 시: 0=timeout 없음)
- [ ] Oplog Size 확인 (db.printReplicationInfo) -- 복제 안정성 (위반 시: Oplog 부족 시 복제 중단)
- [ ] electionTimeoutMillis >= 10000 -- 기본값 유지 (위반 시: 과단축 시 불필요한 페일오버 빈번)
- [ ] Cache Hit Ratio >= 95% (db.serverStatus().wiredTiger.cache) -- 캐시 효율성 (위반 시: 디스크 I/O 증가, 성능 저하)
- [ ] Sum(maxPoolSize) <= maxIncomingConnections * 0.7 (직접 연결 시) -- 70% Ceiling Rule (위반 시: 타 서비스 커넥션 고갈)
- [ ] vm.swappiness = 1 (MongoDB 서버) -- DB 서버 안정성 (위반 시: 스와핑 시 캐시 성능 급감)
- [ ] THP = enabled (MongoDB 8.0+ 서버) -- TCMalloc per-CPU 성능 (위반 시: per-CPU 캐시 비활성화로 성능 저하)
- [ ] THP 영구 설정 적용 -- IT ONE 변경 요청 완료 여부 확인 (미적용 시: 리부팅 후 THP 비활성화로 성능 저하)
- [ ] Kernel 6.19 환경에서는 MongoDB 8.0.4 이상 사용 -- 8.0.4 미만 Kernel 6.19 구동 금지 (위반 시: 시작 시 crash)
- [ ] ulimit -n = 1048576 (MongoDB 서버) -- 모든 서버 공통 (위반 시: 기동 경고, 커넥션 제한)
- [ ] systemd 서비스 LimitNOFILE/LimitNPROC/LimitFSIZE/LimitCPU override 설정 -- 서비스 데몬 ulimit 적용 (미적용 시: limits.conf 무시되어 기본값으로 동작)
- [ ] ulimit -f / -t = unlimited (MongoDB 서버) -- 대용량 데이터 보호 (위반 시: 파일 크기 제한 도달 시 데이터 손상 위험)
- [ ] security.authorization = enabled -- 클라이언트 인증 (위반 시: 인증 없는 접근 허용)
- [ ] security.keyFile 설정 -- 멤버 간 인증 (위반 시: Replica Set 내 노드 간 인증 불가)

---

## 5. 모니터링 체크리스트

| 모니터링 항목 | 조회 방법 | 경고(Warning) | 위험(Critical) | 조치 |
|:---|:---|:---|:---|:---|
| Active Connections | `db.serverStatus().connections` | > 70% of maxIncoming | > 85% of maxIncoming | 커넥션 풀 설정 재검토 |
| COLLSCAN (컬렉션 스캔) | `db.system.profile.find({ "planSummary": "COLLSCAN" })` | 발생 시 | 빈발 시 | 즉시 인덱스 추가 |
| Slow Query | `db.system.profile.find({ millis: { $gt: 100 } })` | > 100ms | > 1s | 인덱스 추가 / 쿼리 튜닝 |
| Replication Lag | `rs.printSecondaryReplicationInfo()` | > 5s | > 30s | 네트워크/부하 점검 |
| Cache Hit Ratio | `db.serverStatus().wiredTiger.cache` | < 97% | < 95% | cacheSizeGB 증설 검토 |
| Oplog Window | `db.printReplicationInfo()` | < 1h | < 10min | Oplog Size 확장 필요 |
| Election 이벤트 | `rs.status()` | 발생 시 | 빈발 시 | 네트워크/부하 원인 분석 |

### 모니터링 구축 순서

| 단계 | 구축 항목 | 완료 기준 |
|:---:|:---|:---|
| 1 | Profiling Level 1 활성화 (slowms: 100) | slowOp 기록 확인 |
| 2 | COLLSCAN 감지 (system.profile에서 planSummary: COLLSCAN) | COLLSCAN 발생 시 알림 수신 |
| 3 | Replication Lag 알림 (rs.printSecondaryReplicationInfo) | Lag > 5s 시 알림 수신 |
| 4 | Active Session 임계치 알림 (db.serverStatus().connections.current) | maxIncoming 70% 도달 시 알림 수신 |
| 5 | Cache Hit Ratio 추이 (db.serverStatus().wiredTiger.cache) | < 95% 시 알림 수신 확인 |
