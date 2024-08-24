---
title: Redis 配置主从复制与高可用
date: 2021-02-17 14:16:43
updated: 
tags: 
categories: 
---

## Redis 配置主从复制与高可用

[前文](https://solidspoon.xyz/2021/02/17/Win10DockerRedis%E6%90%AD%E5%BB%BA%E4%B8%BB%E4%BB%8E%E5%A4%8D%E5%88%B6/)搭建了 Redis 的主从复制，主从复制的一大好处就是可以实现高可用，那就试试吧
## 搭建主从复制环境
这里复习一下如何搭建主从复制，启动三个 Docker，第一个当作主节点，剩下的当作从节点
```bash
docker run -dit --name redis1 -p 6381:6379 -p 16381:16381 redis 
docker run -dit --name redis2 -p 6382:6379 -p 16382:16382 redis 
docker run -dit --name redis3 -p 6383:6379 -p 16383:16383 redis 
```
查看 Docker IP
```bash
➜  docker inspect -f '{{.Name}} - {{.NetworkSettings.IPAddress }}' $(docker ps -aq)
/redis3 - 172.17.0.4
/redis2 - 172.17.0.3
/redis1 - 172.17.0.2
```
在两个从分别执行
```bash
127.0.0.1:6379> REPLICAOF 172.17.0.2 6379
OK

127.0.0.1:6379> info replication
## Replication
role:slave
master_host:172.17.0.2
master_port:6379
master_link_status:up
master_last_io_seconds_ago:1
master_sync_in_progress:0
slave_repl_offset:392
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:4cfa8280f14d9085063811cd368b00f393217c23
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:392
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:57
repl_backlog_histlen:336
```
主节点可看到
```bash
127.0.0.1:6379> info replication
## Replication
role:master
connected_slaves:2
slave0:ip=172.17.0.3,port=6379,state=online,offset=574,lag=1
slave1:ip=172.17.0.4,port=6379,state=online,offset=574,lag=1
master_replid:4cfa8280f14d9085063811cd368b00f393217c23
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:574
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:574
```
## 高可用
Redis 的高可用是通过 Sentinel 实现的，它可以做到监控主从节点的在线状态，并做主从切换（基于 raft 协议）。

![Sentin.jpg](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210217142122.jpeg)
这次我们将这三个 Sentinel 分别放在三个 Redis 上

配置 Sentinel 需要一个 sentinel.conf 文件，内容如下：
```
port 16381
daemonize no
pidfile /var/run/redis-sentinel.pid
logfile ""
dir /tmp
sentinel monitor mymaster 172.17.0.2 6379 2
sentinel down-after-milliseconds mymaster 10000
sentinel failover-timeout mymaster 30000
sentinel parallel-syncs mymaster 1
```
然后将这个文件拷贝到 Docker 中并运行，注意文件位置别写错了：


下面是运行第一个节点的命令，剩下两个节点类似，修改一下容器名称就行。
```
docker cp sentinel.conf redis1:/etc/sentinel.conf
docker exec -ti redis1 redis-sentinel /etc/sentinel.conf
```
## 验证
如果想验证的话，登录主节点 Docker 的 bash，可以通过下面方法模拟主节点宕机，然后观察 Sentinel 的反应，或者在剩下两个节点上运行 `info replication` 看看是不是有一个切换成 master 了
```bash
[root@localhost 6379]# ps -ef | grep redis 
root      15382  13771  0 17:34 pts/3    00:00:00 redis-cli
root      15496      1  0 17:38 ?        00:00:00 redis-server 0.0.0.0:6379
root      15588  14748  0 17:43 pts/1    00:00:00 grep --color=auto redis
[root@localhost 6379]# kill -9 15496
```


### 参考链接

- [Docker Official Images](https://hub.docker.com/_/redis?tab=description&page=1&ordering=last_updated)
- [六、Redis 主从复制 Replicaof、哨兵 Sentinel](https://blog.csdn.net/huanghuitan/article/details/108044983)
- [redis cluster集群模式原理](https://juejin.cn/post/6844903984294002701)
- [Windows下部署redis主从、哨兵（sentinel）、集群（cluster）](https://blog.csdn.net/baidu_27627251/article/details/112143714)
- [配置 redis 的主从复制，sentinel 高可用，Cluster 集群](https://github.com/Johar77/JAVA-000/tree/main/Week_12)