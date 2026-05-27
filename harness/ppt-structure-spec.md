# PPT Structure & Research Specification

> 본 파일은 컨설팅 보고서 PPT의 장표 구조 명세와 벤더별 리서치 로그를 관리한다.
> **Phase 3 전략:** 본 PPT는 "내부 확정안"이 아닌 **"컨설팅사 피드백 및 검증 유도형"** 스펙으로 설계된다.

---

## 0. PPT 설계 철학 (Discussion-Driven Structure)

```
모든 기술 영역별 슬라이드는 아래 3단계 논의 구조를 따른다:

Step 1: 현황 (Current State) - 데이터에 기반한 파편화 현행 설정값 제시
Step 2: 당사 자체 진단 (Internal Diagnosis) - Gap 분석 및 개선 가설 제시
Step 3: 컨설팅사 제언 요청 (Questions for Consultants) - 가설 검증 및 추가 제언 유도
```

본 PPT의 핵심 목적:
- 당사가 파악한 현황과 자체 진단(개선안 가설)을 명확히 제시
- 이에 대한 컨설팅사의 전문적 피드백, 검증, 그리고 당사가 놓친 추가 제언을 이끌어내는 것

---

## 1. PPT 슬라이드 덱 마스터 아웃라인 (Phase 3)

### 섹션 A: 도입부 (Framing)

| ID | 슬라이드 제목 | 구조 | Discussion Points | 상태 |
| :--- | :--- | :--- | :--- | :--- |
| A-01 | 미팅 목적 및 아젠다 | 미팅 목적, 진행 순서, 기대 산출물 | 없음 (프레이밍) | 스크립트 작성 완료 |
| A-02 | 당사 인프라 표준화 프로젝트 개요 | 프로젝트 배경, 조사 범위(4개 팀, WAS/JVM 중심), 방법론(질문지 기반) | 없음 (정보 제공) | 스크립트 작성 완료 |
| A-03 | 조사 범위 및 제외 영역 | 분석 대상: WAS/JVM/ConnPool/GC/MongoDB. 제외: DB2(전담 DBA), DB 내부 파라미터 | Q: "이 범위 설정이 적절한지, 컨설팅 측에서 추가 분석이 필요한 영역이 있는지?" | 스크립트 작성 완료 |

### 섹션 B: 전사 인프라 현황 (Current State Overview)

| ID | 슬라이드 제목 | 구조 | Discussion Points | 상태 |
| :--- | :--- | :--- | :--- | :--- |
| B-01 | 전사 인프라 아키텍처 토폴로지 | 인프라 랜드스케이프 다이어그램, WAS/Java/DB 분포 | Q: "이 수준의 파편화가 유사 기업 사례에서 흔히 관찰되는 패턴인지?" | 스크립트 작성 완료 |
| B-02 | 파편화 심각도 히트맵 | Critical/Warning/Info 3단계 분류, 10개 인사이트 매핑 | Q: "이 중 단기적으로 최우선 해결해야 할 영역에 대한 업체의 판단은?" | 스크립트 작성 완료 |
| B-03 | 4개 팀 인프라 설정 비교 총괄표 | JVM/ConnPool/GC/Thread 전체 비교 매트릭스 | 없음 (참조용) | 스크립트 작성 완료 |

### 섹션 C: 기술 영역별 심층 논의 (Deep-Dive Discussion)

#### C-1: Java 버전 파편화 및 LTS 표준화

| 단계 | 내용 |
| :--- | :--- |
| **현황** | Java 15(3팀), 17(1팀), 25(1팀, 비LTS) 혼재. 4팀 중 3팀이 2021년 3월 EOL인 Java 15.0.2 사용 |
| **당사 진단** | Java 15 비LTS EOL 상태로 보안 패치 미제공. Java 17 최소 표준, Java 21 권장 표준으로 가설 설정 |
| **Discussion Points** | Q1: "운영 중인 레거시 서비스 환경(Tomcat 9, Liberty 23, CLS WAS)을 고려할 때, Java 15에서 17 혹은 21로 가기 위한 가장 안정적인 마이그레이션 경로와 리스크 최소화 방안에 대한 업체의 의견은?" |
| | Q2: "Java 21 LTS 도입 시 ZGC 등 신규 GC 기능의 실질적 이점이 우리 워크로드에 적용 가능한지?" |
| | Q3: "동일 팀(플랫폼개발) 내 Java 15/25 혼합 운영에 대한 권장 통합 방향은?" |

#### C-2: GC 알고리즘 이원화 및 CL플랫폼팀 Old 영역 90% 리스크

| 단계 | 내용 |
| :--- | :--- |
| **현황** | G1 GC(3팀) vs Parallel GC(2팀: CL플랫폼, 현금정보계). CL플랫폼팀 ParOldGen 90.2% (1,418,698K / 1,572,864K)로 임계치 도달 |
| **당사 진단** | Parallel GC는 배치/ETL 전용으로 웹 서비스에 부적합. CL플랫폼팀은 Full GC 시 서비스 중단 위험. G1 GC 전환 및 Heap 증설(2048m -> 최소 4096m) 가설 설정 |
| **Discussion Points** | Q1: "현재 Old 영역이 90%에 육박한 상태에서 즉각적인 G1 GC 전환이 성능에 미칠 임팩트와, 힙 메모리 증설 적정 규모에 대한 업체 측 검증 의견을 요청한다" |
| | Q2: "Parallel GC에서 G1 GC로 전환 시 예상되는 일시적 성능 변화(트랜지션 기간) 및 안정화 소요 시간에 대한 경험적 가이드가 있는지?" |
| | Q3: "현금정보계팀의 경우 7개 컨테이너(고정 Heap 3개 + 동적 Heap 4개)가 모두 Parallel GC인데, 컨테이너별 특성(NIBS 8GB 고정 vs POS 1~2GB 동적)을 고려한 GC 전환 우선순위 의견은?" |

#### C-3: JVM Heap 및 Metaspace 전략 비표준화

| 단계 | 내용 |
| :--- | :--- |
| **현황** | Heap 전략 4가지 혼재: 비율(NP 70%), 고정(NC/CL 2048m), 가변(PK 2048~4096m), 혼합(현금정보계). 현금정보계팀 6개 컨테이너 Metaspace Min > Max 역전(입력오류 의심) |
| **당사 진단** | 프로덕션에서는 Xms=Xmx 고정이 벤더 공통 권장. Metaspace Max 16m은 Min(256~512m) 대비 비현실적이며 입력 오류로 추정 |
| **Discussion Points** | Q1: "Xms=Xmx 고정 할당이 모든 컨테이너 유형(고정/동적)에 동일하게 적용되어야 하는지, 아니면 동적 할당이 유리한 시나리오가 있는지?" |
| | Q2: "현금정보계팀의 고정/동적 메모리 이원화 운영 방식(고정 3개 + 동적 4개)이 업계 표준인지, 아니면 전수 고정 전환을 권장하는지?" |
| | Q3: "Metaspace Max 값을 명시하지 않고 무제한(unlimited)으로 운영할 경우의 실무적 위험도 평가를 부탁한다" |

#### C-4: Connection Pool 설정 편차 및 Timeout 연계

| 단계 | 내용 |
| :--- | :--- |
| **현황** | maxPoolSize 5~100 (20배 편차). Pool 구현체 3종(HikariCP, DBCP, Liberty ConnMgr) 혼재. maxLifetime 단위 불일치(ms vs sec). 주차서비스팀 maxLifetime/maxWaitMillis 혼동 의심 |
| **당사 진단** | HikariCP는 fixed-size pool(min=max) 권장. maxLifetime은 DB wait_timeout보다 짧아야 함. NC의 2,000,000ms는 PgPool timeout(30분) 초과 위험 |
| **Discussion Points** | Q1: "HikariCP의 fixed-size pool(min=max) 전환 시 초기 커넥션 생성으로 인한 시작 시간 지연에 대한 실무적 대응 방안은?" |
| | Q2: "WAS-DB 간 타임아웃 캐스케이드(connectionTimeout -> maxLifetime -> DB wait_timeout -> PgPool)의 권장 설정값 체인에 대한 업체의 베스트 프랙티스를 요청한다" |
| | Q3: "Pool 구현체를 전사 표준으로 하나(HikariCP 등)로 통일하는 것이 권장되는지, 아니면 WAS 종류별(Tomcat/Liberty)로 다른 구현체를 사용해도 무방한지?" |

#### C-5: Thread 설정 극단적 편차

| 단계 | 내용 |
| :--- | :--- |
| **현황** | CL플랫폼팀 maxThreads -1(무제한) vs 주차서비스팀 500 vs 플랫폼개발팀 기본값(200). 현금정보계팀은 Liberty 자동 튜닝(coreThreads=-1, maxThreads=-1) |
| **당사 진단** | CL플랫폼팀의 -1(무제한)은 백프레셔(backpressure) 부재로 자원 고갈 위험. Liberty의 -1은 정상 동작(자동 튜닝). Tomcat 기본값 200이 대부분의 워크로드에 적합 |
| **Discussion Points** | Q1: "CL플랫폼팀의 maxThreads=-1(무제한) 설정이 현재 문제 없이 운영되고 있다면, 굳이 변경해야 하는지? 변경 시 적정값 산출 기준은?" |
| | Q2: "주차서비스팀 maxThreads=500은 기본값(200) 대비 과대 설정인데, 트래픽 패턴을 모를 때의 권장 접근법은?" |
| | Q3: "Liberty의 자동 튜닝 메커니즘(1.5초 간격 throughput 측정)이 우리 워크로드에 적합한지 검증 방법은?" |

#### C-6: MongoDB Replica Set 운영 및 모니터링

| 단계 | 내용 |
| :--- | :--- |
| **현황** | 플랫폼개발팀 MongoDB: Replica Set(M1/S2/A0), Read:Write=6:4, COLLSCAN 모니터링 미수행. 인덱스는 고정 필드 위주 |
| **당사 진단** | COLLSCAN 미감시는 인덱스 누락 시 선형적 성능 저하 위험. Read 60%를 Secondary 분산(secondaryPreferred)으로 부하 완화 가능 |
| **Discussion Points** | Q1: "서비스 초기 단계(데이터 증가량 미측정)에서 MongoDB 인덱스 전략을 어떻게 수립해야 하는지? 사전 인덱스 설계 vs 사용 패턴 기반 점진적 인덱싱?" |
| | Q2: "현재 COLLSCAN 미감시 상태에서 프로덕션 환경의 실제 COLLSCAN 발생 빈도를 확인하기 위한 최소 모니터링 체계(Profiling Level, 주기)를 권장해달라" |
| | Q3: "Read:Write=6:4 비율에서 secondaryPreferred 적용 시 Primary-Secondary 간 데이터 일관성(oplog lag)에 대한 실무적 주의점은?" |

### 섹션 D: 종합 개선 로드맵 및 우선순위 논의

| ID | 슬라이드 제목 | 구조 | Discussion Points | 상태 |
| :--- | :--- | :--- | :--- | :--- |
| D-01 | 당사 제안 우선순위 로드맵 | P0(Critical)~P3(Low) 단계별 개선 항목 정리. P0: CL플랫폼 G1 GC 전환/Heap 증설, P1: Metaspace 정정/Java 업그레이드 | Q1: "이 우선순위가 적절한지? 현장 경험상 다르게 접근해야 할 항목이 있는지?" | 스크립트 작성 완료 |
| | | | Q2: "P0(P critical) 항목을 단계별로 적용할 때, 각 단계 간 검증(Validation) 포인트로 어떤 지표를 모니터링해야 하는지?" | |
| D-02 | 당사가 놓친 영역에 대한 질문 | 현재 분석에서 다루지 못한 영역(GC 로그 수집, Full GC 이력, Peak 타임 데이터, 모니터링 체계 전무) | Q: "우리가 놓치고 있거나 간과한 중요한 인프라 설정 항목이 있는지? 특히 JVM 모니터링, GC 로그 수집, 알림 체계 측면에서" | 스크립트 작성 완료 |
| D-03 | 후속 워크숍 계획 및 Next Steps | 후속 미팅 일정, 추가 데이터 수집 요청, 컨설팅사 최종 Deliverable 기대사항 | Q: "이번 미팅 결과를 바탕으로, 컨설팅 측에서 다음 단계까지 준비해 주실 수 있는 산출물은?" | 스크립트 작성 완료 |

### 섹션 E: 부록 (Reference)

| ID | 슬라이드 제목 | 내용 | 상태 |
| :--- | :--- | :--- | :--- |
| E-01 | 4개 팀 원천 데이터 요약 | 각 팀 질문지 핵심 데이터 테이블 | 참조용 |
| E-02 | 벤더 권장사양 참조 | Tomcat, Liberty, JVM, HikariCP, MongoDB 공식 권장값 | 참조용 |
| E-03 | 용어 정리 | GC, Heap, Connection Pool, Metaspace 등 기술 용어 | 참조용 |

---

## 2. 벤더별 리서치 로그

> 각 벤더의 최신 권장 튜닝 파라미터를 리서치한 결과를 기록한다.

### 2.1 리서치 대상 벤더 및 항목

| 벤더 | 제품 | 핵심 리서치 항목 | 상태 |
| :--- | :--- | :--- | :--- |
| Apache | Tomcat (9.x) | maxThreads, acceptCount, connectionTimeout 권장값 | 완료 |
| Pivotal/VMware | Spring Boot (내장 Tomcat) | server.tomcat.* 기본값 및 권장 튜닝 | 완료 |
| IBM | WebSphere Liberty (23.x) | executor, connectionManager, JVM 권장 설정 | 완료 |
| MongoDB | Replica Set | 인덱스 전략, COLLSCAN 방지, 모니터링 | 완료 |
| Oracle/OpenJDK | JVM (15/17/21) | G1 GC vs Parallel GC 권장, Heap 사이징 공식 | 완료 |
| Brett Wooldridge | HikariCP | maximumPoolSize, maxLifetime, connectionTimeout | 완료 |

### 2.2 리서치 결과 상세

#### 2.2.1 Apache Tomcat 9.x 권장 설정

| 파라미터 | 기본값 | 권장 범위 | 출처 |
| :--- | :--- | :--- | :--- |
| **maxThreads** | 200 | 200 ~ 500 (CPU 코어 수 * 8~20, 워크로드에 따라) | Tomcat 공식 config |
| **minSpareThreads** | 25 | 25 (기본값 유지 권장) | Tomcat 공식 config |
| **acceptCount** | 100 | 100 (기본값 유지). maxThreads와 동일 수준 | Tomcat 공식 config |
| **connectionTimeout** | 60000 ms | 20000~60000 ms (20~60초) | Tomcat 공식 config |
| **maxConnections** | 8192 (NIO) | 8192 (기본값 유지). 시스템 FD 한계 고려 | Tomcat 공식 config |

**참고:** Tomcat은 특정 워크로드에 대한 "정답"을 제공하지 않음. 기본값이 대부분의 웹 애플리케이션에 적합하며, 튜닝은 부하 테스트 후 수행할 것을 권장.

#### 2.2.2 Spring Boot 내장 Tomcat 권장 설정

| 파라미터 | 기본값 | 비고 |
| :--- | :--- | :--- |
| **server.tomcat.threads.max** | 200 | Tomcat maxThreads와 동일 |
| **server.tomcat.threads.min-spare** | 10 | 독립 Tomcat(25)보다 낮음 |
| **server.tomcat.max-connections** | 8192 | NIO 기본값 |
| **server.tomcat.accept-count** | 100 | 기본값 |
| **server.tomcat.connection-timeout** | 20000 ms (20초) | 독립 Tomcat(60초)보다 짧음 |

**플랫폼개발팀 적용:** NP/NC 모두 "기본값 사용"으로 응답. Spring Boot의 기본값 자체가 합리적이므로, 현행 설정이 벤더 권장 범위 내에 있음.

#### 2.2.3 IBM WebSphere Liberty 23.x 권장 설정

**Thread Pool (Executor):**

| 파라미터 | 기본값 | 권장 사항 | 출처 |
| :--- | :--- | :--- | :--- |
| **coreThreads** | CPUs * 2 | 기본값 권장. 컨테이너 환경에서는 명시적 설정 고려 | Open Liberty 공식 |
| **maxThreads** | -1 (무제한) | 대부분의 워크로드에서 기본값 권장 (자동 튜닝). 문제 발생 시 CPUs * 2로 시작 | Open Liberty 블로그 |
| **자동 튜닝** | 1.5초 간격 | Throughput 기반으로 동적 조정. 수동 튜닝 불권장 | IBM Performance Cookbook |

**Connection Manager:**

| 파라미터 | 기본값 | 권장 사항 | 출처 |
| :--- | :--- | :--- | :--- |
| **maxPoolSize** | 50 | coreThreads와 1:1 매핑으로 시작 | Open Liberty 공식 |
| **minPoolSize** | 0 | 성능 극대화 시 maxPoolSize와 동일하게 설정 | IBM Performance Cookbook |
| **connectionTimeout** | 30s | 기본값 유지 | Open Liberty 공식 |
| **maxIdleTime** | - | 성능 극대화 시 -1 (비활성화) | IBM Performance Cookbook |
| **reapTime** | 3m (180s) | 성능 극대화 시 -1 (비활성화). 단, 방화벽 환경에서는 유지 | IBM Performance Cookbook |
| **purgePolicy** | EntirePool | **FailingConnectionOnly** 권장 (전체 풀 플러시 방지) | Open Liberty 공식 |

**현금정보계팀 적용:** coreThreads=-1 / maxThreads=-1은 Liberty의 기본 동작(자동 튜닝)에 부합. 단, 컨테이너 환경에서 자원 제한이 있을 경우 명시적 maxThreads 설정 필요.

#### 2.2.4 HikariCP 권장 설정

| 파라미터 | 기본값 | 권장 사항 | 출처 |
| :--- | :--- | :--- | :--- |
| **maximumPoolSize** | 10 | `connections = ((core_count * 2) + effective_spindle_count)`. 단, 이 공식은 일반적인 가이드이며 실제 워크로드에 따라 조정 필요 | HikariCP Wiki (Brett Wooldridge) |
| **minimumIdle** | maximumPoolSize와 동일 | **미설정 권장** (fixed-size pool). HikariCP는 고정 크기 풀에서 최고 성능 발휘 | HikariCP README |
| **maxLifetime** | 1,800,000 ms (30분) | DB의 wait_timeout보다 **몇 초 짧게** 설정. 최소 30,000 ms | HikariCP README |
| **connectionTimeout** | 30,000 ms (30초) | 기본값 유지 | HikariCP README |
| **idleTimeout** | 600,000 ms (10분) | minimumIdle < maximumPoolSize인 경우에만 적용 | HikariCP README |

**핵심 권장:** HikariCP 저자(Brett Wooldridge)는 "최대 성능을 위해 minimumIdle을 설정하지 말고 fixed-size pool로 운영하라"고 명시.

#### 2.2.5 JVM GC 및 메모리 권장 설정

**GC 알고리즘 선택 기준:**

| GC 유형 | 적합 워크로드 | 대상 Heap | 최대 Pause | 권장 여부 |
| :--- | :--- | :--- | :--- | :--- |
| **G1 GC** | 범용 웹/앱 서버 | 4GB ~ 256GB | 50~200 ms | **권장 (Java 9+ 기본)** |
| **Parallel GC** | 배치/ETP/오프라인 | 제한 없음 | 수초 (Heap 비례) | **웹 서비스 비권장** |
| **ZGC** | 초저지연 실시간 | 8MB ~ 16TB | <1 ms (Java 16+) | Java 17+ 고급 옵션 |
| **Shenandoah** | 저지연 범용 | 제한 없음 | <10 ms | Java 15+ 대안 |

**출처:** Oracle/OpenJDK 공식 문서, GC 튜닝 벤치마크 (prgrmmng.com, heaphero.io, roninsway.dev)

**Heap 사이징 권장:**

| 항목 | 권장 사항 | 출처 |
| :--- | :--- | :--- |
| **Xms = Xmx** | 프로덕션에서 반드시 동일 설정 (리사이즈 pause 방지) | JVM 튜닝 가이드 전반 |
| **Heap 크기 기준** | 컨테이너/VM 메모리의 **50~70%** | jarviix.com, heaphero.io |
| **GC 시간 비율** | 전체 런타임 대비 **5~10% 미만** 유지. 초과 시 Heap 증설 필요 | jarviix.com |
| **Old Gen 경고 임계치** | **70~80%** 도달 시 경고. 90% 이상은 즉각 조치 필요 | 일반 JVM 모니터링 권고 |
| **Metaspace Min** | 128m ~ 256m (소규모), 256m ~ 512m (대규모/Spring Boot) | 실무 권장 |
| **Metaspace Max** | Min의 **2배** 이상 설정. 미설정 시 무제한(권장) 또는 256m~512m | 실무 권장 |

**Java LTS 권장:**

| 버전 | LTS 여부 | EOL | 권장 |
| :--- | :--- | :--- | :--- |
| **Java 15** | 비LTS | 2021-03 EOL | **표준화 대상에서 제외 필요** |
| **Java 17** | LTS | 2029-09 (예정) | **최소 표준** (현행 NP 적합) |
| **Java 21** | LTS | 2031-09 (예정) | **권장 표준** (최신 LTS) |
| **Java 25** | 비LTS | 2025-09 (예정 EOL) | 비LTS, 장기 운영 비권장 |

**Parallel GC 웹 서비스 비권장 근거:**
- Parallel GC는 Full GC 시 **stop-the-world 단일 스레드**로 전체 Old 영역 정리
- Pause 시간이 Heap 크기에 비례하여 증가 (16GB Heap에서 수초)
- CL플랫폼팀(Heap 2GB, Old 90%)은 Full GC 발생 시 예측 불가한 서비스 중단 위험

#### 2.2.6 MongoDB Replica Set 권장 설정

| 항목 | 권장 사항 | 출처 |
| :--- | :--- | :--- |
| **COLLSCAN 감지** | `db.setProfilingLevel(1, { slowms: 100 })` 설정. `system.profile` 컬렉션에서 `stage: "COLLSCAN"` 모니터링 | MongoDB 공식 |
| **explain() 활용** | 정기적으로 `db.collection.find(...).explain("executionStats")` 실행. `totalDocsExamined`가 `nReturned`의 10배 이상이면 인덱스 누락 의심 | MongoDB 공식 |
| **인덱스 전략** | ESR 규칙 (Equality, Sort, Range 순서)으로 복합 인덱스 구성. 고정 쿼리 패턴에 맞춰 설계 | MongoDB 공식 |
| **Read Preference** | Read:Write = 6:4 환경에서 `secondaryPreferred` 권장. Primary 부하 분산 | MongoDB 공식 |
| **Write Concern** | `w: 1` (기본) 또는 `w: majority` (강한 일관성 필요 시). RTO 10초/RPO 5초 요구사항 시 `w: majority` 권장 | MongoDB 공식 |
| **Replica Set 구성** | Master 1 / Slave 2 / Arbiter 0은 유효한 구성. 자동 장애조치 가능 | MongoDB 공식 |
| **Connection Pool** | MongoDB Driver 기본값: maxPoolSize=100, minPoolSize=0. 애플리케이션 스레드 수와 매칭 | MongoDB 공식 |

---

## 3. 표준화 연구 기록

### 3.1 분석 기준 (Criteria)

| 영역 | 비교 기준 | 출처 |
| :--- | :--- | :--- |
| **JVM Heap** | 컨테이너 메모리 50~70%, Xms=Xmx 고정 | JVM 튜닝 가이드 |
| **GC 알고리즘** | 웹 서비스 = G1 GC (Java 9+ 기본) | Oracle/OpenJDK, 벤치마크 |
| **Connection Pool** | HikariCP: fixed-size pool, maxLifetime < DB wait_timeout | HikariCP Wiki |
| **Thread** | Tomcat: 기본값 유지. Liberty: 자동 튜닝 (수동 설정 불권장) | Tomcat/Liberty 공식 |
| **Java 버전** | 최소 Java 17 LTS, 권장 Java 21 LTS | Oracle Java SE 지원 로드맵 |

### 3.2 표준값 도출 현황

#### 전사 표준 권장값 (Phase 2 도출)

| 영역 | 파라미터 | 표준 권장값 | 근거 |
| :--- | :--- | :--- | :--- |
| **Java 버전** | 최소 LTS | Java 17 | 비LTS(15) EOL, 보안 패치 미제공 |
| **Java 버전** | 권장 LTS | Java 21 | 최신 LTS, ZGC 프로덕션 레디 |
| **GC** | 표준 알고리즘 | G1 GC (`-XX:+UseG1GC`) | 웹 서비스 표준, Java 9+ 기본 |
| **Heap** | Xms/Xmx | 동일하게 설정 (고정) | 리사이즈 pause 방지 |
| **Heap** | 크기 기준 | 컨테이너 메모리 50~70% | off-heap, thread stack 고려 |
| **Metaspace** | Min | 256m | Spring Boot/대규모 앱 기준 |
| **Metaspace** | Max | 512m 이상 (또는 미설정=무제한) | Min의 2배 이상 |
| **Conn Pool** | maxLifetime | 1,740,000 ms (29분) | DB wait_timeout(30분)보다 짧게 |
| **Conn Pool** | connectionTimeout | 30,000 ms (30초) | HikariCP 기본값 |
| **Conn Pool** | minimumIdle | maximumPoolSize와 동일 | Fixed-size pool 권장 |
| **Thread** | maxThreads (Tomcat) | 200 (기본값) | 대부분의 워크로드에 적합 |
| **Thread** | Liberty Executor | 기본값 (자동 튜닝) | Liberty 공식 불권장 |

---

## 4. PPT 프레젠테이션 스크립트

> 각 슬라이드별 발표 스크립트(발표자용). 컨설팅사 미팅 시 참조.

### A-01: 미팅 목적 및 아젠다

**발표 스크립트:**
"안녕하십니까. 오늘 미팅은 당사 전사 인프라 표준화 프로젝트의 현황을 공유하고, 컨설팅 측의 전문적 검증과 추가 제언을 구하기 위해 마련되었습니다. 당사에서 4개 팀의 WAS 및 JVM 설정 현황을 분석한 결과와 자체 개선 가설을 제시할 테니, 각 영역에 대한 업체의 의견과 베스트 프랙티스를 부탁드립니다. 오늘 논의된 내용을 바탕으로 최종 표준화 방안을 수립할 계획입니다."

### A-03: 조사 범위

**발표 스크립트:**
"분석 범위는 WAS 환경, Java 런타임, JVM 메모리 전략, GC 알고리즘, Connection Pool 설정입니다. DB2는 전담 DBA가 관리하는 별도 영역이므로 제외했습니다. 이 범위 설정이 적절한지, 컨설팅 측에서 추가 분석이 필요한 영역이 있다면 말씀 부탁드립니다."

### B-01: 전사 인프라 아키텍처 토폴로지

**발표 스크립트:**
"현재 당사는 4개 팀이 각기 다른 WAS, Java 버전, GC 알고리즘, Connection Pool 설정을 사용하고 있습니다. Apache Tomcat(독립 및 Spring Boot 내장), IBM WebSphere Liberty, CLS 전용 WAS 등 3종의 WAS가 혼재하며, Java는 15, 17, 25 버전이 공존합니다. 이 정도의 파편화가 유사 규모 기업에서도 흔히 관찰되는 패턴인지 궁금합니다."

### C-1: Java 버전 파편화

**발표 스크립트:**
"Java 버전 현황을 보면, 4개 팀 중 3개 팀이 2021년 3월에 EOL된 Java 15를 사용 중입니다. 한 팀 내에서도 15와 25가 혼합 운영되고 있습니다. 당사의 자체 진단으로는 Java 17을 최소 표준, 21을 권장 표준으로 설정해야 한다고 판단하고 있습니다. 운영 중인 레거시 서비스 환경을 고려할 때, Java 15에서 17 혹은 21로 가기 위한 가장 안정적인 마이그레이션 경로와 리스크 최소화 방안에 대한 업체의 의견을 부탁드립니다."

### C-2: GC 알고리즘 및 CL플랫폼팀 리스크

**발표 스크립트:**
"가장 심각한 이슈입니다. 2개 팀이 웹 서비스에 부적합한 Parallel GC를 사용 중이며, 특히 CL플랫폼팀은 Old 영역 사용률이 90%에 육박하고 있습니다. Heap은 2048m으로 고정되어 있어 확장 불가 상태입니다. 당사는 G1 GC 전환과 Heap 4096m 증설을 가설로 설정했습니다. 현재 Old 영역이 90%에 육박한 상태에서 즉각적인 G1 GC 전환이 성능에 미칠 임팩트와, 힙 메모리 증설 적정 규모에 대한 업체 측 검증 의견을 요청드립니다."

### C-3: Heap 및 Metaspace 전략

**발표 스크립트:**
"Heap 할당 전략이 비율 기반, 고정, 가변, 혼합 등 4가지 방식으로 파편화되어 있습니다. 현금정보계팀의 경우 6개 컨테이너에서 Metaspace Max값이 16m로 설정되어 있는데, 이는 Min값(256~512m)보다 작아 입력 오류로 의심됩니다. Xms=Xmx 고정 할당이 모든 컨테이너 유형에 동일하게 적용되어야 하는지, 아니면 동적 할당이 유리한 시나리오가 있는지 업체의 의견을 듣고 싶습니다."

### C-4: Connection Pool 설정 편차

**발표 스크립트:**
"maxPoolSize가 5에서 100까지 20배 편차가 있습니다. Pool 구현체도 HikariCP, DBCP, Liberty ConnMgr 3종이 혼재합니다. 특히 타임아웃 설정에서 단위 불일치(ms vs 초)가 발견되었고, 한 팀은 maxLifetime과 maxWaitMillis를 혼동한 것으로 의심됩니다. WAS-DB 간 타임아웃 캐스케이드의 권장 설정값 체인에 대한 업체의 베스트 프랙티스를 요청드립니다."

### C-5: Thread 설정 편차

**발표 스크립트:**
"CL플랫폼팀은 maxThreads를 -1(무제한)으로 설정하여 백프레셔가 전혀 없는 상태입니다. 반면 주차서비스팀은 500으로 설정했습니다. CL플랫폼팀의 무제한 설정이 현재 문제 없이 운영되고 있다면 굳이 변경해야 하는지, 변경 시 적정값 산출 기준은 무엇인지 궁금합니다."

### C-6: MongoDB 운영

**발표 스크립트:**
"플랫폼개발팀의 MongoDB는 COLLSCAN(풀 스캔) 발생 여부를 전혀 모니터링하지 않고 있습니다. 서비스 초기 단계라 데이터가 적어 문제가 없을 수 있으나, 데이터 증가 시 선형적 성능 저하 위험이 있습니다. 초기 단계에서의 인덱스 전략 수립 방법과 최소 모니터링 체계에 대한 권장을 부탁드립니다."

### D-01: 우선순위 로드맵

**발표 스크립트:**
"당사가 제안하는 우선순위는 다음과 같습니다. P0로 CL플랫폼팀의 G1 GC 전환 및 Heap 증설, P1로 현금정보계팀 Metaspace 정정 및 GC 전환, 그리고 전사적 Java 17 마이그레이션을 제안합니다. 이 우선순위가 적절한지, 현장 경험상 다르게 접근해야 할 항목이 있다면 말씀 부탁드립니다."

### D-02: 당사가 놓친 영역

**발표 스크립트:**
"마지막으로, 현재 분석에서 다루지 못한 영역이 있습니다. 4개 팀 모두 GC 로그 수집 여부, Full GC 발생 이력, Peak 타임 메모리 사용률 데이터가 누락되어 있어 정확한 리스크 평가에 한계가 있습니다. 우리가 놓치고 있거나 간과한 중요한 인프라 설정 항목, 특히 JVM 모니터링과 알림 체계 측면에서 추가로 검토해야 할 사항이 있다면 꼭 말씀 부탁드립니다."

---

## 5. 섹션 C 상세 슬라이드 스펙 (Phase 3 생성)

> C-01 ~ C-06 각 슬라이드의 3단계 구조(현황/진단/질문) 상세 콘텐츠.
> 본 섹션은 PPT 아티팩트 생성(Phase 4) 시 직접 소스로 활용된다.

### C-01: Java 버전 파편화 및 비LTS(Java 15) EOL 리스크

**Step 1 (현황):** 5개 런타임 중 4개(80%)가 EOL 상태인 Java 15 사용. 플랫폼개발(NP)만 Java 17 LTS. NC는 15/25 혼재.
**Step 2 (진단):** 최소 Java 17 LTS, 권장 Java 21 LTS 표준화 가설. 보안 패치 미제공 상태 5년 경과.
**Step 3 (질문):** Q1 마이그레이션 경로, Q2 Java 17 vs 21 선택, Q3 NC 혼재 해소. (상세 콘텐츠는 본 세션 출력 참조)

### C-02: GC 알고리즘 이원화 및 CL플랫폼 Old 영역 90% 임계치

**Step 1 (현황):** G1 GC 3팀 vs Parallel GC 2팀. CL플랫폼 Old Gen 90.2% (1,418,698K/1,572,864K). Heap 2048m 고정.
**Step 2 (진단):** 전사 G1 GC 통일 가설. CL플랫폼 Heap 4096m 증설 필요. Full GC 시 수초~수십초 STW 위험.
**Step 3 (질문):** Q1 G1 전환 임팩트/안정화 소요, Q2 Heap 증설 적정 규모, **Q3 CLS 전용 WAS 엔진의 GC 전환 호환성 (신규 추가)**, Q4 현금정보계 컨테이너별 전환 우선순위. (상세 콘텐츠는 본 세션 출력 참조)

### C-03: Heap/Metaspace 비표준 할당 및 현금정보계 설정 정정

**Step 1 (현황):** 4가지 Heap 전략 혼재(비율/고정/가변/혼합). 현금정보계 6개 컨테이너 Metaspace Min > Max 역전(16m).
**Step 2 (진단):** Xms=Xmx 고정 통일 가설. Metaspace Max 16m은 입력 오류 의심.
**Step 3 (질문):** Q1 Xms=Xmx 보편적 적용 여부, Q2 Metaspace Max 정책, Q3 역전 현상 실제 영향. (상세 콘텐츠는 본 세션 출력 참조)

### C-04: Connection Pool 구현체 편차 및 Timeout 캐스케이드

**Step 1 (현황):** maxPool 5~100 (20배 편차). 3종 구현체 혼재. maxLifetime 단위 불일치. NC 2,000,000ms(33분) > PgPool timeout.
**Step 2 (진단):** HikariCP fixed-size pool(min=max) 권장. maxLifetime < DB timeout 원칙. 주차 maxLifetime/maxWaitMillis 혼동.
**Step 3 (질문):** Q1 Timeout 캐스케이드 권장값, Q2 Pool 구현체 통일 여부, Q3 Fixed-size Pool 트레이드오프. (상세 콘텐츠는 본 세션 출력 참조)

### C-05: Thread Pool 극단적 편차

**Step 1 (현황):** CL플랫폼 maxThreads -1(무제한), 주차 500, 플랫폼개발 기본값(200), 현금정보계 Liberty 자동 튜닝.
**Step 2 (진단):** CL플랫폼 무제한은 백프레셔 부재로 HIGH RISK. Liberty 자동 튜닝은 정상 동작이나 Java 15 + 컨테이너 환경에서 CPU 오인식 리스크 존재.
**Step 3 (질문):** Q1 무제한 스레드 위험도 평가, Q2 과대 설정 진단 방법, **Q3 Java 15 + Liberty 자동 튜닝 결합 리스크: 컨테이너 환경에서 CPU 자원 오인식으로 인한 과도한 스레드 생성 및 CPU 쓰로틀링 위험 팩트 체크 (신규 추가)**. (상세 콘텐츠는 본 세션 출력 참조)

### C-06: MongoDB 아키텍처 패턴 및 COLLSCAN 모니터링

**Step 1 (현황):** Replica Set(M1/S2/A0), Read:Write=6:4, COLLSCAN 미감시, 고정 인덱스, 서비스 초기.
**Step 2 (진단):** Profiling Level 1 도입, secondaryPreferred 검토, ESR 인덱스 설계 가설.
**Step 3 (질문):** Q1 초기 인덱스 전략, Q2 최소 모니터링 체계, Q3 secondaryPreferred 일관성. (상세 콘텐츠는 본 세션 출력 참조)

---

## 6. 섹션 B 상세 슬라이드 스펙 (Phase 3 생성)

> B-01 ~ B-03 각 슬라이드의 콘텐츠(아키텍처 토폴로지, 히트맵, 비교 총괄표).
> 본 섹션은 PPT 아티팩트 생성(Phase 4) 시 직접 소스로 활용된다.

### B-01: 전사 인프라 아키텍처 토폴로지

**Mermaid 다이어그램:** 클라이언트 -> WAS 계층(3종 혼재: Tomcat/Spring Boot, CLS 전용, Liberty) -> DB 계층(PostgreSQL/PgPool-II, MongoDB Replica Set, DB2) 전경. CL플랫폼 Old Gen 90% CRITICAL 강조. (상세 콘텐츠는 본 세션 출력 참조)
**요약 테이블:** 4개 팀 x 6개 설정 항목(WAS/Java/GC/Heap/Pool/DB) 매핑.
**Discussion Point:** "이 수준의 파편화가 유사 규모 기업 사례에서 흔히 관찰되는 패턴인지?"
**발표 스크립트:** "전사 인프라 아키텍처 전경입니다. 4개 팀이 3종의 WAS를 사용 중이며... 유사 규모 기업에서도 흔히 관찰되는지 업체의 경험적 관점이 궁금합니다."

### B-02: 파편화 심각도 히트맵

**Mermaid quadrantChart:** x축=파편화 수준, y축=운영 리스크. Critical 3건(CL Old 90%, Java 15 EOL, Metaspace 역전), Warning 4건(GC 이원화, 무제한 스레드, maxLifetime 초과, COLLSCAN 미감시), Info 3건(Heap 4종, Pool 3종, NC 버전 혼재). (상세 콘텐츠는 본 세션 출력 참조)
**심각도 분류 테이블:** 10개 인사이트의 등급/팀/현황 데이터/근거 상세.
**Discussion Point:** "단기적으로 최우선 해결해야 할 영역에 대한 업체의 판단은?"
**발표 스크립트:** "파편화 심각도를 Critical, Warning, Info 3단계로 분류했습니다. Critical은 3건... CL플랫폼 Old Gen 90%를 최우선으로 분류했는데, 업체 관점에서 이 우선순위가 적절한지 궁금합니다."

### B-03: 4개 팀 인프라 설정 비교 총괄표

**비교 테이블 3종:** (1) JVM Runtime 및 메모리 설정 비교 (Java/WAS/GC/Xms/Xmx/Metaspace/Old Gen 10행 x 5팀), (2) Connection Pool 및 Thread 설정 비교 (Pool/maxPool/minIdle/maxLifetime/timeout/maxThreads 10행 x 5팀), (3) DB 연결 구조 비교 (DB/연결방식/HA/Read:Write/SlowQuery 5행 x 5팀). (상세 콘텐츠는 본 세션 출력 참조)
**성격:** 참조용 (Discussion Point 없음. 컨설팅사가 전체 현황을 한눈에 파악할 수 있도록 제공).
**발표 스크립트:** "4개 팀의 전체 설정 비교 총괄표입니다... 상세한 내용은 다음 섹션 C에서 기술 영역별로 깊게 다루겠습니다."

---

## 7. 섹션 A/D/E 상세 슬라이드 스펙 (Phase 3 생성)

> A-01~A-03, D-01~D-03, E-01~E-03 슬라이드의 콘텐츠.
> 본 섹션은 PPT 아티팩트 생성(Phase 4) 시 직접 소스로 활용된다.

### A-01: 미팅 목적 및 아젠다

**Mermaid 다이어그램:** 4단계 아젠다 (도입 -> 현황 공유 -> 심층 논의 -> 로드맵). 소요 시간 5분/15분/40분/15분. (상세 콘텐츠는 본 세션 출력 참조)
**핵심:** 당사 현황과 자체 진단(개선 가설) 제시 -> 컨설팅사 검증/피드백/추가 제언 유도.
**발표 스크립트:** "오늘 미팅은 당사 전사 인프라 표준화 프로젝트의 현황을 공유하고, 컨설팅 측의 전문적 검증과 추가 제언을 구하기 위해 마련되었습니다..."

### A-02: 당사 인프라 표준화 프로젝트 개요

**Mermaid 다이어그램:** 프로젝트 배경(파편화/가이드 부재/컨설팅 도입) -> 조사 방법론(질문지 -> 수집 -> Gap 분석). (상세 콘텐츠는 본 세션 출력 참조)
**프로젝트 개요 테이블:** 프로젝트명, 배경, 조사 대상(4개 팀), 조사 범위, 조사 방법, 현재 Phase.
**발표 스크립트:** "4개 팀이 각기 다른 WAS, Java 버전, GC 알고리즘, Connection Pool 설정을 사용하고 있습니다. 표준화된 튜닝 가이드가 없어 담당자의 경험에 의존해서 설정이 관리되고 있는 실정입니다..."

### A-03: 조사 범위 및 제외 영역

**Mermaid 다이어그램:** In Scope(WAS/Java/JVM/GC/Pool/MongoDB 6개 영역) vs Out of Scope(DB2/DB 내부/OS/현금정보계 DB). (상세 콘텐츠는 본 세션 출력 참조)
**범위 설정 근거 테이블:** 포함 6개 항목 + 제외 4개 항목의 근거 명시. HITL-001/002 Closed 근거 포함.
**Discussion Point:** "이 범위 설정이 적절한지, 컨설팅 측에서 추가 분석이 필요한 영역이 있는지?"
**발표 스크립트:** "분석 범위는 WAS 환경 3종, Java 런타임, JVM Heap과 Metaspace, GC 알고리즘, Connection Pool 설정, 그리고 플랫폼개발팀의 MongoDB입니다..."

### D-01: 당사 제안 우선순위 로드맵

**Mermaid gantt chart:** P0(Critical, 1주) -> P1(High, 1~2주) -> P2(Medium, 2~3주) -> P3(Low, 3~4주). 12개 개선 항목 일정 매핑. (상세 콘텐츠는 본 세션 출력 참조)
**우선순위 상세 테이블:** P0 3건(CL GC 전환, Metaspace 정정, GC 로그 활성화), P1 3건, P2 3건, P3 3건. 각 항목별 대상 팀, 개선 내용, 검증 지표, 예상 소요.
**Discussion Points:** Q1 우선순위 검증, Q2 단계별 검증 포인트.
**발표 스크립트:** "종합 개선 로드맵을 제안드립니다. P0 Critical은 3건으로, CL플랫폼 G1 GC 전환과 Heap 증설, 현금정보계 Metaspace 정정, 그리고 전사 GC 로그 수집 활성화입니다..."

### D-02: 당사가 놓친 영역에 대한 질문

**Mermaid 다이어그램:** 4개 한계점(GC 로그 미수집, Peak 데이터 부재, 모니터링 전무, 부하테스트 이력 불명) -> 컨설팅사에 요청(누락 영역 식별, 모니터링 권장, 추가 데이터 제안). (상세 콘텐츠는 본 세션 출력 참조)
**한계점 상세 테이블:** 4개 항목의 현재 상태, 영향, 필요 조치.
**Discussion Point:** "우리가 놓치고 있거나 간과한 중요한 인프라 설정 항목이 있는지?"
**발표 스크립트:** "솔직하게 말씀드리면, 당사 분석에 한계가 있습니다. GC 로그 수집 여부가 불명확하고, Peak 데이터가 CL플랫폼팀 외에는 없으며, JVM 실시간 모니터링 체계가 구축되어 있지 않습니다..."

### D-03: 후속 워크숍 계획 및 Next Steps

**Mermaid 다이어그램:** 금일 미팅 -> 후속 워크숍(GC 로그 분석, Peak 데이터 수집, 모니터링 설계) -> 최종 산출물(표준 설정 가이드, 실행 계획, 모니터링 가이드). (상세 콘텐츠는 본 세션 출력 참조)
**후속 액션 아이템 테이블:** 당사 담당(데이터 수집 4건) + 컨설팅사 담당(산출물 3건) + 검증 1건. 목표 일정 명시.
**Discussion Point:** "컨설팅 측에서 다음 단계까지 준비해 주실 수 있는 산출물은?"
**발표 스크립트:** "후속 계획입니다. 당사에서는 GC 로그 활성화와 Peak 타임 데이터 수집을 즉시 시작하겠습니다. 컨설팅 측에는 P0 실행 가이드와 최소 모니터링 체계 권장안을 준비해 주시기를 요청드립니다..."

### E-01: 4개 팀 원천 데이터 요약

**플랫폼개발팀 테이블:** NP/NC 12개 설정 항목 비교 (WAS/Java/GC/Heap/Metaspace/Pool/Lifetime/Threads/DB). (상세 콘텐츠는 본 세션 출력 참조)
**CL플랫폼팀 테이블:** 13개 설정 항목. Old Gen 90.2% 수치 포함.
**주차서비스팀 테이블:** 13개 설정 항목. maxWaitMillis 연동 주석 포함.
**현금정보계팀 테이블:** 7개 컨테이너별 Heap/Metaspace 매핑 + 공통 설정 9개 항목.
**성격:** 참조용. 컨설팅사가 원천 데이터를 직접 확인할 수 있도록 제공.

### E-02: 벤더 권장사양 참조

**Tomcat 9.x 권장 테이블:** maxThreads 200, minSpare 25, acceptCount 100, connectionTimeout 20~60s, maxConnections 8192. (상세 콘텐츠는 본 세션 출력 참조)
**Liberty 23.x 권장 테이블:** coreThreads CPUs*2, maxThreads -1(자동), maxPoolSize 50, purgePolicy FailingConnectionOnly.
**HikariCP 권장 테이블:** maximumPoolSize 공식, minimumIdle 미설정 권장, maxLifetime < DB timeout.
**JVM GC/메모리 권장 테이블:** G1 GC 권장, Xms=Xmx, Old Gen 70~80% 경고, Metaspace Min 256m.
**MongoDB 권장 테이블:** Profiling Level 1, ESR 규칙, secondaryPreferred, w:majority.
**성격:** 참조용. 본 PPT의 분석 기준이 되는 벤더 공식 권장값 정리.

### E-03: 용어 정리

**용어 테이블:** GC, G1 GC, Parallel GC, STW, Full GC, Old Gen, Metaspace, Xms/Xmx, Connection Pool, maxLifetime, maxPoolSize, fixed-size pool, maxThreads, COLLSCAN, Replica Set, LTS, EOL, RTO/RPO, PgPool-II, Backpressure -- 총 20개 항목의 한국어 정의. (상세 콘텐츠는 본 세션 출력 참조)
**성격:** 참조용. 컨설팅사 미팅 시 기술 용어의 오해 방지.
