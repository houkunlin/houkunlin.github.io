---
title: 部署 SpringBoot 项目时一个 Linux Service 模板
date: 2020-10-26 12:09:04
updated: 2020-10-27 10:22:10
tags:

---

## `app.jar` 的启动

在服务器上启动项目时最简单的启动项目方法是直接执行 `java -jar app.jar` 命令，或者使用 `nohup` 进入到后台运行 `nohup java -jar app.jar &` 。

但是当我们重新部署、重启项目时会比较麻烦，我们需要通过 `ps -ef|grep app.jar` 来找到我们项目执行时的进程ID `PID` ，然后再执行 `kill -9 $PID` 来杀掉当前正在运行的项目，之后再重新运行项目。

虽然只有几个简单的步骤，但还是稍微有点麻烦。整个过程：**运行->上传新程序->查找当前程序PID->杀掉当前程序->重新运行**，那我们能不能稍微简化一下上述过程，把中间查找PID和杀掉进程这两个不去掉，也把命令简化一下，变成：**运行->上传新程序->重新运行** 。



### 手动编写一个启动脚本：`run.sh`

此时我们最简单的方法就是写一个 `SHELL` 脚本 `run.sh`：

```bash
#!/bin/bash

echo '---------------kill_app process----------------'

KILL_PROCESS_NAME='/application/app.jar'

PROCESS_ID=`ps -ef | grep $KILL_PROCESS_NAME | grep -v 'grep' | awk '{print $2}'`

echo 'ProcessId: ' $PROCESS_ID

for id in $PROCESS_ID

do
	echo 'KILL_ID: ' $id
	kill -s 9 $id	
done

echo '---------------killed_app.jar----------------'

echo '---------------start_app.jar----------------'

nohup  java -Xms512m -Xmx512m -jar  $KILL_PROCESS_NAME --spring.profiles.active=file  >/dev/null 2>&1 &

echo '---------------started_app.jar----------------'

```



以上脚本能够有效的解决我们的问题，但是新问题又来了，如果想查看程序是否运行呢？我们想的肯定是直接执行命令 `ps -ef|grep app.jar` 查找这个程序是否正在运行，或者把查询运行状态的命令写进我们前面的脚本里面，然后执行 `./run.sh status` 查询程序执行状态。

假如我们把查询状态、启动、停止这几个操作都写入到一个脚本中，并提供参数来调用，就需要修改脚本代码，此时我们的脚本是这样的：

```bash
#!/bin/bash

KILL_PROCESS_NAME='/application/app.jar'

getProcessId() {
  PID=$(ps -ef | grep $KILL_PROCESS_NAME | grep -v 'grep' | awk '{print $2}')
  return $PID
}

start() {
  nohup java -Xms512m -Xmx512m -jar $KILL_PROCESS_NAME --spring.profiles.active=file >/dev/null 2>&1 &
  echo "运行成功"
}

stop() {
  getProcessId
  PROCESS_ID=$?
  if [ "$PROCESS_ID" != "0" ]; then
    echo "进程ID：$PROCESS_ID"
    kill -s 9 $PROCESS_ID
  else
    echo "没有运行"
  fi
}

case $1 in
start)
  start
  ;;
status)
  getProcessId
  PROCESS_ID=$?
  if [ "$PROCESS_ID" != "0" ]; then
    echo "正在运行..."
  else
    echo "没有运行"
  fi
  ;;
stop)
  stop
  ;;
restart)
  stop
  start
  ;;
*)
  echo "参数错误"
  ;;
esac

```

此时我们有了四个可以执行的命令：

- `./run.sh start` 启动程序
- `./run.sh status` 查询运行状态
- `./run.sh stop` 停止运行
- `./run.sh restart` 重启程序

此时我们就有了4个可以正常使用的命令，但是现在还不能直接设置程序在系统启动的时候随着系统启动，因为还需要做一些配置，最简单的就是修改 `/etc/profile` 文件加上启动命令，比较麻烦一点就是写一个启动脚本放到系统的 `/etc/init.d/` 初始化路径中，然后把脚本加入到系统启动。

但同时我们上面的几个命令看起来是不是非常像Linux系统的系统服务呢？Linux系统服务也是有4个命令：

- `systemctl start app.service` 启动服务
- `systemctl status app.service` 查询服务运行状态
- `systemctl stop app.service` 停止服务
- `systemctl restart app.service` 重启服务

那我们能不能把它改造成系统服务呢？我们该如何为自己的 `SpringBoot` 应用编写一个Linux系统服务呢？



## `app.service` 示例

以下给出了一个示例 `app.service` 文件：

```ini
# http://www.jinbuguo.com/systemd/systemd.unit.html
[Unit]
# 单元的解释说明 http://www.jinbuguo.com/systemd/systemd.unit.html#Description=
Description=Spring Boot Application
# 启动顺序设置
After=network.target
# 配合 After 设置强依赖服务，依赖单元必须已经全部处于启动成功的状态时才能启动当前单元 http://www.jinbuguo.com/systemd/systemd.unit.html#Requisite=
#Requisite=

[Service]
Type=simple
User=nobody
# 重启服务配置， on-failure 表示仅在服务进程异常退出时重启 http://www.jinbuguo.com/systemd/systemd.service.html#Restart=
Restart=on-failure
# 在重启服务前暂停的时间 http://www.jinbuguo.com/systemd/systemd.service.html#RestartSec=
RestartSec=30s
# 设置日志与标准输入输出 http://www.jinbuguo.com/systemd/systemd.exec.html#StandardOutput=
# 关闭了 Service 的输出，依赖 Spring Boot 应用程序的日志输出
# 请注意，一定要把 WorkingDirectory 目录设置 nobody:nogroup 的权限，否则此目录无写入权限
StandardOutput=null
# 设置工作路径 http://www.jinbuguo.com/systemd/systemd.exec.html#WorkingDirectory=
WorkingDirectory=/application/
ExecStart=/usr/bin/java -Xms512m -Xmx512m -jar /application/app.jar --spring.profiles.active=dev
# 其实这里还可以向下面这样写，java执行的jar不写全路径，直接写文件名，因为设置了 WorkingDirectory 会在该路径下找相应的文件
#ExecStart=/usr/bin/java -Xms512m -Xmx512m -jar app.jar --spring.profiles.active=dev

[Install]
WantedBy=multi-user.target

```



### 如何使 `app.service` 生效

该 `app.service` 文件存放在Linux系统的 `/lib/systemd/system/` 路径下。

我们可以在我们的项目下存放该 `app.service` 文件，类似于以下目录结构：

```
/
/src
/src/main
/src/test
/pom.xml
/app.service
```



然后在Linux系统中的项目路径下，可以通过 `ln` 把 `app.service` 文件链接到 `/lib/systemd/system/` 路径下：

```bash
cd project-dir
ln -s app.service /lib/systemd/system/
# 或者 ln -s app.service /lib/systemd/system/app.service
# 重新加载 service
systemctl daemon-reload
# 然后就可以使用Linux的系统服务管理软件来启动运行我们的 SpringBoot 应用了
```



接下来我们就可以使用 `systemctl` 来启动我们的 `SpringBoot` 应用了

- `systemctl start app.service` 启动应用
- `systemctl status app.service` 查看应用启动状态
- `systemctl restart app.service` 重启应用
- `systemctl stop app.service` 停止应用

或者使用 `service` 命令来启动我们的 `SpringBoot` 应用

- `service app.service start` 启动应用
- `service app.service status` 查看应用启动状态
- `service app.service restart` 重启应用
- `service app.service stop` 停止应用



## `app.service` 解读

以下使用 `单元` 来表示一个系统服务 `service` 

- `After=network.target` 在网卡启动之后启动当前单元
- `Requisite=` 必须在一个单元启动成功后才启动当前单元
- `Restart=on-failure` 在异常退出的时候重新启动单元
- `RestartSec=30s` 重启单元前暂停的时间
- `StandardOutput=null` 关闭单元的标准输出，实际上也就是抛弃 `ExecStart` 命令中在控制台产生的输出、日志记录
- `WorkingDirectory=/application/` 设置这个单元的工作路径，默认工作在 `/` 路径下，设置为存放 `app.jar` 的路径，这样可以使 `SpringBoot` 应用识别到路径下的配置文件并应用到运行环境中
- `ExecStart=/usr/bin/java -jar /application/app.jar` 启动 `app.jar` 项目，直接使用 `java` 命令来启动，也不用把其放到后台进程中运行

其实还隐含了以下 `[Service]` 配置，但是并不需要我们进行特殊的配置也能产生相应的作用：

- `ExecRestart=` 重启单元时执行的重启应用命令，可以不配置，系统会自动先 `stop` 再 `start`
- `ExecStop=` 停止单元时执行的停止应用命令，可以不配置，系统会自动 `kill` 掉当前单元中 `ExecStart` 运行的进程


