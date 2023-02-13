---
title: Matrix Synapse 服务安装(Docker)
date: 2023-02-09 10:29:35
updated: 2023-02-09 10:29:35
tags:
---

## 1. 创建一个工作目录

```bash
mkdir -p /var/matrix-synapse-data/
```

## 2. 生成 Synapse 配置文件

```bash
docker run -it --rm -v /var/matrix-synapse-data/:/data/ -e SYNAPSE_SERVER_NAME=matrix.houkunlin.cn -e SYNAPSE_REPORT_STATS=no matrixdotorg/synapse:latest generate
```

- `-v /var/matrix-synapse-data/:/data/` 映射的具体路径，按需修改
- `-e SYNAPSE_SERVER_NAME=matrix.houkunlin.cn` 域名
- `-e SYNAPSE_REPORT_STATS=no` 是否发送匿名统计数据

### 3. 安装运行

```bash
docker run -d --name synapse -v /var/matrix-synapse-data/:/data/ -p 8008:8008 -p 8009:8009 -p 8448:8448 matrixdotorg/synapse:latest
```

### 4. 创建用户

```bash
docker exec -it synapse register_new_matrix_user http://localhost:8008 -c /data/homeserver.yaml -a -u 用户名 -p 密码
```

## 5. 安装管理面板

```bash
docker run -d -p 12234:80 awesometechnologies/synapse-admin
```

## 6. 配置 synapse

编辑 `/var/matrix-synapse-data/homeserver.yaml` 

配置可注册：

```yaml
# 为新用户启用注册。
# https://matrix-org.github.io/synapse/latest/usage/configuration/config_documentation.html#enable_registration
enable_registration: true
# 无需电子邮件或验证码验证即可启用注册。
# https://matrix-org.github.io/synapse/latest/usage/configuration/config_documentation.html#enable_registration_without_verification
enable_registration_without_verification: true
```

配置其他服务器与我们的服务器通信（更多详情可参考 [Delegation of incoming federation traffic](https://matrix-org.github.io/synapse/latest/delegate.html) ）：

```yaml
# 客户端用于访问此 Homeserver 的面向公众的基本 URL，这与用户可能在其客户端的“自定义主服务器 URL”字段中输入的 URL 相同。如果您将 Synapse 与反向代理一起使用，这应该是通过代理访问 Synapse 的 URL。
# https://matrix-org.github.io/synapse/latest/usage/configuration/config_documentation.html#public_baseurl
public_baseurl: https://matrix.houkunlin.cn
# 默认情况下，其他服务器将尝试通过端口 8448 访问我们的服务器，告诉其他服务器将流量发送到端口 443
# https://matrix-org.github.io/synapse/latest/usage/configuration/config_documentation.html#serve_server_wellknown
serve_server_wellknown: true
# 是否应阻止对该服务器上用户的房间邀请（本地服务器管理员发送的除外）
# https://matrix-org.github.io/synapse/latest/usage/configuration/config_documentation.html#block_non_admin_invites
block_non_admin_invites: false
```

配置访客匿名查看公开房间：

```yaml
# 允许用户在没有密码/电子邮件/等的情况下注册为访客，并参与此服务器上托管的房间，匿名用户可以访问这些房间。
# https://matrix-org.github.io/synapse/latest/usage/configuration/config_documentation.html#allow_guest_access
allow_guest_access: true
```

更多的配置请查阅 [官方文档](https://matrix-org.github.io/synapse/latest/welcome_and_overview.html)

## 7. 反向代理（HTTP+HTTPS）

Element App 和 [Element Web](https://app.element.io/) 版需要启用 HTTPS 功能，下面是 HTTP+HTTPS 合并配置：

```
map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
}
server {
    listen 80;
    listen [::]:80;
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    server_name matrix.houkunlin.cn;

    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets on;
    #按照这个协议配置
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    #按照这个套件配置
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES256-SHA256:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA;
    #指定当使用 SSLv3 和 TLS 协议时，服务器密码应优先于客户端密码。
    ssl_prefer_server_ciphers on;
    ssl_stapling on;
    #启用或禁用服务器对 OCSP 响应的验证。
    ssl_stapling_verify on;

    #HSTS策略, 一年：31536000 ，180天：15552000，30天：2592000
    add_header Strict-Transport-Security "max-age=15552000; includeSubDomains; preload" always;

    #防XSS攻击
    add_header X-Xss-Protection 1;

    ssl_certificate https/houkunlin.cn/domain.cer;
    ssl_certificate_key https/houkunlin.cn/domain.key;

    ignore_invalid_headers off;
    client_max_body_size 0;
    proxy_read_timeout 600s;

    error_page 403 404 500 502 503 504 /index.html;

    index index.htm index.html;

    # 访问 Element.io 的 WEB 客户端
    location / {
        alias /var/www/matrix/;
        try_files $uri $uri/ =404;
    }

    # 把 /.well-known/ 和 /_matrix/ 路径下的请求都转发给后端服务器
    location ~ ^/(_matrix|.well-known)/ {
        proxy_pass                          http://127.0.0.1:8008;
        proxy_set_header Host               $http_host;
        proxy_set_header Upgrade            $http_upgrade;
        proxy_set_header Connection         $connection_upgrade;
        proxy_set_header X-Proxy-Host       $proxy_host;
        proxy_set_header X-Forwarded-Host   $host;
        proxy_set_header X-Forwarded-Server $host:$server_port;
        proxy_set_header X-Forwarded-Proto  $scheme;
        proxy_set_header X-Forwarded-For    $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP          $remote_addr;
        proxy_ssl_protocols                 TLSv1.2 TLSv1.3;
    }
}
```

下面提供一个 HTTP 的配置（删掉了HTTPS配置），在使用 [cloudflare.com](http://cloudflare.com) 来保护我们的网站的时候可以免掉 HTTPS 的配置

```
map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
}
server {
    listen 80;
    listen [::]:80;

    server_name matrix.houkunlin.cn;

    ignore_invalid_headers off;
    client_max_body_size 0;
    proxy_read_timeout 600s;

    error_page 403 404 500 502 503 504 /index.html;

    index index.htm index.html;

    # 访问 Element.io 的 WEB 客户端
    location / {
        alias /var/www/matrix/;
        try_files $uri $uri/ =404;
    }

    # 把 /.well-known/ 和 /_matrix/ 路径下的请求都转发给后端服务器
    location ~ ^/(_matrix|.well-known)/ {
        proxy_pass                          http://127.0.0.1:8008;
        proxy_set_header Host               $http_host;
        proxy_set_header Upgrade            $http_upgrade;
        proxy_set_header Connection         $connection_upgrade;
        proxy_set_header X-Proxy-Host       $proxy_host;
        proxy_set_header X-Forwarded-Host   $host;
        proxy_set_header X-Forwarded-Server $host:$server_port;
        proxy_set_header X-Forwarded-Proto  $scheme;
        proxy_set_header X-Forwarded-For    $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP          $remote_addr;
        proxy_ssl_protocols                 TLSv1.2 TLSv1.3;
    }
}
```
