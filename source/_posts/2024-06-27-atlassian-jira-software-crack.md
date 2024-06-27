---
title: Atlassian Jira Software 激活方法
date: 2024-06-27 11:51:36
updated: 2024-06-27 11:51:36
tags:
  - Atlassian
  - Jira
---

> 据说此插件可激活 Atlassian 系列软件。

假设：

- `JIRA_HOME` 为 `/opt/atlassian/jira-home`
- `JIRA_INSTALL` 为 `/opt/atlassian/jira-software`
- `JIRA_ATLASSIAN` 为 `/opt/atlassian/atlassian-agent.jar`

## 下载 Jira Software 程序

进入到 [Download Jira Software Server](https://www.atlassian.com/software/jira/download.) 下载 Jira Software。
并把下载的安装包解压到 `JIRA_INSTALL` 目录下。

我是下载了 TAR.GZ 压缩包文件，解压到 `JIRA_INSTALL` 目录。

## 下载 Atlassian Agent 插件

点击 [此下载连接](./atlassian-agent.jar) （来自：https://github.com/haxqer/jira/blob/build-your-own/docker/jira/atlassian-agent.jar ）下载 Atlassian Agent 插件，并把插件放到 `JIRA_ATLASSIAN` 。

设置 JAVA OPTIONS 环境变量 
```bash
export CATALINA_OPTS="-javaagent:/opt/atlassian/atlassian-agent.jar ${CATALINA_OPTS}"
```

或把配置写到文件
```bash
echo 'export CATALINA_OPTS="-javaagent:/opt/atlassian/atlassian-agent.jar ${CATALINA_OPTS}"' >> /${JIRA_INSTALL}/bin/setenv.sh
```

## 启动（安装） Jira Software 程序

执行命令 `/${JIRA_INSTALL}/bin/start-jira.sh` 启动 Jira Software 程序。然后根据安装提示进行配置，直到进入到输入许可证界面。

安装过程可参考官方文档 [Installing Jira applications on Linux from Archive File](https://confluence.atlassian.com/adminjiraserver/installing-jira-applications-on-linux-from-archive-file-938846844.html)。

## 生成许可证内容

在上一步的许可证界面，可得到 `服务器ID` 参数值，在服务器执行以下命令生成许可证内容，并把许可证内容填入到页面中。
```bash
java -jar ${JIRA_ATLASSIAN} -p jira -m test@test.com -n BAT -o https://192.168.0.10:8080 -s AAAA-BBBB-CCCC-DDDD
```


参数说明：
```
# 通过破解包生成激活码
# -p 产品名称 jira 或者插件ID
# -m 邮箱（test@test.com）
# -n 用户名，这个随意
# -o 部署的入口地址
# -s 服务器ID（AAAA-BBBB-CCCC-DDDD）
```


## 参考链接

- Atlassian全家桶及其插件激活方法 [https://www.iots.vip/post/atlassian-series-crack.html](https://www.iots.vip/post/atlassian-series-crack.html)
- Github: haxqer/jira [https://github.com/haxqer/jira](https://github.com/haxqer/jira)
