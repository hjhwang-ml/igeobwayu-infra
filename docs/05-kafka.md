# 5. 카프카 서버 — kafka (.111)

수집·정제·적재 파이프라인을 전부 이 서버에서 돌립니다. WAS에는 Flask만 있습니다.

```
collect.py (수집, Producer)
  → ott.content ─┬→ load_maria.py (정제 + MariaDB 12테이블 적재, Consumer)
  → ott.providers┘      → ott.maria.loaded
                              → load_mongo.py (정제: 문서 만들기, Consumer)
                                    → ott.mongo.docs
                                          → mongo-sink (Kafka Connect, MongoDB Sink 커넥터)
                                                → MongoDB ott_service.media_contents
```

**왜 카프카인가 (발표 포인트):** 실시간이 목적이 아니라 **DB가 죽어도 데이터가 안 사라지는 것**.
- 수집은 원본 JSON 파일을 먼저 저장하고, 그다음 토픽에 보냅니다. 카프카가 죽어 있어도 수집은 계속되고 원본은 남습니다.
- 적재 컨슈머는 **DB 커밋이 끝난 뒤에만 오프셋을 커밋**합니다. MariaDB가 죽어 있으면 커밋하지 않고 30초마다 같은 메시지부터 다시 시도하고, DB가 살아나면 멈춘 곳부터 이어서 넣습니다.
- 같은 메시지를 두 번 받아도 결과가 같습니다 (MariaDB `INSERT IGNORE`, MongoDB `_id` 기준 ReplaceOne upsert).
- MongoDB 쓰기는 Kafka Connect의 MongoDB Sink 커넥터가 맡습니다. 몽고 Primary가 바뀌어도 커넥터가 새 Primary를 찾아 이어서 씁니다.

## 5-1. VM·자바

Rocky 10, 2 vCPU, 메모리 **1.7GB**(실제 값), 디스크 20GB. 메모리가 작아서 브로커 힙 512MB, Connect 힙 256MB로 줄였습니다.

```bash
hostnamectl set-hostname kafka
dnf install -y java-21-openjdk-headless tar python3 python3-pip
java -version          # openjdk 21
```

## 5-2. 카프카 설치 (4.4.0)

```bash
KV=4.4.0
cd /opt && curl -LO https://downloads.apache.org/kafka/$KV/kafka_2.13-$KV.tgz    # 없으면 archive.apache.org/dist/kafka/
tar xzf kafka_2.13-$KV.tgz && ln -s kafka_2.13-$KV kafka && rm -f kafka_2.13-$KV.tgz
id kafka >/dev/null 2>&1 || useradd -r -s /sbin/nologin kafka
mkdir -p /data/kafka-logs
```

## 5-3. 설정 `/opt/kafka/config/server.properties` (단일 노드 KRaft)

ZooKeeper 없이 한 프로세스가 브로커와 컨트롤러를 같이 합니다.

```properties
process.roles=broker,controller
node.id=1
controller.quorum.voters=1@localhost:9093
listeners=PLAINTEXT://0.0.0.0:9092,CONTROLLER://localhost:9093
advertised.listeners=PLAINTEXT://192.168.56.111:9092
log.dirs=/data/kafka-logs
num.partitions=1
message.max.bytes=10485760        # 출연·이미지가 많은 작품 원본이 커서 10MB까지
log.retention.hours=168           # 메시지 7일 보관
```

- `advertised.listeners`가 중요합니다. `localhost`로 남아 있으면 다른 서버에서 붙었을 때 메시지를 못 주고받습니다.
- 같은 키가 파일에 두 번 있으면 **아래쪽 값**이 적용됩니다. 실제 서버 파일 끝에 설정 묶음이 한 번 더 붙어 있으니 고칠 때는 아래쪽을 고칩니다.

```bash
cd /opt/kafka
bin/kafka-storage.sh format --standalone -t $(bin/kafka-storage.sh random-uuid) -c config/server.properties   # 처음 한 번만
chown -R kafka: /data/kafka-logs /opt/kafka_2.13-4.4.0
```

`already formatted`가 나오면 이미 된 것이니 넘어갑니다. **돌고 있는 서버에서 다시 포맷하지 말 것** (메시지가 전부 사라짐).

## 5-4. 서비스

```bash
cat > /etc/systemd/system/kafka.service <<'EOF'
[Unit]
Description=Apache Kafka (KRaft)
After=network-online.target

[Service]
User=kafka
ExecStart=/opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
ExecStop=/opt/kafka/bin/kafka-server-stop.sh
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# 메모리 1.7GB라 힙을 512MB로 제한
mkdir -p /etc/systemd/system/kafka.service.d
cat > /etc/systemd/system/kafka.service.d/override.conf <<'EOF'
[Service]
Environment="KAFKA_HEAP_OPTS=-Xms256m -Xmx512m"
EOF

systemctl daemon-reload && systemctl enable --now kafka
sleep 10
systemctl status kafka --no-pager | head -5
ss -lntp | grep -E '9092|9093'
```

방화벽: 프로듀서·컨슈머·Connect가 전부 이 서버 안에 있어서 9092·8083은 **열지 않습니다**. (다른 서버에서 붙여야 하면 그 IP만 rich-rule로 허용)

## 5-5. 토픽

파티션 1개: 정형 적재 컨슈머가 출연·제작진 ID(`CAST_CREDIT_ID` 등)를 직접 매기기 때문에 컨슈머를 하나만 띄웁니다. 순서도 지켜집니다.

```bash
cd /opt/kafka
for t in ott.content ott.providers ott.maria.loaded ott.mongo.docs; do
  bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --if-not-exists --topic $t \
    --partitions 1 --replication-factor 1 --config max.message.bytes=10485760
done
bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

| 토픽 | 보내는 쪽 → 받는 쪽 | 메시지 (key = `movie_{ID}` / `tv_{ID}`) |
|---|---|---|
| `ott.content` | collect.py → load_maria.py | 작품 상세 원본 한 편 (출연·키워드·영상·이미지·제공 현황·한국어+영어 리뷰 포함) |
| `ott.providers` | collect.py → load_maria.py | 작품별 오늘 한국 OTT 제공 현황 (하루 한 번) |
| `ott.maria.loaded` | load_maria.py → load_mongo.py | 정형 적재 완료 신호 `{content_id}` |
| `ott.mongo.docs` | load_mongo.py → mongo-sink | 완성된 MongoDB 문서 (Mongo 확장 JSON, 날짜는 `{"$date": ...}`) |

- 예전에는 영어 리뷰를 따로 받아 `ott.reviews` 토픽으로 보냈지만, 지금은 원본 하나(`reviews_en`)에 들어가 `ott.content`로 갑니다. 11/15에 `load_mongo.py` 구독에서 빼고 **토픽을 삭제**했습니다.
- 목록에 보이는 나머지 토픽은 카프카가 쓰는 것입니다: `__consumer_offsets`(컨슈머 그룹별 오프셋 = 어디까지 처리했는지), `connect-configs`·`connect-offsets`·`connect-status`(Kafka Connect의 커넥터 설정·진행 위치·상태)
- 몽고 적재가 MariaDB 뒤에 오는 이유: 몽고 문서의 배역 이름이 MariaDB `CAST_CREDIT_ID`를 참조합니다.

## 5-6. 파이프라인 배치

```
/opt/ott-was/
  config.ini                      설정 (TMDB 토큰·DB 비밀번호 있음, 레포에 올리지 말 것)
  pipeline/common.py              공통: TMDB 호출(초당 rps회 제한), 정제 도우미, DB 연결, 카프카 보내기·받기
  pipeline/collect/collect.py     수집 (Producer)
  pipeline/maria/load_maria.py    정형 정제·적재 (Consumer → MariaDB)
  pipeline/mongo/load_mongo.py    반정형 정제 (Consumer → ott.mongo.docs). 파일 모드(--rebuild/--all)는 몽고에 직접 씀
  pipeline/clear_cache.py         Redis 캐시 비우기
  pipeline/systemd/               서비스 파일 원본
  raw/{movie|tv}_{ID}.json        수집 원본 (작품 하나 = 파일 하나, 영어 리뷰 포함) — 약 7,400편 / 700MB
  raw/providers/YYYYMMDD.json     날짜별 OTT 제공 현황 원본
```

프로젝트 서버는 가상환경(venv) 없이 시스템 파이썬을 씁니다.

```bash
python3 -m pip install kafka-python pymysql pymongo redis
python3 -m pip list 2>/dev/null | grep -iE 'kafka|pymysql|pymongo|redis'
```

**`/opt/ott-was/config.ini`**

```ini
[tmdb]
token = (TMDB API 읽기 토큰, eyJ...)
rps = 10                          ; TMDB 호출 초당 10회 (제한은 약 40회)

[mariadb]
host = 192.168.56.200             ; VIP
port = 3306
user = ott
password = <PASSWORD>
database = OTT

[mongodb]
uri = mongodb://192.168.56.101:27017,192.168.56.102:27017,192.168.56.103:27017/?replicaSet=rs0
database = ott_service
collection = media_contents

[redis]
host = 192.168.56.108
port = 6379
password = <PASSWORD>

[kafka]
bootstrap = 192.168.56.111:9092

[collect]
raw_dir = /opt/ott-was/raw
batch = 30                        ; 라운드당 신작 수
max_total = 10000                 ; 전체 상한 (선생님 기준 만 건 이하)
rest_sec = 86400                  ; 라운드 사이 → 사실상 하루 한 번
idle_sec = 3600                   ; 새 작품이 없을 때 쉬는 시간
providers_hour = 23               ; OTT 제공 현황 받는 시각
```

수집 루프는 쉬는 중에도 23시를 놓치지 않게 깨어나서, **매일 23시에 신작 30편 + 전체 작품 OTT 현황**을 받습니다.

## 5-7. 파이프라인 서비스 3개

로그는 journal이 아니라 `/var/log/ott_*.log` 파일로 남깁니다. ※ 확인: `pipeline/systemd/` 원본과 같은지

```bash
cat > /etc/systemd/system/ott-collect.service <<'EOF'
[Unit]
Description=이거봐유 수집 (TMDB -> raw + Kafka)
After=network-online.target kafka.service

[Service]
WorkingDirectory=/opt/ott-was/pipeline
ExecStart=/usr/bin/python3 -u /opt/ott-was/pipeline/collect/collect.py --loop
StandardOutput=append:/var/log/ott_collect.log
StandardError=append:/var/log/ott_collect.log
Restart=always
RestartSec=30

[Install]
WantedBy=multi-user.target
EOF

cat > /etc/systemd/system/ott-load-maria.service <<'EOF'
[Unit]
Description=이거봐유 정형 적재 (Kafka -> MariaDB)
After=network-online.target kafka.service

[Service]
WorkingDirectory=/opt/ott-was/pipeline
ExecStart=/usr/bin/python3 -u /opt/ott-was/pipeline/maria/load_maria.py --kafka
StandardOutput=append:/var/log/ott_load_maria.log
StandardError=append:/var/log/ott_load_maria.log
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

cat > /etc/systemd/system/ott-load-mongo.service <<'EOF'
[Unit]
Description=이거봐유 반정형 정제 (Kafka -> ott.mongo.docs)
After=network-online.target kafka.service

[Service]
WorkingDirectory=/opt/ott-was/pipeline
ExecStart=/usr/bin/python3 -u /opt/ott-was/pipeline/mongo/load_mongo.py --kafka
StandardOutput=append:/var/log/ott_load_mongo.log
StandardError=append:/var/log/ott_load_mongo.log
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now ott-load-maria ott-load-mongo ott-collect
tail -f /var/log/ott_load_maria.log        # "구독 시작", "MariaDB 연결" 이 보이면 정상
```

| 서비스 | 컨슈머 그룹 | 구독 토픽 |
|---|---|---|
| `ott-load-maria` | `ott-maria-loader` | ott.content, ott.providers |
| `ott-load-mongo` | `ott-mongo-loader` | ott.maria.loaded |
| `ott-connect` (mongo-sink) | `connect-mongo-sink` | ott.mongo.docs |

**주의:** 컨슈머가 떠 있을 때 파일 모드(`load_maria.py` 옵션 없이)를 같이 돌리지 않습니다. 출연·제작진 ID가 겹칩니다. 파일 모드로 재구축할 때는 `systemctl stop ott-load-maria ott-load-mongo` 먼저.

## 5-8. Kafka Connect + MongoDB Sink (11/15 추가)

몽고에 **쓰는 일**은 Connect가, **문서를 만드는(정제) 일**은 파이썬이 하도록 나눴습니다.

```
[바꾸기 전] ott.maria.loaded → load_mongo.py(정제 + 몽고 쓰기) → MongoDB
[바꾼 뒤]   ott.maria.loaded → load_mongo.py(정제만) → ott.mongo.docs → mongo-sink → MongoDB
```

`load_mongo.py` 카프카 모드는 문서를 `bson.json_util`의 확장 JSON으로 바꿔 `ott.mongo.docs`에 보냅니다 (key = CONTENT_ID). 그냥 JSON으로 보내면 `collected_at`·`updated_at`이 문자열이 되므로, `{"$date": ...}` 형식으로 보내서 몽고에 **Date 타입** 그대로 들어가게 했습니다.

```python
publish(T_DOCS, str(cid), json.loads(json_util.dumps(doc)), must=True)   # 도착 확인 후 다음 메시지
```

- 몽고가 죽어 있어도 정제는 계속되고, 문서는 토픽에 쌓였다가 몽고가 살아나면 커넥터가 넣습니다.
- 몽고에 직접 쓰는 건 파일 모드(`--rebuild`, `--all`, 옵션 없음)뿐입니다.

> **영어 리뷰 주의 (11/15 확인):** 서버의 `build()`는 영어 리뷰를 `raw/reviews/{key}.json` 파일에서만 읽습니다. 지금 수집은 영어 리뷰를 **원본 안의 `reviews_en`**에 넣고 `raw/reviews/` 파일은 만들지 않으므로, 이대로면 새 작품 문서에 영어 리뷰가 빠집니다. `build()` 첫 줄에 아래 두 줄을 넣어 원본 안의 값을 쓰게 합니다.
> ```python
>     if en_reviews is None:                                   # 예전 raw/reviews/ 파일이 없으면 원본 안의 영어 리뷰
>         en_reviews = (d.get("reviews_en") or {}).get("results")
> ```
>
> 11/15 적용 후, 이미 들어간 문서 중 영어 리뷰가 빠진 작품만 `ott.maria.loaded`로 다시 보내 문서를 다시 만들었습니다 (서비스 중단 없음, Connect가 `_id` 기준으로 덮어씀). 결과: 영어 리뷰가 실제로 있는 작품 3,731편 전부 반영 (1,502편 → 3,731편). 예전 방식이 리뷰가 없는 작품에도 빈 `raw/reviews/` 파일을 만들어 둔 898편은 원래 리뷰가 없는 작품.
> ```bash
> cd /opt/ott-was/pipeline && python3 - <<'EOF'
> import sys; sys.path.insert(0, ".")
> from common import RAW, raw_files, read_json, content_id, mongo_db, publish, flush, T_LOADED
> want = {}
> for k, i, p in raw_files():
>     if ((read_json(p).get("reviews_en") or {}).get("results")) or (RAW / "reviews" / p.name).exists():
>         want[content_id(k, i)] = f"{k}_{i}"
> have = {d["_id"] for d in mongo_db()["media_contents"].find({"reviews.lang": "en"}, {"_id": 1})}
> todo = [(cid, key) for cid, key in want.items() if cid not in have]
> for cid, key in todo:
>     publish(T_LOADED, key, {"content_id": cid})
> flush()
> print(f"ott.maria.loaded 로 {len(todo):,}건 보냄")
> EOF
> ```
> 이 방법은 "문서 규칙을 바꿨을 때 몽고만 다시 만들기"에 그대로 쓸 수 있습니다 (`--all`과 달리 컨슈머를 멈추지 않아도 됨).

### 1) 커넥터 jar

```bash
mkdir -p /opt/kafka/plugins/mongo
V=3.1.3
curl -fL -o /opt/kafka/plugins/mongo/mongo-kafka-connect-$V-all.jar \
  https://repo1.maven.org/maven2/org/mongodb/kafka/mongo-kafka-connect/$V/mongo-kafka-connect-$V-all.jar
chown -R kafka:kafka /opt/kafka/plugins
ls -lh /opt/kafka/plugins/mongo
```

### 2) Connect 설정·서비스 (분산 모드, 힙 256MB)

```bash
cat > /opt/kafka/config/ott-connect.properties <<'EOF'
bootstrap.servers=localhost:9092
group.id=ott-connect
listeners=http://0.0.0.0:8083
plugin.path=/opt/kafka/plugins
key.converter=org.apache.kafka.connect.storage.StringConverter
value.converter=org.apache.kafka.connect.storage.StringConverter
config.storage.topic=connect-configs
offset.storage.topic=connect-offsets
status.storage.topic=connect-status
config.storage.replication.factor=1
offset.storage.replication.factor=1
status.storage.replication.factor=1
EOF

cat > /etc/systemd/system/ott-connect.service <<'EOF'
[Unit]
Description=이거봐유 Kafka Connect (MongoDB Sink)
After=kafka.service
Requires=kafka.service

[Service]
User=kafka
Environment="KAFKA_HEAP_OPTS=-Xms128m -Xmx256m"
ExecStart=/opt/kafka/bin/connect-distributed.sh /opt/kafka/config/ott-connect.properties
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload && systemctl enable --now ott-connect
sleep 40                                   # 처음 뜰 때 30초쯤 걸림
curl -s localhost:8083/connector-plugins | grep -o '"class":"com.mongodb[^"]*"'    # MongoSinkConnector
```

### 3) 커넥터 등록

```bash
curl -s -X PUT localhost:8083/connectors/mongo-sink/config -H 'Content-Type: application/json' -d '{
 "connector.class":"com.mongodb.kafka.connect.MongoSinkConnector",
 "tasks.max":"1",
 "topics":"ott.mongo.docs",
 "connection.uri":"mongodb://192.168.56.101:27017,192.168.56.102:27017,192.168.56.103:27017/?replicaSet=rs0",
 "database":"ott_service",
 "collection":"media_contents",
 "document.id.strategy":"com.mongodb.kafka.connect.sink.processor.id.strategy.ProvidedInValueStrategy",
 "writemodel.strategy":"com.mongodb.kafka.connect.sink.writemodel.strategy.ReplaceOneDefaultStrategy",
 "max.num.retries":"100",
 "retries.defer.timeout":"30000"
}'
sleep 5
curl -s localhost:8083/connectors/mongo-sink/status; echo     # "state":"RUNNING" 이 두 번 (커넥터, 태스크)
```

- `ProvidedInValueStrategy`: 문서 안의 `_id`(= CONTENT_ID)를 그대로 씀
- `ReplaceOneDefaultStrategy`: `_id`로 찾아 통째로 바꾸고 없으면 넣음 (upsert) → 같은 메시지가 두 번 와도 결과가 같음
- 몽고가 오래 죽어 있으면 재시도를 다 쓰고 태스크가 `FAILED`가 될 수 있습니다. 몽고를 살린 뒤 재시작:
  `curl -X POST "localhost:8083/connectors/mongo-sink/restart?includeTasks=true"`

## 5-9. 확인 명령

```bash
systemctl status kafka ott-connect ott-collect ott-load-maria ott-load-mongo --no-pager | grep -E "●|Active"
curl -s localhost:8083/connectors/mongo-sink/status; echo
/opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --all-groups   # LAG = 아직 처리 안 한 메시지 수
tail -n 20 /var/log/ott_collect.log /var/log/ott_load_maria.log /var/log/ott_load_mongo.log
free -m                                   # 메모리 여유 확인 (1.7GB VM)

# 토픽 안의 메시지 몇 개 보기 (key만)
/opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic ott.maria.loaded \
  --from-beginning --max-messages 5 --property print.key=true
```

## 5-10. 카프카가 죽었을 때 (파일 모드)

수집은 원본 파일을 먼저 저장하므로 데이터는 `raw/`에 남아 있습니다. 카프카를 살릴 수 없으면 컨슈머를 멈추고 파일 모드로 적재합니다.

```bash
systemctl stop ott-load-maria ott-load-mongo
cd /opt/ott-was/pipeline
python3 maria/load_maria.py          # DB에 없는 작품 + 오늘 제공 현황
python3 mongo/load_mongo.py          # 몽고에 없는 작품만 (몽고에 직접 씀)
python3 clear_cache.py
```

전부 처음부터 다시 만들 때: `load_mongo.py --rebuild` (컬렉션 비우고 다시 + 인덱스)
