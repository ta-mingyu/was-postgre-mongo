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
  - [x] `harness/ppt-outline.md` -- PPT 아웃라인 + 발표 스크립트 (후속 PPT 제거 시 함께 삭제)
  - [x] `harness/vendor-research.md` -- 벤더 리서치 + 표준값
- [x] 원본 `harness/ppt-structure-spec.md` 삭제
- [x] `harness/agent-context.md` 업데이트 (프로젝트 확장 반영) -- 이후 Phase 6 재구성 시 분산/삭제
- [x] `AGENTS.md` 업데이트 (신규 소스, 확장 로드맵, harness 분할 인덱스)

## Phase 6: 지시체계 init-project 규격 재구성 [Completed]

- [x] `AGENTS.md` 본문 -> `harness/` 이전, 목차(Index) 전용화
- [x] harness/ 표준 핵심 파일 6종 생성 (overview/structure/commands/conventions/workflow/gotchas)
- [x] `harness/team-profiles.md` 신규 (팀 메타데이터 정규화, agent-context §2 이관)
- [x] `harness/agent-context.md` 내용 분산(overview/team-profiles/workflow/gotchas) 후 삭제
- [x] 도메인 rules(was/postgresql/mongodb) + vendor-research + webserver-standard-guide 보존
- [x] 프로젝트 스킬 2종 생성: `.opencode/skills/{update-guide,gap-analysis}/SKILL.md`
- [x] 프로젝트 스킬 추가: `.opencode/skills/verify-standards/SKILL.md` (reports 변화 시 최신 자료 리서치 기반 표준값 시효성 검증)

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

## Phase 7: TA 기본 소양 학습 문서 작성 [Completed]

- [x] `study/` 폴더 신설 + README(커리큘럼 인덱스 + 학습 경로 + 불변량 지도)
- [x] 01 Linux 커널/ulimit (page cache/fd/overcommit/THP/TCP, 서버 역할별 차이)
- [x] 02 WAS/JVM (GC 분기 Parallel/G1/ZGC, Heap 분할, 스레드/풀 경제학)
- [x] 03 PostgreSQL 내부 (MVCC/WAL/double buffering/4종 타임아웃/플래너)
- [x] 04 PgPool-II (다중화 모델/로드밸런싱/Watchdog 합의/failover)
- [x] 05 MongoDB 8.0 (PSS/PSA 합의/WiredTiger/Write Concern/ESR)
- [x] 외부 리서치 기반 정확성 보정 (G1 공식 분기 6GB, swappiness=0 위험, tcp_tw_recycle 제거, Mongo 8.0 THP 전환)
- [x] AGENTS.md·structure.md study 섹션 추가

## Phase 8: Linux OS 심화 학습서 작성 [Completed]

- [x] 기존 `study/01-linux-kernel-and-ulimit.md` -> `study/_archive/` 이관(히스토리 보존)
- [x] `study/linux/` 하위 6장 심화 시리즈(OS 기초 -> 튜닝) + README 인덱스
- [x] linux/01 시스템 아키텍처/실행 모델 (부팅·systemd·시스템 콜·인터럽트·cgroup)
- [x] linux/02 프로세스/스케줄링 (fork+CoW·EEVDF 6.6+ 전환·시그널·System V 세마포어·cgroup v2)
- [x] linux/03 메모리 관리 (page cache·double buffering·OOM·overcommit·THP 전환·NUMA) -- 핵심 장
- [x] linux/04 파일시스템/I/O (VFS·fd 3계층 함정·ext4 저널링 3모드·fsync)
- [x] linux/05 네트워킹 스택 (TCP 상태머신·SYN/accept 큐·TIME_WAIT 포트 고갈·keepalive vs 방화벽 30min·NAPI)
- [x] linux/06 통합 튜닝 매트릭스/체크리스트 (역할별 차이·도메인 불변량 다리)
- [x] 외부 리서치 반영 (Oracle 커리큘럼 컨설팅 + librarian kernel.org 문서: EEVDF/mTHP/somaxconn 기본 4096/tcp_tw_recycle 4.12 제거)
- [x] AGENTS.md·study/README.md Linux 심화(프리퀄) 섹션 갱신

## Phase 9: reports/final 표준값 검수 + 가독성 개편 + V4 버저닝 [Completed]

- [x] 외부 리서치 기반 시효성 검증 (Oracle 가독성 진단 + librarian PostgreSQL/PgPool/MongoDB/WAS 최신값 검증)
- [x] P0 사고위험: MongoDB 8.0.4 미만 + Kernel 6.19 사용 금지 경고 추가 (mongodb.md, final-standard-guide.md)
- [x] 가독성 구조 통일: 4개 파일 0장(적용 전제) 신설, 검증(4)↔모니터링(5) 순서 교환, 3장 제목 "타임아웃 & 커넥션 캐스케이드" 통일, postgresql h4→h3, PG 매트릭스 분할, MongoDB limits 중복 제거, defaultMaxTimeMS cluster parameter 예시, pgpool 나이스M 사례 부록 이동, WAS 캐스케이드 3단계 확장
- [x] TA 결정 4건 반영: backend_weight 1:3(harness 올림) / maintenance_work_mem 상한 0.0625(PGTune 정합) / num_init_children 120 유지+모니터링 강화 / work_mem 공식 *3 통일+매트릭스 표준화(8/16/32/64MB)
- [x] P1 사실 오류 정정: Spring Boot "2.4 Breaking Change" -> 3.0 (was.md, final-standard-guide.md)
- [x] verify-standards: harness/vendor-research.md 에 변경 이력 3건 기록 (2026-07-02)
- [x] V4 버저닝: was-standard-guide-v4.md, db-standard-guide-v4.md 생성 + final-standard-guide.md 갱신 + AGENTS.md 링크 V3->V4
