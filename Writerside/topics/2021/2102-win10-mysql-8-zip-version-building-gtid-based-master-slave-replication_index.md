---
slug: win10-mysql-8-zip-version-building-gtid-based-master-slave-replication
title: Win10 - MySQL 8 Zip 版 - 搭建基于 GTID 的主从复制
authors: [solidSpoon]
tags: [教程]
---

> 配置环境：Windows 10 - MySQL 压缩版

## 前言

GTID 是干嘛的？


> GTID (Global Transaction IDentifier) 是全局事务标识。它具有全局唯一性，一个事务对应一个GTID。唯一性不仅限于主服务器，GTID在所有的从服务器上也是唯一的。一个GTID在一个服务器上只执行一次，从而避免重复执行导致数据混乱或主从不一致。
>
> 在传统的复制里面，当发生故障需要主从切换时，服务器需要找到binlog和pos点，然后将其设定为新的主节点开启复制。相对来说比较麻烦，也容易出错。在MySQL 5.6里面，MySQL会通过内部机制自动匹配GTID断点，不再寻找binlog和pos点。我们只需要知道主节点的ip，端口，以及账号密码就可以自动复制。
> [http://mysql.taobao.org/monthly/2020/05/09/](http://mysql.taobao.org/monthly/2020/05/09/)



在传统的主从复制中，我们需要在从库中指定主库的 Log 文件与 Position ，在基于 DTID 的主从复制中，不需要这一步骤。

## 准备两个 MySQL 服务实例

8.0 压缩版下载地址：

> [https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-8.0.16-winx64.zip](https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-8.0.16-winx64.zip)

解压后再复制一份，假设命名为 `mysql-8.0.16-winx64` 和 `mysql-8.0.16-winx64-2`

## 修改主 MySQL-8的 `my.ini`

在 `mysql-8.0.16-winx64` 目录下添加 my.ini 文件，内容如下：

```yaml
[mysqld]
basedir = ./
datadir = ./data
port = 3306
server_id = 1

gtid_mode=ON
enforce-gtid-consistency
sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES 
log_bin=mysql-bin
binlog-format=Row

```

## 修改从 MySQL-8的 `my.ini`

在从 `mysql-8.0.16-winx64-2` 目录下添加 my.ini 文件，内容如下：

```yaml
[mysqld]
basedir = ./
datadir = ./data
port = 3316
server_id = 2

gtid_mode=ON
enforce-gtid-consistency
sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES 
log_bin=mysql-bin
binlog-format=Row
```


## 初始化和启动数据库

空数据库需要初始化，分别在两个数据库的 `\bin` 目录下执行 `mysqld --initialize-insecure` 进行初始化。


分别启动主和从，在两个数据库的 `\bin` 目录下直接执行 `mysqld` 或 `start mysqld` 命令即可。

## 配置主节点

用 `mysql` 命令登录到主节点：

```bash
mysql -uroot -P3306
```

进入后执行下面命令

```bash
mysql> CREATE USER 'repl'@'%' IDENTIFIED BY '123456';
Query OK, 0 rows affected (0.11 sec)

mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
Query OK, 0 rows affected (0.12 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.10 sec)
```


创建数据库：

```bash
create schema db;
```

### 主节点证书文件


```bash
mysql> SHOW GLOBAL VARIABLES LIKE 'caching_sha2_password_public_key_path';
+---------------------------------------+----------------+
| Variable_name                         | Value          |
+---------------------------------------+----------------+
| caching_sha2_password_public_key_path | public_key.pem |
+---------------------------------------+----------------+
1 row in set (0.00 sec)
```


这个文件在初始化后位于 `\bin\data` 下


因为 MySQL 8 默认是用 `caching_sha2_password` 做认证插件，这点与 MySQL 5 不同：

```bash
error connecting to master 'repl@localhost:3306' - retry-time: 60 retries: 18 message: Authentication plugin 'caching_sha2_password' reported error: Authentication requires secure connection.
```


这个文件就是基于默认设置 `caching_sha2_password` 下的通讯公钥文件。默认情况服务器不会给客户端发送，所以需要拷贝到从节点的目录下。


如果不想拷贝的话 ：

MySQL 8.0 的版本要在**从数据库**初始设置（CHANGE MASTER TO）加：

`MASTER_PUBLIC_KEY_PATH = 'key_file_name'`

或者

`GET_MASTER_PUBLIC_KEY = {0|1}`

## 配置从节点


把刚才的 `public_key.pem` 文件改名为 `master_public_key.pem` 然后拷贝到从服务器的 `\bin\data` 文件夹中，这个文件夹在用上面的命令初始化之后才有。

`mysql` 命令登录到从节点：

```bash
mysql -uroot -P3316
```


```bash
CHANGE MASTER TO
    MASTER_HOST='localhost',  
    MASTER_PORT = 3306,
    MASTER_USER='repl',      
    MASTER_PASSWORD='123456',   
    MASTER_PUBLIC_KEY_PATH='master_public_key.pem',
    MASTER_AUTO_POSITION = 1;
```


这里有个问题，MySQL 8 下面不需要创建 db 。否则会报错说已经存在 db 。
--创建数据库：create schema db;--


直接开始执行同步
`start/stop slave;`


## 验证操作

### 主库

在主库执行：


```bash
mysql> use db
Database changed
mysql> create table t1(id int);
Query OK, 0 rows affected (0.17 sec)

mysql>
mysql>
mysql> insert into t1(id) values(1),(2);
Query OK, 2 rows affected (0.04 sec)
```


### 从库

在从库查看数据同步情况


```bash
mysql> use db
Database changed
mysql>
mysql>
mysql> show tables;
+--------------+
| Tables_in_db |
+--------------+
| t1           |
+--------------+
1 row in set (0.00 sec)

mysql>
mysql>
mysql> select * from t1;
+------+
| id   |
+------+
|    1 |
|    2 |
+------+
2 rows in set (0.00 sec)
```


## 查看命令

可以通过 `show master status\G`，`show slave status\G` 查看状态，或定位一些问题


可以能改过 `stop slave;`  `start slave;` 来停止复制。

