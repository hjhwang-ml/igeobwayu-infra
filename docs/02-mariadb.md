# 2. MariaDB 서버 — maria(.104) / maria2(.109)

평소 구성: **maria(.104) Primary + VIP 192.168.56.200**, maria2(.109) Replica(read_only). GTID 복제, keepalived가 1초마다 헬스체크해서 Primary가 죽으면 VIP를 maria2로 옮기고 승격합니다. WAS와 카프카 적재는 VIP로만 접속합니다.

## 2-1. 설치 (두 서버 모두)

```bash
dnf -y install mariadb-server mariadb
systemctl enable --now mariadb
mariadb-secure-installation        # root 비밀번호 <PASSWORD>, 익명 계정·test DB 삭제, 원격 root 차단

# root 비밀번호를 파일로 (cron 백업·keepalived 스크립트용)
cat > /root/.my.cnf <<'EOF'
[client]
user=root
password=<PASSWORD>
EOF
chmod 600 /root/.my.cnf
```

## 2-2. 설정 `/etc/my.cnf.d/server.cnf` (`[mariadb]` 절)

**maria (.104)**

```ini
[mariadb]
bind-address            = 0.0.0.0
character-set-server    = utf8mb4
collation-server        = utf8mb4_unicode_ci
server_id               = 104
log_bin                 = mysql-bin
binlog_format           = ROW
log_slave_updates       = ON
gtid_strict_mode        = ON
expire_logs_days        = 7          # binlog 7일 보관 (복구 때 이 안에서 따라잡음)
```

**maria2 (.109)** — 위와 같고 아래만 다름

```ini
server_id               = 109
read_only               = ON
```

```bash
systemctl restart mariadb
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.56.0/24" service name="mysql" accept'
firewall-cmd --reload
```

> `read_only`는 설정 파일 값이 재시작 때 다시 적용됩니다. 장애 전환으로 역할이 바뀐 뒤에는 **server.cnf의 read_only도 지금 역할에 맞게** 바꿔 둬야 재부팅해도 꼬이지 않습니다 (Primary는 줄 삭제, Replica는 `read_only = ON`).

## 2-3. DB·계정 (Primary에서만, 복제로 maria2에도 넘어감)

```sql
CREATE DATABASE OTT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- WAS·카프카 적재용  ※ 확인: 실제 계정명
CREATE USER 'ott'@'192.168.56.%' IDENTIFIED BY '<PASSWORD>';
GRANT ALL PRIVILEGES ON OTT.* TO 'ott'@'192.168.56.%';

-- 복제 계정
CREATE USER 'repl'@'192.168.56.%' IDENTIFIED BY '<PASSWORD>';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'192.168.56.%';

-- 헬스체크용 (keepalived 스크립트)
CREATE USER 'health'@'localhost' IDENTIFIED BY '<PASSWORD>';
CREATE DATABASE ha;
CREATE TABLE ha.heartbeat (id TINYINT PRIMARY KEY, ts DATETIME(3));
GRANT SELECT, INSERT, UPDATE, DELETE ON ha.* TO 'health'@'localhost';
FLUSH PRIVILEGES;
```

- 스키마: `table12.mwb` 기준 12개 테이블 SQL 실행 (`PERSON.ORIGINAL_NAME` 포함본)
  CONTENT · MOVIE · SERIES · GENRE · CONTENT_GENRE · KEYWORD · CONTENT_KEYWORD · PERSON · CAST_CREDIT · CREW_CREDIT · PLATFORM · CONTENT_PLATFORM
- `REPLACE INTO`는 내부적으로 DELETE + INSERT라서 health 계정에 DELETE 권한도 줍니다.

## 2-4. GTID 복제 연결 (maria2에서)

maria2 → maria로 SSH 키가 있어야 합니다 (`ssh-copy-id root@192.168.56.104`).

```bash
# Primary 덤프를 GTID 위치와 함께 가져와서 복원
ssh root@192.168.56.104 "mariadb-dump --all-databases --single-transaction --master-data=2 --gtid --routines --events" > /root/full.sql
mariadb < /root/full.sql
grep -m1 "gtid_slave_pos" /root/full.sql      # 덤프 안의 GTID 위치 확인 (주석으로 들어 있음)
```

```sql
-- maria2
STOP SLAVE;
SET GLOBAL gtid_slave_pos = '0-104-XXXX';     -- 위에서 확인한 값
CHANGE MASTER TO
  MASTER_HOST='192.168.56.104', MASTER_USER='repl', MASTER_PASSWORD='<PASSWORD>',
  MASTER_USE_GTID=slave_pos;
START SLAVE;
SHOW SLAVE STATUS\G      -- Slave_IO_Running: Yes / Slave_SQL_Running: Yes / Seconds_Behind_Master: 0
```

## 2-5. keepalived로 VIP 자동 전환 (두 서버 모두)

```bash
dnf -y install keepalived
firewall-cmd --permanent --add-rich-rule='rule protocol value="vrrp" accept' && firewall-cmd --reload
```

**헬스체크 `/usr/local/bin/maria_check.sh`** — 단순 ping이 아니라 시간 제한 SELECT + (Primary일 때) 쓰기 확인

```bash
cat > /usr/local/bin/maria_check.sh <<'EOF'
#!/bin/bash
# 0 = 정상, 1 = 장애. 3번 연속 1이면 keepalived가 VIP를 내려놓음
Q="mariadb -uhealth -p<PASSWORD> --connect-timeout=2 -N -e"
timeout 3 $Q "SELECT 1" >/dev/null 2>&1 || exit 1                      # 접속·응답
RO=$(timeout 3 $Q "SELECT @@read_only" 2>/dev/null)
if [ "$RO" = "0" ]; then                                                 # Primary면 실제로 써지는지도
  timeout 3 $Q "REPLACE INTO ha.heartbeat VALUES (1, NOW(3))" >/dev/null 2>&1 || exit 1
fi
exit 0
EOF
chmod 700 /usr/local/bin/maria_check.sh
```

**승격 스크립트 `/usr/local/bin/maria_promote.sh`** — VIP를 받으면 Replica를 Primary로 바꾸고 Slack 알림

```bash
cat > /usr/local/bin/maria_promote.sh <<'EOF'
#!/bin/bash
export HOME=/root                                    # keepalived가 부를 때도 /root/.my.cnf 를 읽게
WEBHOOK="https://hooks.slack.com/services/XXXX"     # ※ 실제 웹훅
RO=$(mariadb -N -e "SELECT @@read_only")
if [ "$RO" = "1" ]; then
  mariadb -e "STOP SLAVE; RESET SLAVE ALL; SET GLOBAL read_only=0;"
  MSG="[MariaDB] $(hostname) Primary 승격 (VIP 192.168.56.200)"
else
  MSG="[MariaDB] $(hostname) VIP 보유 (이미 Primary)"
fi
logger -t maria_promote "$MSG"
curl -s -X POST -H 'Content-type: application/json' --data "{\"text\":\"$MSG\"}" "$WEBHOOK" >/dev/null
EOF
chmod 700 /usr/local/bin/maria_promote.sh
```

**`/etc/keepalived/keepalived.conf`** (maria .104 기준. maria2는 `router_id maria2`, `priority 100`만 다름)

```conf
global_defs {
  router_id maria
  script_user root
  enable_script_security
}
vrrp_script chk_maria {
  script "/usr/local/bin/maria_check.sh"
  interval 1          # 1초마다
  timeout 3
  fall 3              # 3번 연속 실패하면 장애
  rise 2
}
vrrp_instance VI_MARIA {
  state BACKUP        # nopreempt는 두 서버 모두 BACKUP일 때만 동작 (높은 priority가 먼저 VIP를 가짐)
  interface enp0s8    # ※ 확인: 호스트 전용망 NIC 이름
  virtual_router_id 51
  priority 110        # maria2는 100
  advert_int 1
  nopreempt           # 복구된 서버가 VIP를 다시 뺏지 않게 (되돌리기는 수동)
  authentication { auth_type PASS  auth_pass <VRRP_PW> }
  virtual_ipaddress { 192.168.56.200/24 }
  track_script { chk_maria }
  notify_master "/usr/local/bin/maria_promote.sh"
}
```

```bash
systemctl enable --now keepalived
ip -br addr show enp0s8          # Primary 쪽에 192.168.56.200 이 붙어 있어야 함
journalctl -u keepalived -n 20 --no-pager
```

- WAS·카프카 `config.ini` → `[mariadb] host = 192.168.56.200`
- VIP를 `.110`이 아니라 `.200`으로 정한 이유: `.110`은 관제 서버가 씀
- `nopreempt`가 있으면 maria가 복구돼도 VIP는 maria2에 남습니다. 데이터가 뒤처진 서버가 VIP를 다시 뺏어 가면 안 되기 때문입니다.

## 2-6. 원상 복구 — 죽었던 서버를 Replica로 다시 붙이기

장애 전환 뒤에는 **maria2가 Primary**, 죽었던 maria는 데이터가 뒤처진 상태입니다. 현재 Primary의 덤프로 다시 만들어 Replica로 붙입니다. 손으로 하다 틀리기 쉬워서 스크립트로 만들었습니다.

죽었던 서버 → 현재 Primary로 SSH 키가 있어야 합니다.

```bash
cat > /usr/local/bin/maria_rejoin.sh <<'EOF'
#!/bin/bash
# 사용: maria_rejoin.sh <현재 Primary IP>   (죽었던 서버에서 실행)
set -euo pipefail
export HOME=/root
P=${1:?현재 Primary IP를 주세요 (예: 192.168.56.109)}
F=/root/rejoin_$(date +%Y%m%d_%H%M).sql

systemctl start mariadb
mariadb -e "SET GLOBAL read_only=1; STOP SLAVE; RESET SLAVE ALL;"     # 먼저 쓰기부터 막음

echo "[1] $P 에서 덤프"
ssh root@$P "mariadb-dump --all-databases --single-transaction --master-data=2 --gtid --routines --events" > $F
POS=$(grep -m1 -oP "gtid_slave_pos='\K[^']*" $F)
echo "    GTID 위치: $POS"

echo "[2] 복원 (내 binlog에는 안 남김 -> GTID 충돌 방지)"
mariadb --init-command="SET SESSION sql_log_bin=0" < $F

echo "[3] 복제 연결"
mariadb -e "SET GLOBAL gtid_slave_pos='$POS';
  CHANGE MASTER TO MASTER_HOST='$P', MASTER_USER='repl', MASTER_PASSWORD='<PASSWORD>', MASTER_USE_GTID=slave_pos;
  START SLAVE;"
sleep 3
mariadb -e "SHOW SLAVE STATUS\G" | grep -E "Slave_(IO|SQL)_Running:|Seconds_Behind_Master|Last_(IO|SQL)_Error:"
echo "server.cnf 에 read_only = ON 넣는 것 잊지 말 것"
EOF
chmod 700 /usr/local/bin/maria_rejoin.sh

# 예: maria(.104)가 죽었다 살아났고 지금 Primary가 maria2(.109)일 때, maria에서
/usr/local/bin/maria_rejoin.sh 192.168.56.109
```

IO·SQL이 둘 다 `Yes`, `Seconds_Behind_Master: 0`이면 끝입니다. keepalived는 계속 켜 둬도 됩니다 (nopreempt라 VIP를 안 가져감).

## 2-7. Primary를 다시 maria(.104)로 되돌리기 (선택)

발표 전 원래 구성(maria Primary)으로 돌릴 때만 합니다. 순서가 중요합니다.

1. 쓰기 멈춤: 카프카 서버에서 `systemctl stop ott-load-maria ott-load-mongo`
2. maria(.104)가 maria2를 다 따라잡았는지 확인: `SHOW SLAVE STATUS\G` → `Seconds_Behind_Master: 0`
3. maria2(.109): `SET GLOBAL read_only=1;`
4. maria(.104): `STOP SLAVE; RESET SLAVE ALL; SET GLOBAL read_only=0;`
5. maria2(.109)를 maria의 Replica로: `/usr/local/bin/maria_rejoin.sh 192.168.56.104` (덤프로 다시 맞추는 게 가장 확실)
6. VIP 이동: maria2에서 `systemctl restart keepalived` → VIP가 maria(.104)로 감 (`ip -br addr`로 확인)
7. 두 서버 server.cnf의 read_only 정리, 카프카에서 `systemctl start ott-load-maria ott-load-mongo`

현재(11/13 밤 이후): maria .104 Primary + VIP, maria2 .109 Replica

## 2-8. 백업

두 서버 모두 0시 cron으로 `backup_OTT.sh`를 돌리고, **Replica일 때만** 덤프합니다. → [09-backup.md](09-backup.md)
