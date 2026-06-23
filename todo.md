# WAS/DB 표준 가이드라인 작성 로드맵 (Todo)

> 프로젝트 확장: Web Server 튜닝 가이드(Apache) 분석 -> WAS Server 표준 가이드라인 -> DB 설정 표준 가이드라인
> 본 파일은 전체 작업 항목과 진행 상태를 관리한다.
> **에이전트는 작업 완료 후 반드시 본 파일의 체크박스를 갱신해야 한다.**

---

## Phase 1: Web Server 튜닝 가이드 분석 [Completed]

- [x] `source/apache-tuning-guide.md` 분석
- [x] 핵심 원칙, 공식, 전략 분기 로직, 매트릭스 구조 추출
- [x] `harness/webserver-standard-guide.md` 분석 결과 기록
- [x] WAS 가이드 설계 청사(Blueprint) 작성

## Phase 2: Harness 파일 구조 개선 [Completed]

- [x] `harness/ppt-structure-spec.md` (480행) 파트별 분할
  - [x] `harness/ppt-outline.md` -- PPT 아웃라인 + 발표 스크립트
  - [x] `harness/vendor-research.md` -- 벤더 리서치 + 표준값
- [x] 원본 `harness/ppt-structure-spec.md` 삭제
- [x] `harness/agent-context.md` 업데이트 (프로젝트 확장 반영)
- [x] `AGENTS.md` 업데이트 (신규 소스, 확장 로드맵, harness 분할 인덱스)

## Phase 3: WAS Server 표준 가이드라인 작성 [In Progress]

- [x] 가이드라인 문서 구조 설계 (Apache 가이드 구조 차용)
- [x] Anti-Pattern 분석: 선형 비례 1/4 규칙의 실패 메커니즘 정리
- [x] Thread 설정: CPU 코어 수 기반 산정 (maxThreads = min(CPU_cores * 50, 500))
- [x] DB Connection Pool: HikariCP 공식 기반 산정 (인스턴스당 20)
- [x] 스케일 아웃 전략: 단일 호스트 RAM 분할 원칙 (4/8/16/32GB 매트릭스)
- [x] 공유 DB 환경 커넥션 풀 분할 원칙 (70% Ceiling Rule, Tier별 쿼터 할당)
- [x] JVM 메모리 설정: 인스턴스당 Heap 분할 매트릭스
- [x] WAS 종류별 설정 예시 (Tomcat / Spring Boot / Liberty)
- [x] Timeout 캐스케이드 (ProxyPass ttl < keepAliveTimeout, maxLifetime < wait_timeout)
- [x] 검증 체크리스트 작성
- [x] 기존 4개 팀 설정 vs 표준값 Gap 매핑
- [x] 컨설팅사(데이타뱅크) 리뷰 요청 및 피드백 수령
- [x] 컨설팅사 피드백 반영 답변 메일 발송 (`email/re-was-guide-review.md`)
- [x] 컨설팅사 최종 답변 수령 후 가이드라인 최종 확정 (코어 배정 비율, 기준선 프로세스, 현금정보계 105 커넥션)
  - [x] 1차 답변 수령: 1/4 공격적/1/10 안정적 검증, 코어 정렬 사례별 상이, 105 커넥션 안정 범위 확인
  - [x] 2차 질의 발송: G1GC 임계점(Heap 6GB), ParallelGC vs G1HC 전환 기준, Region 크기 조정 방안
  - [ ] 2차 답변 수령 후 GC 권장 섹션 최종 확정

## Phase 4: DB 설정 표준 가이드라인 작성 [Completed]

- [x] DB 가이드 뼈대 설계 (WAS 가이드 구조 차용)
- [x] Anti-Pattern 분석 (과대 shared_buffers, COLLSCAN, 커넥션 과부하)
- [x] PostgreSQL 표준 튜닝 가이드라인
  - [x] shared_buffers, work_mem, effective_cache_size, maintenance_work_mem 산출 원칙 및 RAM별 매트릭스
  - [x] WAL 설정 (wal_buffers, checkpoint_completion_target, max_wal_size)
  - [x] autovacuum 설정
  - [x] 커넥션 설정 (max_connections, superuser_reserved)
  - [x] RAM/코어별 매트릭스 테이블
- [x] MongoDB 표준 튜닝 가이드라인
  - [x] WiredTiger 캐시 설정 (cacheSizeGB = RAM * 25%)
  - [x] Replica Set 설정 (writeConcern, readPreference)
  - [x] 인덱스 전략 (ESR 규칙, COLLSCAN 방지)
  - [x] MongoDB 8.0/8.3 전면 갱신 (defaultMaxTimeMS, cacheSizePct, TCMalloc, Write Concern 변경, Deprecated 기능 안내)
- [x] PgPool-II 설정 표준
  - [x] num_init_children, max_pool, child_life_time
  - [x] WAS -> PgPool -> PostgreSQL 타임아웃 캐스케이드
  - [x] 로드 밸런싱 설정 (load_balance_mode, backend_weight)
- [x] WAS-DB 간 타임아웃 캐스케이드 표준 (WAS maxLifetime -> PgPool -> DB wait_timeout)
  - [x] 기본 캐스케이드 구조 설계
- [x] DB 모니터링 최소 체계 (Slow Query, COLLSCAN, Lock Wait, Active Sessions)
  - [x] 7개 항목 모니터링 매트릭스 작성
- [x] 기존 팀 설정 vs 표준값 Gap 매핑 (플랫폼개발팀 PostgreSQL + MongoDB)
- [x] 검증 체크리스트 작성 (8개 항목)

## Phase 5: MongoDB RAM/코어별 매트릭스 테이블 상세 수치 보완 [Pending]

- [ ] MongoDB RAM/코어별 매트릭스 테이블 상세 수치 보완
