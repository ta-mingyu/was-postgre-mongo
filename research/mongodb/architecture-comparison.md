# MongoDB 구성 아키텍처 리서치

> 리서치 일자: 2026-06-10
> 목적: db-standard-guide-v2.md 작성을 위한 MongoDB 배포 아키텍처 전수 조사

---

## 1. MongoDB 구성 방식 분류

### 1.1 Standalone (단일 노드)

| 항목 | 내용 |
|---|---|
| **개요** | 단일 `mongod` 인스턴스. 복제 없음, 페일오버 없음 |
| **페일오버** | 없음 (서버 다운 = 서비스 중단) |
| **RPO** | 마지막 백업 시점 |
| **RTO** | 수 시간 (서버 복구 또는 백업 복원) |
| **트랜잭션** | **미지원** (MongoDB Standalone은 멀티 도큐먼트 트랜잭션 불가) |
| **핵심 파라미터** | `storage.wiredTiger.engineConfig.cacheSizeGB`, `net.maxIncomingConnections` |
| **최소 서버** | 1대 |
| **적합 서비스** | 개발/테스트, 프로토타입, 로컬 캐시, RTO/RPO 무관한 임시 데이터 |
| **운영 복잡도** | 최저 |
| **MongoDB 공식 입장** | "Replica sets are the basis for all production deployments" -- 프로덕션 사용 비권장 |

### 1.2 Replica Set (PSS / PSA)

| 항목 | 내용 |
|---|---|
| **개요** | 1 Primary + N Secondary 그룹. 자동 선거(Election)로 페일오버. 모든 프로덕션 배포의 기본 단위 |
| **페일오버** | 자동 (Primary 장애 시 5~15초 이내 새 Primary 선출) |
| **RPO** | 비동기 복제: 마지막 oplog 적용 시점 (수 초). `w: majority`: 0 (Majority 확인 후 응답) |
| **RTO** | 5~15초 (electionTimeoutMillis 기본 10초) |
| **트랜잭션** | **지원** (4.0+, 8.x 기본 지원) |
| **핵심 파라미터** | `cacheSizeGB`, `replSetName`, `writeConcern`, `readPreference`, Profiling Level |
| **최소 서버** | 3대 (PSS 표준) / 3대 (PSA, 하드웨어 제약 시) |
| **적합 서비스** | 대부분의 상용 서비스. HA 필수, 트랜잭션 필요, 데이터 규모 < 1TB, 쓰기 TPS < 20,000 |
| **운영 복잡도** | 중간 |

#### Replica Set 변형

| 변형 | 구성 | 장점 | 단점 | 적합 환경 |
|---|---|---|---|---|
| **PSS (표준)** | Primary 1 + Secondary 2 | 데이터 3중 복제, 읽기 분산 | 서버 3대 필요 | 프로덕션 표준 |
| **PSA (예외)** | Primary 1 + Secondary 1 + Arbiter 1 | 서버 자원 절약 | Arbiter는 데이터 미보관, 투표만 수행 | 하드웨어 극도 제약 |
| **다중 DC** | DC별 노드 분산 | DC 장애 대비 | 네트워크 레이턴시 | 재해 복구 (DR) 필요 시 |
| **Hidden Member** | 숨겨진 Secondary | 리포팅/백업 전용, 트래픽 영향 없음 | 추가 서버 비용 | 분석/백업 전용 노드 |

### 1.3 Sharded Cluster

| 항목 | 내용 |
|---|---|
| **개요** | 여러 Replica Set(Shard)에 데이터를 분산. mongos 라우터 + Config Server + Shard로 구성 |
| **페일오버** | 샤드(Replica Set) 단위 자동. mongos 다중화 시 라우터도 HA |
| **RPO** | 샤드별 Replica Set의 복제 정책에 따름 |
| **RTO** | 샤드 장애: 해당 샤드 데이터만 일시 불가. 전체 클러스터 다운: 없음 (다중 mongos) |
| **트랜잭션** | **지원** (4.2+, 분산 트랜잭션, 8.x 안정화) |
| **핵심 파라미터** | Shard Key, `chunkSize`, `balancer` 설정, Config Server RS |
| **최소 서버** | 9+ (Shard 2개 x RS 3 + Config RS 3 + mongos 1~2) |
| **적합 서비스** | 대규모 (>1TB), 초고속 쓰기 (>20k wps), 수평 확장 필수, 단일 서버 리소스 한계 도달 |
| **운영 복잡도** | 높음 |

#### Sharded Cluster 구성 요소

| 컴포넌트 | 역할 | 프로덕션 최소 |
|---|---|---|
| **Shard** | 데이터의 부분 집합 보관. 각 Shard는 Replica Set | 2개 Shard x 3노드 RS |
| **Config Server** | 클러스터 메타데이터/설정 저장. RS로 배포 | 3노드 RS |
| **mongos** | 클라이언트-클러스터 간 쿼리 라우터 | 2개 이상 (HA) |

> **[MongoDB 8.0 변경]** 8.0부터 샤드 노드에 직접 연결하여 명령을 실행할 수 없음. `mongos`를 경유하거나 `directShardOperations` 유지보수 전용 역할이 필요. 단, 1-shard 클러스터(Replica Set 전환 직후)에서는 예외적으로 허용.

---

## 2. 아키텍처 비교 매트릭스

| 기준 | Standalone | Replica Set (PSS) | Sharded Cluster |
|---|---|---|---|
| **페일오버** | 없음 | 자동 (5~15초) | 샤드 단위 자동 |
| **RPO** | 백업 시점 | 수 초~0 | 샤드별 수 초~0 |
| **RTO** | 수 시간 | 5~15초 | 샤드 장애 시 해당 샤드만 영향 |
| **데이터 용량 한계** | 단일 서버 디스크 | 단일 서버 디스크 | 이론적 무제한 (샤드 추가) |
| **쓰기 TPS 한계** | 단일 Primary | 단일 Primary (~20k wps) | 샤드 수 x Primary TPS |
| **읽기 확장** | 불가 | Secondary 읽기 (readPreference) | 샤드 + Secondary 동시 활용 |
| **트랜잭션** | 미지원 | 지원 (4.0+) | 지원 (4.2+, 분산) |
| **최소 노드** | 1 | 3 | 9+ |
| **운영 복잡도** | 최저 | 중간 | 높음 |
| **Shard Key 선택** | N/A | N/A | **필수** (잘못된 선택 시 되돌리기 어려움) |

---

## 3. 샤딩 도입 시점 판단 기준

MongoDB 공식 및 실무 권장: **단일 Replica Set이 한계에 도달하기 전까지 샤딩 도입을 지양**.

| 기준 | 단일 RS로 충분 | 샤딩 검색 필요 |
|---|---|---|
| **데이터 크기** | < 1TB | > 1TB 또는 사용 가능한 최대 디스크 도달 예상 |
| **쓰기 TPS** | < 20,000 wps | > 20,000 wps (워크로드에 따라 50k까지 가능) |
| **Working Set** | RAM 내 수용 가능 | RAM 초과, 캐시 적중률 < 95% |
| **연결 수** | < 10,000 동시 | > 10,000 동시 |
| **인덱스 크기** | RAM 내 수용 | 단일 노드 RAM 초과 |

> **Shard Key 주의**: Shard Key 선택은 신중해야 함. MongoDB 5.0부터 resharding 지원, 8.0부터 동일 Shard Key로 resharding(`forceRedistribution: true`) 및 데이터 분산 최대 50배 개선. 그러나 resharding은 여전히 리소스 집약적이므로, 카디널리티, 분포 균등성, 쿼리 패턴을 종합 분석 후 초기 설계에서 최적 Key 선택이 중요.

---

## 4. 서비스 특성 -> 아키텍처 매핑 기준

| 서비스 유형 | 데이터 규모 | TPS | RTO | 추천 아키텍처 | 사유 |
|---|---|---|---|---|---|
| 개발/테스트 | 소 | 극소 | 무관 | **Standalone** | 단순성, 빠른 프로비저닝 |
| 프로토타입/MVP | 소 | 소 | 수 분 | **Standalone** -> **Replica Set** 전환 계획 | 초기 단순, 사용자 증가 시 RS 전환 |
| 소규모 상용 (HA 필요) | 중 | 중 | 15초 | **Replica Set (PSS)** | 프로덕션 기본 요구사항 |
| 중간 규모 상용 (읽기 분산) | 중 | 중 | 15초 | **Replica Set + Secondary 읽기** | readPreference 활용 |
| 대규모 상용 (데이터 > 1TB) | 대 | 대 | 15초 | **Sharded Cluster** | 단일 노드 용량 한계 |
| 초고속 쓰기 (>20k wps) | 변동 | 초대 | 15초 | **Sharded Cluster** | 쓰기 수평 분산 |
| 분석/리포팅 전용 | 대 | 낮음 | 무관 | **Replica Set + Hidden Member** | Primary/Secondary에 영향 없는 전용 분석 노드 |

---

## 5. Replica Set 구성별 파라미터 차이

### 5.1 WiredTiger 캐시

| 구성 | cacheSizeGB 공식 | 비고 |
|---|---|---|
| Standalone | `0.5 * (RAM - 1GB)` | DB 전용 서버 기준 |
| Replica Set | `0.5 * (RAM - 1GB)` | Standalone과 동일 |
| Sharded Cluster (Shard 노드) | `0.5 * (RAM - 1GB)` | 샤드당 독립 산정 |
| Config Server | `0.5 * (RAM - 1GB)` | 일반적으로 소규모 |
| 공유 환경 (WAS+DB 혼합) | `RAM * 0.25` | 명시적 제한 필수 |

### 5.2 MongoDB 8.0 스토리지/메모리 관리 변경사항

| 항목 | 변경 내용 | 비고 |
|---|---|---|
| **TCMalloc 업그레이드** | per-Thread 캐시 → **per-CPU 캐시** 전환 | 메모리 단편화 감소, 고부하 워크로드 안정성 향상 |
| **tcmallocEnableBackgroundThread** | 8.0부터 **기본 활성화** | 주기적으로 OS로 메모리 반환 |
| **tcmallocReleaseRate** | 기본값 `1` → **`0`** 변경 (8.0) | 단위가 bytes/sec로 변경. 0 = 자동 해제 비활성화 |
| **cacheSizePct** | 8.0 신규 파라미터 | `cacheSizeGB` 대신 **퍼센트 기반** 캐시 사이징 지원 |
| **Linux Kernel 6.19** | 초기 비호환 → **8.0.4에서 패치 완료** | 현재 정상 동작 |

> **참고**: TCMalloc per-CPU 캐시 도입으로 인해 `w:majority` Write Concern의 ack 반환 시점이 변경됨. 8.0부터 majority 노드가 oplog 엔트리를 **write한 시점**에 ack 반환 (기존: oplog **적용 완료** 후). 이로 인해 `w:majority` 쓰기 성능이 향상됨.

### 5.3 Write Concern / Read Preference

| 서비스 유형 | writeConcern | readPreference | 사유 |
|---|---|---|---|
| 정산/결제 (정합성 필수) | `w: majority` | `primary` | 데이터 유실 허용 불가, 항상 최신 데이터 |
| 일반 상용 (HA 필요) | `w: 1` (기본) | `primary` | 기본 안정성 |
| 조회성 (Replication Lag 허용) | `w: 1` | `secondaryPreferred` | 읽기 부하 분산 |
| 대시보드/통계 (실시간성 낮음) | `w: 1` | `secondary` | Primary 읽기 부하 제로화 |

### 5.4 Sharded Cluster 전용 고려사항

| 항목 | 비고 |
|---|---|
| **Shard Key** | 카디널리티 높고, 쓰기 분산 균등하고, 자주 쿼리되는 필드. ESR 규칙 준수 |
| **Chunk Size** | 기본 64MB. 대량 쓰기 환경에서는 128MB로 상향 검토 |
| **Balancer** | 기본 활성. 데이터 마이그레이션 중 성능 영향 가능, 피크 시간 외 스케줄링 권장 |
| **mongos 배치** | 애플리케이션 서버에 병설 또는 전용 인스턴스. 다중화 필수 |

---

## 6. 출처

- MongoDB 공식 문서: Sharding, Replica Set Deployment Architectures, Production Considerations
- MongoDB 공식 문서: Transactions Production Considerations (Standalone = 트랜잭션 미지원)
- MongoDB 공식 문서: Replica Set Deployment Architectures (3-member 표준)
- MongoDB Kubernetes Operator v1.31: Architecture (Standalone/ReplicaSet/ShardedCluster)
- JusDB Blog: "MongoDB Explained 2026: Replica Sets, Sharding & Production Guide" (2026-05)
- MongoDB 8.0 공식 문서: Release Notes, Compatibility Changes in MongoDB 8.0
- MongoDB 8.3 공식 문서: Release Notes for MongoDB 8.3
- MongoDB 공식 문서: TCMalloc Performance Optimization for Self-Managed Deployment
- MongoDB 공식 블로그: "MongoDB 8.0 Migration Guide" (2025-02)
