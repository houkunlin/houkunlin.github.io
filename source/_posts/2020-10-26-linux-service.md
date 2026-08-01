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

- `systemctl start service-name.service` 启动服务
- `systemctl status service-name.service` 查询服务运行状态
- `systemctl stop service-name.service` 停止服务
- `systemctl restart service-name.service` 重启服务

那我们能不能把它改造成系统服务呢？我们该如何为自己的 `SpringBoot` 应用编写一个Linux系统服务呢？



## `service-name.service` 示例

以下给出了一个示例 `service-name.service` 文件：

```ini
# http://www.jinbuguo.com/systemd/systemd.unit.html
[Unit]
# 单元的解释说明 http://www.jinbuguo.com/systemd/systemd.unit.html#Description=
# Description=取值：自由文本，无固定枚举；服务的描述信息，会显示在 systemctl status 中
Description=Spring Boot Application
# Wants= 取值：空格分隔的单元名列表；弱依赖，仅负责拉入被依赖单元启动，
# 被依赖单元启动失败/不存在不会阻塞本服务启动（本服务无需等待其完成，等待需配 After=）
Wants=network-online.target
# After= 取值：空格分隔的单元名列表；纯顺序约束（不拉入依赖），
# 本服务会等列出的单元启动成功后才启动；
# 这里 After=network-online.target 确保网络完全就绪（配合 NetworkManager-wait-online）后再启动服务，避免启动时网络不可用
After=network-online.target
# 配合 After 设置强依赖服务，依赖单元必须已经全部处于启动成功的状态时才能启动当前单元 http://www.jinbuguo.com/systemd/systemd.unit.html#Requisite=
#Requisite=
# 启动失败重试的限制：在 120 秒时间窗内最多允许 5 次启动失败，超过则放弃并进入 failed 状态
# StartLimitIntervalSec= 取值：时间跨度（裸数字默认秒，支持 ms/s/min/h/d/w 或 infinity）
# StartLimitBurst= 取值：正整数，时间窗内允许的最大启动失败次数
StartLimitIntervalSec=120s
StartLimitBurst=5

[Service]
# 进程定义
# Type= 取值：simple（默认，ExecStart 启动的进程即主进程（非 fork 守护进程），Spring Boot 适用）、exec（与 simple 类似但会先做进程完整初始化）、
#   forking（传统守护进程：先 fork，父进程退出后子进程成为主进程）、oneshot（一次性任务，执行完即结束）、
#   dbus（需在 D-Bus 上注册）、notify（需通过 sd_notify 通知就绪）、notify-reload、idle（空闲时再启动）
# 本项目用 simple：java 进程就是主服务进程，systemd 直接跟踪其状态
Type=simple
# User= 取值：用户名或数字 UID（如 nobody、root、1000）
# 运行用户：使用 nobody 低权限用户，避免以 root 运行应用，降低安全风险
User=nobody
# Group= 取值：组名或数字 GID；未显式设置时默认使用 User= 的主组
# 运行用户组：使用 nogroup 组（与 User=nobody 搭配，降低权限）
Group=nogroup

# 运行环境
# 设置工作路径 http://www.jinbuguo.com/systemd/systemd.exec.html#WorkingDirectory=
# WorkingDirectory= 取值：绝对路径、特殊值 ~（即 User= 的家目录）、前缀 -（目录不存在时不报错）
# 服务的工作目录，相对路径的日志/文件都会基于此目录；需确保 nobody 用户可写，
# 否则此目录无写入权限（请把 /app/service-name 目录设置为 nobody:nogroup 权限）
WorkingDirectory=/app/service-name
# 加载环境变量文件
# EnvironmentFile= 取值：绝对路径（可含通配符），可多次使用（后读覆盖先读），前缀 - 表示文件不存在时不报错
# 可用于注入 DB/MQ 等运行时密钥与配置，注意该文件权限应设为 600 防止泄露
EnvironmentFile=-/app/service-name/service-name.env
# Environment= 取值：空格分隔的 VAR=VALUE 列表，可多次使用；适合设置"默认值"
# 默认 profile：当 service-name.env 未设置 SPRING_PROFILES_ACTIVE 时使用此默认值 dev
# 注意：EnvironmentFile 中配置的同名变量会覆盖这里的默认值，即 service-name.env 优先
Environment=SPRING_PROFILES_ACTIVE=dev
# ExecStart= 取值：空格分隔的命令及其参数（非 shell 命令，不支持管道/重定向/变量赋值）
#   前缀 - ：命令失败不记入启动失败统计；前缀 + ：以 root 权限执行；前缀 @ ：按绝对路径执行且不做二次拆分
#   支持 $VAR / ${VAR} 环境变量展开（Environment= 与 EnvironmentFile= 中的变量均可引用），$$ 表示字面 $
# 启动命令：java 启动 Spring Boot fat jar，并指定激活环境
# ${SPRING_PROFILES_ACTIVE} 由 systemd 从 EnvironmentFile(service-name.env) 展开，
# ps -ef 查看进程时可直接看到实际激活的环境（如 --spring.profiles.active=dev）
# -Xms512m -Xmx512m：JVM 堆初始/最大均为 512MB
# -XX:+HeapDumpOnOutOfMemoryError：OOM 时自动生成堆转储文件，便于排查内存问题
# -XX:HeapDumpPath=/app/service-name/logs：堆转储文件输出目录（需保证 nobody 用户可写）
ExecStart=/usr/bin/java -Xms512m -Xmx512m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/app/service-name/logs -jar /app/service-name/app.jar --spring.profiles.active=${SPRING_PROFILES_ACTIVE}

# 生命周期与重启策略
# 重启服务配置 http://www.jinbuguo.com/systemd/systemd.service.html#Restart=
# Restart= 取值：no（默认，不自动重启）、on-success（仅正常退出时重启）、on-failure（仅异常退出时重启）、
#   on-abnormal（信号终止/超时/看门狗等非正常退出时重启）、on-abort、on-watchdog、always（无论何因都重启）
# 本项目用 on-failure：进程异常退出（退出码非 0 且不在 SuccessExitStatus 中）时自动重启；
# 注意：on-failure 不会因正常退出（exit code 0 或 143）而重启
Restart=on-failure
# 在重启服务前暂停的时间 http://www.jinbuguo.com/systemd/systemd.service.html#RestartSec=
# RestartSec= 取值：时间跨度（裸数字默认秒，支持 ms/s/min/h/d/w），可设 infinity 表示立即重启
RestartSec=5s
# SuccessExitStatus= 取值：退出码（0-255）、信号名（如 SIGTERM）、范围（如 0-2），
#   可空格分隔多个，也可用 码:信号 混写（如 143:SIGTERM）
# 将退出码 143（128 + SIGTERM=15，即被 SIGTERM 终止）视为"成功"退出，
# 避免 systemd 因优雅停机导致的 143 而误判为失败
SuccessExitStatus=143
# KillSignal= 取值：信号名（如 SIGTERM、SIGINT）或数字（如 15）；指定停止时发送给主进程的信号
# 停止服务时发送的信号：SIGTERM 让 Spring Boot 优雅停机（触发 shutdown hook 完成资源释放）
KillSignal=SIGTERM
# TimeoutStopSec= 取值：时间跨度（裸数字默认秒），或 infinity（无限等待优雅退出）
# 优雅停机最长等待时间：超过 120 秒仍未退出则强制 SIGKILL
TimeoutStopSec=120s

# 资源限制
# LimitNOFILE= 取值：非负整数、infinity（无限制）、或 soft:hard 形式（如 1024:65535）
# 设置进程可打开的最大文件描述符数量（ulimit -n），防止高并发下句柄耗尽
LimitNOFILE=65535

# 安全加固
# NoNewPrivileges= 取值：布尔值（yes/no/true/false/1/0），默认 no
# 禁止服务进程获取新的权限（防提权，配合 User=nobody 加固）
NoNewPrivileges=true
# PrivateTmp= 取值：布尔值（yes/no/true/false/1/0），默认 no
# 为服务提供独立的 /tmp 和 /var/tmp 私有挂载命名空间，避免与其他进程共享临时目录
PrivateTmp=true

# 日志与标准输入输出
# 设置日志与标准输入输出 http://www.jinbuguo.com/systemd/systemd.exec.html#StandardOutput=
# 可用值说明（StandardOutput= 默认值为 journal；StandardError= 默认值为 inherit 即跟随 StandardOutput=）：
#   null                : 丢弃输出（等价重定向到 /dev/null），配合应用自身日志（如 logback）使用
#   inherit             : 继承前一个设置（StandardError 默认值）
#   tty                 : 输出到当前控制台终端（一般在调试时使用）
#   journal             : 写入 journald 日志系统，可用 journalctl -u service-name.service 查看（默认值）
#   kmsg                : 写入内核环形缓冲区（dmesg 可看）
#   syslog              : 写入传统 syslog（自 v245 起已弃用，建议用 journal）
#   file:/path          : 写入指定文件（不存在则创建；已存在不截断也不追加，重启后从头覆盖，慎用）
#   append:/path        : 追加写入指定文件（O_APPEND），重启不丢旧日志，推荐用于落盘
#   truncate:/path      : 清空后再写入指定文件（每次启动只见新日志）
#   socket              : 输出到由 socket 单元创建的套接字（配合 Accept= 使用）
#   fd:名称             : 输出到进程继承的指定文件描述符
#   以上值可加 +console 后缀（如 append:/path+console）实现"文件 + 终端"同时输出
# 注意：设为 null 时 journald 不再捕获输出，若应用自身又不写日志文件，则日志会完全丢失
StandardOutput=append:/app/service-name/logs/stdout.log
StandardError=append:/app/service-name/logs/stderr.log

[Install]
# WantedBy= 取值：空格分隔的 target/单元名列表；列出哪些单元会以"弱依赖"方式在开机时把本服务拉入启动
# 当系统进入多用户模式（multi-user.target）时自动启动本服务
WantedBy=multi-user.target

```

`service-name.env` 文件参考

```dotenv
# service-name 服务运行时环境变量（被 service-name.service 的 EnvironmentFile 加载）
# 注意：此文件包含运行时密钥/敏感配置，权限必须设为 600（chmod 600 service-name.env），防止泄露

# Spring Boot 激活的 profile，决定加载 application-{profile}.yml
# 会被 systemd 展开进 ExecStart 的 --spring.profiles.active=${SPRING_PROFILES_ACTIVE}
# ps -ef 查看进程时可见实际生效的环境（如 dev / prod / uat）
# 若此处不配置（注释掉本行），则回退使用 service-name.service 中 Environment= 设置的默认值 dev
SPRING_PROFILES_ACTIVE=dev

```


### 如何使 `service-name.service` 生效

该 `service-name.service` 文件存放在Linux系统的 `/lib/systemd/system/` 路径下。

我们可以在我们的项目下存放该 `service-name.service` 文件，类似于以下目录结构：

```
/
/src
/src/main
/src/test
/pom.xml
/service-name.service
```



然后在Linux系统中的项目路径下，可以通过 `ln` 把 `service-name.service` 文件链接到 `/lib/systemd/system/` 路径下：

```bash
cd project-dir
ln -s service-name.service /lib/systemd/system/
# 或者 ln -s service-name.service /lib/systemd/system/service-name.service
# 重新加载 service
systemctl daemon-reload
# 然后就可以使用Linux的系统服务管理软件来启动运行我们的 SpringBoot 应用了
```



接下来我们就可以使用 `systemctl` 来启动我们的 `SpringBoot` 应用了

- `systemctl start service-name.service` 启动应用
- `systemctl status service-name.service` 查看应用启动状态
- `systemctl restart service-name.service` 重启应用
- `systemctl stop service-name.service` 停止应用

或者使用 `service` 命令来启动我们的 `SpringBoot` 应用

- `service service-name.service start` 启动应用
- `service service-name.service status` 查看应用启动状态
- `service service-name.service restart` 重启应用
- `service service-name.service stop` 停止应用



## `service-name.service` 解读

以下使用 `单元` 来表示一个系统服务 `service` 

- `After=network.target` 在网卡启动之后启动当前单元
- `Requisite=` 必须在一个单元启动成功后才启动当前单元
- `Restart=on-failure` 在异常退出的时候重新启动单元
- `RestartSec=30s` 重启单元前暂停的时间
- `StandardOutput=null` 关闭单元的标准输出，实际上也就是抛弃 `ExecStart` 命令中在控制台产生的输出、日志记录
- `WorkingDirectory=/app/service-name` 设置这个单元的工作路径，默认工作在 `/` 路径下，设置为存放 `app.jar` 的路径，这样可以使 `SpringBoot` 应用识别到路径下的配置文件并应用到运行环境中
- `ExecStart=/usr/bin/java -jar /app/service-name/app.jar` 启动 `app.jar` 项目，直接使用 `java` 命令来启动，也不用把其放到后台进程中运行

其实还隐含了以下 `[Service]` 配置，但是并不需要我们进行特殊的配置也能产生相应的作用：

- `ExecRestart=` 重启单元时执行的重启应用命令，可以不配置，系统会自动先 `stop` 再 `start`
- `ExecStop=` 停止单元时执行的停止应用命令，可以不配置，系统会自动 `kill` 掉当前单元中 `ExecStart` 运行的进程


