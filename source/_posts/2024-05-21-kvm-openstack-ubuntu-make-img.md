---
title: 制作 OpenStack 虚拟机镜像（Ubuntu Server 镜像）
date: 2024-05-21 16:07:26
updated: 2024-05-21 16:07:26
tags:
  - OpenStack
---
> 关键词：制作 OpenStack 虚拟机镜像；使用 QEMU 制作虚拟机镜像；制作 KVM 虚拟机镜像；


最近在使用 OpenStack KVM 虚拟化时，想要创建一个 Ubuntu Server 虚拟机，发现无法直接用 ISO 镜像安装新的虚拟机，后来查阅资料发现需要先制作虚拟机镜像，然后才能创建虚拟机。

## 1. 准备工作

- 下载 Ubuntu Server ISO 镜像
- 安装 qemu 虚拟机工具（windows下可用Linux子系统安装，也可以直接安装）

## 2. 制作 Ubuntu Server 虚拟机镜像

1. 创建一个 8G 大小的虚拟机镜像文件
```bash
qemu-img create -f raw ubuntu-24.04-live-server-amd64.img 8g
```
2. 通过 ISO 镜像启动虚拟机
```bash
# 图形化界面启动
qemu-system-x86_64 -enable-kvm -smp 2 -m 1024  -cdrom ubuntu-24.04-live-server-amd64.iso -drive file=ubuntu-24.04-live-server-amd64.img -boot d
# VNC 服务启动
qemu-system-x86_64 -enable-kvm -smp 2 -m 1024  -cdrom ubuntu-24.04-live-server-amd64.iso -drive file=ubuntu-24.04-live-server-amd64.img -vnc :0
```
虚拟机启动后，需要慢慢等待，然后就可以看到正常的 Ubuntu Server 安装界面了，根据Ubuntu Server 的安装步骤，安装完成后，就可以退出虚拟机。
3. 验证安装是否成功
```bash
# 启动虚拟机
# 图形化界面启动
qemu-system-x86_64 -enable-kvm -smp 2 -m 1024 -drive file=ubuntu-24.04-live-server-amd64.img -boot c
# VNC 服务启动
qemu-system-x86_64 -enable-kvm -smp 2 -m 1024 -drive file=ubuntu-24.04-live-server-amd64.img -vnc :0
```
4. 转换为 qcow2 格式（可以忽略）
```bash
qemu-img convert -f raw -O qcow2 ubuntu-24.04-live-server-amd64.img ubuntu-24.04-live-server-amd64.qcow2
```

## 参考链接
1. [OpenStack镜像制作 # Ubuntu镜像](https://ztblog.readthedocs.io/en/latest/openstack/image-create-guide.html#ubuntu)
