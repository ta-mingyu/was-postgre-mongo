# PostgreSQL Domain Rules

> PostgreSQL 및 PgPool-II 표준 설정 가이드라인 작업 시 에이전트가 반드시 준수해야 할 구동 규칙.
> 기준 산출물: `reports/db-standard-guide.md` (PostgreSQL + PgPool-II 섹션)

---

## 1. 도메인 스코프

### 대상 시스템

| 시스템 | 대상 팀 | 설정 파일 | 비고 |
| :--- | :--- | :--- | :--- |
| **PostgreSQL** | 플랫폼개발팀 | `postgresql.conf` | Master/Replica 스트리밍 복제 |
| **PgPool-II** | 플랫폼개발팀 | `pgpool.conf` | 커넥션 풀링 + 읽기 분산 + 페일오버 |

### 아키텍처 기준

```
WAS (HikariCP) --> PgPool-II --> PostgreSQL (Master)
                                                  +--> PostgreSQL (Replica/Standby)
```

- HA 표준: 내장 스트리밍 복제(Streaming Replication) + PgPool-II 연계
- 읽기 분산: PgPool-II `load_balance_mode = on`

### 스코프 내 (IN Scope)

- PostgreSQL 메모리 파라미터 (shared_buffers, work_mem, effective_cache_size 등)
- WAL 및 체크포인트 설정
- autovacuum 설정
- 커넥션 및 타임아웃 설정
- 쿼리 플래너 튜닝 (SSD 환경)
- PgPool-II 커넥션 풀링, 로드 밸런싱, 페일오버

### 스코프 외 (OUT of Scope)

- WAS/JVM 설정 (WAS harness 참조)
- MongoDB 설정 (MongoDB harness 참조)
- DB2 내부 파라미터
- OS 인프라 튜닝 (WAS harness 참조)

---

## 2. 핵심 산정 공식 (절대 준수)

### 2.1 메모리 산정 (DB 전용 서버 기준)

```
shared_buffers = RAM * 0.25
effective_cache_size = RAM * 0.75  (실제 할당 아님, 플래너 참고값)
work_mem = (RAM - shared_buffers) / (max_connections * 3)  (상한선)
maintenance_work_mem = RAM * 0.03 ~ 0.0625  (PGTune 기준, 상한 0.0625)
wal_buffers = 16MB 고정 (또는 기본값 -1, 자동 계산 시 최대 16MB 캡)
```

> **64GB+ 대형 서버 권장**: `autovacuum_work_mem`을 maintenance_work_mem과 분리해 autovacuum_max_workers(3) 동시 실행 시 메모리 폭발(최대 3 × maintenance_work_mem)을 방지. 예: 64GB 서버 maintenance_work_mem=4GB + autovacuum_work_mem=1GB. (2026-07-02 TA 확정)

**RAM별 매트릭스 (DB 전용 서버)**:

| DB 서버 RAM | shared_buffers | effective_cache_size | work_mem | maintenance_work_mem | wal_buffers |
| :---: | :---: | :---: | :---: | :---: | :---: |
| 8 GB | 2 GB | 6 GB | 8 MB | 384 MB | 16 MB |
| 16 GB | 4 GB | 12 GB | 16 MB | 1 GB | 16 MB |
| 32 GB | 8 GB | 24 GB | 32 MB | 2 GB | 16 MB |
| 64 GB | 16 GB | 48 GB | 64 MB | 4 GB | 16 MB |

> work_mem 매트릭스는 운영 적용 표준값. 이론 상한 공식 `(RAM-shared_buffers)/(max_conn*3)`(kofemann/pgtune)보다 보수적(OLTP/PgPool 환경 최적화, RAM 2배마다 2배 패턴). 2026-07-02 TA 결정으로 reports/final과 정합.

### 2.2 WAL 및 체크포인트

```
max_wal_size: RAM ~16GB -> 2GB / 16~32GB -> 4GB / 32GB+ -> 16~32GB
min_wal_size = 1 GB (공통)
checkpoint_completion_target = 0.9 (공통)
max_wal_senders = 3 ~ 5 (Replica 수 + 여유)
wal_level = replica (스트리밍 복제 필수)
```

### 2.3 autovacuum 설정

| 파라미터 | 표준값 | 비고 |
| :--- | :--- | :--- |
| `autovacuum` | **on** | 비활성화 절대 금지 |
| `autovacuum_max_workers` | 3 ~ 5 | 5 이하 유지, 무작정 증설 금지 |
| `autovacuum_naptime` | 1 min | 점검 주기 |
| `autovacuum_vacuum_scale_factor` | 0.1 | Dead Tuple 10% 도달 시 VACUUM |
| `autovacuum_vacuum_cost_limit` | 1,000 ~ 2,000 | 기본값(200) 대비 상향 |

### 2.4 커넥션 산정

```
max_connections >= Sum(WAS maxPoolSize) * 1.5
superuser_reserved_connections = 3 (고정)
hot_standby = on (Replica 읽기 쿼리 수용 필수)
```

### 2.5 쿼리 플래너 (SSD 환경)

```
random_page_cost = 1.1  (HDD: 4.0)
effective_io_concurrency = 200  (HDD: 2)
```

---

## 3. PgPool-II 설정 기준

### 3.1 커넥션 풀링

| 파라미터 | 산정 기준 | 비고 |
| :--- | :--- | :--- |
| `num_init_children` | Sum(WAS maxPoolSize) + 여유량 (표준 120) | PgPool 공식(max_pool×num_init_children ≤ max_conn−superuser_reserved=97) 초과 위험 수용. 쿼리 취소 시 ×2 연결 소모 주의. 피크 SHOW POOL_PROCESSES 모니터링 필수 (2026-07-02 TA 확정) |
| `max_pool` | 단일 DB/계정: 1 / 복수 DB/계정: 조합 수 | 불필요한 상향 시 백엔드 연결 기하급수적 증가 |
| `child_life_time` | 1,680 (28min) | DB idle_session_timeout(30min)보다 짧게 |
| `connection_life_time` | 1,680 (28min) | PgPool -> DB 연결 수명 |
| `client_idle_limit` | 600 (10min) | 클라이언트 유휴 타임아웃 |
| `reserved_connections` | 1 | PgPool 관리용 예약 슬롯 |

### 3.2 로드 밸런싱

```conf
load_balance_mode = on
backend_clustering_mode = 'streaming_replication'
backend_weight0 = 1  # Primary (쓰기 전담 + 최소 읽기)
backend_weight1 = 3  # Replica (Primary 1 : Replica 3 = 25%:75%, 읽기 부하 Replica 집중)
```

> 비율 근거: PgPool 스트리밍 복제 표준 패턴(Primary 쓰기 전담, Replica 읽기 집중). 정산/결제는 애플리케이션에서 readPreference=primary 고정하므로 weight와 무관. (2026-07-02 TA 확정, reports/final과 정합)

### 3.3 페일오버 및 고가용성

- `use_watchdog = on` -- SPOF 방지 (VIP 기반)
- `failover_command` 지정 -- Primary 다운 시 Replica 승격 스크립트
- `delegate_ip` 설정 -- Watchdog이 관리할 VIP (v4.2+ 파라미터명, 기존 `wd_vip` 폐지)
- `trusted_servers` 2개 이상 -- upstream(WAS) 방향 네트워크 생존 확인, Split-Brain 방지. K8s 환경: 워커 노드 물리 IP 사용 (Service ClusterIP는 ping 미응답). PostgreSQL 서버 IP 지정 금지 (공식 문서 경고). 상세 가이드: `reports/final/pgpool-ii.md` §2.3

> **이중화 의무 (표준 2대)**: PgPool-II 단일 구성은 SPOF. 표준은 Active+Standby 2대(Watchdog VIP 이중화). **1대 운영 중인 곳은 2대로 증설 필수**. 서비스 규모가 작아 오버엔지니어링 우려 시 **IT기획실 문의** (예외 승인 후 단일 유지 가능). 상세 절차는 `reports/final/pgpool-ii.md` §6.7.

### 3.4 4GB RAM 독립 서버 제약

PgPool-II 전용 서버가 4GB RAM인 경우:
- `num_init_children = 120` 구동 시 프로세스 메모리 약 1GB (안정 범위)
- OS 커널 세마포어 상한선 필수: `kernel.sem = 250 32000 250 128`

---

## 4. 타임아웃 캐스케이드 (PostgreSQL 계층)

엄격한 부등호(`<`)로 계층 간 타임아웃을 격리. 등호(`<=`) 사용 금지.

### 4.1 WAS -> PgPool -> PostgreSQL 캐스케이드

```
WAS HikariCP maxLifetime (27min)
    |  maxLifetime < child_life_time < idle_session_timeout
    v
PgPool-II child_life_time (28min)
    |  child_life_time < idle_session_timeout
    v
PostgreSQL idle_session_timeout (30min)
```

### 4.2 PostgreSQL 내부 타임아웃 계층

```
statement_timeout (30s)
    |
    +-- idle_in_transaction_session_timeout (60s) -- 트랜잭션 유휴 강제 종료
    +-- lock_timeout (10s) -- Lock 대기 자동 취소
```

### 4.3 파라미터별 표준값

| 파라미터 | 표준값 | 비고 |
| :--- | :--- | :--- |
| `idle_session_timeout` | 1,800,000ms (30min) | 캐스케이드 최하위 |
| `idle_in_transaction_session_timeout` | 60,000ms (60s) | 교착 방지 |
| `statement_timeout` | 30,000ms (30s) | 장기 실행 쿼리 방지 |
| `lock_timeout` | 10,000ms (10s) | Lock 대기 제한 |

---

## 5. 70% Ceiling Rule (공유 DB 커넥션 관리)

```
DB max_connections = X
    |
    +-- 30% 예약 (관리자, 모니터링, 긴급 접속)
    +-- 70% 가용 (애플리케이션 커넥션 풀 전체 합산 상한)
```

**절대 제약**: `Sum(모든 WAS 인스턴스 maxPoolSize) <= DB max_connections * 0.7`

### 팀별 maxPoolSize 확정값

| 팀 | maxPoolSize/인스턴스 | 비고 |
| :--- | :---: | :--- |
| 플랫폼개발 (Nice M) | 20 | PostgreSQL + MongoDB 둘 다 사용 |
| 플랫폼개발 (Nice Charger) | 20 | 웹 100 -> 20 축소 |
| CL플랫폼 | 20 | 현금정보계와 동일 서버 |
| 주차서비스 | 20 | 과대 설정 축소 |
| 현금정보계 | 20 | 7 컨테이너 x 20 = 140 |

### PgPool 산출 예시 (플랫폼개발팀)

| 항목 | 수치 |
| :--- | :--- |
| WAS 인스턴스 | 4개 |
| 풀 합산 | 80~100 |
| num_init_children | **120** (안전 마진) |
| max_pool | **1** (단일 DB/계정) |
| PG max_connections 권장 | **200** (PgPool 백엔드 120 x 1.5 이상) |

---

## 6. 모니터링 최소 체계 (PostgreSQL)

| 우선순위 | 항목 | 조회 방법 | 임계치 | 조치 |
| :---: | :--- | :--- | :--- | :--- |
| 높음 | Active Sessions | `pg_stat_activity` | max_conn 70% 경고 / 85% 위험 | 커넥션 풀 재검토 |
| 높음 | Slow Query | `pg_stat_statements` | >= 1s | 인덱스 / 쿼리 튜닝 |
| 높음 | COLLSCAN | `seq_scan / idx_scan` 비율 | 발생 시 즉시 | 인덱스 설계 |
| 중간 | Lock Wait | `pg_locks` | > 1s | 트랜잭션 분석 |
| 높음 | Replication Lag | `pg_stat_replication` | > 5s 경고 / > 30s 위험 | 네트워크/부하 점검 |
| 중간 | Dead Tuples | `pg_stat_user_tables` | 테이블 크기 10% 초과 | autovacuum 강제 |
| 중간 | Cache Hit Ratio | `pg_stat_database` | < 95% | shared_buffers 증설 검토 |

---

## 7. 검증 체크리스트

에이전트는 PostgreSQL 설정 변경 후 반드시 아래 항목을 검증.

| # | 검증 항목 | 충족 조건 |
| :---: | :--- | :--- |
| 1 | `shared_buffers` <= RAM * 0.25 | PostgreSQL 공식 권장 |
| 2 | `max_connections` >= Sum(WAS maxPoolSize) * 1.5 | 70% Ceiling Rule 역산 |
| 3 | WAS maxLifetime < PgPool child_life_time < PG idle_session_timeout | 엄격 부등호 |
| 4 | `autovacuum` = on | 비활성화 금지 |
| 5 | `autovacuum_vacuum_cost_limit` >= 1,000 | 기본값(200) 상향 |
| 6 | `idle_in_transaction_session_timeout` 설정 | 교착 방지 |
| 7 | `random_page_cost` = 1.1 (SSD) | SSD 환경 필수 |
| 8 | PgPool `reserved_connections` >= 1 | 장애 시 DBA 접속 보장 |
| 9 | PgPool `max_pool` = 1 (단일 DB/계정) | 불필요한 커넥션 폭증 방지 |
| 10 | Sum(maxPoolSize) <= DB max_conn * 0.7 | 70% Ceiling |
| 11 | postmaster context 파라미터(shared_buffers, max_connections, wal_buffers, wal_level, max_wal_senders, archive_mode 등) 변경 시 PgPool detach/attach 롤링 절차 준수 | 무중단 restart (상세: `reports/final/postgresql.md` §6) |
| 12 | Primary 노드 restart 전 `pcp_promote_node -s` switchover 또는 `backend_flag{N} = DISALLOW_TO_FAILOVER` 설정 | 의도치 않은 자동 failover 방지 (미준수 시: Replica 승격, 복제 토폴로지 역전) |
| 13 | reload 가능 파라미터(sighup/user context) 변경 시 `systemctl reload postgresql` 우선, `pending_restart` 조회로 restart 필요성 사전 확인 | 불필요한 서비스 중단 회피 |
| 14 | PgPool-II 2대(Active+Standby) 이중화 구성 (단일 구성 시 2대 증설 또는 IT기획실 예외 승인) | SPOF 제거, 무중단 유지보수 가능 (`reports/final/pgpool-ii.md` §6.7) |
| 15 | `delegate_ip` 설정 (빈 값 불가) | VIP 미설정 시 Watchdog 페일오버 무효 (`reports/final/pgpool-ii.md` §2.3) |
| 16 | `trusted_servers` 2개 이상 지정 (PostgreSQL IP 제외) | Split-Brain 방지. upstream 네트워크 단절 감지 (`reports/final/pgpool-ii.md` §2.3) |

---

## 8. 에이전트 작업 규칙

1. **DB 전용 서버 기준**: 모든 산출 공식은 DB 전용 서버(또는 VM) 기준. 공유 환경은 별도 명시
2. **스트리밍 복제 필수**: HA 아키텍처는 내장 스트리밍 복제 + PgPool-II 연계를 표준으로 함
3. **버저닝**: 가이드 갱신 시 `reports/db-standard-guide-v{N}.md` 생성 후 AGENTS.md 링크 갱신
4. **Cross-domain 참조**: WAS Connection Pool 변경이 필요한 작업은 WAS harness 규칙을 추가 로드
5. **방화벽 제약**: 사내망 TCP Established Timeout = 30분(1,800초). 모든 타임아웃 산정의 최상위 기준
6. **PgPool 단일 DB 기준**: `max_pool = 1`을 기본으로 하되, 복수 DB/계정 환경은 조합 수만큼 상향 명시
