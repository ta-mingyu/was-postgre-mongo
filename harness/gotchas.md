# 함정 및 절대 금지

> 고생해서 얻은 비자명한 제약. 산출/변경 전 반드시 확인.

## 절대 금지 (Hard Boundaries)

| # | 금지 사항 | 이유 |
| :---: | :--- | :--- |
| 1 | `source/` 하위 파일 **수정** | 4개 팀 제출 원천 데이터. 읽기 전용. 변경 시 신뢰성 상실 |
| 2 | `harness/` 에 **Report 본문** 작성 | 규칙·메타데이터만. 본문은 항상 `reports/` |
| 3 | TA 승인 전 **HITL 이슈 확정/변경** | HITL-003(CL플랫폼 Old 영역), HITL-004(MongoDB COLLSCAN) |
| 4 | 이전 버전 Report 파일(`-v1`, `-v2`) **삭제** | 히스토리 보존. AGENTS.md는 최신만 가리킴 |
| 5 | 타임아웃 캐스케이드에 **등호(`<=`)** 사용 | 엄격 부등호(`<`)로 계층 격리 필수 |
| 6 | 정산/결제 도메인에 **PSA(Primary+Secondary+Arbiter) 구성** | Secondary 1대 다운 시 w:majority 쓰기 영구 정지(Stall). 반드시 PSS |
| 7 | `autovacuum = off` (PostgreSQL) | 비활성화 절대 금지 |
| 8 | WAS `maxThreads = -1`(무제한) 설정 | 반드시 0 초과 명시값 |
| 9 | Metaspace `Max < Min` (역전) | 역전 현상 절대 금지 |
| 10 | 산출물에 **일반 블로그/커뮤니티 글** 출처 인용 | 공식 문서·벤더 소스 코드·JIRA·release notes·context7 MCP만 허용. 블로그(Medium, 개인 블로그 등)는 초기 힌트 only, 최종 인용은 일차 출처로 교차 검증 필수. `verify-standards` SKILL §2 신뢰 계층 준수 |

## 도메인 공통 불변량 (한 도메인 변경 시 전체 재검증)

### 방화벽 TCP Established Timeout = 30~60분 (범위, 최단 30분=1,800초 기준 설계)

**모든 타임아웃 산정의 최상위 기준.** 방화벽 실제 timeout은 30~60분 범위(불확정)이며, 최단 30분(1,800초)을 기준으로 하위 타임아웃을 설계한다. 어느 계층 타임아웃도 최단 30분을 넘을 수 없다. HikariCP keepaliveTime(60s)이 주기적 ping으로 방화벽 유휴 타이머를 리셋해, 방화벽이 실제로 30min에 drop되는 경계 시나리오의 레이스를 방어한다.

```
사내망 방화벽(30min) > 모든 하위 타임아웃
```

### 70% Ceiling Rule

```
Sum(모든 WAS 인스턴스 maxPoolSize) <= DB max_connections * 0.7
```

- 나머지 30%는 관리자/모니터링/긴급 접속 예약.
- WAS 커넥션 풀 변경 시 PostgreSQL `max_connections` 와 MongoDB `maxIncomingConnections` 를 역산하여 위반 여부 확인.

### 타임아웃 캐스케이드 (엄격 부등호)

```
WAS HikariCP maxLifetime (27min)
    < PgPool child_life_time (28min)
        < PostgreSQL idle_session_timeout (30min)
```

MongoDB 경유:
```
WAS maxLifetime (27min) < MongoDB connectionPool maxIdleTimeMS (30min)
```

## 도메인별 핵심 산정 함정

### WAS
- Heap 매트릭스 예외: **4GB 단일 인스턴스**는 `호스트_RAM * 0.50`(OS 가용량 보존). 일반식은 `floor(호스트_RAM * 0.625) / 인스턴스_수`.
- GC 분기 기준선: Heap `<= 4096m` -> Parallel GC, `> 4096m` -> G1 GC. 소형 힙에서 G1의 Region 메타데이터/Humongous Object 오버헤드 회피.
- `minimumIdle`은 `maxPoolSize`와 동일(fixed-size pool). 미설정 권장이 아님.

### PostgreSQL
- 모든 산정 공식은 **DB 전용 서버 기준**. 공유 환경은 별도 명시.
- `effective_cache_size`는 **실제 할당이 아님**(플래너 참고값). shared_buffers와 혼동 주의.
- PgPool `max_pool = 1`(단일 DB/계정)이 기본. 불필요 상향 시 백엔드 연결 기하급수적 증가.
- 4GB RAM PgPool 전용 서버: `num_init_children=120` 구동 시 프로세스 메모리 약 1GB. 세마포어 상한 필수(`kernel.sem = 250 32000 250 128`).

### MongoDB
- WiredTiger 캐시 `cacheSizeGB = 0.5 * (RAM - 1)`은 **DB 전용 서버 기준**. 공유 환경(WAS/DB 혼합)은 25% 수준으로 명시적 제한.
- 정산/결제 서비스: Read Preference **반드시 `primary`**, Write Concern `w: majority`.
- COLLSCAN Zero-Tolerance: 발생 시 즉시 인덱스 추가. 프로파일링 미설정 상태 허용 불가.
- 버전 기준: MongoDB 8.0 LTS(온프레미스 장기지원) / 8.3 Latest. 8.0/8.3 전면 갱신 사항(defaultMaxTimeMS, cacheSizePct, TCMalloc, Write Concern 변경)은 mongodb-rules.md 참조.

## 산출물 권위 혼동

`reports/` 에 같은 가이드의 사본이 여러 버전 공존. 정본은 `reports/final/` + `reports/final-standard-guide.md`. 상세는 structure.md 의 권위 계층도.
