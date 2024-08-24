---
title: 教你在 Windows 10 Docker 上搭建 Kafka  集群
date: 2021-02-16 23:43:17
updated: 
tags:
- 教程
- 消息队列
categories: 
---

## 前言
Kafka 是一款比较优秀的消息队列，简单介绍如下：


> Kafka 是一种分布式的，基于发布 / 订阅的消息系统。该消息系统是由 LinkedIn 于 2011 年设计开发，用作 LinkedIn 的活动流（Activity Stream）和运营数据处理管道（Pipeline）的基础。



本文介绍如何在 Windows 10 Docker 上搭建 Kafka 集群

## 运行 ZooKeeper
Kafka 使用 ZK 保存状态


运行如下命令启动 ZK
```bash
docker run -dit --name zk -p 2181:2181 zookeeper
```
注意有时候这个端口回被其他的应用占用，如果占用的话可以换一个


附上重启方法：
```bash
docker restart zk
```
如下命令查看日志，也可以在 Docker Desktop 客户端中查看：
```bash
docker logs -f zk
```
## 运行 Kafka
运行三个 Kafka ，注意替换你的宿主机 IP
```bash
docker run -dit --name kafka0 -p 9092:9092 -e KAFKA_BROKER_ID=0 -e KAFKA_ZOOKEEPER_CONNECT=<你的宿主机 IP>:2181 -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://<你的宿主机 IP>:9092 -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092 -t wurstmeister/kafka
docker run -dit --name kafka1 -p 9093:9093 -e KAFKA_BROKER_ID=1 -e KAFKA_ZOOKEEPER_CONNECT=<你的宿主机 IP>:2181 -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://<你的宿主机 IP>:9093 -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9093 -t wurstmeister/kafka
docker run -dit --name kafka2 -p 9094:9094 -e KAFKA_BROKER_ID=2 -e KAFKA_ZOOKEEPER_CONNECT=<你的宿主机 IP>:2181 -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://<你的宿主机 IP>:9094 -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9094 -t wurstmeister/kafka
```
附上重启方法：
```bash
docker restart kafka0
docker restart kafka1
docker restart kafka2
```
如下命令查看日志，也可以在 Docker Desktop 客户端中查看：
```bash
docker logs -f kafka0
docker logs -f kafka1
docker logs -f kafka2
```
附上删除命令：
```bash
docker rm -f kafka0
docker rm -f kafka1
docker rm -f kafka2
```
## 测试
测试建立 3 Replica 和 5 Partition 的 Topic
```bash
docker exec -ti kafka0 kafka-topics.sh --create --zookeeper <你的宿主机 IP>:2181 --replication-factor 3 --partitions 5 --topic TestTopic
```
查看是否配置成功
```bash
docker exec -ti kafka0 kafka-topics.sh --describe --zookeeper <你的宿主机 IP>:2181 --topic TestTopic
docker exec -ti kafka1 kafka-topics.sh --describe --zookeeper <你的宿主机 IP>:2181 --topic TestTopic
docker exec -ti kafka2 kafka-topics.sh --describe --zookeeper <你的宿主机 IP>:2181 --topic TestTopic
```
运行结果如下：
```bash
➜  ~cedar docker exec -ti kafka0 kafka-topics.sh --describe --zookeeper 172.29.192.247:22181 --topic TestTopic
Topic: TestTopic        PartitionCount: 5       ReplicationFactor: 3    Configs:
        Topic: TestTopic        Partition: 0    Leader: 2       Replicas: 2,1,0 Isr: 2,1,0
        Topic: TestTopic        Partition: 1    Leader: 0       Replicas: 0,2,1 Isr: 0,2,1
        Topic: TestTopic        Partition: 2    Leader: 1       Replicas: 1,0,2 Isr: 1,0,2
        Topic: TestTopic        Partition: 3    Leader: 2       Replicas: 2,0,1 Isr: 2,0,1
        Topic: TestTopic        Partition: 4    Leader: 0       Replicas: 0,1,2 Isr: 0,1,2
➜  ~cedar docker exec -ti kafka1 kafka-topics.sh --describe --zookeeper 172.29.192.247:22181 --topic TestTopic
Topic: TestTopic        PartitionCount: 5       ReplicationFactor: 3    Configs:
        Topic: TestTopic        Partition: 0    Leader: 2       Replicas: 2,1,0 Isr: 2,1,0
        Topic: TestTopic        Partition: 1    Leader: 0       Replicas: 0,2,1 Isr: 0,2,1
        Topic: TestTopic        Partition: 2    Leader: 1       Replicas: 1,0,2 Isr: 1,0,2
        Topic: TestTopic        Partition: 3    Leader: 2       Replicas: 2,0,1 Isr: 2,0,1
        Topic: TestTopic        Partition: 4    Leader: 0       Replicas: 0,1,2 Isr: 0,1,2
➜  ~cedar docker exec -ti kafka2 kafka-topics.sh --describe --zookeeper 172.29.192.247:22181 --topic TestTopic
Topic: TestTopic        PartitionCount: 5       ReplicationFactor: 3    Configs:
        Topic: TestTopic        Partition: 0    Leader: 2       Replicas: 2,1,0 Isr: 2,1,0
        Topic: TestTopic        Partition: 1    Leader: 0       Replicas: 0,2,1 Isr: 0,2,1
        Topic: TestTopic        Partition: 2    Leader: 1       Replicas: 1,0,2 Isr: 1,0,2
        Topic: TestTopic        Partition: 3    Leader: 2       Replicas: 2,0,1 Isr: 2,0,1
        Topic: TestTopic        Partition: 4    Leader: 0       Replicas: 0,1,2 Isr: 0,1,2
```
然后在 kafka1 和 kafka2 上启动消费者，kafka0 生产消息


启动后在 kafka0 的终端随便输入内容，可以看到 kafka1 和 kafka2 的终端接收到了该消息
```bash
docker exec -ti kafka1 kafka-console-consumer.sh --bootstrap-server <你的宿主机 IP>:9093 --topic TestTopic --from-beginning
docker exec -ti kafka2 kafka-console-consumer.sh --bootstrap-server <你的宿主机 IP>:9094 --topic TestTopic --from-beginning
docker exec -ti kafka0 kafka-console-producer.sh --broker-list <你的宿主机 IP>:9092 --topic TestTopic
```
性能测试
```bash
docker exec -ti kafka0 kafka-producer-perf-test.sh --topic TestTopic --num-records 100000 --record-size 1000 --throughput 2000 --producer-props bootstrap.servers=<你的宿主机 IP>:9092
docker exec -ti kafka0 kafka-consumer-perf-test.sh --bootstrap-server <你的宿主机 IP>:9092 --topic TestTopic --fetch-size 1048576 --messages 100000 --threads 1
```


## Cluster Manager for Apache Kafka
这是一个管理 Kafka 的图形界面，下面是它的 GitHub
> [https://github.com/yahoo/CMAK](https://github.com/yahoo/CMAK)



可以使用 Docker 启动该软件，然后访问 [http://localhost:9000/](http://localhost:9000/)，命令如下：
```bash
docker run -dit -p 9000:9000 -e ZK_HOSTS="<你的宿主机 IP>:2181" hlebalbau/kafka-manager:stable
```
按照下图方式添加配置，然后就可以使用了

![image.png](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210216234835.png)

---

参考链接：

- [【Kafka精进系列003】Docker环境下搭建Kafka集群](https://blog.csdn.net/noaman_wgs/article/details/103757791)
- [kafka如何彻底删除topic及数据](https://blog.csdn.net/belalds/article/details/80575751)
- [Kafka集群搭建（使用kafka自带的zookeeper）](https://blog.csdn.net/qq_34834325/article/details/78743490)
