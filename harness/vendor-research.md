# 벤더별 리서치 로그 및 표준화 연구 기록

> 각 벤더의 최신 권장 튜닝 파라미터 리서치 결과와 전사 표준값 도출 기록.
> WAS / PostgreSQL / MongoDB 3개 도메인에 걸친 공통 참조 자료.

---

## 1. 리서치 대상 벤더 및 항목

| 벤더 | 제품 | 핵심 리서치 항목 | 상태 |
| :--- | :--- | :--- | :--- |
| Apache | Tomcat (9.x) | maxThreads, acceptCount, connectionTimeout 권장값 | 완료 |
| Pivotal/VMware | Spring Boot (내장 Tomcat) | server.tomcat.* 기본값 및 권장 튜닝 | 완료 |
| IBM | WebSphere Liberty (23.x) | executor, connectionManager, JVM 권장 설정 | 완료 |
| MongoDB | Replica Set | 인덱스 전략, COLLSCAN 방지, 모니터링 | 완료 |
| Oracle/OpenJDK | JVM (15/17/21) | G1 GC vs Parallel GC 권장, Heap 사이징 공식 | 완료 |
| Brett Wooldridge | HikariCP | maximumPoolSize, maxLifetime, connectionTimeout | 완료 |

---

## 2. 리서치 결과 상세

### 2.1 Apache Tomcat 9.x 권장 설정

| 파라미터 | 기본값 | 권장 범위 | 출처 |
| :--- | :--- | :--- | :--- |
| **maxThreads** | 200 | 200 ~ 500 (CPU 코어 수 * 8~20) | Tomcat 공식 config |
| **minSpareThreads** | 25 | 25 (기본값 유지 권장) | Tomcat 공식 config |
| **acceptCount** | 100 | 100 (기본값 유지) | Tomcat 공식 config |
| **connectionTimeout** | 60000 ms | 20000~60000 ms | Tomcat 공식 config |
| **maxConnections** | 8192 (NIO) | 8192 (기본값 유지) | Tomcat 공식 config |

**참고:** Tomcat은 특정 워크로드에 대한 "정답"을 제공하지 않음. 기본값이 대부분의 웹 애플리케이션에 적합하며, 튜닝은 부하 테스트 후 수행 권장.

### 2.2 Spring Boot 내장 Tomcat 권장 설정

| 파라미터 | 기본값 | 비고 |
| :--- | :--- | :--- |
| **server.tomcat.threads.max** | 200 | Tomcat maxThreads와 동일 |
| **server.tomcat.threads.min-spare** | 10 | 독립 Tomcat(25)보다 낮음 |
| **server.tomcat.max-connections** | 8192 | NIO 기본값 |
| **server.tomcat.accept-count** | 100 | 기본값 |
| **server.tomcat.connection-timeout** | 20000 ms (20초) | 독립 Tomcat(60초)보다 짧음 |

### 2.3 IBM WebSphere Liberty 23.x 권장 설정

**Thread Pool (Executor):**

| 파라미터 | 기본값 | 권장 사항 | 출처 |
| :--- | :--- | :--- | :--- |
| **coreThreads** | CPUs * 2 | 기본값 권장. 컨테이너 환경에서는 명시적 설정 고려 | Open Liberty 공식 |
| **maxThreads** | -1 (무제한) | 대부분 기본값 권장 (자동 튜닝) | Open Liberty 블로그 |
| **자동 튜닝** | 1.5초 간격 | Throughput 기반 동적 조정. 수동 튜닝 불권장 | IBM Performance Cookbook |

**Connection Manager:**

| 파라미터 | 기본값 | 권장 사항 | 출처 |
| :--- | :--- | :--- | :--- |
| **maxPoolSize** | 50 | coreThreads와 1:1 매핑으로 시작 | Open Liberty 공식 |
| **minPoolSize** | 0 | 성능 극대화 시 maxPoolSize와 동일하게 설정 | IBM Performance Cookbook |
| **connectionTimeout** | 30s | 기본값 유지 | Open Liberty 공식 |
| **maxIdleTime** | - | 성능 극대화 시 -1 (비활성화) | IBM Performance Cookbook |
| **reapTime** | 3m (180s) | 성능 극대화 시 -1. 단, 방화벽 환경에서는 유지 | IBM Performance Cookbook |
| **purgePolicy** | EntirePool | **FailingConnectionOnly** 권장 | Open Liberty 공식 |

### 2.4 HikariCP 권장 설정

| 파라미터 | 기본값 | 권장 사항 | 출처 |
| :--- | :--- | :--- | :--- |
| **maximumPoolSize** | 10 | `(core_count * 2) + effective_spindle_count` | HikariCP Wiki |
| **minimumIdle** | maximumPoolSize와 동일 | **미설정 권장** (fixed-size pool) | HikariCP README |
| **maxLifetime** | 1,800,000 ms (30분) | DB wait_timeout보다 **몇 초 짧게** | HikariCP README |
| **connectionTimeout** | 30,000 ms (30초) | 기본값 유지 | HikariCP README |

### 2.5 JVM GC 및 메모리 권장 설정

**GC 알고리즘 선택 기준:**

| GC 유형 | 적합 워크로드 | 대상 Heap | 최대 Pause | 권장 여부 |
| :--- | :--- | :--- | :--- | :--- |
| **G1 GC** | 범용 웹/앱 서버 | 4GB ~ 256GB | 50~200 ms | **권장 (Java 9+ 기본)** |
| **Parallel GC** | 배치/ETP/오프라인 | 제한 없음 | 수초 (Heap 비례) | **웹 서비스 비권장** |
| **ZGC** | 초저지연 실시간 | 8MB ~ 16TB | <1 ms (Java 16+) | Java 17+ 고급 옵션 |

**Heap 사이징 권장:**

| 항목 | 권장 사항 | 출처 |
| :--- | :--- | :--- |
| **Xms = Xmx** | 프로덕션에서 반드시 동일 설정 | JVM 튜닝 가이드 전반 |
| **Heap 크기 기준** | 컨테이너/VM 메모리의 **50~70%** | jarviix.com, heaphero.io |
| **Old Gen 경고** | **70~80%** 경고, 90% 이상 즉각 조치 | JVM 모니터링 권고 |
| **Metaspace Min** | 256m ~ 512m (대규모/Spring Boot) | 실무 권장 |
| **Metaspace Max** | Min의 **2배** 이상 또는 미설정(무제한) | 실무 권장 |

**Java LTS 권장:**

| 버전 | LTS 여부 | EOL | 권장 |
| :--- | :--- | :--- | :--- |
| **Java 15** | 비LTS | 2021-03 EOL | **표준화 대상에서 제외 필요** |
| **Java 17** | LTS | 2029-09 | **최소 표준** |
| **Java 21** | LTS | 2031-09 | **권장 표준** |
| **Java 25** | 비LTS | 2025-09 예상 | 비LTS, 장기 운영 비권장 |

### 2.6 MongoDB Replica Set 권장 설정

| 항목 | 권장 사항 | 출처 |
| :--- | :--- | :--- |
| **COLLSCAN 감지** | `db.setProfilingLevel(1, { slowms: 100 })` | MongoDB 공식 |
| **인덱스 전략** | ESR 규칙 (Equality, Sort, Range) | MongoDB 공식 |
| **Read Preference** | `secondaryPreferred` (Read 비중 높을 시) | MongoDB 공식 |
| **Write Concern** | `w: majority` (강한 일관성 필요 시) | MongoDB 공식 |
| **Connection Pool** | maxPoolSize=100, minPoolSize=0 | MongoDB 공식 |

---

## 3. 전사 표준 권장값 (Phase 2 도출)

| 영역 | 파라미터 | 표준 권장값 | 근거 |
| :--- | :--- | :--- | :--- |
| **Java 버전** | 최소 LTS | Java 17 | 비LTS(15) EOL, 보안 패치 미제공 |
| **Java 버전** | 권장 LTS | Java 21 | 최신 LTS, ZGC 프로덕션 레디 |
| **GC** | 표준 알고리즘 | G1 GC (`-XX:+UseG1GC`) | 웹 서비스 표준, Java 9+ 기본 |
| **Heap** | Xms/Xmx | 동일하게 설정 (고정) | 리사이즈 pause 방지 |
| **Heap** | 크기 기준 | 컨테이너 메모리 50~70% | off-heap, thread stack 고려 |
| **Metaspace** | Min | 256m | Spring Boot/대규모 앱 기준 |
| **Metaspace** | Max | 512m 이상 (또는 미설정=무제한) | Min의 2배 이상 |
| **Conn Pool** | maxLifetime | 1,620,000 ms (27분) | DB wait_timeout(30분)보다 짧게 |
| **Conn Pool** | connectionTimeout | 30,000 ms (30초) | HikariCP 기본값 |
| **Conn Pool** | minimumIdle | maximumPoolSize와 동일 | Fixed-size pool 권장 |
| **Thread** | maxThreads (Tomcat) | 200 (기본값) | 대부분의 워크로드에 적합 |
| **Thread** | Liberty Executor | 기본값 (자동 튜닝) | Liberty 공식 불권장 |
