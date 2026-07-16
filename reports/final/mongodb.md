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

---

## 6. 운영서버 적용 가이드: 무중단 롤링 restart 절차

> **검증 기준**: MongoDB 8.0 LTS 공식 문서 (perform-maintenance-on-replica-set-members, setParameter, rs.stepDown).
> **전제 아키텍처**: §0의 Replica Set PSS (Primary 1 + Secondary 2).
> **대칭 문서**: `reports/final/postgresql.md` §6, `reports/final/pgpool-ii.md` §6.

MongoDB는 PostgreSQL/PgPool과 달리 **런타임 파라미터 변경 명령(`setParameter`, `setClusterParameter`, `setProfilingLevel`)을 풍부하게 제공**한다. 대부분의 튜닝이 restart 없이 적용 가능하며, Replica Set 환경에서는 노드별 롤링 restart로 무중단 유지보수가 가능하다.

### 6.1 restart 시 서비스 영향

`systemctl restart mongod`는 mongod 프로세스를 종료 후 재기동한다. 종료 시점에 해당 노드의 클라이언트 연결이 단절된다.

```mermaid
sequenceDiagram
    participant Op as 운영자
    participant M as mongod
    participant RS as Replica Set
    participant WAS as WAS (Driver)

    Op->>M: systemctl restart mongod
    Note over M: graceful shutdown
    M-->>WAS: 해당 노드 커넥션 단절
    alt Primary 노드인 경우
        M->>RS: Primary 강등 (자동 또는 stepDown)
        Note over RS: 새 Primary 선거<br/>electionTimeoutMillis=10s 내
        WAS->>RS: 새 Primary로 재연결
    else Secondary 노드인 경우
        Note over RS: 과반수 유지<br/>서비스 영향 최소
    end
    Op->>M: 재기동
    M->>RS: Secondary로 재편입<br/>oplog 동기화
```

- Secondary restart: 과반수 유지되어 쓰기 서비스 영향 없음 (읽기 분산 시 일시적 해당 노드 제외 가능).
- Primary restart: `electionTimeoutMillis`(10s) 내 새 Primary 선출, 그 간 쓰기 일시 중단.
- Standalone: 서비스 완전 중단 (선거 메커니즘 없음).

### 6.2 파라미터 적용 경로 분류 (런타임 vs restart)

MongoDB는 4가지 적용 경로가 있다.

| 적용 경로 | 방법 | restart 필요? | 영구성 |
|:---|:---|:---:|:---|
| `setParameter` | `db.adminCommand({ setParameter: 1, ... })` | 아니오 | restart 시 mongod.conf 값으로 복원 (동기화 필요) |
| `setClusterParameter` (8.0+) | `db.adminCommand({ setClusterParameter: ... })` | 아니오 | 클러스터 전체 영구 (config DB 저장) |
| `setProfilingLevel` | `db.setProfilingLevel(1, { slowms: 100 })` | 아니오 | restart 시 mongod.conf 값으로 복원 |
| `mongod.conf` 변경 | 파일 수정 후 restart | 예 | 영구 |

§2.1/§2.4 표준 파라미터의 적용 경로 분류:

| 파라미터 | 적용 경로 | restart 없이 변경? |
|:---|:---|:---:|
| `cacheSizeGB` | mongod.conf (storage.wiredTiger) | 아니오 |
| `maxIncomingConnections` | mongod.conf (net) | 아니오 |
| `net.bindIp` / `net.port` | mongod.conf (net) | 아니오 |
| `replication.replSetName` | mongod.conf | 아니오 |
| `security.keyFile` / `authorization` | mongod.conf | 아니오 |
| `operationProfiling.mode` / `slowOpThresholdMs` | mongod.conf 또는 `db.setProfilingLevel()` | 예 (런타임) |
| `internalQueryExecMaxBlockingSortBytes` | `setParameter` 또는 mongod.conf | 예 (런타임, 영구는 conf) |
| `localLogicalSessionTimeoutMinutes` | `setParameter` 또는 mongod.conf | 예 (런타임, 영구는 conf) |
| `defaultMaxTimeMS` (8.0+) | `setClusterParameter` | 예 (런타임, 영구) |
| `electionTimeoutMillis` | `rs.reconfig()` | 예 (런타임) |
| Write Concern / Read Preference | 드라이버 연결 문자열 | 예 (앱 수준) |

> 핵심: `cacheSizeGB`, `maxIncomingConnections`, `bindIp`/`port`, 보안 설정은 restart 필요. Profiling, `setParameter`, cluster parameter는 런타임 변경 가능. 단, 런타임 변경(`setParameter`)은 restart 시 mongod.conf 값으로 복원되므로 **영구 적용을 원하면 mongod.conf에도 동일 값 기록 필수**.

### 6.3 의사결정 플로우

```mermaid
flowchart TD
    Q1{변경 파라미터가<br/>런타임 명령 가능?}
    Q1 -->|예 setParameter/setClusterParameter| A[절차 A: 런타임 명령<br/>서비스 중단 없음]
    Q1 -->|아니오 mongod.conf| Q2{아키텍처?}
    Q2 -->|Replica Set PSS| B[절차 B: 롤링 restart<br/>Secondary -> stepDown -> Primary]
    Q2 -->|Standalone| C[절차 C: restart<br/>downtime 수용]
```

### 6.4 절차 A: 런타임 명령 (서비스 영향 없음) — 1순위

`cacheSizeGB`, `maxIncomingConnections`가 아닌 변경은 런타임 명령으로 해결한다. 대표적으로 `internalQueryExecMaxBlockingSortBytes`, `defaultMaxTimeMS`, profiling level, `localLogicalSessionTimeoutMinutes` 등.

```javascript
// mongosh 접속 (Primary에서 실행)

// 1. setParameter -- 즉시 적용 (해당 노드만)
db.adminCommand({ setParameter: 1, internalQueryExecMaxBlockingSortBytes: 67108864 })  // 64MB
db.adminCommand({ setParameter: 1, localLogicalSessionTimeoutMinutes: 30 })

// 2. setClusterParameter (8.0+) -- 클러스터 전체 영구 적용
db.adminCommand({ setClusterParameter: { defaultMaxTimeMS: { readOperations: 60000 } } })

// 3. Profiling Level
db.setProfilingLevel(1, { slowms: 100 })

// 4. 적용 확인
db.adminCommand({ getParameter: 1, internalQueryExecMaxBlockingSortBytes: 1 })
db.getProfilingStatus()
```

```bash
# 5. (중요) 영구 적용을 위해 mongod.conf에도 동일 값 기록
#    setParameter로 런타임 변경한 값은 restart 시 mongod.conf 값으로 복원됨
#    각 노드의 mongod.conf setParameter 섹션에 동일 값 기록 필요
```

- 서비스 영향: 없음.
- 주의: `setParameter`로 런타임 변경 후 mongod.conf를 동기화하지 않으면, 다음 restart 때 원래 값으로 복원된다. **영구 적용하려면 mongod.conf에도 기록 필수**.
- Replica Set의 경우 `setParameter`는 각 노드별로 실행 필요 (`setClusterParameter`는 클러스터 전체 자동 전파이므로 Primary에서 1회).

### 6.5 절차 B: Replica Set 롤링 restart (무중단)

`cacheSizeGB`, `maxIncomingConnections` 등 mongod.conf 파라미터 변경 시. PSS(3노드) 환경에서 노드별로 순차 restart하며 항상 과반수(2노드)를 유지한다.

```mermaid
flowchart LR
    S1[Secondary 1<br/>restart] --> W1[복제 동기화 대기<br/>lag=0 확인]
    W1 --> S2[Secondary 2<br/>restart]
    S2 --> W2[복제 동기화 대기<br/>lag=0 확인]
    W2 --> SD[Primary<br/>rs.stepDown]
    SD --> S3[구 Primary restart<br/>현재 Secondary]
    S3 --> W3[복제 동기화 대기<br/>lag=0 확인]
```

상세 절차:

```bash
# 사전 점검: 복제 상태, oplog window 확인
mongosh --host <primary> --eval '
  rs.status();
  rs.printSecondaryReplicationInfo();
  db.printReplicationInfo();
'

# --- Secondary 1 restart ---
# 1. Secondary 1에서 mongod.conf 수정 후 restart
sudo systemctl restart mongod    # Secondary 1 서버

# 2. Secondary 1이 SECONDARY 상태 + lag=0 확인 후 다음 진행
mongosh --host <primary> --eval 'rs.status().members.filter(m => m.stateStr == "SECONDARY")'

# --- Secondary 2 restart (동일 패턴) ---
sudo systemctl restart mongod    # Secondary 2 서버
# SECONDARY 상태 + lag=0 확인

# --- Primary restart ---
# 3. Primary를 Secondary로 강등 (새 Primary 선출 트리거)
mongosh --host <primary> --eval 'rs.stepDown(60, 15)'
#   stepDownSecs=60: 60초간 이 노드는 Primary 후보 제외
#   secondaryCatchUpPeriodSecs=15: Secondary가 따라잡을 때까지 최대 15초 대기

# 4. 새 Primary 선출 확인
mongosh --eval 'rs.status().members.filter(m => m.stateStr == "PRIMARY")'

# 5. 구 Primary(현재 Secondary)에서 mongod.conf 수정 후 restart
sudo systemctl restart mongod    # 구 Primary 서버

# 6. 전 노드 복구 확인 (PRIMARY 1 + SECONDARY 2, lag=0)
mongosh --host <new_primary> --eval 'rs.printSecondaryReplicationInfo()'

# 7. (선택) priority 조정으로 원래 Primary 역할 복귀
mongosh --host <new_primary> --eval '
  var cfg = rs.conf();
  cfg.members[0].priority = 2;
  cfg.members[1].priority = 1;
  cfg.members[2].priority = 1;
  rs.reconfig(cfg);
'
```

- 서비스 영향: 쓰기 중단 없음 (stepDown 시 수 초~10초 선거 간격, 드라이버가 자동 재연결).
- 과반수 유지: 항상 2노드 가동으로 쿼럼 보장.
- 전제: PSS 3노드 구성. 2노드에서는 롤링 불가 (1노드 다운 시 과반수 상실).
- 주의: 한 노드 restart 후 반드시 SECONDARY 상태 + lag=0 확인 후 다음 노드로 진행. **동시에 2노드 이상 restart 금지** (과반수 상실 → 선거 불가 → 서비스 중단).

### 6.6 절차 C: Standalone restart (개발/테스트, downtime 수용)

개발/테스트 환경의 Standalone은 복제/선거 메커니즘이 없어 롤링이 불가능하다.

```bash
# 1. mongod.conf 수정
# 2. restart
sudo systemctl restart mongod
# 3. 기동 확인
mongosh --eval 'db.adminCommand({ ping: 1 })'
```

- 서비스 영향: restart 동안 해당 인스턴스 전체 중단 (개발/테스트 환경이므로 허용).
- 프로덕션 Standalone: 허용하지 않음 (§0 아키텍처 참조, 멀티 도큐먼트 트랜잭션 미지원).

### 6.7 Replica Set 구성 정책 (PSS 의무)

무중단 롤링 restart의 전제는 **PSS(Primary 1 + Secondary 2) 3노드 구성**이다.

| 구성 | 롤링 restart 가능? | 사유 |
|:---|:---:|:---|
| PSS (3노드, 표준) | 예 | 1노드씩 restart, 항상 과반수(2) 유지 |
| PSA (3노드, Arbiter 포함) | 제한적 | 데이터 보유 노드 2대. Secondary restart 시 데이터 노드 2->1, w:majority 쓰기 가능하나 내결함성 감소 |
| 2노드 | 아니오 | 1노드 다운 시 과반수(2/2) 상실, 선거 불가 |
| Standalone | 아니오 | 복제/선거 없음 |

- **정산/결제 도메인: PSS 의무, PSA 절대 금지** (PSA에서 Secondary 다운 시 w:majority 쓰기 영구 stall).
- 상세는 §0 PSS vs PSA 비교, `harness/mongodb-rules.md` §1 참조.

### 6.8 버전 업그레이드 시 동일 패턴 적용

MongoDB 버전 업그레이드(예: 8.0 -> 8.3) 시에도 절차 B와 동일한 롤링 패턴을 사용한다. 공식 권장 순서:

1. Secondary 노드부터 순차 업그레이드 (구 버전 -> 신 버전 바이너리로 restart)
2. 마지막 Primary를 `rs.stepDown()`으로 교체 후 업그레이드
3. featureCompatibilityVersion 최신으로 상향 (`db.adminCommand({ setFeatureCompatibilityVersion: "8.3" })`)

> 주의: Replica Set 내 버전 혼합 운영은 인접 메이저 버전까지만 허용 (예: 8.0 + 8.3 가능, 7.0 + 8.3 불가). 업그레이드 전 공식 호환성 매트릭스 확인 필수.

### 6.9 절차 요약표

| 상황 | 절차 | 서비스 중단 | 전제 조건 |
|:---|:---|:---:|:---|
| 런타임 명령 가능 파라미터 | A (`setParameter`/`setClusterParameter`) | 없음 | mongod.conf 동기화로 영구화 |
| Replica Set mongod.conf 변경 | B (롤링: Secondary -> stepDown -> Primary) | 없음 (수 초 선거 간격) | PSS 3노드 |
| Standalone mongod.conf 변경 | C (restart) | 전체 | 개발/테스트만 |
| 버전 업그레이드 | B (롤링 패턴 동일) | 없음 | 인접 메이저 버전 호환성 확인 |

> 핵심: MongoDB는 런타임 파라미터 변경 명령이 풍부하여 대부분의 튜닝이 restart 없이 가능 (절차 A). restart가 필요한 변경(`cacheSizeGB` 등)은 PSS Replica Set에서 롤링(절차 B), Standalone에서는 downtime 수용(절차 C). 런타임 변경 시 mongod.conf 동기화로 영구화하는 것을 잊지 말 것.
