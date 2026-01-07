---
title: OpenStack KVM CentOS 7.9 LVM 扩容磁盘（不新增分区，维持原有分区）
date: 2026-01-07 16:10:22
updated: 2026-01-07 16:10:22
tags:
  - OpenStack
  - LVM
  - LVM扩容
---

> 关键词：OpenStack KVM CentOS 7.9 LVM 扩容磁盘；虚拟机 CentOS 7.9 LVM 扩容磁盘；CentOS 7.9 LVM 扩容磁盘；扩容 LVM 分区空间；扩容 LVM 分区；扩容 LVM 卷；

## 如何在不改变分区结构的情况下扩容 LVM 分区空间
> 条件：空闲空间紧跟在 LVM 分区后面，并且没有其他分区。

根据镜像创建一个实例，但是实例的磁盘空间太小，所以我们需要扩容磁盘。

因为我的镜像只有 8G，这个容量肯定是不够用的，所以在创建实例的时候，我把卷大小设置为 20G。

```text
|+++++8g+++++|----------12g----------|
原始镜像占了8g，我创建的卷有20g，那么后面就有12g的空闲空间。
```
而其中8g空间分了三个分区，
1. `/dev/vda1` BOOT 分区
2. `/dev/vda2` LVM 分区，系统分区

```
[root@host centos]# lsblk
NAME            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vda             252:0    0    8G  0 disk
├─vda1          252:1    0    1G  0 part /boot
└─vda2          252:2    0    7G  0 part
  ├─centos-root 253:0    0  6.2G  0 lvm  /
  └─centos-swap 253:1    0  820M  0 lvm  [SWAP]
```

剩下12g的空闲空间紧跟着在 `/dev/vda2` 分区后面。

理论上，我要做的就是把 `/dev/vda2` 分区扩容到最大，然后再把 LVM 分区扩容到最大，看起来步骤非常简单。

## 扩容 `/dev/vda2` 分区
参考：https://serverfault.com/a/1021192

在 CentOS 7.9 上是 `/dev/vda2` 分区
```bash
growpart /dev/vda 2
```

如果提示了类型下面的内容，可以先不用管，这是因为分区已经扩充到最大了，没有更多可扩充的范围了
```
[root@host centos]# growpart /dev/vda 2
NOCHANGE: partition 2 is size 60815327. it cannot be grown
```

如果不存在这个命令需要进行安装
```bash
yum install cloud-utils-growpart gdisk
```

## LVM 重新识别分区大小

```bash
pvresize /dev/vda2
```

## 扩容 LVM 分区

```bash
lvdisplay # 查看 LV 路径
lvextend -l +100%FREE /dev/mapper/centos-root -r
```

## 提供一个在 CentOS 7.9 上的脚本

```bash
#!/bin/bash

# 备份原来的源配置
sudo mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak

# 阿里云源
sudo curl -o /etc/yum.repos.d/Centos7-aliyun.repo https://mirrors.wlnmp.com/centos/Centos7-aliyun-x86_64.repo
# 网易云原
# curl -o /etc/yum.repos.d/Centos7-163.repo https://mirrors.wlnmp.com/centos/Centos7-163-x86_64.repo
# 腾讯云原
# curl -o /etc/yum.repos.d/Centos7-tencent.repo https://mirrors.wlnmp.com/centos/Centos7-tencent-x86_64.repo
# 中国科学技术大学源
# curl -o /etc/yum.repos.d/Centos7-ustc.repo https://mirrors.wlnmp.com/centos/Centos7-ustc-x86_64.repo
# 荆楚理工学院源
# curl -o /etc/yum.repos.d/Centos7-jcut.repo https://mirrors.wlnmp.com/centos/Centos7-jcut-x86_64.repo
# 清华源
# curl -o /etc/yum.repos.d/Centos7-tuna.repo https://mirrors.wlnmp.com/centos/Centos7-tuna-x86_64.repo
# 南阳理工学院源
# curl -o /etc/yum.repos.d/Centos7-nyist.repo https://mirrors.wlnmp.com/centos/Centos7-nyist-x86_64.repo

sudo yum clean all
sudo yum makecache

sudo yum install cloud-utils-growpart gdisk

if [ -b /dev/vda ]; then
    sudo growpart /dev/vda 2
elif [ -b /dev/sda ]; then
    sudo growpart /dev/sda 2
fi

sudo pvresize /dev/vda2

sudo lvextend -l +100%FREE /dev/mapper/centos-root -r

```
