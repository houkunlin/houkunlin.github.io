---
title: Docker PostgreSQL 16 升级 17
date: 2024-12-24 11:21:00
updated: 2024-12-24 11:21:00
tags:
  - docker
  - postgres
---

之前从 postgres 15 升级到 postgres 16 搞了好久，但是没有把操作记录下来。

这次从 postgres 16 升级到 postgres 17 又搞了半天，学乖了，这次顺便把操作过程记录下来。

我们需要起3个容器，一个用于旧版的数据库，一个用于新版的数据库，一个用于迁移。

```yaml
version: '3.5'

# 未完成
services:
  # 旧版的数据库容器
  postgres16:
    image: postgres:16.3
    restart: no
    networks:
      - app
    volumes:
      - /volume1/docker/postgresql/data_16:/var/lib/postgresql/data_16
    environment:
      - 'TZ=Asia/Shanghai'
      - 'POSTGRES_USER=postgres'
      - 'POSTGRES_PASSWORD=postgres'
      - 'PGDATA=/var/lib/postgresql/data_16'
    ports:
      - "5433:5432"
  # 新版的数据库容器
  postgres17:
    image: postgres:17.2
    restart: no
    networks:
      - app
    volumes:
      - /volume1/docker/postgresql/data_17:/var/lib/postgresql/data_17
    environment:
      - 'TZ=Asia/Shanghai'
      - 'POSTGRES_USER=postgres'
      - 'POSTGRES_PASSWORD=postgres'
      - 'PGDATA=/var/lib/postgresql/data_17'
    ports:
      - "5434:5432"
  # 迁移用的数据库容器
  postgres170:
    image: postgres:17.2
    restart: no
    networks:
      - app
    volumes:
      - /volume1/docker/postgresql/data_17:/var/lib/postgresql/data_17
      - /volume1/docker/postgresql/data_16:/var/lib/postgresql/data_16
    environment:
      - 'TZ=Asia/Shanghai'
      - 'POSTGRES_USER=postgres'
      - 'POSTGRES_PASSWORD=postgres'
      - 'PGDATA=/var/lib/postgresql/data'
    ports:
      - "5435:5432"
networks:
  app:
    name: app
```

在 postgres16 容器中打包旧版的bin相关文件和停掉 postgres16 服务
```bash
# 打包 postgres16 的bin相关文件
tar -czvf postgres16.tar.gz /usr/lib/postgresql/16/ /usr/share/postgresql/16/
mv postgres16.tar.gz /var/lib/postgresql/data_16/
# 用以下命令停止 postgres16 服务
/usr/lib/postgresql/16/bin/pg_ctl -D /var/lib/postgresql/data_16 stop -s -m fast
# 请不要直接停掉 postgres16 的容器，可能因为 postgres16 未正确停止，导致后续 pg_upgrade 检查失败
```

在 postgres17 启动完毕后并初始化完毕后停掉 postgres17 服务
```bash
# 用以下命令停止 postgres17 服务
/usr/lib/postgresql/17/bin/pg_ctl -D /var/lib/postgresql/data_17 stop -s -m fast
# 请不要直接停掉 postgres17 的容器，可能因为 postgres17 未正确停止，导致后续 pg_upgrade 检查失败
```

在 postgres170 中把 postgres16 的bin相关文件解压，这个容器中需要同时存在 postgres16 和 postgres17 的bin相关文件
```bash
# 解压 postgres16 的bin相关文件
mv /var/lib/postgresql/data_16/postgres16.tar.gz /
cd /
tar -xvzf postgres16.tar.gz
# 切换到 postgres 用户
su - postgres
# 执行升级检查
/usr/lib/postgresql/17/bin/pg_upgrade -c -b /usr/lib/postgresql/16/bin -B /usr/lib/postgresql/17/bin -d /var/lib/postgresql/data_16 -D /var/lib/postgresql/data_17
# 执行升级
/usr/lib/postgresql/17/bin/pg_upgrade -b /usr/lib/postgresql/16/bin -B /usr/lib/postgresql/17/bin -d /var/lib/postgresql/data_16 -D /var/lib/postgresql/data_17
# 命令参数说明：pg_upgrade -c -b oldbindir -B newbindir -d olddatadir -D newdatadir
```

一切顺利的情况下控制台日志内容
```
postgres@4c367e5fdc31:~$ /usr/lib/postgresql/17/bin/pg_upgrade -c -b /usr/lib/postgresql/16/bin -B /usr/lib/postgresql/17/bin -d /var/lib/postgresql/data_16 -D /var/lib/postgresql/data_17                                                        
Performing Consistency Checks
-----------------------------
Checking cluster versions                                     ok
Checking database user is the install user                    ok
Checking database connection settings                         ok
Checking for prepared transactions                            ok
Checking for contrib/isn with bigint-passing mismatch         ok
Checking data type usage                                      ok
Checking for presence of required libraries                   ok
Checking database user is the install user                    ok
Checking for prepared transactions                            ok
Checking for new cluster tablespace directories               ok

*Clusters are compatible*
postgres@4c367e5fdc31:~$ /usr/lib/postgresql/17/bin/pg_upgrade -b /usr/lib/postgresql/16/bin -B /usr/lib/postgresql/17/bin -d /var/lib/postgresql/data_16 -D /var/lib/postgresql/data_17                                                           
Performing Consistency Checks
-----------------------------
-----------------------------
Checking cluster versions                                     ok
Checking database user is the install user                    ok
Checking database connection settings                         ok
Checking for prepared transactions                            ok
Checking for contrib/isn with bigint-passing mismatch         ok
Checking data type usage                                      ok
Creating dump of global objects                               ok
Creating dump of database schemas
                                                              ok
Checking for presence of required libraries                   ok
Checking database user is the install user                    ok
Checking for prepared transactions                            ok
Checking for new cluster tablespace directories               ok

If pg_upgrade fails after this point, you must re-initdb the new cluster before continuing.

Performing Upgrade
------------------
Setting locale and encoding for new cluster                   ok
Analyzing all rows in the new cluster                         ok
Freezing all rows in the new cluster                          ok
Deleting files from new pg_xact                               ok
Copying old pg_xact to new server                             ok
Setting oldest XID for new cluster                            ok
Setting next transaction ID and epoch for new cluster         ok
Deleting files from new pg_multixact/offsets                  ok
Copying old pg_multixact/offsets to new server                ok
Deleting files from new pg_multixact/members                  ok
Copying old pg_multixact/members to new server                ok
Setting next multixact ID and offset for new cluster          ok
Resetting WAL archives                                        ok
Setting frozenxid and minmxid counters in new cluster         ok
Restoring global objects in the new cluster                   ok
Restoring database schemas in the new cluster
                                                              ok
Copying user relation files
                                                              ok
Setting next OID for new cluster                              ok
Sync data directory to disk                                   ok
Creating script to delete old cluster                         ok
Checking for extension updates                                ok

Upgrade Complete
----------------
Optimizer statistics are not transferred by pg_upgrade.
Once you start the new server, consider running:
    /usr/lib/postgresql/17/bin/vacuumdb --all --analyze-in-stages
Running this script will delete the old cluster's data files:
    ./delete_old_cluster.sh
```

启动 postgres17 容器，进入容器内部运行 `/usr/lib/postgresql/17/bin/vacuumdb --all --analyze-in-stages` 命令，之后就可以正常使用 postgres17 服务了。

## 错误内容

在执行 pg_upgrade 可能会报以下几个错误

**错误一**
```
check for "xxxx" failed: incorrect version: found "postgres (PostgreSQL) 17.0", expected "postgres (PostgreSQL) 16.3"

Failure, exiting
```
> 如果看到此错误，则可能是由于运行旧版本 (16) 而不是新版本 (17) 附带的 pg_upgrade 二进制文件造成的。使用版本 17 的 pg_upgrade 二进制文件的绝对路径来运行正确的二进制文件。
> 
> 也有可能是 -b -B -d -D 的参数搞反了

**错误二**
```
postgres@f14bfc4c19a3:~$ /var/lib/postgresql/data_17/17/bin/pg_upgrade -c -b /var/lib/postgresql/data_16/16/bin -B /var/lib/postgresql/data_17/17/bin -d /var/lib/postgresql/data_16 -D /var/lib/postgresql/data_17                                

There seems to be a postmaster servicing the new cluster.
Please shutdown that postmaster and try again.
Failure, exiting
```
> 这有可能是旧版本的 PostgreSQL 服务器在运行，或者没有正确关闭。
> 
> 请使用 `/usr/lib/postgresql/16/bin/pg_ctl -D /var/lib/postgresql/data_16 stop -s -m fast` 命令关闭服务


## 参考

[1] [Docker版PostgreSQL升级迁移（慎用Watchtower更新基础服务）](https://nigzu.com/docker-postgres-watchtower-upgrade/)

[2] [PostgreSQL升级：使用pg_upgrade进行大版本（16.3）升级（17.0）](https://blog.csdn.net/DBDeep/article/details/142688019)
