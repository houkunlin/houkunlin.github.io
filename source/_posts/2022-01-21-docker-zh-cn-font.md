---
title: Docker 运行 Java 程序时遇到中文字体异常问题解决方案
date: 2022-01-21 22:54:58
updated: 2022-01-21 22:54:58
tags:
- docker
- java
- springboot
---

## 问题描述

在我们日常开发中通常会遇到需要在程序中导出Excel电子表格，或者生成带有中文汉字的图片，此时通常会遇到异常报错，一般情况下都是缺少相关字体导致的。

此时我们需要在 `docker` 容器中安装相关字体应用程序库，以及加入中文字体到 `docker` 容器中。

**宋体** 字库在 `Windows` 是系统的 `C:\Windows\Fonts\simsun.ttc` 字体文件，可能需要把字体转换成 `ttf` 格式字体 `simsun.ttf` （没有在容器内试过`ttc`） 



# `openjdk:8-alpine` 镜像（`alpine` 系统）

> 需要执行 `apk add ttf-dejavu fontconfig` 安装相关字体库


```dockerfile
FROM openjdk:8-alpine
LABEL maintainer=HouKunLin
# 安装相关字库程序
RUN apk add ttf-dejavu fontconfig
# 加入中文字体文件到指定路径中
ADD ./simsun.ttf /usr/share/fonts/simsun.ttf
# Docker 容器中通常遇到时区问题，因此可以加上这个环境变量把时间变为东八区
ENV TZ Asia/Shanghai
WORKDIR /app
COPY app.jar ./
ENTRYPOINT ["java", "-jar", "-Xms512m", "-Xmx512m", "app.jar", "--spring.profiles.active=prod"]
EXPOSE 8080
```



# `openjdk:11-jre` 镜像（`Debian` 系统）

> 需要执行 `apt-get install -y fontconfig libfreetype6` 安装相关字体库

```dockerfile
FROM openjdk:11-jre
LABEL maintainer=HouKunLin
# 安装相关字库程序
RUN apt-get install -y fontconfig libfreetype6
# 加入中文字体文件到指定路径中
ADD ./simsun.ttf /usr/share/fonts/simsun.ttf
# Docker 容器中通常遇到时区问题，因此可以加上这个环境变量把时间变为东八区
ENV TZ Asia/Shanghai
WORKDIR /app
COPY app.jar ./
ENTRYPOINT ["java", "-jar", "-Xms512m", "-Xmx512m", "app.jar", "--spring.profiles.active=prod"]
EXPOSE 8080

```



# CentOS 系统可参考下方链接

- https://segmentfault.com/a/1190000040275198


# 切换到 JDK 11 容器镜像是遇到的问题

之前在切换到 `openjdk:11-jre-slim` 镜像后导出Excel遇到一个问题 `NoClassDefFoundError: Could not initialize class sun.awt.X11FontManager` 异常，换成 `openjdk:11-jre` 镜像即可。


**相关链接**
- https://stackoverflow.com/questions/55454036/noclassdeffounderror-could-not-initialize-class-sun-awt-x11fontmanager
- https://stackoverflow.com/questions/53375613/why-is-the-java-11-base-docker-image-so-large-openjdk11-jre-slim
- https://medium.com/azulsystems/using-jlink-to-build-java-runtimes-for-non-modular-applications-9568c5e70ef4
