# [인프라 설정 표준화 컨설팅] 사전 질문지 답변 현황 (Platform Develop 팀)

## 1. WAS (Apache Tomcat) - JVM 및 리소스 관리

### 항목별 상세 설정 (나이스파크 vs 나이스차저)

| 조사 항목 | 나이스파크 (Nice Park) | 나이스차저 (Nice Charger) |
| :--- | :--- | :--- |
| **Thread 설정**<br>*(maxThreads, minSpareThreads, acceptCount)* | - **기본값** 사용 | - **기본값** 사용 |
| **Connection Pool 설정**<br>*(HikariCP / DBCP)* | - `maximum-pool-size`: 5<br>- `minimum-idle`: 2<br>- `max-lifetime`: 1,800,000 ms (30분) | - `maximum-pool-size`: 100(webapp) / 20(admin)<br>- `minimum-idle`: 5<br>- `max-lifetime`: 2,000,000 ms |
| **Runtime 환경**<br>*(Java 및 Tomcat 버전)* | - **Java 17**<br>- Spring Boot 3.5.3 내장 톰캣 | - **Java 15** (Tomcat 9)<br>- **Java 25** (Spring Boot 4.0.5 내장 톰캣) |
| **Memory 전략**<br>*(JVM Heap & Metaspace)* | - `heapPercent`: 70%<br>- `metaspace`: 128m<br>- `MaxMetaspaceSize`: 128m | - `Xms` / `Xmx`: 2048m / 2048m<br>- `MetaspaceSize`: 512m<br>- `MaxMetaspaceSize`: 512m |

### 공통 설정 및 운영 현황
* **GC (Garbage Collection) 알고리즘:** 전사 **G1 GC** 적용 중
* **Timeout 관리 전략:** WAS-DB 간 유휴 연결 정리를 위해 별도의 `wait_timeout` 없이 `maxLifetime`만 단독으로 관리 중

---

## 2. PostgreSQL - 관계형 데이터베이스 안정성

* **트랜잭션 패턴 (Read : Write)**
  * **7 : 3** 비율로 읽기 비중이 높음
* **쿼리 형태 및 Slow Query 현황**
  * **JOIN 및 복합 통계 쿼리:** 전체의 약 **2%** 수준으로 미미함
  * **Slow Query (3초 이상):** 현재 자주 발생하는 구간 **없음**
* **데이터 규모 및 파티셔닝**
  * 현재 파티셔닝을 적용한 테이블 **없음**
* **연결 방식 (Connection Management)**
  * WAS에서 직접 관리하지 않고, **WAS ↔ PgPool-II** 연계를 통해 커넥션 풀 운영
* **고가용성 (HA) 기준**
  * **RTO (복구 목표 시간):** 10초
  * **RPO (데이터 손실 허용 범위):** 5초
* **격리 수준 (Isolation Level)**
  * 특정 트랜잭션의 상위 격리 수준 사용 없이, PostgreSQL 기본값(**Read Committed**) 사용 중

---

## 3. MongoDB - 비정형 데이터 아키텍처

* **데이터 모델링 패턴**
  * Schema-less 활용도가 낮으며, **고정된 스키마 형식**의 데이터 위주로 적재
* **읽기/쓰기 비율 (Read : Write)**
  * **6 : 4** 비율로 운영 중
* **샤딩 및 데이터 증가량**
  * **샤딩 상태:** 현재 **미적용**이며, 향후 샤딩 도입 계획도 없음
  * **데이터 증가량:** 서비스 오픈 초기 단계로 인해 정확한 월간 데이터 증가량 추정 불가
* **인덱스 전략**
  * 동적 쿼리는 거의 없으며, **고정된 인덱스 필드 위주의 검색**이 대부분을 차지
* **배포 아키텍처**
  * **Replica Set** 구조로 운영
  * **멤버 구성:** Master 1개 / Slave 2개 / Arbiter 0개
* **Slow Query 모니터링**
  * 인덱스 미적용으로 인한 **COLLSCAN(풀 스캔) 발생 여부는 현재 주기적으로 체크하고 있지 않음**
