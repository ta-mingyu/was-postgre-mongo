# 인프라 표준 설정 규정

> **배포**: IT기획실 → 전 사업팀
> **적용 범위**: WAS / PostgreSQL / MongoDB 프로덕션 환경 전반
> **기준 인프라**: 4 Core CPU / 4~32 GB RAM (Apache 튜닝 가이드 기준과 동일)
> **대상 플랫폼**: Apache Tomcat 9.x, Spring Boot 내장 Tomcat, IBM WebSphere Liberty 23.x, PostgreSQL, MongoDB
> **버전**: v4 (2026-07-02 갱신 — TA 결정 4건: backend_weight 1:3 / maintenance_work_mem 상한 0.0625 / num_init_children 120 유지 / work_mem 공식 *3 통일+매트릭스 표준화. 가독성 구조 통일. Spring Boot 3.0 Breaking Change 정정. Kernel 6.19 주의 추가)

---

## 1. 총칙

### 1.1 목적

본 규정은 전사 Web/WAS 및 Database 서버의 설정 파편화를 해소하고, 인프라 계층별 튜닝 파라미터의 표준값을 확정하여 각 팀에 배포하는 것을 목적으로 한다.

### 1.2 적용 원칙

1. 모든 프로덕션 WAS 및 DB 서버는 본 규정의 표준값을 적용해야 한다
2. 표준값 변경이 필요한 경우 IT기획실에 사전 승인을 요청해야 한다
3. 각 계층(WAS, DB)은 자원 비용 구조가 근본적으로 다르므로 독립적인 산정 기준을 적용한다
4. 동일한 선형 비율을 모든 계층에 일괄 적용하는 방식은 아키텍처 Anti-Pattern이므로 금지한다

### 1.3 도메인 구성

```
Web Server (Apache)     --->     WAS Server     --->     Database
MaxRequestWorkers                maxThreads                maxPoolSize
      |                             |                          |
      +--- 처리량 비례 감소 ------->+                          |
                                     +--- 자원 독립 산정 ----->+
```

---

## 2. WAS 서버 표준 설정

### 2.1 계층별 핵심 파라미터 산정 기준

| 파라미터 | 표준값 | 산정 공식 | 역할 |
|:---|:---|:---|:---|
| **maxThreads** | 200 (4 Core 기준) | `min(CPU_cores * 50, 500)` | WAS가 동시에 요청을 처리하는 작업 스레드(Worker Thread)의 최대 개수. 톰캣이 한 번에 수행할 수 있는 연산의 상한선을 정의하며, 이 값을 초과하는 요청은 `acceptCount` 대기 큐에서 대기함 |
| **minSpareThreads** | 25 (200 스레드 기준) | `maxThreads / 8` | 트래픽 유입 전 항상 대기 상태로 유지하는 최소 스레드 수. 요청이 도착하는 즉시 할당할 수 있는 예비 스레드를 확보하여 초기 응답 지연을 방지함 |
| **acceptCount** | 100 (고정) | -- | 모든 스레드가 처리 중일 때 OS TCP 레벨에서 임시로 대기시키는 최대 요청 수. 이 큐마저 포화되면 후속 요청은 즉시 거부(Connection Refused)되어 상위 프록시가 빠르게 실패를 인지하고 재시도 또는 차단(Backpressure)을 수행함 |
| **maxConnections** | 8,192 (고정) | -- | NIO 커넥터가 동시에 열어둘 수 있는 총 TCP 소켓의 상한선. 스레드 수보다 크게 설정하는 이유는 NIO가 Keep-Alive 상태의 유휴 소켓을 스레드 할당 없이 메모리에 대기시키기 때문임. 이 한계에 도달하면 후속 연결은 `acceptCount` 대기 큐로 넘어감 |
| **maxPoolSize (DB)** | 20 | `(DB_Cores * 2) + 여유량` | WAS 인스턴스가 DB와 맺는 상시 TCP 커넥션(HikariCP)의 최대 개수. **WAS 자원이 아닌 DB 서버 스펙 기준으로 독립 산정**함. DB 디스크 I/O 및 메모리 한계를 보호하기 위해 WAS 스레드 수와 무관하게 설정해야 하며, 과다 설정 시 DB Lock 경합 및 전체 TPS 저하를 유발함 |

### 2.2 인프라 스펙별 표준 설정값 매트릭스

#### 2.2.1 인스턴스별 설정 매트릭스

| 호스트 RAM | 인스턴스 수 | 인스턴스당 Heap (`Xms=Xmx`) | GC 전략 | 인스턴스당 maxPoolSize |
|:---|:---:|:---|:---|:---:|
| **4 GB** | 1 | **2,048m** | **Parallel GC** | 20 |
| **8 GB** | 2 | **2,560m** | **Parallel GC** | 20 |
| **16 GB** | 3 | **3,413m** | **Parallel GC** | 20 |
| **32 GB** | 4 | **5,120m** | **G1 GC** | 20 |

#### 2.2.2 Heap 분할 원칙

- **Heap 분할 공식**: `floor(호스트_RAM * 0.625) / 인스턴스_수`
- **예외**: 4 GB 단일 인스턴스 호스트는 OS 가용량 확보를 위해 보수적으로 **50%** (`2,048m`) 할당
- **Metaspace (모든 스펙 공통)**: Min `256m` / Max `512m` 고정 (역전 현상 및 입력 오류 방지)
- **검증**: 16 GB / 3인스턴스 = `floor(16,384m * 0.625) / 3 = floor(10,240m) / 3 = 3,413m`

### 2.3 GC 전략 결정 기준

| 기준 | Parallel GC | G1 GC | ZGC (Java 21+) |
|:---|:---|:---|:---|
| **적용 조건** | Heap ≤ 4 GB | Heap > 4 GB | Heap > 4 GB + Java 21+ |
| **적용 스펙** | 4 GB / 8 GB / 16 GB 호스트 | 32 GB 이상 호스트 | 32 GB 이상 + Java 21+ 환경 |
| **선택 사유** | 소형 힙에서 G1의 Region 메타데이터 오버헤드 및 Humongous Object 이슈를 회피하고 최고의 처리량(TPS)을 확보 | Heap 5 GB 이상에서 G1의 최소 Region 크기가 4 MB 이상 확보되어 Humongous Object 리스크가 감소하며, Heap 크기 대비 STW 시간 제어 능력이 실질적 이점을 제공 | 대규모 Heap에서 STW 시간을 1ms 이하로 제어 |

### 2.4 WAS 벤더별 적용 설정

#### 2.4.1 Apache Tomcat 9.x (독립형)

**`server.xml` — Connector 설정 (4 Core / 8 GB 호스트 / 인스턴스당 2,560m 기준)**

```xml
<Connector port="8080" protocol="org.apache.coyote.http11.Http11NioProtocol"
           maxThreads="200"
           minSpareThreads="25"
           acceptCount="100"
           maxConnections="8192"
           connectionTimeout="20000"
           keepAliveTimeout="15000"
           maxKeepAliveRequests="100"
           enableLookups="false"
           compression="on"
           compressionMinSize="2048" />
```

**`setenv.sh` — JVM 옵션 (8 GB 호스트 / 인스턴스당 Heap 2.5 GB 기준)**

```bash
JAVA_OPTS="-Xms2560m -Xmx2560m \
           -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m \
           -XX:+UseParallelGC \
           -XX:+HeapDumpOnOutOfMemoryError \
           -XX:HeapDumpPath=/var/log/tomcat/heapdump.hprof \
           -Xlog:gc*:file=/var/log/tomcat/gc.log:time,uptime,level,tags:filecount=10,filesize=50M"
```

#### 2.4.2 Spring Boot 내장 Tomcat

**`application.yml` (Spring Boot 3.x)**

```yaml
server:
  tomcat:
    threads:
      max: 200
      min-spare: 25
    max-connections: 8192
    accept-count: 100
    connection-timeout: 20s
    keep-alive-timeout: 15s
    max-keep-alive-requests: 100
```

**JVM 옵션 (8 GB 호스트 / 인스턴스당 Heap 2.5 GB 기준)**

```bash
JAVA_OPTS="-Xms2560m -Xmx2560m \
           -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m \
           -XX:+UseParallelGC \
           -XX:+HeapDumpOnOutOfMemoryError \
           -Xlog:gc*:file=/var/log/app/gc.log:time,uptime,level,tags:filecount=10,filesize=50M"
```

**하위 호환성 매핑**

본 절의 프로퍼티는 **Spring Boot 3.x 전용 규격**으로 작성되었음. 각 팀의 프레임워크 환경에 따라 아래 기준에 맞게 적용해야 함.

> **프레임워크별 적용 경로**:
> - **Spring Boot 3.x**: 본 규정의 `application.yml` 값을 그대로 적용
> - **Spring Boot 2.x**: 아래 매핑표에 따라 프로퍼티 키 및 값을 변환하여 적용
> - **Spring (Non-Boot) / 전자정부프레임워크**: `application.yml` 프로퍼티가 아닌 `server.xml` 또는 `TomcatServletWebServerFactory` Bean 설정으로 직접 제어해야 함. 2.4.1절(Apache Tomcat 9.x 독립형)의 `server.xml` Connector 설정을 참조할 것

> **Spring Boot 3.0에서 구 프로퍼티 키(`server.tomcat.max-threads` 등)가 제거되고 `threads.*` 네임스페이스로 단일화되는 파괴적 변경(Breaking Change)이 있었음.** 단, `threads.max`/`threads.min-spare`는 **Boot 2.0부터 이미 사용 가능**했으며, Boot 2.x에서는 구 키와 신 키가 deprecated 병존 상태였음. Boot 2.x 환경에서는 운영 중인 세부 버전을 확인할 것. (2026-07-02 정정 — 기존 "2.4 기점" 표기는 사실 오류)

| Boot 3.x (본 규정) | Boot 2.x (2.0~2.7) | Boot 1.x | 비고 |
|:---|:---|:---|:---|
| `server.tomcat.threads.max` | `server.tomcat.threads.max` | `server.tomcat.max-threads` | `threads.*` 네임스페이스는 Boot 2.0 도입. 구 키는 Boot 3.0에서 제거 |
| `server.tomcat.threads.min-spare` | `server.tomcat.threads.min-spare` | `server.tomcat.min-spare-threads` | 동일 |
| `server.tomcat.connection-timeout: 20s` | `20000` (ms 정수) | `20000` (ms 정수) | Boot 3.x는 Duration, Boot 2.x는 밀리초 정수 |
| `server.tomcat.keep-alive-timeout: 15s` | `15000` (ms 정수, **2.5.0+**) | Bean 오버라이드 필요 | 프로퍼티 지원은 **Boot 2.5.0**부터 (기존 "2.3.0" 정정) |
| `server.tomcat.max-keep-alive-requests: 100` | `100` (**2.4.0+**) | Bean 오버라이드 필요 | 프로퍼티 지원은 Boot 2.4.0부터 |

#### 2.4.3 IBM WebSphere Liberty 23.x

**`server.xml` (4 Core / 8 GB 호스트 / 인스턴스당 2,560m)**

```xml
<httpEndpoint id="defaultHttpEndpoint" host="*" httpPort="9080" httpsPort="9443">
  <httpOptions keepAliveTimeout="15s" />
</httpEndpoint>

<executor id="defaultExecutor" coreThreads="8" maxThreads="200" />

<connectionManager id="defaultConnectionManager"
                   maxPoolSize="20"
                   minPoolSize="20"
                   connectionTimeout="30s"
                   maxIdleTime="900s"
                   agedTimeout="1620s"
                   reapTime="60s"
                   purgePolicy="FailingConnectionOnly" />
```

**방화벽 임계치(TCP 30분 / 1,800초) 검증 결과 반영**

기존 `maxIdleTime="1800s"`는 방화벽의 TCP 유휴 타임아웃(1800s)과 정확히 일치하여, 방화벽이 커넥션을 Drop하는 시점과 Liberty가 커넥션을 정리하려는 시점 사이에 **Race Condition 구간**이 존재했음. Fixed-size pool (`minPoolSize=20` = `maxPoolSize=20`) 환경에서는 방화벽에 의해 무단 차단된 커넥션이 풀에 잔류하여 애플리케이션 오류를 유발할 수 있음.

| 파라미터 | 변경 전 | 변경 후 | 설계 근거 |
|:---|:---|:---|:---|
| `maxIdleTime` | 1800s | **900s** | 방화벽 타임아웃(1800s)의 50% 수준으로 단축. 유휴 커넥션을 방화벽 Drop 이전에 풀에서 안전하게 제거 |
| `agedTimeout` | (미설정) | **1620s** | 커넥션 최대 수명을 PG `idle_session_timeout`(1800s)보다 180초(3분) 짧게 설정하여 DB 서버 측 강제 차단 방지 |
| `reapTime` | 300s | **60s** | 풀 정리(Reaper) 주기를 60초로 단축하여 만료 커넥션을 신속히 회수 |

**`jvm.options` — JVM 옵션 (8 GB 호스트 / 인스턴스당 Heap 2.5 GB 기준)**

```text
-Xms2560m
-Xmx2560m
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m
-XX:+UseParallelGC
-XX:+HeapDumpOnOutOfMemoryError
-Xlog:gc*:file=/var/log/wlp/gc.log:time,uptime,level,tags:filecount=10,filesize=50M
```

**JVM 벤더(배포판)별 GC 옵션 호환성**

| 항목 | HotSpot (Temurin 등) | OpenJ9 (IBM Semeru) |
|:---|:---|:---|
| GC 정책 | `-XX:+UseParallelGC` | `-Xgcpolicy:gencon` (Gencon 권장) |
| GC 로그 | `-Xlog:gc*:file=...` | `-Xverbosegclog:/var/log/wlp/gc.log` |
| Heap Dump | `-XX:+HeapDumpOnOutOfMemoryError` | `-XX:+HeapDumpOnOutOfMemoryError` (호환) |

> 본 규정의 JVM 옵션은 **HotSpot 엔진**(Eclipse Temurin 등) 기준임. 가동 환경이 **IBM Semeru Runtime**(OpenJ9 엔진) 기반일 경우, HotSpot 전용 GC 옵션이 인식되지 않거나 무시되므로 해당 벤더 스펙에 맞는 GC 정책 옵션을 별도 확인 후 적용해야 함.

### 2.5 OS 커널 최적화 설정

서버 유형(WAS / PostgreSQL / MongoDB / PgPool-II)에 따라 **서로 다른 커널 튜닝이 필요**함. WAS는 JVM 기반 단기 커넥션 처리에 최적화되고, DB는 자체 버퍼 캐시를 통한 대용량 데이터 처리에 최적화되므로 스와핑, 메모리 오버커밋, 더티 페이지 정책 등에서 요구사항이 근본적으로 다름. 각 팀은 자신이 운영하는 **서버 유형에 해당하는 설정만** 적용한다.

#### 서버 유형별 OS 커널 파라미터 비교 매트릭스

| 파라미터 | WAS 서버 | PostgreSQL 서버 | MongoDB 서버 (8.0+) | PgPool-II 서버 | 기본값 | 비고 |
|:---|:---:|:---:|:---:|:---:|:---:|:---|
| **vm.swappiness** | `10` | `1` | `1` | `10` | `60` | 커널이 디스크 스왑을 시작하는 임계치. DB는 shared_buffers/WiredTiger 캐시가 디스크로 내려가면 성능 붕괴 → 거의 허용 안 함(1). WAS는 JVM Heap이 독립적이라 커널 캐시 영역 스왑이 비교적 무해 → 완화(10) |
| **vm.overcommit_memory** | `0` | `2` | `1` | `0` | `0` | PG는 fork 기반 백엔드 생성 시 OOM Killer가 postmaster 종료 → 인스턴스 전체 장애. `2`(엄격 모드)로 사전 거부(ENOMEM). Mongo 8.0은 TCMalloc per-CPU 캐시가 `1`(항상 허용) 요구. **둘이 충돌 → 물리 서버 분리 필수** |
| **vm.overcommit_ratio** | — | `90` | — | — | `50` | `overcommit_memory=2` 전용. 커밋 한도 = `Swap + (RAM × ratio%)`. 기본 50%는 16GB 서버 기준 8GB만 커밋 → PG shared_buffers + 백엔드 메모리 부족. 90%로 상향하여 10% 안전 마진 확보 |
| **Transparent Huge Pages** | enabled | **disabled** | **enabled** | enabled | enabled | PG OLTP는 희소 접근(sparse) 패턴 → THP compaction이 수백 ms 지연 스파이크 유발 → disabled. Mongo 8.0은 TCMalloc이 THP 활용 → enabled. **정반대 방향이므로 주의** |
| **vm.dirty_background_ratio** | `10` | `5` | `5` | `10` | `10` | 더티 페이지가 전체 메모리의 N% 도달 시 백그라운드 플러시 시작. DB는 WAL/체크포인트 쓰기가 한꺼번에 몰리는 I/O 버스트 방지를 위해 더 일찍 시작(5%). WAS/PgPool은 기본값 유지 |
| **vm.dirty_ratio** | `20` | `10` | `15` | `20` | `20` | 더티 페이지 N% 도달 시 **쓰기 프로세스 블로킹 + 동기 플러시 강제**. DB는 동기 플러시 = 쿼리 응답 지연 → 낮게 설정. Mongo는 WiredTiger 자체 스케줄링이 있어 PG보다 여유(15%) |
| **kernel.sem** | — | — | — | `250 32000 250 128` | `250 32000 32 128` | PgPool이 child 프로세스당 System V 세마포어 사용. SEMOPM(3번째 값) 32→250 상향: `num_init_children=120` 구동 시 한 번의 `semop()`으로 처리 가능한 연산 수를 늘려 세마포어 병목 해소 |
| **ip_local_port_range** | `32768~65535` | `32768~60999` | `32768~60999` | `32768~65535` | `32768~60999` | 아웃바운드 임시 포트 범위. WAS/PgPool은 대량 아웃바운드 커넥션 → 상한 65535로 확장 (약 4,500개 추가 가용 포트). DB는 인바운드 위주 → 기본값 유지. 시작점을 1024까지 내리지 않는 이유는 서비스 포트(8080, 5432, 27017 등)와 중첩 장애 방지 |
| **tcp_fin_timeout** | `15` | `60` | `60` | `15` | `60` | 연결 종료 후 FIN-WAIT-2 소켓 유지 시간. WAS/PgPool은 요청 1건당 커넥션 획득-반납하는 단기 커넥션 빈번 → TIME_WAIT 누적 방지를 위해 15초로 단축. DB는 장기 커넥션 위주 → 변경 불필요 |
| **tcp_tw_reuse** | `1` | `2` | `2` | `1` | `2` | TIME_WAIT 소켓 재사용 허용 범위. 기본=2는 loopback만. WAS/PgPool은 대량 아웃바운드 → global(1)로 확장하여 포트 고갈(Ephemeral Port Exhaustion) 방지. DB는 인바운드 위치 → 기본값 유지 |
| **tcp_keepalive_time** | `300` | `300` | `300` | `300` | `7200` | 최초 Keepalive 프로브 대기 시간. 기본 7,200초(2시간)는 죽은 커넥션을 너무 오래 방치 → 300초(5분)로 단축. 모든 서버 공통 적용 |
| **tcp_keepalive_intvl** | `30` | `30` | `30` | `30` | `75` | 프로브 재전송 간격 단축. 기본 75초 → 30초 |
| **tcp_keepalive_probes** | `5` | `5` | `5` | `5` | `9` | 연속 실패 시 dead 판정. **dead 확정 시간**: `300+30×5=450초(7.5분)` vs 기본 `7200+75×9=7,875초(2시간 11분)` |
| **tcp_max_syn_backlog** | `4,096` | `4,096` | `4,096` | `4,096` | `1,024` | SYN 수신 후 ACK 대기 중인 반개방(Half-Open) 연결의 SYN Queue 상한. **somaxconn과 반드시 동일하게 설정**: 커널이 `min(tcp_max_syn_backlog, somaxconn)`을 실제 크기로 사용하므로, 어느 한쪽이 작으면 그 값이 병목이 됨 |
| **somaxconn** | `4,096` | `4,096` | `4,096` | `4,096` | `4096` (kernel 5.4+) | 3-way handshake 완료 후 accept() 대기열(Accept Queue) 상한. 트래픽 스파이크 시 OS 관문에서 패킷 Drop 방지. kernel 5.4+ 기본값과 동일 |
| **fs.file-max** | `2,097,152` | `2,097,152` | `2,097,152` | `2,097,152` | 메모리 비례 자동 산정 | 시스템 전체 파일 디스크립터 상한. 모든 소켓 통신이 파일로 취급되어 대규모 동시 접속 시 Too many open files 방지. 미사용 시 메모리 비용이 0이므로 공통 높게 설정 |
| **ulimit -n** | `1,048,576` | `1,048,576` | `1,048,576` | `1,048,576` | `1,024` | 프로세스당 FD 한도. 모든 서버 1M 통일. `infinity` 설정 시 커널이 FD 테이블용 ~8.6GB 할당하여 시스템 불안정 유발 (Red Hat Kernel Bug 2394600) |
| **ulimit -u** | `65,536` | `65,536` | `65,536` | `65,536` | `4,096` | 프로세스/스레드 수 상한. `unlimited` 시 Fork Bomb에 무방비. JVM 스레드당 ~1MB 스택을 고려하면 물리적 한계가 자연스러운 절대 상한 역할 |
| **ulimit -f / -t** | unlimited | unlimited | `unlimited` | unlimited | `unlimited` | 파일 크기 및 CPU 시간 제한. Mongo는 대용량 데이터 처리 중 파일 크기 제한 도달 시 데이터 손상 위험으로 공식 명시적 설정 권장 |
| **vm.min_free_kbytes** | — | `102400` | — | — | 자동 산정 (`4×√(RAM_KB)`) | 커널이 항상 확보하는 최소 여유 RAM. PG 대량 정렬/해시조인 시 메모리 급할 때 Direct Reclaim(전체 프로세스 일시정지) 방지. 100MB 예약 |
| **vm.zone_reclaim_mode** | — | `0` | — | — | `0` | NUMA 환경에서 다른 노드 메모리 회수 금지. PG는 여러 백엔드가 shared_buffers 공유 접근 → NUMA 노드 간 회수 시 성능 급감. 기본값과 동일하나 설정 누락 방지를 위해 명시적 적용 |

> **PostgreSQL과 MongoDB는 동일 호스트 병설 금지**: `vm.overcommit_memory` 설정이 PostgreSQL(`2`)과 MongoDB 8.0(`1`)에서 서로 충돌하므로, PostgreSQL과 MongoDB는 **반드시 물리적으로 분리된 서버**에서 운영해야 함.

---

#### 2.5.1 공통 파라미터 (모든 서버 유형에 적용)

```ini
# /etc/sysctl.d/99-infra-common.conf -- 모든 서버 공통
fs.file-max = 2097152
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 5
```

```bash
# /etc/security/limits.d/99-infra-common.conf -- 모든 서버 공통 (PAM 기반 적용)
# 또는 기동 스크립트에 적용
*  soft  nofile  1048576
*  hard  nofile  1048576
*  soft  nproc   65536
*  hard  nproc   65536
```

> **systemd 서비스 필수 추가 설정**: 위 limits.conf는 PAM 기반 세션 접속에만 적용되며, **systemd가 관리하는 서비스 데몬(tomcat, postgresql, pgpool, mongod 등)에는 적용되지 않음** (Red Hat 공식: "Limits set in limits.conf are ignored by systemd"). 각 서비스별로 아래와 같이 systemd drop-in override 파일을 추가로 생성해야 함.
>
> ```ini
> # /etc/systemd/system/<service-name>.service.d/override.conf
> [Service]
> LimitNOFILE=1048576
> LimitNPROC=65536
> # MongoDB 서버의 경우 추가:
> # LimitFSIZE=infinity
> # LimitCPU=infinity
> ```
>
> ```bash
> systemctl daemon-reload
> systemctl restart <service-name>
> ```

| 파라미터 | 값 | 역할 |
|:---|:---|:---|
| **fs.file-max** | `2,097,152` | 시스템 전체 파일 디스크립터 상한. 모든 소켓 통신이 파일로 취급되므로, 대규모 동시 접속 시 **Too many open files** 에러 및 인스턴스 다운을 방지함. 미사용 시 메모리 비용이 0이므로 높게 설정해도 무해함 |
| **net.core.somaxconn** | `4,096` | OS 커널 레벨의 TCP Listen Backlog 큐 크기. 트래픽 스파이크 시 서버 엔진에 도달하기 전 OS 관문에서 패킷이 Drop되는 현상을 방지함. WAS는 `min(acceptCount, somaxconn)`, DB는 `min(backlog, somaxconn)`으로 실제 Backlog가 결정됨 |
| **net.ipv4.tcp_max_syn_backlog** | `4,096` | SYN 수신 후 ACK 대기 중인 반개방(Half-Open) 연결의 최대 대기열. `somaxconn`과 세트로 설정해야 하며, 이 값이 `somaxconn`보다 작으면 `somaxconn`을 올려도 SYN 단계에서 패킷이 Drop됨 |
| **net.ipv4.tcp_keepalive_time** | `300` (5분) | OS 레벨 TCP Keepalive 최초 대기 시간(초). 기본값 7,200초(2시간)는 죽은 커넥션을 너무 오래 방치함. WAS→DB 간 죽은 커넥션 조기 감지 |
| **net.ipv4.tcp_keepalive_intvl** | `30` | Keepalive 프로브 재전송 간격(초). 기본값 75초를 30초로 단축 |
| **net.ipv4.tcp_keepalive_probes** | `5` | Keepalive 연속 실패 시 dead 판정. `30초 × 5 = 150초` 내에 죽은 커넥션 확정 정리 |
| **ulimit -n** (nofile) | `1,048,576` | 프로세스당 열 수 있는 파일(소켓) 수 상한. 커널 `fs.nr_open` 기본 hard limit과 동일한 값으로, K8s/LXD 등 프로덕션 환경 표준값. `infinity` 또는 `2^30`(1,073,741,816) 이상으로 설정할 경우, 프로세스가 높은 FD 번호를 사용할 때 **커널이 FD 테이블용으로 ~8.6GB 메모리를 할당**하여 할당 실패 및 시스템 불안정을 유발함 (Red Hat Kernel Bug 2394600). `1,048,576`(1M) 설정 시 커널 FD 테이블은 약 8MB에 불과하여 안전함. 단, JDK 8u252 미만 구버전에서는 JVM 자체 FD 추적 배열(~50 bytes/FD)이 추가로 ~50MB 소요될 수 있으나 최신 JDK에서는 sparse array 방식으로 해결됨 (OpenJDK JDK-8150460) |
| **ulimit -u** (nproc) | `65,536` | 프로세스/스레드 수 상한. `unlimited` 설정 시 Fork Bomb에 무방비. 65,536은 실무 표준값으로, JVM 스레드당 ~1MB 스택을 고려하면 물리적 한계가 자연스럽게 절대 상한 역할 |

---

#### 2.5.2 WAS 서버 전용 파라미터

```ini
# /etc/sysctl.d/99-was-tuning.conf -- WAS 서버 전용
vm.swappiness = 10
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_tw_reuse = 1
net.ipv4.ip_local_port_range = 32768 65535
```

| 파라미터 | 값 | 역할 |
|:---|:---|:---|
| **vm.swappiness** | `10` | JVM 환경에서 물리 메모리 고갈 전 디스크 스와핑으로 인한 GC 성능 마비 및 STW 리스크를 최소화. 기본값(60)은 JVM Heap과 격돌하여 빈번한 GC Pause 유발. DB 서버(1)보다 높은 값을 허용하는 이유는 JVM이 OS 페이지 캐시에 의존하지 않고 자체 Heap으로 메모리를 관리하므로, 커널 캐시 영역은 스왑 아웃해도 JVM 동작에 직접 영향이 없기 때문 |
| **net.ipv4.tcp_fin_timeout** | `15` | 연결 종료 후 소켓이 TIME_WAIT 상태로 머무는 시간(초) 단축. WAS는 요청 1건당 DB 커넥션을 획득-반납하는 단기 커넥션 패턴이 빈번하므로 TIME_WAIT 소켓이 급격히 누적될 수 있음 |
| **net.ipv4.tcp_tw_reuse** | `1` | TIME_WAIT 상태의 소켓을 안전하게 재사용하도록 허용. 대량 커넥션 요청 시 로컬 포트 고갈(Ephemeral Port Exhaustion) 차단 |
| **net.ipv4.ip_local_port_range** | `32768~65535` | WAS→DB 등 아웃바운드 연결 시 커널이 자동 할당하는 임시 포트 범위. 기본값 상한(60999)에서 65535로 확장하여 약 33,000개 가용 포트 확보. 시작점을 1024까지 내리지 않는 이유는 서비스 포트(8080, 5432, 27017 등)와 중첩 시 `Address already in use` 장애가 간헐적으로 발생하기 때문 |

> **somaxconn ↔ acceptCount 연동**: 실제 Listen Backlog는 `min(acceptCount, somaxconn)`으로 결정됨. WAS 기본 `acceptCount`(100)에서는 somaxconn(4,096)이 더 크므로 `acceptCount`가 실제 Backlog가 됨. `acceptCount`를 500~1,000으로 상향 운영하는 환경에서도 somaxconn 4,096이 충분한 상한을 제공함.

---

#### 2.5.3 PostgreSQL 서버 전용 파라미터

```ini
# /etc/sysctl.d/99-postgresql-tuning.conf -- PostgreSQL 서버 전용
vm.swappiness = 1
vm.overcommit_memory = 2
vm.overcommit_ratio = 90
vm.dirty_background_ratio = 5
vm.dirty_ratio = 10
vm.min_free_kbytes = 102400
vm.zone_reclaim_mode = 0
```

```bash
# THP (Transparent Huge Pages) 비활성화 -- OS 리부팅 시 초기화되는 1회성 명령임.
# 영구 설정은 root 권한이 필요하므로 IT ONE을 통해 IT 운영실에 변경 요청할 것.
#
# [참고: 영구 설정 방법 -- IT 운영실 적용용]
# 방법 1 (권장): GRUB 커널 파라미터 (리부팅 필요)
#   grubby --update-kernel=ALL --args="transparent_hugepage=never"
#
# 방법 2: TuneD 프로파일
#   /etc/tuned/<profile>/tuned.conf 에 [vm] transparent_hugepages=never 설정
```

| 파라미터 | 값 | 역할 |
|:---|:---|:---|
| **vm.swappiness** | `1` | PostgreSQL은 `shared_buffers`로 자체 버퍼 캐시를 관리하며, 커널 페이지 캐시와 이중 캐싱됨. 스와핑이 발생하면 `shared_buffers`의 데이터가 디스크로 내려가 전체 쿼리 성능이 급격히 저하됨. JVM(10)과 달리 거의 스왑을 허용하지 않아야 함 |
| **vm.overcommit_memory** | `2` | 메모리 오버커밋을 엄격 모드로 설정. PostgreSQL은 fork 기반으로 백엔드 프로세스를 생성하므로, 기본값(0)에서는 OOM Killer가 postmaster를 죽여 **전체 인스턴스 장애**를 유발할 수 있음. `2`로 설정하면 물리 메모리 및 Swap 범위를 초과하는 과도한 오버커밋 할당 요청을 커널 단에서 사전에 거부(ENOMEM 반환)함으로써, OOM Killer가 데이터베이스 메인 프로세스(postmaster)를 임의로 강제 종료하여 전체 인스턴스 장애로 전파되는 리스크를 실질적으로 방지함 (PostgreSQL 공식 권장) |
| **vm.overcommit_ratio** | `90` | `overcommit_memory=2` 모드에서 **커밋 가능한 총 메모리 한도 비율**. `Committable = Swap + (Physical RAM × ratio%)`. 기본값 50%는 현대 서버에 너무 낮아, 예를 들어 16GB 서버의 경우 8GB까지만 커밋 가능 → PostgreSQL `shared_buffers`(4GB) + 백엔드 프로세스 메모리가 한도 초과로 "Cannot allocate memory" 장애 발생. 90%로 상향하여 물리 RAM의 90%까지 커밋을 허용하면, 모든 RAM 스펙(8/16/32/64GB)에서 PostgreSQL 최대 메모리 요구량을 수용하면서도 10% 안전 마진 확보 (PostgreSQL 공식 문서 권장: "You might also wish to modify the related setting vm.overcommit_ratio") |
| **Transparent Huge Pages** | **disabled** (`never`) | PostgreSQL의 OLTP 접근 패턴은 희소(sparse)하여 THP의 연속 메모리 할당 방식과 맞지 않음. THP 활성화 시 백그라운드 compaction에 의해 **수백 ms 단위의 지연 스파이크**가 간헐적으로 발생하여 쿼리 응답 시간 편차가 커짐. MongoDB 8.0과 **정반대 방향**이므로 주의 |
| **vm.dirty_background_ratio** | `5` | 더티 페이지(수정 후 디스크에 아직 기록되지 않은 페이지)가 전체 메모리의 5%에 도달하면 백그라운드 플러시 시작. 기본값(10)보다 낮추어 WAL 및 체크포인트 쓰기 부하를 시간에 걸쳐 분산시킴. 그렇지 않으면 더티 페이지가 한꺼번에 플러시되어 I/O 버스트 발생 |
| **vm.dirty_ratio** | `10` | 더티 페이지가 전체 메모리의 10%에 도달하면 쓰기 프로세스를 블로킹하고 동기 플러시 강제. `dirty_background_ratio`(5)보다 높게 설정해야 백그라운드 플러시가 먼저 개입할 여유가 있음. 기본값(20~30)은 DB 쓰기 부하에서 I/O 마비 유발 |
| **vm.min_free_kbytes** | `102400` (100MB) | 커널이 항상 확보하는 최소 여유 메모리. PostgreSQL이 대량 정렬(sort)이나 해시 조인으로 메모리를 급격히 할당할 때, 여유 메모리가 부족하면 커널이 모든 프로세스를 일시 정지시키며 메모리를 회수하는 Direct Reclaim이 발생함. 100MB를 예약해두어 이를 방지 |
| **vm.zone_reclaim_mode** | `0` | NUMA 아키텍처에서 다른 NUMA 노드의 메모리를 회수하지 않도록 설정. PostgreSQL은 여러 백엔드 프로세스가 공유 메모리(shared_buffers)에 접근하므로, NUMA 노드 간 메모리 회수가 발생하면 성능이 급격히 저하됨 |

---

#### 2.5.4 MongoDB 서버 전용 파라미터 (8.0+ 기준)

```ini
# /etc/sysctl.d/99-mongodb-tuning.conf -- MongoDB 서버 전용
vm.swappiness = 1
vm.overcommit_memory = 1
vm.dirty_background_ratio = 5
vm.dirty_ratio = 15
```

> **Kernel 6.19 주의**: MongoDB 8.0.4 미만 버전에서 Linux Kernel 6.19 구동 시 알려진 오류가 있음. **MongoDB 8.0.4 이상 사용 권장**(공식 문서 확인). 커널 6.19 환경에서 8.0.4 미만 사용 금지.

```bash
# THP (Transparent Huge Pages) 활성화 -- OS 리부팅 시 초기화되는 1회성 명령임.
# 영구 설정은 root 권한이 필요하므로 IT ONE을 통해 IT 운영실에 변경 요청할 것.
#
# [참고: 영구 설정 방법 -- IT 운영실 적용용]
# 방법 1 (권장): GRUB 커널 파라미터 (리부팅 필요)
#   grubby --update-kernel=ALL --args="transparent_hugepage=always"
#
# 방법 2: TuneD 프로파일
#   /etc/tuned/<profile>/tuned.conf 에 [vm] transparent_hugepages=always 설정
```

```bash
# /etc/security/limits.d/99-mongodb.conf -- MongoDB ulimit
mongod  soft  nofile   1048576
mongod  hard  nofile   1048576
mongod  soft  nproc    65536
mongod  hard  nproc    65536
mongod  soft  fsize    unlimited
mongod  hard  fsize    unlimited
mongod  soft  cpu      unlimited
mongod  hard  cpu      unlimited
```

| 파라미터 | 값 | 역할 |
|:---|:---|:---|
| **vm.swappiness** | `1` | MongoDB는 WiredTiger 스토리지 엔진이 내부 캐시(cacheSizeGB)를 자체 관리함. 스와핑이 발생하면 캐시 페이지가 디스크로 내려가 전체 쿼리 성능이 급감. MongoDB 공식 권장값 (MongoDB Production Notes) |
| **vm.overcommit_memory** | `1` | MongoDB 8.0의 TCMalloc per-CPU 캐시가 정상 동작하려면 메모리 오버커밋을 항상 허용해야 함. PostgreSQL(`2`)과 **충돌**하므로 동일 호스트 병설 불가 |
| **Transparent Huge Pages** | **enabled** (`always`) | MongoDB 8.0부터 TCMalloc per-CPU 캐시가 THP를 활용하여 성능을 향상시킴. 7.0 이하에서는 비활성화가 권장이었으나, **8.0부터는 방향이 전환**되어 활성화가 필수. PostgreSQL(disabled)과 **정반대** (MongoDB 8.2 TCMalloc Performance 공식 문서) |
| **vm.dirty_background_ratio** | `5` | 더티 페이지가 5% 도달 시 백그라운드 플러시 시작. 기본값보다 낮추어 WiredTiger의 체크포인트와 커널 플러시가 충돌하는 I/O 버스트를 완화 |
| **vm.dirty_ratio** | `15` | 동기 플러시 임계치. PostgreSQL(10)보다 높게 설정하는 이유는 WiredTiger가 자체적으로 쓰기 스케줄링을 수행하므로, 커널의 동기 플러시 개입을 조금 더 늦추어 I/O 패턴을 안정화 |
| **ulimit -n** (nofile) | `1,048,576` | 모든 서버 공통값 (MongoDB 공식 최소 64,000 이상 충족) |
| **ulimit -f / -t** | `unlimited` | 파일 크기 및 CPU 시간 제한 해제. 대용량 데이터 처리 중 파일 크기 제한 도달 시 데이터 손상 위험 (MongoDB 공식 ulimit 권장) |

---

#### 2.5.5 PgPool-II 서버 전용 파라미터

```ini
# /etc/sysctl.d/99-pgpool-tuning.conf -- PgPool-II 서버 전용
vm.swappiness = 10
kernel.sem = 250 32000 250 128
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_tw_reuse = 1
net.ipv4.ip_local_port_range = 32768 65535
```

| 파라미터 | 값 | 역할 |
|:---|:---|:---|
| **kernel.sem** | `250 32000 250 128` | PgPool-II가 child 프로세스당 System V 세마포어를 사용함. `num_init_children = 120` 구동 시 필요한 세마포어 세트를 수용하기 위한 최소 설정. 값이 부족하면 PgPool 기동 시 "could not create semaphore set" 에러 발생. 형식: `SEMMSL SEMMNS SEMOPM SEMMNI` (PgPool-II 공식 문서) |
| **vm.swappiness** | `10` | PgPool은 WAS와 유사한 네트워크 프록시 역할이므로 JVM과 동일한 수준 적용 |
| **tcp_fin_timeout** | `15` | PgPool→PostgreSQL 연결 종료 후 TIME_WAIT 소켓 신속 정리 |
| **tcp_tw_reuse** | `1` | TIME_WAIT 소켓 재사용으로 포트 고갈 방지 |
| **ip_local_port_range** | `32768~65535` | PgPool이 WAS로부터 커넥션을 받아 PostgreSQL로 아웃바운드 연결을 생성하므로 WAS와 동일하게 적용 |

> **PgPool-II + PostgreSQL 병설 서버**: PgPool-II와 PostgreSQL이 동일 호스트에 구성된 경우, 양쪽 파라미터를 **모두 적용**해야 함. 단, `vm.swappiness`는 충돌하므로 DB 서버 기준인 `1`을 우선 적용.
>
> **PgPool-II 독립 서버 (4GB RAM)**: `num_init_children = 120` 구동 시 프로세스 메모리 점유율(약 1GB 내외)은 안정 범위이나, 반드시 `kernel.sem` 설정이 선행되어야 함.

---

## 3. Database 서버 표준 설정

### 3.1 PostgreSQL — PgPool-II + Streaming Replication (프로덕션 표준)

중~대규모 상용 서비스, 다수 WAS 인스턴스가 DB를 공유하는 환경의 **프로덕션 표준 아키텍처**임. 읽기/쓰기 분산, 커넥션 풀링, 자동 페일오버(RTO 10~30s)를 제공함.

#### 아키텍처

```mermaid
graph LR
    subgraph WAS Layer
        W1[WAS-1<br/>HikariCP]
        W2[WAS-2<br/>HikariCP]
        W3[WAS-N<br/>HikariCP]
    end

    subgraph PgPool Layer
        PP1[PgPool-II<br/>Active]
        PP2[PgPool-II<br/>Standby]
    end

    subgraph PostgreSQL Layer
        PG_M[(PostgreSQL<br/>Primary)]
        PG_R[(PostgreSQL<br/>Replica)]
    end

    W1 --> VIP[Virtual IP]
    W2 --> VIP
    W3 --> VIP
    VIP --> PP1
    PP1 -->|Write / Read| PG_M
    PP1 -->|Read Only<br/>Load Balance| PG_R
    PP1 -.->|Watchdog<br/>SPOF 방지| PP2
    PG_M -->|WAL Streaming| PG_R

    style PP1 fill:#d4edda,stroke:#28a745
    style PP2 fill:#fff3cd,stroke:#856404
    style PG_M fill:#cce5ff,stroke:#004085
    style PG_R fill:#cce5ff,stroke:#004085
    style VIP fill:#f8d7da,stroke:#721c24
```

#### PostgreSQL 핵심 파라미터

| 파라미터 | Primary | Replica | 역할 |
|:---|:---:|:---:|:---|
| **wal_level** | `replica` | (상속) | WAL(Write-Ahead Log)에 기록할 정보의 수준을 정의함. `replica`는 스트리밍 복제에 필요한 충분한 정보를 WAL에 포함시키는 설정으로, 복제 구성에서 필수임. `minimal`로 설정하면 복제가 불가능해짐. Replica는 Primary의 wal_level 설정을 상속받음 |
| **max_wal_senders** | `5` | `5` | WAL 스트리밍 연결(Replication Connection)의 최대 허용 개수. 장애 시 Replica가 Primary로 승격(Promote)한 후 다운스트림 복제본 수용 및 백업 연결을 즉시 허용해야 하므로 Primary와 동일하게 설정함 |
| **hot_standby** | (해당 없음) | `on` | Replica 노드에서 읽기 쿼리를 수용할지 여부. PgPool-II의 읽기 분산(Load Balance) 기능이 동작하려면 반드시 `on`이어야 함 |
| **hot_standby_feedback** | (해당 없음) | `on` | Replica 읽기 분산 시 Primary의 Vacuum 작업으로 인한 쿼리 취소(Conflict) 방지. 이 설정이 없으면 Replica에서 실행 중인 긴 SELECT 쿼리가 Primary의 VACUUM에 의해 강제 취소될 수 있음 |
| **archive_mode** | `always` | `always` | WAL 아카이빙 활성화. `always` 설정은 Primary뿐만 아니라 **Replica(Standby) 상태에서도 아카이빙을 수행**함. 이를 통해 Replica 노드에서 직접 백업을 수행하거나, 계층형(Cascaded) 복제 구성에서 하위 Replica에게 WAL을 제공할 수 있음. 또한 Primary 아카이브에 결누락이 발생하더라도 Replica의 독립 아카이브로 복구 가능. 참고: `on` 설정에서도 승격(Promote) 후에는 아카이빙이 정상 작동하지만, Standby 상태에서는 아카이빙이 수행되지 않아 WAL 공백이 발생할 수 있음 (PostgreSQL 18 공식 문서) |
| **max_connections** | `100` | `100` | PostgreSQL이 수용할 수 있는 최대 동시 클라이언트 연결 수. OOM 예방을 위해 100으로 엄격히 제한함. PgPool-II의 `num_init_children`(120)이 `max_connections`(100)을 초과하더라도, PgPool-II의 연결 풀링(커넥션 캐싱 및 재사용)으로 인해 실제 동시 백엔드 연결은 100 이하로 유지됨 (자세한 내용은 PgPool-II 전용 파라미터 참조) |

#### PgPool-II 전용 파라미터

| 파라미터 | 표준값 | 역할 |
|:---|:---|:---|
| **num_init_children** | `120` | PgPool이 수용할 동시 클라이언트 연결(프로세스) 수. WAS로부터의 연결을 120개까지 수용하되, PgPool-II의 연결 풀링(커넥션 캐싱 및 재사용)으로 인해 실제 PostgreSQL 백엔드 동시 연결은 100 이하로 유지됨. 클라이언트가 연결을 해제하면 해당 child 프로세스의 백엔드 연결이 캐시되어 다음 클라이언트가 재사용하므로, 120개 프로세스가 모두 동시에 백엔드 연결을 점유하지 않음. 120개를 초과하는 클라이언트 연결 요청은 PgPool Listen Queue(kernel backlog)에서 대기하다가 child 프로세스가 해제되면 순차적으로 처리됨.<br><br>**⚠️ 최악 시나리오 경고**: PgPool-II 공식 문서는 `max_pool × num_init_children ≤ (max_connections - superuser_reserved_connections)` 공식을 권고함. `num_init_children=120`은 이 공식을 초과(1×120 > 97)하므로, 120개 child 프로세스가 **동시에** 백엔드 연결을 요구하는 극단적 피크 상황에서는 PostgreSQL이 `"FATAL: sorry, too many clients already"` 오류를 반환하고 failover가 트리거될 수 있음. 단, 실제 운영 환경에서는 WAS 커넥션 풀의 특성상 120개 연결이 모두 동시에 활성화되는 경우는 드물며, PgPool의 연결 캐싱으로 백엔드 연결이 재사용되므로 실무적으로 안정적으로 운영됨. **주기적인 피크 타임 백엔드 연결 수 모니터링(`SHOW POOL_PROCESSES`)이 필수** |
| **max_pool** | 단일 DB/계정: `1` | 하나의 child 프로세스가 유지할 수 있는 DB 연결 수. 불필요한 상향 시 백엔드 연결(`num_init_children * max_pool`)이 기하급수적으로 증가하여 PgPool 서버 메모리 고갈을 초래함 |
| **child_life_time** | `1,680` (28min) | PgPool child 프로세스의 최대 생존 시간(초). DB `idle_session_timeout`(30min)보다 짧게 설정하여 DB 측 강제 종료 전에 PgPool이 먼저 프로세스를 회수하도록 타임아웃 캐스케이드를 유지함 |
| **connection_life_time** | `1,680` (28min) | PgPool → PostgreSQL 백엔드 연결의 최대 수명(초). DB 세션 타임아웃(30min)보다 짧게 설정하여 DB 또는 방화벽에 의한 강제 차단 전에 연결을 안전하게 갱신함 |
| **client_idle_limit** | `600` (10min) | 클라이언트(WAS)가 아무런 요청 없이 유휴 상태로 머무는 최대 시간(초). 이 시간을 초과하면 PgPool이 해당 클라이언트 연결을 강제 종료하여 좀비 커넥션이 PgPool 프로세스를 점유하는 것을 방지함 |
| **reserved_connections** | `1` | PgPool 관리자 접속을 위한 예약 슬롯. 장애 발생 시 DBA가 PgPool에 접속할 수 없는 상황을 방지함. `num_init_children`에 포함되지 않는 별도 예약 공간임 |
| **load_balance_mode** | `on` | 읽기 쿼리(SELECT)를 Replica 노드로 자동 분산하는 기능. PgPool이 SQL을 분석하여 SELECT 문은 `backend_weight` 비율에 따라 분산 라우팅하고, 그 외(INSERT/UPDATE/DELETE)는 항상 Primary로 라우팅함 |
| **backend_clustering_mode** | `'streaming_replication'` | PgPool-II v4.x+의 스트리밍 복제 모드 설정. 기존 `master_slave_mode`는 폐지되었으므로 v4.x 이상에서는 반드시 이 값을 사용해야 함 |
| **backend_weight0 / weight1** | Primary `1` / Replica `3` | 읽기 쿼리 분산 비율. Primary는 모든 쓰기 트랜잭션(INSERT/UPDATE/DELETE) 및 MVCC 가비지 관리(VACUUM) 부하를 전담하므로, 읽기 부하까지 동등하게 분배(1:1)하면 Primary의 자원이 과부하 상태가 될 수 있음. Primary의 가중치를 낮추고 Replica에 읽기 부하를 집중시키는 차등 구성(1:3)으로 Primary의 자원을 쓰기 트랜잭션 격리 및 정합성 보장에 집중시킴. 비율 산출 근거: Primary 1 : Replica 3 = SELECT 쿼리의 약 25%는 Primary, 약 75%는 Replica로 분산됨 |

> **[4GB RAM 독립 서버 주의]** PgPool-II 전용 서버가 4GB RAM인 경우, `num_init_children = 120` 구동 시 프로세스 메모리 점유율(약 1GB 내외)은 안정 범위이나, OS 커널 세마포어 상한선 설정이 필수임. PgPool-II 서버 전용 커널 파라미터는 **2.5.5절**을 참조.
>
> 향후 복수 DB/계정 매핑으로 `max_pool`이 2 이상으로 증가할 경우, 백엔드 최대 연결 수(`num_init_children * max_pool`)가 기하급수적으로 늘어나 PgPool 서버 메모리 고갈을 초래할 수 있음. 멀티 DB 환경 확장 시에는 `num_init_children` 하향 조정 또는 PgPool 서버 RAM 증설이 선행되어야 함.

#### RAM별 파라미터 매트릭스 (PostgreSQL 노드)

| DB 서버 RAM | shared_buffers | effective_cache_size | work_mem | maintenance_work_mem | wal_buffers | max_wal_size | max_wal_senders | max_connections |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **8 GB** | 2 GB | 6 GB | 8 MB | 384 MB | 16 MB | 2 GB | 5 | 100 |
| **16 GB** | 4 GB | 12 GB | 16 MB | 1 GB | 16 MB | 4 GB | 5 | 100 |
| **32 GB** | 8 GB | 24 GB | 32 MB | 2 GB | 16 MB | 16 GB | 5 | 100 |
| **64 GB** | 16 GB | 48 GB | 64 MB | 4 GB | 16 MB | 32 GB | 5 | 100 |

#### 실무 설정 스크립트

**PostgreSQL (`postgresql.conf`):**

```conf
# -------------------------------------------------------
# Memory (8GB DB 전용 서버 기준)
# -------------------------------------------------------
shared_buffers = 2GB                # RAM * 0.25
effective_cache_size = 6GB          # RAM * 0.75
work_mem = 8MB                     # 운영 표준값 (이론 상한: (RAM-shared_buffers)/(max_conn*3), kofemann/pgtune)
maintenance_work_mem = 384MB        # RAM * 0.047~0.0625 (PGTune 기준)
wal_buffers = 16MB                  # 고정

# -------------------------------------------------------
# WAL & Checkpoint
# -------------------------------------------------------
wal_level = replica
max_wal_size = 2GB
min_wal_size = 1GB
checkpoint_completion_target = 0.9
max_wal_senders = 5                 # Replica + 여유

# -------------------------------------------------------
# Connections
# -------------------------------------------------------
max_connections = 100               # OOM 예방 100 고정, PgPool 연결 풀링으로 백엔드 연결 제어
superuser_reserved_connections = 3
hot_standby = on
hot_standby_feedback = on           # Replica: Vacuum 충돌 방지
archive_mode = always               # WAL 아카이빙 (승격 대비)
listen_addresses = '*'

# -------------------------------------------------------
# Timeouts
# -------------------------------------------------------
statement_timeout = 30000                       # 30s
lock_timeout = 10000                            # 10s
idle_in_transaction_session_timeout = 60000     # 60s
idle_session_timeout = 1800000                  # 30min (캐스케이드 최하위)

# -------------------------------------------------------
# Autovacuum
# -------------------------------------------------------
autovacuum = on
autovacuum_max_workers = 3
autovacuum_naptime = 1min
autovacuum_vacuum_scale_factor = 0.1
autovacuum_vacuum_cost_limit = 2000

# -------------------------------------------------------
# Query Planner (SSD)
# -------------------------------------------------------
random_page_cost = 1.1
effective_io_concurrency = 200
```

**PgPool-II (`pgpool.conf`):**

```conf
# -------------------------------------------------------
# Connection Pooling
# -------------------------------------------------------
num_init_children = 120           # DBA 운영 권장값 (PgPool 연결 풀링으로 백엔드 100 이하 유지)
max_pool = 1                     # 단일 DB/단일 계정
child_life_time = 1680           # 28min
connection_life_time = 1680      # 28min
client_idle_limit = 600          # 10min
reserved_connections = 1         # 관리 접속 보장

# -------------------------------------------------------
# Load Balancing
# -------------------------------------------------------
load_balance_mode = on
backend_clustering_mode = 'streaming_replication'
backend_weight0 = 1              # Primary (쓰기 전담 + 최소 읽기)
backend_weight1 = 3              # Replica (읽기 부하 집중)

# -------------------------------------------------------
# Health Check
# -------------------------------------------------------
health_check_period = 30
health_check_timeout = 10
health_check_max_retries = 3

# -------------------------------------------------------
# Watchdog (SPOF 방지)
# -------------------------------------------------------
use_watchdog = on
wd_hostname = 'pgpool-node1'
wd_vip = '10.0.0.100'

# -------------------------------------------------------
# Auto Failover
# -------------------------------------------------------
failover_command = '/etc/pgpool-II/failover.sh'
```

---

### 3.2 PostgreSQL — Standalone (개발/테스트 전용)

> **본 구성은 개발 및 테스트 환경에 한해서만 허용됨.** 프로덕션 환경에서는 PgPool-II + Streaming Replication 구성(3.1절)을 적용해야 함.

#### 핵심 파라미터 차이 (PgPool+SR 대비)

| 파라미터 | Standalone | PgPool+SR 대비 차이 | 역할 |
|:---|:---:|:---|:---|
| **wal_level** | `replica` | PgPool+SR: `replica` (동일) | 기본 표준은 `replica` (PITR 및 아카이브 백업 허용). `minimal`은 백업이 전혀 필요 없는 순수 개발계 및 휘발성 임시/로그 데이터 장비에 한해서만 허용 |
| **max_wal_senders** | `0` | PgPool+SR: `5` | 복제 연결이 필요 없으므로 불필요한 WAL Sender 프로세스 생성을 방지함 |
| **hot_standby** | `off` | PgPool+SR: `on` (Replica) | Standby 노드가 없으므로 비활성화 |
| **archive_mode** | `off` | PgPool+SR: `always` | 아카이빙이 불필요한 개발/테스트 환경 |
| **max_connections** | `100` | PgPool+SR: `100` | 단일 WAS 또는 소수 WAS만 연결되므로 100으로 충분함 |

#### RAM별 파라미터 매트릭스

| DB 서버 RAM | shared_buffers | effective_cache_size | work_mem | maintenance_work_mem | wal_buffers | max_connections |
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **8 GB** | 2 GB | 6 GB | 8 MB | 384 MB | 16 MB | 100 |
| **16 GB** | 4 GB | 12 GB | 16 MB | 768 MB | 16 MB | 100 |
| **32 GB** | 8 GB | 24 GB | 32 MB | 1.5 GB | 16 MB | 100 |
| **64 GB** | 16 GB | 48 GB | 64 MB | 3 GB | 16 MB | 100 |

> `wal_level` 기본 표준은 `replica`로 설정하여 PITR 및 아카이브 백업을 항상 허용함. 순수 개발계 및 휘발성 데이터 장비에 한해 `minimal`로 변경을 허용함. `max_wal_size`는 기본값(1GB) 사용 가능하나, 쓰기 빈도에 따라 2GB까지 상향 허용.

---

### 3.3 MongoDB — Replica Set PSS (프로덕션 표준)

대부분의 상용 서비스에 적용하는 **프로덕션 표준 아키텍처**임. HA 필수, 트랜잭션 필요, 데이터 규모 < 1TB, 쓰기 TPS < 20,000인 환경에 적합함.

#### 아키텍처 (PSS 표준)

```mermaid
graph LR
    APP[WAS<br/>MongoDB Driver] --> MG_P[(mongod<br/>Primary)]
    MG_P -->|Oplog<br/>Replication| MG_S1[(mongod<br/>Secondary 1)]
    MG_P -->|Oplog<br/>Replication| MG_S2[(mongod<br/>Secondary 2)]
    MG_S1 -.->|Read<br/>Optional| APP
    MG_S2 -.->|Read<br/>Optional| APP

    style MG_P fill:#d4edda,stroke:#28a745
    style MG_S1 fill:#fff3cd,stroke:#856404
    style MG_S2 fill:#fff3cd,stroke:#856404
```

#### PSS vs PSA 비교

| 기준 | PSS (표준) | PSA (금지) |
|:---|:---|:---|
| **구성** | Primary 1 + Secondary 2 | Primary 1 + Secondary 1 + Arbiter 1 |
| **데이터 복제** | 3중 복제 | 2중 복제 (Arbiter는 데이터 미보관) |
| **읽기 분산** | Secondary 2노드 활용 | Secondary 1노드만 활용 |
| **데이터 안전성** | 1노드 장애까지 정상 서비스 유지 가능 (2노드 장애 시 쓰기 불가. 데이터 복구만 가능) | 1노드(Secondary) 장애 시 fail-over는 가능하나, 데이터 노드 과반수 미달로 `w:majority` 쓰기 불가 (stall 장애 발생) |

> **PSA 구조 치명적 제약**: PSA 구조에서 단 1대의 Secondary 노드가 다운될 경우, 남은 데이터 노드가 Primary 1대뿐이므로 과반수 합의(Majority Consensus)가 불가능해짐. 이 경우 `w:majority` 설정이 적용된 모든 쓰기 트랜잭션이 영구 정지(Stall)되는 치명적인 장애가 발생함. 따라서 정산, 결제 등 트랜잭션 정합성이 필수적인 도메인에는 **PSA 구성을 절대 금지**하며, 반드시 PSS 구성을 준수해야 함.

#### 핵심 파라미터

| 파라미터 | 표준값 | 역할 |
|:---|:---|:---|
| **cacheSizeGB** | `0.5 * (RAM - 1)` 단, 32GB+는 하향 조정 | WiredTiger 스토리지 엔진이 사용할 내부 캐시의 크기(GB). DB 전용 서버 기준으로 RAM에서 OS 및 기타 프로세스용 1GB를 제외한 50%를 할당함. 단, 32GB 이상 서버에서는 대량 커넥션 스레드 스택(1MB/커넥션) 및 OS page cache(Oplog 복제, 파일 I/O) 마진 확보를 위해 기본 공식(50%)보다 하향 설정함 (8/16GB: 공식 적용, 32GB: 12GB, 64GB: 24GB). 과다 설정 시 OS 메모리 부족으로 스와핑이 발생하여 전체 성능이 급감함. 공유 환경(WAS/DB 혼합)에서는 `RAM * 0.25`로 명시적 제한 |
| **replSetName** | `rs0` (환경에 맞게 명명) | Replica Set의 식별자. 클라이언트(Driver)는 이 이름으로 Replica Set에 연결하며, 노드 간 통신 및 선거(Election) 과정에서도 이 이름으로 클러스터를 식별함. 동일 클러스터에 속한 모든 노드가 동일한 `replSetName`을 가져야 함 |
| **writeConcern** | 서비스 특성에 따라 (아래 표 참조) | 쓰기 연산이 몇 개의 노드에까지 반영되어야 성공으로 간주할지를 결정하는 설정. 데이터 정합성과 쓰기 성능 간의 트레이드오프를 제어함 |
| **readPreference** | 서비스 특성에 따라 (아래 표 참조) | 클라이언트가 어느 노드(Primary 또는 Secondary)에서 데이터를 읽을지를 결정하는 설정. 읽기 부하 분산과 데이터 최신성 간의 트레이드오프를 제어함 |
| **Profiling Level** | `1 (slowms: 100)` | 슬로우 쿼리 및 COLLSCAN(컬렉션 스캔)을 감지하기 위한 프로파일링 수준. Level 1은 100ms 이상 소요된 연산만 기록함. COLLSCAN은 인덱스가 없어 전체 도큐먼트를 순회하는 비효율적 쿼리 패턴으로, 프로덕션에서 발생 시 즉시 인덱스 추가가 필요함 |
| **electionTimeoutMillis** | `10000` (10s, 기본값) | Secondary 노드가 Primary로부터 하트비트 신호를 받지 못했을 때, Primary 장애로 판단하고 새 Primary 선거(Election)를 시작하기까지 대기하는 시간. 너무 짧으면 네트워크 일시 지연에도 불필요한 페일오버가 발생하고, 너무 길면 실제 장애 시 복구가 지연됨 |
| **defaultMaxTimeMS** | 권장: `60000` | **MongoDB 8.0 신규** 파라미터. 개별 읽기 연산의 기본 시간 제한(ms). 장기 실행 쿼리가 무한정 실행되어 서버 자원을 독점하는 것을 방지함 |
| **maxIncomingConnections** | RAM별 차등 (1,000~10,000) | MongoDB가 수용할 최대 동시 클라이언트 연결 수. MongoDB는 커넥션당 1개 스레드를 할당하며, Linux 스레드 스택은 기본 1MB이므로 커넥션 수에 비례하여 메모리 소모 (예: 5,000커넥션 ≈ 5GB). 기본값(65536)은 소형 서버에서 OOM 유발 위험이 있으므로 RAM 용량에 따라 명시적 상한 설정이 필수임. WAS 30대 × 100풀 = 3,000이므로, 상용 환경에서는 최소 5,000 이상 권장 |
| **internalQueryExecMaxBlockingSortBytes** | RAM별 차등 (32~256 MB) | 인덱스가 없는 필드에 대한 블로킹 정렬(Blocking Sort, 인메모리 정렬) 시 세션당 허용하는 최대 메모리 바이트 상한선. PostgreSQL의 `work_mem`과 유사한 개념. 악성 쿼리 1개가 시스템 전체 메모리를 고갈시키지 않도록 격리하되, 대규모 상용 척력(32G~64G)에서는 정상 대용량 쿼리가 제한에 걸려 실패하는 현상을 방지하기 위해 가용 마진 내에서 배수 상향 조정함 |

#### Write Concern / Read Preference 의사결정표

| 서비스 유형 | writeConcern | readPreference | 사유 |
|:---|:---|:---|:---|
| 정산/결제 (정합성 필수) | `w: majority` | `primary` | 데이터 유실 허용 불가, 항상 최신 데이터 보장 |
| 일반 상용 (HA 필요) | `w: 1` (기본) | `primary` | 기본 안정성 확보 |
| 조회성 (Replication Lag 허용) | `w: 1` | `secondaryPreferred` | 읽기 부하 분산 |
| 대시보드/통계 (실시간성 낮음) | `w: 1` | `secondary` | Primary 읽기 부하 제로화 |

> **핵심 제약**: 정산/결제 서비스는 **반드시 `primary` readPreference**를 유지해야 함.
>
> **MongoDB 8.0 Write Concern 동작 변경**: `w:majority` 설정 시, 8.0부터 majority 노드가 oplog 엔트리를 **write한 시점**에 acknowledgment를 반환함 (기존: oplog **적용 완료** 후 ack). 이로 인해 `w:majority` 쓰기 **성능이 향상**됨. 단, ack 반환 시점과 실제 데이터 적용 시점 간에 미세한 갭이 존재할 수 있으나 정합성 보장에는 영향 없음.
>
> **MongoDB 8.0 LDAP 인증 폐지 안내**: MongoDB 8.0부터 Legacy 방식인 **LDAP Authorization using Query Mode**가 **deprecated** 처리됨. 단, 표준 LDAP 통신을 위한 기본 인증 및 표준 그룹 인가 기능은 여전히 정상 지원됨. 향후 인증 체계를 신규 도입하거나 시스템 고도화를 검토하는 경우, 장기적인 버전 업그레이드 시에도 완벽한 보안 호환성을 보장받을 수 있는 **OIDC(OpenID Connect)** 인증 체계를 대안 표준으로 검토해야 함.

#### RAM별 파라미터 매트릭스 (노드당)

| DB 서버 RAM | cacheSizeGB | maxIncomingConnections | internalQueryExecMaxBlockingSortBytes | 비고 |
|:---:|:---:|:---:|:---:|:---|
| **8 GB** | 3.5 GB | 1,000 | 32 MB | PSS 3노드 각각 동일 적용 |
| **16 GB** | 7.5 GB | 2,000 | 64 MB | 표준 프로덕션 |
| **32 GB** | 12.0 GB | 5,000 | 128 MB | 고성능. cacheSizeGB 하향 (OS page cache 마진 확보) |
| **64 GB** | 24.0 GB | 10,000 | 256 MB | 대규모. cacheSizeGB 하향 (대량 커넥션 + page cache 마진) |

#### 실무 설정 스크립트

**`mongod.conf` (각 노드 공통, 8GB DB 전용 서버 기준):**

```yaml
# -------------------------------------------------------
# Storage (8GB DB 전용 서버 기준)
# -------------------------------------------------------
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 3.5            # 0.5 * (8 - 1) = 3.5GB

# -------------------------------------------------------
# Query Settings
# -------------------------------------------------------
setParameter:
  internalQueryExecMaxBlockingSortBytes: 33554432  # 32MB (8GB RAM 기준)

# -------------------------------------------------------
# Replica Set
# -------------------------------------------------------
replication:
  replSetName: rs0                # Replica Set 명

# -------------------------------------------------------
# Profiling (COLLSCAN 감지 필수)
# -------------------------------------------------------
operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100

# -------------------------------------------------------
# Network
# -------------------------------------------------------
net:
  port: 27017
  bindIp: 0.0.0.0
  maxIncomingConnections: 1000    # 8GB RAM 기준 (커넥션당 1MB 스레드 스택)

# -------------------------------------------------------
# Security (프로덕션 필수)
# -------------------------------------------------------
security:
  keyFile: /etc/mongodb/keyfile   # 멤버 간 인증
  authorization: enabled          # 클라이언트 인증

# -------------------------------------------------------
# Logging
# -------------------------------------------------------
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
```

**Replica Set 초기화 (mongosh):**

```javascript
// Primary에서 실행
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo-primary:27017", priority: 2 },
    { _id: 1, host: "mongo-secondary1:27017", priority: 1 },
    { _id: 2, host: "mongo-secondary2:27017", priority: 1 }
  ]
})

// 초기화 완료 후 Profiling 설정
db.setProfilingLevel(1, { slowms: 100 })

// Write Concern / Read Preference 설정 (연결 문자열 예시)
// mongodb://user:pass@mongo-primary:27017,mongo-secondary1:27017,mongo-secondary2:27017/?replicaSet=rs0&w=majority&readPreference=primary
```

---

### 3.4 MongoDB — Standalone (개발/테스트 전용)

> **본 구성은 개발 및 테스트 환경에 한해서만 허용됨.** 프로덕션 환경에서는 Replica Set PSS 구성(3.3절)을 적용해야 함. MongoDB Standalone은 **멀티 도큐먼트 트랜잭션을 지원하지 않음**.

#### RAM별 파라미터 매트릭스

| DB 서버 RAM | cacheSizeGB | maxIncomingConnections | internalQueryExecMaxBlockingSortBytes | 비고 |
|:---:|:---:|:---:|:---:|:---|
| **8 GB** | 3.5 GB | 1,000 | 32 MB | 개발/테스트 |
| **16 GB** | 7.5 GB | 2,000 | 64 MB | 개발/테스트 |
| **32 GB** | 12.0 GB | 5,000 | 128 MB | 프로토타입 (RS 전환 계획 필수) |
| **64 GB** | 24.0 GB | 10,000 | 256 MB | 프로토타입 (RS 전환 계획 필수) |

#### 실무 설정 스크립트

**`mongod.conf` (개발/테스트용, 8GB 기준):**

```yaml
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 3.5            # 0.5 * (8 - 1)

setParameter:
  internalQueryExecMaxBlockingSortBytes: 33554432  # 32MB (8GB RAM 기준)

net:
  port: 27017
  bindIp: 0.0.0.0
  maxIncomingConnections: 1000    # 8GB RAM 기준

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true

operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100
```

> Standalone은 `replication` 섹션을 설정하지 않음. 사용자 증가가 예상되는 프로토타입은 Replica Set 전환 계획을 수립해야 함.

---

### 3.5 DB 공통: RAM별 핵심값 치트시트

| DB 서버 RAM | PG shared_buffers | PG max_wal_size | Mongo cacheSizeGB | 비고 |
|:---:|:---:|:---:|:---:|:---|
| **8 GB** | 2 GB | 2 GB | 3.5 GB | 소규모 |
| **16 GB** | 4 GB | 4 GB | 7.5 GB | 표준 |
| **32 GB** | 8 GB | 16 GB | 12.0 GB | 고성능 (cacheSizeGB 하향: page cache 마진) |
| **64 GB** | 16 GB | 32 GB | 24.0 GB | 대규모 (cacheSizeGB 하향: page cache 마진) |

---

## 4. 타임아웃 캐스케이드 (Timeout Cascade)

### 핵심 원칙

타임아웃 캐스케이드는 **상위 계층이 하위 계층보다 먼저 연결을 종료**하도록 설정하여, 프록시 레이스 컨디션(간헐적 502/503 에러) 및 무효 커넥션 예외를 차단함. 등호(`<=`)가 아닌 **엄격한 부등호(`<`)**로 계층 간 타임아웃을 격리해야 함.

**방화벽 제약**: 사내망 TCP Established Timeout = 30분(1,800초). 모든 타임아웃 산정의 최상위 기준.

### 4.1 Web Server → WAS

| 파라미터 | 표준값 | 역할 |
|:---|:---|:---|
| **Apache keepAliveTimeout** | `3s` | Apache가 클라이언트와의 Keep-Alive 연결을 유지하는 최대 유휴 시간. 이 시간 내에 새 요청이 없으면 연결을 종료함 |
| **Apache ProxyPass ttl** | `10s` (권장, default 60s) | Apache가 WAS로 프록시한 연결을 유휴 상태로 유지하는 최대 시간. WAS `keepAliveTimeout` 만료 이전에 유휴 커넥션을 관리해야 함 |
| **WAS keepAliveTimeout** | `15s` | WAS가 Keep-Alive 연결을 유지하는 최대 유휴 시간 |

> **핵심 제약**: 리버스 프록시 아키텍처에서 상위 계층(Apache)이 하위 계층(WAS)보다 **먼저** 연결 종료를 개시해야 함. 이 순서가 위반되면 Apache가 이미 종료된 WAS 연결을 재사용하려 시도하여 간헐적 **502/503** 응답이 발생함.

```apache
# httpd.conf - ProxyPass 설정 예시
ProxyPass / http://was:8080/ ttl=10 keepalive=On
ProxyPassReverse / http://was:8080/
```

### 4.2 WAS → PostgreSQL (PgPool-II 경유)

```
WAS HikariCP maxLifetime (1,620,000ms = 27min)
     |
     |  maxLifetime < child_life_time < idle_session_timeout
     v
PgPool-II child_life_time (1,680s = 28min)
     |
     |  child_life_time < idle_session_timeout
     v
PostgreSQL idle_session_timeout (1,800,000ms = 30min)
```

### 4.3 WAS → PostgreSQL (직접 연결)

```
WAS HikariCP maxLifetime (1,620,000ms = 27min)
     |
     |  maxLifetime < idle_session_timeout < 방화벽 timeout
     v
PostgreSQL idle_session_timeout (1,800,000ms = 30min)
     |
     v
방화벽 TCP Established Timeout (30min / 1,800s)
```

### 4.4 WAS → MongoDB (Replica Set)

```
WAS HikariCP / MongoDB Driver maxLifetime (1,620,000ms = 27min)
     |
     v
MongoDB connectionPool maxIdleTimeMS (1,800,000ms = 30min)
     |
     v
MongoDB driver socketTimeoutMS (0 = 무제한, 애플리케이션 레벨 제어)
```

> MongoDB는 논리 세션(`localLogicalSessionTimeoutMinutes`, 30min)과 물리 커넥션(Driver `maxIdleTimeMS`)의 이중 구조를 가짐. HikariCP `maxLifetime`(27min)은 두 계층 모두보다 짧게 유지하여야 함.

### 4.5 WAS 커넥션 풀 공통 설정 (HikariCP)

모든 데이터베이스 연동에 공통 적용되는 HikariCP 파라미터 표준값임.

| 파라미터 | 표준값 | 역할 |
|:---|:---|:---|
| **HikariCP maxLifetime** | `1,620,000ms` (27분) | 풀(Pool) 내부 커넥션의 최대 생존 수명. 하위 모든 DB의 유휴 세션 제한(30분) 및 사내망 방화벽 임계치(30분)보다 3분 먼저 커넥션을 스스로 폐기 및 재생성(Recycle)하여, DB 또는 방화벽이 먼저 연결을 강제 차단함에 따라 발생하는 커넥션 단절 예외(**Connection reset**)를 원천 차단함. PgPool-II 경유 시 `child_life_time`(28분)과의 1분 갭도 확보 |
| **HikariCP connectionTimeout** | `30,000ms` (30초) | 애플리케이션 스레드가 풀에서 커넥션을 획득하기 위해 대기하는 최대 시간. DB 병목 시 스레드가 무한 대기하여 WAS 전체 스레드 풀이 고갈되는 현상을 방지하는 최후의 안전장치(**Fail-Fast**) |
| **HikariCP minimumIdle** | `= maxPoolSize` | 상시 유지할 가용 유휴 커넥션의 최소 개수를 `maxPoolSize`와 동일하게 강제. 트래픽 스파이크 유입 시 커넥션 풀이 동적으로 축소 및 확장되면서 발생하는 CPU 및 네트워크 핸드셰이크 오버헤드를 제거하는 **Fixed-size pool** 전략 |
| **HikariCP keepaliveTime** | `60,000ms` (1분) | 유휴 커넥션의 유효성을 검증하기 위해 주기적으로 DB에 송신하는 초경량 생존 신호(Ping) 주기. 중간 방화벽의 세션 유휴 무단 차단(**Silent Drop**)을 예방하고, 일시적 네트워크 순단 발생 시 죽은 커넥션(**Stale Connection**)을 최대 1분 이내에 신속히 감지하여 제거함 |

> **keepaliveTime 설정 필수사항**: WAS와 DB 사이에 방화벽이 존재하는 환경에서는 방화벽의 유휴 세션 타임아웃이 양 단말에 통보 없이 커넥션을 **Silent Drop**할 수 있음.
> - `keepaliveTime`은 방화벽의 유휴 타임아웃보다 **짧은 값**으로 설정
> - JDBC 4+ 드라이버를 기준으로 함. `connectionTestQuery`를 별도 지정하지 말고 드라이버 자체 `isValid()` 메커니즘을 활용할 것
> - **Fixed-size pool** (`minimumIdle = maxPoolSize`) 환경은 유효하지 않은 커넥션이 풀에 잔류할 위험이 특히 크므로 keepalive 검증이 **필수**

### 4.6 DB 벤더별 유휴 세션 제한 설정

사내망 Established 방화벽 임계치(**30분 / 1,800초**)와 연동되는 각 DBMS별 유휴 세션 강제 종료 파라미터임. HikariCP `maxLifetime`(27분)이 아래 모든 DB의 세션 제한(30분)보다 선행하여 커넥션을 Recycle함으로써 타임아웃 캐스케이드 일관성을 확보하는 **최하부 거버넌스 한계선**임.

| DBMS | 파라미터 | 표준값 | 역할 |
|:---|:---|:---|:---|
| **PostgreSQL** | `idle_session_timeout` | `1,800,000ms` (30분) | 클라이언트 유휴 세션을 강제 종료하여 연결 누수(Leak) 및 좀비 세션으로부터 DB 프로세스 자원 및 메모리를 보호함 |
| **MongoDB** | `localLogicalSessionTimeoutMinutes` | `30` (30분) | 서버 내 **논리 세션(Logical Session) 상태의 유휴 만료 규격**을 정의하여 세션 누적으로 인한 서버 자원 고갈을 방지함. 이 파라미터는 물리적인 TCP 소켓 커넥션을 강제 차단하는 타임아웃이 아니며, MongoDB 서버 프로세스가 추적하는 세션 레코드의 논리적 만료 시간을 의미함 |
| **DB2** | `CONNECTIONIDLETIME` (LUW) / `IDTHTOIN` (z/OS) | `30 MINUTE` / `1,800s` | 유휴 세션의 DB 엔진 자원(Memory, Lock) 점유를 해제하여 전체 가용성 확보 |

> **MongoDB 물리적 유휴 커넥션 관리**: `localLogicalSessionTimeoutMinutes`는 논리 세션의 만료 규격일 뿐, 물리적 TCP 커넥션 자체의 유휴 상태를 직접 관리하지 않음. 물리적 유휴 커넥션의 수명 및 회수는 **드라이버단의 `maxIdleTimeMS` 설정**을 통해 제어되므로, HikariCP의 `maxLifetime` 및 `keepaliveTime`과 연동하여 커넥션 풀 수준에서 관리되어야 함. 이중 구조(논리 세션 / 물리 커넥션)를 가지는 MongoDB의 특성상 두 계층을 혼동하지 않도록 주의.

> **타임아웃 캐스케이드 인과관계**:
>
> ```
> HikariCP maxLifetime (27분)
>       |  maxLifetime < DB 유휴 세션 제한 (30분) 보장
>       |  maxLifetime < 방화벽 임계치 (30분) 보장
>       v
> DB 유휴 세션 제한 (30분)
>       |-- PostgreSQL: idle_session_timeout  -- 물리적 유휴 세션 강제 종료
>       |-- MongoDB: localLogicalSessionTimeoutMinutes  -- 논리 세션 상태 만료
>       |         (물리적 커넥션 관리는 드라이버 maxIdleTimeMS로 제어)
>       +-- DB2: CONNECTIONIDLETIME (LUW) / IDTHTOIN (z/OS)
>       |
>       v
> 방화벽 TCP Established Timeout (30분 / 1,800초)
> ```
>
> HikariCP가 모든 하위 계층(DB, 방화벽)보다 **3분 먼저** 커넥션을 폐기 및 재생성(Recycle)함으로써 DB 또는 방화벽에 의한 강제 차단으로 발생하는 **Connection reset** 예외를 원천 차단함.

### 4.7 PostgreSQL 내부 타임아웃 Guardrails

각 파라미터는 종속 관계가 아닌, 서로 다른 시점에 동작하는 **독립적인 가드레일**임.

| 파라미터 | 표준값 | 역할 |
|:---|:---|:---|
| **statement_timeout** | `30,000ms` (30s) | 현재 실행 중인 쿼리(Active Query)의 최대 지속 시간을 제한함. 장기 실행 쿼리가 DB 서버의 CPU, 메모리, 디스크 I/O를 독점하여 타 쿼리의 응답 지연 및 전체 장애로 이어지는 것을 방지함 |
| **lock_timeout** | `10,000ms` (10s) | 쿼리 실행 도중 Lock(잠금) 획득을 대기하는 최대 시간을 제한함. 10초 이내에 Lock을 획득하지 못하면 해당 쿼리를 자동 취소함. 교착(Deadlock) 상태로 인한 전파 장애를 방지하는 핵심 가드레일. `lock_timeout > statement_timeout`으로 설정하면 의미가 없으므로 반드시 `lock_timeout < statement_timeout`을 유지해야 함 |
| **idle_in_transaction_session_timeout** | `60,000ms` (60s) | 트랜잭션이 시작(`BEGIN`)된 후 쿼리 수행은 완료되었으나, 이후 아무런 쿼리도 없이 유휴(Idle in Transaction) 상태로 머무는 시간을 제한함. `COMMIT`/`ROLLBACK` 없이 방치되는 세션은 Lock을 계속 점유하여 타 트랜잭션을 차단하므로, 60초 이상 유휴 시 강제 종료하여 전파 장애를 예방함 |

```
PostgreSQL Session Timeout Guardrails
  |
  |-- statement_timeout (30s)
  |     쿼리가 실행 중인 상태(Active Query)의 최대 지속 시간 제어.
  |     장기 실행 쿼리로 인한 리소스 독점 방지.
  |     |
  |     +-- lock_timeout (10s)
  |            statement_timeout 실행 도중 Lock 대기 시간에만 관여.
  |            Lock 획득 대기 10초 초과 시 자동 취소 (교착 방지).
  |            lock_timeout > statement_timeout 설정은 의미 없음.
  |
  |-- idle_in_transaction_session_timeout (60s)
        트랜잭션 시작 후 쿼리 수행이 완료된 상태에서
        이후 아무런 쿼리도 없이 유휴(Idle in Transaction) 상태로
        머무는 시간 제어. BEGIN 이후 COMMIT/ROLLBACK 없이
        방치되는 세션의 Lock 점유 및 커넥션 낭비 방지.
```

---

## 5. 공유 DB 환경 커넥션 풀 가용 가이드 (70% Ceiling Rule)

### 5.1 핵심 원칙

다수의 서비스가 단일 DB 인스턴스를 공유하는 환경에서, 특정 애플리케이션의 트래픽 스파이크가 타 서비스의 커넥션을 고갈시키는 것을 차단하기 위한 자원 격리(Partitioning) 규칙임.

#### 직접 연결 (WAS → PostgreSQL Standalone)

PgPool-II 없이 WAS가 DB에 직접 연결하는 환경에서 적용함.

```
DB max_connections = 100 (DB 서버 설정)
      |
      +-- 30 예약: 관리자 세션(superuser_reserved_connections=3), 모니터링, 긴급 접속
      |
      +-- 70 가용: 애플리케이션 커넥션 풀 전체 합산 상한 (max_connections * 0.7)
             |
             +-- 팀별 쿼터 할당 (서비스 중요도 + 트래픽 가중치 기반)
```

> **절대 제약 (직접 연결 시)**: 모든 애플리케이션의 `maxPoolSize` 합산값은 `DB max_connections * 0.7`(=70)을 **초과해서는 안 됨**

#### PgPool-II 경유 연결 (WAS → PgPool-II → PostgreSQL)

PgPool-II가 중간 계층으로 존재하는 프로덕션 환경에서는 PgPool-II가 DB 연결을 통제하므로, WAS 풀 합산이 `max_connections`를 초과할 수 있음.

```
PgPool-II 공식 권고 (참고):
  max_pool × num_init_children ≤ (max_connections - superuser_reserved_connections)
  → 공식 준수 시: 1 × 97 ≤ (100 - 3) = 97

DBA 운영 권장 (적용값):
  num_init_children = 120
  → 공식 초과(1 × 120 > 97)이나, PgPool-II 연결 풀링(커넥션 캐싱 및 재사용)으로
     실제 동시 백엔드 연결은 100 이하로 유지됨
  → 120개 child 프로세스가 동시에 백엔드 연결을 점유하지 않음:
     유휴 세션은 백엔드 연결을 캐시에서 대기, 활성 쿼리 프로세스만 백엔드 점유

WAS → PgPool 계층:
  WAS 전체 풀 합산(80~100) ≤ num_init_children(120)
  → 초과 클라이언트는 PgPool Listen Queue에서 대기 후 순차 처리
  → 피크 타임에도 120 이하인 것이 일반적임

⚠️ 최악 시나리오 (모니터링 필수):
  120개 child 프로세스가 동시에 백엔드 연결을 요구하는 극단적 상황에서
  PostgreSQL이 "too many clients already" 반환 + failover 트리거 가능
  → SHOW POOL_PROCESSES로 피크 타임 활성 백엔드 연결 수 주기적 모니터링 권장
```

> **PgPool-II 환경에서의 70% Rule 재해석**: 직접 연결 환경의 70% Ceiling Rule은 PgPool-II 환경에서는 PgPool-II의 연결 풀링으로 대체됨. WAS 풀 합산이 `num_init_children`을 초과하더라도 PgPool Listen Queue가 초과분을 안전하게 흡수함.

### 5.2 팀별 최종 확정 maxPoolSize

| 팀 / 서비스 | 현행 Java | 현행 풀 설정 | **최종 확정 maxPoolSize** | 보정 사유 |
|:---|:---:|:---:|:---:|:---|
| **플랫폼개발 (Nice Park)** | 17 | 5 | **20** | 기존 풀 과소 설정으로 인한 처리량 병목 개선 |
| **플랫폼개발 (Nice Charger)** | 15, 25 | 100 / 20 | **20** | 웹 풀 100을 20으로 축소 (공유 DB 보호) |
| **CL플랫폼 (CLS 전용)** | 15.0.2 | 50 | **20** | 현금정보계와 동일 서버 사용. 인스턴스당 20으로 통일 |
| **주차서비스 (Tomcat 9.x)** | 15.0.2 | 100 | **20** | 과대 설정 축소 (공유 DB 리소스 고갈 방지) |
| **현금정보계 (Liberty 23.x)** | 15.0.2 | 50 | **20** | 7개 컨테이너 다중화 환경. 컨테이너당 20 (총 7 x 20 = 140) |

> **CL플랫폼 및 현금정보계 적용 전 필수 검증**: `maxPoolSize` 축소 적용 전, APM 모니터링을 통해 실제 피크 타임의 **Active Connection Peak 수치를 반드시 검증**해야 하며, 커넥션 고갈 우려 시 **WAS 인스턴스 스케일 아웃을 병행**해야 함.

### 5.3 PgPool-II 커넥션 풀 산출 예시 (플랫폼개발팀 나이스M 기준)

| 항목 | 산출 수치 | 비고 |
|:---|:---|:---|
| 대상 서비스 | 나이스M (Nice M) | PostgreSQL(via PgPool-II) + MongoDB 운영 |
| 총 WAS 인스턴스 수 | 4개 (이중화) | WAS 표준 설정 기준 |
| 전체 WAS 풀 합산 | **80 ~ 100개** | 인스턴스당 20~25 |
| **PgPool num_init_children** | **120** | DBA 운영 권장값. PgPool 연결 풀링으로 실제 백엔드 동시 연결은 100 이하 유지 |
| **PgPool max_pool** | **1** | 단일 DB/단일 계정 |
| PgPool → PG 최대 백엔드 연결 | 100개 (이론적 상한) | PgPool 연결 풀링(캐싱/재사용)으로 실제 동시 점유는 100 이하. 120개가 동시 활성화되는 극단적 피크 시 "too many clients" 위험 존재 |
| **PG max_connections** | **100** | OOM 예방 100 고정 |
| WAS 풀(80~100) ≤ num_init_children(120) | 정상 | 초과 클라이언트는 PgPool Listen Queue에서 대기 후 순차 처리. 피크 타임에도 120 이하인 것이 일반적임 |

### 5.4 아키텍처별 커넥션 풀 전략

| 아키텍처 | 커넥션 풀 계층 | WAS 설정 | 중간 계층 | DB 설정 |
|:---|:---|:---|:---|:---|
| **PG Standalone** (개발/테스트) | WAS → PG | HikariCP maxPoolSize=20 | 없음 | max_conn = 100 고정 |
| **PG PgPool+SR** (프로덕션) | WAS → PgPool → PG | HikariCP maxPoolSize=20~25 | PgPool num_init_children = 120 (DBA 운영 권장, 연결 풀링으로 백엔드 100 이하 유지) | max_conn = 100 고정 |
| **MongoDB RS** (프로덕션) | WAS → RS | maxPoolSize=20~50 (MongoDB Driver) | 없음 | maxIncomingConnections=RAM별 차등 (1,000~10,000) |

---

## 6. 모니터링 최소 체계

프로덕션 환경에서 **반드시 구축해야 할 최소 모니터링 항목**을 계층별로 정의함. 각 팀은 본 장의 기준에 따라 모니터링 체계를 구축하고, 임계치 도달 시 즉시 알림을 수신할 수 있도록 설정해야 함.

### 6.1 WAS 모니터링

| 모니터링 항목 | 조회 방법 | 경고(Warning) | 위험(Critical) | 조치 |
|:---|:---|:---|:---|:---|
| **Heap 사용률** | GC 로그 / APM (JMX) | Old Gen > 70% | Old Gen > 85% | Heap 덤프 분석, 메모리 누수 의심 |
| **GC Pause Time** | GC 로그 (`-Xlog:gc*`) | STW > 500ms 빈발 | STW > 2s | GC 전략 재검토 (Parallel → G1 전환) |
| **Active Thread 수** | APM / JMX | maxThreads 70% | maxThreads 85% | 스레드 덤프 분석, 병목 구간 식별 |
| **커넥션 풀 대기** | HikariCP 메트릭 (JMX / Micrometer) | 대기 발생 | 대기 > 5s | maxPoolSize 증설 검토 또는 DB 병목 확인 |
| **에러율 (5xx)** | APM / Access Log | > 1% | > 5% | 원인 분석 (DB 타임아웃 / 스레드 고갈 등) |
| **커넥션 누수** | HikariCP `leakDetectionThreshold` | 감지 시 | -- | 누수 발생 지점(스택 트레이스) 분석 |

### 6.2 PostgreSQL 모니터링

| 모니터링 항목 | 조회 방법 | 경고(Warning) | 위험(Critical) | 조치 |
|:---|:---|:---|:---|:---|
| **Active Sessions** | `SELECT count(*) FROM pg_stat_activity WHERE state = 'active'` | max_connections 70% | max_connections 85% | 커넥션 풀 설정 재검토 |
| **Slow Query** | `pg_stat_statements` (>= 1s) | 발생 시 | 빈발 시 | 인덱스 추가 / 쿼리 튜닝 |
| **Replication Lag** | `SELECT now() - pg_last_xact_replay_timestamp() FROM pg_stat_replication` | > 5s | > 30s | 네트워크/부하 점검, Replica 증설 검토 |
| **Dead Tuples** | `SELECT n_dead_tup FROM pg_stat_user_tables` | 테이블 크기 10% | 테이블 크기 20% | `VACUUM` 강제 실행, autovacuum 파라미터 조정 |
| **Cache Hit Ratio** | `SELECT sum(blks_hit)::float / NULLIF(sum(blks_hit + blks_read), 0) FROM pg_stat_database` | < 99% | < 95% | `shared_buffers` 증설 검토 |
| **Lock Wait** | `SELECT * FROM pg_locks WHERE NOT granted` | 대기 > 1s | 대기 > 5s | 트랜잭션 분석, `lock_timeout` 확인 |
| **PgPool 커넥션 사용률** | `SHOW POOL_NODES` + `SHOW POOL_PROCESSES` | 사용률 > 80% | 사용률 > 95% | `num_init_children` 증설 검토 |
| **Autovacuum 진행 상태** | `SELECT * FROM pg_stat_progress_vacuum` | 장시간 미실행 | Dead Tuple 누적 | `autovacuum_vacuum_cost_limit` 상향 |

### 6.3 MongoDB 모니터링

| 모니터링 항목 | 조회 방법 | 경고(Warning) | 위험(Critical) | 조치 |
|:---|:---|:---|:---|:---|
| **Active Connections** | `db.serverStatus().connections` | > 70% of maxIncoming | > 85% of maxIncoming | 커넥션 풀 설정 재검토 |
| **COLLSCAN (컬렉션 스캔)** | `db.system.profile.find({ "planSummary": "COLLSCAN" })` | 발생 시 | 빈발 시 | **즉시** 인덱스 추가 |
| **Slow Query** | `db.system.profile.find({ millis: { $gt: 100 } })` | > 100ms | > 1s | 인덱스 추가 / 쿼리 튜닝 |
| **Replication Lag** | `rs.printSecondaryReplicationInfo()` | > 5s | > 30s | 네트워크/부하 점검 |
| **Cache Hit Ratio** | `db.serverStatus().wiredTiger.cache` (bytes read into cache / bytes requested) | < 97% | < 95% | `cacheSizeGB` 증설 검토 |
| **Oplog Window** | `db.printReplicationInfo()` | < 1h | < 10min | Oplog Size 확장 필요 |
| **Election 이벤트** | `rs.status()` | 발생 시 | 빈발 시 | 네트워크/부하 원인 분석 |

### 6.4 모니터링 구축 체크리스트

각 팀은 다음 단계를 순서대로 점검하여 모니터링 체계를 구축해야 함.

| 단계 | 구축 항목 | 상세 지침 | 완료 기준 |
|:---:|:---|:---|:---|
| **1** | **GC 로그 활성화** | 모든 WAS 인스턴스에 `-Xlog:gc*` 옵션 적용. 로그 순환 설정(`filecount=10, filesize=50M`) 필수 | GC 로그 파일 생성 및 정상 순환 확인 |
| **2** | **HikariCP 메트릭 노출** | HikariCP JMX 활성화 또는 Micrometer 메트릭 연동. 수집 항목: 활성 커넥션 수, 유휴 커넥션 수, 대기 시간, 총 대기 횟수 | 커넥션 풀 사용률 및 대기 시간 그래프 정상 출력 확인 |
| **3** | **DB 슬로우 쿼리 감지** | PG: `pg_stat_statements` 확장 활성화 (`shared_preload_libraries`). Mongo: Profiling Level 1 설정 (`db.setProfilingLevel(1, {slowms: 100})`) | 슬로우 쿼리 발생 시 알림 수신 확인 |
| **4** | **COLLSCAN 감지 (MongoDB)** | Profiling Level 1 활성화 후 `system.profile` 컬렉션에서 `planSummary: "COLLSCAN"` 이벤트 모니터링. COLLSCAN은 인덱스 누락의 명확한 신호이므로 발생 즉시 알림 설정 | COLLSCAN 발생 시 알림 수신 확인 |
| **5** | **Replication Lag 알림** | PG: `pg_stat_replication` 뷰의 `write_lag`, `flush_lag`, `replay_lag` 컬럼 모니터링. Mongo: `rs.printSecondaryReplicationInfo()` 주기적 조회 | Lag > 5s 시 알림 수신 확인 |
| **6** | **Active Session 임계치 알림** | PG: `pg_stat_activity` WHERE state = 'active' 카운트. Mongo: `db.serverStatus().connections.current` | max_connections 70% 도달 시 알림 수신 확인 |
| **7** | **Cache Hit Ratio 추이** | PG: `pg_stat_database` 의 `blks_hit / (blks_hit + blks_read)`. Mongo: `db.serverStatus().wiredTiger.cache` 의 캐시 적중률 | < 95% 시 알림 수신 확인 |
| **8** | **에러율 (5xx) 추이** | WAS Access Log 또는 APM에서 5xx 응답 코드 비율 모니터링 | > 1% 시 알림 수신 확인 |
| **9** | **Dead Tuple 누적 감지 (PG)** | `pg_stat_user_tables.n_dead_tup` 주기적 조회. 테이블 크기 대비 비율 모니터링 | Dead Tuple 10% 초과 시 알림 수신 확인 |
| **10** | **커넥션 누수 감지** | HikariCP `leakDetectionThreshold=60000` 설정. 누수 감지 시 로그에 스택 트레이스 출력 | 누수 감지 시 알림 수신 확인 |

> **구축 우선순위**: 1~4단계는 **즉시 구축** 필수. 5~10단계는 1~4단계 완료 후 **1개월 이내** 구축 완료를 권장함.

---

## 7. 검증 체크리스트

### 7.1 WAS 검증 항목

| 검증 항목 | 충족 조건 | 위반 시 영향 |
|:---|:---|:---|
| `Xms` = `Xmx` | 프로덕션에서 반드시 동일하게 설정 | Heap 리사이즈 시 GC Pause 발생 |
| Heap < Container RAM * 0.7 | OOM 방지 | Metaspace, Thread Stack, Native Memory 부족 |
| 인스턴스당 Heap = 호스트 RAM * 0.625 / N | 다중 인스턴스 분할 원칙 | 단일 인스턴스가 호스트 RAM 과점유 |
| `maxPoolSize` = 20 | 인스턴스당 기본 표준 준수 | DB 리소스 고갈, Lock 경합 |
| Sum(`maxPoolSize`) <= DB `max_conn` * 0.7 | 공유 DB 70% Ceiling Rule | 타 서비스 커넥션 고갈, 장애 전파 |
| `minimumIdle` = `maxPoolSize` | Fixed-size pool 유지 | 풀 축소/확장 오버헤드 발생 |
| `ProxyPass ttl` < WAS `keepAliveTimeout` | 프록시 레이스 컨디션 방지 | 간헐적 502/503 에러 발생 |
| `maxLifetime` < 각 DB별 유휴 세션 제한값 | 커넥션 무효화 방지 | Connection reset 예외 발생 |
| GC 로그 활성화 | 모든 WAS 인스턴스에서 필수 | 장애 발생 시 원인 분석 불가 |
| Metaspace Max >= Min | 역전 현상 방지 | 메모리 설정 오류 |
| `maxThreads` > 0 | 무제한(-1) 설정 금지 | Backpressure 부재, 리소스 고갈 |
| `leakDetectionThreshold` 활성화 | 권장값 **60,000ms** | 커넥션 누수 무감지 |
| `vm.swappiness` = 10 (WAS 서버) | JVM 환경 안정성 | 기본값(60) 시 GC Pause 빈발 |
| `ip_local_port_range` = 32768~65535 (WAS 서버) | 아웃바운드 포트 확보 | 포트 고갈 시 커넥션 실패 |
| systemd 서비스 LimitNOFILE/LimitNPROC override 설정 | 서비스 데몬 ulimit 적용 | limits.conf 무시되어 기본값 1024로 동작 |

### 7.2 PostgreSQL (PgPool+SR) 검증 항목

| 검증 항목 | 충족 조건 | 위반 시 영향 |
|:---|:---|:---|
| `shared_buffers` <= RAM * 0.25 | PostgreSQL 공식 권장 | OOM, 커널 페이지 캐시 부족 |
| `max_connections` = 100 고정 | OOM 예방 | OOM 발생, DB 서버 다운 |
| PgPool `SHOW POOL_PROCESSES` 피크 활성 연결 < 100 | 최악 시나리오 방지 | 120개 동시 활성 시 "too many clients already" + failover 트리거 |
| `vm.swappiness` = 1 (PostgreSQL 서버) | DB 서버 안정성 | 스와핑 시 쿼리 성능 급감 |
| `vm.overcommit_memory` = 2 (PostgreSQL 서버) | OOM Killer 방지 | postmaster 강제 종료 → 전체 장애 |
| `vm.overcommit_ratio` = 90 (PostgreSQL 서버) | overcommit_memory=2 시 커밋 한도 보장 | 기본 50% → "Cannot allocate memory" 장애 |
| THP = disabled (PostgreSQL 서버) | OLTP 지연 스파이크 방지 | 간헐적 수백 ms 쿼리 지연 |
| THP 영구 설정 적용 (PostgreSQL 서버) | IT ONE 변경 요청 완료 | 리부팅 후 THP 활성화로 성능 저하 |
| systemd 서비스 LimitNOFILE/LimitNPROC override 설정 | 서비스 데몬 ulimit 적용 | limits.conf 무시되어 기본값으로 동작 |
| `autovacuum` = on | 필수 | Dead Tuple 누적, 성능 점진 저하 |
| `autovacuum_vacuum_cost_limit` >= 1000 | 기본값(200) 대비 상향 | VACUUM 처리 지연 |
| `idle_in_transaction_session_timeout` 설정 | 교착 방지 | Lock 점유로 인한 전파 장애 |
| WAS maxLifetime < PgPool child_life_time < DB idle_session_timeout | 엄격 부등호 | 레이스 컨디션 |
| PgPool `reserved_connections` >= 1 | 관리 접속 보장 | 장애 시 DBA 접속 불가 |
| PgPool `max_pool` = 1 (단일 DB/계정) | 불필요한 커넥션 폭증 방지 | 백엔드 연결 기하급수적 증가 |
| Watchdog 활성화 | SPOF 방지 | PgPool 단일 구성 시 전체 장애 |
| `random_page_cost` = 1.1 (SSD 환경) | SSD 환경 필수 | 비효율적 실행 계획 |
| Replication Slot 구성 | `pg_replication_slots` 확인 | Slot 누적 시 디스크 Full 위험 |

### 7.3 MongoDB (Replica Set PSS) 검증 항목

| 검증 항목 | 충족 조건 | 위반 시 영향 |
|:---|:---|:---|
| Replica Set >= 3노드 (PSS 표준) | Quorum 보장 | 2노드 시 Primary 선출 불가 |
| 정산/결제 서비스 readPreference = `primary` | 정합성 보장 | Secondary 읽기 시 과거 데이터 반환 |
| Profiling Level >= 1 | COLLSCAN 감지 필수 | 인덱스 누락 무감지 |
| Oplog Size 확인 | `db.printReplicationInfo()` | Oplog 부족 시 복제 중단 |
| `electionTimeoutMillis` >= 10000 | 기본값 유지 | 과단축 시 불필요한 페일오버 빈번 |
| Cache Hit Ratio >= 95% | `db.serverStatus().wiredTiger.cache` | 디스크 I/O 증가, 성능 저하 |
| `vm.swappiness` = 1 (MongoDB 서버) | DB 서버 안정성 | 스와핑 시 캐시 성능 급감 |
| THP = enabled (MongoDB 8.0+ 서버) | TCMalloc per-CPU 성능 | per-CPU 캐시 비활성화로 성능 저하 |
| `ulimit -n` = 1048576 (MongoDB 서버) | 모든 서버 공통 | 기동 경고, 커넥션 제한 |
| THP 영구 설정 적용 (MongoDB 서버) | IT ONE 변경 요청 완료 | 리부팅 후 THP 비활성화로 성능 저하 |
| systemd 서비스 LimitNOFILE/LimitNPROC/LimitFSIZE/LimitCPU override 설정 | 서비스 데몬 ulimit 적용 | limits.conf 무시되어 기본값으로 동작 |

---

## 부록 A. Java 버전 마이그레이션 정책

운영 중인 서비스의 Java 버전은 즉시 변경하지 않으며, 아래 원칙에 따라 점진적 전환을 진행함.

| 구분 | 정책 | 상세 |
|:---|:---|:---|
| **신규 프로젝트** | **Java 25 LTS 적용** | 2026년 6월 기준 최신 LTS로 빌드 타겟 설정. Spring Boot 4.x 이상 사용 권장 |
| **기존 운영 서비스** | **현행 유지** | 현재 사용 중인 Java 버전(15, 17)을 그대로 유지. 강제 업그레이드 지양 |
| **점진적 마이그레이션** | **각 팀 재량** | JDK 업데이트 시기 및 방식은 각 팀의 배포 주기, QA 여력, 의존성 호환성을 고려하여 자율 결정 |
| **권장 전환 순서** | Non-LTS → 최신 LTS | Java 15 (Non-LTS) 운영 서비스는 차기 정기 배포 주기에 맞춰 Java 25 LTS 전환을 권장. 단, 강제 일정은 없으며 시스템 안정성이 최우선 |
| **호환성 검증** | **필수** | JDK 버전 변경 시 애플리케이션 전체 회귀 테스트 및 GC 재튜닝 필수. 특히 G1 GC ↔ Parallel GC 전환 시 GC 로그 기반 성능 검증 필요 |

## 부록 B. 출처

- PostgreSQL 16/18 공식 문서: High Availability, Load Balancing, and Replication
- MongoDB 공식 문서: Replica Set Deployment Architectures, Production Considerations
- MongoDB 공식 문서: Transactions Production Considerations (Standalone = 트랜잭션 미지원)
- MongoDB 8.0 공식 문서: Release Notes, Compatibility Changes in MongoDB 8.0
- MongoDB 8.0 공식 문서: New Parameters (defaultMaxTimeMS, cacheSizePct, tcmallocEnableBackgroundThread)
- Apache Tomcat 9 공식 문서: Configuration Reference (Connector, Executor)
- IBM WebSphere Liberty 23.x 공식 문서: Server Configuration
- HikariCP 공식 문서: Configuration (maxLifetime, keepaliveTime, minimumIdle)
