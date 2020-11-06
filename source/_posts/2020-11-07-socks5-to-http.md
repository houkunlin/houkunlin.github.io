---
title: 通过 Privoxy 把 Shadowsocks 转成 Http 代理
date: 2020-11-07 02:18:20
updated: 2020-11-07 02:18:20
tags: shell
---



这个脚本放在草稿里面很久了，今天放出来。



通过一个脚本一键把一个普通的 `Ubuntu Docker` 容器变成一个代理转发 `Socks5` 的服务器。

```bash
#!/bin/bash

# https://qastack.cn/superuser/423563/convert-http-requests-to-socks5
# https://gist.github.com/xwsg/5ecd015be95a61875d43df87c451aca4
# https://edxi.github.io/2018/07/09/Shadowsocks_Privoxy_Squid_GFWlist/

# 安装软件
apt update && apt install shadowsocks-libev privoxy systemctl -y

# 替换 Privoxy 配置文件
# 去除注释内容
echo "$(awk '/^[^#]/' /etc/privoxy/config)" >/etc/privoxy/config
# 替换监听IP配置
echo "$(sed 's/127.0.0.1/0.0.0.0/g' /etc/privoxy/config | sed '/listen.*\[.*/'d)" >/etc/privoxy/config
# 加入转发配置
echo "forward-socks5t / 127.0.0.1:1080 ." >>/etc/privoxy/config

# 替换 ss-local 配置文件
cat <<EOF >/etc/shadowsocks-libev/config.json
{
    "server":"server.host",
    "mode":"tcp_and_udp",
    "server_port":8044,
    "local_port":1080,
    "local_address":"0.0.0.0",
    "password":"my password",
    "timeout":60,
    "method":"aes-256-cfb"
}
EOF

# 启动本地 SS
nohup ss-local -c /etc/shadowsocks-libev/config.json >sslocal.log &
# 启动 Privoxy
/etc/init.d/privoxy start

```



接下来就可以使用 `curl` 来测试 HTTP 代理是否正确了

```bash
curl -v ifconfig.pro # 返回本机IP
curl -v -x socks5://127.0.0.1:1080 ifconfig.pro # 返回 SS 代理 IP
curl -v -x http://127.0.0.1:8118 ifconfig.pro # 返回 SS 代理 IP
```



[1]: https://qastack.cn/superuser/423563/convert-http-requests-to-socks5
[2]: https://gist.github.com/xwsg/5ecd015be95a61875d43df87c451aca4
[3]: https://edxi.github.io/2018/07/09/Shadowsocks_Privoxy_Squid_GFWlist/

