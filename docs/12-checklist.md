# 12. 전체 점검 체크리스트

발표 전에 위에서부터 차례로 확인합니다.

| 확인 | 어디서 | 명령 | 기대 결과 |
|---|---|---|---|
| 시간 동기화 | 모든 서버 | `timedatectl \| grep -E 'Time zone\|synchronized'` | Asia/Seoul, yes |
| VIP 위치 | maria | `ip -br addr show enp0s8` | 192.168.56.200 붙어 있음 |
| MariaDB 역할 | maria / maria2 | `mariadb -N -e "SELECT @@hostname, @@read_only"` | maria 0, maria2 1 |
| 복제 | maria2 | `mariadb -e "SHOW SLAVE STATUS\G" \| grep -E 'Running:\|Behind'` | IO/SQL Yes, 지연 0 |
| keepalived | maria / maria2 | `systemctl is-active keepalived` | active |
| 테이블 수 | maria | `mariadb -e "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema='OTT'"` | 12 |
| MongoDB | mongo1 | `mongosh --quiet --eval "rs.status().members.map(m=>m.name+' '+m.stateStr)"` | .101 PRIMARY, .102·.103 SECONDARY |
| 몽고 문서 수 | mongo1 | `mongosh ott_service --quiet --eval "db.media_contents.countDocuments()"` | MariaDB `CONTENT` 행 수와 같음 |
| Redis | WAS | `valkey-cli -h 192.168.56.108 -a '<PASSWORD>' ping` | PONG |
| 카프카 서비스 | kafka | `systemctl is-active kafka ott-connect ott-collect ott-load-maria ott-load-mongo` | active × 5 |
| 커넥터 | kafka | `curl -s localhost:8083/connectors/mongo-sink/status` | RUNNING × 2 |
| 컨슈머 LAG | kafka | `/opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --all-groups` | LAG 0 (또는 줄어드는 중) |
| WAS | was | `systemctl is-active ott-was` | active |
| web | web | `curl -sk "https://127.0.0.1/api/suggest?q=오징"` | JSON |
| 외부 접속 | 다른 PC | `https://192.168.0.228` | 화면 표시 |
| 백업 | mongobackup | `ls -lt /root/backup_maria /root/dbbackup \| head` | 오늘 날짜 파일 |
| 백업 cron | maria·maria2·.107 | `crontab -l` | 0시 백업, (.107) 1시 정리 |
| 관제 | monitor | `curl -s localhost:9090/-/ready; curl -s localhost:3100/ready` | Ready / ready |
| Alloy | 모든 서버 | `systemctl is-active alloy` | active |
| Grafana | 다른 PC | `http://192.168.0.228:3000` → `up` 쿼리 | 모든 서버 1 |

MariaDB와 MongoDB 작품 수 비교 (한 번에):

```bash
# kafka 서버에서
python3 - <<'EOF'
import sys; sys.path.insert(0, "/opt/ott-was/pipeline")
from common import maria, mongo_db
c = maria(); cur = c.cursor(); cur.execute("SELECT COUNT(*) FROM CONTENT"); m = cur.fetchone()[0]
g = mongo_db()["media_contents"].count_documents({})
print(f"MariaDB {m:,} / MongoDB {g:,} -> {'일치' if m == g else '차이 ' + str(m - g)}")
EOF
```
