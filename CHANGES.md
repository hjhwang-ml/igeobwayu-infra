# 수정 내역 (2026-11-15 취합)

예전 문서 두 개(Redis·MariaDB·web·WAS 워크북, MongoDB·관제 워크북 docx)를 합치면서 고친 것들입니다.

## 고친 것 (그대로 실행하면 문제가 됐던 부분)

| 위치 | 예전 | 바꾼 것 | 이유 |
|---|---|---|---|
| MariaDB keepalived | 1번 `state MASTER` + `nopreempt` | 두 서버 모두 `state BACKUP` (priority 110/100) | keepalived의 `nopreempt`는 BACKUP 상태에서만 동작. MASTER로 두면 1번이 복구될 때 VIP를 다시 뺏어 감 |
| `maria_promote.sh` | `mariadb` 바로 실행 | `export HOME=/root` 추가 | keepalived가 부를 때 HOME이 없으면 `/root/.my.cnf`를 못 읽을 수 있음 |
| health 계정 권한 | SELECT, INSERT, UPDATE | DELETE 추가 | `REPLACE INTO`는 DELETE + INSERT |
| MariaDB 원상 복구 | 손으로 SQL | `maria_rejoin.sh` 스크립트 | 복원 중 binlog 기록을 끄고(`sql_log_bin=0`) GTID 위치를 자동으로 읽어서 실수 방지 |
| 원상 복구 후 | — | server.cnf `read_only` 맞추기 안내 | SET GLOBAL은 재시작하면 설정 파일 값으로 돌아감 |
| WAS 설치 | `python3 -m venv venv` | 시스템 파이썬 `python3 -m pip` | 프로젝트 서버는 venv를 쓰지 않음 |
| WAS 재시작 (MongoDB STEP 6) | `systemctl restart gunicorn` | `systemctl restart ott-was` | 실제 서비스 이름 |
| mongod.conf, Loki config | 들여쓰기가 사라진 YAML | 들여쓰기 복원 | YAML은 들여쓰기가 틀리면 서비스가 안 뜸 |
| prometheus.service | `Restart=always # 핵심 옵션...` | 주석을 따로 한 줄로 | systemd는 줄 중간 `#` 주석을 지원하지 않음 |
| Loki | 시작만 | `chown loki /var/lib/loki`, `/ready` 확인 | 폴더 권한이 root면 loki 계정이 못 씀 |
| Grafana | 설치만, 시작 명령 없음 | `systemctl enable --now grafana-server` | 빠져 있었음 |
| 관제 서버 방화벽 | `--zone=internal` 고정 | NIC의 실제 zone을 먼저 확인 | NIC가 public zone이면 포트가 안 열림 |
| 백업·web 서버 Alloy 설정 | `[내용은 위 패턴과 동일]` 자리표시 | `alloy_setup.sh` 하나로 모든 서버 설정 생성 | 자리표시를 그대로 실행하면 설정 파일이 깨짐 |
| Alloy logrotate | `/etc/logrotate.d/alloy-acl` 따로 | 기존 `rsyslog` 설정의 postrotate에 한 줄 추가 | 같은 파일이 두 설정에 있으면 logrotate가 duplicate 오류로 건너뜀 |
| NAT ip_forward | `/etc/sysctl.conf`에 추가 | `/etc/sysctl.d/99-forward.conf` | Rocky 10 권장 위치 |
| Redis 키 표 | search·similar·recommend·recent_view만 | `recent_search`, `content:*`, `rate:*`, `stats:cache` 추가 | WAS `cache.py`에서 실제로 쓰는 키 |
| `mongo_backup.sh` (서버) | `LOG_FILE=/root/dbbackup/backup.log` | `/var/log/mongo_backup.log` | cron 출력은 `/var/log/mongo_backup.log`로 감 → 실패 알림에 붙는 로그가 엉뚱한(옛) 파일이었음 |
| `load_mongo.py` (서버) | 영어 리뷰를 `raw/reviews/` 파일에서만 읽음 | 파일이 없으면 원본 `reviews_en` 사용 | 통합 수집 후에는 그 파일이 안 생겨서 새 작품 문서에 영어 리뷰가 빠짐 |
| `ott.reviews` 토픽 (서버) | 아무도 안 보내는데 `load_mongo.py`가 구독 | 구독에서 빼고 토픽 삭제, `common.py`·`load_mongo.py` 설명 정리 | 영어 리뷰는 원본 `reviews_en`으로 통합됨 (프로듀서 `collect.py`는 이미 맞게 되어 있었음) |
| 체크리스트 카프카 | "수집은 현재 꺼 둠" | 5개 서비스 모두 active | 11/15 수집 다시 켬, `ott-connect` 추가 |

## 새로 넣은 것

- **카프카 (docs/05-kafka.md):** Kafka 4.4.0 KRaft 설치·설정, 브로커 힙 512MB, 토픽 4개, 파이프라인 서비스 3개, Kafka Connect + MongoDB Sink 3.1.3, 확인 명령, 카프카 장애 시 파일 모드
- **백업 서버 (docs/09-backup.md):** 백업 전략 표, `mongo_backup.sh`, `cleanup_backup.sh`, cron, 복원 (MariaDB 시험 복원 → 실제 복원, mongorestore, 원본에서 재구축)
- **관제:** 서버별 로그 경로 표 (WAS·Redis·카프카·NAT 포함), 지표별 한 패널에 전 서버 PromQL
- **NAT:** 3000 → Grafana 포트포워딩을 443과 한곳에
- **시연 (docs/11-demo.md):** MariaDB 장애 전환, MongoDB 재선출, Primary가 바뀐 뒤 쓰기 확인
- 서버 구성표에 monitor(.110), nat 추가, `/etc/hosts`에도 추가

## 서버에서 확인할 것 (※ 확인 표시)

- NIC 이름 (`enp0s8` 등)
- `pipeline/systemd/` 서비스 파일 — 문서는 기록을 바탕으로 다시 쓴 것이라 서버 파일과 비교 (백업 스크립트 3개는 11/15 서버 파일로 맞춤)
- WAS `config.ini` `[cache]` 키 이름, `gunicorn` 경로
- Valkey 로그 경로
- 프로젝트 폴더의 `load_mongo.py`는 Connect 전(11/12) 버전 — 카프카 서버 것(ott.mongo.docs로 보냄)이 최신
