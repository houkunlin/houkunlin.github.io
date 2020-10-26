---
title: 一些Docker镜像使用
date: 2020-09-19 14:32:45
updated: 2020-09-19 14:32:45
tags:
- docker
---



### Portainer

```bash
#!/bin/bash

docker run --name portainer -d -p 8000:8000 -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock -v /opt/portainer/data:/data -e TZ=Asia/Shanghai portainer/portainer
```



### Rabbitmq

```bash
#!/bin/bash

# 无WEB管理界面
docker run --name rabbitmq -p 4369:4369 -p 5671:5671 -p 5672:5672 -p 25672:25672 -e TZ=Asia/Shanghai -d rabbitmq
# 有WEB管理界面
docker run --name rabbitmq -p 4369:4369 -p 5671:5671 -p 5672:5672 -p 25672:25672 -e TZ=Asia/Shanghai -d rabbitmq:3.8.7-management
```



### MySQL

```bash
#!/bin/bash

docker run --name mysql8 -p 3306:3306 -v /opt/mysql/conf:/etc/mysql/conf.d -v /opt/mysql/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 -e TZ=Asia/Shanghai -d mysql:8.0 --innodb-use-native-aio=0
```



### Redis

```bash
#!/bin/bash

docker run --name redis -p 6379:6379 -e TZ=Asia/Shanghai -d redis
```



### ipsec-vpn-server

```bash
#!/bin/bash

# 这里有个 vpn.env 环境变量文件请自行配置，请自行查阅相关文档
docker run --name ipsec-vpn-server --env-file ./vpn.env -p 500:500/udp -p 4500:4500/udp -d --privileged hwdsl2/ipsec-vpn-server
```



### Teamcity

```bash
#!/bin/bash

docker run -it --name teamcity-server-instance -v /opt/teamcity/data:/data/teamcity_server/datadir -v /opt/teamcity/logs:/opt/teamcity/logs -p 8111:8111 jetbrains/teamcity-server
```



### Shadowsocks

```bash
#!/bin/bash

docker run --name shadowsocks --restart always -e PASSWORD=密码 -p 8388:8388 -p 8388:8388/udp -d shadowsocks/shadowsocks-libev
```



### Squid 代理服务器

```bash
#!/bin/bash

docker run --name squid --restart always -p 3128:3128 -p 3128:3128/udp -v /opt/squid/cache/:/var/spool/squid -v /opt/squid/squid.conf:/etc/squid/squid.conf -d sameersbn/squid

# 要在正在运行的实例上重新加载Squid配置，可以将HUP信号发送到容器。 https://hub.docker.com/r/sameersbn/squid
# docker kill -s HUP squid
```

