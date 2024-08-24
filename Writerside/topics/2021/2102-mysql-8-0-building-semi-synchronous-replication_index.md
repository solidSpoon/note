---
slug: mysql-8-0-building-semi-synchronous-replication
title: MySQL 8.0 搭建半同步复制
authors: [solidSpoon]
tags: [教程]
---

说明：搭建半同步复制需要预先有基本的主从复制环境，具体可以看我之前的文章：

> https://github.com/solidSpoon/solidSpoon.github.io/wiki

半同步超时的时候，会自动降为异步工作。当Slave开启半同步后，或者当主从之间网络延迟恢复正常的时候，半同步复制会自动从异步复制又转为半同步复制，还是相当智能的。

## Master 配置


安装半同步模块并启动


这里 Windows 版本有个坑

对于 Linux 来说：此模块位置在 `/usr/local/mysql/lib/plugin/semisync_master.so`
对于 Windows Zip 版本来说：插件的位置本来在 `..\lib\semisync_master.dll` ，但是 MySQL 默认的位置是 `\bin\lib\plugin\semisync_master.so` 


要解决这个问题，我们把 lib 目录复制一份过去就好了。


也可以在配置文件中指定插件目录：

```bash
[mysqld]
plugin_dir=/path/to/plugin/directory
```

如果 [plugin_dir](https://www.docs4dev.com/docs/zh/mysql/5.7/reference/server-system-variables.html#sysvar_plugin_dir) 的值是相对路径名，则将其视为相对于 MySQL 基本目录（ [basedir](https://www.docs4dev.com/docs/zh/mysql/5.7/reference/server-system-variables.html#sysvar_basedir) 系统变量的值）。


下面是安装模块的命令：


Linux：

```bash
mysql> install plugin rpl_semi_sync_master soname 'semisync_master.so';
```

Windows：

```bash
mysql> install plugin rpl_semi_sync_master soname 'semisync_master.dll';
```


安装完后：

```bash
mysql> show global variables like '%semi%';
```

```bash
mysql> show global variables like '%semi%';
+-------------------------------------------+------------+
| Variable_name                             | Value      |
+-------------------------------------------+------------+
| rpl_semi_sync_master_enabled              | OFF        | -> 是否启用半同步协议
| rpl_semi_sync_master_timeout              | 10000      | -> 链接 Slave 超时时间
| rpl_semi_sync_master_trace_level          | 32         |
| rpl_semi_sync_master_wait_for_slave_count | 1          |
| rpl_semi_sync_master_wait_no_slave        | ON         |
| rpl_semi_sync_master_wait_point           | AFTER_SYNC | -> MySQL 5.7 之后默认值
+-------------------------------------------+------------+
6 rows in set, 1 warning (0.02 sec)
```


```bash
mysql> set global rpl_semi_sync_master_enabled = 1;
mysql> set global rpl_semi_sync_master_timeout = 2000;
```


安装后启动和定制主从连接错误的超时时间默认是 10s 可改为 2s，一旦有一次超时自动降级为异步。（以上内容要想永久有效需要写到配置文件中）


```bash
[root@localhost ~]# cat /etc/my.cnf
[mysqld]
rpl_semi_sync_master_enabled = 1;
rpl_semi_sync_master_timeout = 2000;
```


## Slave 配置


1）安装半同步模块并启动


Linux：

```bash
mysql> install plugin rpl_semi_sync_slave soname 'semisync_slave.so';
```

WIndows：

```bash
mysql> install plugin rpl_semi_sync_slave soname 'semisync_slave.dll';
```

同样，如果报错就把插件目录移动一下

`lib` -> `bin\lib`


接着执行


```bash
mysql> set global rpl_semi_sync_slave_enabled = 1;
mysql> show global variables like '%semi%';
+-------------------------------------------+------------+
| Variable_name                             | Value      |
+-------------------------------------------+------------+
| rpl_semi_sync_master_enabled              | ON         |
| rpl_semi_sync_master_timeout              | 10000      |
| rpl_semi_sync_master_trace_level          | 32         |
| rpl_semi_sync_master_wait_for_slave_count | 1          |
| rpl_semi_sync_master_wait_no_slave        | ON         |
| rpl_semi_sync_master_wait_point           | AFTER_SYNC |
| rpl_semi_sync_slave_enabled               | ON         |
| rpl_semi_sync_slave_trace_level           | 32         |
+-------------------------------------------+------------+
8 rows in set, 1 warning (0.00 sec)
```


2）从节点需要重新连接主服务器半同步才会生效


```bash
mysql> stop slave io_thread;
mysql> start slave io_thread;
```


PS：如果想卸载异步模块就使用 uninstall 即可。


Master 上查看是否启用了半同步


![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210203214530.png)


现在半同步已经正常工作了，主要看 `Rpl_semi_sync_master_clients` 是否不为 0，`Rpl_semi_sync_master_status` 是否为 ON。如果 `Rpl_semi_sync_master_status` 为 OFF，说明出现了网络延迟或 Slave IO 线程延迟。

## 验证

半同步超时，是否会自动降为异步工作

### Slave

## 关闭半同步;

```bash
mysql> set global rpl_semi_sync_slave_enabled = 0 ;
mysql> stop slave io_thread;
mysql> start slave io_thread;
```

### Master

```bash
mysql> insert into t1(id) values(5),(4);
Query OK, 2 rows affected (10.06 sec)
mysql> insert into t1(id) values(6),(7);
Query OK, 2 rows affected (0.03 sec)
```