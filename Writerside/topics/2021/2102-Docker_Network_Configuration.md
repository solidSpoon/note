---
slug: docker-network-configuration
title: Docker 网络设置
authors: [solidSpoon]
tags: [Docker]
---

这里列出 Docker 配置网络的常用命令

> - docker network create
> - docker network connect
> - docker network ls
> - docker network rm
> - docker network disconnect
> - docker network inspect



列出所有网络
```bash
docker network ls
```
创建网络
```bash
docker network create <my-net>
```
查看网络信息
```bash
docker network inspect <my-net>
```
查看容器的信息
```bash
docker network inspect <my-docker-name>
```
将一个容器连接到网络
```bash
docker network connect <my-net> <my-docker-name>
```
创建容器时就指定网络，以 busybox 为例
```bash
docker run -it --rm --name busybox1 --network <my-net> busybox sh
```
给容器在指定网络中起一个别名，网络中的容器可以通过别名访问
```bash
docker run --net <my-net> -itd --name <my-docker-name> --net-alias <alias-name> busybox
```
多个容器起一个别名时第一个起的有效，第一个下线后第二个有效


将容器从网络断开
```bash
docker network disconnect <my-net> <my-docker-name>
```
删除创建的网络
```bash
docker network rm <my-net>
```
需要保证该网络没有容器链接

---

参考链接

- https://blog.csdn.net/gezhonglei2007/article/details/51627821
- https://www.cnkirito.moe/docker-network-bridge/