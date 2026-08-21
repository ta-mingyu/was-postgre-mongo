---
name: mongodb-tuning
description: MongoDB 표준값 산정·튜닝 절차. WiredTiger cacheSizeGB 산정, Replica Set PSS 구성(PSA 금지, 정산/결제 절대 금지), Write Concern(w:majority)/Read Preference(primary), COLLSCAN Zero-Tolerance 프로파일링, ESR 인덱스 규칙, 타임아웃(maxIdleTimeMS), MongoDB 8.0 LTS/8.3 변경사항을 다룬다. MongoDB 표준값 산정, Replica Set 구성 검토, 인덱스/슬로우 쿼리 전략, Mongo 관련 가이드 갱신이나 Gap 분석을 할 때 반드시 이 스킬을 사용할 것. PostgreSQL/PgPool은 postgresql-pgpool-tuning을 사용.
---

# MongoDB 튜닝

## 다루는 것

MongoDB Replica Set(플랫폼개발팀, Master 1/Slave 2, R:W=6:4)의 표준값 산정. 정본은 `harness/mongodb-rules.md`(222행). 버전 기준: MongoDB 8.0 LTS(온프레미스 장기지원)/8.3 Latest.

## 핵심 공식 (요약 — 정본은 mongodb-rules.md §2~§5)

```
WiredTiger cacheSizeGB = 0.5 * (RAM - 1)   (DB 전용 서버 기준)
  공유 환경(WAS/DB 혼합): RAM의 25% 수준으로 명시적 제한
Replica Set 표준: PSS(Primary 1 + Secondary 2) 최소 3노드
  정산/결제: PSA 절대 금지(Secondary 1대 다운 -> w:majority 쓰기 영구 정지)
  정산/결제: Write Concern w:majority + Read Preference primary 고정
타임아웃: WAS maxLifetime(27min) < connectionPool maxIdleTimeMS(30min) (엄격 부등호)
COLLSCAN Zero-Tolerance: 발견 즉시 인덱스 추가, 프로파일링 미설정 허용 불가
인덱스: ESR 규칙(Equality -> Sort -> Range)
```

## 워크플로우

1. **스펙 수집** — RAM, 전용/공유, Replica Set 토폴로지, 서비스 도메인(정산/결제 여부), 워크로드(R:W), 버전.
2. **rules 로드** — `harness/mongodb-rules.md` 전체. 특히 §2(공식), §3(Write Concern/Read Preference), §4(Profiling/ESR), §5(타임아웃), §8(검증 체크리스트), §9(작업 규칙). 8.0/8.3 전면 갱신 사항(defaultMaxTimeMS, cacheSizePct, TCMalloc, Write Concern 변경)은 rules와 `harness/vendor-research.md` §4 확인.
3. **공식 적용** — 전용/공유 여부에 따라 cacheSizeGB 공식이 갈린다. 정산/결제 여부가 불명확하면 보수 표준(PSS + majority + primary)을 기본으로 하고 확인 필요를 명시.
4. **불변량 검증** — PSA 여부, 캐스케이드 부등호, PostgreSQL과 병설 불가(THP always vs never, overcommit 1 vs 2).
5. **체크리스트 실행** — §8 항목 전부 판정.
6. **산출** — 컨벤션 준수(`harness/conventions.md`).

## HITL 가드

HITL-004(플랫폼개발팀 COLLSCAN 모니터링 미수행) 관련 확정은 TA 승인 전 금지. 프로파일링 도입안은 제안에 머문다.

## 출처 정책 (특히 중요)

내부 파라미터는 블로그가 아니라 **벤더 소스 코드·JIRA**로 확인한다. 선례: 2026-07-27 `internalQueryExecMaxBlockingSortBytes` 오기(블로그 32MB vs 실제 8.0 기본 100MB + rename). 상세는 `verify-standards` 스킬의 신뢰 계층과 `harness/gotchas.md` 절대 금지 #10.

## 관련 스킬

- 팀 비교: `gap-analysis` | 가이드 반영: `update-guide` | 시효성 검증: `verify-standards`
- Mongo 서버 커널 값: `linux-kernel-tuning`(swappiness 1, overcommit 1, THP always)
