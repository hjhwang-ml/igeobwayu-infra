# 1. 모든 VM 공통 준비

서버마다 호스트명과 IP만 바꿔서 실행합니다. (예: maria .104)

```bash
# 호스트명
hostnamectl set-hostname maria

# 호스트 전용망 고정 IP (연결 이름은 nmcli con show 로 확인)  ※ 확인: NIC 이름
nmcli con mod enp0s8 ipv4.method manual ipv4.addresses 192.168.56.104/24
nmcli con up enp0s8

# /etc/hosts 에 서로 이름 등록 (모든 서버 같게)
cat >> /etc/hosts <<'EOF'
192.168.56.101 mongo1
192.168.56.102 mongo2
192.168.56.103 mongo3
192.168.56.104 maria
192.168.56.105 was
192.168.56.106 web
192.168.56.107 mongobackup
192.168.56.108 redis
192.168.56.109 maria2
192.168.56.110 monitor
192.168.56.111 kafka
192.168.56.254 nat
EOF

dnf -y update
dnf -y install vim wget tar rsync chrony acl
systemctl enable --now chronyd firewalld
timedatectl set-timezone Asia/Seoul
```

- `acl`은 관제(Alloy)가 로그 파일을 읽을 때 쓰는 `setfacl` 때문에 넣었습니다. (Rocky 기본 설치에 보통 들어 있음)
- 시간대를 맞춰야 cron 백업(0시)·수집(23시)·로그 시각이 서버끼리 어긋나지 않습니다.
- SELinux는 켜 둔 채로 구축했습니다(Enforcing). 막히는 곳은 문서에 따로 적었습니다 (Nginx → WAS 프록시, Prometheus 실행 파일).
- VM을 복제해서 만들었으면 `/etc/machine-id`와 Tailscale 같은 고유값이 겹치지 않는지 확인합니다. (NAT 서버는 web 복제라 Tailscale을 지웠음)
