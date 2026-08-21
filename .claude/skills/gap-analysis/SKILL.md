---
name: gap-analysis
description: 팀별 현재 인프라 설정 vs 표준값 Gap 분석 절차. source/ 팀 질문지 + harness/team-profiles.md(현재 설정 정규화) + 해당 도메인 rules 산정 공식으로 표준값을 계산하고, 현재/표준/권장 3열 Gap 테이블과 공통 불변량 위반 여부를 산출한다. 특정 팀(WAS/PostgreSQL/MongoDB/Web) 설정의 표준 준수도 평가, 개선안 도출, 신규 팀 인프라 분석, 변경 포인트 식별 시 반드시 이 스킬을 사용할 것. "다시 분석", "특정 팀만 다시" 등 후속 요청도 포함.
---

# 팀별 Gap 분석

4개 팀이 제출한 현재 인프라 설정(`source/`)을 표준값(도메인 rules 산정 공식)과 비교해 Gap과 개선안을 도출한다. 이 프로젝트 컨설팅의 핵심 반복 작업이다.

## 절차

### 1. 팀 현재 설정 수집
두 소스를 교차 확인(불일치 시 `source/`가 원본이므로 우선):
- `source/{팀}.md` — 팀 제출 원본(**읽기 전용**)
- `harness/team-profiles.md` — 정규화된 요약

### 2. 해당 도메인 스킬 로드 (의무)
팀 스택에 맞춰 도메인 스킬(산정 공식 + 검증 체크리스트)을 적용한다.

| 팀 스택 | 스킬 |
| :--- | :--- |
| WAS/Tomcat/Spring Boot/Liberty/JVM | `was-tuning` |
| PostgreSQL/PgPool-II | `postgresql-pgpool-tuning` |
| MongoDB | `mongodb-tuning` |
| Apache/Web 서버 | `webserver-tuning` |
| OS 커널/sysctl | `linux-kernel-tuning` |
| 다중 스택 | 사용 도메인 모두 + `harness/gotchas.md`(공통 불변량) |

### 3. 표준값 산출 (공식 적용)
팀 환경(호스트 RAM, CPU 코어, 인스턴스 수)을 대입해 계산. 주요 공식:
- WAS: `maxThreads = min(cores*50, 500)`, `Heap = floor(RAM*0.625)/N`, `maxPoolSize = 20/인스턴스`
- PostgreSQL: `shared_buffers = RAM*0.25`, `max_connections >= Sum(maxPoolSize)*1.5`
- MongoDB: `cacheSizeGB = 0.5*(RAM-1)`, PSS 표준

### 4. Gap 테이블 산출
현재값(source/) vs 표준값(rules) vs 권장 조치 3열 매트릭스:

| 항목 | 현재 | 표준 | Gap/권장 |
| :--- | :--- | :--- | :--- |
| maxPoolSize | 100 | 20 | 축소(70% Ceiling 위반) |
| maxLifetime | 30min | 27min | 단축(방화벽 30min 캐스케이드) |

### 5. 공통 불변량 검증 (harness/gotchas.md)
- `Sum(maxPoolSize) <= DB max_connections * 0.7`
- 타임아웃 캐스케이드 엄격 부등호
- WAS `maxThreads > 0`(무제한 -1 금지)
- 정산/결제 MongoDB: Read Preference `primary`, PSS 구성(PSA 금지)

### 6. HITL 분류
위반/권장 사항 중 TA 결정이 필요한 트레이드오프는 HITL로 분류(`harness/workflow.md`). 임의 확정 금지. 현재 활성: HITL-003(CL플랫폼 Old 영역), HITL-004(MongoDB COLLSCAN).

## 산출 위치

- Gap 테이블은 해당 도메인 Report(`reports/...`)의 "팀별 Gap" 섹션 또는 `harness/team-profiles.md`의 "표준 적용 시 주요 변경 포인트"에 반영.
- `harness/`에 Report 본문 작성 금지.

## 산출 규칙

- 한국어, 이모지 금지, mermaid 구조(`harness/conventions.md`). 수치는 단위 병기, 타임아웃은 엄격 부등호. `source/` 수정 금지.

## 완료 조건

- [ ] source/ 원본과 team-profiles 교차 확인
- [ ] 해당 도메인 스킬 산정 공식 적용
- [ ] Gap 테이블(현재/표준/권장) 산출
- [ ] 공통 불변량 위반 여부 점검
- [ ] TA 결정 필요 사항 HITL 분류
