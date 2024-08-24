---
title: MySQL 手动将主库的数据导入从库
date: 2021-02-16 21:42:04
updated: 
tags: 教程
categories: MySQL 主从复制
---

> 运行环境：Windows 10 Docker Desktop WSL 2 based engine

## 搭建演示环境

### 主库

拉取 MySQL 8.0 版本镜像并启动，监听在 3306 端口，设置 root 用户密码为 root

```bash
docker run --name mysql -p 3316:3306 -e MYSQL_ROOT_PASSWORD=root -e MYSQL_ROOT_HOST=% -d mysql:latest
```

为了演示，首先在主库插入一些数据，用到下面的 SQL 语句

- [初始化数据库表](https://github.com/solidSpoon/CodeSnippet/blob/main/InitDB/InitSchema.sql)
- [在这个表中插入数据](https://github.com/solidSpoon/CodeSnippet/blob/main/InitDB/InsertData.sql)

### 从库

运行一个新的 MySQL 8.0，名字叫做 mysql_bk1 作为从库

```bash
docker run --name mysql_bk1 -p 3309:3306 -e MYSQL_ROOT_PASSWORD=root -e MYSQL_ROOT_HOST=% -d mysql:latest
```

## 数据导入导出
首先确保主库的数据不再变化


进入准备好的目录

```bash
mkdir backup
cd backup
```

将主库的数据备份到本地

```bash
docker exec mysql-master /usr/bin/mysqldump -u root --password=admin123 test > backup.sql
```

进入从库，建立数据库 test

```bash
docker exec -ti mysql_bk1 mysql -u root -p
create database test;
```

Ctrl-D 退出，执行下面的命令导入数据到新开的 MySQL 中，这个需要花费好几分钟，请耐心等待；


当然也可以 `docker cp backup.sql` 文件到容器中，然后连上数据库使用 `source backup.sql`

```bash
cat backup.sql | docker exec -i mysql_bk1 /usr/bin/mysql -u root --password=root test
```
完成
