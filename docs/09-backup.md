# 9. 백업 — mongobackup (.107) + 마리아 2대

## 9-1. 전략

| 대상 | 어디서 뜨나 | 언제 | 어디에 | 보관 | 알림 |
|---|---|---|---|---|---|
| MariaDB `OTT` + 계정·권한 | **Replica**인 서버 (Primary면 건너뜀) | 매일 0시 | 로컬 `/root` → .107 `/root/backup_maria` | 로컬 7일, .107 30일 | Slack 성공·실패 |
| MongoDB `ott_service` | .107이 **Secondary**에서 mongodump | 매일 0시 | .107 `/root/dbbackup/YYYYMMDD_HHMMSS/` | 30일 | Slack 성공·실패 |
| 수집 원본 JSON | 카프카 서버 `raw/` (작품 하나 = 파일 하나) | 수집할 때마다 | 카프카 서버 | 계속 | — |
| Redis | 백업 안 함 (캐시·세션뿐) | — | — | — | — |

- **하루 1회로 충분한 이유:** 데이터가 바뀌는 건 하루 한 번(23시) 수집뿐이고, 사용자 쪽 쓰기(회원·주문 등)가 없습니다.
- **Replica·Secondary에서 뜨는 이유:** 덤프가 서비스(Primary)에 부하를 주지 않게.
- **원본 JSON도 백업의 일부:** DB가 둘 다 깨져도 `load_maria.py` → `load_mongo.py --rebuild`로 원본에서 다시 만들 수 있습니다.
- 3중 보호: 복제(maria2·mongo2/3) → 매일 덤프(.107) → 수집 원본(kafka)

## 9-2. MariaDB 백업 `backup_OTT.sh` (maria·maria2 둘 다, 서버 파일 기준)

Slack 웹훅은 `/etc/backup_slack.env`에 둡니다 (몽고 백업과 같은 파일·같은 알림 방식).

````bash
cat > /usr/local/bin/backup_OTT.sh <<'EOF'
#!/bin/bash
[ "$(mariadb -N -e "SELECT @@read_only")" = "1" ] || { echo "$(date "+%F %T") Primary라 백업 건너뜀"; exit 0; }
# OTT 서비스 MariaDB 백업: 데이터(OTT DB) + 계정·권한 -> 로컬 보관 + 백업 서버(mongobackup .107)로 전송
set -euo pipefail

# ===== Slack 알림 =====
source /etc/backup_slack.env          # SLACK_WEBHOOK_URL="https://hooks.slack.com/..."
JOB_NAME="OTT-MariaDB"                # mongobackup 쪽은 "OTT-MongoDB"
LOG_FILE="/var/log/backup_OTT.log"    # mongobackup 쪽은 "/var/log/mongo_backup.log"
START_TS=$(date +%s)

slack_notify() {
  local rc=$?
  local dur=$(( $(date +%s) - START_TS ))
  local text
  if [ $rc -eq 0 ]; then
    text=":white_check_mark: *[$(hostname -s)] ${JOB_NAME} 백업 성공* (${dur}초)"
  else
    text=":x: *[$(hostname -s)] ${JOB_NAME} 백업 실패* (exit ${rc}, line ${FAIL_LINE:-?}, ${dur}초)"
    [ -f "$LOG_FILE" ] && text+=$'\n```\n'"$(tail -n 8 "$LOG_FILE")"$'\n```'
  fi
  curl -sS -m 10 -X POST -H 'Content-type: application/json' \
    --data "$(printf '%s' "$text" | python3 -c 'import json,sys;print(json.dumps({"text":sys.stdin.read()}))')" \
    "$SLACK_WEBHOOK_URL" >/dev/null || echo "Slack 전송 실패" >&2
}
trap 'FAIL_LINE=$LINENO' ERR
trap slack_notify EXIT
# ===== 여기까지 =====

DIR=/root
DAY=$(date +%Y%m%d)
FILE="$DIR/ott_backup_$DAY.sql.gz"          # 테이블 + 데이터
USERS="$DIR/ott_users_$DAY.sql.gz"          # 계정 + GRANT (DB 백업에는 안 들어감)

mariadb-dump --single-transaction --routines --triggers --databases OTT | gzip > "$FILE"
mariadb-dump --system=users | gzip > "$USERS"
ls -lh "$FILE" "$USERS"

# 원격 서버로 전송
scp -o BatchMode=yes "$FILE" "$USERS" root@192.168.56.107:/root/backup_maria/

# 로컬 백업은 7일 지난 것 삭제 (백업 서버에는 30일)
find "$DIR" -maxdepth 1 \( -name 'ott_backup_*.sql.gz' -o -name 'ott_users_*.sql.gz' \) -mtime +7 -delete
echo "$(date '+%F %T') 백업 및 전송 완료"
EOF
chmod 700 /usr/local/bin/backup_OTT.sh

# .107로 비밀번호 없이 scp 되게 (처음 한 번)
[ -f ~/.ssh/id_ed25519 ] || ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519
ssh-copy-id root@192.168.56.107
ssh root@192.168.56.107 "mkdir -p /root/backup_maria"

(crontab -l 2>/dev/null | grep -v backup_OTT; echo "0 0 * * * /usr/local/bin/backup_OTT.sh >> /var/log/backup_OTT.log 2>&1") | crontab -
crontab -l
````

- 파일: `ott_backup_YYYYMMDD.sql.gz` (OTT DB, 약 8MB), `ott_users_YYYYMMDD.sql.gz` (계정·권한)
- Primary면 Slack 알림 없이 로그 한 줄만 남기고 끝납니다 (알림은 실제로 백업한 Replica에서 옴).
- `scp -o BatchMode=yes`: 키가 없으면 비밀번호를 묻고 멈추는 대신 바로 실패 → Slack 실패 알림

두 서버에 다 걸어 두면 장애 전환으로 역할이 바뀌어도 **그때의 Replica**가 알아서 백업합니다.

## 9-3. 백업 서버 (.107)

```bash
# mongodump 설치 (MongoDB 저장소 등록은 03-mongodb.md STEP 1과 같음)
dnf install -y mongodb-database-tools --nogpgcheck
mkdir -p /root/dbbackup /root/backup_maria
```

**몽고 백업 `/root/dbbackup/mongo_backup.sh`** (서버 파일 기준 + `LOG_FILE` 수정)

Slack 웹훅은 스크립트에 쓰지 않고 `/etc/backup_slack.env`에 둡니다 (`SLACK_WEBHOOK_URL="https://hooks.slack.com/..."`, `chmod 600`).

````bash
cat > /root/dbbackup/mongo_backup.sh <<'EOF'
#!/bin/bash
set -euo pipefail

# ===== Slack 알림 =====
source /etc/backup_slack.env          # SLACK_WEBHOOK_URL="https://hooks.slack.com/..."
JOB_NAME="OTT-MongoDB"
LOG_FILE="/var/log/mongo_backup.log"  # cron이 출력을 쌓는 파일 (실패 알림에 마지막 8줄을 붙임)
START_TS=$(date +%s)

slack_notify() {
  local rc=$?
  local dur=$(( $(date +%s) - START_TS ))
  local text
  if [ $rc -eq 0 ]; then
    text=":white_check_mark: *[$(hostname -s)] ${JOB_NAME} 백업 성공* (${dur}초)"
  else
    text=":x: *[$(hostname -s)] ${JOB_NAME} 백업 실패* (exit ${rc}, line ${FAIL_LINE:-?}, ${dur}초)"
    [ -f "$LOG_FILE" ] && text+=$'\n```\n'"$(tail -n 8 "$LOG_FILE")"$'\n```'
  fi
  curl -sS -m 10 -X POST -H 'Content-type: application/json' \
    --data "$(printf '%s' "$text" | python3 -c 'import json,sys;print(json.dumps({"text":sys.stdin.read()}))')" \
    "$SLACK_WEBHOOK_URL" >/dev/null || echo "Slack 전송 실패" >&2
}
trap 'FAIL_LINE=$LINENO' ERR
trap slack_notify EXIT
# ===== 여기까지 =====

BACKUP_DIR="/root/dbbackup"
DATE=$(date +%Y%m%d_%H%M%S)
TARGET_DIR="$BACKUP_DIR/$DATE"
RETENTION_DAYS=30

# readPreference=secondary: Primary를 건너뛰고 Secondary(mongo2·mongo3 중 하나)에서 덤프
MONGO_URI="mongodb://192.168.56.101:27017,192.168.56.102:27017,192.168.56.103:27017/?replicaSet=rs0&readPreference=secondary"

mkdir -p "$TARGET_DIR"
mongodump --uri="$MONGO_URI" --db=ott_service --gzip --out="$TARGET_DIR"      # 서비스 DB만 (testDB 제외)
find "$BACKUP_DIR" -mindepth 1 -maxdepth 1 -type d -mtime +$RETENTION_DAYS -exec rm -rf {} \;
EOF
chmod 700 /root/dbbackup/mongo_backup.sh
````

- 성공·실패를 `trap ... EXIT` 하나로 알립니다. 실패하면 몇 번째 줄에서 죽었는지(`FAIL_LINE`)와 로그 마지막 8줄이 Slack에 같이 옵니다.
- `readPreference=secondary`라서 **Secondary가 둘 다 죽으면 백업이 실패**합니다 (Primary에 부하를 안 주는 쪽을 택함). 그럴 때도 받고 싶으면 `secondaryPreferred`로 바꿉니다.

**마리아 백업 정리 `/usr/local/bin/cleanup_backup.sh`** — 30일 지난 것 삭제, 단 **최근 7일 안의 백업이 하나도 없으면 안 지움** (백업이 계속 실패하는 중에 옛 백업까지 지워져 0개가 되는 걸 막음). 서버 파일 그대로.

```bash
cat > /usr/local/bin/cleanup_backup.sh <<'EOF'
#!/bin/bash
# 백업 서버 정리: 30일 지난 MariaDB 백업 삭제. 최근 7일 안에 새 백업이 없으면 아무것도 지우지 않음
DIR=/root/backup_maria
if ! find "$DIR" -name 'ott_backup_*.sql.gz' -mtime -7 | grep -q .; then
  echo "$(date '+%F %T') 최근 7일 백업이 없어서 정리하지 않음 (백업 실패 확인 필요)"; exit 1
fi
find "$DIR" -maxdepth 1 -name 'ott_*.sql.gz' -mtime +30 -print -delete
echo "$(date '+%F %T') 정리 완료"
EOF
chmod 700 /usr/local/bin/cleanup_backup.sh
```

- 백업 파일 이름: `ott_backup_YYYYMMDD.sql.gz` (.107 `/root/backup_maria`, 11/15 확인)
- 최근 7일 백업이 없으면 `exit 1`로 끝내서 cron 로그에 실패가 남게 함

**cron**

```bash
cat <<'EOF' | crontab -
# 몽고 백업: 매일 0시, Secondary에서 mongodump (30일 보관은 스크립트 안에서)
0 0 * * * /root/dbbackup/mongo_backup.sh >> /var/log/mongo_backup.log 2>&1
# 마리아 백업 정리: 매일 1시, 30일 지난 것 삭제 (최근 7일 백업이 없으면 안 지움)
0 1 * * * /usr/local/bin/cleanup_backup.sh >> /var/log/cleanup_backup.log 2>&1
EOF
crontab -l
```

| 서버 | 시각 | 하는 일 |
|---|---|---|
| maria (.104) | 0시 | `backup_OTT.sh` (Replica일 때만 덤프 → .107) |
| maria2 (.109) | 0시 | `backup_OTT.sh` (Replica일 때만 덤프 → .107) |
| mongobackup (.107) | 0시 | `mongo_backup.sh` (몽고 Secondary에서 덤프) |
| mongobackup (.107) | 1시 | `cleanup_backup.sh` (마리아 백업 30일 정리) |

## 9-4. 수동 실행·확인

```bash
# Replica 쪽 마리아에서
/usr/local/bin/backup_OTT.sh; echo "종료 코드 $?"
# .107
/root/dbbackup/mongo_backup.sh; echo "종료 코드 $?"
ls -lt /root/backup_maria | head; ls -lt /root/dbbackup | head     # .107
```

종료 코드 0 + Slack 성공 알림이면 정상. (11/13 수동 테스트: 마리아 12개 테이블·몽고 7,400문서 확인)

## 9-5. 복원

**MariaDB** — 운영 DB를 덮기 전에 다른 이름의 DB로 먼저 확인합니다.

```bash
# 시험 복원: OTT_TEST 로
mariadb -e "CREATE DATABASE IF NOT EXISTS OTT_TEST CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci"
gunzip -c ott_backup_YYYYMMDD.sql.gz | sed 's/`OTT`/`OTT_TEST`/g' | mariadb
mariadb -e "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema='OTT_TEST'"   # 12

# 실제 복원: 지금 Primary인 서버에서 (root는 원격 접속이 막혀 있음) — 복제로 Replica에도 반영됨
scp root@192.168.56.107:/root/backup_maria/ott_backup_YYYYMMDD.sql.gz /root/
gunzip -c ott_backup_YYYYMMDD.sql.gz | mariadb
gunzip -c ott_users_YYYYMMDD.sql.gz | mariadb     # 계정이 날아갔을 때만
```

**MongoDB** (.107에서)

```bash
D=/root/dbbackup/$(ls -t /root/dbbackup | grep -v log | head -1); echo $D
mongorestore --uri="mongodb://192.168.56.101:27017,192.168.56.102:27017,192.168.56.103:27017/?replicaSet=rs0" \
  --gzip --drop --nsInclude='ott_service.*' "$D"
```

- `--nsInclude='ott_service.*'`: 서비스 DB만 복원
- `--drop`: 기존 컬렉션을 지우고 백업으로 바꿈 (인덱스도 백업에 들어 있어서 같이 복원됨)
- 레플리카셋 주소로 붙으면 Primary에 쓰이고 Secondary로 복제됩니다.

**원본에서 다시 만들기** (덤프까지 쓸 수 없을 때, 카프카 서버)

```bash
systemctl stop ott-load-maria ott-load-mongo
cd /opt/ott-was/pipeline
python3 maria/load_maria.py               # 테이블은 스키마 SQL로 미리 만들어 둘 것
python3 mongo/load_mongo.py --rebuild
systemctl start ott-load-maria ott-load-mongo
```
