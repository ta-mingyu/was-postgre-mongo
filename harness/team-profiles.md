# 팀별 인프라 메타데이터

> 4개 팀의 **현재 운영 설정** 정규화 요약. Gap 분석 기준(현재값)으로 사용.
> 원본은 `source/{팀}.md` (읽기 전용). 본 파일은 정규화된 참조용 요약.

## 플랫폼개발팀 (Platform Develop Team)

| 항목 | 나이스파크 (Nice Park) | 나이스차저 (Nice Charger) |
| :--- | :--- | :--- |
| WAS | Spring Boot 3.5.3 내장 Tomcat | Tomcat 9 / Spring Boot 4.0.5 내장 Tomcat |
| Java | 17 | 15, 25 (혼합) |
| GC | G1 GC | G1 GC |
| Heap | 70% (heapPercent) | Xms/Xmx 2048m 고정 |
| Metaspace | 128m / Max 128m | 512m / Max 512m |
| Thread | 기본값 | 기본값 |
| Conn Pool | HikariCP max 5 / min-idle 2 | HikariCP max 100(web)/20(admin) / min-idle 5 |
| maxLifetime | 1,800,000ms (30min) | 2,000,000ms (~33min) |
| DB - PostgreSQL | PgPool-II 경유, R:W=7:3, RTO 10s/RPO 5s | PgPool-II 경유 |
| DB - MongoDB | Replica Set(Master 1/Slave 2), R:W=6:4, COLLSCAN 모니터링 미수행 | 해당없음 |
| 특이사항 | 파티셔닝 미적용, Slow Query 없음 | Java 버전 혼합 운영 |

### 이력: 마이크로서비스 구조 표준 편차 (2026-08-21, 특수 케이스로 종결)

| 항목 | 내용 |
| :--- | :--- |
| 구조 | WAS 14개 마이크로서비스 x 읽기/쓰기 계정 2개 = HikariCP 풀 28개 |
| 실제 풀 설정 | max 5 / min-idle 2 / idle-timeout 30s / max-lifetime 30min / connection-timeout 30s (was.md 표준 이탈) |
| 장비 | PgPool 4core/8GB, PostgreSQL 4core/32GB |
| 팀 자체 변경 | max_connections 100 -> 400, max_wal_senders 5 -> 10 (사전 협의 없음, postmaster 재시작 수반) |
| 관측(2026-08) | active 2 / idle 243 / idle-in-txn 3 / null 6. 실부하 대비 유휴 연결 과잉. 원인 추정: idle-timeout 30s 처닝 + PgPool 캐시 곱셈(계정 28개) |
| TA 결정 | max_wal_senders 10 유지 허용. max_connections는 100이 아닌 **200** 복귀 지시 (풀 28개 x 5 = 140 > 100 x 0.7이라 100 불가) |
| 권장 조치 순서 | 1) HikariCP fixed-size(min=max 5, idle-timeout 제거, max-lifetime 27min, keepalive 60s) 2) PgPool max_pool 1 / num_init_children 160 3) DB max_connections 200 (PgPool detach/attach 롤링). 순서 변경 시 연결 거부 장애 |
| 미확인 사항 | PgPool num_init_children/max_pool 실제값, WAS 서비스당 레플리카 수, 에러 메시지 원문 |
| 분류 | **특수 케이스** -- 가이드 v5 개정 없이 팀 한정 처리 (TA 결정 2026-08-21). "max_connections 100 고정" 표준 자체는 유지 |
| 공유 문서 | `reports/platform-team-settings-guide.md` (2026-08-21 팀장 배포용 최종 설정 안내: WAS/PgPool/PG 3단계 적용 절차 포함) |

## CL플랫폼팀 (CLS Team)

| 항목 | 값 |
| :--- | :--- |
| WAS | CLS 전용 WAS |
| Java | 15.0.2 |
| GC | Parallel GC |
| Heap | Xms/Xmx 2048m 고정 |
| Metaspace | Used 157,772K / Committed 166,528K |
| Thread | maxThreads -1 (무제한), 나머지 기본값 |
| Conn Pool | maxPoolSize 50 / minIdle 0 / maxLifetime 1800 (초 추정) |
| Timeout | connectionTimeout 30s / reapTime 300s / maxIdleTime 1800s |
| DB | PostgreSQL/MongoDB 미사용 |
| **위험 신호** | **Old 영역 90.2% 사용(1,418,698K / 1,572,864K) - CRITICAL** -- HITL-003 |
| Young 영역 | 정상(Eden 6% / From 7% / To 0%) |

## 주차서비스팀 (Park Service Team)

| 항목 | 값 |
| :--- | :--- |
| WAS | Apache Tomcat 9.0.70 |
| Java | OpenJDK 15.0.2 (64-Bit Server VM) |
| GC | G1 GC (유일) |
| Heap | Xms 2048m / Xmx 4096m (가변) |
| Thread | maxThreads 500 / minSpareThreads 25 / acceptCount 기본값(100) |
| Conn Pool | maxTotal 100 / minIdle 100 / maxWaitMillis 10,000ms |
| Timeout | connectionTimeout 20,000ms (20s) |
| DB | DB2 (본 컨설팅 범위 외 - 전담 DBA 관리) |
| 분석 범위 | WAS/JVM 설정에 한정 |

## 현금정보계팀 (Info Service Team)

| 항목 | 값 |
| :--- | :--- |
| WAS | IBM WebSphere Liberty v23.0.0.2 ND |
| Java | OpenJDK 15.0.2+7-27 |
| GC | Parallel GC (`-XX:+UseParallelGC`) |
| Thread | Liberty 동적 확장 (coreThreads -1 / maxThread -1) |
| Conn Pool | maxPoolSize 50 / minPoolSize 0 |
| Timeout | connectionTimeout 30s / maxIdleTime 1800s / reapTime 300s |
| 컨테이너 수 | 7개 (고정 3 + 동적 4) |
| DB | 본 컨설팅 범위 외 (WAS/JVM 분석에 한정) |
| 특이사항 | 고정/동적 메모리 이원 운영. NIBS 컨테이너 Heap 8GB가 최대 |

## 표준 적용 시 주요 변경 포인트 (요약)

도메인 rules에 따른 표준값 대비 주요 Gap. 상세 산출/검증은 각 도메인 rules + gap-analysis 스킬.

- **플랫폼개발**: Nice Park maxPoolSize 5->20(과소 병목), Nice Charger 웹 100->20(공유 DB 보호), maxLifetime 30/33min->27min
- **CL플랫폼**: maxThreads -1->200, Old 영역 90% CRITICAL(HITL-003 대기)
- **주차서비스**: maxPoolSize 100->20 축소
- **현금정보계**: maxPoolSize=20/컨테이너(총 105~140), Liberty Executor 명시 설정 권장, Metaspace Max 정정
