---
title: Linux sysctl 内核参数配置
date: 2023-04-20 17:09:01
updated: 2023-04-20 17:09:01
tags:
  - Linux
---

在安装完 Linux 后，通常需要对内核做一些优化配置，以下是一些示例配置，创建 `/etc/sysctl.d/user.conf` 文件并编辑：
```hocon
# vm.max_map_count=65530 # 默认最大虚拟内存
vm.max_map_count=262144
```

然后执行命令 `sysctl -p /etc/sysctl.d/user.conf` 应用修改


或者修改 `/etc/sysctl.conf` 文件，并增加以上内容，然后执行 `sysctl -p` 应用修改
