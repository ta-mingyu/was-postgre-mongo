# MongoDB Domain Rules

> MongoDB 표준 설정 가이드라인 작업 시 에이전트가 반드시 준수해야 할 구동 규칙.
> 기준 산출물: `reports/db-standard-guide.md` (MongoDB 섹션)
> **버전 기준**: MongoDB 8.0 LTS (온프레미스 장기지원) / MongoDB 8.3 Latest (2026-06-11 기준 최신)

---

## 1. 도메인 스코프

### 대상 시스템

| 시스템 | 대상 팀 | 설정 파일 | 비고 |
| :--- | :--- | :--- | :--- |
| **MongoDB** (Replica Set) | 플랫폼개발팀 | `mongod.conf` | Master 1 / Slave 2, R:W = 6:4 |

### 아키텍처 기준

```
WAS (HikariCP / MongoDB Driver) --> MongoDB Primary
                                        |
                                        +--> MongoDB Secondary 1
                                        +--> MongoDB Secondary 2
```

- Replica Set 표준: **PSS (Primary 1 + Secondary 2) = 최소 3노드**
- 하드웨어 극도 제약 시에만 PSA (Primary + Secondary + Arbiter) 허용
- **[PSA 경고]** PSA 구조에서 Secondary 1대 다운 시 과반수 합의 불가로 `w:majority` 쓰기 트랜잭션 영구 정지(Stall). 정산/결제 도메인에는 PSA 절대 금지, 반드시 PSS 구성 준수
- 스플릿 브레인 방지 및 Quorum 성립 보장

### 스코프 내 (IN Scope)

- WiredTiger 스토리지 엔진 캐시 설정
- Replica Set 구성 및 Write Concern / Read Preference
- Profiling 및 COLLSCAN 감지
- 인덱스 전략 (ESR 규칙)
- 커넥션 풀 및 타임아웃 설정
- 모니터링 최소 체계

### 스코프 외 (OUT of Scope)

- WAS/JVM 설정 (WAS harness 참조)
- PostgreSQL / PgPool-II 설정 (PostgreSQL harness 참조)
- DB2 내부 파라미터
- Sharding (현재 Replica Set만 운영 중)

---

## 2. 핵심 산정 공식 (절대 준수)

### 2.1 WiredTiger 캐시

```
cacheSizeGB = 0.5 * (RAM - 1GB)
```

**DB 전용 서버 기준**: RAM의 50% 수준 할당.
**공유 환경**(WAS/DB 혼합 배포): 25% 수준으로 명시적 제한 권장.

| DB 서버 RAM | cacheSizeGB | 계산 |
| :---: | :---: | :--- |
| 8 GB | 3.5 GB | 0.5 * (8 - 1) = 3.5 |
| 16 GB | 7.5 GB | 0.5 * (16 - 1) = 7.5 |
| 32 GB | 15.5 GB | 0.5 * (32 - 1) = 15.5 |
| 64 GB | 31.5 GB | 0.5 * (64 - 1) = 31.5 |

### 2.2 커넥션 풀

```
maxPoolSize = 인스턴스당 20 (WAS 표준 기준)
minPoolSize = 0 (기본)
maxIncomingConnections = 65,536 (서버 측)
```

### 2.3 Replica Set 멤버 구성 표준

| 구성 | 노드 수 | 구성원 | 허용 조건 |
| :--- | :---: | :--- | :--- |
| **PSS (표준)** | 3 | Primary 1 + Secondary 2 | 기본 표준 |
| **PSA (예외)** | 3 | Primary 1 + Secondary 1 + Arbiter 1 | 하드웨어 극도 제약 시만 |

---

## 3. Write Concern & Read Preference

### 3.1 Write Concern

| 설정값 | 적용 기준 | 비고 |
| :--- | :--- | :--- |
| `w: 1` (기본) | 일반 서비스 | Primary에만 쓰기 확인 |
| `w: majority` | 정산/결제 등 정합성 필수 서비스 | Majority 노드에 쓰기 확인 후 응답 |

### 3.2 Read Preference

| 설정값 | 적용 기준 | 비고 |
| :--- | :--- | :--- |
| `primary` (기본) | 정산/결제 등 정합성 필수 서비스 | 항상 Primary에서 읽기 |
| `secondaryPreferred` | 조회성 서비스 (Replication Lag 허용 시) | Secondary 우선, 불가 시 Primary |

**핵심 제약**: 정산/결제 서비스는 **반드시 `primary`** 유지. 과거 데이터 조회 허용 여부에 따라 선택.

---

## 4. Profiling & 인덱스 전략

### 4.1 Profiling (COLLSCAN 감지 필수)

```javascript
db.setProfilingLevel(1, { slowms: 100 })
```

| 설정 | 값 | 비고 |
| :--- | :--- | :--- |
| Profiling Level | 1 (slowOp) | Slow Query만 기록 |
| slowOpThresholdMs | 100 | 100ms 이상 쿼리 기록 |
| `operationProfiling.mode` | `slowOp` | mongod.conf 설정 |

**HITL-004**: 플랫폼개발팀 현재 COLLSCAN 모니터링 미수행. **즉시 활성화 필요**.

### 4.2 인덱스 설계 (ESR 규칙)

```
Equality (정확히 일치) -> Sort (정렬) -> Range (범위)
```

- Equality 조건을 인덱스 선행 컬럼에 배치
- Sort 조건을 중간에 배치
- Range 조건을 마지막에 배치
- COLLSCAN 발생 시 즉시 인덱스 추가

---

## 5. 타임아웃 설정

### 5.1 WAS -> MongoDB 타임아웃 캐스케이드

```
WAS HikariCP maxLifetime (27min)
    |
    v
MongoDB connectionPool maxIdleTimeMS (30min)
    |
    v
MongoDB driver socketTimeoutMS (0 = 무제한, 애플리케이션 레벨 제어)
```

### 5.2 서버 측 타임아웃

| 파라미터 | 표준값 | 비고 |
| :--- | :--- | :--- |
| `localLogicalSessionTimeoutMinutes` | 30 (30min) | 논리 세션 유휴 만료 |
| `maxIncomingConnections` | 65,536 | 서버 측 커넥션 상한 |

---

## 6. 모니터링 최소 체계 (MongoDB)

| 우선순위 | 항목 | 조회 방법 | 임계치 | 조치 |
| :---: | :--- | :--- | :--- | :--- |
| 높음 | Active Connections | `db.serverStatus().connections` | max_conn 70% 경고 / 85% 위험 | 커넥션 풀 재검토 |
| 높음 | Slow Query | Profiling Level 1 | >= 100ms | 인덱스 추가 / 쿼리 튜닝 |
| 높음 | COLLSCAN | `system.profile` stage: COLLSCAN | 발생 시 즉시 | 인덱스 설계 |
| 중간 | Lock Wait | `db.currentOp()` | > 1s | 트랜잭션 분석 |
| 높음 | Replication Lag | `rs.printSecondaryReplicationInfo()` | > 5s 경고 / > 30s 위험 | 네트워크/부하 점검 |
| 중간 | Cache Hit Ratio | WiredTiger cache percent | < 95% | cacheSizeGB 증설 검토 |
| 중간 | Oplog Buffer | `db.serverStatus().metrics.repl.buffer` | 8.0+ apply/write 분리 | 복제 지연 분석 |
| 중간 | Cost-Based Ranker | `db.serverStatus().metrics.query.cbr` | 8.3+ 쿼리 플랜 최적화 | 쿼리 성능 분석 |

---

## 7. 플랫폼개발팀 (Nice Charger) 특이사항

에이전트는 MongoDB 관련 작업 시 아래 이력을 반드시 참조.

### 현재 운영 상태

- Replica Set: Master 1 / Slave 2 (PSS 구성 -- 표준 준수)
- Read:Write = 6:4 (읽기 비중 높음)
- **COLLSCAN 모니터링 미수행** -- HITL-004 (TA 응답 대기 중)

### 즉시 적용 필요

| 항목 | 현재 | 권장 | 비고 |
| :--- | :--- | :--- | :--- |
| Profiling Level | 미설정 | **Level 1 (slowms: 100)** | COLLSCAN 감지 필수 |
| maxPoolSize | -- | **20** (인스턴스당) | 70% Ceiling Rule 준수 |
| Read Preference | -- | 서비스 특성에 따라 선택 | 정산/결제: `primary` / 조회: `secondaryPreferred` |

### 향후 보완 필요 (Phase 5)

- RAM/코어별 매트릭스 테이블 상세 수치 보완
- Write Concern 세부 가이드라인 (서비스별 분류 기준)

---

## 8. 검증 체크리스트

에이전트는 MongoDB 설정 변경 후 반드시 아래 항목을 검증.

| # | 검증 항목 | 충족 조건 |
| :---: | :--- | :--- |
| 1 | `cacheSizeGB` = 0.5 * (RAM - 1) | DB 전용 서버 기준 |
| 2 | Profiling Level >= 1 | COLLSCAN 감지 필수 |
| 3 | Replica Set >= 3노드 (PSS 표준) | Quorum 보장 |
| 4 | 정산/결제 서비스 Read Preference = `primary` | 정합성 보장 |
| 5 | Sum(maxPoolSize) <= DB max_conn * 0.7 | 70% Ceiling |
| 6 | MongoDB 버전 >= 8.0 (온프레미스 LTS) | 보안 패치 및 기능 지원 |
| 7 | Cache Hit Ratio >= 95% | 캐시 효율성 |
| 8 | Slow Query 모니터링 활성 | 성능 저하 조기 감지 |
| 9 | `defaultMaxTimeMS` 설정 검토 (8.0+) | 장기 실행 쿼리 방어 |

---

## 9. 에이전트 작업 규칙

1. **DB 전용 서버 기준**: WiredTiger 캐시 공식은 DB 전용 서버 기준. 공유 환경은 25% 수준으로 명시적 제한
2. **PSS 표준 준수**: Replica Set 구성은 PSS(3노드)를 기본으로. PSA는 예외적 허용. 단, PSA에서 Secondary 다운 시 w:majority 쓰기 영구 정지(Stall) 위험이 있으므로 정산/결제 도메인은 PSA 절대 금지
3. **COLLSCAN Zero-Tolerance**: COLLSCAN 발생 시 즉시 인덱스 추가. 프로파일링 미설정 상태는 허용하지 않음
4. **버저닝**: 가이드 갱신 시 `reports/db-standard-guide-v{N}.md` 생성 후 CLAUDE.md 링크 갱신
5. **Cross-domain 참조**: WAS Connection Pool 변경 시 WAS harness 로드, PostgreSQL PgPool 변경 시 PostgreSQL harness 로드
6. **HITL-004 준수**: COLLSCAN 모니터링 관련 사항은 TA 승인 전 확정하지 않음
7. **서비스 분류 기준**: Read Preference / Write Concern 선택 시 서비스 성격(정산/결제 vs 조회성)을 명확히 구분하여 적용
