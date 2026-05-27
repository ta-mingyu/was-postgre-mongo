# [인프라 설정 표준화 컨설팅] 사전 질문지 답변 현황 (CLS 팀)

## 1. WAS (CLS 시스템) - JVM 및 리소스 관리

### ⚙️ 핵심 인프라 설정 요약

| 분류 | 조사 항목 | 설정값 및 운영 현황 |
| :--- | :--- | :--- |
| **Runtime 환경** | **Java 버전** | • **Java 15.0.2** |
| **Thread 설정** | `maxThreads`<br>`minSpareThreads`<br>`acceptCount` | • **-1**<br>• **Default** (기본값)<br>• **Default** (기본값) |
| **Connection Pool** | `maximumPoolSize`<br>`minimumIdle`<br>`maxLifetime` | • **50**<br>• **0**<br>• **1800** |
| **Timeout 및 Pool 관리** | WAS-DB 간 타임아웃 | • `connectionTimeout`: 30s<br>• `reapTime`: 300<br>• `maxIdleTime`: 1800<br>• `minPoolSize`: 0 |
| **GC 알고리즘** | 적용 알고리즘 | • **Parallel GC** 사용 중 |

---

### Memory 전략 및 Peak 타임 사용 현황 분석

* **JVM Heap Size 설정:** `Xms 2048m` / `Xmx 2048m` (동일 할당)
* **Peak 타임 Runtime 메모리 사용률 (지표 요약):**
  * **ParOldGen (Old 영역):** 총 1,572,864K 중 1,418,698K 사용 (**★ 약 90% 임계치 도달**)
  * **PSYoungGen (Young 영역):** 총 458,752K 중 30,767K 사용 (Eden 6% / From 7% / To 0%)
  * **Metaspace 현황:** Used 157,772K / Committed 166,528K / Reserved 401,408K
    * *Class Space:* Used 16,196K / Committed 19,072K

---

## 2. DB 시스템 (PostgreSQL / MongoDB)
* **해당 사항 없음** (CLS 팀은 해당 데이터베이스를 전사 운영 구조에서 사용하지 않음)
