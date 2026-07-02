# 디렉토리 구조

> 본 파일은 구조(무엇이 어디 있는가)를 담당한다. 매 수정/탐색 전 본다.

## 트리

```mermaid
graph TD
    ROOT["was-postgre-mongo/"] --> A[AGENTS.md<br/>목차 전용]
    ROOT --> T[todo.md<br/>작업 로드맵]
    ROOT --> PPT[Infrastructure_Standardization_Consulting.pptx]
    ROOT --> WS[server-spec.code-workspace]

    ROOT --> H[harness/<br/>에이전트 구동 규칙 + 메타데이터]
    ROOT --> R[reports/<br/>표준 가이드라인 산출물]
    ROOT --> ST[study/<br/>TA 기본 소양 학습 자료]
    ROOT --> RS[research/<br/>아키텍처 리서치]
    ROOT --> S[source/<br/>원천 데이터 읽기전용]
    ROOT --> E[email/<br/>컨설팅사 커뮤니케이션]

    R --> RF[final-standard-guide.md<br/>통합 확정본 정본]
    R --> RFD[final/<br/>서버별 실무 배포본 정본]
    R --> RH['{was,db}-standard-guide*' V1~V3<br/>참조용 히스토리]

    RFD --> RFD1[was.md]
    RFD --> RFD2[postgresql.md]
    RFD --> RFD3[pgpool-ii.md]
    RFD --> RFD4[mongodb.md]

    S --> S1[apache-tuning-guide.md]
    S --> S2[4개 팀 질문지]

    style A fill:#d4edda,stroke:#28a745
    style H fill:#cce5ff,stroke:#004085
    style S fill:#f8d7da,stroke:#721c24
    style RF fill:#d4edda,stroke:#28a745
    style RFD fill:#d4edda,stroke:#28a745
```

## 최상위 디렉토리 역할

| 경로 | 역할 | 비고 |
| :--- | :--- | :--- |
| `AGENTS.md` | 목차(Index). 본문 없음, harness/ 참조만 | 규격: 목차 전용 |
| `harness/` | 에이전트 구동 규칙, 산정 공식, 팀 메타, 함정 | 본 파일이 속한 곳 |
| `reports/` | 표준 가이드라인 산출물 | Report 본문은 항상 여기 |
| `reports/final/` | **서버별 정본**(운영자 배포용) | 가장 권위 있는 사본 |
| `research/` | 아키텍처 비교 리서치 (postgresql/, mongodb/, was/) | 출처 명시 의무 |
| `study/` | TA 기본 소양 학습 문서 (Explanation). 5개 도메인 근본 개념·트레이드오프 | 학습 시. 설정값 정본은 아님 |
| `source/` | 4개 팀 질문지 + Apache 튜닝 가이드 | **수정 금지**(읽기 전용) |
| `email/` | 컨설팅사(데이타뱅크) / DBA 리뷰 답변 메일 | |
| `todo.md` | 전체 작업 로드맵 + 진행 상태 | 작업 완료 시 체크박스 갱신 의무 |

## 산출물 권위 계층 (혼동 주의)

동일 가이드의 사본이 여러 군데에 존재한다. 우선순위:

```mermaid
graph TD
    A[1순위<br/>reports/final-standard-guide.md<br/>+ reports/final/*.md<br/>통합 확정 정본] --> B[2순위<br/>reports/{was,db}-standard-guide-v3.md<br/>개별 가이드 최신본 참조용]
    B --> C[3순위<br/>reports/...-v2.md / v1.md<br/>이전 버전 히스토리]
    style A fill:#d4edda,stroke:#28a745
    style C fill:#e8e8e8,stroke:#999
```

- **정본(final/) 변경 시**: 반드시 `reports/` 의 해당 개별 최신본(-v{N})도 같이 갱신하고 AGENTS.md 링크를 최신으로 맞춘다.
- harness/ 에는 Report 본문을 절대 두지 않는다(규칙·메타데이터만).

## 주요 산출물 엔트리포인트

- WAS 표준값 산출: `harness/was-rules.md` 의 산정 공식 + 매트릭스
- PostgreSQL/PgPool 표준값 산출: `harness/postgresql-rules.md`
- MongoDB 표준값 산출: `harness/mongodb-rules.md`
- 팀 현재 설정 비교 기준: `harness/team-profiles.md`
