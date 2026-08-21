---
name: postgresql-pgpool-tuning
description: PostgreSQL + PgPool-II 표준값 산정·튜닝 절차. shared_buffers/work_mem/effective_cache_size 메모리 산정, max_connections과 70% Ceiling 역산, autovacuum, WAL/체크포인트, 스트리밍 복제, PgPool-II num_init_children·커넥션 풀링·로드밸런싱·페일오버·Watchdog 이중화, kernel.sem 세마포어 상한을 다룬다. PostgreSQL 표준값 산정, PgPool 구성 설계, 복제/페일오버 검토, DB 관련 가이드 갱신이나 Gap 분석을 할 때 반드시 이 스킬을 사용할 것. MongoDB는 mongodb-tuning, WAS 커넥션 풀은 was-tuning을 사용.
---

# PostgreSQL/PgPool-II 튜닝

## 다루는 것

PostgreSQL(플랫폼개발팀, Master/Replica 스트리밍 복제)과 PgPool-II(풀링 + 읽기 분산 + 페일오버)의 표준값 산정. 정본은 `harness/postgresql-rules.md`(263행).

아키텍처 기준: `WAS(HikariCP) -> PgPool-II -> PostgreSQL(Master/Replica)`

## 핵심 공식 (요약 — 정본은 postgresql-rules.md §2~§5)

```
shared_buffers = RAM * 0.25        (DB 전용 서버 기준)
effective_cache_size = RAM * 0.75  (할당 아님, 플래너 참고값)
max_connections >= Sum(WAS maxPoolSize) * 1.5
70% Ceiling: Sum(maxPoolSize) <= max_connections * 0.7
idle_session_timeout = 30min
random_page_cost = 1.1 (SSD)
autovacuum: off 절대 금지, cost_limit 1000+

PgPool: num_init_children = 인스턴스별 동시 제어(4GB 전용 서버 120 시 메모리 약 1GB)
  max_pool = 1 기본 | child_life_time = 28min | 2대 이중화 + Watchdog 의무
  kernel.sem = 250 32000 250 128 (선행 조건)
```

타임아웃 캐스케이드(엄격 부등호, 등호 금지):
```
WAS maxLifetime(27min) < PgPool child_life_time(28min) < PG idle_session_timeout(30min) < 방화벽(30min)
```

## 워크플로우

1. **스펙 수집** — DB 서버 RAM/코어/스토리지(SSD 여부), 전용/공유 여부, 아키텍처(Standalone/SR/PgPool 경유), **WAS 인스턴스 수 × maxPoolSize**(70% Ceiling 검증 필수 입력).
2. **rules 로드** — `harness/postgresql-rules.md` 전체. 특히 §2(공식), §3(PgPool 기준), §4(타임아웃), §5(70% Ceiling), §7(검증 체크리스트 16항목), §8(작업 규칙).
3. **공식 적용** — 전용 서버 기준임을 명시. 공유 환경은 예외 처리 사실을 산출물에 남긴다.
4. **불변량 검증** — 70% Ceiling 실제 대입 계산, 캐스케이드 부등호, PgPool 세마포어/이중화, MongoDB와 병설 불가 확인.
5. **체크리스트 실행** — §7 항목 전부 판정(롤링 절차, Primary restart 전 failover 억제, reload 우선 포함).
6. **산출** — 컨벤션(`harness/conventions.md`): 한국어, 이모지 금지, mermaid, 단위 병기.

## 주의 함정

- `effective_cache_size`를 실제 할당처럼 안내하면 안 됨(면책 문구 필수)
- PgPool `max_pool` 불필요 상향 → 백엔드 연결 기하급수 증가
- fencing 없는 자동 페일오버 → dual-primary 데이터 분기
- PostgreSQL과 MongoDB 동일 호스트 병설 금지(overcommit/THP 충돌) — `linux-kernel-tuning` §병설 참조

## 관련 스킬

- 팀 비교: `gap-analysis` | 가이드 반영: `update-guide` | 시효성 검증: `verify-standards`
- PgPool 서버 커널 값: `linux-kernel-tuning`(kernel.sem, swappiness)
