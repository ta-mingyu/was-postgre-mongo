---
name: linux-kernel-tuning
description: Linux 커널/OS 튜닝 절차. 서버 역할(Web/WAS/PostgreSQL/MongoDB/PgPool)별 sysctl·ulimit·systemd 매트릭스를 다룬다. fs.file-max, somaxconn, tcp keepalive, vm.swappiness, overcommit_memory, THP(transparent_hugepage), dirty_ratio, kernel.sem, NUMA, systemd LimitNOFILE drop-in, GRUB 영구 설정 등 커널 파라미터 산정·점검·병설 판정을 할 때 반드시 이 스킬을 사용할 것. "커널 튜닝", "sysctl", "ulimit", "oom", "스왑", "THP", "세마포어", "fd 한계", "too many open files" 관련 작업도 여기에 포함된다.
---

# Linux 커널 튜닝

## 다루는 것

서버 역할별 커널 파라미터 표준값. 근거 정본은 `study/linux/06-tuning-matrix-and-checklist.md`(매트릭스 + 체크리스트)이며, 값의 "왜"는 `study/linux/01~05` 각 장이 담당한다. 산출물에는 `reports/final/*.md` §1의 값이 최종 반영된다.

## 기반 파라미터 (모든 서버 공통)

```
fs.file-max = 2097152        ulimit -n (LimitNOFILE) = 1048576     ulimit -u = 65536
net.core.somaxconn = 4096    net.ipv4.tcp_max_syn_backlog = 4096
tcp_keepalive_time=300, intvl=30, probes=5   (450초 내 dead 판정 < 방화벽 30min)
```

fd는 3계층(file-max → nr_open → RLIMIT_NOFILE)이 모두 커야 한다. 서비스 데몬은 PAM을 거치지 않으므로 **systemd drop-in 필수**.

## 역할별 매트릭스 (요약 — 정본은 study/linux/06 §2)

| 파라미터 | WAS | PostgreSQL | MongoDB 8.0 | PgPool-II |
|:---|:---:|:---:|:---:|:---:|
| `vm.swappiness` | 10 | 1 | 1 | 10 |
| `vm.overcommit_memory` | — | **2** | **1** | — |
| THP | — | **never** | **always** | — |
| `vm.dirty_background_ratio`/`dirty_ratio` | — | 5 / 10 | 5 / 15 | — |
| `tcp_fin_timeout` / `tcp_tw_reuse` | 15 / 1 | — | — | 15 / 1 |
| `ip_local_port_range` | 32768 65535 | — | — | 32768 65535 |
| `kernel.sem` | — | — | — | 250 32000 250 128 |

**병설 불가**: PostgreSQL과 MongoDB 8.0은 overcommit(2 vs 1)·THP(never vs always)가 커널 전역값이라 동일 호스트 병설 불가. cgroup으로도 overcommit은 분리 불가. 호스트 분리가 정답.

## 워크플로우

1. **역할 확인** — 서버 역할과 병설 대상. 병설 조합이 PG+Mongo면 즉시 분리 권고.
2. **매트릭스 로드** — `study/linux/06-tuning-matrix-and-checklist.md` 전체. 특히 §1(기반), §2(역할별 + 메커니즘 근거), §3(도메인 불변량), §4(적용 경로), §5(체크리스트).
3. **설정 블록 산출** — 3계층 세트로: `/etc/sysctl.d/` + `/etc/security/limits.d/` + 서비스별 systemd drop-in(`LimitNOFILE`, `LimitNPROC`).
4. **적용 경로 명시** — 즉시 적용 명령과 영구 설정(THP는 GRUB `grubby` 또는 TuneD)을 구분. 프로덕션 root/리부팅 작업은 "IT ONE 경유 IT 운영실 요청" 절차를 명시.
5. **체크리스트 판정** — §5.1~5.5(공통/WAS/PG/Mongo/PgPool) 항목별 판정 표.
6. **확인 명령 제공** — `sysctl -p`, `cat /proc/<pid>/limits`, `cat /sys/kernel/mm/transparent_hugepage/enabled` 등.

## 주의 함정

- `nofile=infinity` 금지(약 8.6GB 예약 버그), `swappiness=0` 금지(OOM 즉사), `tcp_tw_recycle` 제거됨(4.12+)
- 실제 accept 큐 = `min(somaxconn, 앱 backlog)` — 앱(Tomcat acceptCount 등) 동반 상향 필수
- sysctl 값 변경이 keepalive 450초 < 방화벽 30min 불변량을 깨지 않는지 확인

## 관련 스킬

- 도메인 값과 연계: `was-tuning`, `postgresql-pgpool-tuning`, `mongodb-tuning`, `webserver-tuning`
- 가이드 반영: `update-guide` | 시효성 검증(kernel.org 문서 기준): `verify-standards`
