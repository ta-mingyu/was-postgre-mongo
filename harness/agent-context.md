# Agent Context & Token Management

> 본 파일은 에이전트 구동 시 실시간으로 갱신되는 컨텍스트 허브이다.
> 세션 간 연속성을 보장하기 위해 Phase 진행 상황, 팀별 메타데이터, 분석 인사이트를 기록한다.

---

## 1. 에이전트 상태 스냅샷

| 항목 | 값 |
| :--- | :--- |
| **현재 Phase** | Phase 3 - PPT 슬라이드 아웃라인 및 스크립트 빌드 |
| **Phase 2 상태** | Completed (리서치 및 Gap 분석 완료) |
| **Phase 3 상태** | Discussion-Driven Structure로 슬라이드 덱 설계 중 |
| **마지막 갱신** | 2026-05-27 (Phase 3 착수, PPT 패러다임 전환) |
| **활성 HITL 이슈** | 2건 (HITL-003, HITL-004) |
| **종료 HITL 이슈** | 2건 (HITL-001, HITL-002 - Scope-out) |

### 프로젝트 스코프 (TA 확정)

본 컨설팅 표준화 작업은 다음 영역에 **집중**한다:

- WAS 환경 (Tomcat, WebSphere Liberty, CLS WAS)
- Java 런타임 버전 파편화 해소
- JVM (Heap/Metaspace) 메모리 전략
- GC 알고리즘 표준화
- WAS 레벨의 Connection Pool 설정
- 플랫폼개발팀의 MongoDB 구조

**제외 영역:**

- DB2 데이터베이스 내부 파라미터 (별도 전담 DBA 관리)
- 현금정보계팀 DB 시스템 정보 (WAS/JVM 분석으로 한정)

---

## 2. 팀별 메타데이터 요약

### 2.1 플랫폼개발팀 (Platform Develop Team)

| 항목 | 나이스파크 (Nice Park) | 나이스차저 (Nice Charger) |
| :--- | :--- | :--- |
| **WAS** | Spring Boot 3.5.3 내장 Tomcat | Tomcat 9 / Spring Boot 4.0.5 내장 Tomcat |
| **Java** | 17 | 15, 25 (혼합) |
| **GC** | G1 GC | G1 GC |
| **Heap** | 70% (heapPercent) | Xms/Xmx 2048m 고정 |
| **Metaspace** | 128m / Max 128m | 512m / Max 512m |
| **Thread** | 기본값 | 기본값 |
| **Conn Pool** | HikariCP max 5 / min-idle 2 | HikariCP max 100(web)/20(admin) / min-idle 5 |
| **maxLifetime** | 1,800,000 ms (30분) | 2,000,000 ms (~33분) |
| **DB - PostgreSQL** | PgPool-II 경유, Read:Write = 7:3, RTO 10s / RPO 5s | PgPool-II 경유 |
| **DB - MongoDB** | Replica Set (Master 1 / Slave 2), Read:Write = 6:4, COLLSCAN 모니터링 미수행 | 해당없음 |
| **특이사항** | 파티셔닝 미적용, Slow Query 없음 | Java 버전 혼합 운영 |

### 2.2 CL플랫폼팀 (CLS Team)

| 항목 | 값 |
| :--- | :--- |
| **WAS** | CLS 전용 WAS |
| **Java** | 15.0.2 |
| **GC** | Parallel GC |
| **Heap** | Xms/Xmx 2048m 고정 |
| **Metaspace** | Used 157,772K / Committed 166,528K |
| **Thread** | maxThreads -1 (무제한), 나머지 기본값 |
| **Conn Pool** | maxPoolSize 50 / minIdle 0 / maxLifetime 1800 (단위 불명, 초 추정) |
| **Timeout** | connectionTimeout 30s / reapTime 300s / maxIdleTime 1800s |
| **DB** | PostgreSQL/MongoDB 미사용 |
| **위험 신호** | Old 영역 90.2% 사용률 (1,418,698K / 1,572,864K) - CRITICAL |
| **Young 영역** | 정상 (Eden 6% / From 7% / To 0%) |

### 2.3 주차서비스팀 (Park Service Team)

| 항목 | 값 |
| :--- | :--- |
| **WAS** | Apache Tomcat 9.0.70 |
| **Java** | OpenJDK 15.0.2 (64-Bit Server VM) |
| **GC** | G1 GC |
| **Heap** | Xms 2048m / Xmx 4096m (가변) |
| **Thread** | maxThreads 500 / minSpareThreads 25 / acceptCount 기본값(100) |
| **Conn Pool** | maxTotal 100 / minIdle 100 / maxWaitMillis 10,000ms |
| **Timeout** | connectionTimeout 20,000ms (20초) |
| **DB** | DB2 (본 컨설팅 범위 외 - 전담 DBA 관리) |
| **분석 범위** | WAS/JVM 설정에 한정 |

### 2.4 현금정보계팀 (Info Service Team)

| 항목 | 값 |
| :--- | :--- |
| **WAS** | IBM WebSphere Liberty v23.0.0.2 ND |
| **Java** | OpenJDK 15.0.2+7-27 |
| **GC** | Parallel GC (`-XX:+UseParallelGC`) |
| **Thread** | Liberty 동적 확장 (coreThreads -1 / maxThread -1) |
| **Conn Pool** | maxPoolSize 50 / minPoolSize 0 |
| **Timeout** | connectionTimeout 30s / maxIdleTime 1800s / reapTime 300s |
| **컨테이너 수** | 7개 (고정 3 + 동적 4) |
| **DB** | 본 컨설팅 범위 외 (WAS/JVM 분석에 한정) |
| **특이사항** | 고정/동적 메모리 이원 운영. NIBS 컨테이너 Heap 8GB가 최대 |

---

## 3. HITL 이슈 관리

### HITL-001: 주차서비스팀 - DB2 타임아웃 설정 확인

| 항목 | 내용 |
| :--- | :--- |
| **상태** | **Closed (Scope-out: 별도 IBM DB2 DBA 관리 영역)** |
| **폐지 사유** | TA 결정: DB2 영역은 전담 DBA가 관리하므로 본 컨설팅 대상에서 제외 |

### HITL-002: 현금정보계팀 - DB 시스템 정보 누락

| 항목 | 내용 |
| :--- | :--- |
| **상태** | **Closed (Scope-out: DB 자체는 별도 관리 영역)** |
| **폐지 사유** | TA 결정: 현금정보계팀 분석 범위를 WAS/JVM/Connection Pool로 한정 |

### HITL-003: CL플랫폼팀 - Old 영역 90% 임계치 대응 (CRITICAL)

| 항목 | 내용 |
| :--- | :--- |
| **상태** | Open (TA 응답 대기) |
| **배경** | ParOldGen 90.2% (1,418,698K / 1,572,864K). Parallel GC + Heap 2048m 고정 |

### HITL-004: 플랫폼개발팀 - MongoDB COLLSCAN 모니터링 미수행

| 항목 | 내용 |
| :--- | :--- |
| **상태** | Open (TA 응답 대기) |
| **배경** | MongoDB Replica Set 운영 중 COLLSCAN 체크 미수행 |
