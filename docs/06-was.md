# 6. WAS 서버 (Gunicorn + Flask) — was (.105)

WAS에는 Flask API만 둡니다. 수집·적재는 카프카 서버에서 돌고, WAS의 예전 수집 cron은 지웠습니다.

## 6-1. 설치

프로젝트 서버는 가상환경(venv) 없이 **시스템 파이썬**을 씁니다.

```bash
dnf -y install python3 python3-pip
python3 -m pip install flask gunicorn pymysql pymongo redis
which gunicorn                     # 보통 /usr/local/bin/gunicorn  ※ 확인: 아래 서비스 파일 경로와 맞출 것
mkdir -p /opt/ott-was
```

`/opt/ott-was/`에 배치하는 파일

| 파일 | 내용 |
|---|---|
| `app.py` | Flask 라우트 (`/api/...`) |
| `services.py` | 검색(띄어쓰기 무시)·목록·상세·추천(`_similar_rows`)·통계·자동완성 쿼리 |
| `cache.py` | Redis 기능 모음 (검색 캐시, 최근 본 작품, 추천 무효화, 요청 제한, 캐시 통계) |
| `settings.py` / `db.py` | config.ini 읽기, MariaDB·MongoDB·Redis 연결 |
| `config.ini` | 접속 정보 (비밀번호 있음, 레포에 올리지 말 것) |

안 쓰는 예전 파일(수집 스크립트 등)은 지우지 않고 압축 파일 하나로 묶어 보관합니다.

## 6-2. `config.ini`

```ini
[mariadb]
host = 192.168.56.200              ; VIP (Primary가 바뀌어도 그대로)
port = 3306
user = ott
password = <PASSWORD>
database = OTT

[mongodb]
uri = mongodb://192.168.56.101:27017,192.168.56.102:27017,192.168.56.103:27017/?replicaSet=rs0&readPreference=secondaryPreferred
database = ott_service

[redis]
host = 192.168.56.108
port = 6379
password = <PASSWORD>

[cache]                            ; 초 단위  ※ 확인: 실제 키 이름
search_ttl = 3600
recommend_ttl = 3600
recent_ttl = 604800
content_ttl = 86400
recent_max = 10

[app]
session_days = 30                  ; 세션 쿠키 ott_sid 유지 기간
```

## 6-3. systemd `/etc/systemd/system/ott-was.service`

```ini
[Unit]
Description=igeobwayu WAS (Gunicorn + Flask)
After=network-online.target
Wants=network-online.target

[Service]
WorkingDirectory=/opt/ott-was
ExecStart=/usr/local/bin/gunicorn -w 4 -b 0.0.0.0:5000 --timeout 60 app:app
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload && systemctl enable --now ott-was
# 5000은 web 서버에서만 받음
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.56.106" port port="5000" protocol="tcp" accept'
firewall-cmd --reload
curl -s "http://127.0.0.1:5000/api/suggest?q=오징" | head
```

- 서비스 이름은 `ott-was`입니다 (`gunicorn`이 아님). 설정을 바꾸면 `systemctl restart ott-was`.
- Gunicorn 워커 4개 = 요청을 동시에 4개까지 처리. DB·Redis 연결은 워커마다 따로 맺습니다.
