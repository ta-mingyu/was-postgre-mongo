# 운용 명령 (문서 프로젝트)

> 본 프로젝트는 애플리케이션이 없다. 빌드/테스트/lint 명령이 존재하지 않는다.
> 대신 아래 산출물 갱신 절차를 명령처럼 취급한다.

## 표준 가이드라인 갱신 (핵심 절차)

갱신 순서가 중요하다. 순서를 어기면 정본과 히스토리 사본이 어긋난다.

```bash
# 1. 해당 도메인 harness 규칙 로드 (산정 공식 + 검증 체크리스트)
#    - WAS 작업 -> harness/was-rules.md
#    - PostgreSQL/PgPool 작업 -> harness/postgresql-rules.md
#    - MongoDB 작업 -> harness/mongodb-rules.md
#    - Cross-domain(HikariCP 등) -> 관련 도메인 모두 로드

# 2. reports/ 에 신규 버전 파일 생성 (버저닝 규칙은 conventions.md)
#    {기본파일명}-v{N}.md

# 3. reports/final/ 의 정본 사본 동기화 (final/ 가 항상 최신)

# 4. AGENTS.md 링크를 신규 버전으로 갱신

# 5. todo.md 체크박스 [x] 갱신
```

상세(도메인 로드 순서, 검증 체크리스트, HITL 가드)는 `.opencode/skills/update-guide/SKILL.md` 스킬 참조.

## 리서치 추가

```bash
# 1. research/{postgresql,mongodb,was}/ 하위에 파일 생성
#    파일명: 간결 영문 소문자 + 하이픈
# 2. 문서 말미에 출처(Source) 명시 (의무)
# 3. AGENTS.md 리서치 섹션에 링크 추가
```

> 리서치 본문은 research/ 에만 둔다. harness/ 에 리서치 본문 작성 금지.

## Gap 분석 (팀별)

팀 현재 설정 vs 표준값 비교 절차는 `.opencode/skills/gap-analysis/SKILL.md` 스킬 참조.
요약: `source/{팀}.md` + `harness/team-profiles.md` + 해당 도메인 rules -> Gap 테이블 산출.

## 사전 확인 (작업 전)

- `git log --oneline -10` 으로 직전 커밋 메시지 형식(`#add`, `#fix`) 확인 후 동일 스타일 유지.
- `source/` 하위 파일은 읽기 전용. 절대 수정하지 않는다.
