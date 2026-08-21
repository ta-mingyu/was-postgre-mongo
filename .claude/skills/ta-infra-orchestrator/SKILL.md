---
name: ta-infra-orchestrator
description: 전사 인프라 튜닝 TA 하네스의 오케스트레이터. Web 서버(Apache), WAS/JVM(Tomcat, Spring Boot, WebSphere Liberty), PostgreSQL, PgPool-II, MongoDB, Linux 커널 튜닝 관련 모든 작업의 진입점 — 표준값 산정, 팀별 Gap 분석, 표준 가이드 갱신·버전업, 표준값 시효성 검증, 신규 팀/서버 인프라 분석, 커널 파라미터 산정을 이 스킬로 시작한다. "가이드 다시 갱신", "MongoDB 부분만 다시", "이전 결과 개선", "하네스 점검/수정" 등 후속 요청에도 사용. 단순한 사실 질문(한 파라미터 뜻 등)은 직접 응답한다.
---

# TA 인프라 튜닝 오케스트레이터

## 실행 모드: 서브 에이전트 패턴 (Supervisor + 전문가 풀)

메인 세션(오케스트레이터)이 Agent 도구로 도메인 전문가를 작업별 호출한다. 에이전트 팀(상시 팀 채팅)을 쓰지 않는 이유: 이 프로젝트는 HITL 게이트가 핵심인 문서 프로젝트로, 진짜 조율자는 TA(사용자)다. 에이전트 간 자체 채팅은 HITL 게이트를 우회할 수 있어 위험하다. 도메인 간 협업이 필요한 값(70% Ceiling, 타임아웃 캐스케이드)은 cross-domain-verifier가 산출 후 일괄 검증한다.

**Agent 호출 규칙**: 반드시 `model: "opus"`를 명시한다. 에이전트 정의는 `.claude/agents/{name}.md`에 있으며, `subagent_type`으로 지정한다.

## 라우팅 매트릭스 (에이전트 x 스킬)

| 작업 | 에이전트 | 스킬 |
| :--- | :--- | :--- |
| Apache/Web 서버 튜닝 | `webserver-tuner` | `webserver-tuning` |
| WAS/JVM 튜닝 | `was-jvm-tuner` | `was-tuning` |
| PostgreSQL/PgPool 튜닝 | `postgresql-pgpool-tuner` | `postgresql-pgpool-tuning` |
| MongoDB 튜닝 | `mongodb-tuner` | `mongodb-tuning` |
| 커널/sysctl/ulimit | `linux-kernel-tuner` | `linux-kernel-tuning` |
| 팀별 준수도 평가 | 해당 도메인 에이전트 | `gap-analysis` |
| 가이드 버전업·피드백 반영 | 해당 도메인 에이전트 | `update-guide` (+직전 `verify-standards`) |
| 표준값 시효성 점검 | 메인이 직접 또는 도메인 에이전트 | `verify-standards` |
| 산출물 경계면 검증 | `cross-domain-verifier` | (에이전트 정의에 내장) |

Cross-domain 작업(HikariCP 풀 변경, 타임아웃 재설계)은 관련 도메인 에이전트를 **병렬 호출**한 뒤 반드시 cross-domain-verifier를 실행한다.

## Phase 0: 컨텍스트 확인

1. 활성 HITL 확인 — `harness/workflow.md`의 HITL 표. 요청 범위와 겹치면 Phase 2로.
2. 실행 모드 판별:
   - 요청이 기존 산출물의 일부 수정/보완 → **부분 재실행** (해당 도메인만)
   - 새로운 입력(신규 팀, 신규 서버 스펙) → **새 실행**
   - "하네스 점검/수정" 요청 → `.claude/agents/`, `.claude/skills/`, CLAUDE.md 변경 이력 감사 후 진화 절차
3. 프로젝트 상태 확인 — `harness/overview.md`(Phase 상태), `todo.md`(진행 중 작업).

## Phase 1: 요청 분류

요청을 라우팅 매트릭스의 작업 유형으로 분류한다. 분류가 애매하면 사용자에게 확인한다. 스펙(RAM/코어/인스턴스/버전/팀)이 주어지지 않았으면 이 단계에서 질의한다 — 에이전트가 임의 가정하지 않도록 입력을 완전히 만든다.

## Phase 2: HITL 게이트

- 요청 범위가 활성 HITL과 겹치는가 → 겹치면 **TA에게 질문을 올리고 응답 전까지 확정 금지**. 분석·제안은 가능.
- 구조적 트레이드오프(표준값 변경, 복제 구조 변경, 스코프 확장) → HITL로 분류.
- HITL 신규 등록 시 `harness/workflow.md` 표에 추가하고 상태를 기록한다.

## Phase 3: 전문가 실행

- 각 에이전트 호출 프롬프트에 포함할 것: 작업 유형, 대상 스펙/팀, 참조할 소스 파일 경로, 산출 형식(Gap 테이블/설정 블록/보고서 섹션), 해당 도메인 스킬 이름.
- 독립 도메인이면 `run_in_background: true`로 병렬 실행.
- 각 에이전트는 자기 도메인 검증 체크리스트를 1차 통과한 결과를 반환한다.
- 에이전트가 HITL 분류를 요청하면 Phase 2로 되돌린다.

## Phase 4: 경계면 검증

도메인 산출물이 있으면 `cross-domain-verifier`를 호출한다(model: opus). 검증 보고서의 CRITICAL가 0건일 때만 Phase 5로 진행. CRITICAL 잔존 시:
1. 해당 도메인 에이전트에 재작업 지시 (1회)
2. 재검증 후에도 잔존하면 TA에게 상충 내용을 보고하고 결정 요청 (HITL)

## Phase 5: 산출물 반영

가이드에 반영해야 하는 변경이면 `update-guide` 절차를 실행한다:
버저닝(`-v{N}`) → `reports/final/` 정본 동기화 → CLAUDE.md 링크 갱신 → `todo.md` 체크박스 갱신. 커밋 메시지는 기존 형식(`#add`, `#fix`, `#change`)을 따른다.

## 에러 핸들링

| 상황 | 처리 |
| :--- | :--- |
| 에이전트 1회 실패/타임아웃 | 1회 재시도. 재실패 시 해당 결과 없이 진행, 보고서에 누락 명시 |
| 스펙 누락 | 임의 가정 금지, 사용자 질의 |
| 도메인 문서 간 값 상충 | 삭제하지 않고 출처 병기, cross-domain-verifier가 상충으로 분류 |
| 외부 리서치 실패 | "미확인" 표기, 추측 금지 |
| HITL 미응답 | 해당 부분을 "보류"로 두고 나머지 완료 |

## 하네스 진화 (운영/유지보수)

- 실행 후 사용자에게 개선 여부를 1회 질의한다. 피드백 유형별 반영: 결과 품질 → 해당 스킬, 역할 → 에이전트 정의, 순서 → 본 오케스트레이터, 트리거 누락 → description.
- 모든 변경은 CLAUDE.md의 **변경 이력** 표에 기록한다.
- `.claude/agents/`, `.claude/skills/` 실제 파일과 CLAUDE.md 기록을 대조해 drift를 점검한다.

## 테스트 시나리오

**정상 흐름**: "주차서비스팀 WAS 표준값 산정해줘" → Phase 0(HITL 없음 확인) → Phase 1(was-jvm-tuner + was-tuning + gap-analysis 분류) → Phase 3(에이전트 실행, source/park-service-team.md 전달) → Phase 4(cross-domain-verifier가 maxThreads 500/maxPoolSize 100의 70% Ceiling 검증) → Phase 5(Gap 테이블 산출, 가이드 반영 여부는 TA 확인).

**에러 흐름**: "CL플랫폼팀 Old 영역 개선안 확정해줘" → Phase 0에서 HITL-003 활성 확인 → Phase 2에서 TA 승인 전 확정 불가 안내 → 분석·제안만 수행하고 "TA 응답 대기" 상태로 종료.
