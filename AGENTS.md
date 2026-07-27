# Infrastructure Standardization Consulting — 지식 베이스

본문 내용은 `harness/` 폴더에 있다. 이 파일은 **목차(Index)** 역할만 수행한다.
에이전트는 작업 전 `harness/overview.md` 와 대상 도메인 rules를 먼저 로드한다.

## 프로젝트 유형

**document** -- 전사 Web/WAS 및 RDBMS/NoSQL 설정 표준화 컨설팅 산출물 저장소. 애플리케이션 코드 없음.

## harness/ 목차

### 핵심 파일

| 파일 | 역할 | 언제 읽을지 |
| :--- | :--- | :--- |
| [overview.md](./harness/overview.md) | 정체성, 목적, 스코프, 현재 Phase 상태 | 프로젝트 시작 시 |
| [structure.md](./harness/structure.md) | 디렉토리 트리, 산출물 권위 계층(정본 vs 히스토리) | 탐색/수정 전 |
| [commands.md](./harness/commands.md) | 가이드 갱신 절차, 리서치 추가, Gap 분석 진입점 | 산출물 작업 시 |
| [conventions.md](./harness/conventions.md) | 한국어/이모지금지/mermaid, 파일명·버저닝, 용어 | 산출 전 |
| [workflow.md](./harness/workflow.md) | 도메인 harness 로드 의무, HITL/TA 거버넌스, 루프 정책, todo 갱신 | 작업 착수 전 |
| [gotchas.md](./harness/gotchas.md) | 함정 + `## 절대 금지`(source 읽기전용, 방화벽 30min, 70% Ceiling, PSA 금지 등) | 산출/변경 전 필수 |

### 참조 데이터

| 파일 | 역할 | 언제 읽을지 |
| :--- | :--- | :--- |
| [team-profiles.md](./harness/team-profiles.md) | 4개 팀 현재 인프라 설정 정규화 요약(Gap 분석 기준) | 팀 분석 시 |

### 도메인 산정 규칙 (작업 대상에 따라 필수 로드)

| 파일 | 역할 | 적용 도메인 |
| :--- | :--- | :--- |
| [was-rules.md](./harness/was-rules.md) | WAS 산정 공식(Thread/Heap/GC/Pool) + 타임아웃 캐스케이드 + 검증 체크리스트 | WAS/JVM |
| [postgresql-rules.md](./harness/postgresql-rules.md) | PostgreSQL + PgPool-II 산정 공식 + 70% Ceiling 산출 + 검증 체크리스트 | PostgreSQL/PgPool |
| [mongodb-rules.md](./harness/mongodb-rules.md) | MongoDB 산정 공식(WiredTiger/Replica Set/ESR) + Write Concern + 검증 체크리스트 | MongoDB |
| [vendor-research.md](./harness/vendor-research.md) | 벤더별 권장 튜닝값 리서치 로그 + 전사 표준값 도출 근거 | 표준값 근거 확인 시 |
| [webserver-standard-guide.md](./harness/webserver-standard-guide.md) | Apache 튜닝 가이드 분석 청사(WAS/DB 가이드 설계 기준) | 가이드 구조 설계 시 |

## 통합 규정 (최종 산출물 정본)

- [인프라 표준 설정 규정(확정본)](./reports/final-standard-guide.md) -- WAS + DB 통합 배포본(IT기획실 -> 전 사업팀)

### 서버별 실무 배포 가이드 (정본, 운영자용)

- [Web 서버 설정 가이드](./reports/final/web.md) -- Apache HTTP Server 2.4 + Event/Worker/Prefork MPM + 역방향 프록시 + 혼합 용도 주의사항
- [WAS 서버 설정 가이드](./reports/final/was.md) -- OS 커널 + Tomcat/Spring Boot/WebSphere + HikariCP + 체크리스트
- [PostgreSQL 서버 설정 가이드](./reports/final/postgresql.md) -- OS 커널 + PgPool+SR/Standalone + 타임아웃 Guardrails
- [PgPool-II 서버 설정 가이드](./reports/final/pgpool-ii.md) -- 커넥션 풀링/로드밸런싱/페일오버 + 체크리스트
- [MongoDB 서버 설정 가이드](./reports/final/mongodb.md) -- OS 커널 + Replica Set PSS/Standalone + Write Concern

## 버전 히스토리 (참조용)

> 권위 계층: `reports/final/`(정본) > 최신 `-v{N}`(개별 최신: WAS v5 / DB v4) > 이전 버전. 상세는 structure.md.

- [WAS Standard Guide V5](./reports/was-standard-guide-v5.md) | [V4](./reports/was-standard-guide-v4.md) | [V3](./reports/was-standard-guide-v3.md) | [V2](./reports/was-standard-guide-v2.md) | [V1](./reports/was-standard-guide.md)
- [DB Standard Guide V4](./reports/db-standard-guide-v4.md) | [V3](./reports/db-standard-guide-v3.md) | [V2](./reports/db-standard-guide-v2.md) | [V1](./reports/db-standard-guide.md)
## 학습 문서 (study/)

- [TA 학습 커리큘럼](./study/README.md) -- 커리큘럼 인덱스 + 학습 경로 + 도메인 간 불변량 지도
- [Linux OS 심화(프리퀄)](./study/linux/README.md) -- OS 기초부터 튜닝까지. 부팅/프로세스/메모리/FS/네트워크/통합 튜닝 6장
  - [01 시스템 아키텍처/실행 모델](./study/linux/01-architecture-and-execution.md) -- 부팅·systemd·시스템 콜·인터럽트·cgroup
  - [02 프로세스/스케줄링](./study/linux/02-process-and-scheduling.md) -- fork+CoW·EEVDF(6.6+)·시그널·세마포어·cgroup v2
  - [03 메모리 관리(핵심)](./study/linux/03-memory-management.md) -- page cache·OOM·overcommit·THP·NUMA
  - [04 파일시스템/I/O](./study/linux/04-filesystem-and-io.md) -- VFS·fd 3계층·ext4 저널링·fsync
  - [05 네트워킹 스택](./study/linux/05-networking-stack.md) -- TCP 상태머신·TIME_WAIT·keepalive·방화벽 30min
  - [06 통합 튜닝/체크리스트](./study/linux/06-tuning-matrix-and-checklist.md) -- 역할별 매트릭스·도메인 불변량 다리
- [02 WAS/JVM](./study/02-was-jvm-and-thread-pool.md) -- GC 분기(Parallel/G1/ZGC), Heap 분할, 스레드/풀 경제학
- [03 PostgreSQL 내부](./study/03-postgresql-internals.md) -- MVCC/WAL/double buffering/4종 타임아웃/플래너
- [04 PgPool-II](./study/04-pgpool-ii.md) -- 다중화 모델/로드밸런싱/Watchdog 합의/failover
- [05 MongoDB 8.0](./study/05-mongodb-wiredtiger.md) -- PSS/PSA 합의/WiredTiger/Write Concern/ESR

> study/는 TA 기본 소양 학습 자료(Explanation). 설정값 정본은 항상 `reports/final/`.

## 리서치 (research/)

- [PostgreSQL 아키텍처 비교](./research/postgresql/architecture-comparison.md) -- Standalone/SR/PgPool+SR/Patroni/repmgr
- [MongoDB 아키텍처 비교](./research/mongodb/architecture-comparison.md) -- Standalone/Replica Set/Sharded Cluster
- [WAS-DB 연동 연구](./research/was/was-db-integration.md) -- 아키텍처별 타임아웃/커넥션 풀 산정 기준

## 원천 소스 (source/, 읽기 전용 -- 수정 금지)

| 소스 | 대상 팀 | 주요 특이 |
| :--- | :--- | :--- |
| [platform-develop-team.md](./source/platform-develop-team.md) | 플랫폼개발팀 | Tomcat(Spring Boot 내장), PostgreSQL(PgPool 경유), MongoDB(Replica Set), RTO 10s/RPO 5s |
| [cl-platform-team.md](./source/cl-platform-team.md) | CL플랫폼팀 | CLS 전용 WAS, Old 영역 90% 임계(HITL-003), Parallel GC |
| [park-service-team.md](./source/park-service-team.md) | 주차서비스팀 | Tomcat 9.0.70, DB2(범위 외), G1 GC |
| [info-service-team.md](./source/info-service-team.md) | 현금정보계팀 | WebSphere Liberty 23.0.0.2 ND, 7 컨테이너 고정/동적 이원 |
| [apache-tuning-guide.md](./source/apache-tuning-guide.md) | -- | 사내 Web Server 튜닝 가이드 V3.1(WAS/DB 가이드 설계 기준) |

## 커뮤니케이션 (email/)

- [컨설팅사 리뷰 답변 1차](./email/re-was-guide-review.md)
- [컨설팅사 답변 2차](./email/re-was-guide-review-2.md)
- [DBA 검수 의견 반영 + 통합 규정 최종 검수 요청](./email/re-was-guide-review-3.md) -- 데이타뱅크 조도형 차장 PostgreSQL 6건 피드백 반영

## 작업 관리

- [Todo Roadmap](./todo.md) -- 전체 Phase 로드맵. **작업 완료 후 반드시 체크박스 `[x]` 갱신**

## 빠른 참조

- 도메인 산정 공식/체크리스트: `harness/{was,postgresql,mongodb}-rules.md`
- 도메인 공통 불변량(70% Ceiling, 방화벽 30min, 타임아웃 캐스케이드): `harness/gotchas.md`
- 반복 워크플로우: `.opencode/skills/update-guide`(가이드 갱신), `.opencode/skills/gap-analysis`(팀 Gap 분석), `.opencode/skills/verify-standards`(표준값 시효성 외부 리서치 검증)
- 산출 규칙·버저닝: `harness/conventions.md`
- 작업 절차·HITL: `harness/workflow.md`
