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

## 6. 配置用户可注册

编辑 `/var/matrix-synapse-data/homeserver.yaml` 增加两行配置：

```yaml
enable_registration: true
enable_registration_without_verification: true
```

更多的配置请查阅 [官方文档](https://matrix-org.github.io/synapse/latest/welcome_and_overview.html)

## 7. 反向代理

Element App 和 [Element Web](https://app.element.io/) 版需要启用 HTTPS 功能

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

    # 其他的所有请求都转发给后端服务器
    location / {
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

