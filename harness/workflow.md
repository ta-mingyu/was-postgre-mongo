# 작업 워크플로우

> 이 프로젝트에서 작업이 일어나는 방식과 Human-in-the-Loop 거버넌스.

## 도메인 harness 로드 의무

에이전트는 작업 대상 도메인에 따라 **반드시 해당 harness 규칙을 먼저 로드** 후 작업을 시작한다.

| 작업 대상 | 로드할 파일 |
| :--- | :--- |
| WAS(JVM/Thread/Pool/OS 커널) | `was-rules.md` |
| PostgreSQL / PgPool-II | `postgresql-rules.md` |
| MongoDB | `mongodb-rules.md` |
| Cross-domain(커넥션 풀, 타임아웃 캐스케이드) | 관련 도메인 rules **모두** + `gotchas.md`(불변량) |

이유: 도메인별 산정 공식과 검증 체크리스트가 분산되어 있고, 한 도메인 변경이 타 도메인 불변량(70% Ceiling, 방화벽 타임아웃)에 영향을 준다.

## 산출 기준선 유지

- 각 도메인 rules의 **산정 공식과 매트릭스를 기본선**으로 삼는다.
- 명확한 근거 없이 기본선을 변경하지 않는다.
- 변경 시 근거를 해당 rules 파일에 기록한다.

## Human-in-the-Loop (TA 에스컬레이션)

모호한 사안, 데이터 누락, 구조적 트레이드오프 발생 시 **TA에게 질문 후 진행**. 임의 결정 금지.

### 현재 활성 HITL 이슈

| ID | 팀 | 내용 | 상태 | 관련 도메인 |
| :--- | :--- | :--- | :--- | :--- |
| HITL-003 | CL플랫폼팀 | Old 영역 90.2% 사용률(CRITICAL). Parallel GC + Heap 2048m 고정. 대응 방안/GC 튜닝 이력 | TA 응답 대기 | WAS |
| HITL-004 | 플랫폼개발팀 | MongoDB COLLSCAN 모니터링 미수행. 운영 위험도 평가 | TA 응답 대기 | MongoDB |

### 종료된 HITL

| ID | 팀 | 결과 |
| :--- | :--- | :--- |
| HITL-001 | 주차서비스팀 | Closed(Scope-out) -- DB2는 전담 DBA 관리 영역 |
| HITL-002 | 현금정보계팀 | Closed(Scope-out) -- WAS/JVM 분석으로 한정 |

### HITL 가드 규칙

- HITL 이슈와 관련된 사항은 **TA 승인 전 확정하지 않는다**.
- HITL-003: CL플랫폼 Old 영역 관련 변경 금지(TA 응답 전).
- HITL-004: MongoDB COLLSCAN/프로파일링 관련 확정 금지(TA 응답 전).

## todo.md 갱신 의무

- 작업 완료 후 **반드시** `todo.md` 의 해당 체크박스를 `[x]`로 갱신한다.
- 신규 작업 발생 시 todo.md에 추가한다.

## 점진적 고도화

- 한 번에 전체 장표를 생성하지 않는다.
- 최신 벤더 권장값/튜닝 알고리즘 리서치를 병행하며 점진적 갱신.
- 리서치 결과는 `research/` 에 기록(출처 명시).

## 루프 사용 정책 (ralph vs ulw)

이 프로젝트는 document + Human-in-the-Loop 성격이므로 **자율 루프 사용을 지양**한다.

- 기본: 단일 작업 단위로 TA 확인(HITL 게이트)을 거치며 진행.
- ralph/code-loop: 부적합. 코드가 없고 TA 승인이 빈번해 루프가 HITL에서 반복 정지.
- ulw-research-doc-loop: **예외적 허용**. 순수 리서치 + 문서화(예: Phase 5 MongoDB 매트릭스 보완)처럼 TA 개입 없이 완결 가능한 경우만. `max-iterations` 낮게(3~5) 설정.
- 루프 진입 전 반드시: HITL 활성 이슈가 해당 작업 범위에 없는지 확인. 있으면 루프 금지.

## 버저닝 워크플로우

가이드 갱신 시 (conventions.md 버저닝 규칙 + commands.md 순서 준수):
1. 신규 버전 파일 생성 `reports/{name}-v{N}.md`
2. `reports/final/` 정본 사본 동기화
3. AGENTS.md 링크 최신화
4. todo.md 갱신
