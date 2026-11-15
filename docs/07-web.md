# 7. web 서버 (Nginx, HTTPS) — web (.106)

정적 파일(html·js·css)을 주고, `/api/`는 WAS로 넘깁니다. 외부에서는 NAT(192.168.0.228)의 443 → web 443으로 들어옵니다.

## 7-1. 설치

```bash
dnf -y install nginx
mkdir -p /usr/share/nginx/ott            # 정적 파일 위치
```

## 7-2. 인증서 (자체 서명)  ※ 확인: 실제로 쓴 방식

```bash
mkdir -p /etc/nginx/ssl
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/ott.key -out /etc/nginx/ssl/ott.crt \
  -subj "/C=KR/O=igeobwayu/CN=192.168.0.228" \
  -addext "subjectAltName=IP:192.168.0.228,IP:192.168.56.106"
chmod 600 /etc/nginx/ssl/ott.key
```

자체 서명이라 브라우저에 경고가 뜹니다 → "고급 → 계속 진행". 발표 PC에서 미리 한 번 들어가 둡니다.

## 7-3. `/etc/nginx/conf.d/ott.conf`

```nginx
server {
    listen 80;
    return 301 https://$host$request_uri;
}
server {
    listen 443 ssl;
    http2 on;
    server_name _;
    ssl_certificate     /etc/nginx/ssl/ott.crt;
    ssl_certificate_key /etc/nginx/ssl/ott.key;

    root /usr/share/nginx/ott;
    index index.html;

    location /api/ {
        proxy_pass http://192.168.56.105:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_read_timeout 30s;
    }
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

```bash
nginx -t && systemctl enable --now nginx
firewall-cmd --permanent --add-service={http,https} && firewall-cmd --reload
setsebool -P httpd_can_network_connect 1        # SELinux: Nginx → WAS 프록시 허용 (안 하면 502)
curl -k "https://127.0.0.1/api/suggest?q=오징" | head
```

- `http2 on;` 문법은 Nginx 1.25.1 이상입니다 (Rocky 10 기본 Nginx는 해당).
- 요청 제한(분당 횟수)은 Nginx가 아니라 WAS가 Redis `rate:*` 키로 합니다. WAS가 실제 사용자 IP를 알 수 있게 `X-Real-IP`·`X-Forwarded-For`를 넘깁니다.
- 예전 주소 192.168.0.226(web 브리지)은 NAT를 만들면서 껐습니다. 정적 파일 안에 `0.226`이 남아 있으면 안 됩니다 ※ 확인: `grep -rn 0.226 /usr/share/nginx/ott /etc/nginx` 결과가 비어야 함
