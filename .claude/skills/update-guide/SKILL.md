---
name: update-guide
description: 표준 설정 가이드라인(Web/WAS/PostgreSQL/MongoDB/PgPool-II) 갱신 절차. 도메인 스킬 로드 -> 산정 공식 baseline 적용 -> 검증 체크리스트 실행 -> 버저닝(-v{N}) + reports/final/ 정본 동기화 + CLAUDE.md 링크 갱신 + todo.md 체크 + HITL 가드. 가이드라인 수정·버전업, 표준값 반영, 벤더 리서치 결과 반영, 컨설팅사/DBA 피드백 반영, "가이드 다시 갱신" 등 후속 요청 시 반드시 이 스킬을 사용할 것.
---

# update-guide — 표준 가이드라인 갱신 절차

이 프로젝트의 산출물(표준 설정 가이드라인)은 버저닝 규칙과 정본 동기화 규칙을 반드시 따른다. 순서를 어기면 `reports/final/`(정본)과 `reports/`(히스토리) 사본이 어긋나고 CLAUDE.md 링크가 깨진다.

## 트리거
- Web/WAS/PostgreSQL/MongoDB/PgPool-II 표준값 변경
- 벤더 권장값 리서치 결과 반영
- 컨설팅사/DBA 피드백(email/) 반영
- 기존 가이드라인 버전업

## 사전 가드 (HITL 확인)

갱신 전 활성 HITL 이슈가 갱신 범위와 겹치는지 확인(`harness/workflow.md`).
- HITL-003(CL플랫폼 Old 영역) 관련 변경 -> TA 승인 전 금지
- HITL-004(MongoDB COLLSCAN) 관련 변경 -> TA 승인 전 금지

겹치면 갱신 중단, TA 질의 후 재개.

## 절차

### 1. 도메인 스킬 로드 (의무)
작업 대상 도메인의 산정 공식 + 검증 체크리스트를 먼저 적용한다.

| 대상 | 스킬 |
| :--- | :--- |
| Web 서버(Apache) | `webserver-tuning` |
| WAS(JVM/Thread/Pool/OS) | `was-tuning` |
| PostgreSQL/PgPool-II | `postgresql-pgpool-tuning` |
| MongoDB | `mongodb-tuning` |
| OS 커널 | `linux-kernel-tuning` |
| Cross-domain | 관련 도메인 **모두** + `harness/gotchas.md`(공통 불변량) |

### 2. 산정 공식 baseline 적용
- 도메인 rules(`harness/{was,postgresql,mongodb}-rules.md`)의 공식/매트릭스를 **기본선**으로 삼는다.
- 명확한 근거 없이 기본선 변경 금지. 변경 시 근거를 해당 rules 파일에 기록.

### 3. 검증 체크리스트 실행
해당 도메인 rules의 "검증 체크리스트" 항목을 모두 충족하는지 확인. 특히 공통 불변량:
- `Sum(maxPoolSize) <= DB max_connections * 0.7` (70% Ceiling)
- 타임아웃 캐스케이드 엄격 부등호(`<`), 방화벽 30min 최상위
- `maxThreads > 0`, `Xms == Xmx`, Metaspace `Max >= Min`

### 4. 버저닝 (reports/)
- `reports/{기본파일명}-v{N}.md` 신규 버전 파일 생성.
- 기본 파일명 예: `was-standard-guide`, `db-standard-guide`. N = 기존 최대 버전 + 1.
- 서버별 정본(`reports/final/{web,was,postgresql,pgpool-ii,mongodb}.md`)은 파일명에 버전을 붙이지 않는다.

### 5. 정본 동기화 (reports/final/)
- `reports/final/{web,was,postgresql,pgpool-ii,mongodb}.md` 해당 정본을 신규 내용으로 갱신.
- `reports/final-standard-guide.md`(통합 확정본)도 해당 섹션 갱신.
- 정본은 항상 최신이어야 한다.

### 6. CLAUDE.md 링크 최신화
CLAUDE.md의 "버전 히스토리" 섹션 링크가 신규 버전을 가리키도록 갱신. 이전 버전 파일은 보존(삭제 금지).

### 7. todo.md 갱신
해당 작업 체크박스 `[x]` 처리. 신규 작업 발생 시 추가.

## 산출 규칙 (준수)
- 한국어, 이모지 금지, 구조는 mermaid. 상세는 `harness/conventions.md`.
- Report 본문은 항상 `reports/`. `harness/`에 본문 작성 금지.

## 완료 조건
- [ ] 검증 체크리스트 전 항목 충족
- [ ] 신규 버전 파일 생성
- [ ] final/ 정본 동기화
- [ ] CLAUDE.md 링크 최신화
- [ ] todo.md 체크
- [ ] HITL 가드 통과(또는 TA 승인 완료)
