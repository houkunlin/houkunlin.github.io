---
title: 达梦数据库培训内容操作记录
date: 2019-09-16 16:14:20
updated: 2019-09-16 16:14:20
tags:
- 达梦
- 数据库
---

## 1. 数据库安装准备 <span id="index01"></span>

### 1.1 系统参数检验：<span id="index01.01"></span>
- 检验最大可打开文件数目 
    - `ulimit -n` 检查可打开文件最大数目
    - `ulimit -n ${number}` 设置可打开文件最大数目
    - `vi /etc/security/limits.conf` 在里面添加
        ```
        # 给所有用户添加最大文件打开数目，该方式无法对root用户生效，root用户需要单独配置
        *   soft    nofile  65535
        *   hard    nofile  65536
        
        # 给root用户添加最大文件打开数目，root用户需要单独的配置
        root    soft    nofile  65535
        root    hard    nofile  65536
        ```
        这个操作对root用户和其他所有用户设置最大文件打开数目，设置完后注销当前用户即可生效。

### 1.2 添加数据库用户：`dmdba`<span id="index01.02"></span>
- 在`root`用户下操作
- `id dmdba` 检查用户是否存在
- `groupadd dinstall` 添加用户群组
- `useradd -g dinstall dmdba` 添加用户`dmdba`并分配到群组`dinstall`
- `password dmdba` 给用户设置密码

### 1.3 挂载达梦数据库ISO镜像文件 <span id="index01.03"></span>
在`root`用户执行：`mount -o loop $DM_ISO_PATH $INSTALL_PATH`这条命令把镜像文件`$DM_ISO_PATH`挂载到系统的路径`$INSTALL_PATH`。
`mount`命令必须在root权限下执行，在普通用户下执行会报错。

## 2. 安装数据库 <span id="index02"></span>

### 2.1 安装 <span id="index02.01"></span>
- 个人建议在root用户下直接安装，安装好后再切换到普通用户
- `cd $INSTALL_PATH` 进入到安装包路径
- `./DMInstall.bin` 执行安装文件
    - 如果提示权限不足，请在`root`用户下执行`chmod +x DMInstall.bin`给安装文件添加可执行权限
- 根据提示一路下一步
- 如果在`roo`用户下执行安装，在安装完后执行`chown dmdba:dinstall -R $DM_HOME`命令，把达梦数据库的所有权限赋给`dmdba`用户，否则会出现在`dmdba`用户无法使用一些基础命令问题
- 让系统所有用户都可以执行达梦数据库命令：`chmod -R 777 $DM_HOME/bin $DM_HOME/tool`

### 2.2 系统环境变量配置 <span id="index02.02"></span>
- `vi ~/.bash_profile` 配置当前用户的环境变量
- `vi /etc/profile` 在以`root`权限打开该文件，配置所有用户的环境变量
- 在文件的尾部插入内容

    ```bash
    export DM_HOME=#达梦数据库安装路径
    export PATH=$PATH:$DM_HOME/bin:$DM_HOME/tool
    ```
- 然后刷新环境变量：`source ~/.bash_profile` 和 `source /etc/profile`

## 3. 创建数据库实例（初始化数据库） <span id="index03"></span>
初始化数据库使用图形化界面初始化比较方便，很直观，一般情况下直接下一步就行了，其他的基本上都不用管，如果想要自定义，那就在初始化的时候注意一下相关的参数，稍微改一下就行了。

## 4. 数据库实例的启动和关闭 <span id="index04"></span>
根据前面的创建数据库实例，我们得到一个实例名：`DMSERVER`(默认的)

- `service DmServiceDMSERVER start` 启动数据库实例
- `service DmServiceDMSERVER stop` 停止数据库实例
- `service DmServiceDMSERVER restart` 重启数据库实例
- 其中的`DmServiceDMSERVER`服务名称是根据创建数据库实例的时候有个实例名`DMSERVER`然后和`DmService`拼接起来得到的，也可以执行`ls /etc/init.d/|grep DmService`查看系统中注册了哪些达梦数据库实例

## 5. 连接数据库命令： disql <span id="index05"></span>
在命令行中连接达梦数据库是使用`disql`这个命令，如果直接输入`disql`提示无此命令的时候可能是环境变量没有配置好，此时请参考[安装数据库:系统环境变量配置](#index02.02)

- 语法：`disql [用户名]/[密码]@[主机]:[端口]`，示例：`disql sysdba/password@dameng.com:5236`
- 也可以直接输入`disql`命令，然后根据提示输入用户名和密码

## 6. 切换数据库状态 <span id="index06"></span>
数据库状态有三种：`mount`配置模式、`open`打开模式、`suspend`挂起模式。
- 语法：`alter database 状态值`，示例切换为打开状态：`alter database open`
- 注意：该命令需要在`disql`连接到数据库后执行

## 7. SQL语句一些基础使用
- `between` 示例：`where age between 18 and 25`
- `round` 四舍五入 `round(4.5) = 5` `round(56.789) = 56.79`
- `asb` 绝对值 `asb(-1) = 1`
- `mod` 取模 `10 mod 3 = 1`
- `upper` 转换成大写 `upper('aBc') = 'ABC'`
- `lower` 转换成小写 `lower('AbC') = 'abc'`
- `initcap` 单词首字母大写 `initcap('user name') = 'User Name'`
- `instr` 查找字符串位置 `instr('this is varchar','var') = 9`
- `substr` 截取字符串 `substr('this is varchar',3,7) = 'is is v'`
- `lpad` 左侧补齐空格 `lpad('a',3) = '  a'`
- `rpad` 右侧补齐空格 `rpad('a',3) = 'a  '`
- `trim` 删除首尾空格 `trim(' a  ') = 'a'`
- `concat` 字符串连接 `concat('1','a','c','2,') = '1ac2,'`
- `||` 字符串连接 `'1'||'a'||'c'||'2,' = '1ac2,'`
- `sysdate` 系统时间（精确到秒）
- `now` 系统时间（精确到百万分之一秒）
- `next_day` 计算某日期下周某一天`下一个周日：next_day('2019-09-12',1) = '2019-09-15'`，其中1-7表示周日到周六
- `last_day` 某月最后一天 `last_day('2019-09-12') = '2019-09-30'`
- `to_char` 时间日期格式化 `to_char(sysdate,'yyyy-mm-dd hh24:mi:ss') = '2019-09-06 22:51:29'`
- `any` 查询一些列集合的信息，只要匹配成功其中一条数据即可 `where field >any(select...) and field <any(select...)`
- `all` 查询一些列集合的信息，需要匹配成功其中全部数据 `where field >all(select...) and field <all(select...)`
- `count` 计数 `select count(*),count(field)`
- `sum` 求和 `select sum(field)`
- `avg` 平均数 `select avg(field)`
- `max` 最大值 `select max(field)`
- `min` 最小值 `select min(field)`
- `group by` 分组统计 `select count(field),sum(field),avg(field),max(field),min(field),type from tableName group by type`
- `order by asc/desc` 排序（`asc`升序/`desc`降序） `select * from tableName where id > 100 order by id desc limit,2,10`
- `join` 内连接（自然连接） `select * from t1 join t2 on t1.id1 = t2.id2`
- `left join` 左外连接 `select * from t1 left join t2 on t1.id1 = t2.id2`
- `right join` 右外连接 `select * from t1 right join t2 on t1.id1 = t2.id2`
- `full join` 全外连接 `select * from t1 full join t2 on t1.id1 = t2.id2`
- `exsits` 是否存在 `select * from t1 where id exsits (select id from t1 where name is null)`

## 8. 数据库设置成归档模式运行

### 8.1 管理工具操作方式
其实这个操作在管理工具里面进行是非常方便的，并且可以很直观的配置。

1. 打开DM管理工具
2. 连接上数据库
3. 在数据库连接的条目上单击右键，在弹出的菜单中选择**管理服务器**
4. 在**管理服务器**窗口左侧的**系统管理**切换数据库状态为**配置**模式
5. 在**管理服务器**窗口左侧的**归档配置**中开启归档模式和配置归档路径、归档类型、文件大小等
6. 然后再到**系统管理**把数据库切换为**打开**状态
7. 最后确定即可

### 8.2 命令行操作方式
- `alter database mount;` 切换到配置模式
- `alter database add archivelog 'type=local,dest=/dm7/arch,file_size=64,space_limit=0';` 配置归档文件
- `alter database archivelog;` 切换到归档状态
- `alter database open;` 数据库切换到打开状态
- `select * from v$dm_arch_ini;` 检验是否切换成功

数据库切换到归档状态后只有重启数据库服务才能实现热备，否则在进行热备的时候将提示以下错误信息：
```
错误号:   -8216

错误消息: 指定或者默认归档目录中找不到完整归档日志
```

## 9. 同义词管理
同义词是数据库对象的一个别名，经常用于简化对象访问和提高对象访问的安全性，和视图的功能类似，是一种映射关系，能够在不同的数据库用户之间实现无缝交互。
同义词有两种类型，分别是公用同义词与私有同义词。普通用户创建的同义词一般都是私有同义词，公有同义词一般由DBA创建，普通用户如果希望创建同义词，则需要CREATE PUBLIC SYNONYM这个系统权限。

### 9.1 公用同义词
由一个特殊的用户组Public所拥有。顾名思义，数据库中所有的用户都可以使用公用同义词。公用同义词往往用来标示一些比较普通的数据库对象，这些对象往往大家都需要引用。
- 创建命令：`create public synonym 同义词名称 for 模式名称.表名称;`
- 修改命令：`create or replace public synonym 同义词名称 for 模式名称.表名称;`
- 删除命令：`drop public synonym 同义词名称`

### 9.2 私有同义词
它是跟公用同义词所对应，他是由创建他的用户所有。当然，这个同义词的创建者，可以通过授权控制其他用户是否有权使用属于自己的私有同义词。
- 创建命令：`create synonym 模式名称.同义词名称 for 模式名称.表名称;`
- 修改命令：`create or replace synonym 模式名称.同义词名称 for 模式名称.表名称;`
- 删除命令：`drop synonym 模式名称.同义词名称`

## 10. 数据库备份
### 10.1 物理备份
#### 10.1.1 冷备
条件：DMAP服务打开状态，数据库关闭状态
##### 命令行执行命令备份
命令：`backup database '$DM_HOME/data/实例名称/dm.ini'`
命令执行条件：需要进入`$DM_HOME/bin`路径下，执行`dmrman`命令，在`dmrman`交互中输入上方的备份命令

##### 操作界面执行命令备份
打开**DM控制台工具**，在左侧菜单**备份还原**里面进行备份，此方式可以很方便的操作备份数据库。

#### 10.1.2 热备
##### 命令行执行命令备份
条件：DMAP服务打开状态，数据库打开状态，数据库开启归档模式
- 完全备份：`backup database full backupset 备份保存路径;`
- 增量备份：`backup database increment backupset 备份保存路径;`

##### 操作界面执行命令备份
在**DM管理工具**中，连接数据库后，在左侧有一个**备份**菜单，在此处可以很方便的对**库、表、表空间、归档**进行备份，只需要根据页面提示进行操作

### 10.2 逻辑备份
- 导出：`dexp sysdba/密码 file=/dm7/backup/dexp_bak.dmp tables=emp;`
- 导入：`dimp sysdba/密码 file=/dm7/backup/dexp_bak.dmp tables=emp ignore=y`

### 10.3 表空间和表备份
- 表空间备份：`backup tablespace dmhr backupset 备份保存路径;`
- 表备份：`backup table dmhr.employee backupset 备份保存路径;`

## 11. 作业管理
在**DM管理工具**中，连接数据库后，在左侧有一个**代理**菜单，我们创建代理后才能够创建作业，每个作业可以有多个步骤，每个作业也可以有多个作业调度，一个调度就是一个作业的时间触发条件，比如按天执行、按月执行、定时执行等，每个作业可以有多个时间触发条件，作业被触发执行后，将会按照作业设置的步骤一次执行这些步骤。

## 12. 存储过程
存储过程（Stored Procedure）是一种在数据库中存储复杂程序，以便外部程序调用的一种数据库对象。
存储过程是为了完成特定功能的SQL语句集，经编译创建并保存在数据库中，用户可通过指定存储过程的名字并给定参数(需要时)来调用执行。
存储过程思想上很简单，就是数据库 SQL 语言层面的代码封装与重用。
```SQL
## 语句模板
CREATE PROCEDURE 模式名称.触发器名称( IN VACHAR(50))
AS
    /* 变量说明部分 */
    NAME VARCHAR(50)
BEGIN
    /* 触发器执行体 */
END;

## 示例：
CREATE OR REPLACE PROCEDURE "SYSDBA"."aa"("BH" IN INT)
AS
    dept INT;
BEGIN
    select department_id into dept from emp where employee_id = bh;
    print(dept);
END;
```
## 13. 触发器
触发器分为**库级、模式级、表级、视图级**，触发器可以在操作之前或者操作之后执行，可以是DD、DML、系统事件条件触发执行
触发器有如下作用：
- 可在写入数据表前，强制检验或转换数据。
- 触发器发生错误时，异动的结果会被撤销。
- 部分数据库管理系统可以针对数据定义语言（DDL）使用触发器，称为DDL触发器。
- 可依照特定的情况，替换异动的指令 (INSTEAD OF)。
```SQL
## 语句模板
create trigger 模式名称.触发器名称
before update of 字段 on 模式名称.表名称
for each row
begin
    /* 触发器执行体 */
    insert into TAB2 values(:old.salary,:new.salary);
end;

## 示例：在班级与学生关联关系的记录（学号、班级名称）被发生改变时插入日志
create  or replace trigger "USER01"."ttt01"
after INSERT or UPDATE of "STUID","NAME"
on "USER01"."CLASSNAME"
referencing OLD ROW AS "OLD" NEW ROW AS "NEW"
for each row
BEGIN
        /*触发器体*/
        print(:OLD.STUID);
        print(:OLD.NAME);
        print(:NEW.STUID);
        print(:NEW.NAME);
        insert into "USER01"."LOG"("TEXT")
        VALUES
        (concat('学号：旧STUID：',:OLD.STUID,'，新STUID：',:NEW.STUID,'，时间：',now()));
        insert into "USER01"."LOG"("TEXT")
        VALUES
        (concat('姓名：旧NAME：',:OLD.NAME,'，新NAME：',:NEW.NAME,'，时间：',now()));
END;
```

## 14. 控制文件管理
控制文件管理常用的有两个，一个是控制文件备份路径，另一个是控制文件备份数目，这两个参数可以到**DM控制台工具**配置，在左侧的**DM控制台>服务器配置>实例配置>数据库实例**里面搜索`CTL`可以看到控制文件的相关参数，修改保存后重启服务器即可生效。

## 15. 重做日志文件管理
重置日志文件管理在**DM管理工具**中，连上服务器后，在左侧的实例右键选择**管理服务器**，在弹出的窗口中可以看到有**日志文件**选项，在这里可以管理数据库的日志文件，包含添加日志文件和修改日志文件大小。

## 16. 角色管理
角色管理在管理工具中操作更加方便。
在**DM管理工具**中，连上服务器后，在左侧有个角色菜单，在这里管理系统中的角色信息，可以很方便的新增、修改、删除角色信息，并且给角色分配权限，这些操作使用管理工具将会让我们的工作变得更加简单。
- 系统权限：对系统的一些操作，比如用户、模式、表、视图等
- 对象权限：针对某个模式下的表、视图、过程进行授权，可以精确到某个表的某个列属性来授权

## 17. 用户管理
系统中存在四种类型的用户**管理用户、审计用户、安全用户、系统用户**，一般情况下我们使用的是**管理用户**类型的用户。
在创建用户的时候，我们可以对用户分配指定的表空间、索引表空间、角色、系统权限、对象权限、资源权限，其中资源权限定义了用户的资源限制，比如会话数、会话空闲期、登录失败次数、口令有效期、口令锁定期等对用户与数据库连接、登录的一些限制。
如果有多个用户属于同一种类型，他们都拥有一些相同的权限，那么我们可以把这些相同的权限分配给一个新的角色，然后给用户分配这个角色，这样便于用户与权限的管理。

## 18. 常见问题

### 18.1 DMAP服务无法启动
删除`$DM_HOME/bin/`目录下以`DM_PIPE_DMAP`开头的文件：`rm $DM_HOME/bin/DM_PIPE_DMAP*`
### 18.2 `dmrman`命令无法备份，提示连接管道超时或者打开/创建文件失败
需要进入`$DM_HOME/bin`路径下再执行`dmrman`命令。
因为该命令会在当前目录下访问DMAP服务的管道文件，而DMAP的管道文件存放`$DM_HOME/bin`目录，但是`dmrman`命令又不够智能自动识别DMAP管道文件，该命令只会尝试在当前运行该命令的目录找管道文件，因此不在`$DM_HOME/bin`运行`dmrman`命令就会报错。
### 18.3 使用DM控制台工具冷备数据库报连接管道超时
在DM服务查看器中启动DMAP服务，切换到`dmdba`用户启动DM控制台工具，然后再执行备份操作，如果还有问题，可能是数据库安装目录里面的一些文件权限有问题，此时切换到`root`用户，给数据库目录设置权限`chown dmdba:dinstall -R $DM_HOME`，此时就能够正常的执行备份了。
为什么需要设置目录权限？可能是使用root用户安装数据库导致的遗留问题，里面有些文件/目录的权限被分配给了`root:root`，因此需要重新设置为`dmdba:dinstall`。
### 18.4 为什么我的模式名、表名、列名写对了，但是语句还是报错
**当遇到这个问题的时候，其实是真的输错了，错在大小写敏感的问题上了。**

    重点：
    当我们在命令行中创建表时，对表名、列名不使用双引号包含起来，此时这条语句无论是大写字母还是小写字母或者大小写混合，系统都会把没有用双引号包含的内容自动转换成大写的。
    同理，在select/insert/update/delete的时候，只要模式名、表名、列名没有用双引号包含起来，系统就会自动转换成为大写的。

在DM管理工具中创建表的时候，管理工具为了保证我们的视觉与系统的一致性，会自动给所有的模式名、表名、列名加上双引号，因此此时你写的小写，那么保存的时候就是小写，这时候就容易造成一个很迷人的问题。

- 示例说明：命令行操作
就是在命令行中创建表`create table test.tableName(name varchar(20) not null)`，会被系统翻译为：`create table "TEST"."TABLENAME"("NAME" varchar(20) not null)`
在命令行中查询时情况是这样的：
    - `select * from test.tableName` 正常
    - `select * from test.TableName` 正常
    - `select * from test.TabLeName` 正常
    - `select * from test.TABLENAME` 正常
    - `select * from test.TABLENAME where name = 1` 正常
    - `select * from test.TABLENAME where NAME = 1` 正常
    - `select * from test.TABLENAME where "NAME" = 1` 正常
    - `select * from test."TABLENAME" where "NAME" = 1` 正常
    - `select * from test."tableName"` 错误
    - `select * from test.TABLENAME where "name" = 1` 错误

- 示例说明：管理工具操作
但是我们在DM管理工具中创建一个`tableName`和列`name`，工具会为我们自动给表名、列名添加双引号，实际上的语句就是这样的：`create table "TEST"."tableName"("name" varchar(20) not null)`，此时我们在命令行中插数据的时候会发现迷人的错误：
    - `select * from test."tableName"` 正常
    - `select * from test."tableName" where "name" = 1` 正常
    - `select * from test.TableName` 错误
    - `select * from test.TabLeName` 错误
    - `select * from test.TABLENAME` 错误
    - `select * from test."tableName" where name = 1` 错误
    - `select * from test."tableName" where NAME = 1` 错误

因此这个大小写敏感是一个很感人的事情，所有没有加引号的字母都被转为大写，而DM管理工具会为我们自动添加引号。

## 附录
### dm.ini 几个常见关键词说明
- `RECYCLE` 快速回收池
- `KEEP` 保留池
- `DICT_BUF_SIZE` 字典缓冲区
- `BUFFER` 缓冲区
- `RLOG_BUF_SIZE` 日志缓冲区的大小，单位page设置成2的幂次方
- `RLOG_POOL_SIZE` 最大日志缓冲区大小，单位为M
- `SORT` 排序缓冲区
- `CTL_PATH` 控制文件的存放路径
- `CTL_BAK_NUM` 控制文件的备份数量
- `CTL_BAK_PATH` 对控制文件做备份的路径

### 系统的一些表和视图
- `dba_data_files` 数据文件、表空间
- `v$datafile` 数据文件
- `dba_tablespaces` 表空间
- `v$tablespace` 表空间
- `dba_free_space` 表空间剩余
- `v$instance` 数据库实例状态
- `v$dm_ini` 数据库参数，可以通过DM控制台工具更直观的查看和修改
- `v$sessions` 会话信息
- `v$datafile` 数据文件
- `v$rlogfile` 日志文件

### 常用的几个查询语句

1. 查询缓冲设置信息

    ```
    select para_name,para_value from v$dm_ini where para_name = 'BUFFER';
    select para_name,para_value from v$dm_ini where para_name like '%BUFFER%';
    ```
2. 查询数据库状态(查询实例)

    ```
    select status$ from v$instance;
    ```

3. 查看某个参数的类型：

    ```
    select name,type from v$parameter where name = 'BUFFER';
    ```

4. 修改数据库的缓冲区大小：

    ```
    公式：sp_set_para_value(scope, para_name, para_value)
    ```

5. 查看当前正在激活（运行）的会话

    ```
    select SESS_ID,sql_text,state from v$sessions where state = 'ACTIVE';
    ```

6. 查看当前数据库文件存放路径

    ```
    select path from v$datafile;
    ```

7. 查看当前数据库的日志文件路径

    ```
    路径：select path from v$rlogfile;
    路径+大小：select path, rlog_size/1024/1024 from v$rlogfile;
    修改日志文件大小：alter database resize logfile 'log file path' to 512;
    ```

8. 查询表空间和数据文件对应关系

    ```
    select file_name,tablespace_name from dba_data_files;
    ```

9. 关闭一个长时间未完成的会话

    ```
    sp_close_session(会话ID);
    ```