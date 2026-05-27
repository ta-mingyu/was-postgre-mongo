# [인프라 설정 표준화 컨설팅] 사전 질문지 답변 현황 (주차서비스 팀)

## 1. WAS (Apache Tomcat) - JVM 및 리소스 관리

### ⚙️ 핵심 인프라 설정 요약

| 분류 | 조사 항목 | 설정값 및 운영 현황 |
| :--- | :--- | :--- |
| **Runtime 환경** | **Java / Tomcat 버전** | • **Java 15.0.2** (OpenJDK 64-Bit Server VM)<br>• **Apache Tomcat 9.0.70** |
| **Thread 설정** | `maxThreads`<br>`minSpareThreads`<br>`acceptCount` | • **500**<br>• **25**<br>• **미설정** (Tomcat 기본값 100 적용) |
| **Connection Pool** | `maximumPoolSize`<br>`minimumIdle`<br>`maxLifetime` | • **100** (maxTotal: 100)<br>• **100**<br>• **10,000 ms** (maxWaitMillis 연동) |
| **Timeout 설정** | `connectionTimeout`<br>WAS-DB 유휴 정리 | • **20,000 ms** (20초)<br>• DB2 환경으로 인해 별도 확인 필요 (IT운영실 요청 필요) |
| **GC 알고리즘** | 적용 알고리즘 | • **G1 GC** 사용 중 |

---

### Memory 전략 및 특이사항

* **JVM Heap Size 설정:** `Xms 2048m` / `Xmx 4096m` (가변 할당 구조)
* **메모리 및 타임아웃 미기입 항목 (확인 필요):**
  * Peak 타임 시 실제 사용률 및 Metaspace 사용 현황 데이터는 제외됨
  * DB2 데이터베이스 서버 권한 부재로 인해 DB 유휴 연결 정리 타임아웃(`maxLifetime`, `wait_timeout`) 설정을 확인하려면 IT운영실의 협조가 필요함

---

## 2. DB 시스템 (PostgreSQL / MongoDB)
* **해당 사항 없음** * 주차서비스는 **DB2**를 메인 데이터베이스로 사용하고 있으므로, PostgreSQL 및 MongoDB 관련 질문 답변은 제외됨
