# MongoDB 서버 설정 가이드

> **기준 문서**: `reports/final-standard-guide.md`
> **적용 범위**: MongoDB 8.0+ (WiredTiger 스토리지 엔진)
> **프로덕션 표준**: Replica Set PSS (Primary 1 + Secondary 2)
> **개발/테스트**: Standalone

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
# /etc/security/limits.d/99-mongodb.conf -- MongoDB ulimit (PAM 기반)
mongod  soft  nofile   1048576
mongod  hard  nofile   1048576
mongod  soft  nproc    65536
mongod  hard  nproc    65536
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
```

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

### 2.1 프로덕션 표준: Replica Set PSS

```mermaid
graph LR
    APP[WAS<br/>MongoDB Driver] --> MG_P[(mongod<br/>Primary)]
    MG_P -->|Oplog<br/>Replication| MG_S1[(mongod<br/>Secondary 1)]
    MG_P -->|Oplog<br/>Replication| MG_S2[(mongod<br/>Secondary 2)]
    MG_S1 -.->|Read<br/>Optional| APP
    MG_S2 -.->|Read<br/>Optional| APP
```

#### PSS vs PSA 비교

| 기준 | PSS (표준) | PSA (금지) |
|:---|:---|:---|
| 구성 | Primary 1 + Secondary 2 | Primary 1 + Secondary 1 + Arbiter 1 |
| 데이터 복제 | 3중 복제 | 2중 복제 (Arbiter는 데이터 미보관) |
| 데이터 안전성 | 1노드 장애까지 정상 서비스 유지 | Secondary 1 노드 장애 시 과반수 미달로 w:majority 쓰기 불가 (stall 장애) |

> **PSA 구조 치명적 제약**: 정산, 결제 등 트랜잭션 정합성이 필수인 도메인에는 PSA 구성 절대 금지. 반드시 PSS 구성 준수.

### 2.2 핵심 파라미터

| 파라미터 | 표준값 | 역할 |
|:---|:---|:---|
| cacheSizeGB | `0.5 * (RAM - 1)`, 32GB+는 하향 조정 | WiredTiger 내부 캐시 크기(GB). DB 전용 서버: RAM에서 1GB 제외한 50%. 과다 설정 시 OS 메모리 부족으로 스와핑 |
| replSetName | rs0 (환경에 맞게 명명) | Replica Set 식별자. 클러스터 내 모든 노드가 동일한 이름 사용 |
| Profiling Level | 1 (slowms: 100) | 슬로우 쿼리 및 COLLSCAN 감지. 100ms 이상 연산만 기록 |
| electionTimeoutMillis | 10000 (10s, 기본값) | Primary 하트비트 수신 불가 시 선거(Election) 시작 대기 시간 |
| defaultMaxTimeMS | 60000 (권장) | MongoDB 8.0 신규. 개별 읽기 연산 기본 시간 제한(ms). 장기 실행 쿼리 자원 독점 방지 |
| maxIncomingConnections | RAM별 차등 (1,000~10,000) | 최대 동시 클라이언트 연결. 커넥션당 1MB 스레드 스택. 기본(65536)은 소형 서버에서 OOM 위험 |
| internalQueryExecBlockingSortBytes | RAM별 차등 (32~256 MB) | 인덱스 없는 블로킹 정렬 시 세션당 최대 메모리. 악성 쿼리 1개 시스템 전체 고갈 방지 |

### 2.3 Write Concern / Read Preference 의사결정표

| 서비스 유형 | writeConcern | readPreference | 사유 |
|:---|:---|:---|:---|
| 정산/결제 (정합성 필수) | w: majority | primary | 데이터 유실 허용 불가, 항상 최신 데이터 보장 |
| 일반 상용 (HA 필요) | w: 1 (기본) | primary | 기본 안정성 확보 |
| 조회성 (Replication Lag 허용) | w: 1 | secondaryPreferred | 읽기 부하 분산 |
| 대시보드/통계 (실시간성 낮음) | w: 1 | secondary | Primary 읽기 부하 제로화 |

> 핵심 제약: 정산/결제 서비스는 반드시 primary readPreference 유지.
>
> MongoDB 8.0 Write Concern 동작 변경: w:majority 설정 시 8.0부터 majority 노드가 oplog 엔트리를 write한 시점에 ack 반환 (기존: 적용 완료 후 ack). 성능 향상.

### 2.4 RAM별 파라미터 매트릭스 (노드당)

| DB 서버 RAM | cacheSizeGB | maxIncomingConnections | internalQueryExecMaxBlockingSortBytes | 비고 |
|:---:|:---:|:---:|:---:|:---|
| 8 GB | 3.5 GB | 1,000 | 32 MB | PSS 3노드 각각 동일 적용 |
| 16 GB | 7.5 GB | 2,000 | 64 MB | 표준 프로덕션 |
| 32 GB | 12.0 GB | 5,000 | 128 MB | 고성능. cacheSizeGB 하향 (OS page cache 마진) |
| 64 GB | 24.0 GB | 10,000 | 256 MB | 대규모. cacheSizeGB 하향 (대량 커넥션 + page cache 마진) |

### 2.5 mongod.conf 전문 (프로덕션, 8GB 기준)

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

> 각 RAM 스펙별로 cacheSizeGB, maxIncomingConnections, internalQueryExecMaxBlockingSortBytes 값을 2.4절 매트릭스에 맞게 변경. 나머지 설정은 모든 RAM 공통.

### 2.6 Replica Set 초기화 (mongosh)

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

### 2.7 개발/테스트: Standalone

> 본 구성은 개발 및 테스트 환경에 한해서만 허용. 프로덕션에서는 Replica Set PSS 구성(2.1절) 적용 필수. MongoDB Standalone은 멀티 도큐먼트 트랜잭션을 지원하지 않음.

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

## 3. 타임아웃 & 세션 설정

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

### 3.2 유휴 세션 제한

| 파라미터 | 표준값 | 역할 |
|:---|:---|:---|
| localLogicalSessionTimeoutMinutes | 30 (30min) | 서버 내 논리 세션(Logical Session) 유휴 만료. 세션 누적으로 인한 서버 자원 고갈 방지 |

> localLogicalSessionTimeoutMinutes는 논리 세션 만료 규격일 뿐, 물리적 TCP 커넥션 유휴 상태를 직접 관리하지 않음. 물리적 유휴 커넥션은 드라이버단 maxIdleTimeMS로 제어. HikariCP maxLifetime 및 keepaliveTime과 연동하여 커넥션 풀 수준에서 관리 필요.

---

## 4. 모니터링 체크리스트

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

---

## 5. 검증 체크리스트

- [ ] Replica Set >= 3노드 (PSS 표준) -- Quorum 보장 (위반 시: 2노드 시 Primary 선출 불가)
- [ ] 정산/결제 서비스 readPreference = primary -- 정합성 보장 (위반 시: Secondary 읽기 시 과거 데이터 반환)
- [ ] Profiling Level >= 1 (slowms: 100) -- COLLSCAN 감지 필수 (위반 시: 인덱스 누락 무감지)
- [ ] Oplog Size 확인 (db.printReplicationInfo) -- 복제 안정성 (위반 시: Oplog 부족 시 복제 중단)
- [ ] electionTimeoutMillis >= 10000 -- 기본값 유지 (위반 시: 과단축 시 불필요한 페일오버 빈번)
- [ ] Cache Hit Ratio >= 95% (db.serverStatus().wiredTiger.cache) -- 캐시 효율성 (위반 시: 디스크 I/O 증가, 성능 저하)
- [ ] vm.swappiness = 1 (MongoDB 서버) -- DB 서버 안정성 (위반 시: 스와핑 시 캐시 성능 급감)
- [ ] THP = enabled (MongoDB 8.0+ 서버) -- TCMalloc per-CPU 성능 (위반 시: per-CPU 캐시 비활성화로 성능 저하)
- [ ] THP 영구 설정 적용 -- IT ONE 변경 요청 완료 여부 확인 (미적용 시: 리부팅 후 THP 비활성화로 성능 저하)
- [ ] ulimit -n = 1048576 (MongoDB 서버) -- 모든 서버 공통 (위반 시: 기동 경고, 커넥션 제한)
- [ ] systemd 서비스 LimitNOFILE/LimitNPROC/LimitFSIZE/LimitCPU override 설정 -- 서비스 데몬 ulimit 적용 (미적용 시: limits.conf 무시되어 기본값으로 동작)
- [ ] ulimit -f / -t = unlimited (MongoDB 서버) -- 대용량 데이터 보호 (위반 시: 파일 크기 제한 도달 시 데이터 손상 위험)
- [ ] security.authorization = enabled -- 클라이언트 인증 (위반 시: 인증 없는 접근 허용)
- [ ] security.keyFile 설정 -- 멤버 간 인증 (위반 시: Replica Set 내 노드 간 인증 불가)
