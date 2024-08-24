---
title: Soul 网关 2：divide 插件与 Http 代理
date: 2021-03-05 19:57:52
updated: 
tags: 
 - Soul
 - Http
categories: Soul 学习笔记
mathjax: false
---

> divide 插件是进行 http 正向代理的插件，所有 http 类型的请求，都是由该插件进行负载均衡的调用。

## 配置插件
文档：[https://dromara.org/zh/projects/soul/divide-plugin/](https://dromara.org/zh/projects/soul/divide-plugin/)
据文档描述，在 soul-bootsrap 的 `pom.xml` 引入如下依赖
```xml
  <!--if you use http proxy start this-->
   <dependency>
       <groupId>org.dromara</groupId>
       <artifactId>soul-spring-boot-starter-plugin-divide</artifactId>
        <version>${last.version}</version>
   </dependency>

   <dependency>
       <groupId>org.dromara</groupId>
       <artifactId>soul-spring-boot-starter-plugin-httpclient</artifactId>
        <version>${last.version}</version>
   </dependency>
```
然后在 soul-admin 界面开启 divide 插件
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210305195835.png)

## 运行示例代码
示例代码在下面这个位置
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210305195846.png)
如果运行不了就在项目设置中导入这个模块
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210305195903.png)
看一下配置文件，该服务运行在 `localhost:8188`
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210305195939.png)
整理一下思路

- soul-admin 端口 9095
- soul-bootstrap 端口 9195
- examples 下的 http 服务端口 8188



用 PostMan 构建一个请求，发往 8188 测试一下
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210305195959.png)

- [http://localhost:8188/order/findById?id=1](http://localhost:8188/order/findById?id=1)

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210305200018.png)
现在将请求发往网关试一下，根据 soul-admin 提示，应该构建如下请求
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210305200036.png)

- [http://localhost:9195/http/order/findById?id=1](http://localhost:9195/http/order/findById?id=1)

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210305200054.png)
可见网关成功的转发了我们的请求

## 负载均衡
测试一下负载均衡
修改一下配置，允许并行
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210305200114.png)
端口改成 8189
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210305200131.png)
在 soul-admin divide 界面点击修改
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210305200150.png)
发现有两个配置，权重都是 50
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210305200208.png)
用 PostMan 发送几次请求，观察日志发现请求均匀的打到了两个地址
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210305200221.png)

## 过滤
通过 waf 界面可以对流量进行过滤
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210305200234.png)
这个界面的选择器会对流量进行初始匹配，匹配通过后会由选择器规则列表进行最终匹配

在选择器列表和选择器规则列表都添加下面的规则，设置返回状态码 504
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210328110046.png)
使用 PostMan 测试，成功拦截
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210305200306.png)