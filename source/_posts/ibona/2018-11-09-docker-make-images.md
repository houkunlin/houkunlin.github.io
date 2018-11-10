---
title: 把SpringBoot项目部署到docker中创建自己的镜像
date: 2018-11-09 17:59
updated: 2018-11-10 11:18:34
categories:
 - 深圳博纳移动
 - 学习
 - 教程
tags: 
 - 博纳
 - 学习计划
 - docker
 - spring
 - SpringBoot
---

## 把SpringBoot打包成jar包并部署到docker容器中
目前最流行的SpringBoot框架，简单好用，快速搭建项目，开发web项目时，即可以打包成war部署到tomcat中，也可以直接打包成jar使用`java -jar spring-project.jar`运行我们的web项目。
把SpringBoot打包成jar运行的时候，已经给我们默认内置了tomcat，当然也可以改为其他服务器容器，比如jetty等。
## 制作容器镜像

### 全程使用Dockerfile

使用到Dockerfile的时候需要用到一些原语操作，接下来说一个等下会用到的的原语。

- `FROM` 使用一个基础镜像。
 - `FROM ubuntu:latest` 使用最新版的Ubuntu镜像为基础镜像
 - `FROM my-create-images-name:tag-name` 使用一个本地已经搭建好相关环境的镜像。
- `WORKDIR` 切换工作目录
 - `WORKDIR /var/www` 切换工作目录到 `/var/www` 会创建该目录，之后的所有操作命令操作都会基于该路径
- `ADD` 向容器中复制文件
 - `ADD ./app.jar /var/www` 把宿主机当前目录的 app.jar 复制到容器的 /var/www目录中
- `RUN` 在镜像创建的过程中运行一个特定的命令
 - `RUN apt update -y` 在制作镜像的过程中，给镜像更新系统
 - `RUN apt install openjdk-8-jre -y` 在制作镜像的过程中，给镜像安装 java 环境
- `EXPOSE` 向外界暴露一个端口
 - `EXPOSE 80` 向外界暴露一个80端口，允许外界访问容器的80端口
- `ENV` 设置环境变量
 - `ENV AppName my-web-server` 设置一个系统环境变量
- `CMD` 在启动容器的时候执行一条命令，通常用于启动我们的服务
 - `CMD java -jar app.jar --server.port=80` 在容器启动的时候执行我们的SpringBoot jar包
 - `CMD ["python", "app.py"]` 网上有些代码的CMD呈这种格式，但是该命令格式无法执行java命令。
 判断原因：这表明CMD执行了两次命令，首先执行`python`然后再执行`app.py`，之所以在Python能够成功，是因为执行完`python`命令后会进行
 交互状态，此时执行`app.py`时会使Python执行该文件，因而不会出现错误。但是java命令，执行完后会直接退出，而不是进入java终端交互状态，
 也就是说在执行`CMD ["java","-jar app.jar"]`命令时，先执行完java后会退回终端，再执行`-jar app.jar`这条命令就会因为找不到命令而报错。
 

通过以上的原语，我们可以完全从一个全新的Ubuntu容器镜像制作出一个可以直接运行我们web程序的容器镜像，并可以把这个镜像放到不同的机器直接运行，而不需要重新配置任何环境。
#### 操作步骤
1. 新建一个工作文件夹：`mkdir make-docker`
2. 进入到刚刚的文件夹：`cd make-docker`
3. 新建一个Dockerfile文件：`touch Dockerfile`，文件的内容我们可以稍后再添加
4. 把我们的app.jar包放到当前的`make-docker`目录中。此时make-docker目录下共有以下文件：`app.jar` `Dockerfile`
5. 编辑Dockerfile文件（稍后附上参考内容）
6. 执行docker镜像构建命令：`docker build -t Images-Name .` 注：Images-Name的格式可以有：`myImagesName`直接一个镜像名，它默认tag为`latest`表示最新版本，也可以指定镜像版本`myImagesName:v18.04`

Dockerfile参考内容：
```
	# 使用官方的Ubuntu镜像
	FROM ubuntu:18.04
	
	# 切换工作目录到 /var/www
	WORKDIR /var/www
	
	# 把当前目录的 app.jar 复制到 /var/www 目录
	ADD ./app.jar /var/www
	
	# 更新软件
	RUN apt update -y
	
	# 更新系统
	RUN apt upgrade -y
	
	# 给 /var/www 赋予最高权限
	RUN chmod -R 777 /var/www
	
	# 安装 jdk 运行环境
	RUN apt install -y openjdk-8-jre
	
	# 暴露 80 端口
	EXPOSE 80
	
	# 设置一个系统环境变量名 NAME= my-web-server
	ENV NAME my-web-server
	
	# 容器启动时运行 jar 程序
	CMD java -jar app.jar --server.port=80

```
等待系统更新和环境安装。
完成后，我们可以通过`docker images`查看到我们刚刚制作的镜像。
接下来可以通过`docker run -p 8080:80 -v /var/www/public:/var/www/static -it myImagesName`来运行我们的容器，此时可以看到控制台打印的运行记录信息。

- `-p 8080:80` 把宿主机的8080端口映射到容器的80端口，当访问宿主机的8080端口时就相当于访问容器的80端口。
- `-v /var/www/public:/var/www/static` 把宿主机的目录 `/var/www/public`挂载到容器的`/var/www/static`目录中。容器所拥有的挂载目录权限取决于宿主机对该目录的权限。

### 先创建一个配置好环境的容器，再制作容器镜像
#### 制作一个拥有运行环境的容器

- 创建并启动一个容器：`docker run -it ubuntu:18.04 /bin/bash`此时会同时进入到容器的bash中
- 在容器中手动安装所需环境
 - `apt update -y`
 - `apt upgrade -y`
 - `apt install openjdk-8-jre -y`
- 退出到宿主机的bash中
- 把容器提交创建本地镜像：`docker commit [ContainerID] [ImagesName]`，此时我们可以运行`docker images`看到刚刚创建好的镜像`[ImagesName]`
- 然后再使用Dockerfile来构建镜像（同上方法）
