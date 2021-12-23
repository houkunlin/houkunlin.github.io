---
title: Docker 容器中文字体乱码、无法加载中文字体
date: 2021-12-23 10:38:42
updated: 2021-12-23 10:38:42
tags:
 - docker
 - 容器字体
---



在使用 `java:8-alpine` 镜像打包 java 项目运行后发现有些需要生成含中文文字图片的接口出现异常，或者生成的中文乱码，主要是容器没有中文字体导致的。

可以在打包容器的使用增加字体安装命令：

```sh
apk add ttf-dejavu fontconfig
```

