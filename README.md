# 이거봐유 인프라 구축 워크북

한국 OTT 7곳(넷플릭스·디즈니+·티빙·웨이브·왓챠·프라임 비디오·애플TV)의 작품을 TMDB에서 수집해 검색·추천하는 서비스 **이거봐유**의 서버 구축 기록입니다. 부트캠프 P2 DB 프로젝트 3팀.

- 대상: Rocky Linux 10.2 VM (VirtualBox, 호스트 전용망 `192.168.56.0/24`)
- 비밀번호·토큰은 `<PASSWORD>`, `<VRRP_PW>`(keepalived, 8자 이하), TMDB 토큰, Slack 웹훅(`XXXX`)처럼 자리표시로 적었습니다. 실제 값으로 바꿔서 실행하세요
- `※ 확인` 표시는 서버마다 다를 수 있는 값 (NIC 이름, 포트, 경로)

## 서버 구성

| 호스트명 | IP | 역할 | 주요 서비스 |
|---|---|---|---|
| nat | 192.168.0.228 (브리지) / 192.168.56.254 | 외부 진입, 포트포워딩 | firewalld masquerade (443 → web, 3000 → monitor) |
| web | 192.168.56.106 | 정적 파일 + 리버스 프록시 | Nginx (HTTPS 443) |
| was | 192.168.56.105 | API 서버 | Gunicorn + Flask (`ott-was`, 5000) |
| redis | 192.168.56.108 | 캐시·세션 | Valkey 8 (Redis 호환), 6379 |
| maria | 192.168.56.104 | MariaDB 1번 (평소 Primary) | MariaDB, keepalived |
| maria2 | 192.168.56.109 | MariaDB 2번 (Replica, read_only) | MariaDB, keepalived |
| (VIP) | 192.168.56.200 | MariaDB 접속 주소 | keepalived가 Primary 쪽에 붙임 |
| mongo1 | 192.168.56.101 | MongoDB rs0 Primary (priority 2) | mongod 27017 |
| mongo2 / mongo3 | .102 / .103 | MongoDB rs0 Secondary (priority 1) | mongod 27017 |
| mongobackup | 192.168.56.107 | 백업 서버 (마리아·몽고 덤프 보관) | cron 백업·정리 |
| kafka | 192.168.56.111 | 수집·적재 파이프라인 | Kafka 4.4.0 KRaft, Kafka Connect, ott-collect, ott-load-maria/mongo |
| monitor | 192.168.56.110 | 관제 | Prometheus 9090, Loki 3100, Grafana 3000 |

## 흐름

```
[사용자]  브라우저 → https://192.168.0.228 → NAT → web(Nginx) → WAS(Flask)
                                                                 ├─ MariaDB VIP .200  (검색·목록·추천: 정형 12테이블)
                                                                 ├─ MongoDB rs0       (상세 화면 묶음·자동완성, 읽기는 Secondary)
                                                                 └─ Redis(Valkey)     (검색 캐시·최근 본 작품·추천)

[데이터]  collect.py (수집, Producer)
            → ott.content / ott.providers
            → load_maria.py (정제 + MariaDB 적재, Consumer)
            → ott.maria.loaded
            → load_mongo.py (문서 만들기, Consumer)
            → ott.mongo.docs
            → mongo-sink (Kafka Connect) → MongoDB
          적재가 한동안 없으면 clear_cache.py → Redis 검색·추천 캐시 비우기

[운영]    0시 백업 (마리아 Replica 덤프, 몽고 Secondary 덤프 → mongobackup) · Slack 알림
          모든 서버 Alloy → Prometheus(지표) / Loki(로그) → Grafana
```

## 문서

| 순서 | 문서 | 내용 |
|---|---|---|
| 1 | [docs/01-common.md](docs/01-common.md) | 모든 VM 공통 준비 (호스트명·IP·hosts·시간) |
| 2 | [docs/02-mariadb.md](docs/02-mariadb.md) | MariaDB 2대 GTID 복제 + keepalived VIP 자동 전환 + 복구 |
| 3 | [docs/03-mongodb.md](docs/03-mongodb.md) | MongoDB 8.0 레플리카셋 rs0 + 읽기 분산 |
| 4 | [docs/04-redis.md](docs/04-redis.md) | Valkey 설치·설정, 서비스에서 쓰는 키 |
| 5 | [docs/05-kafka.md](docs/05-kafka.md) | Kafka 4.4.0 KRaft, 파이프라인 서비스, Kafka Connect(MongoDB Sink) |
| 6 | [docs/06-was.md](docs/06-was.md) | Gunicorn + Flask |
| 7 | [docs/07-web.md](docs/07-web.md) | Nginx HTTPS + 리버스 프록시 |
| 8 | [docs/08-nat.md](docs/08-nat.md) | NAT 서버 포트포워딩 |
| 9 | [docs/09-backup.md](docs/09-backup.md) | 백업 전략·스크립트·복원 |
| 10 | [docs/10-monitoring.md](docs/10-monitoring.md) | Prometheus + Loki + Grafana + Alloy |
| 11 | [docs/11-demo.md](docs/11-demo.md) | 발표 시연 순서 (장애 전환) |
| 12 | [docs/12-checklist.md](docs/12-checklist.md) | 전체 점검 체크리스트 |

구축 순서는 표 순서대로 하면 됩니다 (DB → 캐시 → 파이프라인 → 서비스 → 외부 진입 → 운영).
예전 문서와 달라진 점은 [CHANGES.md](CHANGES.md)에 정리했습니다.
