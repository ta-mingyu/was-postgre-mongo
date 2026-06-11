# PostgreSQL 구성 아키텍처 리서치

> 리서치 일자: 2026-06-10
> 목적: db-standard-guide-v2.md 작성을 위한 PostgreSQL 배포 아키텍처 전수 조사

---

## 1. PostgreSQL 구성 방식 분류

### 1.1 Standalone (단일 노드)

| 항목 | 내용 |
|---|---|
| **개요** | 단일 PostgreSQL 인스턴스. 복제 없음, 페일오버 없음 |
| **페일오버** | 수동 (서버 재부팅 또는 백업 복원) |
| **RPO** | 마지막 백업 시점 (수 시간~수 일) |
| **RTO** | 수 분~수 시간 (장애 대응 인력 가용성에 따라) |
| **핵심 파라미터 차이** | `wal_level = minimal`, `max_wal_senders = 0`, `hot_standby = off` |
| **최소 서버** | 1대 |
| **적합 서비스** | 개발/테스트 환경, 소규모 내부 도구, RTO/RPO 무관한 로깅/통계 서비스 |
| **운영 복잡도** | 최저 |

### 1.2 Streaming Replication (Primary-Replica, 페일오버 도구 없음)

| 항목 | 내용 |
|---|---|
| **개요** | PostgreSQL 내장 스트리밍 복제. Primary 1 + Replica N. 별도 페일오버 오케스트레이션 없이 수동 또는 커스텀 스크립트로 운영 |
| **페일오버** | 수동 (`pg_ctl promote` 또는 커스텀 스크립트 + Keepalived/HAProxy) |
| **RPO** | 비동기: 마지막 WAL 전송 시점 (수 초~수 분). 동기: 0 |
| **RTO** | 수 분 (수동 개입) ~ 30초 이내 (Keepalived VIP + 스크립트 자동화 시) |
| **핵심 파라미터 차이** | `wal_level = replica`, `max_wal_senders = 3~5`, `hot_standby = on` |
| **최소 서버** | 2대 (Primary 1 + Replica 1) |
| **적합 서비스** | 중간 규모 상용 서비스, 읽기 부하 분산 필요, RTO 수 분 허용, DBA가 수동 페일오버 가능한 환경 |
| **운영 복잡도** | 낮음 |
| **강점** | PostgreSQL 내장 기능만 사용, 외부 의존성 최소 |
| **약점** | 페일오버 자동화 불가 (별도 도구 필요), 스플릿 브레인 수동 해결 |

### 1.3 PgPool-II + Streaming Replication

| 항목 | 내용 |
|---|---|
| **개요** | PgPool-II가 WAS와 PostgreSQL 사이에서 커넥션 풀링, 읽기/쓰기 분리, 자동 페일오버, 헬스체크 수행 |
| **페일오버** | 자동 (Watchdog VIP 기반, `failover_command` 스크립트) |
| **RPO** | 비동기 복제: 수 초. 동기 복제 설정 시: 0 |
| **RTO** | 10~30초 (Watchdog VIP 전환 + Replica 승격) |
| **핵심 파라미터 차이** | Primary/Replica: `wal_level = replica`, PgPool: `num_init_children`, `load_balance_mode`, `master_slave_mode` |
| **최소 서버** | 2~3대 PG + 2대 PgPool (Active-Standby) |
| **적합 서비스** | 중~대규모 상용 서비스, 다수 WAS 인스턴스가 DB 공유, 읽기 분산 필요, RTO 30초 이내 |
| **운영 복잡도** | 중간 |
| **강점** | 커넥션 풀링 + 읽기 분산 + 페일오버 통합, SQL 레벨 라우팅 (SELECT -> Replica 자동 분산) |
| **약점** | PgPool 자체가 SPOF 가능성 (Watchdog 2중화 필수), 쿼리 파싱 오버헤드, 설정 복잡도 |

### 1.4 Patroni + etcd/Consul + HAProxy

| 항목 | 내용 |
|---|---|
| **개요** | DCS(Distributed Configuration Store) 기반 완전 자동 페일오버. Patroni 에이전트가 각 PG 노드에서 구동, etcd/Consul로 분산 합의 |
| **페일오버** | 완전 자동 (분산 합의 기반, 스플릿 브레인 원천 차단) |
| **RPO** | 동기 복제: 0. 비동기: maximum_lag_on_failover 설정으로 제어 |
| **RTO** | 10~30초 (etcd 리더 키 만료 + 새 Primary 선출 + HAProxy 라우팅 전환) |
| **핵심 파라미터 차이** | Patroni YAML: `synchronous_mode`, `maximum_lag_on_failover`. PG: Patroni가 자동 관리 |
| **최소 서버** | 3대 PG + 3대 etcd (PG 노드에 병설 가능) + 1~2대 HAProxy |
| **적합 서비스** | 미션 크리티컬 상용 서비스, RTO < 30초, RPO = 0, 쿠버네티스/클라우드 네이티브 환경 |
| **운영 복잡도** | 높음 |
| **강점** | 완전 자동 페일오버, 스플릿 브레인 원천 방지, K8s 호환 (Zalando PostgreSQL Operator), REST API |
| **약점** | DCS(etcd/Consul) 추가 운영 부담, 설정 복잡도 높음, 최소 5~6대 서버 필요 |

### 1.5 repmgr (Replication Manager)

| 항목 | 내용 |
|---|---|
| **개요** | PostgreSQL 전용 복제 관리 도구. etcd 없이 witness 노드 패턴으로 페일오버 수행 |
| **페일오버** | 자동 (repmgrd 데몬, witness 노드 기반) |
| **RPO** | 비동기: 수 초. 동기 설정 가능 |
| **RTO** | 30초~수 분 |
| **최소 서버** | 2대 PG + 1대 witness |
| **적합 서비스** | 중간 규모, Patroni는 과한 환경, DBA 팀이 etcd 운영 부담을 원하지 않는 경우 |
| **운영 복잡도** | 중간 |
| **강점** | etcd 불필요, 설정 상대적 간단, 기본 토폴로지 가시성 |
| **약점** | Patroni 대비 기능 한계, 스플릿 브레인 방어 약함, K8s 미지원, 커뮤니티 축소 추세 |

### 1.6 PgBouncer (커넥션 풀러 전용)

| 항목 | 내용 |
|---|---|
| **개요** | 경량 커넥션 풀러. PgPool-II의 커넥션 풀링 기능만 필요한 경우 대안 |
| **페일오버** | 없음 (HAProxy/VIP와 조합 필요) |
| **모드** | Session pooling, Transaction pooling, Statement pooling |
| **적합 서비스** | 커넥션 수가 많고 PgPool-II의 쿼리 파싱 오버헤드가 부담되는 환경 |
| **운영 복잡도** | 낮음 |
| **강점** | 매우 경량, Transaction mode 시 200 직접 연결 -> 2,000+ 클라이언트 수용 |
| **약점** | 읽기 분산 없음, 페일오버 없음, Prepared Statement 제한 (Transaction mode) |

---

## 2. 아키텍처 비교 매트릭스

| 기준 | Standalone | SR Only | PgPool+SR | Patroni+etcd | repmgr |
|---|---|---|---|---|---|
| **페일오버** | 수동 | 수동/스크립트 | 자동 (VIP) | 완전 자동 (DCS) | 자동 |
| **RPO** | 백업 시점 | 수 초~0 | 수 초~0 | 0 (동기 시) | 수 초~0 |
| **RTO** | 수 시간 | 수 분~30초 | 10~30초 | 10~30초 | 30초~수 분 |
| **읽기 분산** | 불가 | 앱 레벨 | PgPool 자동 | HAProxy | 수동 |
| **커넥션 풀** | 없음 | 없음 | PgPool 내장 | PgBouncer 별도 | 없음 |
| **스플릿 브레인 방어** | N/A | 없음 | Watchdog | etcd 분산 락 | witness 노드 |
| **최소 노드** | 1 | 2 | 4+ | 5+ | 3 |
| **운영 복잡도** | 최저 | 낮음 | 중간 | 높음 | 중간 |
| **K8s 호환** | 가능 | 가능 | 제한적 | 완전 | 불가 |

---

## 3. 서비스 특성 -> 아키텍처 매핑 기준

| 서비스 유형 | 트래픽 | RTO | RPO | 추천 아키텍처 | 사유 |
|---|---|---|---|---|---|
| 개발/테스트 | 극소 | 무관 | 무관 | **Standalone** | 단순성 최우선, HA 불필요 |
| 소규모 내부 도구 | 소 | 수 시간 | 수 시간 | **Standalone** | 장애 시 수동 복구 가능 |
| 중간 규모 상용 (RTO 수 분 허용) | 중 | 수 분 | 수 초 | **SR Only** | DBA 수동 페일오버 가능, 외부 도구 최소 |
| 중간 규모 상용 (자동 페일오버) | 중 | 30초 | 수 초 | **PgPool+SR** | 읽기 분산 + 커넥션 풀 통합 |
| 대규모 상용 (다수 서비스 공유 DB) | 대 | 30초 | 수 초 | **PgPool+SR** | 커넥션 풀링 필수, 읽기 분산으로 부하 분산 |
| 미션 크리티컬 (RPO=0) | 대 | <30초 | 0 | **Patroni+etcd** | 동기 복제 + 완전 자동 페일오버 |
| 쿠버네티스 환경 | 변동 | <30초 | 0 | **Patroni+etcd** | Zalando Operator 연동, 클라우드 네이티브 |

---

## 4. 아키텍처별 파라미터 차이

### 4.1 `wal_level`

| 아키텍처 | 설정값 | 사유 |
|---|---|---|
| Standalone | `minimal` | 복제 불필요, WAL 최소화로 쓰기 성능 극대화 |
| SR / PgPool / Patroni / repmgr | `replica` | 스트리밍 복제 필수 |
| 논리 복제 필요 시 | `logical` | 논리 복제(Per-table) 사용 시 |

### 4.2 `max_wal_senders`

| 아키텍처 | 설정값 |
|---|---|
| Standalone | `0` |
| SR (2노드) | `3` |
| PgPool+SR / Patroni | `3 ~ 5` |
| 다수 Replica | `5 ~ 10` |

### 4.3 `hot_standby`

| 아키텍처 | 설정값 |
|---|---|
| Standalone | `off` |
| SR / PgPool / Patroni / repmgr | `on` (Replica 노드) |

### 4.4 커넥션 풀 전략

| 아키텍처 | 커넥션 풀 | 비고 |
|---|---|---|
| Standalone | HikariCP (WAS 수준) | 애플리케이션 풀만으로 충분 |
| SR Only | HikariCP (WAS 수준) | 중간 계층 없이 앱 직접 연결 |
| PgPool+SR | PgPool-II | PgPool이 풀링 + 라우팅 통합 |
| Patroni+PgBouncer | PgBouncer (Transaction mode) | 경량 풀러 + HAProxy 라우팅 |
| Patroni 단독 | HikariCP (WAS 수준) | 소규모에서는 중간 계층 생략 가능 |

---

## 5. 출처

- PostgreSQL 16 공식 문서: High Availability, Load Balancing, and Replication
- PostgreSQL 18 공식 문서: Comparison of Different Replication Solutions
- Ashnik: "Architecting PostgreSQL HA: Patroni vs Repmgr vs Native Streaming" (2025-05)
- Percona: "Achieving PostgreSQL High Availability: Strategies and Setup Guide" (2025-03)
- pgEdge: "PostgreSQL High Availability: Strategies, Tools, and Best Practices" (2024-11)
- DEV Community: "PostgreSQL High Availability: A Practical Guide for Production" (2026-03)
- Grizzly Peak Software: "PostgreSQL Replication and High Availability" (2026-02)
- Linode Docs: "A Comparison of High Availability PostgreSQL Solutions" (2024-03)
