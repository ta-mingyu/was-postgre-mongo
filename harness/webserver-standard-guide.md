# Web Server (Apache) 표준 튜닝 가이드 분석 결과

> 원천 파일: `source/apache-tuning-guide.md` (Apache 성능 튜닝 가이드 V3.1, 336행)
> 본 파일은 Apache 가이드의 구조와 핵심 원칙을 분석하여, 향후 WAS 표준 가이드라인 작성 시 설계 기준으로 활용한다.

---

## 1. 분석 목적

- 사내 Web Server 튜닝 가이드(Apache)의 구조, 공식, 전략 분기 로직을 분석
- 동일한 구조를 WAS Server 표준 가이드라인에 적용하기 위한 설계 청사진 확보
- 이후 DB 설정 표준 가이드라인에도 동일 프레임워크 적용 예정

---

## 2. Apache 가이드 핵심 원칙 (Summary)

| 원칙 | 내용 | WAS 가이드 적용 포인트 |
| :--- | :--- | :--- |
| **RAM 90% 활용** | 가용_RAM = Total_RAM x 0.9 (OS/커널 10% 예약) | JVM Heap = 컨테이너 RAM 50~70% 공식과 동일 맥락 |
| **ServerLimit <= 200** | 4 Core CPU 컨텍스트 스위칭 한계 | maxThreads 산정 시 CPU 코어 수 고려 필요 |
| **TPC <= 128 (안전)** | 외부 모듈(DB 드라이버, mod_security) 환경 락 경합 방지 | WAS thread pool 산정 시 DB 드라이버/커넥션 풀 연동 고려 |
| **TPC > 128 (극대화)** | 순수 리버스 프록시/SSL 종단 환경만 허용 | 순수 API 서버 vs DB 연동 서버 전략 분기 적용 |
| **Event MPM 기본** | KeepAlive를 Listener Thread가 관리, Worker 대비 2~5배 연결 수용 | NIO vs BIO 커넥터 선택, async servlet 지원 여부와 유사 |
| **KeepAliveTimeout 3** | HTTP/1.1 평균 요청 간격(2~4초) 하한선, 유휴 연결 빠른 정리 | connectionTimeout, keepAliveTimeout 산정 기준과 연관 |

---

## 3. 가이드 구조 분석 (WAS 가이드 설계 청사진)

Apache 가이드의 문서 구조를 WAS 가이드에 그대로 적용한다:

```
Apache 가이드 구조                  WAS 가이드 적용 구조
=====================              =====================
1. 핵심 원칙                  -->  1. 핵심 원칙 (JVM Heap 비율, GC 선택, Thread 수 산정)
2. 핵심 공식                  -->  2. 핵심 공식 (Heap 산정, Thread 수 = CPU * N, Pool size 공식)
3. MPM Prefork 매트릭스       -->  3. Tomcat (독립) 매트릭스 (RAM x Thread x Pool)
4. MPM Worker 매트릭스        -->  4. Spring Boot 내장 Tomcat 매트릭스
5. MPM Event 매트릭스         -->  5. WebSphere Liberty 매트릭스
6. 공통 설정                  -->  6. 공통 설정 (KeepAlive, Timeout, Logging)
7. OS 튜닝                   -->  7. OS/컨테이너 튜닝 (ulimit, sysctl, cgroup)
8. reload vs restart          -->  8. 무중단 배포/재시작 전략
9. 다중 인스턴스              -->  9. 다중 컨테이너/인스턴스 확장
```

---

## 4. 핵심 공식 추출

### 4.1 RAM 산정 공식

```
가용_RAM = Total_RAM x 0.9
MaxRequestWorkers = 가용_RAM / 프로세스당_비용
```

WAS 적용:
```
Heap_권장 = 컨테이너_RAM x 0.5 ~ 0.7
  (나머지는 OS, Metaspace, Thread Stack, Direct Buffer, Native Memory에 할당)
Thread_수 = CPU_코어수 x 8 ~ 20 (워크로드에 따라)
ConnPool_수 = Thread_수 x 0.5 ~ 1.0 (동시 DB 연결 비율)
```

### 4.2 전략 분기 로직

```
Apache: 순수 리버스 프록시/SSL 종단 --> 전략 A (자원 극대화, TPC>128)
        DB 드라이버/외부 모듈 사용 --> 전략 B (안전, TPC<=128)

WAS 적용:
  순수 API 서버 (DB 연결 없음)     --> 전략 A (Thread 높게, Pool 불필요)
  DB 연동 서버 (CRUD 위주)         --> 전략 B (Thread-Pool 1:1 매칭, 안전)
  혼합 워크로드 (API + DB + 외부)   --> 전략 C (모니터링 기반 동적 조정)
```

### 4.3 다중 인스턴스 확장 공식

```
Apache: 가용_RAM_인스턴스 = (Total_RAM x 0.9) / N
        ServerLimit_인스턴스 <= 200/N (CPU 총합 제약)
        
WAS 적용:
  Heap_컨테이너 = (Total_RAM x 0.9) / N - Metaspace - Overhead
  maxThreads_컨테이너 = CPU_코어 / N x 8~20
  주의: 컨테이너당 DB 연결 풀 합산이 DB max_connections 초과하지 않도록 제약
```

---

## 5. 매트릭스 구조 (환경별 설정 매트릭스)

Apache 가이드는 RAM 크기(4/8/16/32GB)를 행으로, 설정 파라미터를 열로 하는 매트릭스를 사용.

WAS 가이드 적용 시 매트릭스 구조:

| 컨테이너 RAM | Heap (Xms=Xmx) | Metaspace Min | Metaspace Max | maxThreads | maxPoolSize | GC | 비고 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 2 GB | 1024m | 128m | 256m | 100 | 10 | G1 GC | 최소 사양 |
| 4 GB | 2048m | 256m | 512m | 200 | 20 | G1 GC | 기본 |
| 8 GB | 4096m~5120m | 256m | 512m | 200 | 30 | G1 GC | 표준 |
| 16 GB | 8192m~10240m | 256m | 512m | 200~400 | 50 | G1 GC | 고성능 |
| 32 GB | 16384m~20480m | 512m | 1024m | 200~500 | 100 | G1 GC | 대규모 |

> 위 수치는 초기 설계안. 실제 값은 WAS 종류별(Tomcat/Spring Boot/Liberty) 상세 매트릭스에서 조정.

---

## 6. OS/컨테이너 튜닝 체계

Apache 가이드의 OS 튜닝 항목과 WAS 환경에서의 대응:

| Apache OS 튜닝 | 설정값 | WAS 환경 대응 |
| :--- | :--- | :--- |
| `vm.swappiness` | 10 | 컨테이너 환경에서도 동일 적용 (swap 최소화) |
| `fs.file-max` | 100000 | FD 한계 증가 (Tomcat maxConnections, DB 연결) |
| `net.ipv4.tcp_fin_timeout` | 15 | TCP 연결 정리 가속 |
| `net.ipv4.tcp_tw_reuse` | 1 | TIME_WAIT 소켓 재사용 |
| `net.core.somaxconn` | 2048 | ListenBacklog와 매칭 (acceptCount와 연관) |

WAS 추가 항목:
- `ulimit -n` (open files): FD 한계 (maxConnections + Pool size + Log files + buffer)
- `ulimit -u` (max processes): Thread 수 상한
- cgroup `memory.limit_in_bytes`: 컨테이너 메모리 제한 시 JVM 인식 보장

---

## 7. WAS 가이드라인 작성 시 참고 포인트

1. **Apache 가이드의 가장 큰 장점**: 환경(RAM 크기)별로 구체적인 숫자 매트릭스를 제공하여, 운영자가 계산 없이 바로 적용 가능
2. **전략 분기가 명확**: 순수 프록시 vs DB 연동을 명시적으로 구분
3. **검증 로직 포함**: Scoreboard 검증(SL x TL >= 총 연결 한계), Thrashing 방지 조건 등
4. **다중 인스턴스 환경까지 커버**: 단일 -> 다중 확장 시 자원 분할 공식 제공
5. **적용 방법(reload/restart) 명시**: 운영 편의성 확보

WAS 가이드에서도 동일하게:
- RAM/코어 수별 매트릭스 표 제공
- 워크로드 유형별 전략 분기
- 설정값 간 검증 로직 포함 (예: maxThreads >= maxPoolSize)
- 컨테이너 다중 배포 시 자원 분할 공식
- 재시작/무중단 배포 가이드
