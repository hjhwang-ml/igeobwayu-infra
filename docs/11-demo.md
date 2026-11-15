# 11. 발표 시연 순서 (Part 4: 백업 전략 + 장애 전환)

설명 → 실제로 보여 주기 순서. 창을 미리 띄워 둡니다.

| 창 | 서버 | 띄워 둘 것 |
|---|---|---|
| 1 | 발표 PC | 브라우저 `https://192.168.0.228` (검색 화면) |
| 2 | kafka | `tail -f /var/log/ott_load_maria.log` |
| 3 | maria / maria2 | `watch -n1 "ip -br addr show enp0s8"` (VIP 위치) |
| 4 | mongo1 | `mongosh` |
| 5 | 발표 PC | Grafana `http://192.168.0.228:3000` (서버 살아 있음 `up` 패널) · Slack |

## 시연 1. MariaDB 1번을 죽여도 서비스 유지

1. 웹에서 검색 → 정상 확인
2. maria(.104): `systemctl stop mariadb`
3. 약 3~4초 뒤 (헬스체크 1초 × 3번 실패) VIP `192.168.56.200`이 maria2로 이동 → `maria_promote.sh`가 maria2를 Primary로 승격 → **Slack 알림**
4. 웹에서 다시 검색 → 정상 (WAS는 VIP로만 붙으므로 설정 변경 없음)
5. 카프카 적재 로그: 끊긴 동안 `[대기] DB 연결 실패 - 커밋하지 않고 30초 뒤 같은 메시지부터 다시` → VIP가 넘어간 뒤 이어서 적재 (**유실 없음**)
6. maria2에서 확인: `mariadb -e "SELECT @@read_only"` → 0

말할 것: "단순 ping이 아니라 시간 제한 SELECT와 실제 쓰기(heartbeat)까지 확인해서, 프로세스는 살아 있는데 응답이 없는 경우도 장애로 잡습니다. 복구된 서버는 VIP를 다시 뺏지 않습니다(nopreempt)."

**시연 뒤 복구** (발표 끝나고): maria에서 `/usr/local/bin/maria_rejoin.sh 192.168.56.109` → [02-mariadb.md 2-6](02-mariadb.md)

## 시연 2. MongoDB Primary 중지 → 재선출

```javascript
// mongo1 mongosh: 지금 상태
rs.status().members.map(m => ({ name: m.name, state: m.stateStr }))
```

1. mongo1(.101): `systemctl stop mongod`
2. mongo2 mongosh에서 몇 초 간격으로 위 명령 → 10초 안팎에 mongo2 또는 mongo3가 `PRIMARY`
3. 웹 상세 화면 정상 (읽기는 Secondary, `secondaryPreferred`)
4. mongo1: `systemctl start mongod` → priority 2라 다시 `PRIMARY`로 돌아옴

## 시연 3. 쓰기 확인 (Primary가 바뀌어도 적재됨)

새 Primary 상태에서 카프카 서버로 메시지 하나를 다시 흘려 봅니다. 이미 MariaDB에 있는 작품이라 몽고 문서만 다시 만들어집니다.

```bash
# kafka 서버: 아무 작품 key 하나로 "정형 적재 완료" 신호 보내기  ※ 확인: 실제 있는 작품 ID로
K=movie_550
echo "$K|{\"content_id\": 5501}" | /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9092 --topic ott.maria.loaded --property parse.key=true --property key.separator='|'
sleep 5
curl -s localhost:8083/connectors/mongo-sink/status; echo
```

```javascript
// 새 Primary mongosh: updated_at 이 방금 시각(UTC)이면 성공
use ott_service
db.media_contents.findOne({ _id: 5501 }, { title: 1, updated_at: 1 })
```

말할 것: "몽고 쓰기는 Kafka Connect의 MongoDB Sink 커넥터가 맡습니다. Primary가 바뀌어도 커넥터가 새 Primary를 찾아서 씁니다."

## 시연 순서에서 뺀 것

- 백업 파일을 지우고 복원하는 시연 (시간 때문에 설명만): 백업 전략·복원 명령은 [09-backup.md](09-backup.md)
