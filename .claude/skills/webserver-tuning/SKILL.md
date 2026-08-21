---
name: webserver-tuning
description: Web 서버(Apache HTTP Server 2.4) 표준값 산정·튜닝 절차. Event/Worker/Prefork MPM 선택과 매트릭스, ServerLimit(<=200)/ThreadsPerChild(안전 128 이하) 산정, 전략 분기(순수 역방향 프록시/SSL 종단 vs 외부 모듈·DB 드라이버 연동), KeepAliveTimeout 3, 다중 인스턴스 확장 공식, Scoreboard 검증을 다룬다. Apache 튜닝값 산정, MPM 구성 검토, Web 서버 가이드 갱신이나 새 서버 스펙 산정을 할 때 반드시 이 스킬을 사용할 것. WAS/JVM 튜닝은 was-tuning을 사용.
---

# Web 서버(Apache) 튜닝

## 다루는 것

Apache HTTP Server 2.4 표준값 산정. 지식 3층 구조:

| 층 | 파일 | 역할 |
| :--- | :--- | :--- |
| 원본(읽기 전용) | `source/apache-tuning-guide.md` | 사내 Web Server 튜닝 가이드 V3.1. **수정 금지** |
| 청사 | `harness/webserver-standard-guide.md` | 핵심 원칙·공식 추출(본 스킬의 근거) |
| 정본 | `reports/final/web.md` | 운영자 배포 산출물 |

## 핵심 공식 (요약 — 정본은 webserver-standard-guide.md §4)

```
가용_RAM = Total_RAM * 0.9          (OS/커널 10% 예약)
MaxRequestWorkers = 가용_RAM / 프로세스당_비용
ServerLimit <= 200                   (4 Core 컨텍스트 스위칭 한계)
전략 분기:
  순수 역방향 프록시/SSL 종단  -> ThreadsPerChild > 128 허용(자원 극대화)
  DB 드라이버/외부 모듈 사용   -> ThreadsPerChild <= 128 (안전)
MPM: Event 기본(KeepAlive를 Listener Thread가 관리, Worker 대비 2~5배 수용)
KeepAliveTimeout = 3                 (요청 간격 2~4초 하한선)
다중 인스턴스: 가용_RAM_인스턴스 = (Total_RAM * 0.9) / N
              ServerLimit_인스턴스 <= 200/N
검증: Scoreboard(SL x TL >= 총 연결 한계), Thrashing 방지 조건
```

## 워크플로우

1. **스펙·용도 수집** — RAM, 코어 수, 인스턴스 수, **용도(순수 프록시/SSL 종단/외부 모듈 연동/혼합)**. 용도가 전략 분기를 결정하므로 불명확하면 안전 전략 기본 제시 후 확인.
2. **청사 로드** — `harness/webserver-standard-guide.md` §2(원칙), §4(공식), §5(매트릭스 구조) 읽기. 원본 확인 필요 시 `source/apache-tuning-guide.md`(수정 금지).
3. **공식 적용** — RAM(4/8/16/32GB) × 파라미터 매트릭스 형태로 산출. 운영자가 계산 없이 적용할 수 있게.
4. **검증** — Scoreboard 검증식, 전략 분기 판정 근거 명시. 혼합 용도면 용도 분리 권고.
5. **Web→WAS 계층 확인** — Web 타임아웃(KeepAlive/Proxy Timeout)이 WAS connectionTimeout보다 짧게 계층화되었는지 확인. WAS 값 자체는 was-tuning 영역.
6. **산출** — 컨벤션 준수(`harness/conventions.md`). 정본 반영 시 `update-guide` 절차.

## 주의 함정

- somaxconn과 ListenBacklog는 `min()`으로 묶인다 — sysctl만 올리고 앱 backlog를 그대로 두면 무의미(`linux-kernel-tuning` 연계)
- 한 서버에서 극대화/안전 전략 혼용 금지(안전 상한 무의미해짐)
- Prefork는 레거시 모듈 호환 시만 — 기본은 Event

## 관련 스킬

- Web→WAS 타임아웃 상호 검증: `was-tuning`
- Web 서버 커널 값: `linux-kernel-tuning`
- 가이드 갱신: `update-guide` | 시효성 검증: `verify-standards`
