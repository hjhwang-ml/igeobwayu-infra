# 8. NAT 서버 — nat (192.168.0.228 / 192.168.56.254)

학원 망(192.168.0.x)의 다른 PC가 내부 서버에 들어오는 유일한 입구입니다.

| 외부 포트 | 내부로 | 용도 |
|---|---|---|
| 443 | web 192.168.56.106:443 | 서비스 화면 |
| 3000 | monitor 192.168.56.110:3000 | Grafana 관제 화면 |

web VM을 복제해서 만들었으므로 **Tailscale은 지웁니다** (web과 Tailscale IP가 겹침).

```bash
dnf -y remove tailscale 2>/dev/null; true

# 인터페이스 역할  ※ 확인: NIC 이름 (nmcli con show)
nmcli con mod enp0s3 connection.zone external      # 브리지 192.168.0.228
nmcli con mod enp0s8 connection.zone internal      # 192.168.56.254
nmcli con up enp0s3; nmcli con up enp0s8
firewall-cmd --get-active-zones

# 패킷 전달 + 마스커레이드
echo "net.ipv4.ip_forward = 1" > /etc/sysctl.d/99-forward.conf && sysctl --system | grep ip_forward
firewall-cmd --permanent --zone=external --add-masquerade

# 포트포워딩
firewall-cmd --permanent --zone=external --add-forward-port=port=443:proto=tcp:toport=443:toaddr=192.168.56.106
firewall-cmd --permanent --zone=external --add-forward-port=port=3000:proto=tcp:toport=3000:toaddr=192.168.56.110
firewall-cmd --reload
firewall-cmd --zone=external --list-all
```

- `--add-forward-port`가 있으면 그 포트를 따로 열지 않아도 됩니다 (forward-port가 받아서 넘김).
- Rocky 10에는 `/etc/sysctl.conf`가 비어 있거나 없을 수 있어서 `/etc/sysctl.d/`에 파일로 넣었습니다.
- 확인: 다른 PC 브라우저에서 `https://192.168.0.228` (서비스), `http://192.168.0.228:3000` (Grafana)
