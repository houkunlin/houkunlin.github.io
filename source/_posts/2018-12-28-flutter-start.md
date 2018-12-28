---
title: 开始 flutter hello word 遇到问题记录
date: 2018-12-28 09:16:21
updated: 2018-12-28 09:16:21
tags:
- flutter
---
 ### 参考教程
- flutter中文网: [https://flutterchina.club/get-started/install/](https://flutterchina.club/get-started/install/)

#### 基本步骤跟着中文网教程走就行,下面记录一些在过程中遇到的问题

开发环境: Ubuntu + IDEA + flutter插件 + Android插件

## 问题列表
- 运行时提示JAVA_HOME不存在问题
    - 解决方法: 把`~/.bashrc`中的环境变量配置移到`/etc/profile`然后重启电脑
- 手机连上电脑有显示,但是无权限问题(通过`adb devices`命令可以检验),或者提示如下内容: `Error retrieving device properties for ro.product.cpu.abi: error: insufficient permissions for device: user in plugdev group; are your udev rules wrong?`
    - 解决方法
        - 参考: [https://developer.android.com/studio/run/device](https://developer.android.com/studio/run/device)
        - 参考: [https://blog.csdn.net/liuxu0703/article/details/60956006](https://blog.csdn.net/liuxu0703/article/details/60956006)
        - 创建文件`/etc/udev/rules.d/51-android.rules`并加入`SUBSYSTEM=="usb", ATTR{idVendor}=="0bb4", MODE="0666", GROUP="plugdev"`到文件,
        其中`ATTR{idVendor}`是手机的制造商ID,可以在控制台通过`lsusb`获得当前手机的制造商ID,
        编辑完文件保存后,执行`chmod a+r /etc/udev/rules.d/51-android.rules`赋予相关权限