# WAS 서버 표준 설정 가이드라인

> **설계 기준** : 3-Tier Capacity Rule (Web Server -> WAS -> DB), 계층별 독립 자원 산정

> **대상 플랫폼** : Apache Tomcat 9.x, Spring Boot 내장 Tomcat, IBM WebSphere Liberty 23.x 

> **인프라 기준** : 4 Core CPU / 4~32 GB RAM (Apache 튜닝 가이드 기준과 동일)

---

## 1. 계층별 핵심 파라미터 산정 가이드

```
Web Server (Apache)     --->     WAS Server     --->     Database
MaxRequestWorkers                maxThreads                maxPoolSize
       |                             |                          |
       +--- 처리량 비례 감소 ------->+                          |
                                      +--- 자원 독립 산정 ----->+
```

| 파라미터 | 산정 공식 및 기준 | 비고 | 역할 |
| :--- | :--- | :--- | :--- |
| **maxThreads** | `min(CPU_cores * 50, 500)` | 4 Core 기준 **200** (Tomcat 기본값) | 요청을 실제로 처리하는 작업 스레드(Worker Thread)의 최대 개수. 톰캣이 동시에 수행할 수 있는 연산의 상한선을 정의함 |
| **minSpareThreads** | `maxThreads / 8` | 200 스레드 기준 **25** 설정 | 트래픽 유입 전 항상 대기 상태로 유지하는 최소 스레드 수. 요청 도착 즉시 할당 가능한 예비 스레드를 확보하여 초기 응답 지연을 방지함 |
| **acceptCount** | `100` (고정) | 대기 큐 Backlog 표준값 | 모든 스레드가 처리 중일 때 OS TCP 레벨에서 임시로 대기시키는 최대 요청 수. 이 큐마저 포화되면 후속 요청은 즉시 거부(Connection Refused)되어 상위 프록시가 빠르게 실패를 인지하고 재시도 또는 차단(Backpressure)을 수행함 |
| **maxConnections** | `8,192` (고정) | NIO 커넥터 수용 한계 수치 | NIO 커넥터가 동시에 열어둘 수 있는 총 TCP 소켓(커넥션)의 상한선. 스레드 수보다 크게 설정하는 이유는 NIO가 Keep-Alive 상태의 유휴 소켓을 스레드 할당 없이 메모리에 대기시키기 때문임. 이 한계에 도달하면 후속 연결은 acceptCount 대기 큐로 넘어감 |
| **maxPoolSize (DB)** | `(DB_Cores * 2) + 여유량` | WAS 자원이 아닌 **DB 서버 스펙** 기준 산정 | WAS 인스턴스가 DB와 맺는 상시 TCP 커넥션(HikariCP)의 최대 개수. DB 디스크 I/O 및 메모리 한계를 보호하기 위해 WAS 스레드 수와 무관하게 DB 서버 코어 수 기준으로 독립 산정함. 과다 설정 시 DB Lock 경합 및 전체 TPS 저하를 유발함 |

> **핵심 주의**: 모든 계층에 동일한 선형 비율(예: 1/4)을 적용하는 방식은 **아키텍처 Anti-Pattern**임.
> 각 계층은 자원 비용 구조가 근본적으로 다르므로 독립적인 산정 기준을 적용해야 함.

---

## 2. 인프라 자원 스펙별 표준 설정값 (인스턴스당)

### 2.1 인스턴스별 설정 매트릭스

| 호스트 RAM | 인스턴스 수 | 인스턴스당 Heap (`Xms=Xmx`) | GC 전략 | 인스턴스당 maxPoolSize |
| :--- | :---: | :--- | :--- | :---: |
| **4 GB** | 1 | **2,048m** | **Parallel GC** (`-XX:+UseParallelGC`) | 20 |
| **8 GB** | 2 | **2,560m** | **Parallel GC** (`-XX:+UseParallelGC`) | 20 ~ 30 |
| **16 GB** | 3 | **3,413m** | **Parallel GC** (`-XX:+UseParallelGC`) | 20 ~ 30 |
| **32 GB** | 4 | **5,120m** | **G1 GC** (`-XX:+UseG1GC`) | 30 |

### 2.2 Metaspace 공통 설정 및 Heap 분할 원칙

- **Metaspace (모든 스펙 공통)** : Min `256m` / Max `512m` 고정 (역전 현상 및 입력 오류 방지)
- **Heap 분할 공식** : `floor(호스트_RAM * 0.625) / 인스턴스_수`
  - 예외: 4 GB 단일 인스턴스 호스트는 OS 가용량 확보를 위해 보수적으로 **50%** (`2,048m`) 할당

> **16 GB / 3인스턴스 산출 검증** : `floor(16,384m * 0.625) / 3 = floor(10,240m) / 3 = 3,413m`

### 2.3 GC 전략 선택 가이드

#### Parallel GC -- 호스트 4 GB / 8 GB / 16 GB (인스턴스당 Heap 2 GB ~ 3.5 GB)

- 소형 힙 환경에서 G1GC의 불필요한 Region 메타데이터 오버헤드를 제거
- 제한된 힙에서 G1의 1M~2M Region 분할로 인한 Humongous Object 이슈를 회피
- 제한된 CPU 및 메모리 자원 내에서 최고의 처리량(TPS)을 확보

#### G1 GC -- 호스트 32 GB 이상 (인스턴스당 Heap 5 GB 이상, Scale-out)

- 이 규모에서는 G1의 최소 Region 크기가 4 MB 이상 확보되어 Humongous Object 리스크가 감소
- G1의 핵심 강점인 Heap 크기 대비 Stop-the-World 시간 제어능력이 실질적인 이점을 제공
- 인스턴스당 Heap이 5 GB를 초과하는 경우 **ZGC (Java 21+)** 로의 추가 전환을 검토

> **GC 결정 기준선**: Heap <= 4 GB --> **Parallel GC** / Heap > 4 GB --> **G1 GC** (또는 Java 21+의 경우 ZGC)

---

## 3. 타임아웃 캐스케이드 (Timeout Cascade) 표준값

타임아웃 캐스케이드는 **상위 계층이 하위 계층보다 먼저 연결을 종료**하도록 설정하여,
프록시 레이스 컨디션(간헐적 502/503 에러) 및 무효 커넥션 예외를 차단함.

### 3.1 Web Server (Apache) -> WAS (Tomcat / Liberty)

| 파라미터 | 표준값 | 적용 범위 |
| :--- | :--- | :--- |
| **Apache keepAliveTimeout** | `3s` | Apache httpd 설정 |
| **Apache ProxyPass ttl** | `10s` (권장, default 60s) | `httpd.conf` ProxyPass 지시자 |
| **WAS keepAliveTimeout** | `15s` | Spring Boot / 독립형 Tomcat |

> **핵심 제약**: `ProxyPass ttl`은 WAS `keepAliveTimeout` 만료 이전에 유휴 커넥션을 관리해야 함.
> 리버스 프록시 아키텍처에서 상위 계층(Apache)이 하위 계층(WAS)보다 **먼저** 연결 종료를 개시해야 함.
> 이 순서가 위반되면 Apache가 이미 종료된 WAS 연결을 재사용하려 시도하여 간헐적 **502/503** 응답이 발생함.

```apache
# httpd.conf - ProxyPass 설정 예시
ProxyPass / http://was:8080/ ttl=10 keepalive=On
ProxyPassReverse / http://was:8080/
```

### 3.2 WAS -> DB

#### 3.2.1 WAS 커넥션 풀 공통 설정 (HikariCP 기준)

모든 데이터베이스 연동에 공통 적용되는 HikariCP 파라미터 표준값임.

| 파라미터 | 표준값 | 역할 | 설계 근거 (Rationale) |
| :--- | :--- | :--- | :--- |
| **HikariCP maxLifetime** | `1,620,000ms` (27분) | 풀(Pool) 내부 커넥션의 최대 생존 수명 정의 | 하위 모든 DB의 유휴 세션 제한(30분) 및 사내망 방화벽 임계치(30분)보다 3분 먼저 커넥션을 스스로 폐기 및 재생성(Recycle)하여, DB 또는 방화벽이 먼저 연결을 강제 차단함에 따라 발생하는 커넥션 단절 예외(**Connection reset**)를 원천 차단. PgPool-II 경유 시 `child_life_time`(28분)과의 1분 갭도 확보 |
| **HikariCP connectionTimeout** | `30,000ms` (30초) | 애플리케이션 스레드가 풀에서 커넥션을 획득하기 위해 대기하는 최대 시간 | DB 병목 시 스레드가 무한 대기하여 WAS 전체 스레드 풀이 고갈되는 현상을 방지하는 최후의 안전장치(**Fail-Fast**) |
| **HikariCP minimumIdle** | `= maxPoolSize` | 상시 유지할 가용 유휴 커넥션의 최소 개수를 `maxPoolSize`와 동일하게 강제 | 트래픽 스파이크 유입 시 커넥션 풀이 동적으로 축소 및 확장되면서 발생하는 CPU 및 네트워크 핸드셰이크 오버헤드를 제거하는 **Fixed-size pool** 전략 |
| **HikariCP keepaliveTime** | `60,000ms` (1분) | 유휴 커넥션의 유효성을 검증하기 위해 주기적으로 DB에 송신하는 초경량 생존 신호(Ping) 주기 | 중간 방화벽의 세션 유휴 무단 차단(**Silent Drop**)을 예방하고, 일시적 네트워크 순단 발생 시 죽은 커넥션(**Stale Connection**)을 최대 1분 이내에 신속히 감지하여 제거 (장애 복구 골든타임 최소화) |

> **keepaliveTime 설정 필수사항**:
>
> WAS와 DB 사이에 방화벽이 존재하는 환경에서는 방화벽의 유휴 세션 타임아웃이 양 단말에 통보 없이
> 커넥션을 **Silent Drop**할 수 있음. 다음 지침을 준수해야 함.
>
> - `keepaliveTime`은 방화벽의 유휴 타임아웃보다 **짧은 값**으로 설정
> - 표준값: **60,000ms (1분)** -- 장애 복구 골든타임 최소화 관점에서 산정
> - JDBC 4+ 드라이버를 기준으로 함. `connectionTestQuery`를 별도 지정하지 말고 드라이버 자체 `isValid()` 메커니즘을 활용할 것
> - **Fixed-size pool** (`minimumIdle = maxPoolSize`) 환경은 유효하지 않은 커넥션이 풀에 잔류할 위험이 특히 크므로 keepalive 검증이 **필수**

---

#### 3.2.2 데이터베이스 벤더별 유휴 세션 제한 설정

사내망 Established 방화벽 임계치(**30분 / 1,800초**)와 연동되는 각 DBMS별 유휴 세션 강제 종료 파라미터임.
HikariCP `maxLifetime`(27분)이 아래 모든 DB의 세션 제한(30분)보다 선행하여 커넥션을 Recycle함으로써
타임아웃 캐스케이드 일관성을 확보하는 **최하부 거버넌스 한계선**임.

| DBMS | 파라미터 | 표준값 | 역할 |
| :--- | :--- | :--- | :--- |
| **PostgreSQL** | `idle_session_timeout` | `1,800,000ms` (30분) | 클라이언트 유휴 세션을 강제 종료하여 연결 누수(Leak) 및 좀비 세션으로부터 DB 프로세스 자원 및 메모리를 보호 |
| **MongoDB** | `localLogicalSessionTimeoutMinutes` | `30` (30분) | 서버 내 **논리 세션(Logical Session) 상태의 유휴 만료 규격**을 정의하여 세션 누적으로 인한 서버 자원 고갈을 방지. 이 파라미터는 물리적인 TCP 소켓 커넥션을 강제 차단하는 타임아웃이 아니며, MongoDB 서버 프로세스가 추적하는 세션 레코드의 논리적 만료 시간을 의미함 |
| **DB2** | `CONNECTIONIDLETIME` (LUW) / `IDTHTOIN` (z/OS) | `30 MINUTE` / `1,800s` | 유휴 세션의 DB 엔진 자원(Memory, Lock) 점유를 해제하여 전체 가용성 확보 |

> **MongoDB 물리적 유휴 커넥션 관리 (보충 설명)**:
>
> `localLogicalSessionTimeoutMinutes`는 논리 세션의 만료 규격일 뿐, 물리적 TCP 커넥션 자체의
> 유휴 상태를 직접 관리하지 않음. 물리적 유휴 커넥션의 수명 및 회수는 **드라이버단의 `maxIdleTimeMS` 설정**
> 을 통해 제어되므로, HikariCP의 `maxLifetime` 및 `keepaliveTime`과 연동하여 커넥션 풀 수준에서
> 관리되어야 함. 이중 구조(논리 세션 / 물리 커넥션)를 가지는 MongoDB의 특성상 두 계층을 혼동하지 않도록 주의.

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
> HikariCP가 모든 하위 계층(DB, 방화벽)보다 **3분 먼저** 커넥션을 폐기 및 재생성(Recycle)함으로써
> DB 또는 방화벽에 의한 강제 차단으로 발생하는 **Connection reset** 예외를 원천 차단함.

---

## 4. WAS 벤더별 실무 설정 스크립트 및 프로퍼티

### 4.1 Apache Tomcat 9.x (독립형)

#### `server.xml` -- Connector 설정 (4 Core / 8 GB 호스트 / 인스턴스당 2,560m)

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

#### `setenv.sh` -- JVM 옵션 (8 GB 호스트 / 인스턴스당 Heap 2.5 GB 기준)

```bash
JAVA_OPTS="-Xms2560m -Xmx2560m \
           -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m \
           -XX:+UseParallelGC \
           -XX:+HeapDumpOnOutOfMemoryError \
           -XX:HeapDumpPath=/var/log/tomcat/heapdump.hprof \
           -Xlog:gc*:file=/var/log/tomcat/gc.log:time,uptime,level,tags:filecount=10,filesize=50M"
```

---

### 4.2 Spring Boot 내장 Tomcat (Spring Boot 3.x)

#### `application.yml`

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

#### JVM 옵션 (8 GB 호스트 / 인스턴스당 Heap 2.5 GB 기준)

```bash
JAVA_OPTS="-Xms2560m -Xmx2560m \
           -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m \
           -XX:+UseParallelGC \
           -XX:+HeapDumpOnOutOfMemoryError \
           -Xlog:gc*:file=/var/log/app/gc.log:time,uptime,level,tags:filecount=10,filesize=50M"
```

> **프로퍼티 검증**:
> - `server.tomcat.threads.max` / `min-spare` : Spring Boot 2.x/3.x 표준, **int** 타입
> - `server.tomcat.connection-timeout` : **Duration** 타입, `20s`로 표기
> - `server.tomcat.keep-alive-timeout` : **Duration** 타입, Spring Boot 3.1+ 지원, `15s`로 표기
> - `server.tomcat.max-keep-alive-requests` : **int** 타입, Spring Boot 3.x 지원
> - Spring Boot의 `min-spare` 기본값은 **10** (독립형 Tomcat의 25보다 낮음). 정상 동작임.

> **하위 호환성 주의 (Java 15 / Spring Boot 2.x 환경)**:
>
> CL플랫폼, 주차서비스, 현금정보계 팀은 현재 Java 15(Non-LTS) 및 Spring Boot 2.x 환경을 유지하고 있음.
> 본 가이드의 4.2절 프로퍼티 및 Duration 표기법(`20s`, `15s`)은 **Spring Boot 3.x 전용 규격**으로 작성되었음.
> Boot 2.x 환경에 그대로 적용 시 파싱 에러가 발생할 수 있으므로, 해당 팀은 아래와 같이 역매핑하여 적용해야 함.
>
> **프로퍼티별 지원 버전 확인 (적용 전 세부 패치 버전 필수 확인)**:
>
> | 프로퍼티 | Spring Boot 2.x 지원 범위 | 비고 |
> | :--- | :--- | :--- |
> | `server.tomcat.keep-alive-timeout` | **2.3.0+** 지원 | 해당 팀의 Boot 버전이 2.3.0 미만인 경우 `TomcatServletWebServerFactory` Bean으로 프로그래밍 방식 설정 필요 |
> | `server.tomcat.max-keep-alive-requests` | **2.4.0+** 지원 | 해당 팀의 Boot 버전이 2.4.0 미만인 경우 동일하게 Bean 오버라이드 필요 |
> | `server.tomcat.connection-timeout` | 2.x 전 버전 지원 | Duration 표기 대신 밀리초(ms) 정수로 변환 필요 |
>
> **Breaking Change 주의 (Spring Boot 2.4 경계)**:
>
> Spring Boot **2.4.0**을 기점으로 톰캣 스레드 제어 프로퍼티 키가 **파괴적 변경(Breaking Change)** 되었음.
> 역매핑 적용 시 반드시 현재 운영 중인 Boot 세부 버전을 확인한 후 올바른 키를 사용해야 함.
>
> | Boot 3.x (본 가이드) | Boot 2.4+ | Boot 2.3 이하 | 비고 |
> | :--- | :--- | :--- | :--- |
> | `server.tomcat.threads.max` | `server.tomcat.threads.max` | `server.tomcat.max-threads` | **2.4 미만에서는 `max-threads` 키 사용**. 2.4+ 에서 `threads.max`로 변경됨 |
> | `server.tomcat.threads.min-spare` | `server.tomcat.threads.min-spare` | `server.tomcat.min-spare-threads` | 동일한 Breaking Change 적용 |
> | `server.tomcat.connection-timeout: 20s` | `20000` (ms 정수) | `20000` (ms 정수) | Boot 2.x에서는 Duration 문자열 대신 밀리초 정수로 변환 |
> | `server.tomcat.keep-alive-timeout: 15s` | `15000` (ms 정수, **2.3.0+**) | Bean 오버라이드 필요 | Boot 2.3.0부터 프로퍼티 직접 지원 |
> | `server.tomcat.max-keep-alive-requests: 100` | `100` (**2.4.0+**) | Bean 오버라이드 필요 | Boot 2.4.0부터 프로퍼티 직접 지원 |

---

### 4.3 IBM WebSphere Liberty 23.x

#### `server.xml` (4 Core / 8 GB 호스트 / 인스턴스당 2,560m)

```xml
<httpEndpoint id="defaultHttpEndpoint" host="*" httpPort="9080" httpsPort="9443">
  <httpOptions keepAliveTimeout="15s" />
</httpEndpoint>

<executor id="defaultExecutor" coreThreads="8" maxThreads="200" />

<connectionManager id="defaultConnectionManager"
                   maxPoolSize="15"
                   minPoolSize="15"
                   connectionTimeout="30s"
                   maxIdleTime="900s"
                   agedTimeout="1620s"
                   reapTime="60s"
                   purgePolicy="FailingConnectionOnly" />
```

#### 방화벽 임계치(TCP 30분 / 1,800초) 검증 결과 반영

> 기존 `maxIdleTime="1800s"`는 방화벽의 TCP 유휴 타임아웃(1800초)과 정확히 일치하여,
> 방화벽이 커넥션을 Drop하는 시점과 Liberty가 커넥션을 정리하려는 시점 사이에
> **Race Condition 구간**이 존재했음.
> Fixed-size pool (`minPoolSize=15` = `maxPoolSize=15`) 환경에서는
> 방화벽에 의해 무단 차단된 커넥션이 풀에 잔류하여 애플리케이션 오류를 유발할 수 있음.

| 파라미터 | 변경 전 | 변경 후 | 설계 근거 |
| :--- | :--- | :--- | :--- |
| `maxIdleTime` | 1800s | **900s** | 방화벽 타임아웃(1800s)의 50% 수준으로 단축. 유휴 커넥션을 방화벽 Drop 이전에 풀에서 안전하게 제거 |
| `agedTimeout` | (미설정) | **1620s** | 커넥션 최대 수명을 PG `idle_session_timeout`(1800s)보다 180초(3분) 짧게 설정하여 DB 서버 측 강제 차단 방지 |
| `reapTime` | 300s | **60s** | 풀 정리(Reaper) 주기를 60초로 단축하여 `maxIdleTime`/`agedTimeout` 도달 커넥션을 신속히 회수 |

#### `jvm.options` -- JVM 옵션 (8 GB 호스트 / 인스턴스당 Heap 2.5 GB 기준)

```text
-Xms2560m
-Xmx2560m
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m
-XX:+UseParallelGC
-XX:+HeapDumpOnOutOfMemoryError
-Xlog:gc*:file=/var/log/wlp/gc.log:time,uptime,level,tags:filecount=10,filesize=50M
```

> **JVM 벤더(배포판)별 GC 옵션 호환성 주의**:
>
> 가이드에 기재된 JVM 옵션(`-XX:+UseParallelGC`, `-Xlog:gc*` 등)은 **HotSpot 엔진**(Eclipse Temurin 등) 기준임.
> 가동 환경이 **IBM Semeru Runtime**(OpenJ9 엔진) 기반일 경우, HotSpot 전용 GC 옵션이 인식되지 않거나 무시됨.
> 해당 벤더 스펙에 맞는 GC 정책 옵션을 별도 확인 후 적용해야 함.
>
> | 항목 | HotSpot (Temurin 등) | OpenJ9 (IBM Semeru) |
> | :--- | :--- | :--- |
> | GC 정책 | `-XX:+UseParallelGC` | `-Xgcpolicy:gencon` (Gencon 권장) |
> | GC 로그 | `-Xlog:gc*:file=...` | `-Xverbosegclog:/var/log/wlp/gc.log` (또는 `-Xgc:verbose`) |
> | Heap Dump | `-XX:+HeapDumpOnOutOfMemoryError` | `-XX:+HeapDumpOnOutOfMemoryError` (호환) |

> **Liberty Executor 참고**: Liberty의 executor는 기본적으로 Auto-tuning (`maxThreads=-1`)을 사용함.
> 컨테이너 환경에서는 CPU 자원 인식 오류 방지를 위해 명시적 설정을 권장함.
> - `coreThreads` = CPU 코어 수 * 2
> - `maxThreads` = CPU 코어 수 * 50 (Tomcat 기준선과 동일)

---

## 5. 공유 DB 환경 커넥션 풀 가용 가이드 (70% Ceiling Rule)

다수의 서비스가 단일 DB 인스턴스를 공유하는 환경에서, 풀 사이즈를 제어하지 않으면
특정 애플리케이션의 트래픽 스파이크가 타 서비스의 커넥션을 고갈시킬 수 있음.
**70% Ceiling Rule**은 서비스 간 장애 전파를 차단하기 위한 자원 격리(Partitioning) 가이드라인임.

### 5.1 핵심 원칙

```
DB max_connections = 1,000 (DB 서버 설정)
      |
      +-- 30% 예약 (300): 관리자 세션, 모니터링, pg_terminate_backend, 긴급 접속
      |
      +-- 70% 가용 (700): 애플리케이션 커넥션 풀 전체 합산 상한
             |
             +-- 팀별 쿼터 할당 (서비스 중요도 + 트래픽 가중치 기반)
```

> **절대 제약**: 모든 애플리케이션의 `maxPoolSize` 합산값은 `DB max_connections * 0.7`을 **초과해서는 안 됨**

### 5.2 팀별 설정 점검 및 최종 확정값

| 팀 / 서비스 | 현행 Java | 마이그레이션 목표 | 현행 풀 설정 | **최종 확정 maxPoolSize** | 보정 사유 |
| :--- | :---: | :---: | :---: | :---: | :--- |
| **플랫폼개발 (Nice Park)** | 17 | **현행 유지** | 5 | **20** | 기존 풀 과소 설정으로 인한 처리량 병목 개선 |
| **플랫폼개발 (Nice Charger)** | 15, 25 | **현행 유지** | 100 / 20 | **20 ~ 30** | 웹 풀 100을 20~30으로 축소 (공유 DB 보호) |
| **CL플랫폼 (CLS 전용)** | 15.0.2 | **현행 유지** | 50 | **15** | 현금정보계와 동일 서버 사용. 인스턴스당 15로 제한. `maxThreads=200` 적용 |
| **주차서비스 (Tomcat 9.x)** | 15.0.2 | **현행 유지** | 100 | **20 ~ 30** | 과대 설정 축소 (공유 DB 리소스 고갈 방지) |
| **현금정보계 (Liberty 23.x)** | 15.0.2 | **현행 유지** | 50 | **15** | 7개 컨테이너 다중화 환경. 컨테이너당 15 (총 7 x 15 = 105) |

### 5.3 Java 버전 마이그레이션 정책

> 운영 중인 서비스의 Java 버전은 즉시 변경하지 않으며, 아래 원칙에 따라 점진적 전환을 진행함.

| 구분 | 정책 | 상세 |
| :--- | :--- | :--- |
| **신규 프로젝트** | **Java 25 LTS 적용** | 2026년 6월 기준 최신 LTS로 빌드 타겟 설정. Spring Boot 4.x 이상 사용 권장 |
| **기존 운영 서비스** | **현행 유지** | 현재 사용 중인 Java 버전(15, 17, 25)을 그대로 유지. 강제 업그레이드 지양 |
| **점진적 마이그레이션** | **각 팀 재량** | JDK 업데이트 시기 및 방식은 각 팀의 배포 주기, QA 여력, 의존성 호환성을 고려하여 자율 결정 |
| **권장 전환 순서** | Non-LTS -> 최신 LTS | Java 15 (Non-LTS) 운영 서비스는 차기 정기 배포 주기에 맞춰 Java 25 LTS 전환을 권장. 단, 강제 일정은 없으며 시스템 안정성이 최우선 |
| **호환성 검증** | **필수** | JDK 버전 변경 시 애플리케이션 전체 회귀 테스트 및 GC 재튜닝 필수. 특히 G1 GC <-> Parallel GC 전환 시 GC 로그 기반 성능 검증 필요 |

---

## 6. OS 커널 최적화 설정 파라미터

WAS 인스턴스가 실행되는 모든 인프라 호스트에 필수 반영해야 할 OS 수준 설정값임.
파라미터의 기술적 성격에 따라 **성향 제어형**(환경 무관 고정 적용)과 **용량 제한형**(인프라 규모에 따라 가변 조정)으로 분류함.

---

### 6.1 성향 제어형 파라미터 (호스트 스펙 및 인스턴스 개수와 무관하게 고정 적용)

```ini
# /etc/sysctl.d/99-was-tuning.conf -- 성향 제어형 (전 환경 공통 고정값)
vm.swappiness = 10
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_tw_reuse = 1
```

| 파라미터 | 고정값 | 설계 근거 |
| :--- | :--- | :--- |
| **vm.swappiness** | `10` | JVM 환경에서 물리 메모리(RAM) 고갈 전 디스크 스와핑(Swap) 발생으로 인한 GC 성능 마비 및 시스템 정지(STW) 리스크 최소화 |
| **net.ipv4.tcp_fin_timeout** | `15` | 연결 종료 후 소켓 회수 타임아웃을 단축하여 불필요한 네트워크 자원 점유 시간 최소화 |
| **net.ipv4.tcp_tw_reuse** | `1` | TIME_WAIT 상태의 소켓을 안전하고 신속하게 재사용하여 대량의 커넥션 요청 시 발생할 수 있는 로컬 포트 고갈(Ephemeral Port Exhaustion) 차단 |

---

### 6.2 용량 제한형 파라미터 (호스트 스펙 및 인스턴스 개수에 따라 가변 조정)

```ini
# /etc/sysctl.d/99-was-tuning.conf -- 용량 제한형 (인프라 규모별 확장 필요)
fs.file-max = 100000
net.core.somaxconn = 2048
```

```bash
# 실행 셸 또는 기동 스크립트에 적용 (ulimit) -- 용량 제한형
ulimit -n 65536          # open files (maxConnections + Pool + Log + Buffer)
ulimit -u 4096           # max processes (= threads)
```

| 파라미터 | 설계 근거 |
| :--- | :--- |
| **fs.file-max** | 모든 소켓 통신을 파일로 취급하는 리눅스 특성을 감안, 대규모 동시 접속자 유입 시 **Too many open files** 에러 및 인스턴스 다운 방지 |
| **net.core.somaxconn** | OS 커널 레벨의 TCP Listen Backlog 큐를 확장하여, 트래픽 스파이크 발생 시 WAS 엔진에 도달하기 전 OS 관문에서 패킷이 Drop되는 현상 방지 |
| **ulimit -n** (open files) | 모든 소켓 통신을 파일로 취급하는 리눅스 특성을 감안, 대규모 동시 접속자 유입 시 **Too many open files** 에러 및 인스턴스 다운 방지 |
| **ulimit -u** (max processes) | 단일 호스트 내 멀티 인스턴스 구동 및 JVM 내부 백그라운드 관리 스레드 폭증 시 OS의 스레드 생성 한계 제한 리스크 방어 |

#### 인프라 규모별 표준화 Matrix

> 본 가이드라인에 명시된 용량 제한형 파라미터의 기본값은 **소형 VM / 컨테이너 환경 (표준 Baseline)** 기준임.
> 호스트 RAM 64 GB 이상, 단일 호스트에 WAS 인스턴스 4개 이상이 고밀도로 구동되는
> **초대형 단일 호스트 / Bare-Metal 환경 (확장형)** 에서는 아래 확장값을 적용해야 함.

| 파라미터 | 표준 Baseline | 확장형 (64 GB+ / 인스턴스 4개+) | 적용 영역 |
| :--- | :---: | :---: | :--- |
| **fs.file-max** | `100,000` | `500,000 ~ 1,000,000` | `sysctl.conf` |
| **net.core.somaxconn** | `2,048` | `4,096` | `sysctl.conf` |
| **ulimit -n** (open files) | `65,536` | `131,072` | 실행 스크립트 (`ulimit`) |
| **ulimit -u** (max processes) | `4,096` | `16,384` | 실행 스크립트 (`ulimit`) |

> **적용 기준**: 확장형 수치는 호스트 RAM 64 GB 이상, WAS 인스턴스 4개 이상이 동시 구동되는 환경에 해당함.
> 이 기준에 미달하는 환경에서는 표준 Baseline 값을 그대로 적용함.

> **※ OS 커널 `somaxconn`과 WAS `acceptCount` 연동 지침**:
>
> 확장형 환경에서 OS 커널의 `net.core.somaxconn`을 **4,096**으로 상향하더라도, 실제 포트의 Listen Backlog는
> 애플리케이션의 설정과 연동되므로 **`min(acceptCount, somaxconn)`** 으로 결정됨.
> 따라서 OS 커널 파라미터 상향 시 Tomcat 및 Liberty의 `acceptCount` 설정값도 매칭하여 상향
> (**권장: 500 ~ 1,000**)해야 정책이 온전히 작동함.

---

## 7. 검증 체크리스트

| 검증 항목 | 충족 조건 | 위반 시 영향 |
| :--- | :--- | :--- |
| `Xms` = `Xmx` | 프로덕션에서 반드시 동일하게 설정 | Heap 리사이즈 시 GC Pause 발생 |
| Heap < Container RAM * 0.7 | OOM 방지 | Metaspace, Thread Stack, Native Memory 부족 |
| 인스턴스당 Heap = 호스트 RAM * 0.625 / N | 다중 인스턴스 분할 원칙 | 단일 인스턴스가 호스트 RAM 과점유 |
| `maxPoolSize` <= 30 (기본) | 인스턴스당 기본 표준 준수 | DB 리소스 고갈, Lock 경합 |
| Sum(`maxPoolSize`) <= DB `max_conn` * 0.7 | 공유 DB 70% Ceiling Rule | 타 서비스 커넥션 고갈, 장애 전파 |
| `maxLifetime` < 각 DB별 유휴 세션 제한값 | 커넥션 무효화 방지 | DB 또는 방화벽이 먼저 연결을 강제 종료하여 애플리케이션 단에서 커넥션 단절 예외(Connection reset) 발생. (PG: `idle_session_timeout`, Mongo: `localLogicalSessionTimeoutMinutes`, DB2: `CONNECTIONIDLETIME` 규격 준수 확인) |
| `ProxyPass ttl` < WAS `keepAliveTimeout` | 프록시 레이스 컨디션 방지 | 간헐적 502/503 에러 발생 |
| `minimumIdle` = `maxPoolSize` | Fixed-size pool 유지 | 풀 축소/확장 오버헤드 발생 |
| GC 로그 활성화 | 모든 WAS 인스턴스에서 필수 | 장애 발생 시 원인 분석 불가 |
| Metaspace Max >= Min | 역전 현상 방지 | 현금정보계팀 이슈 재발 (16m < 256m) |
| `maxThreads` > 0 | CL플랫폼팀 `-1` (무제한) 설정 금지 | Backpressure 부재, 리소스 고갈 |
| `leakDetectionThreshold` 활성화 | 권장값 **60,000ms** | 커넥션 누수 무감지 |
