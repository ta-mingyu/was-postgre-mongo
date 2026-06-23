# WAS 서버 아키텍처 리서치

> 리서치 일자: 2026-06-10
> 목적: db-standard-guide-v2.md 작성 시 참조용 WAS 아키텍처 컨텍스트.
> WAS 자체는 was-standard-guide.md에서 이미 최종본 완성. 본 문서는 DB 가이드 작성 시 필요한 WAS-DB 연동 관점의 참조 자료.

---

## 1. WAS 아키텍처 구성 방식 (현재 운영 현황)

| 구성 방식 | 대상 팀 | 특징 | DB 연동 방식 |
|---|---|---|---|
| **Spring Boot 내장 Tomcat** (단일 JAR) | 플랫폼개발팀 (Nice Park, Nice Charger) | 컨테이너 친화적, 무상태 | HikariCP -> PgPool-II -> PostgreSQL, MongoDB Driver |
| **독립 Apache Tomcat 9.x** | 주차서비스팀 | 전통적 WAR 배포 | DBCP -> DB2 |
| **IBM WebSphere Liberty 23.x** | 현금정보계팀 | 엔터프라이즈 컨테이너, 다중 인스턴스 | Liberty ConnectionManager -> DB |
| **CLS 전용 WAS** | CL플랫폼팀 | 커스텀 WAS, 상세 스펙 불명 | HikariCP -> DB |

---

## 2. WAS-DB 연동 아키텍처 분류

DB 가이드 작성 시 WAS가 DB에 어떻게 연결되는지에 따라 DB 설정이 달라짐.

### 2.1 직접 연결 (WAS -> DB)

```
WAS (HikariCP / Liberty ConnMgr)
      |
      +-- maxPoolSize 커넥션
      |
      v
DB (PostgreSQL / MongoDB)
```

- WAS 내부 커넥션 풀만 존재
- 적합: 소규모, Standalone/SR Only 환경
- DB max_connections = Sum(WAS maxPoolSize) * 1.5

### 2.2 커넥션 풀러 경유 (WAS -> PgPool-II -> PostgreSQL)

```
WAS (HikariCP)
      |
      +-- maxPoolSize 커넥션
      |
      v
PgPool-II (num_init_children)
      |
      +-- PgPool -> DB 백엔드 커넥션
      |
      v
PostgreSQL (max_connections)
```

- WAS 풀 + PgPool 풀의 이중 풀 구조
- 적합: 다수 WAS 인스턴스가 DB 공유, 읽기 분산 필요
- DB max_connections >= PgPool num_init_children * 1.5
- PgPool num_init_children >= Sum(WAS maxPoolSize) + 여유

### 2.3 샤드 라우터 경유 (WAS -> mongos -> MongoDB Sharded Cluster)

```
WAS (MongoDB Driver)
      |
      v
mongos (쿼리 라우터)
      |
      +-- Shard 1 (RS)
      +-- Shard 2 (RS)
      +-- ...
      v
Config Server (메타데이터)
```

- WAS는 mongos 연결 문자열만 지정
- mongos가 Shard Key 기반으로 자동 라우팅
- 적합: MongoDB Sharded Cluster 환경
- WAS 풀 설정은 mongos 연결 기준

---

## 3. WAS-DB 타임아웃 캐스케이드 (아키텍처별)

### 3.1 WAS -> PgPool-II -> PostgreSQL

```
HikariCP maxLifetime (29min)
    |
    v
PgPool-II child_life_time (30min)
    |
    v
PostgreSQL idle_session_timeout (35min)
    |
    v
방화벽 TCP Established Timeout (30min / 1,800s)
```

### 3.2 WAS -> PostgreSQL (직접)

```
HikariCP maxLifetime (29min)
    |
    v
PostgreSQL idle_session_timeout (30min)
    |
    v
방화벽 TCP Established Timeout (30min / 1,800s)
```

### 3.3 WAS -> MongoDB (Replica Set / Sharded Cluster 공통)

```
HikariCP / MongoDB Driver maxLifetime (29min)
    |
    v
MongoDB connectionPool maxIdleTimeMS (30min)
    |
    v
방화벽 TCP Established Timeout (30min / 1,800s)
```

---

## 4. 커넥션 풀 산정 기준 (아키텍처별)

| WAS-DB 연동 | 풀 계층 | 산정 기준 |
|---|---|---|
| 직접 연결 | WAS 풀만 | maxPoolSize = 20 / 인스턴스. DB max_conn >= Sum * 1.5 |
| PgPool 경유 | WAS 풀 + PgPool 풀 | WAS maxPoolSize = 20. PgPool num_init_children >= Sum(WAS 풀) + 여유. DB max_conn >= PgPool * 1.5 |
| mongos 경유 | WAS 풀 (mongos 대상) | WAS maxPoolSize = 20~50. mongos가 내부적으로 샤드 분산 |
| Liberty ConnMgr | Liberty 풀 | maxPoolSize = 15 / 컨테이너. DB max_conn >= Sum(컨테이너 * maxPoolSize) * 1.5 |

---

## 5. WAS 가이드 확장 포인트 (DB 아키텍처 다변화에 따른)

현재 was-standard-guide.md는 PgPool-II 경유를 기본으로 작성됨.
DB 가이드 v2에서 아키텍처가 다변화되면, WAS 가이드도 아래 포인트를 보완 필요:

1. **타임아웃 캐스케이드 분기**: 직접 연결 vs PgPool 경유 vs mongos 경유
2. **커넥션 풀 대상 분기**: DB 직접 vs PgPool vs mongos
3. **HikariCP vs PgBouncer**: 대규모 환경에서 PgBouncer 도입 시 WAS 풀 설정 변경사항
