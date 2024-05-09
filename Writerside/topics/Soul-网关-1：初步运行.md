---
title: Soul 网关 1：初步运行
date: 2021-03-05 19:49:32
updated: 
tags: Soul
categories: Soul 学习笔记
mathjax: false
---

官网：[https://dromara.org/](https://dromara.org/)
GitHub：[https://github.com/dromara/soul](https://github.com/dromara/soul)

## 下载
拉取代码
```bash
git clone git@github.com:dromara/soul.git
```
新建临时分支
```bash
git checkout -b read
```
```bash
mvn clean package install -Dmaven.test.skip=true -Dmaven.javadoc.skip=true -Drat.skip=true -Dcheckstyle.skip=true
```
## 配置
SoulAdmin 需要配置数据源，使用 Docker 启动一个 MySQL。

- username: root
- password: 123456
```bash
docker run --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql:latest
```
然后修改 SoulAdmin  配置文件
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210305195101.png)
**或者**直接启动一个内存数据库 H2，只需去掉下图配置文件的注释部分
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210305195118.png)

## 启动
### soul-admin
运行 soul-admin 模块下的启动类 SoulAdminBootstrap
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210305195131.png)
此时可以浏览器访问 Soul 管理界面


- localhost:9095
- admin
- 123456

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210305195157.png)
### soul-bootstarp
现在还什么也干不了，因为还需要启动 soul-bootstarp，这个模块是 soul 的核心
运行  soul-bootstarp 模块下的启动类 SoulBootstrapApplication
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210305195234.png)
可以看到

```bash
websocket reconnect is successful.....
```
启动成功！