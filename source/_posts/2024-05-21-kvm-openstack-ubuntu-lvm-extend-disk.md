---
title: OpenStack KVM Ubuntu LVM 扩容磁盘（不新增分区，维持原有分区）
date: 2024-05-21 16:10:22
updated: 2024-05-21 16:10:22
tags:
---

> 关键词：OpenStack KVM Ubuntu LVM 扩容磁盘；虚拟机 Ubuntu LVM 扩容磁盘；Ubuntu LVM 扩容磁盘；扩容 LVM 分区空间；扩容 LVM 分区；扩容 LVM 卷；

## 如何在不改变分区结构的情况下扩容 LVM 分区空间
> 条件：空闲空间紧跟在 LVM 分区后面，并且没有其他分区。

在上一篇文章中我把制作好的 Ubuntu 镜像上传到 OpenStack 上，然后根据镜像创建一个实例，但是实例的磁盘空间太小，所以我们需要扩容磁盘。

因为我的镜像只有 8G，这个容量肯定是不够用的，所以在创建实例的时候，我把卷大小设置为 20G。

```text
|+++++8g+++++|----------12g----------|
原始镜像占了8g，我创建的卷有20g，那么后面就有12g的空闲空间。
```
而其中8g空间分了三个分区，
1. `/dev/vda1` BIOS 分区
2. `/dev/vda2` BOOT 分区
3. `/dev/vda3` LVM 分区，系统分区

剩下12g的空闲空间紧跟着在 `/dev/vda3` 分区后面。

理论上，我要做的就是把 `/dev/vda3` 分区扩容到最大，然后再把 LVM 分区扩容到最大，看起来步骤非常简单。

## 修复磁盘 GPT PMBR size mismatch 问题
在扩容 `/dev/vda3` 分区之前，我需要修复磁盘 GPT PMBR size mismatch 问题，因为分区大小和磁盘大小不一致。
错误提示：
```text
gpt pmbr size mismatch will be corrected by write.
the backup gpt table is not no the end of the device.
```
执行修复命令：
```bash
parted -l
```

## 扩容 `/dev/vda3` 分区
参考：https://serverfault.com/a/1021192
```bash
growpart /dev/vda 3
# resize2fs /dev/vda3 # 发现此命令执行失败，但并不影响后面的操作
```

## 扩容 LVM 分区

```bash
lvdisplay # 查看 LV 路径
lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
resize2fs /dev/ubuntu-vg/ubuntu-lv
```
