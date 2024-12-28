---
title: "CTFd 리버스 프록시 이용하여 https 설정"
author: d0razi
date: 2024-12-28 21:46
categories: [Dev, Web]
tags: []
image: /assets/img/media/banner/Dev_web.png
---

# CTFd 설치

```bash
docker run -p 8000:8000 -itd ctfd/ctfd
```

# https 설정

## Let’s Encrypt를 사용해 HTTPS 인증서 발급

```bash
sudo apt update
sudo apt install -y certbot python3-certbot-nginx
```

## Nginx 설치 및 구성

### 설치

```bash
sudo apt install -y nginx
```

### 설정 파일

```bash
sudo vi /etc/nginx/sites-available/ctfd
```

```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/ctfd /etc/nginx/sites-enabled/
sudo nginx -t  # 설정 테스트
sudo systemctl reload nginx
```

## HTTPS 설정

### Certbot으로 HTTPS 인증서 발급 및 설정

```bash
sudo certbot --nginx -d your-domain.com
```

```bash
sudo systemctl reload nginx
```