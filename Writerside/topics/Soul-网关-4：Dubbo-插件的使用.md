---
title: Soul 网关 4：Dubbo 插件的使用
date: 2021-03-10 13:04:05
updated: 2021-03-10 13:46:29
tags: 
 - Soul
 - Dubbo
categories: Soul 学习笔记
mathjax: false
---

文档地址：
> [https://dromara.org/zh/projects/soul/dubbo-plugin/](https://dromara.org/zh/projects/soul/dubbo-plugin/)

## ZooKeeper

本次实例需要 ZK，这里提供 Docker 和下载压缩文件两种方式。

### Docker

```bash
docker run -dit --name zk -p 2181:2181 zookeeper
```

### 安装

下载地址
> [https://zookeeper.apache.org/releases.html](https://zookeeper.apache.org/releases.html)

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210310130616.png)
撰写本文时 ZooKepper 版本为 3.6.2，下载后解压。

将 conf 文件夹下的 zoo_sample.cfg 文件复制一份，命名为 zoo.cfg。
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210310130636.png)
新建用于存放 data 和 dataLog 的目录，然后编辑 zoo.cfg 文件，指定 dataDir 和 dataLogDir。
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210310130652.png)
运行 bin 目录下 zkServer.cmd 启动 ZooKeeper。
![image.png](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210310130717.png)
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210310130729.png)

## 运行 SouAdminBootstrap
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210310130742.png)
启动 soul-admin 之后，进入管理页面，关闭 divide 插件，打开 dubbo 插件。
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210310130801.png)
如果在这个界面找不到 dubbo 插件，那可能被隐藏了，直接搜索就会出来
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210310130814.png)

## 运行 soul-bootstarp
根据文档
> [https://dromara.org/zh/projects/soul/dubbo-proxy/](https://dromara.org/zh/projects/soul/dubbo-proxy/)

修改soul-bootstrap 的 pom.xml 。Dubbo 插件的依赖在 `<!-- soul  apache dubbo plugin start-->` 和 `<!-- soul  apache dubbo plugin end-->` 之间。
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210310130845.png)
![image.png](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210310130858.png)
本次实需要将该部分下面这几行取消注释

```xml
        <!--soul  apache dubbo plugin start-->
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-plugin-apache-dubbo</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo</artifactId>
            <version>2.7.5</version>
        </dependency>
```
```xml
        <!-- Dubbo zookeeper registry dependency start -->
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-client</artifactId>
            <version>4.0.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-framework</artifactId>
            <version>4.0.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.curator</groupId>
            <artifactId>curator-recipes</artifactId>
            <version>4.0.1</version>
        </dependency>
```
将之前的 Http 插件注释掉
```xml
        <!--if you use http proxy start this-->
<!--        <dependency>-->
<!--            <groupId>org.dromara</groupId>-->
<!--            <artifactId>soul-spring-boot-starter-plugin-divide</artifactId>-->
<!--            <version>${project.version}</version>-->
<!--        </dependency>-->
<!--        <dependency>-->
<!--            <groupId>org.dromara</groupId>-->
<!--            <artifactId>soul-spring-boot-starter-plugin-httpclient</artifactId>-->
<!--            <version>${last.version}</version>-->
<!--        </dependency>-->

        <!--if you use http proxy end this-->
```
运行 soulBootstrapApplication
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210310130919.png)

## 运行 soul-examples-apache-dubbo-service
这是本次演示的示例程序
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210310130932.png)

## 检验
打开 soul-admin 的 dubbo 界面，发现已经注册了好多路径，随便找一个测试一下
![image.png](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210310130946.png)
如下所示：代理成功！
![image.png](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210310130957.png)