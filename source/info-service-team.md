# [인프라 설정 표준화 컨설팅] 사전 질문지 답변 현황 (현금 정보계 시스템)

## 1. Runtime 환경 및 WAS 기본 정보
* **WAS 솔루션:** **IBM WebSphere Liberty v23.0.0.2 ND**
* **Java 환경:** **OpenJDK 15.0.2+7-27**
* **GC (Garbage Collection):** 모든 컨테이너 공통으로 **Parallel GC** (`-XX:+UseParallelGC`) 사용 중 (Throughput 중심)

---

## 2. Thread 및 Connection Pool 정책

### 🧵 Thread 관리 정책 (Liberty 동적 확장)
* **설정 상태:** `<executor name="Default Executor" coreThreads="-1" maxThread="-1" />`
* **운영 특징:** 특정 스레드 개수를 고정하지 않고, Liberty WAS가 실시간으로 스레드를 자동 확장/축소함. 각 컨테이너 시스템 자원의 한계치까지 유연하게 자원을 활용하는 구조임.

### 🔌 Connection Pool 설정 (Connection Manager)
* **Maximum Connections (maxPoolSize):** **50** (최대 50개 커넥션 허용)
* **Minimum Connections (minPoolSize):** **0** (유휴 상태 시 불필요한 DB 세션 점유를 방지하기 위해 최소값을 0으로 최적화)

---

## 3. 컨테이너별 메모리(JVM) 전략

주요 컨테이너는 고정 할당(Static) 방식을 채택하고 있으며, 그 외 4개 컨테이너는 동적 할당(Dynamic) 방식으로 이원화하여 운영 중입니다.

| 컨테이너명 | 할당 방식 | Heap 최소 (Xms) | Heap 최대 (Xmx) | Metaspace (Min) | Metaspace (Max) |
| :--- | :---: | :--- | :--- | :--- | :--- |
| **NIBS** | **고정 (Static)** | 8,192m (8G) | 8,192m (8G) | 2,048m (2G) | *미기재* |
| **CLS** | **고정 (Static)** | 2,048m (2G) | 2,048m (2G) | 256m (256M) | 16m |
| **NPS** | **고정 (Static)** | 1,024m (1G) | 1,024m (1G) | 256m (256M) | 16m |
| **PARTNER** | **동적 (Dynamic)** | 1,024m (1G) | 5,120m (5G) | 512m (512M) | 16m |
| **EAUDIT** | **동적 (Dynamic)** | 1,024m (1G) | 5,120m (5G) | 512m (512M) | 16m |
| **MOBILE** | **동적 (Dynamic)** | 2,048m (2G) | 4,096m (4G) | 512m (512M) | 16m |
| **POS** | **동적 (Dynamic)** | 1,024m (1G) | 2,048m (2G) | 256m (256M) | 16m |

---

## 4. Timeout 설정 (WAS-DB 유휴 연결 관리)
* **Connection Timeout:** **30초** (DB 연결 요청 시 최대로 대기하는 시간)
* **Max Idle Time (유휴 대기 시간):** **1,800초 (30분)** (연결 유지 후 해제 기준)
* **Reap Time (점검 주기):** **300초 (5분)** (유휴 커넥션 정리 및 풀 점검 사이클)

---

## 5. DB 시스템 (PostgreSQL / MongoDB)
* **해당 사항 없음** (현금 정보계 시스템은 해당 설문에서 WAS 환경 및 기본 인프라 정보만 제출함)
