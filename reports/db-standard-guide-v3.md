# DB 서버 표준 설정 가이드라인 V3

> **버전**: V3 -- MongoDB 8.0 LTS / 8.3 기준 전면 갱신 (PostgreSQL V2 내용 유지)
> **V2 대비 주요 변경**: MongoDB 섹션 MongoDB 8.3(2026-06-11 기준 최신) 기준 전면 업데이트. 8.0 신규 파라미터(`defaultMaxTimeMS`, `cacheSizePct`) 추가. Write Concern 동작 변경 반영. Shard Key resharding 개선사항 반영. TCMalloc per-CPU 캐시 변화 반영. Deprecated 기능(Index Filters, Hedged Reads, LDAP) 안내 추가.
> **V2 원본**: `reports/db-standard-guide-v2.md` 보존

---

## Quick Cheatsheet

> **[참고사항]** 아래 표는 DB 전용 서버 기준의 핵심 값만 추출한 요약입니다. 상세 산정 근거는 각 절을 참조하세요.

### RAM별 핵심값 요약

| DB 서버 RAM | PostgreSQL `shared_buffers` | PostgreSQL `max_wal_size` | MongoDB `cacheSizeGB` | 비고 |
| :---: | :---: | :---: | :---: | :--- |
| **8 GB** | 2 GB | 2 GB | 3.5 GB | 소규모 |
| **16 GB** | 4 GB | 4 GB | 7.5 GB | 표준 |
| **32 GB** | 8 GB | 16 GB | 15.5 GB | 고성능 |
| **64 GB** | 16 GB | 32 GB | 31.5 GB | 대규모 |

### 아키텍처별 핵심 차이점 요약

| 구분 | PostgreSQL Standalone | PostgreSQL SR Only | PostgreSQL PgPool+SR | PostgreSQL Patroni+etcd | MongoDB Standalone | MongoDB Replica Set | MongoDB Sharded |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **페일오버** | 수동 | 수동/스크립트 | 자동(VIP) | 완전 자동(DCS) | 없음 | 자동(5~15s) | 샤드 단위 자동 |
| **RTO** | 수 시간 | 수 분~30s | 10~30s | 10~30s | 수 시간 | 5~15s | 샤드 장애 시 해당 샤드 |
| **RPO** | 백업 시점 | 수 초~0 | 수 초~0 | 0(동기 시) | 백업 시점 | 수 초~0 | 샤드별 수 초~0 |
| **최소 노드** | 1 | 2 | 4+ | 5+ | 1 | 3 | 9+ |
| **커넥션 풀** | HikariCP | HikariCP | PgPool-II | PgBouncer | 앱 풀 | 앱 풀 | mongos |
| **운영 복잡도** | 최저 | 낮음 | 중간 | 높음 | 최저 | 중간 | 높음 |

---

## 1. 아키텍처 선택 의사결정 매트릭스

### 1.1 의사결정 흐름도

#### PostgreSQL 아키텍처 의사결정

```mermaid
flowchart TD
    START(["PostgreSQL 아키텍처 선택"]) --> Q1{"트랜잭션/HA 필요?"}
    Q1 -->|"아니오"| PG_SA[("Standalone")]
    Q1 -->|"예"| Q2{"RTO 요구사항?"}
    Q2 -->|"수 분 단위 허용"| Q3{"DBA 수동 페일오버<br/>가능?"}
    Q2 -->|"30초 이내"| Q4{"다수 WAS 공유 또는<br/>읽기 분산 필요?"}
    Q3 -->|"예"| PG_SR[("SR Only")]
    Q3 -->|"아니오"| Q4
    Q4 -->|"예"| PG_PGPOOL[("PgPool+SR")]
    Q4 -->|"아니오"| Q5{"etcd 운영<br/>수용 가능?"}
    Q5 -->|"예"| PG_PATRONI[("Patroni+etcd")]
    Q5 -->|"아니오"| PG_PGPOOL

    style PG_SA fill:#e8e8e8,stroke:#333
    style PG_SR fill:#fff3cd,stroke:#856404
    style PG_PGPOOL fill:#d4edda,stroke:#28a745
    style PG_PATRONI fill:#cce5ff,stroke:#004085
```

#### MongoDB 아키텍처 의사결정

```mermaid
flowchart TD
    START(["MongoDB 아키텍처 선택"]) --> Q1{"환경?"}
    Q1 -->|"개발/테스트"| MG_SA[("Standalone")]
    Q1 -->|"프로덕션"| Q2{"데이터 > 1TB 또는<br/>쓰기 TPS > 20k?"}
    Q2 -->|"아니오"| MG_RS[("Replica Set<br/>PSS")]
    Q2 -->|"예"| MG_SHARD[("Sharded<br/>Cluster")]

    style MG_SA fill:#e8e8e8,stroke:#333
    style MG_RS fill:#d4edda,stroke:#28a745
    style MG_SHARD fill:#f8d7da,stroke:#721c24
```

### 1.2 의사결정 기준표

| 기준 | Standalone | SR / Replica Set | PgPool+SR | Patroni+etcd | Sharded |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **RTO** | 수 시간 | 수 분~30s | 10~30s | <30s | 샤드 단위 5~15s |
| **RPO** | 백업 시점 | 수 초~0 | 수 초~0 | 0 | 샤드별 수 초~0 |
| **트래픽** | 극소~소 | 소~중 | 중~대 | 대 | 대~초대 |
| **데이터 규모** | 소 | 중 | 중~대 | 대 | >1TB |
| **운영팀 규모** | 1인 | 1~2인 | 2~3인 | 3인+ | 3인+ |
| **최소 서버** | 1 | 2~3 | 4+ | 5+ | 9+ |

### 1.3 서비스 유형별 아키텍처 매핑

| 서비스 유형 | 트래픽 | RTO | RPO | PostgreSQL | MongoDB | 사유 |
| :--- | :---: | :---: | :---: | :--- | :--- | :--- |
| 개발/테스트 | 극소 | 무관 | 무관 | **Standalone** | **Standalone** | 단순성 최우선 |
| 소규모 내부 도구 | 소 | 수 시간 | 수 시간 | **Standalone** | **Standalone** | 장애 시 수동 복구 가능 |
| 중간 규모 상용 (수동 페일오버) | 중 | 수 분 | 수 초 | **SR Only** | **Replica Set (PSS)** | 외부 도구 최소 |
| 중간 규모 상용 (자동 페일오버) | 중 | 30s | 수 초 | **PgPool+SR** | **Replica Set (PSS)** | 읽기 분산 + 커넥션 풀 통합 |
| 대규모 상용 (다수 서비스 공유 DB) | 대 | 30s | 수 초 | **PgPool+SR** | **Replica Set (PSS)** | 커넥션 풀링 필수 |
| 미션 크리티컬 (RPO=0) | 대 | <30s | 0 | **Patroni+etcd** | **Replica Set (PSS)** | 동기 복제 + 완전 자동 페일오버 |
| 대규모 (>1TB, 쓰기 >20k) | 초대 | 15s | 수 초 | **Patroni+etcd (Citus 등 Sharding 확장 검토 필요)** | **Sharded Cluster** | 쓰기 부하 극복을 위한 애플리케이션 샤딩 또는 미션 크리티컬 자동 페일오버 제어 필수 |
| 분석/리포팅 전용 | 낮음 | 무관 | 수 초 | **SR Only (Replica 읽기)** | **Replica Set + Hidden** | Primary 부하 영향 없음 |

---

## 2. PostgreSQL 구성별 상세 가이드

### 2.1 Standalone

#### 아키텍처

```mermaid
graph LR
    APP[WAS<br/>HikariCP] --> PG[(PostgreSQL<br/>Standalone)]
    style PG fill:#e8e8e8,stroke:#333
```

#### 적용 서비스

- 개발/테스트 환경
- 소규모 내부 도구 (RTO/RPO 무관)
- 로깅/통계 서비스 (수동 복구 허용)

#### 핵심 파라미터 차이

| 파라미터 | Standalone 값 | 복제 구성 대비 차이 | 비고 |
| :--- | :---: | :--- | :--- |
| **wal_level** | `replica` | 복제 구성: `replica` | 기본 표준은 `replica` (PITR 및 아카이브 백업 허용). `minimal`은 백업이 전혀 필요 없는 순수 개발계 및 휘발성 임시/로그 데이터 장비에 한해서만 허용 |
| **max_wal_senders** | `0` | 복제 구성: `3~5` | 복제 불필요 |
| **hot_standby** | `off` | 복제 구성: `on` (Replica) | Standby 없음 |
| **archive_mode** | `off` | 복제 구성: `on` (선택) | 아카이빙 불필요 |

#### RAM별 파라미터 매트릭스

| DB 서버 RAM | shared_buffers | effective_cache_size | work_mem | maintenance_work_mem | wal_buffers | max_connections |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **8 GB** | 2 GB | 6 GB | 10 MB | 384 MB | 16 MB | 100 |
| **16 GB** | 4 GB | 12 GB | 32 MB | 768 MB | 16 MB | 100 |
| **32 GB** | 8 GB | 24 GB | 64 MB | 1.5 GB | 16 MB | 100 |
| **64 GB** | 16 GB | 48 GB | 128 MB | 3 GB | 16 MB | 100 |

> **[참고사항]** Standalone은 단일 WAS 또는 소수 WAS만 연결되므로 `max_connections = 100`으로 충분. `wal_level = replica`를 기본 표준으로 설정하여 PITR 및 아카이브 백업을 항상 허용함. 단, 백업이 전혀 필요 없는 순수 개발계 및 휘발성 임시/로그 데이터 장비에 한해서만 `minimal`로 변경을 허용함. `max_wal_size`는 기본값(1GB) 사용 가능하나, 쓰기 빈도에 따라 2GB까지 상향 허용.

---

### 2.2 Streaming Replication (Primary-Replica, 수동 페일오버)

#### 아키텍처

```mermaid
graph LR
    APP[WAS<br/>HikariCP] --> PG_M[(PostgreSQL<br/>Primary)]
    PG_M -->|WAL Streaming| PG_R[(PostgreSQL<br/>Replica)]
    PG_R -.->|Read Only<br/>App 레벨 분산 시| APP

    style PG_M fill:#d4edda,stroke:#28a745
    style PG_R fill:#fff3cd,stroke:#856404
```

#### 적용 서비스

- 중간 규모 상용 서비스
- 읽기 부하 분산 필요 (애플리케이션 레벨 분산)
- RTO 수 분 허용, DBA 수동 페일오버 가능한 환경

#### 핵심 파라미터 차이

| 파라미터 | Primary | Replica | 비고 |
| :--- | :---: | :---: | :--- |
| **wal_level** | `replica` | -- | 스트리밍 복제 필수 |
| **max_wal_senders** | `3~5` | `3~5` | 장애 시 Replica가 Primary로 승격(Promote) 후 다운스트림 복제본 수용 및 백업 연결을 즉시 허용해야 하므로 Primary와 동일하게 설정 |
| **hot_standby** | -- | `on` | Replica에서 읽기 쿼리 수용 |
| **listen_addresses** | `'*'` | `'*'` | 원격 접속 허용 |

#### 페일오버 절차 (수동)

```bash
# Step 1: Primary 장애 확인
pg_isready -h primary-host -p 5432

# Step 2: Replica를 새 Primary로 승격
pg_ctl promote -D /var/lib/postgresql/data

# Step 3: WAS 연결 문자열을 새 Primary로 수동 변경
# Step 4: (선택) Keepalived VIP 전환 스크립트 실행
```

> **[참고사항]** Keepalived + 커스텀 스크립트 조합 시 VIP 기반 자동 전환(30초 이내) 가능. 단, 스플릿 브레인 방어는 스크립트 품질에 의존하므로 PgPool 또는 Patroni 대비 신뢰도 낮음.

#### RAM별 파라미터 매트릭스

| DB 서버 RAM | shared_buffers | effective_cache_size | work_mem | maintenance_work_mem | wal_buffers | max_wal_size | max_wal_senders |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **8 GB** | 2 GB | 6 GB | 10 MB | 384 MB | 16 MB | 2 GB | 3 |
| **16 GB** | 4 GB | 12 GB | 32 MB | 1 GB | 16 MB | 4 GB | 3 |
| **32 GB** | 8 GB | 24 GB | 64 MB | 2 GB | 16 MB | 16 GB | 3 |
| **64 GB** | 16 GB | 48 GB | 128 MB | 4 GB | 16 MB | 32 GB | 5 |

---

### 2.3 PgPool-II + Streaming Replication

#### 아키텍처

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

    style PP1 fill:#d4edda,stroke:#28a745
    style PP2 fill:#fff3cd,stroke:#856404
    style PG_M fill:#cce5ff,stroke:#004085
    style PG_R fill:#cce5ff,stroke:#004085
    style VIP fill:#f8d7da,stroke:#721c24
```

#### 적용 서비스

- 중~대규모 상용 서비스
- 다수 WAS 인스턴스가 DB 공유
- 읽기/쓰기 분산 필요
- RTO 30초 이내

#### PostgreSQL 핵심 파라미터

| 파라미터 | Primary | Replica | 비고 |
| :--- | :---: | :---: | :--- |
| **wal_level** | `replica` | -- | 스트리밍 복제 필수 |
| **max_wal_senders** | `3~5` | `3~5` | 장애 시 승격(Promote) 대비 Primary/Replica 통일 |
| **hot_standby** | -- | `on` | PgPool 읽기 분산 필수 |
| **max_connections** | `200~500` | `200~500` | PgPool 백엔드 수용량 * 1.5 이상 |

#### PgPool-II 전용 파라미터

| 우선순위 | 파라미터 | 산정 기준 | 비고 |
| :---: | :--- | :--- | :--- |
| [높음] | **num_init_children** | `Sum(WAS maxPoolSize) + 여유 (최소 120)` | PgPool이 수용할 동시 클라이언트 연결 수 |
| [높음] | **max_pool** | 단일 DB/계정: `1`, 복수 DB/계정: `조합 수` | 불필요한 상향 시 백엔드 연결 기하급수적 증가 |
| [중간] | **child_life_time** | `1,680` (28min) | DB idle_session_timeout(30min)보다 짧게 |
| [중간] | **connection_life_time** | `1,680` (28min) | PgPool -> DB 연결 수명 |
| [중간] | **client_idle_limit** | `600` (10min) | 클라이언트 유휴 타임아웃 |
| [높음] | **reserved_connections** | `1~2` | PgPool 관리용 예약 슬롯 |
| [높음] | **load_balance_mode** | `on` | 읽기 분산 활성화 |
| [높음] | **backend_clustering_mode** | `'streaming_replication'` | PgPool-II v4.x+ 스트리밍 복제 모드 (기존 master_slave_mode 폐지) |

> **[참고사항]** **[4GB RAM 독립 서버 가이드]** PgPool-II 전용 서버가 4GB RAM인 경우, `num_init_children = 120` 구동 시 프로세스 메모리 점유율(약 1GB 내외)은 안정 범위이나, OS 커널 세마포어 상한선 설정 필수: `kernel.sem = 250 32000 250 128`
>
> **※ 주의**: 향후 복수 DB/계정 매핑으로 인해 `max_pool`이 2 이상으로 증가할 경우, 백엔드 이론상 최대 연결 수(`num_init_children * max_pool`)가 기하급수적으로 늘어나 PgPool 서버의 메모리(RAM) 고갈을 초래할 수 있음. 따라서 멀티 DB 환경 확장 시에는 `num_init_children` 값을 하향 조정하거나 PgPool 서버의 RAM 증설이 선행되어야 함.

#### RAM별 파라미터 매트릭스 (PostgreSQL)

| DB 서버 RAM | shared_buffers | effective_cache_size | work_mem | maintenance_work_mem | wal_buffers | max_wal_size | max_wal_senders |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **8 GB** | 2 GB | 6 GB | 10 MB | 384 MB | 16 MB | 2 GB | 3~5 |
| **16 GB** | 4 GB | 12 GB | 32 MB | 1 GB | 16 MB | 4 GB | 3~5 |
| **32 GB** | 8 GB | 24 GB | 64 MB | 2 GB | 16 MB | 16 GB | 3~5 |
| **64 GB** | 16 GB | 48 GB | 128 MB | 4 GB | 16 MB | 32 GB | 5 |

---

### 2.4 Patroni + etcd + HAProxy

#### 아키텍처

```mermaid
graph LR
    subgraph WAS Layer
        W1[WAS-1<br/>HikariCP]
        W2[WAS-2<br/>HikariCP]
    end

    subgraph HAProxy Layer
        HA1[HAProxy-1]
        HA2[HAProxy-2]
    end

    subgraph Patroni Cluster
        PAT1[Patroni<br/>+ PostgreSQL<br/>Node 1]
        PAT2[Patroni<br/>+ PostgreSQL<br/>Node 2]
        PAT3[Patroni<br/>+ PostgreSQL<br/>Node 3]
    end

    subgraph DCS
        ETCD1[etcd-1]
        ETCD2[etcd-2]
        ETCD3[etcd-3]
    end

    W1 --> HA1
    W2 --> HA2
    HA1 --> PAT1
    HA1 --> PAT2
    HA1 --> PAT3
    PAT1 <-->|Distributed<br/>Consensus| ETCD1
    PAT2 <-->|Distributed<br/>Consensus| ETCD2
    PAT3 <-->|Distributed<br/>Consensus| ETCD3

    style PAT1 fill:#d4edda,stroke:#28a745
    style PAT2 fill:#fff3cd,stroke:#856404
    style PAT3 fill:#fff3cd,stroke:#856404
    style ETCD1 fill:#cce5ff,stroke:#004085
    style ETCD2 fill:#cce5ff,stroke:#004085
    style ETCD3 fill:#cce5ff,stroke:#004085
```

#### 적용 서비스

- 미션 크리티컬 상용 서비스
- RTO < 30초, RPO = 0 (동기 복제)
- 쿠버네티스/클라우드 네이티브 환경
- 스플릿 브레인 원천 차단 필요

#### 핵심 파라미터 (Patroni YAML)

| 우선순위 | 파라미터 | 설정값 | 비고 |
| :---: | :--- | :--- | :--- |
| [높음] | **synchronous_mode** | `true` | 동기 복제 활성화 (RPO = 0) |
| [높음] | **maximum_lag_on_failover** | `1048576` (1MB) | 페일오버 시 최대 허용 Lag |
| [중간] | **failover.mode** | `automatic` | 자동 페일오버 |
| [높음] | **etcd.hosts** | etcd 클러스터 주소 목록 | DCS 연결 |
| [중간] | **restapi.listen** | `0.0.0.0:8008` | Patroni REST API |

#### Patroni vs PgPool 비교

| 기준 | PgPool+SR | Patroni+etcd |
| :--- | :--- | :--- |
| **페일오버** | Watchdog VIP 기반 | DCS 분산 합의 기반 |
| **스플릿 브레인** | Watchdog로 방어 | etcd 분산 락으로 원천 차단 |
| **커넥션 풀** | PgPool 내장 | PgBouncer 별도 구성 |
| **읽기 분산** | PgPool SQL 레벨 자동 | HAProxy 라우팅 + 라벨 |
| **최소 서버** | 4대 (PG 2 + PgPool 2) | 5대 (PG 3 + etcd 3, PG 노드 병설 가능) + HAProxy 1~2 |
| **K8s 호환** | 제한적 | 완전 (Zalando Operator) |

#### RAM별 파라미터 매트릭스 (PostgreSQL 노드)

| DB 서버 RAM | shared_buffers | effective_cache_size | work_mem | maintenance_work_mem | wal_buffers | max_wal_size | max_wal_senders |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **8 GB** | 2 GB | 6 GB | 10 MB | 384 MB | 16 MB | 2 GB | 3~5 |
| **16 GB** | 4 GB | 12 GB | 32 MB | 1 GB | 16 MB | 4 GB | 3~5 |
| **32 GB** | 8 GB | 24 GB | 64 MB | 2 GB | 16 MB | 16 GB | 5 |
| **64 GB** | 16 GB | 48 GB | 128 MB | 4 GB | 16 MB | 32 GB | 5 |

> **[참고사항]** Patroni 환경에서 PostgreSQL 파라미터는 `patroni.yml`의 `postgresql.parameters` 섹션에서 관리. Patroni가 자동으로 Primary/Replica 역할 전환 시 파라미터를 적용하므로, `postgresql.conf` 직접 편집은 지양.

---

## 3. MongoDB 구성별 상세 가이드

### 3.1 Standalone

#### 아키텍처

```mermaid
graph LR
    APP[WAS<br/>MongoDB Driver] --> MG[(mongod<br/>Standalone)]
    style MG fill:#e8e8e8,stroke:#333
```

#### 적용 서비스

- 개발/테스트 환경
- 프로토타입/MVP (사용자 증가 시 Replica Set 전환 계획 필수)
- 로컬 캐시, RTO/RPO 무관한 임시 데이터

> **[주의]** MongoDB Standalone은 **멀티 도큐먼트 트랜잭션 미지원**. 프로덕션 환경에서는 MongoDB 공식 문서에서도 비권장 ("Replica sets are the basis for all production deployments").

#### 핵심 파라미터

| 우선순위 | 파라미터 | 설정값 | 비고 |
| :---: | :--- | :--- | :--- |
| [높음] | **cacheSizeGB** | `0.5 * (RAM - 1)` | DB 전용 서버 기준 |
| [중간] | **maxIncomingConnections** | `65536` | 기본값 사용 |
| [낮음] | **replSetName** | 미설정 | 복제 불필요 |

#### RAM별 파라미터 매트릭스

| DB 서버 RAM | cacheSizeGB | maxIncomingConnections | 비고 |
| :---: | :---: | :---: | :--- |
| **8 GB** | 3.5 GB | 65536 | 개발/테스트 |
| **16 GB** | 7.5 GB | 65536 | 개발/테스트 |
| **32 GB** | 15.5 GB | 65536 | 프로토타입 (RS 전환 계획 필수) |
| **64 GB** | 31.5 GB | 65536 | 프로토타입 (RS 전환 계획 필수) |

---

### 3.2 Replica Set (PSS / PSA)

#### 아키텍처 (PSS 표준)

```mermaid
graph LR
    APP[WAS<br/>MongoDB Driver] --> MG_P[(mongod<br/>Primary)]
    MG_P -->|Oplog<br/>Replication| MG_S1[(mongod<br/>Secondary 1)]
    MG_P -->|Oplog<br/>Replication| MG_S2[(mongod<br/>Secondary 2)]
    MG_S1 -.->|Read<br/>Optional| APP
    MG_S2 -.->|Read<br/>Optional| APP

    style MG_P fill:#d4edda,stroke:#28a745
    style MG_S1 fill:#fff3cd,stroke:#856404
    style MG_S2 fill:#fff3cd,stroke:#856404
```

#### PSS vs PSA 비교

| 기준 | PSS (표준) | PSA (예외) |
| :--- | :--- | :--- |
| **구성** | Primary 1 + Secondary 2 | Primary 1 + Secondary 1 + Arbiter 1 |
| **데이터 복제** | 3중 복제 | 2중 복제 (Arbiter는 데이터 미보관) |
| **읽기 분산** | Secondary 2노드 활용 | Secondary 1노드만 활용 |
| **서버 자원** | 3대 필요 (전부 데이터 보관) | 3대 필요 (Arbiter는 경량) |
| **허용 조건** | 프로덕션 표준 | 하드웨어 극도 제약 시만 |
| **데이터 안전성** | 2노드 장애까지 복구 가능 | 1노드 장애까지 복구 가능 |

> **[PSA 구조 치명적 제약 경고]** PSA 구조에서 단 1대의 Secondary 노드라도 다운될 경우, 남은 데이터 노드가 Primary 1대뿐이므로 과반수 합의(Majority Consensus)가 불가능해짐. 이 경우 `w:majority` 설정이 적용된 모든 쓰기 트랜잭션이 영구 정지(Stall)되는 치명적인 장애가 발생함. 따라서 정산, 결제 등 트랜잭션 정합성이 필수적인 도메인에는 **PSA 구성을 절대 금지**하며, 반드시 PSS 구성을 준수해야 함.

#### 적용 서비스

- 대부분의 상용 서비스 (프로덕션 기본 요구사항)
- HA 필수, 트랜잭션 필요 (4.0+)
- 데이터 규모 < 1TB, 쓰기 TPS < 20,000

#### 핵심 파라미터

| 우선순위 | 파라미터 | 설정값 | 비고 |
| :---: | :--- | :--- | :--- |
| [높음] | **cacheSizeGB** | `0.5 * (RAM - 1)` | DB 전용 서버 기준 |
| [높음] | **replSetName** | `rs0` (환경에 맞게 명명) | Replica Set 식별자 |
| [중간] | **writeConcern** | 서비스 특성에 따라 (아래 표 참조) | 정합성 vs 성능 트레이드오프 |
| [중간] | **readPreference** | 서비스 특성에 따라 (아래 표 참조) | Primary vs Secondary 읽기 |
| [높음] | **Profiling Level** | `1 (slowms: 100)` | COLLSCAN 감지 필수 |
| [높음] | **electionTimeoutMillis** | `10000` (10s, 기본값) | Primary 장애 감지 타임아웃 |
| [중간] | **defaultMaxTimeMS** | 서비스 특성에 따라 (권장: 30000~60000) | **8.0 신규**. 개별 읽기 연산의 기본 시간 제한(ms). 장기 실행 쿼리 방어 |

#### Write Concern / Read Preference 의사결정표

| 서비스 유형 | writeConcern | readPreference | 사유 |
| :--- | :--- | :--- | :--- |
| 정산/결제 (정합성 필수) | `w: majority` | `primary` | 데이터 유실 허용 불가, 항상 최신 데이터 |
| 일반 상용 (HA 필요) | `w: 1` (기본) | `primary` | 기본 안정성 |
| 조회성 (Replication Lag 허용) | `w: 1` | `secondaryPreferred` | 읽기 부하 분산 |
| 대시보드/통계 (실시간성 낮음) | `w: 1` | `secondary` | Primary 읽기 부하 제로화 |

> **[핵심 제약]** 정산/결제 서비스는 **반드시 `primary`** 유지. 과거 데이터 조회 허용 여부에 따라 readPreference 선택.

> **[MongoDB 8.0 Write Concern 동작 변경]** `w:majority` 설정 시, 8.0부터 majority 노드가 oplog 엔트리를 **write한 시점**에 acknowledgment를 반환 (기존: oplog **적용 완료** 후 ack). 이로 인해 `w:majority` 쓰기 **성능이 향상**됨. 단, ack 반환 시점과 실제 데이터 적용 시점 간에 미세한 갭이 존재할 수 있으나, 정합성 보장에는 영향 없음.

> **[LDAP 인증 폐지 안내 (MongoDB 8.0)]** LDAP(Lightweight Directory Access Protocol)는 조직의 중앙 집중식 사용자 인증/인가를 위한 표준 프로토콜로, Active Directory 등과 연동하여 DB 접근 제어에 활용됨. MongoDB 8.0부터 LDAP 인증/인가가 **deprecated** 처리됨. 현재 LDAP를 사용하지 않더라도, 향후 도입을 검토하는 경우 **OIDC(OpenID Connect)** 인증을 대안으로 고려해야 함. LDAP는 MongoDB 8 수명 주기 동안 계속 작동하나, 향후 메이저 버전에서 제거될 예정.

#### RAM별 파라미터 매트릭스 (노드당)

| DB 서버 RAM | cacheSizeGB | maxIncomingConnections | 비고 |
| :---: | :---: | :---: | :--- |
| **8 GB** | 3.5 GB | 65536 | PSS 3노드 각각 동일 적용 |
| **16 GB** | 7.5 GB | 65536 | 표준 프로덕션 |
| **32 GB** | 15.5 GB | 65536 | 고성능 |
| **64 GB** | 31.5 GB | 65536 | 대규모 |

---

### 3.3 Sharded Cluster

#### 아키텍처

```mermaid
graph LR
    subgraph WAS Layer
        APP1[WAS-1<br/>MongoDB Driver]
        APP2[WAS-2<br/>MongoDB Driver]
    end

    subgraph mongos Router
        MS1[mongos-1]
        MS2[mongos-2]
    end

    subgraph Config Server RS
        CFG1[Config<br/>Primary]
        CFG2[Config<br/>Secondary]
        CFG3[Config<br/>Secondary]
    end

    subgraph Shard 1 RS
        S1_P[Shard-1<br/>Primary]
        S1_S1[Shard-1<br/>Secondary]
        S1_S2[Shard-1<br/>Secondary]
    end

    subgraph Shard 2 RS
        S2_P[Shard-2<br/>Primary]
        S2_S1[Shard-2<br/>Secondary]
        S2_S2[Shard-2<br/>Secondary]
    end

    APP1 --> MS1
    APP2 --> MS2
    MS1 --> S1_P
    MS1 --> S2_P
    MS2 --> S1_P
    MS2 --> S2_P
    MS1 <--> CFG1
    MS2 <--> CFG1
    CFG1 --> CFG2
    CFG1 --> CFG3

    style MS1 fill:#d4edda,stroke:#28a745
    style MS2 fill:#d4edda,stroke:#28a745
    style CFG1 fill:#cce5ff,stroke:#004085
    style CFG2 fill:#cce5ff,stroke:#004085
    style CFG3 fill:#cce5ff,stroke:#004085
    style S1_P fill:#fff3cd,stroke:#856404
    style S2_P fill:#fff3cd,stroke:#856404
```

#### 적용 서비스

- 대규모 (데이터 > 1TB)
- 초고속 쓰기 (>20,000 wps)
- 수평 확장 필수, 단일 서버 리소스 한계 도달

> **[MongoDB 8.0 운영 변경]** 8.0부터 샤드 노드에 직접 연결하여 명령을 실행할 수 없음. 모든 운영 명령은 `mongos`를 경유해야 하며, 예외적으로 `directShardOperations` 유지보수 전용 역할로 직접 연결 가능. 단, 1-shard 클러스터(Replica Set에서 전환 직후)에서는 제한 완화.

#### 샤딩 도입 시점 판단 기준

| 기준 | 단일 RS로 충분 | 샤딩 검토 필요 |
| :--- | :--- | :--- |
| **데이터 크기** | < 1TB | > 1TB 또는 사용 가능한 최대 디스크 도달 예상 |
| **쓰기 TPS** | < 20,000 wps | > 20,000 wps |
| **Working Set** | RAM 내 수용 가능 | RAM 초과, 캐시 적중률 < 95% |
| **연결 수** | < 10,000 동시 | > 10,000 동시 |
| **인덱스 크기** | RAM 내 수용 | 단일 노드 RAM 초과 |

> **[Shard Key 주의]** Shard Key 선택은 신중해야 함. MongoDB 5.0부터 resharding 지원, **8.0부터 동일 Shard Key로 resharding 가능**(`forceRedistribution: true`) 및 **데이터 분산 최대 50배 개선**. 그러나 resharding은 여전히 리소스 집약적 연산이므로, 카디널리티, 분포 균등성, 쿼리 패턴을 종합 분석하여 초기 설계에서 최적 Key 선택이 중요. ESR 규칙(Equality -> Sort -> Range) 준수.

#### 핵심 파라미터

| 우선순위 | 파라미터 | 설정값 | 비고 |
| :---: | :--- | :--- | :--- |
| [높음] | **Shard Key** | 카디널리티 높고, 쓰기 분산 균등, 자주 쿼리되는 필드 | ESR 규칙 준수 |
| [중간] | **chunkSize** | `64` (기본, MB) / 대량 쓰기 시 `128` 검토 | 청크 마이그레이션 단위 |
| [중간] | **balancer** | 기본 활성. 피크 시간 외 스케줄링 권장 | 데이터 마이그레이션 중 성능 영향 가능 |
| [낮음] | **Hedged Reads** | 8.0부터 **deprecated** | `nearest` readPreference에서 더 이상 기본 사용 안 함. 명시적 지정 시 경고 로그 출력 |

#### 구성 요소 최소 사양

| 컴포넌트 | 역할 | 최소 인스턴스 | 최소 RAM | cacheSizeGB | 비고 |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **Shard (Replica Set)** | 데이터의 부분 집합 보관 | 2 Shard x 3노드 = 6대 | 16 GB | 7.5 GB | 노드당 독립 산정 |
| **Config Server (RS)** | 클러스터 메타데이터/설정 저장 | 3노드 RS | 8 GB | 3.5 GB | 일반적으로 소규모 |
| **mongos (Router)** | 클라이언트-클러스터 간 쿼리 라우터 | 2개 이상 (HA) | 4 GB | N/A | WAS 서버 병설 가능 |

#### RAM별 파라미터 매트릭스 (Shard 노드당)

| DB 서버 RAM | cacheSizeGB | maxIncomingConnections | 비고 |
| :---: | :---: | :---: | :--- |
| **8 GB** | 3.5 GB | 65536 | 소규모 Shard |
| **16 GB** | 7.5 GB | 65536 | 표준 Shard |
| **32 GB** | 15.5 GB | 65536 | 고성능 Shard |
| **64 GB** | 31.5 GB | 65536 | 대규모 Shard |

> **[참고사항]** Config Server는 메타데이터만 보관하므로 8GB RAM이면 충분. mongos는 라우팅만 수행하므로 4GB RAM으로 운영 가능하며, WAS 서버에 병설하는 것이 네트워크 홉을 줄이는 방안.

---

## 4. 인프라 스펙별 표준 설정값 매트릭스

### 4.1 PostgreSQL 메모리 설정 (DB 전용 서버 기준)

| DB 서버 RAM | shared_buffers | effective_cache_size | work_mem | maintenance_work_mem | wal_buffers | 비고 |
| :---: | :---: | :---: | :---: | :---: | :---: | :--- |
| **8 GB** | **2 GB** | 6 GB | 10 MB | 384 MB | 16 MB | 소규모 |
| **16 GB** | **4 GB** | 12 GB | 32 MB | 1 GB | 16 MB | 표준 |
| **32 GB** | **8 GB** | 24 GB | 64 MB | 2 GB | 16 MB | 고성능 |
| **64 GB** | **16 GB** | 48 GB | 128 MB | 4 GB | 16 MB | 대규모 |

> **아키텍처별 참고**: Standalone은 `work_mem` 상향 가능(max_connections 적으므로). SR/PgPool/Patroni는 위 표 기준값 적용. Patroni는 `postgresql.conf` 대신 `patroni.yml`에서 관리.

#### WAL/체크포인트 설정

| 우선순위 | 파라미터 | DB 서버 RAM | 표준값 | 비고 |
| :---: | :--- | :---: | :---: | :--- |
| [높음] | **max_wal_size** | ~16 GB | **2 GB** | 체크포인트 간 WAL 누적 상한 |
| [높음] | **max_wal_size** | 16 ~ 32 GB | **4 GB** | 16GB RAM 기준값 |
| [높음] | **max_wal_size** | 32 GB 이상 | **16 ~ 32 GB** | 과소 설정 시 잦은 강제 체크포인트 |
| [중간] | **min_wal_size** | 공통 | **1 GB** | WAL 최소 유지 크기 |
| [중간] | **checkpoint_completion_target** | 공통 | **0.9** | 체크포인트 분산 |
| [높음] | **max_wal_senders** | Standalone: `0` / SR~Patroni: `3~5` | 아키텍처별 상이 | Replica 수 + 여유 |

> **아키텍처별 참고**: Standalone의 `wal_level`은 기본 표준을 `replica`로 설정하여 PITR 및 아카이브 백업을 항상 허용함. 순수 개발계 및 휘발성 데이터 장비에 한해 `minimal` 허용. `max_wal_size`를 기본값(1GB)로 운영해도 무방하나 쓰기 빈도에 따라 2GB까지 상향 가능.
>
> **※ 주의**: 고성능/대규모 서버(32GB–64GB RAM)에서 `max_wal_size`를 16–32GB로 과감하게 상향하는 경우, 대량의 쓰기 배치(Bulk Write) 작업 시 `min_wal_size`와의 격차로 인해 디스크 쓰기 버스트(Spike)가 발생할 수 있음. 따라서 스토리지의 IOPS 제한 및 가상화 환경의 Write Throttling 발생 여부를 반드시 함께 모니터링해야 함.

#### autovacuum 설정

| 우선순위 | 파라미터 | 표준값 | 비고 |
| :---: | :--- | :--- | :--- |
| [높음] | **autovacuum** | on | 비활성화 금지 (모든 아키텍처 공통) |
| [중간] | **autovacuum_max_workers** | `3 ~ 5` | 5 이하 유지, 무작정 증설 금지 |
| [중간] | **autovacuum_naptime** | 1 min | 점검 주기 |
| [중간] | **autovacuum_vacuum_scale_factor** | 0.1 | Dead Tuple 10% 도달 시 VACUUM |
| [중간] | **autovacuum_vacuum_cost_limit** | `1000 ~ 2000` | 기본값(200) 대비 상향 |

#### 커넥션/타임아웃 설정

| 우선순위 | 파라미터 | 표준값 | 비고 |
| :---: | :--- | :--- | :--- |
| [높음] | **max_connections** | 200 ~ 500 | 아키텍처별 상이 (Standalone: 100) |
| [높음] | **superuser_reserved_connections** | 3 | 관리자 예약 |
| [중간] | **statement_timeout** | 30,000 ms (30s) | 장기 실행 쿼리 방지 |
| [높음] | **lock_timeout** | 10,000 ms (10s) | Lock 대기 시간 제한 |
| [높음] | **idle_in_transaction_session_timeout** | 60,000 ms (60s) | 트랜잭션 유휴 강제 종료 |

### 4.2 MongoDB 메모리 및 구성원 설정

#### WiredTiger 캐시

| DB 서버 RAM | cacheSizeGB | 계산 | 비고 |
| :---: | :---: | :--- | :--- |
| **8 GB** | 3.5 GB | `0.5 * (8 - 1)` | 소규모 |
| **16 GB** | 7.5 GB | `0.5 * (16 - 1)` | 표준 |
| **32 GB** | 15.5 GB | `0.5 * (32 - 1)` | 고성능 |
| **64 GB** | 31.5 GB | `0.5 * (64 - 1)` | 대규모 |

> **공식**: `cacheSizeGB = 0.5 * (RAM - 1GB)` -- DB 전용 서버 기준. 공유 환경(WAS/DB 혼합)에서는 `RAM * 0.25`로 명시적 제한.

> **[MongoDB 8.0 추가 옵션]** `cacheSizePct` 파라미터가 8.0에서 추가됨. `cacheSizeGB` 대신 **퍼센트 기반**으로 캐시 사이즈를 지정할 수 있음. 예: `cacheSizePct: 0.5` (RAM의 50%). `cacheSizeGB`와 `cacheSizePct` 중 하나만 설정 가능하며, 둘 다 설정 시 MongoDB가 기동하지 않음.

#### TCMalloc 메모리 관리 (MongoDB 8.0 변경사항)

| 항목 | 8.0 이전 | 8.0 이후 | 비고 |
| :--- | :--- | :--- | :--- |
| **TCMalloc 캐시** | per-Thread | **per-CPU** | 단편화 감소, 고부하 안정성 향상 |
| **tcmallocEnableBackgroundThread** | 비활성 | **기본 활성** | 주기적 OS 메모리 반환 |
| **tcmallocReleaseRate** | 기본값 1 | **기본값 0** (bytes/sec) | 0 = 자동 해제 비활성화. 필요시 명시적 설정 |

> **[참고사항]** TCMalloc per-CPU 캐시 도입으로 메모리 사용 패턴이 변경됨. 기존 대비 RSS(Resident Set Size)가 일시적으로 높게 보일 수 있으나, 실제 메모리 단편화는 감소함. `tcmallocReleaseRate`를 명시적으로 설정하여 OS 메모리 반환 속도 조절 가능.

> **[Linux Kernel 호환성]** MongoDB 8.0 초기 버전은 Linux Kernel 6.19와 TCMalloc 비호환 이슈가 있었으나, **8.0.4에서 패치 완료**. 현재 모든 8.0.4+ 버전에서 정상 동작함.

#### Query Settings (MongoDB 8.0 Index Filters 대체)

> **[MongoDB 8.0 변경]** Index Filters가 **deprecated** 처리됨. 대신 **Query Settings**(`setQuerySettings` 명령) 사용 권장. Query Settings는 Index Filters 대비 다음 장점이 있음:
> - 클러스터 전체 노드에 걸쳐 **영속적** 적용
> - `find`, `distinct`, `aggregate` 명령 모두 지원
> - 인덱스 힌트뿐만 아니라 다양한 쿼리 동작 제어 가능
>
> 기존 Index Filters 사용 환경에서 8.0+ 업그레이드 시 Query Settings로 전환 필요.

#### Replica Set 멤버 구성 표준

| 구성 | 노드 수 | 구성원 | 허용 조건 | 비고 |
| :--- | :---: | :--- | :--- | :--- |
| **PSS (표준)** | 3 | Primary 1 + Secondary 2 | 프로덕션 기본 표준 | 데이터 3중 복제 |
| **PSA (예외)** | 3 | Primary 1 + Secondary 1 + Arbiter 1 | 하드웨어 극도 제약 시만 | Arbiter는 투표만 수행 |

---

## 5. 타임아웃 캐스케이드 (Timeout Cascade)

> **[높음] 핵심 원칙**: 상위 계층이 하위 계층보다 먼저 연결을 끊도록 설정. 등호(`<=`)가 아닌 **엄격한 부등호(`<`)**로 계층 간 타임아웃 격리.
>
> **방화벽 제약**: 사내망 TCP Established Timeout = 30분(1,800초). 모든 타임아웃 산정의 최상위 기준.

### 5.1 WAS -> PgPool-II -> PostgreSQL

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

### 5.2 WAS -> PostgreSQL (직접 연결)

```
WAS HikariCP maxLifetime (1,620,000ms = 27min)
     |
     |  maxLifetime < idle_session_timeout < 방화벽 timeout
     v
PostgreSQL idle_session_timeout (1,800,000ms = 30min)
     |
     v
방화벽 TCP Established Timeout (30min / 1,800s)
```

> **[참고사항]** PgPool 미경유 시 PgPool 계층이 제거되므로, DB `idle_session_timeout`을 30min으로 조정하여 방화벽 타임아웃과 정렬. HikariCP `maxLifetime`(27min)이 DB 타임아웃(30min)보다 짧게 유지.

### 5.3 WAS -> MongoDB (Replica Set)

```
WAS HikariCP / MongoDB Driver maxLifetime (1,620,000ms = 27min)
     |
     v
MongoDB connectionPool maxIdleTimeMS (1,800,000ms = 30min)
     |
     v
MongoDB driver socketTimeoutMS (0 = 무제한, 애플리케이션 레벨 제어)
```

### 5.4 WAS -> mongos -> MongoDB (Sharded)

```
WAS MongoDB Driver maxLifetime (1,620,000ms = 27min)
     |
     v
mongos connectionPool maxIdleTimeMS (1,800,000ms = 30min)
     |
     v
mongos -> Shard 내부 연결 (maxIdleTimeMS 30min)
     |
     v
방화벽 TCP Established Timeout (30min / 1,800s)
```

> **[참고사항]** Sharded Cluster에서 WAS는 mongos에 연결. mongos가 내부적으로 샤드 연결을 관리하므로, WAS -> mongos 간 타임아웃만 설정하면 됨. mongos 수는 WAS 연결 수에 비례하여 증설.

### 5.5 PostgreSQL 내부 타임아웃 Guardrails

> 각 파라미터는 종속 관계가 아닌, 서로 다른 시점에 동작하는 **독립적인 가드레일**임.

```
PostgreSQL Session Timeout Guardrails
  |
  |-- statement_timeout (30s)
  |     쿼리가 실행 중인 상태(Active Query)의 최대 지속 시간 제어.
  |     장기 실행 쿼리로 인한 리소스 독점 방지.
  |     |
  |     +-- lock_timeout (10s)
  |            statement_timeout 실행 도중 Lock 대기 시간에만 관여.
  |            Lock 획득 대기 10초 초과 시 자동 취소 (교착 방지).
  |            lock_timeout > statement_timeout 설정은 의미 없음.
  |
  |-- idle_in_transaction_session_timeout (60s)
        트랜잭션 시작 후 쿼리 수행이 완료된 상태에서
        이후 아무런 쿼리도 없이 유휴(Idle in Transaction) 상태로
        머무는 시간 제어. BEGIN 이후 COMMIT/ROLLBACK 없이
        방치되는 세션의 Lock 점유 및 커넥션 낭비 방지.
```

---

## 6. 실무 설정 스크립트

### 6.1 PostgreSQL - Standalone (`postgresql.conf`)

```conf
# -------------------------------------------------------
# Memory (8GB DB 전용 서버 기준)
# -------------------------------------------------------
shared_buffers = 2GB                # [높음] RAM * 0.25
effective_cache_size = 6GB          # [높음] RAM * 0.75
work_mem = 10MB                     # [높음] (RAM - shared_buffers) / (max_conn * 3)
maintenance_work_mem = 384MB        # [중간] RAM * 0.05
wal_buffers = 16MB                  # [중간] 고정

# -------------------------------------------------------
# WAL (Standalone 최적화)
# -------------------------------------------------------
wal_level = replica                 # [높음] 기본 표준 (PITR 및 백업 허용). 순수 개발계/휘발성 데이터에 한해 minimal 허용
max_wal_senders = 0                 # [높음] 복제 연결 불필요
hot_standby = off                   # [높음] Standby 없음
max_wal_size = 2GB                  # [중간] 쓰기 빈도에 따라 상향 가능
min_wal_size = 1GB                  # [중간]
checkpoint_completion_target = 0.9  # [중간]

# -------------------------------------------------------
# Connections (소규모)
# -------------------------------------------------------
max_connections = 100               # [높음] Standalone은 소수 WAS만 연결
superuser_reserved_connections = 3  # [높음]
listen_addresses = '*'              # [높음]

# -------------------------------------------------------
# Timeouts
# -------------------------------------------------------
statement_timeout = 30000                       # [중간] 30s
lock_timeout = 10000                            # [높음] 10s
idle_in_transaction_session_timeout = 60000     # [높음] 60s
idle_session_timeout = 1800000                  # [높음] 30min (PgPool 미경유)

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

### 6.2 PostgreSQL - Streaming Replication (`postgresql.conf`)

**Primary 노드:**

```conf
# -------------------------------------------------------
# Memory (8GB DB 전용 서버 기준)
# -------------------------------------------------------
shared_buffers = 2GB
effective_cache_size = 6GB
work_mem = 10MB
maintenance_work_mem = 384MB
wal_buffers = 16MB

# -------------------------------------------------------
# WAL & Replication
# -------------------------------------------------------
wal_level = replica                 # [높음] 스트리밍 복제 필수
max_wal_senders = 3                 # [높음] Replica 수 + 여유
max_replication_slots = 5           # [높음] 복제 슬롯 최대 개수 (Replica 수 + 여유)
max_wal_size = 2GB
min_wal_size = 1GB
checkpoint_completion_target = 0.9
hot_standby = on                   # [높음] (Replica에서만 의미 있지만 Primary에도 사전 설정)
listen_addresses = '*'

# -------------------------------------------------------
# Connections
# -------------------------------------------------------
max_connections = 200               # [높음] WAS 풀 합산 * 1.5 이상
superuser_reserved_connections = 3

# -------------------------------------------------------
# Timeouts
# -------------------------------------------------------
statement_timeout = 30000
lock_timeout = 10000
idle_in_transaction_session_timeout = 60000
idle_session_timeout = 1800000                  # [높음] 30min (PgPool 미경유)

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

# -------------------------------------------------------
# Replication (Primary 설정)
# -------------------------------------------------------
# pg_hba.conf에 replication 권한 추가 필수:
# host  replication  replicator  <replica-ip>/32  md5
```

**Replica 노드 (추가 설정):**

```conf
# postgresql.conf (Replica 전용 추가 항목)
hot_standby = on                    # [높음] 읽기 쿼리 수용
listen_addresses = '*'
max_wal_senders = 3                 # [높음] 승격(Promote) 시 다운스트림 복제본 수용을 위해 Primary와 동일 설정

# Standby 신호 파일 (주의: 스트리밍 복제용 Replica에는 반드시 standby.signal만 생성)
# standby.signal -- 스트리밍 복제 지속 모드 (Replica 노드 전용)
# recovery.signal은 PITR(시점 복구) 타겟 모드 전용이므로 스트리밍 복제 레플리카에 절대 사용 금지
```

```bash
# pg_basebackup로 Replica 초기 구성
pg_basebackup -h primary-host -p 5432 -U replicator -D /var/lib/postgresql/data -Fp -Xs -P -R
```

### 6.3 PostgreSQL - PgPool+SR (`postgresql.conf` + `pgpool.conf`)

**PostgreSQL (`postgresql.conf`):**

```conf
# -------------------------------------------------------
# Memory (8GB DB 전용 서버 기준)
# -------------------------------------------------------
shared_buffers = 2GB                # [높음] RAM * 0.25
effective_cache_size = 6GB          # [높음] RAM * 0.75
work_mem = 10MB                     # [높음]
maintenance_work_mem = 384MB        # [중간]
wal_buffers = 16MB                  # [중간]

# -------------------------------------------------------
# WAL & Checkpoint
# -------------------------------------------------------
wal_level = replica                 # [높음] 스트리밍 복제
max_wal_size = 2GB                  # [높음] ~16GB RAM 기준
min_wal_size = 1GB                  # [중간]
checkpoint_completion_target = 0.9  # [중간]
max_wal_senders = 5                 # [높음] Replica + 여유

# -------------------------------------------------------
# Connections
# -------------------------------------------------------
max_connections = 200               # [높음] PgPool 백엔드 * 1.5 이상
superuser_reserved_connections = 3  # [높음]
hot_standby = on                    # [높음] Replica 읽기 쿼리 수용
listen_addresses = '*'              # [높음]

# -------------------------------------------------------
# Timeouts
# -------------------------------------------------------
statement_timeout = 30000                       # [중간] 30s
lock_timeout = 10000                            # [높음] 10s
idle_in_transaction_session_timeout = 60000     # [높음] 60s
idle_session_timeout = 1800000                  # [높음] 30min (캐스케이드 최하위)

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

**PgPool-II (`pgpool.conf`):**

```conf
# -------------------------------------------------------
# PgPool-II 전용 서버 (4GB RAM 독립 서버 기준)
# Connection Pooling
# -------------------------------------------------------
num_init_children = 120          # [높음] WAS 풀 합산 + 여유
max_pool = 1                     # [높음] 단일 DB/단일 계정
child_life_time = 1680           # [높음] 28min
connection_life_time = 1680      # [중간] 28min
client_idle_limit = 600          # [중간] 10min
reserved_connections = 1         # [높음] 관리 접속 보장

# -------------------------------------------------------
# Load Balancing
# -------------------------------------------------------
load_balance_mode = on                       # [높음] 읽기 분산
backend_clustering_mode = 'streaming_replication'  # [높음] PgPool-II v4.x+ (기존 master_slave_mode/master_slave_sub_mode 폐지)

backend_weight0 = 1              # [중간] Primary
backend_weight1 = 1              # [중간] Replica

# -------------------------------------------------------
# Health Check
# -------------------------------------------------------
health_check_period = 30         # [중간] 헬스체크 주기 (초)
health_check_timeout = 10        # [중간] 헬스체크 타임아웃 (초)
health_check_max_retries = 3     # [중간] 최대 재시도

# -------------------------------------------------------
# Watchdog (SPOF 방지)
# -------------------------------------------------------
use_watchdog = on                # [높음]
wd_hostname = 'pgpool-node1'
wd_vip = '10.0.0.100'

# -------------------------------------------------------
# Auto Failover
# -------------------------------------------------------
failover_command = '/etc/pgpool-II/failover.sh'  # [높음]
```

### 6.4 PostgreSQL - Patroni+etcd (`patroni.yml` + `postgresql.conf`)

**Patroni 설정 (`patroni.yml`):**

```yaml
scope: pg-cluster
name: pg-node1

restapi:
  listen: 0.0.0.0:8008
  connect_address: node1:8008

etcd:
  hosts: etcd1:2379,etcd2:2379,etcd3:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    synchronous_mode: true
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        wal_level: replica
        hot_standby: "on"
        max_wal_senders: 5
        max_replication_slots: 5
        wal_log_hints: "on"

postgresql:
  listen: 0.0.0.0:5432
  connect_address: node1:5432
  data_dir: /var/lib/postgresql/data
  pgpass: /tmp/pgpass0
  authentication:
    replication:
      username: replicator
      password: <replication-password>
    superuser:
      username: postgres
      password: <superuser-password>
  parameters:
    shared_buffers: "2GB"
    effective_cache_size: "6GB"
    work_mem: "10MB"
    maintenance_work_mem: "384MB"
    wal_buffers: "16MB"
    max_connections: 200
    superuser_reserved_connections: 3
    max_wal_size: "2GB"
    min_wal_size: "1GB"
    checkpoint_completion_target: 0.9
    statement_timeout: 30000
    lock_timeout: 10000
    idle_in_transaction_session_timeout: 60000
    idle_session_timeout: 1800000
    autovacuum: "on"
    autovacuum_max_workers: 3
    autovacuum_naptime: "1min"
    autovacuum_vacuum_scale_factor: 0.1
    autovacuum_vacuum_cost_limit: 2000
    random_page_cost: 1.1
    effective_io_concurrency: 200

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
```

> **[참고사항]** Patroni 환경에서는 `postgresql.conf` 직접 편집 지양. 모든 PG 파라미터는 `patroni.yml`의 `postgresql.parameters`에서 관리. Patroni가 노드 역할(Leader/Replica) 전환 시 자동 적용.

**HAProxy 설정 (`haproxy.cfg`) 발췌:**

```conf
listen postgres
    bind *:5000
    option httpchk GET /primary
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server node1 node1:5432 maxconn 200 check port 8008
    server node2 node2:5432 maxconn 200 check port 8008
    server node3 node3:5432 maxconn 200 check port 8008

listen postgres-replicas
    bind *:5001
    option httpchk GET /replica
    http-check expect status 200
    balance roundrobin
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server node2 node2:5432 maxconn 200 check port 8008
    server node3 node3:5432 maxconn 200 check port 8008
```

### 6.5 MongoDB - Standalone (`mongod.conf`)

```yaml
# -------------------------------------------------------
# Storage (8GB DB 전용 서버 기준)
# -------------------------------------------------------
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 3.5            # [높음] 0.5 * (8 - 1)

# -------------------------------------------------------
# Network
# -------------------------------------------------------
net:
  port: 27017
  bindIp: 0.0.0.0
  maxIncomingConnections: 65536

# -------------------------------------------------------
# Logging
# -------------------------------------------------------
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true

# -------------------------------------------------------
# Profiling
# -------------------------------------------------------
operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100
```

> **[주의]** Standalone은 `replication` 섹션 미설정. 멀티 도큐먼트 트랜잭션 미지원.

### 6.6 MongoDB - Replica Set (`mongod.conf` + 초기화 스크립트)

**`mongod.conf` (각 노드 공통, 8GB DB 전용 서버 기준):**

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
  mode: slowOp                    # [높음]
  slowOpThresholdMs: 100          # [높음]

# -------------------------------------------------------
# Network
# -------------------------------------------------------
net:
  port: 27017
  bindIp: 0.0.0.0
  maxIncomingConnections: 65536

# -------------------------------------------------------
# Security (프로덕션 필수)
# -------------------------------------------------------
security:
  keyFile: /etc/mongodb/keyfile   # [높음] 멤버 간 인증
  authorization: enabled          # [높음] 클라이언트 인증

# -------------------------------------------------------
# Logging
# -------------------------------------------------------
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
```

**Replica Set 초기화 (mongosh):**

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

// Write Concern / Read Preference 설정 (연결 문자열 예시)
// mongodb://user:pass@mongo-primary:27017,mongo-secondary1:27017,mongo-secondary2:27017/?replicaSet=rs0&w=majority&readPreference=primary
```

**PSA 구성 (예외적 허용 시):**

```javascript
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo-primary:27017", priority: 2 },
    { _id: 1, host: "mongo-secondary:27017", priority: 1 },
    { _id: 2, host: "mongo-arbiter:27017", arbiterOnly: true }
  ]
})
```

> **[PSA 구조 치명적 제약 경고]** PSA 구성에서 Secondary 노드 1대가 다운되면 과반수 합의가 불가능해져 `w:majority` 쓰기 트랜잭션이 영구 정지(Stall)됨. 정산, 결제 등 트랜잭션 정합성이 필수적인 도메인에는 PSA 구성을 절대 금지하며, 반드시 PSS 구성을 준수해야 함. PSA는 하드웨어 극도 제약 환경에서 데이터 유실 허용이 가능한 도메인에 한해서만 예외적으로 허용함.

### 6.7 MongoDB - Sharded Cluster (`mongod.conf` for shard/config/mongos)

**Shard 노드 `mongod.conf` (Shard 1 Primary 예시, 16GB 기준):**

```yaml
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 7.5            # [높음] 0.5 * (16 - 1) = 7.5GB

replication:
  replSetName: shard1             # [높음] 샤드별 고유 RS명

sharding:
  clusterRole: shardsvr           # [높음] 샤드 역할

net:
  port: 27018                     # [높음] 샤드 포트 (27018)
  bindIp: 0.0.0.0
  maxIncomingConnections: 65536

operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100

security:
  keyFile: /etc/mongodb/keyfile
  authorization: enabled

systemLog:
  destination: file
  path: /var/log/mongodb/shard1.log
  logAppend: true
```

**Config Server `mongod.conf`:**

```yaml
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 3.5            # [높음] Config Server는 소규모 (8GB 기준)

replication:
  replSetName: cfgRS              # [높음] Config RS명

sharding:
  clusterRole: configsvr          # [높음] Config Server 역할

net:
  port: 27019                     # [높음] Config 포트 (27019)
  bindIp: 0.0.0.0
  maxIncomingConnections: 65536

security:
  keyFile: /etc/mongodb/keyfile
  authorization: enabled

systemLog:
  destination: file
  path: /var/log/mongodb/config.log
  logAppend: true
```

**mongos 라우터 `mongos.conf`:**

```yaml
net:
  port: 27017                     # [높음] mongos 포트
  bindIp: 0.0.0.0
  maxIncomingConnections: 65536

sharding:
  configDB: cfgRS/config1:27019,config2:27019,config3:27019  # [높음] Config RS 연결

security:
  keyFile: /etc/mongodb/keyfile

systemLog:
  destination: file
  path: /var/log/mongodb/mongos.log
  logAppend: true
```

**Sharded Cluster 초기화 (mongosh on mongos):**

```javascript
// mongos에 접속 후 샤드 추가
sh.addShard("shard1/shard1-primary:27018,shard1-secondary1:27018,shard1-secondary2:27018")
sh.addShard("shard2/shard2-primary:27018,shard2-secondary1:27018,shard2-secondary2:27018")

// 데이터베이스 샤딩 활성화
sh.enableSharding("mydb")

// 컬렉션 샤딩 (Shard Key 지정)
sh.shardCollection("mydb.orders", { customerId: 1, createdAt: 1 })

// 주의: numInitialChunks 옵션은 MongoDB 8.0에서 제거됨.
// 초기 청크 분산은 MongoDB가 자동으로 처리.

// Balancer 상태 확인
sh.getBalancerState()

// Chunk Size 변경 (필요 시, 글로벌 설정)
use config                       // config 데이터베이스 컨텍스트 전환 필수
db.settings.updateOne(
  { _id: "chunksize" },
  { $set: { value: 128 } },
  { upsert: true }
)

// 변경 확인
db.settings.find({ _id: "chunksize" })

// 고아(Orphan) 문서 정리 (MongoDB 8.0 변경)
// 기존: db.runCommand({ cleanupOrphaned: "mydb.orders" })  -- 8.0에서 deprecated
// 8.0+ 권장: $shardedDataDistribution 집계 파이프라인으로 고아 문서 확인

// [클러스터 전체 메타데이터 조회]
db.getSiblingDB("config").getCollection("collections").aggregate([
  { $shardedDataDistribution: {} }
])

// [특정 컬렉션 대상 고아 문서 및 데이터 분산 현황 확인]
// mydb 컨텍스트 전환 후 해당 컬렉션의 샤드별 청크/문서 분포를 직관적으로 조회
use mydb
db.orders.aggregate([{ $shardedDataDistribution: {} }])
// 출력 예시: 각 샤드별 documentCount, orphanDocumentCount, chunkCount 확인 가능
// orphanDocumentCount > 0 인 샤드가 있으면 고아 문서 존재 → 밸런서 동작 또는 수동 정리 검토
```

---

## 7. 공유 DB 환경 커넥션 풀 가용 가이드 (70% Ceiling Rule)

단일 DB를 여러 서비스가 공유할 때 전사 장애 전파를 막기 위한 인스턴스별 배정 수치입니다.

**대원칙**: `모든 애플리케이션의 maxPoolSize 총합 <= DB max_connections * 0.7`

### 7.1 PostgreSQL max_connections 산정 기준

| 전사 WAS 인스턴스 수 | WAS 풀 합산 (maxPoolSize) | 권장 max_connections | 비고 |
| :---: | :---: | :---: | :--- |
| 10개 이하 | ~200 | **200 ~ 300** | 소규모 |
| 10 ~ 20개 | 200 ~ 400 | **300 ~ 500** | 중규모 |
| 20개 이상 | 400+ | **500+** (PgPool-II 필수) | 대규모 |

### 7.2 팀별 DB 설정 점검 및 보정 내역

| 팀 / 서비스 | 사용 DBMS | 현행 주요 설정 | 표준 적용 방향 | 사유 및 보정 방향 |
| :--- | :--- | :--- | :--- | :--- |
| **플랫폼개발 (나이스M)** | PostgreSQL (via PgPool-II) + MongoDB | PgPool 경유, R:W=7:3 | 인스턴스당 **maxPoolSize=20** | PostgreSQL + MongoDB 둘 다 사용. 읽기 비중 높으므로 Replica 읽기 분산 필수 |
| **플랫폼개발 (나이스차저)** | MongoDB | P1/S2/A0, R:W=6:4 | 인스턴스당 **maxPoolSize=20~30**. **Profiling Level 1 즉시 활성화** | MongoDB만 사용. COLLSCAN 무감지 상태 운영 위험. 정산/결제 서비스는 `primary` 유지 필수 |
| **CL플랫폼** | DB2 | 현행 50 | 인스턴스당 **maxPoolSize=15** | 현금정보계와 같은 서버 사용. 향후 신규 서비스 도입 시 공유 환경 합류 대비 |
| **주차서비스** | DB2 | 현행 100 | 인스턴스당 **maxPoolSize=20~30** | 독립 DB2 환경. 과대 설정 축소 보정 |
| **현금정보계** | DB2 | 7개 컨테이너, maxPoolSize=50 | 인스턴스당 **maxPoolSize=15** (총 105) | 7개 컨테이너 다중화 환경. 향후 신규 서비스 도입 시 공유 환경 합류 대비 |

> **[참고사항]** CL플랫폼 및 현금정보계의 `maxPoolSize` 축소 적용 전, APM 모니터링을 통해 실제 피크 타임의 **Active Connection Peak 수치를 반드시 검증**해야 하며, 커넥션 고갈 우려 시 **WAS 인스턴스 스케일 아웃을 병행**해야 합니다.

### 7.3 PgPool-II 커넥션 풀 산출 예시 (플랫폼개발팀 나이스M 기준)

| 항목 | 산출 수치 | 비고 |
| :--- | :--- | :--- |
| **대상 서비스** | 나이스M (Nice M) | PostgreSQL(via PgPool-II) + MongoDB 운영 |
| **총 WAS 인스턴스 수** | 4개 (이중화) | WAS 표준 가이드라인 기준 |
| **전체 WAS 풀 합산** | **80 ~ 100개** | 인스턴스당 20~25 |
| [높음] **PgPool num_init_children** | **120** | WAS 풀 합산(100) 대비 안전 마진 |
| [높음] **PgPool max_pool** | **1** | 단일 DB/단일 계정 |
| **PgPool -> PG 이론상 최대 연결** | 120개 | `num_init_children(120) * max_pool(1)` |
| [높음] **PG 권장 max_connections** | **200** | PgPool 백엔드(120) * 1.5 이상 |

> **※ 주의**: 향후 복수 DB/계정 매핑으로 인해 `max_pool`이 2 이상으로 증가할 경우, 백엔드 이론상 최대 연결 수(`num_init_children * max_pool`)가 기하급수적으로 늘어나 PgPool 서버의 메모리(RAM) 고갈을 초래할 수 있음. 따라서 멀티 DB 환경 확장 시에는 `num_init_children` 값을 하향 조정하거나 PgPool 서버의 RAM 증설이 선행되어야 함.

### 7.4 아키텍처별 커넥션 풀 전략

| 아키텍처 | 커넥션 풀 계층 | WAS 설정 | 중간 계층 | DB 설정 | 비고 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **PG Standalone** | WAS -> PG | HikariCP maxPoolSize=15~30 | 없음 | max_conn >= Sum * 1.5 | 소규모 |
| **PG SR Only** | WAS -> PG | HikariCP maxPoolSize=15~30 | 없음 | max_conn >= Sum * 1.5 | 앱 레벨 읽기 분산 |
| **PG PgPool+SR** | WAS -> PgPool -> PG | HikariCP maxPoolSize=20~25 | PgPool num_init_children >= Sum + 여유 | max_conn >= PgPool * 1.5 | 커넥션 풀링 + 읽기 분산 통합 |
| **PG Patroni** | WAS -> HAProxy -> PG | HikariCP maxPoolSize=20~30 | HAProxy(라우팅) + PgBouncer(선택) | max_conn >= Sum * 1.5 | PgBouncer Transaction mode 시 200 직접 연결 -> 2,000+ 클라이언트 수용 가능 |
| **MongoDB RS** | WAS -> RS | maxPoolSize=20~50 (MongoDB Driver) | 없음 | maxIncomingConnections=65536 | Driver가 Primary/Secondary 자동 라우팅 |
| **MongoDB Sharded** | WAS -> mongos -> Shard | maxPoolSize=20~50 (MongoDB Driver) | mongos(라우팅) | mongos 내부 관리 | mongos 수에 비례하여 WAS 풀 분산 |

---

## 8. DB 모니터링 최소 체계

### 8.1 PostgreSQL + MongoDB 공통 모니터링

| 우선순위 | 모니터링 항목 | PostgreSQL | MongoDB | 임계치 | 조치 |
| :---: | :--- | :--- | :--- | :--- | :--- |
| [높음] | **Active Sessions** | `pg_stat_activity` | `db.serverStatus().connections` | max_connections 70% 경고 / 85% 위험 | 커넥션 풀 설정 재검토 |
| [높음] | **Slow Query** | `pg_stat_statements` (>= 1s) | Profiling Level 1 (>= 100ms) | 발생 시 즉시 분석 | 인덱스 추가 / 쿼리 튜닝 |
| [높음] | **COLLSCAN** | `seq_scan / idx_scan` 비율 | `system.profile` stage: COLLSCAN | 발생 시 즉시 조치 | 인덱스 설계 |
| [중간] | **Lock Wait** | `pg_locks` | `db.currentOp()` | 대기 시간 > 1s | 트랜잭션 분석 |
| [높음] | **Replication Lag** | `pg_stat_replication` | `rs.printSecondaryReplicationInfo()` | > 5s 경고 / > 30s 위험 | 네트워크/부하 점검 |
| [중간] | **Dead Tuples** | `pg_stat_user_tables` n_dead_tup | -- | 테이블 크기 10% 초과 | autovacuum 강제 |
| [중간] | **Cache Hit Ratio** | `pg_stat_database` (blks_hit/blks_read) | WiredTiger cache percent | < 95% 경고 | shared_buffers / cacheSizeGB 증설 검토 |

### 8.2 Sharded Cluster 추가 모니터링

| 우선순위 | 모니터링 항목 | 조회 방법 | 임계치 | 조치 |
| :---: | :--- | :--- | :--- | :--- |
| [높음] | **Balancer 상태** | `sh.getBalancerState()` + `db.currentOp()` | 비활성화 시 경고 | 활성화 및 원인 분석 |
| [높음] | **Chunk 분포** | `db.collection.getShardDistribution()` | 편차 > 20% 경고 | Shard Key 재검토 |
| [중간] | **mongos 연결 수** | `db.serverStatus().connections` (mongos) | > 10,000 경고 | mongos 인스턴스 증설 |
| [중간] | **Config Server RS 상태** | `rs.status()` on config RS | 멤버 비정상 시 즉시 | Config Server 복구 |
| [중간] | **Chunk Migration 성능** | `db.currentOp(true)` for migrate ops | 마이그레이션 지연 > 10min | 피크 시간 외 스케줄링 |

### 8.3 Patroni 클러스터 추가 모니터링

| 우선순위 | 모니터링 항목 | 조회 방법 | 임계치 | 조치 |
| :---: | :--- | :--- | :--- | :--- |
| [높음] | **etcd 클러스터 상태** | `etcdctl endpoint health` | 멤버 다운 시 즉시 | etcd 노드 복구 |
| [높음] | **Patroni Leader 상태** | `patronictl list` | Leader 없음 시 즉시 | 페일오버 또는 수동 개입 |
| [중간] | **Replication Lag (Patroni)** | `patronictl list` + `pg_stat_replication` | > 5s 경고 | 네트워크/부하 점검 |

---

## 9. 검증 체크리스트

### 9.1 공통 검증 항목 (모든 아키텍처)

| 우선순위 | 검증 항목 | 조건 | 위반 시 영향 |
| :---: | :--- | :--- | :--- |
| [높음] | shared_buffers <= RAM * 0.25 | PostgreSQL 공식 권장 | OOM, 커널 페이지 캐시 부족 |
| [높음] | max_connections >= Sum(WAS maxPoolSize) * 1.5 | 70% Ceiling Rule 역산 | 커넥션 거부, 서비스 장애 |
| [높음] | autovacuum = on | 필수 | Dead Tuple 누적, 성능 점진 저하 |
| [중간] | autovacuum_vacuum_cost_limit >= 1000 | 기본값(200) 대비 상향 | VACUUM 처리 지연 |
| [높음] | MongoDB Profiling Level >= 1 | COLLSCAN 감지 필수 | 인덱스 누락 무감지 |
| [중간] | Cache Hit Ratio >= 95% | PostgreSQL / MongoDB 공통 | 디스크 I/O 증가 |
| [높음] | idle_in_transaction_session_timeout 설정 | 교착 방지 | Lock 점유로 인한 전파 장애 |
| [중간] | random_page_cost = 1.1 (SSD) | SSD 환경 필수 | 비효율적 실행 계획 |
| [높음] | Sum(maxPoolSize) <= DB max_conn * 0.7 | 공유 DB 70% Ceiling | 타 서비스 커넥션 고갈 |

### 9.2 아키텍처별 추가 검증 항목

#### Standalone (PostgreSQL / MongoDB)

| 우선순위 | 검증 항목 | 조건 | 비고 |
| :---: | :--- | :--- | :--- |
| [높음] | wal_level = replica (PG) | 기본 표준 (PITR/백업 허용) | `minimal`은 순수 개발계 및 휘발성 데이터 장비에만 허용 |
| [높음] | max_wal_senders = 0 (PG) | 복제 불필요 | 불필요한 WAL Sender 프로세스 방지 |
| [높음] | replication 섹션 미설정 (MongoDB) | Standalone 확인 | 설정 시 인스턴스 기동 실패 가능 |

#### Streaming Replication (PostgreSQL)

| 우선순위 | 검증 항목 | 조건 | 비고 |
| :---: | :--- | :--- | :--- |
| [높음] | wal_level = replica | 복제 필수 | `minimal` 시 복제 불가 |
| [높음] | hot_standby = on (Replica) | 읽기 분산 필수 | off 시 Replica 읽기 불가 |
| [높음] | Replication Slot 구성 | pg_replication_slots 확인 | Slot 누적 시 디스크 Full 위험 |
| [중간] | 페일오버 스크립트 존재 | 수동 pg_ctl promote 테스트 | 장애 시 대응 지연 |

#### PgPool+SR (PostgreSQL)

| 우선순위 | 검증 항목 | 조건 | 비고 |
| :---: | :--- | :--- | :--- |
| [높음] | WAS maxLifetime < PgPool child_life_time < DB idle_session_timeout | 엄격 부등호 | 레이스 컨디션 |
| [높음] | PgPool reserved_connections >= 1 | 관리 접속 보장 | 장애 시 DBA 접속 불가 |
| [높음] | PgPool max_pool = 1 (단일 DB/계정) | 불필요한 커넥션 폭증 방지 | 백엔드 연결 기하급수적 증가 |
| [높음] | Watchdog 활성화 | SPOF 방지 | PgPool 단일 구성 시 전체 장애 |
| [중간] | kernel.sem 설정 (4GB 서버) | `250 32000 250 128` | 세마포어 부족 시 프로세스 생성 실패 |

#### Patroni+etcd (PostgreSQL)

| 우선순위 | 검증 항목 | 조건 | 비고 |
| :---: | :--- | :--- | :--- |
| [높음] | etcd 클러스터 정상 (3노드) | `etcdctl endpoint health` | DCS 장애 시 페일오버 불가 |
| [높음] | synchronous_mode = true | RPO = 0 보장 | 비활성화 시 페일오버 데이터 유실 가능 |
| [높음] | HAProxy 백엔드 상태 | Patroni REST API 응답 확인 | 라우팅 오류 시 WAS 장애 |
| [중간] | patroni.yml과 postgresql.conf 정합성 | Patroni가 PG 파라미터 관리 | 직접 편집 시 Patroni 덮어쓰기 |

#### Replica Set (MongoDB)

| 우선순위 | 검증 항목 | 조건 | 비고 |
| :---: | :--- | :--- | :--- |
| [높음] | Replica Set >= 3노드 (PSS 표준) | Quorum 보장 | 2노드 시 Primary 선출 불가 |
| [높음] | 정산/결제 서비스 readPreference = primary | 정합성 보장 | Secondary 읽기 시 과거 데이터 |
| [높음] | Oplog Size 확인 | `db.printReplicationInfo()` | Oplog 부족 시 복제 중단 |
| [중간] | electionTimeoutMillis >= 10000 | 기본값 유지 | 과단축 시 불필요한 페일오버 빈번 |

#### Sharded Cluster (MongoDB)

| 우선순위 | 검증 항목 | 조건 | 비고 |
| :---: | :--- | :--- | :--- |
| [높음] | Config Server RS 정상 (3노드) | `rs.status()` on config RS | Config 장애 시 전체 클러스터 불가 |
| [높음] | mongos >= 2개 (HA) | 라우터 이중화 | 단일 mongos 장애 시 서비스 중단 |
| [높음] | Shard Key 카디널리티 검증 | 분포 균등성 확인 | 편중 시 핫샤드, 성능 병목 |
| [중간] | Balancer 활성 상태 | 피크 시간 외 스케줄링 | 비활성화 시 chunk 편중 누적 |
| [중간] | Chunk Size 적정성 | 기본 64MB, 대량 쓰기 시 128MB 검토 | 과소 설정 시 마이그레이션 빈번 |

---

## 출처

- PostgreSQL 16/18 공식 문서: High Availability, Load Balancing, and Replication
- PostgreSQL 18 공식 문서: Comparison of Different Replication Solutions
- MongoDB 공식 문서: Sharding, Replica Set Deployment Architectures, Production Considerations
- MongoDB 공식 문서: Transactions Production Considerations (Standalone = 트랜잭션 미지원)
- Ashnik: "Architecting PostgreSQL HA: Patroni vs Repmgr vs Native Streaming" (2025-05)
- Percona: "Achieving PostgreSQL High Availability: Strategies and Setup Guide" (2025-03)
- pgEdge: "PostgreSQL High Availability: Strategies, Tools, and Best Practices" (2024-11)
- MongoDB Kubernetes Operator v1.31: Architecture (Standalone/ReplicaSet/ShardedCluster)
- JusDB Blog: "MongoDB Explained 2026: Replica Sets, Sharding & Production Guide" (2026-05)
- DEV Community: "PostgreSQL High Availability: A Practical Guide for Production" (2026-03)
- Grizzly Peak Software: "PostgreSQL Replication and High Availability" (2026-02)
- Linode Docs: "A Comparison of High Availability PostgreSQL Solutions" (2024-03)
- MongoDB 8.0 공식 문서: Release Notes, Compatibility Changes in MongoDB 8.0
- MongoDB 8.0 공식 문서: New Parameters (defaultMaxTimeMS, indexBuildMinAvailableDiskSpaceMB, tcmallocEnableBackgroundThread)
- MongoDB 8.3 공식 문서: Release Notes for MongoDB 8.3
- MongoDB 공식 문서: TCMalloc Performance Optimization for Self-Managed Deployment
- MongoDB 공식 문서: Reshard a Collection (forceRedistribution)
- MongoDB 공식 블로그: "MongoDB 8.0 Migration Guide: What You Need to Know Before Upgrading" (2025-02)
- MongoDB Software Lifecycle Schedules (2026-05)
