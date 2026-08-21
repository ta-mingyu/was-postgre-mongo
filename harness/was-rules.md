# WAS Domain Rules

> WAS 서버 표준 설정 가이드라인 작업 시 에이전트가 반드시 준수해야 할 구동 규칙.
> 기준 산출물: `reports/was-standard-guide.md`

---

## 1. 도메인 스코프

### 대상 플랫폼

| WAS 엔진 | 대상 팀 | 설정 파일 | 비고 |
| :--- | :--- | :--- | :--- |
| **Apache Tomcat 9.x (독립형)** | 주차서비스팀 | `server.xml`, `setenv.sh` | maxThreads / acceptCount 직접 제어 |
| **Spring Boot 내장 Tomcat** | 플랫폼개발팀 | `application.yml` | `server.tomcat.*` 네임스페이스 |
| **IBM WebSphere Liberty 23.x** | 현금정보계팀 | `server.xml`, `jvm.options` | Executor / ConnectionManager 독자 구조 |

### 스코프 내 (IN Scope)

- JVM 메모리 (Heap, Metaspace)
- GC 알고리즘 선택 (Parallel GC / G1 GC / ZGC)
- Thread Pool (maxThreads, minSpareThreads, acceptCount)
- Connection Pool (HikariCP, Liberty ConnectionManager)
- WAS 수준의 타임아웃 (keepAliveTimeout, connectionTimeout)
- OS 커널 튜닝 (vm.swappiness, ulimit, sysctl)
- Java 버전 표준화 정책

### 스코프 외 (OUT of Scope)

- DBMS 내부 파라미터 (PostgreSQL / MongoDB / DB2)
- PgPool-II 설정
- Web Server (Apache) 설정 -- 본 도메인에서는 Web Server -> WAS 타임아웃 캐스케이드만 다룸

---

## 2. 핵심 산정 공식 (절대 준수)

에이전트는 WAS 파라미터 산출 시 아래 공식을 기본선으로 사용. 특수 환경에서만 예외 허용.

### 2.1 Thread 산정

```
maxThreads = min(CPU_cores * 50, 500)
minSpareThreads = maxThreads / 8
acceptCount = 100 (고정)
maxConnections = 8,192 (고정, NIO)
```

### 2.2 Heap 산정

```
Heap_인스턴스 = floor(호스트_RAM * 0.625) / 인스턴스_수
예외: 4 GB 단일 인스턴스 = 호스트_RAM * 0.50 (OS 가용량 보존)
```

**매트릭스 (인스턴스당)**:

| 호스트 RAM | 인스턴스 수 | Heap (Xms=Xmx) | GC 전략 |
| :--- | :---: | :--- | :--- |
| 4 GB | 1 | 2,048m | Parallel GC |
| 8 GB | 2 | 2,560m | Parallel GC |
| 16 GB | 3 | 3,413m | Parallel GC |
| 32 GB | 4 | 5,120m | G1 GC |

### 2.3 GC 결정 기준선

```
Heap <= 4,096m --> Parallel GC (-XX:+UseParallelGC)
Heap >  4,096m --> G1 GC (-XX:+UseG1GC)
Java 21+ & Heap > 4,096m --> ZGC 검토 (--XX:+UseZGC)
```

**근거**: 소형 힙에서 G1의 Region 메타데이터 오버헤드 및 Humongous Object 리스크 회피.

### 2.4 Metaspace 공통

```
MetaspaceSize = 256m (고정)
MaxMetaspaceSize = 512m (고정)
```

역전 현상(Max < Min) 절대 금지.

### 2.5 Connection Pool 산정

```
maxPoolSize = 인스턴스당 20 (기본 20)
minimumIdle = maxPoolSize (Fixed-size pool)
```

**상한 제약 (70% Ceiling Rule)**:

```
Sum(모든_인스턴스_maxPoolSize) <= DB max_connections * 0.7
```

---

## 3. 타임아웃 캐스케이드 (WAS 계층)

상위 계층이 하위 계층보다 **먼저** 연결을 종료해야 함.

### 3.1 Web Server -> WAS

```
Apache ProxyPass ttl (10s) < WAS keepAliveTimeout (15s)
```

위반 시 간헐적 502/503 에러 발생.

### 3.2 WAS -> DB (HikariCP 공통)

| 파라미터 | 표준값 | 역할 |
| :--- | :--- | :--- |
| `maxLifetime` | 1,620,000ms (27min) | DB 유휴 세션 제한(30min)보다 3분 짧게 |
| `connectionTimeout` | 30,000ms (30s) | 풀 획득 대기 상한 (Fail-Fast) |
| `keepaliveTime` | 60,000ms (1min) | 방화벽 Silent Drop 예방 |
| `minimumIdle` | = maxPoolSize | Fixed-size pool |

**핵심 제약**: `maxLifetime < DB 유휴 세션 제한 < 방화벽 TCP Established Timeout`

### 3.3 WAS -> DB (Liberty ConnectionManager)

| 파라미터 | 표준값 | 설계 근거 |
| :--- | :--- | :--- |
| `maxIdleTime` | 900s | 방화벽 타임아웃(1800s)의 50% |
| `agedTimeout` | 1,620s | DB idle_session_timeout(1800s)보다 180s 짧게 |
| `reapTime` | 60s | Reaper 주기 단축 |
| `purgePolicy` | FailingConnectionOnly | 기본값(EntirePool) 변경 필수 |

---

## 4. 팀별 특이사항 및 제약

에이전트는 팀별 설정 변경 시 아래 이력을 반드시 참조.

### 플랫폼개발팀 (Nice Park / Nice Charger)

- Java 17 / 15,25 혼합 운영 -- **현행 유지** (강제 마이그레이션 지양)
- HikariCP maxPoolSize: Nice Park 기존 5 -> **20** (과소 설정 병목 개선)
- HikariCP maxPoolSize: Nice Charger 웹 100 -> **20** (공유 DB 보호)
- maxLifetime: Nice Park 30분 -> **27분**, Nice Charger 33분 -> **27분**

### CL플랫폼팀

- **CRITICAL**: Old 영역 90.2% 사용률, Parallel GC + Heap 2048m 고정
- maxThreads -1 (무제한) -> **200** 설정 필요
- HITL-003: TA 응답 대기 중 (대응 방안 및 GC 튜닝 이력)

### 주차서비스팀

- G1 GC 사용 중 (유일) -- Heap 2048m~4096m 가변 설정
- maxPoolSize 100 -> **20** 축소 필요
- DB2 영역은 스코프 외

### 현금정보계팀

- 7개 컨테이너 다중화 환경, maxPoolSize=20 per 컨테이너 (총 105)
- Metaspace Max 16m 오류 (Min 256~512m보다 작음) -- 정정 필요
- Liberty Executor: coreThreads = CPU * 2, maxThreads = CPU * 50 명시 설정 권장

---

## 5. Java 버전 표준화 정책

| 구분 | 정책 |
| :--- | :--- |
| **신규 프로젝트** | Java 25 LTS + Spring Boot 4.x |
| **기존 운영 서비스** | 현행 유지 (15, 17, 25 그대로) |
| **권장 전환** | Non-LTS(15) -> 최신 LTS(25), 각 팀 재량 |
| **호환성 검증** | JDK 변경 시 전체 회귀 테스트 + GC 재튜닝 필수 |

---

## 6. OS 커널 튜닝 기준

### 성향 제어형 (고정)

```ini
vm.swappiness = 10
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_tw_reuse = 1
```

### 용량 제한형 (인프라 규모별)

| 파라미터 | 표준 Baseline | 확장형 (64GB+ / 인스턴스 4+) |
| :--- | :---: | :---: |
| `fs.file-max` | 100,000 | 500,000 ~ 1,000,000 |
| `net.core.somaxconn` | 2,048 | 4,096 |
| `ulimit -n` | 65,536 | 131,072 |
| `ulimit -u` | 4,096 | 16,384 |

---

## 7. 검증 체크리스트

에이전트는 WAS 설정 변경 후 반드시 아래 항목을 검증.

| # | 검증 항목 | 충족 조건 |
| :---: | :--- | :--- |
| 1 | `Xms` = `Xmx` | 프로덕션 고정 할당 |
| 2 | Heap < Container RAM * 0.7 | OOM 방지 |
| 3 | Heap = 호스트 RAM * 0.625 / N | 다중 인스턴스 분할 |
| 4 | maxPoolSize = 20 | 인스턴스당 상한 |
| 5 | Sum(maxPoolSize) <= DB max_conn * 0.7 | 70% Ceiling Rule |
| 6 | maxLifetime < DB idle_session_timeout | 캐스케이드 정합 |
| 7 | ProxyPass ttl < WAS keepAliveTimeout | 프록시 레이스 컨디션 방지 |
| 8 | minimumIdle = maxPoolSize | Fixed-size pool |
| 9 | GC 로그 활성화 | 장애 원인 분석 필수 |
| 10 | Metaspace Max >= Min | 역전 현상 방지 |
| 11 | maxThreads > 0 | 무제한(-1) 설정 금지 |
| 12 | leakDetectionThreshold = 60,000ms | 커넥션 누수 감지 |

---

## 8. 에이전트 작업 규칙

1. **기준선 유지**: 산출물(`reports/was-standard-guide.md`)의 공식과 매트릭스를 기본선으로 삼고, 명확한 근거 없이 변경하지 않음
2. **버저닝**: 가이드 갱신 시 `reports/was-standard-guide-v{N}.md` 생성 후 CLAUDE.md 링크 갱신
3. **HITL 준수**: HITL-003 (CL플랫폼 Old 영역) 관련 사항은 TA 승인 전 확정하지 않음
4. **Cross-domain 참조**: DB 설정 변경이 필요한 작업은 PostgreSQL/MongoDB harness 규칙을 추가 로드 후 진행
5. **방화벽 제약**: 사내망 TCP Established Timeout = 30분(1,800초). 모든 타임아웃 산정의 최상위 기준
