---
name: verify-standards
description: reports/ 산출물(표준 설정 가이드라인)의 표준값·산정 공식이 최신 벤더 문서와 여전히 일치하는지 외부 리서치 기반으로 검증(시효성/freshness 체크). 공식 문서·벤더 소스 코드·JIRA·release notes로 크로스체크하여 구식/불일치 값을 탐지하고, vendor-research.md 갱신 + 도메인 rules 수정을 제안한다. 가이드 버전업 직전, 벤더 신버전 출시 후, 정기 시효성 점검, "이 표준값 아직 맞아?"라는 질문 시 반드시 이 스킬을 사용할 것.
---

# verify-standards — 표준값 시효성 검증 (외부 리서치)

`update-guide`가 **내부 정합성**(이미 정해진 baseline 대비 70% Ceiling·타임아웃 캐스케이드 점검)을 다룬다면, 이 스킬은 **외부 시효성**(기존 표준값이 최신 벤더 권장값과 일치하는가)을 검증한다. 인프라 권장값은 시간에 따라 변한다(MongoDB 8.0→8.3 Write Concern/defaultMaxTimeMS 변경, GC 기본값 전환, HikariCP 권장값 업데이트 등).

## 트리거
- `reports/` 산출물 갱신/버전업 직전 또는 병행
- 벤더 주요 버전 출시(PostgreSQL 마이너, MongoDB 8.x, Spring Boot 4.x, Tomcat 10.x)
- 정기 시효성 점검(예: 분기 1회)
- 컨설팅사/DBA 피드백(email/) 수령 후 권장값 재확인

## 핵심: 검증 대상 분류 (리서치 전 반드시 분류)

모든 값을 리서치하지 않는다. 두 부류로 나눈다.

### A. 환경 고정 불변량 (리서치 제외)
사내 인프라/정책에서 파생된 값. 벤더 문서와 무관하므로 **리서치하지 않는다**.

| 불변량 | 근거 | 위치 |
| :--- | :--- | :--- |
| 70% Ceiling Rule | 사내 공유 DB 정책 | `harness/gotchas.md`, 각 rules |
| 방화벽 TCP Established Timeout = 1,800s | 사내망 인프라 고정 | `harness/gotchas.md` |
| 타임아웃 캐스케이드(27/28/30min) | 방화벽 30min에서 파생 | `harness/gotchas.md` |
| maxPoolSize = 20/인스턴스 | 사내 표준 | 각 rules |

### B. 벤더 권장값 (리서치 대상)
벤더가 문서로 권장하는 값. **버전업 시 변경 가능성이 있어 반드시 최신 자료로 검증**.

| 항목 | 현재 기준 | 검증 소스 |
| :--- | :--- | :--- |
| GC 알고리즘 선택 기준(Parallel/G1/ZGC) | Heap 4096m 분기선 | OpenJDK/JEP, Tomcat |
| Heap 비율(Xmx = 컨테이너 RAM 50~70%) | 0.625/N | JVM 튜닝 가이드 |
| shared_buffers / effective_cache_size 비율 | RAM 0.25 / 0.75 | PostgreSQL 공식 wiki |
| WiredTiger cacheSizeGB 공식 | 0.5*(RAM-1) | MongoDB 공식 |
| random_page_cost (SSD=1.1) | SSD 1.1 | PostgreSQL 공식 |
| autovacuum 파라미터 기본/권장 | cost_limit 1000+ | PostgreSQL 공식 |
| HikariCP maxLifetime/minimumIdle | 27min / fixed-size | HikariCP Wiki |
| Liberty purgePolicy/reapTime | FailingConnectionOnly | Open Liberty 공식 |
| OS 커널 파라미터 권장값 | somaxconn, file-max | kernel.org 문서 |

## 절차

### 1. 변경 범위에서 검증 대상 값 추출
분류 A(불변량)는 제외하고 B(권장값)만 검증 큐에 넣는다.

### 2. 최신 자료 리서칭 (신뢰 계층 순)

| 계층 | 소스 | 용도 |
| :---: | :--- | :--- |
| 1 | 벤더 **공식 문서 + release notes + 공식 JIRA 티켓** | 권위적 기준값, 버전별 변경/Deprecated |
| 2 | **벤더 소스 코드**(공식 GitHub repo, `.idl`/`.h`/`.yaml` 파라미터 정의) — GitHub 리포 조회 도구(zread)나 raw 파일 fetch 활용 | 문서에 누락된 internal 파라미터·기본값 회수 (예: MongoDB `internalQuery*`) |
| 3 | 공식 문서 사이트의 웹 조회(WebFetch/WebSearch) | 현재 버전 문서 프로그래매틱 확인 |
| 4 | `harness/vendor-research.md` 기존 출처 | 보조 참조 |

> **금지 소스(인용 불가)**: 일반 블로그(Medium, Dev.to, 개인 블로그), Q&A 커뮤니티 답변 본문(StackOverflow 본문), 비공식 요약 사이트. 초기 탐색 힌트로만 활용하고, **최종 인용은 반드시 계층 1~3의 일차 출처로 교차 검증 후 표기**. (근거: 2026-07-27 MongoDB `internalQueryExecMaxBlockingSortBytes` 오기 사건 — 블로그 출처로 32MB가 전파되었으나 8.0 기본값은 100MB이고 파라미터명도 rename됨. 공식 JIRA + 소스 코드로 정정)

리서칭 시 반드시 확인:
- 현재 프로젝트 기준 버전(PostgreSQL, MongoDB 8.0/8.3, Spring Boot 3.5/4.0, Tomcat 9/10, Liberty 23.x)
- **release notes의 Breaking Change / Deprecated / 기본값 변경**
- 공식 권장값의 전제 조건(SSD vs HDD, DB 전용 vs 공유 서버 등)

### 3. 비교 테이블 작성

| 항목 | harness 현값 | 최신 권장값 | 출처 | 판정 |
| :--- | :--- | :--- | :--- | :--- |
| shared_buffers | RAM*0.25 | RAM*0.25 | PG wiki | 일치 |
| cacheSizeGB | 0.5*(RAM-1) | (조사값) | MongoDB docs | 일치/불일치 |

### 4. 판정별 처리
- **일치**: 조치 없음.
- **불일치/구식**: 해당 도메인 rules 수정 제안 + `harness/vendor-research.md` 갱신(출처·날짜 포함).
- **Deprecated/위험**: 즉시 보고 + HITL 분류(표준값 변경은 TA 결정 사항).

### 5. vendor-research.md 갱신 (불일치 발견 시)
- 결과(값, 출처, 확인일)를 `harness/vendor-research.md` 해당 벤더 섹션에 추가.
- 기존 값은 삭제하지 않고 "변경: 기존값 -> 신값 (근거)" 형태로 이력 보존.

### 6. HITL 가드
표준값 자체(불변량이 아닌 권장값) 변경은 TA 승인 후 반영. 리서치 결과는 **제안**으로만 제시하고 임의 확정 금지.

## update-guide와의 관계
- 순서: `verify-standards`(외부 시효성) -> `update-guide`(내부 정합성 + 버저닝 + 동기화).

## 완료 조건
- [ ] 검증 대상(벤더 권장값)과 제외(환경 불변량) 분류 완료
- [ ] 각 권장값 최신 자료(공식 문서 + release notes + 벤더 소스/JIRA) 조사
- [ ] 기존 vs 최신 비교 테이블 작성
- [ ] 불일치값 발견 시 vendor-research.md 갱신 + rules 수정 제안
- [ ] 표준값 변경 사항은 HITL 분류(TA 승인 전 확정 금지)
- [ ] 모든 인용 출처가 계층 1~3인지 확인 — 블로그/커뮤니티 글 인용 금지
