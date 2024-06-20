---
title: mitmweb 配置 HTTP Basic Auth
date: 2024/06/05 12:30
updated: 2024/06/05 12:30
tags: [抓包,mitmproxy,mitmweb,渗透测试,安全技术]
categories: 安全技术
description: 苦于官方迟迟不支持HTTP Basic Auth，自己动手丰衣足食
cover: https://i3.wp.com/wx1.sinaimg.cn/large/a15b4afegy1fmvjcuwh2oj21hc0u04qe.jpg
---

##  背景

众所周知，mitmweb是mitmproxy的web界面，当我们写了一些mitmproxy的脚本后，可以动态的拦截修改请求，从而实现一些特殊功能。如果只是自己在电脑面前用还好，但如果想要分享给其他人使用或者自己随时随地使用，就需要把mitmproxy部署到公网上，那么就需要考虑两个安全问题：
1. 代理服务器需要设置密码避免被扫到用作非法行为 -> 这个官方原生支持
2. mitmweb页面需要设置密码避免流量被黑客看到并加以利用 -> 这个官方在早期版本中支持，但后续删掉了这一功能，至今未再次支持


##  解决方案

mitmweb启动命令如下:

```bash
./mitmweb --listen-host 0.0.0.0 --listen-port 33899 --ssl-insecure  --proxyauth admin:123456 --set block_global=false --set onboarding_host=m1.it  --set web_port=8082 --set web_host=127.0.0.1 -s mitm_modify_response.py
```

我们将proxy_server设置在0.0.0.0:33899端口，同时给proxy设置用户名密码。然后将web_ui设置在127.0.0.1:8082端口


nginx配置如下:

```nginx
server {
    listen 80;
    server_name 域名;
    return 301 https://$host$request_uri;
}

server {
    listen 33890;
    server_name 域名;
    location / {
        proxy_set_header Host localhost:33899;
        proxy_set_header Origin http://localhost:33899;
        proxy_pass http://localhost:33899;
        proxy_http_version 1.1;
    }
}

server {
    listen 443 ssl;    
    server_name  域名;
    ssl_certificate  /etc/mitm/cert.pem;
    ssl_certificate_key /etc/mitm/key.pem;
    client_max_body_size 128M;
    server_tokens off;
    charset utf-8;
    location / {
        auth_basic "authentication";
        auth_basic_user_file .htpasswd;
        proxy_set_header Host localhost:8082;
        proxy_set_header Origin http://localhost:8082;
        proxy_pass http://localhost:8082;
        expires off;
        proxy_http_version 1.1;
        proxy_redirect http://localhost:8082 http://localhost;
        proxy_buffering off;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Cookie $http_cookie;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

auth_basic_user_file就是我们设置的HTTP Basic Auth，这个就不赘述了


使用时我们带用户名密码连接 x.x.x.x:33899/域名:33899 即可