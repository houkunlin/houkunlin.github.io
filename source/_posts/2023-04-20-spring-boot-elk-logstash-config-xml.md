---
title: SpringBoot ELK 日志部署和配置示例
date: 2023-04-20 15:31:37
updated: 2023-04-20 15:31:37
tags:
  - ELK
  - logstash
---

## ElasticSearch 部署配置（8.7.0版本）

从 [ElasticSearch官网](https://www.elastic.co/cn/downloads/elasticsearch) 下载安装包

它不允许在 root 下运行，如果当前是 root 用户，则需要创建一个单独的用户：

```bash
groupadd elk
useradd elk -g elk -p "elk@123456"
chown -R elk:elk $ES_HOME
```

修改系统内核最大虚拟内存限制，在 `/etc/sysctl.conf` 增加

```hocon
vm.max_map_count = 262144
```

修改 JVM 内存限制，创建 `config/jvm.options.d/memory.options` 文件并编辑：

```text
# 8.7 版本默认是16G，机器内存不够则需要手动修改
-Xms5g
-Xmx5g
```

修改 `config/elasticsearch.yml` 配置文件

```yaml
# 修改数据保存路径
#path.data: /path/to/data
# 修改日志保存路径
#path.logs: /path/to/logs
# 默认情况下，Elasticsearch 仅绑定到环回地址
# 为了与其他服务器上的节点进行通信并形成集群，您的节点将需要绑定到非环回地址
network.host: 127.0.0.1
# 修改端口，已默认 9200
http.port: 9200
# 修改 HTTP API 绑定地址，已默认 0.0.0.0
http.host: 0.0.0.0
```

执行 `./bin/elasticsearch` 启动服务，若无报错则启动成功，同时在控制台会打印一个 `elastic` 用户的初始化密码。
可以使用 `./bin/elasticsearch -d` 在后台运行服务。

访问 `https://[host]:9200` 查看是否启动成功，注意，是 https 协议

启动成功后可以使用 `./bin/elasticsearch-setup-passwords interactive` 来重置所有用户密码（此方式需要手动设置密码）。

也可以使用 `./bin/elasticsearch-setup-passwords auto` 来重置所有用户密码（此方式自动设置所有密码）。

也可以使用 `./bin/elasticsearch-reset-password -u elastic` 来重置 `elastic` 用户密码（此方式无法手动设置密码，由系统自动生成密码）

## Kibana 部署配置（8.7.0版本）

从 [Kibana官网](https://www.elastic.co/cn/downloads/kibana) 下载安装包

它默认不允许在 root 下运行，如果当前是 root 用户，则需要创建一个单独的用户：

```bash
groupadd elk
useradd elk -g elk -p "elk@123456"
chown -R elk:elk $KIBANA_HOME
```

其实在启动的时候也可以增加参数来允许在 root 下运行 `./bin/kibana --allow-root` 但是既然默认不允许，那就不推荐。

```yaml
# 修改默认端口
#server.port: 5601
# 修改绑定地址
#server.host: "0.0.0.0"
# 配置访问地址，虽然绑定了 0.0.0.0 ，任意网口都可以访问，但官方还是建议配置一个请求资源的前缀地址
# 例如在使用反向代理使用域名访问时，这里就配置域名的地址
server.publicBaseUrl: "http://172.22.1.10:5601/"
# 修改连接 ElasticSearch 服务器地址
#elasticsearch.hosts: ["http://localhost:9200"]
# 修改默认中文页面
i18n.locale: "zh-CN"
```

执行 `./bin/kibana` 启动服务，然后可以访问 `http://[host]:5601` ，注意，这里是 http 协议。

由于前面配置文件中没有配置具体的 ElasticSearch 服务地址和用户密码信息，在页面中提示配置 ElasticSearch 服务器信息，此时需要填写服务器地址和
kibana_system 的密码。
当前也可以在配置文件中写死这些配置。

如果不知道 `kibaba_system` 密码，可在 ElasticSearch 下执行 `./bin/elasticsearch-reset-password -u kibana_system` 命令重置密码

```yaml
# 配置访问地址，虽然绑定了 0.0.0.0 ，任意网口都可以访问，但官方还是建议配置一个请求资源的前缀地址
# 例如在使用反向代理使用域名访问时，这里就配置域名的地址
server.publicBaseUrl: "http://172.22.1.10:5601/"
# 注意，这里是 https 协议
elasticsearch.hosts: [ "https://localhost:9200" ]
elasticsearch.username: "kibana_system"
elasticsearch.password: "4HgBCC7482ckdFcuVJvD"
# 由于 ElasticSearch 使用了 https 访问，因此需要把 https 的验证关掉
elasticsearch.ssl.verificationMode: none
i18n.locale: "zh-CN"
```

重启 Kibana 服务，访问 `http://[host]:5601` ，用 `elastic` 用户登录系统，如果忘记了密码，请使用命令重置密码，登录 Kibana
后可重新修改设置密码。

## Logstash 部署配置（8.7.0版本）

从 [Logstash官网](https://www.elastic.co/cn/downloads/logstash) 下载安装包

由于 ElasticSearch 使用了默认的自签名 https ，因此需要复制证书到 Logstash `copy -r $ES_HOME/config/certs/ $LOGSTASH_HOME/config/` ，当然也可以忽略 https 验证。

登录 Kibana 创建一个 logstash_server 专用账号，系统已有的 logstash_system 账号无法创建索引，因此必须定制一个账号，并分配相关的索引权限。

在 `$LOGSTASH_HOME` 创建一个 `springboot-logstash.conf` 配置文件：

```text
input {
  tcp {
    mode => "server"
    # 监听 4567 端口
    port => 4567
    # JSON格式处理
    codec => json_lines
  }
}
output {
  elasticsearch {
    # 注意，这里是 https 协议
    hosts => ["https://localhost:9200"]
    user => "logstash_server"
    # 其实密码可以使用环境变量的，例如：$ES_PASSWORD
    password => "logstash_server@123456"
    # 这里指定 https 协议的证书
    #cacert => "./config/certs/http_ca.crt"
    # 或者忽略 https 验证
    ssl_certificate_verification => false
    # 配置 ES 的索引名称，这里引用了 SpringBoot 传过来的应用名称作为一个索引参数，请注意，索引名称必须小写
    index => "spring-boot-%{APP_NAME}-%{+YYYY-MM-dd}"
  }
  #stdout {
  #  # 在控制台打印日志
  #  codec => rubydebug
  #}
}
```

启动 `./bin/logstash -f springboot-logstash.conf` 服务


## SpringBoot 配置

日志级别从低到高分为TRACE < DEBUG < INFO < WARN < ERROR < FATAL，如果设置为WARN，则低于WARN的信息都不会输出

scan:当此属性设置为true时，配置文档如果发生改变，将会被重新加载，默认值为true

scanPeriod:设置监测配置文档是否有修改的时间间隔，如果没有给出时间单位，默认单位 当scan为true时，此属性生效。默认的时间间隔为1分钟。

debug:当此属性设置为true时，将打印出logback内部日志信息，实时查看logback运行状态。默认值为false。

在项目中引入依赖 `net.logstash.logback:logstash-logback-encoder:7.2`，在 `src/main/resources/`
下配置 `logback-spring.xml` 文件内容：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="false" scanPeriod="10 seconds" debug="false">
    <!-- START: org/springframework/boot/logging/logback/base.xml -->
    <include resource="org/springframework/boot/logging/logback/defaults.xml" />
    <property name="LOG_FILE" value="${LOG_FILE:-${LOG_PATH:-${LOG_TEMP:-${java.io.tmpdir:-/tmp}}}/spring.log}"/>
    <include resource="org/springframework/boot/logging/logback/console-appender.xml" />
    <include resource="org/springframework/boot/logging/logback/file-appender.xml" />

    <root level="INFO">
        <appender-ref ref="CONSOLE" />
        <appender-ref ref="FILE" />
    </root>
    <!-- END: org/springframework/boot/logging/logback/base.xml -->

    <logger name="web" level="ERROR"/>
    <logger name="org.springframework" level="ERROR"/>

    <springProperty scope="context" name="APP_NAME" source="spring.application.name" defaultValue="spring"/>
    <springProperty scope="local" name="LOGSTASH_HOST" source="logstash.host" defaultValue="127.0.0.1"/>
    <springProperty scope="local" name="LOGSTASH_PORT" source="logstash.port" defaultValue="4567"/>

    <!-- 线上环境的配置 -->
    <springProfile name="prod">
        <!-- ELK 的 LOGSTASH 日志配置 -->
        <appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
            <!-- LOGSTASH 服务地址和端口，端口在服务端自定义 -->
            <destination>${LOGSTASH_HOST}:${LOGSTASH_PORT}</destination>
            <encoder charset="UTF-8" class="net.logstash.logback.encoder.LogstashEncoder">
                <!--<customFields>{"appname":"自定义字段"}</customFields>-->
            </encoder>
        </appender>
        <!-- 日志输出级别 -->
        <root level="INFO">
            <appender-ref ref="LOGSTASH"/>
        </root>
    </springProfile>
</configuration>
```

以及修改 `application.yml` 文件：
```yaml
logging:
  config: classpath:logback-spring.xml
```
