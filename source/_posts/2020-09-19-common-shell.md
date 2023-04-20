---
title: 一些Shell命令使用
date: 2020-09-19 14:49:54
updated: 2020-09-19 14:49:54
tags:
- shell
---

### acme.sh

```bash
#!/bin/bash

# 安装 acme.sh
curl  https://get.acme.sh | sh

# 暴露阿里云DNS的KEY，请到阿里云控制台申请
export Ali_Key="***"
export Ali_Secret="***"

# 使用阿里云DNS验证域名来生成 SSL 证书，acme.sh暂时无法为中文域名签发证书
acme.sh --issue --dns dns_ali -d houkunlin.cn -d *.houkunlin.cn

# 使用 acme.sh 生成 SSL 证书后，把 SSL 证书安装到 Nginx
acme.sh --installcert -d houkunlin.cn --key-file /etc/nginx/houkunlin.cn.key --fullchain-file /etc/nginx/houkunlin.cn.cer --reloadcmd "systemctl force-reload nginx"
```

