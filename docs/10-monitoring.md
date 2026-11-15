# 10. 통합 관제 (Prometheus + Loki + Grafana + Alloy) — monitor (.110)

```
각 서버 Alloy ──(지표 push: remote_write)──→ Prometheus :9090 ─┐
              └─(로그 push)─────────────────→ Loki :3100 ──────┴→ Grafana :3000 ← NAT 192.168.0.228:3000
```

- 관제 서버가 각 서버를 긁어 오는(pull) 방식이 아니라, **각 서버의 Alloy가 밀어 넣는(push)** 방식입니다. 그래서 관제 대상 서버에는 포트를 새로 열 필요가 없습니다.
- 모든 서버(NAT 포함)에 Alloy를 설치합니다. Grafana는 내부망 + NAT 3000 포트포워딩으로만 봅니다.

## 10-1. 저장소 등록·패키지 설치 [monitor]

```bash
cat > /etc/yum.repos.d/grafana.repo <<'EOF'
[grafana]
name=grafana
baseurl=https://rpm.grafana.com
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://rpm.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
EOF

dnf list grafana alloy loki            # 처음에는 GPG 키 질문 → y
dnf install -y grafana alloy loki
```

## 10-2. Prometheus 수동 설치 [monitor]

Prometheus는 Grafana 저장소에 없어서 GitHub 릴리스에서 받습니다.

```bash
cd /tmp
VER=$(curl -sIL -o /dev/null -w '%{url_effective}' https://github.com/prometheus/prometheus/releases/latest | sed 's#.*/tag/v##')
echo "VER=$VER"
curl -LO https://github.com/prometheus/prometheus/releases/download/v${VER}/prometheus-${VER}.linux-amd64.tar.gz
tar xzf prometheus-${VER}.linux-amd64.tar.gz

useradd --no-create-home --shell /bin/false prometheus
mkdir -p /etc/prometheus /var/lib/prometheus
cd /tmp/prometheus-${VER}.linux-amd64
cp prometheus promtool /usr/local/bin/
cp prometheus.yml /etc/prometheus/
chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
restorecon -Rv /usr/local/bin/prometheus /usr/local/bin/promtool      # SELinux: 복사한 실행 파일 컨텍스트 복구
cd

cat > /etc/systemd/system/prometheus.service <<'EOF'
[Unit]
Description=Prometheus Time Series Database
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
# --web.enable-remote-write-receiver : 각 서버 Alloy가 지표를 직접 밀어 넣을(push) 수 있게 하는 핵심 옵션
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/ \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-remote-write-receiver
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now prometheus
curl -s localhost:9090/-/ready
```

> systemd 서비스 파일은 줄 중간의 `#` 주석을 지원하지 않습니다. `Restart=always # 설명`처럼 쓰면 값이 깨지므로 주석은 반드시 따로 한 줄로 씁니다.

## 10-3. Loki [monitor]

YAML이라 들여쓰기(공백 2칸)가 틀리면 안 뜹니다.

```bash
mkdir -p /etc/loki /var/lib/loki
cat > /etc/loki/config.yml <<'EOF'
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  instance_addr: 127.0.0.1
  path_prefix: /var/lib/loki
  storage:
    filesystem:
      chunks_directory: /var/lib/loki/chunks
      rules_directory: /var/lib/loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h
EOF
chown -R loki:loki /var/lib/loki            # RPM이 만든 loki 계정이 쓸 수 있게

systemctl enable --now loki
sleep 5; curl -s localhost:3100/ready        # ready
ss -tulnp | grep 3100
```

## 10-4. Grafana [monitor]

```bash
systemctl enable --now grafana-server
ss -tulnp | grep 3000
```

처음 로그인 `admin` / `admin` → 비밀번호 변경 화면에서 `<PASSWORD>`

## 10-5. 방화벽

**관제 서버** — 내부망 NIC가 속한 zone에 9090·3100·3000을 엽니다. (원래 문서는 `internal` zone으로 고정했는데, NIC가 `public`에 있으면 안 열리므로 zone을 먼저 확인)

```bash
Z=$(firewall-cmd --get-zone-of-interface=enp0s8); echo "zone=$Z"       # ※ 확인: NIC 이름
for p in 9090 3100 3000; do firewall-cmd --permanent --zone=$Z --add-port=$p/tcp; done
firewall-cmd --reload
firewall-cmd --zone=$Z --list-ports
```

**NAT 서버** — 외부 3000 → monitor 3000 포트포워딩은 [08-nat.md](08-nat.md)에 있습니다.

## 10-6. Alloy 설치 [관제 대상 서버 전부 — monitor·NAT 포함]

```bash
# 관제 서버와 같은 grafana.repo 등록 후 (10-1의 cat > /etc/yum.repos.d/grafana.repo 부분)
dnf install -y alloy

# 시스템 로그 읽기 권한
setfacl -m u:alloy:r /var/log/messages /var/log/secure

# 로그 로테이션으로 파일이 새로 생겨도 권한 유지: 기존 rsyslog 로테이션 설정의 postrotate에 한 줄 추가
grep -q 'u:alloy' /etc/logrotate.d/rsyslog || \
  sed -i '/postrotate/a\        /usr/bin/setfacl -m u:alloy:r /var/log/messages /var/log/secure 2>/dev/null || true' /etc/logrotate.d/rsyslog
logrotate -d /etc/logrotate.d/rsyslog 2>&1 | grep -i error      # 아무것도 안 나오면 OK
```

> 예전 문서처럼 `/etc/logrotate.d/alloy-acl`에 `/var/log/messages`를 따로 적으면, rsyslog 설정에 이미 같은 파일이 있어서 logrotate가 `duplicate log entry` 오류를 내고 그 설정을 건너뜁니다. 그래서 기존 설정에 한 줄 넣는 방식으로 바꿨습니다. 예전 파일이 있으면 `rm -f /etc/logrotate.d/alloy-acl`.

## 10-7. 서버별 Alloy 설정

예전 문서는 백업·web 서버 설정이 `[내용은 위 패턴과 동일]`이라는 자리표시만 있어서, **그대로 실행하면 설정 파일이 깨집니다.** 서버마다 로그 경로만 다르고 나머지는 같으니, 설정 파일을 만들어 주는 스크립트 하나로 통일했습니다.

**`/root/alloy_setup.sh`** (관제 서버에 만들어 두고 ssh로 각 서버에 실행)

```bash
cat > /root/alloy_setup.sh <<'SCRIPT'
#!/bin/bash
# 사용: alloy_setup.sh <job> [추가 로그 파일 ...]      예) alloy_setup.sh mariadb /var/log/mariadb/mariadb.log
#       UNITS="ott-was" alloy_setup.sh was             (systemd journal 로그도 보낼 때)
set -e
JOB=${1:?job 이름}; shift
MON=192.168.56.110

LOGS=""
for f in "$@"; do
  d=$(dirname "$f")
  setfacl -m u:alloy:rx "$d" 2>/dev/null || true
  setfacl -d -m u:alloy:r "$d" 2>/dev/null || true           # 새로 생기는 파일도 (단, 새 파일 권한이 640 이상일 때만 효과)
  setfacl -m u:alloy:r $f 2>/dev/null || true                 # $f 따옴표 없음: *.log 같은 패턴 허용
  LOGS="$LOGS    {\"__path__\" = \"$f\", \"job\" = \"$JOB\", \"instance\" = constants.hostname},
"
done

JOURNAL=""
if [ -n "${UNITS:-}" ]; then
  usermod -aG systemd-journal alloy
  M=$(for u in $UNITS; do printf '_SYSTEMD_UNIT=%s.service ' "$u"; done)
  JOURNAL="loki.source.journal \"units\" {
  matches    = \"${M% }\"
  labels     = {job = \"$JOB\", instance = constants.hostname}
  forward_to = [loki.write.loki_endpoint.receiver]
}"
fi

cat > /etc/alloy/config.alloy <<CONF
// 지표: 이 서버의 CPU·메모리·디스크·네트워크 (node_exporter와 같은 지표)
prometheus.exporter.unix "default" { }

prometheus.scrape "node_metrics" {
  targets         = prometheus.exporter.unix.default.targets
  forward_to      = [prometheus.remote_write.prom_metrics.receiver]
  scrape_interval = "15s"
  scrape_timeout  = "10s"
}

prometheus.remote_write "prom_metrics" {
  endpoint { url = "http://$MON:9090/api/v1/write" }
}

// 로그: 시스템 로그 + 서버별 서비스 로그
local.file_match "system_logs" {
  path_targets = [
    {"__path__" = "/var/log/messages", "job" = "varlogs", "instance" = constants.hostname},
    {"__path__" = "/var/log/secure",   "job" = "varlogs", "instance" = constants.hostname},
$LOGS  ]
}

loki.source.file "log_scrape" {
  targets    = local.file_match.system_logs.targets
  forward_to = [loki.write.loki_endpoint.receiver]
}

$JOURNAL

loki.write "loki_endpoint" {
  endpoint { url = "http://$MON:3100/loki/api/v1/push" }
}
CONF

alloy fmt /etc/alloy/config.alloy >/dev/null && echo "설정 문법 OK"
systemctl enable alloy
systemctl restart alloy
systemctl is-active alloy
SCRIPT
chmod 700 /root/alloy_setup.sh
```

**서버별 실행** (관제 서버에서, 각 서버에 SSH 키가 있다는 전제)

| 서버 | job | 추가 로그 |
|---|---|---|
| mongo1~3 (.101~.103) | `mongodb` | `/var/log/mongodb/mongod.log` |
| maria, maria2 (.104, .109) | `mariadb` | `/var/log/mariadb/mariadb.log`, `/var/log/backup_OTT.log` |
| mongobackup (.107) | `backup` | `/var/log/cron`, `/var/log/mongo_backup.log`, `/var/log/cleanup_backup.log` |
| web (.106) | `nginx` | `/var/log/nginx/*.log` |
| was (.105) | `was` | journal `ott-was` |
| redis (.108) | `redis` | `/var/log/valkey/valkey.log` ※ 확인 |
| kafka (.111) | `pipeline` | `/var/log/ott_*.log`, `/opt/kafka/logs/server.log`, journal `ott-connect` |
| nat (.254), monitor (.110) | `system` | 없음 (시스템 로그만) |

```bash
S=/root/alloy_setup.sh
for ip in 192.168.56.101 192.168.56.102 192.168.56.103; do
  ssh root@$ip "bash -s mongodb /var/log/mongodb/mongod.log" < $S; done
for ip in 192.168.56.104 192.168.56.109; do
  ssh root@$ip "bash -s mariadb /var/log/mariadb/mariadb.log /var/log/backup_OTT.log" < $S; done
ssh root@192.168.56.107 "bash -s backup /var/log/cron /var/log/mongo_backup.log /var/log/cleanup_backup.log" < $S
ssh root@192.168.56.106 "bash -s nginx '/var/log/nginx/*.log'" < $S
ssh root@192.168.56.105 "UNITS=ott-was bash -s was" < $S
ssh root@192.168.56.108 "bash -s redis /var/log/valkey/valkey.log" < $S
ssh root@192.168.56.111 "UNITS=ott-connect bash -s pipeline '/var/log/ott_*.log' /opt/kafka/logs/server.log" < $S
ssh root@192.168.56.254 "bash -s system" < $S
bash $S system                                   # 관제 서버 자신
```

- 몽고 로그 폴더(`/var/log/mongodb`)는 mongod 계정 전용이라 `setfacl`로 alloy에 읽기 권한을 줍니다 (스크립트가 함).
- 마리아 로그처럼 로테이션 때 `600` 권한으로 새로 만들어지는 파일은 ACL이 사라질 수 있습니다. Grafana에서 그 서버 로그가 끊기면 위 명령을 그 서버에 다시 실행합니다.
- `/root` 아래 파일(`/root/dbbackup/backup.log` 등)은 `/root` 자체에 실행 권한을 줘야 해서 쓰지 않고, cron이 쓰는 `/var/log/*.log`를 봅니다.

## 10-8. Grafana 설정

**데이터 소스 추가** — Connections → Data sources → Add data source
1. Prometheus → URL `http://192.168.56.110:9090` → Save & test
2. Loki → URL `http://192.168.56.110:3100` → Save & test

**서버 대시보드 가져오기** — Dashboards → New → Import → `1860` (Node Exporter Full) → Load → 데이터 소스 Prometheus 선택 → Import
1860은 한 번에 서버 하나씩 보여 줍니다(위쪽 `instance` 선택).

**지표별 한 패널에 전 서버** — 새 대시보드에 패널을 만들고 아래 쿼리를 넣으면 서버마다 선이 하나씩 그려집니다 (Legend `{{instance}}`).

| 패널 | PromQL |
|---|---|
| CPU 사용률 % | `100 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100` |
| 메모리 사용률 % | `(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100` |
| 디스크 사용률 % (/) | `(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100` |
| 네트워크 수신 bytes/s | `sum by (instance) (rate(node_network_receive_bytes_total{device!="lo"}[1m]))` |
| 서버 살아 있음 | `up` |

**로그 보기** — Explore → Loki → 예: `{job="mariadb"}`, `{job="pipeline"} |= "대기"` (카프카 적재가 DB를 기다리는 로그), `{job="varlogs", instance="maria"} |= "keepalived"`
