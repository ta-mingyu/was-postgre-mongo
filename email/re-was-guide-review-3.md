To: 데이타뱅크 조도형 차장
From: 김민규 (NICE인프라 IT기획실)
Subject: PostgreSQL 검수 의견 반영 결과 및 통합 표준 설정 규정 최종 검수 요청

---

안녕하십니까 데이타뱅크 조도형 차장님.

NICE인프라 IT기획실 김민규입니다.

지난번 보내주신 PostgreSQL 아키텍처 검수 의견, 정말 감사드립니다.

말씀해주신 6가지 사항을 전 수용하여 가이드라인에 즉시 반영하였습니다.

[반영 완료 사항]

1. 의사결정 흐름도 명칭 정비 (질문 항목 구체화)
2. work_mem 전 아키텍처 공통 안전 마진 적용 (산정 공식 안전 계수 3→8 확대)
3. max_connections 전 아키텍처 100 고정 (OOM 예방 및 PgPool 큐잉 구조 도입)
4. hot_standby_feedback, archive_mode 파라미터 신규 추가
5. PgPool num_init_children 비고 업데이트 (120 폭주 시 100 즉시 처리, 20 큐 대기)
6. Patroni max_wal_senders 전 RAM 스펙 5 고정

그 후 사내 회의를 통해 당사 인프라 표준 아키텍처가 확정되어,

기존 전체 아키텍처(Standalone, SR, PgPool+SR, Patroni, Sharded Cluster 등)를 다루던 문서에서 아래 두 가지로 범위를 확정 지었습니다.

[확정된 표준 아키텍처]

- PostgreSQL: PgPool-II + Streaming Replication (프로덕션 표준)
  ※ Standalone은 개발/테스트 환경에 한해서만 허용

- MongoDB: Replica Set PSS (Primary 1 + Secondary 2)
  ※ Standalone은 개발/테스트 환경에 한해서만 허용

이에 따라 기존 가이드라인을 WAS 표준 설정을 포함한
단일 확정본으로 통합 재구성하였습니다.

링크: [final-standard-guide.md 링크]

차장님께서 이미 검수 완료하신 PostgreSQL PgPool+SR 파라미터 설정과
신규 추가된 MongoDB Replica Set PSS 설정을 중심으로,
최종 배포본의 실무 적합성을 한 번 더 확인해 주시면 감사하겠습니다.

특히 다음 사항에 대한 고견을 부탁드립니다.

[검수 요청 포커스]

1. PostgreSQL: 차장님 피드백이 정확히 반영되었는지,
   PgPool 큐잉 구조(num_init_children 120 > max_connections 100)의 실무 운영 안정성

2. MongoDB: Replica Set PSS 프로덕션 설정값의 적정성
   (cacheSizeGB, writeConcern, readPreference, Profiling Level 등)

3. OS 커널 파라미터: DB서버를 효율적으로 운영하게 하기 위한 OS 커널 파라미터 값이 제대로 설정되어있는지

MongoDB Sharded Cluster 및 Patroni 아키텍처는 당사 표준에서 제외되었으므로 검토 대상에서 제외해도 무방합니다.

바쁘신 와중에도 연이은 검수 요청에 기꺼이 응해주셔서
진심으로 감사드립니다.

김민규 드림.
