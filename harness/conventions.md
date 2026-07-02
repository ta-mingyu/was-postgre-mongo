# 산출 규칙

> 언어·문체·네이밍·버저닝 규칙. 기본값(Korean, Markdown)과 다른 점만 기록.

## 언어 및 표기

- 산출물은 **한국어**. 단, 시스템 설정 명칭/파라미터/소스코드 식별자/명령어/전문용어(build, migration, checkpoint 등)는 **원어 그대로**.
- **이모지 금지**. 구조(디렉토리 트리, 모듈 관계, 데이터 흐름, 토폴로지, 권위 계층)는 **mermaid 다이어그램**으로 표현. 단순 목록/표는 기호 그대로.
- 문장은 짧고 명확하게. 서술형 단락보다 **단문/개조식** 우선.

## 문서 구조(Diátaxis 분리)

| 성격 | 파일 | 역할 |
| :--- | :--- | :--- |
| Explanation(왜/맥락) | overview.md | 정체성, 목적, 스코프, 상태 |
| Reference(사실) | commands.md, conventions.md, 도메인 rules, team-profiles.md | 검증된 수치/공식/체크리스트 |
| Structure(무엇) | structure.md | 디렉토리와 산출물 지도 |
| Process(어떻게) | workflow.md, gotchas.md | 작업 방식, 함정 |

한 파일에 두 성격(왜 vs 사실)을 섞지 않는다.

## 파일명 규칙 (reports/)

- 간결 **영문 소문자 + 하이픈**. 숫자/버전은 기본 파일명에 넣지 않는다.
- 예: `was-standard-guide.md`, `db-standard-guide.md` (X `was-standard-guide-v3.md` 는 버저닝 시에만)

## 버저닝 규칙

- 내용 갱신 시 `{기본파일명}-v{Int}.md` 형식으로 **새 버전 파일 생성**.
- AGENTS.md 링크는 **항상 최신 버전**을 가리킨다.
- 이전 버전 파일은 히스토리 참조용으로 `reports/` 에 보존(삭제 금지).

## 디렉토리 역할 분리

| 디렉토리 | 허용 내용 | 금지 |
| :--- | :--- | :--- |
| `harness/` | 구동 규칙, 메타데이터, 산정 공식, 함정 | **Report 본문 작성 금지** |
| `reports/` | 표준 가이드라인 산출물 본문 | (본문은 여기만) |
| `research/` | 아키텍처 비교/리서치 (출처 명시 의무) | harness/ 에 리서치 본문 금지 |
| `source/` | (읽기 전용, 수정 금지) | 일체의 수정 금지 |

## 용어 (프로젝트 고정)

- **TA**(Technical Advisor): 의사결정권자. 모호/트레이드오프 이슈는 TA 승인 후 진행(HITL).
- **HITL**: Human-in-the-Loop. TA 응답 대기 이슈. workflow.md 참조.
- **70% Ceiling Rule**: `Sum(WAS maxPoolSize) <= DB max_connections * 0.7`. 도메인 공통 불변량.
- **방화벽 TCP Established Timeout = 1,800초(30분)**: 모든 타임아웃 산정의 최상위 기준.

## 수치 표기

- 타임아웃/수치는 **단위를 함께** 표기. 권장: 원단위 + 괄호 보조. 예: `1,620,000ms (27min)`.
- 타임아웃 캐스케이드는 **엄격 부등호(`<`)** 로 계층 격리. 등호(`<=`) 사용 금지.
