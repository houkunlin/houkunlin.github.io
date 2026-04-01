---
title: Docker PostgreSQL 17 升级 18
date: 2026-04-01 23:35:00
updated: 2026-04-01 23:35:00
tags:
  - docker
  - postgres
---

PostgreSQL 18 已经出来挺久了，最近才抽空开始升级一下，今年升级又又又踩坑了。

这次从 postgres 17 升级到 postgres 18 又搞了一天，去年的教程，有用，但是还是遇到一些新的问题。

我们需要起3个容器，一个用于旧版的数据库，一个用于新版的数据库，一个用于迁移。

```yaml
version: '3.5'

services:
  # 旧版的数据库容器
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
      - "5433:5432"
  # 新版的数据库容器
  postgres18:
    image: postgres:18.3
    restart: no
    networks:
      - app
    volumes:
      - /volume1/docker/postgresql/data_18:/var/lib/postgresql/data_18
    environment:
      - 'TZ=Asia/Shanghai'
      - 'POSTGRES_USER=postgres'
      - 'POSTGRES_PASSWORD=postgres'
      - 'PGDATA=/var/lib/postgresql/data_18'
    ports:
      - "5434:5432"
  # 迁移用的数据库容器
  postgres18-migrate:
    image: postgres:18.3
    restart: no
    networks:
      - app
    volumes:
      - /volume1/docker/postgresql/data_18:/var/lib/postgresql/data_18
      - /volume1/docker/postgresql/data_17:/var/lib/postgresql/data_17
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

在 postgres17 容器中打包旧版的bin相关文件和停掉 postgres17 服务
```bash
# 备份这个动态链接库，后续要用到
cp /usr/lib/x86_64-linux-gnu/lib* /var/lib/postgresql/data_17/lib17_so/
# 打包 postgres17 的bin相关文件
tar -czvf postgres17.tar.gz /usr/lib/postgresql/17/ /usr/share/postgresql/17/
mv postgres17.tar.gz /var/lib/postgresql/data_17/

su - postgres
# 用以下命令停止 postgres17 服务
/usr/lib/postgresql/17/bin/pg_ctl -D /var/lib/postgresql/data_17 stop -s -m fast
# 请不要直接停掉 postgres17 的容器，可能因为 postgres17 未正确停止，导致后续 pg_upgrade 检查失败
```

在 postgres18 启动完毕后并初始化完毕后停掉 postgres18 服务
```bash
su - postgres
# 用以下命令停止 postgres18 服务
/usr/lib/postgresql/18/bin/pg_ctl -D /var/lib/postgresql/data_18 stop -s -m fast
# 请不要直接停掉 postgres18 的容器，可能因为 postgres18 未正确停止，导致后续 pg_upgrade 检查失败
```

在 postgres18-migrate 中把 postgres17 的bin相关文件解压，这个容器中需要同时存在 postgres17 和 postgres18 的bin相关文件
```bash
# 解压 postgres17 的bin相关文件
mv /var/lib/postgresql/data_17/postgres17.tar.gz /
cd /
tar -xvzf postgres17.tar.gz

# 验证是否能够执行旧版本命令，会缺一些动态库，然后需要逐个的从 lib17_so 复制过来。目前是一定无法执行的
/usr/lib/postgresql/17/bin/postgres -V

# 把这几个库复制过来，才能执行旧版本的命令
cp /var/lib/postgresql/data_17/lib17_so/libldap-2.5.so.0 /usr/lib/x86_64-linux-gnu/
cp /var/lib/postgresql/data_17/lib17_so/libicui18n.so.72 /usr/lib/x86_64-linux-gnu/
cp /var/lib/postgresql/data_17/lib17_so/libicuuc.so.72 /usr/lib/x86_64-linux-gnu/
cp /var/lib/postgresql/data_17/lib17_so/liblber-2.5.so.0 /usr/lib/x86_64-linux-gnu/
cp /var/lib/postgresql/data_17/lib17_so/libicudata.so.72 /usr/lib/x86_64-linux-gnu/

# 验证是否能够执行旧版本命令
/usr/lib/postgresql/17/bin/postgres -V

# 切换到 postgres 用户
su - postgres

# 执行升级检查，这个时候是一定会失败的，因为 18 默认启用了校验和
/usr/lib/postgresql/18/bin/pg_upgrade -c -b /usr/lib/postgresql/17/bin -B /usr/lib/postgresql/18/bin -d /var/lib/postgresql/data_17 -D /var/lib/postgresql/data_18

# 删除 18 数据目录下的所有文件，重新初始化
rm -rf /var/lib/postgresql/data_18/*
/usr/lib/postgresql/18/bin/initdb -D /var/lib/postgresql/data_18 --no-data-checksums

# 执行升级检查，这时候才会正常，但是依旧会有一些警告，先不管，最后再处理
/usr/lib/postgresql/18/bin/pg_upgrade -c -b /usr/lib/postgresql/17/bin -B /usr/lib/postgresql/18/bin -d /var/lib/postgresql/data_17 -D /var/lib/postgresql/data_18

# 执行升级，但是依旧会有一些警告，先不管，最后再处理
/usr/lib/postgresql/18/bin/pg_upgrade -b /usr/lib/postgresql/17/bin -B /usr/lib/postgresql/18/bin -d /var/lib/postgresql/data_17 -D /var/lib/postgresql/data_18
# 命令参数说明：pg_upgrade -c -b oldbindir -B newbindir -d olddatadir -D newdatadir

# 重新启用校验和
/usr/lib/postgresql/18/bin/pg_checksums -D /var/lib/postgresql/data_18 --enable --progress --verbose
```


启动 postgres18 容器，进入容器内部运行 `/usr/lib/postgresql/18/bin/vacuumdb --all --analyze-in-stages` 命令，这时候依旧会有很多的警告。

警告内容类似：

```
WARNING:  database "postgres-test" has a collation version mismatch
DETAIL:  The database was created using collation version 2.36, but the operating system provides version 2.41.
HINT:  Rebuild all objects in this database that use the default collation and run ALTER DATABASE "postgres-test" REFRESH COLLATION VERSION, or build PostgreSQL with the right library version.
```

用数据库连接工具，连接到 postgres18 数据库，执行以下命令：

```sql
-- 对每个数据库执行以下命令
ALTER DATABASE "[表名]" REFRESH COLLATION VERSION;
```

然后再进入到 postgres18 容器，在容器内部运行 `/usr/lib/postgresql/18/bin/vacuumdb --all --analyze-in-stages` 命令，这时候就看到没有警告了，此时就可以正常使用 postgres18 服务了。

## 错误内容

遇到错误，可以去看一下去年 16 升级 17 的教程，应该会有一些帮助。

今年遇到的问题，一个是因为 18 默认启用了校验和，导致升级失败，一个是缺少一些动态链接库，导致旧版本命令执行失败，再有就是 18 启用了校验和，导致的一些问题。
