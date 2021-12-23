---
title: Linux三级等保踩坑日记
date: 2021-12-23 10:28:01
updated: 2021-12-23 10:28:01
tags:
 - Linux
 - 三级等保
 - 安全基线
---



## 踩坑1：服务器无法登录

1. 禁用 `root` 帐号登录
   1. 修改 `/etc/ssh/sshd_config` 文件增加 `PermitRootLogin no` 配置
2. 设置密码有效期；例如：设置密码有效期30天，30天后密码过期，必须修改密码才能登录
   1. 修改 `/etc/login.defs` 文件增加 `PASS_MAX_DAYS 30` 配置
3. 设置文件属性：`chattr +i /etc/shadow`



静候30天你会发现惊喜：登录服务器提示必须修改密码，但是修改密码失败

