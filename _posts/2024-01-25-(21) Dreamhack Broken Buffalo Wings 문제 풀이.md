---
title: (21) Dreamhack Broken Buffalo Wings 문제 풀이
date: 2024-01-25 +09:00
categories:
  - 웹해킹
tags:
  - [웹해킹, 드림핵]
---
{% raw %}
## 문제 설명
Buffalo wings, also known as hot wings or chicken wings...

## 문제 풀이
들어오면 맛있는 버팔로 윙의 사진이 챗GPT의 정성스런 설명과 함께 보입니다.
![](https://kyuyeop.github.io/assets/img/post/21/1.png)
첨부된 코드를 보면 `bot/bot/py`에서 flag가 cookie로 추가되는걸 보고 xss 문제인가 싶지만... 딱히 flag를 얻어낼 수 있는 취약점이 보이질 않습니다.  
  
문제의 핵심은 서버 설정의 문제점입니다.
`config/nginx.conf`를 보면
```
user buffer_wing;
pid /run/nginx.pid;
error_log /dev/stderr info;

events {
    worker_connections 1024;
}

http {
    server_tokens off;
    log_format docker '$remote_addr $remote_user $status "$request" "$http_referer" "$http_user_agent" ';
    access_log /dev/stdout docker;

    charset utf-8;
    keepalive_timeout 20s;
    sendfile on;
    tcp_nopush on;
    client_max_body_size 1M;
    include /etc/nginx/mime.types;

    server {
        listen 8000;
        server_name _;

        index index.php;
        root /app;

        location / {
            try_files $uri $uri/ /index.php?$query_string;
            location ~ \.php$ {
                try_files $uri =404;
                fastcgi_pass unix:/run/php-fpm.sock;
                fastcgi_index index.php;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                include fastcgi_params;
            }
        }
    }
}
```
이렇게 작성되어 있습니다.  
여기서 flag.txt를 읽을 수 없도록 설정하지 않았습니다.  
  
그래서 그냥 `/flag.txt`로 접속하면
![](https://kyuyeop.github.io/assets/img/post/21/2.png)
flag가 그냥 나와버립니다.
{% endraw %}