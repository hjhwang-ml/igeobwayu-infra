# 3. MongoDB 8.0 레플리카셋 rs0 — mongo1~3 (.101~.103)

| 노드 | IP | 역할 | priority |
|---|---|---|---|
| mongo1 | 192.168.56.101:27017 | PRIMARY (쓰기) | 2 — 살아 있으면 항상 Primary |
| mongo2 | 192.168.56.102:27017 | SECONDARY (읽기) | 1 — mongo1 장애 시 승격 후보 |
| mongo3 | 192.168.56.103:27017 | SECONDARY (읽기) | 1 — mongo1 장애 시 승격 후보 |
| WAS | 192.168.56.105 | 앱 서버 | `readPreference=secondaryPreferred` (읽기는 Secondary로) |

DB `ott_service`, 컬렉션 `media_contents` 하나. 작품 1편 = 문서 1개, `_id` = MariaDB `CONTENT_ID`와 같은 값 (영화 TMDB ID×10+1, 시리즈 ×10+2).
문서에는 상세 화면에 보여주기만 하는, 작품마다 개수·모양이 다른 묶음(예고편·이미지·리뷰·시즌·방송사·배역 이름·국가·태그라인·시청 링크)이 들어갑니다. 적재는 Kafka Connect가 합니다 → [05-kafka.md](05-kafka.md)

## STEP 1. 저장소 등록·설치 [mongo1~3 모두]

Rocky 기본 저장소에는 MongoDB가 없어서 공식 저장소를 등록합니다.

```bash
cat > /etc/yum.repos.d/mongodb-org-8.0.repo <<'EOF'
[mongodb-org-8.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/9/mongodb-org/8.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://pgp.mongodb.com/server-8.0.asc
EOF

dnf install -y mongodb-org --nogpgcheck
systemctl enable --now mongod
mongod --version
```

- 저장소 주소가 `redhat/9`인 건 MongoDB 8.0이 RHEL 9용 패키지로 나오기 때문입니다 (Rocky 10에서도 설치·실행됨).
- `--nogpgcheck`: Rocky 10에서 서명 확인 단계에서 설치가 막혀서, 설치할 때만 서명 확인을 건너뛰었습니다. (Rocky 10의 더 엄격한 서명 정책 때문으로 보임 ※ 확인)

## STEP 2. 방화벽 [mongo1~3 모두]

노드끼리 복제, WAS·카프카 Connect·백업 서버 접속에 27017을 씁니다. 내부망에서만 열어 둡니다.

```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.56.0/24" port port="27017" protocol="tcp" accept'
firewall-cmd --reload
firewall-cmd --list-rich-rules
```

## STEP 3. 레플리카셋 설정 [mongo1~3 모두]

`/etc/mongod.conf`에서 아래 두 부분을 고칩니다. **YAML이라 들여쓰기(공백 2칸)가 틀리면 mongod가 안 뜹니다.**

```yaml
net:
  port: 27017
  bindIp: 0.0.0.0

replication:
  replSetName: "rs0"
```

```bash
systemctl restart mongod
systemctl status mongod --no-pager | head -3      # active (running)
```

## STEP 4. 클러스터 초기화 [mongo1에서만]

```bash
mongosh
```

```javascript
rs.initiate()
rs.add("192.168.56.102:27017")
rs.add("192.168.56.103:27017")
rs.status()
```

`rs.status()`의 members에 3개가 보이고 mongo1이 `PRIMARY`, 나머지가 `SECONDARY`면 복제가 시작된 것입니다.

> 처음부터 IP와 priority를 넣어 한 번에 만들어도 됩니다. 이렇게 하면 STEP 5가 필요 없습니다.
> ```javascript
> rs.initiate({ _id: "rs0", members: [
>   { _id: 0, host: "192.168.56.101:27017", priority: 2 },
>   { _id: 1, host: "192.168.56.102:27017", priority: 1 },
>   { _id: 2, host: "192.168.56.103:27017", priority: 1 } ] })
> ```

## STEP 5. 멤버 주소 IP로 통일 + Primary 우선순위 [mongo1에서만]

`rs.initiate()`를 인자 없이 하면 mongo1이 호스트명(`mongo1:27017`)으로 등록됩니다. WAS·카프카가 `/etc/hosts` 없이도 붙을 수 있게 IP로 바꾸고, mongo1의 priority를 2로 올립니다.

```javascript
cfg = rs.conf()
cfg.members[0].host = "192.168.56.101:27017"
cfg.members[0].priority = 2
cfg.members[1].priority = 1
cfg.members[2].priority = 1
rs.reconfig(cfg)

// 확인
rs.status().members.map(m => ({ name: m.name, stateStr: m.stateStr }))
rs.conf().members.map(m => ({ host: m.host, priority: m.priority }))     // 101은 2, 102·103은 1
```

## STEP 6. WAS 읽기를 Secondary로 [WAS .105]

`/opt/ott-was/config.ini`의 `[mongodb]` uri 끝을 `readPreference=secondaryPreferred`로 바꿉니다. (예전에는 `primaryPreferred`라서 읽기도 Primary로 갔음)

```ini
[mongodb]
uri = mongodb://192.168.56.101:27017,192.168.56.102:27017,192.168.56.103:27017/?replicaSet=rs0&readPreference=secondaryPreferred
database = ott_service
```

```bash
systemctl restart ott-was
```

- `secondaryPreferred`: Secondary가 하나라도 있으면 거기서 읽고, 둘 다 죽으면 Primary에서 읽습니다.
- 쓰기는 연결 문자열과 상관없이 항상 Primary로 갑니다 (Kafka Connect가 씀).

## STEP 7. Secondary 읽기 허용·쓰기 차단 확인 [mongo2 또는 mongo3]

```javascript
// mongosh (mongo2)
db.getMongo().setReadPref('secondary')
use ott_service
db.media_contents.findOne()                 // 문서가 나오면 복제·읽기 정상
db.media_contents.insertOne({ test: 1 })    // NotWritablePrimary 오류가 나야 정상
```

## STEP 8. 인덱스

적재 스크립트(`load_mongo.py`)가 처음 돌 때 없으면 만듭니다. 손으로 만들 때는 mongo1에서:

```javascript
use ott_service
db.media_contents.createIndex({ tmdb_id: 1, content_type: 1 }, { unique: true })   // 중복 방지
db.media_contents.createIndex({ "videos.type": 1 })                                  // 예고편 있는 작품 세기
db.media_contents.createIndex({ title: 1 })                                          // 검색창 자동완성 (앞부분 일치)
db.media_contents.getIndexes()
```

데이터를 전부 밀 때는 `drop()` 대신 `deleteMany({})`로 지워야 인덱스가 남습니다.

## 몽고를 쓰는 화면

| 기능 | 쿼리 |
|---|---|
| 상세 화면 | `find_one({_id: CONTENT_ID})` |
| 검색창 자동완성 | `title` 앞부분 일치 (`/^검색어/`, 인덱스 사용) |
| 통계 | `videos.type` 집계 (예고편 있는 작품 수), 방송사별 시리즈 수 |

## 장애 전환 시연

[11-demo.md](11-demo.md) 참고: mongo1 중지 → mongo2·3 중 새 Primary 선출(약 10초) → 서비스·적재 유지 → mongo1 복구 시 priority 2라 Primary를 되찾음.
