---
slug: win10-mysql-8-0-docker-building-master-slave-replication
title: Win10 - MySQL 8.0 - Docker 搭建主从复制
authors: [solidSpoon]
tags: [教程]
---


> 搭建环境：Windows 10 Docker Desktop WSL 2 based engine

## 创建

首先创建两个数据库，一个作为主库，另一个作为从库

### 编写 docker-conpose 文件


一般安装了 docker 都会安装 docker-compose，可以使用 `docker-compose -verison` 查看是否安装

```bash
➜  ~ docker-compose -verison
docker-compose version 1.27.4, build 40524192
```

新建一个文件夹「mysqlms」用于存放本次搭建的文件
在 「mysqlms」 文件夹新建 dokcer-compose 文件，文件名为 「docker-compose.yaml」

```yaml
version: '2' 
networks:
  byfn:                                       #配置byfn网络
services:
  master:                                     #配置master服务
    image: 'mysql:latest'                        #使用刚才pull下来的镜像
    restart: on-failure
    container_name: mysql-master              #容器起名 mysql-master
    environment:
      MYSQL_USER: test
      MYSQL_PASSWORD: admin123
      MYSQL_ROOT_PASSWORD: admin123           #配置root的密码
    ports:
      - '3316:3306'                           #配置端口映射
    volumes:
      - "./master/my.cnf:/etc/mysql/my.cnf"   #配置my.cnf文件挂载，
      - "./master/mysql-files:/var/lib/mysql-files" #MySQL 8.0 不加这一行会报错
    networks:
      - byfn                                  #配置当前servie挂载的网络
  slave:                                      #配置slave服务
    image: 'mysql:latest'
    restart: on-failure
    container_name: mysql-slave
    environment:
      MYSQL_USER: test
      MYSQL_PASSWORD: admin123
      MYSQL_ROOT_PASSWORD: admin123
    ports:
      - '3326:3306'
    volumes:
      - "./slave/my.cnf:/etc/mysql/my.cnf"
      - "./master/mysql-files:/var/lib/mysql-files" #MySQL 8.0 不加这一行会报错
    networks:
      - byfn
```

`- "./master/mysql-files:/var/lib/mysql-files"` 这一行的目的详见下面链接

> [https://github.com/docker-library/mysql/issues/541](https://github.com/docker-library/mysql/issues/541)



如果想明确指定 MySQL 版本，如： `image: 'mysql:8'`

### 编写 cnf 文件

在 「mysqlms」新建两个文件夹 「master」 和 「slave」，在其中分别写入文件「my.cnf」


- mysqlms/master/my.cnf 内容如下：

```
[mysqld]
server-id=1
log-bin=/var/lib/mysql/mysql-bin
```

- mysqlms/slave/my.cnf 内容如下

```
[mysqld]
server-id=2
```

在「mysqlms」目录下执行如下命令

```bash
docker-compose -f docker-compose.yaml up -d
```
此时两个主从数据库创建成功

## 配置

### Master

启动之后进入 master 容器

```bash
docker exec -it mysql-master /bin/bash
mysql -uroot -padmin123
```

此时进入了 MySql 终端，创建用于主从复制的用户

```bash
create user 'repl'@'%' identified with 'mysql_native_password' by 'admin123';
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl'@'%'; 
```

这里有一个需要注意的地方，应该是 MySQL 5.0 与 MySQL 8.0 的一个验证身份的比较大的区别：


MySql 8.0 默认指定使用需要 SSL 的身份验证插件 「caching_sha2_password」，意味着我们从库必须使用安全链接来连接到主库才可以，否则从库链接的时候会报错，错误代码为 2061，因此这里选择绕过这个插件，改为使用「mysql_native_password」来验证。有机会可以尝试以下使用安全连接。


```bash
flush privileges;
show master status;
```

```bash
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000003 |      767 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

需要记住 File 名字，和 Position 偏移位置

### Slave

在另一个终端进入 slave  容器


```bash
docker exec -it mysql-slave /bin/bash
mysql -uroot -padmin123
```

进入 MySQL 终端之后

```bash
mysql> CHANGE MASTER TO MASTER_HOST='mysql-master', MASTER_PORT=3306,  MASTER_USER='repl', MASTER_PASSWORD='admin123', MASTER_LOG_FILE='mysql-bin.000003', MASTER_LOG_POS=767;
mysql> start slave;
```

这里两个参数 `MASTER_LOG_FILE` 和 `MASTER_LOG_POS` 就是前面 master 上最后查询出来的

```bash
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: mysql-master
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000003
          Read_Master_Log_Pos: 1116
               Relay_Log_File: eefecaed2964-relay-bin.000002
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table
......
```

可以从中看到一些信息：

```bash
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
```

可见主从配置成功

---

参考链接

- [https://blog.csdn.net/wawa8899/article/details/86689618](https://blog.csdn.net/wawa8899/article/details/86689618)
- [https://dev.mysql.com/doc/refman/8.0/en/replication-howto-repuser.html](https://dev.mysql.com/doc/refman/8.0/en/replication-howto-repuser.html)
- [https://github.com/docker-library/mysql/issues/541](https://github.com/docker-library/mysql/issues/541)
- [https://blog.csdn.net/wangxiaotongfan/article/details/81870258](https://blog.csdn.net/wangxiaotongfan/article/details/81870258)
