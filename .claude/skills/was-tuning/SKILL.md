---
name: was-tuning
description: WAS/JVM 표준값 산정·튜닝 절차. Tomcat(독립형/Spring Boot 내장), WebSphere Liberty의 maxThreads·Heap(Xms=Xmx)·GC 선택(Parallel/G1/ZGC, 4096m 분기선)·Metaspace·HikariCP 풀(maxPoolSize=20, maxLifetime 27min)·타임아웃 산출을 다룬다. JVM 튜닝, 스레드 풀, 커넥션 풀, GC 로그 해석, Old 영역 사용률 진단, WAS 관련 표준 가이드 갱신이나 Gap 분석을 할 때 반드시 이 스킬을 사용할 것. Web 서버(Apache)나 DB 내부 파라미터에는 사용하지 않는다(각각 webserver-tuning, postgresql-pgpool-tuning, mongodb-tuning 사용).
---

# WAS/JVM 튜닝

## 다루는 것

3종 WAS 플랫폼의 JVM·Thread·Connection Pool·타임아웃 표준값 산정. 상세 산정 공식·매트릭스·검증 체크리스트의 정본은 `harness/was-rules.md`(223행)이며, 이 스킬은 그 파일을 언제 어떻게 쓰는지 안내한다.

| 플랫폼 | 설정 파일 | 특이 |
| :--- | :--- | :--- |
| 독립형 Tomcat 9.x | `server.xml`, `setenv.sh` | maxThreads/acceptCount 직접 제어 |
| Spring Boot 내장 Tomcat | `application.yml` | `server.tomcat.*` 네임스페이스 |
| WebSphere Liberty 23.x | `server.xml`, `jvm.options` | Executor/ConnectionManager 독자 구조 |

## 핵심 공식 (요약 — 정본은 was-rules.md §2)

```
maxThreads   = min(CPU_cores * 50, 500)
minSpareThreads = maxThreads / 8
acceptCount  = 100 (고정)
Heap_인스턴스 = floor(호스트_RAM * 0.625) / 인스턴스_수
  예외: 4GB 단일 인스턴스 = 호스트_RAM * 0.50
GC: Heap <= 4096m -> Parallel GC, > 4096m -> G1 GC
HikariCP: maxPoolSize = 20/인스턴스, minimumIdle = maxPoolSize(fixed-size)
  maxLifetime = 27min(1,620,000ms), keepaliveTime = 60s
```

## 워크플로우

1. **스펙 수집** — 호스트 RAM, CPU 코어, 인스턴스/컨페이너 수, WAS 엔진·버전, Java 버전. 누락 시 가정 금지, 질의.
2. **rules 로드** — 반드시 `harness/was-rules.md` 전체를 읽는다. 특히 §2(산정 공식), §3(타임아웃 캐스케이드), §7(검증 체크리스트), §8(에이전트 작업 규칙).
3. **공식 적용** — 실제 숫자를 대입해 산정 과정을 남긴다. 매트릭스(§2.2)에 있는 환경이면 매트릭스 값을 우선한다.
4. **불변량 검증** — `harness/gotchas.md`의 공통 불변량을 확인:
   - `Sum(maxPoolSize) <= DB max_connections * 0.7`(70% Ceiling) — DB 측 값 역산 필요
   - 타임아웃 캐스케이드 엄격 부등호(27 < 28 < 30min), 등호 금지
   - `maxThreads > 0`(무제한 -1 금지), `Xms == Xmx`, Metaspace `Max >= Min`
5. **체크리스트 실행** — was-rules.md §7 항목 전부 판정. 미충족 항목은 산출물에 명시.
6. **산출** — 한국어, 이모지 금지, 구조는 mermaid, 수치는 단위 병기(`harness/conventions.md`).

## HITL 가드

HITL-003(CL플랫폼팀 Old 영역 90.2% 사용률) 관련 확정·변경은 TA 승인 전 금지. 활성 HITL 목록은 `harness/workflow.md`. 분석·제안은 가능하되 "확정" 표현을 쓰지 않는다.

## 관련 스킬

- 팀별 현재 설정과 비교: `gap-analysis`
- 산출물을 가이드에 반영: `update-guide`
- 권장값 시효성 외부 검증: `verify-standards`
- OS 커널 값: `linux-kernel-tuning`(WAS 서버: swappiness 10, tcp_fin_timeout 15 등)
