---
title: CentOS下使用Docker并且不关闭防火墙
date: 2020-09-19 13:54:51
updated: 2020-09-19 13:54:51
tags:
- docker
- centos
---



通常情况下我们使用CentOS第一件事就是关闭防火墙  `systemctl stop firewalld` 和禁止防火墙自启动 `systemctl disable firewalld` 。但是关闭防火墙后，意味着服务器将会暴露在一种不安全的状态。

而我们使用Docker运行一些服务的时候，外部网络请求因为防火墙的原因将无法正确的请求到Docker容器内部，因此需要对防火墙做一些设置。

主要就是配置防火墙 zone 对应的网络接口，和给指定的 zone 增加 service，以下给出一个示例的 shell 命令：



```bash
#!/bin/bash

# 显示所有的 zone 信息
firewall-cmd --list-all-zones
# 获取默认的 zone
firewall-cmd --get-default-zone
# 获取默认的 service
firewall-cmd --get-services

# 给指定的 zone 添加一个网络接口
#firewall-cmd --zone=trusted --add-interface=docker0
# 给指定的 zone 删除一个网络接口
#firewall-cmd --zone=trusted --remove-interface=docker0

# 从当前默认的 zone 删除一个网络接口
firewall-cmd --remove-interface=docker0
firewall-cmd --remove-interface=br-daf5b7d3832b

# 给指定的 zone 添加Docker的网络接口
firewall-cmd --zone=trusted --add-interface=docker0
# 下面的这个 br-XXXX 的网络接口后面部分根据机器、系统不同会不一样，请按需设置
firewall-cmd --zone=trusted --add-interface=br-daf5b7d3832b

# 给指定的 zone 增加 service ，允许一些指定的网络服务通过该 zone ，不设置这个的话，这个 zone 将会不处理这些类型的网络请求，也就是防火墙拦截掉未设置的 service 请求
firewall-cmd --zone=trusted --add-service=cockpit
firewall-cmd --zone=trusted --add-service=dhcp
firewall-cmd --zone=trusted --add-service=dhcpv6-client
firewall-cmd --zone=trusted --add-service=dns
firewall-cmd --zone=trusted --add-service=mdns
firewall-cmd --zone=trusted --add-service=samba-client
firewall-cmd --zone=trusted --add-service=ssh
firewall-cmd --zone=trusted --add-service=http
firewall-cmd --zone=trusted --add-service=https
firewall-cmd --zone=trusted --add-service=mysql

# 显示指定的 zone 的信息
firewall-cmd --zone=trusted --list-all

# 把以上的配置保存，使重启不会丢失
firewall-cmd --runtime-to-permanent
```

