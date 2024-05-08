---
slug: docker-diagnostic-tool-busybox
title: Docker 诊断神器 BusyBox
authors: [solidSpoon]
tags: [Docker]
---

> BusyBox 是一个集成了一百多个最常用 Linux 命令和工具（如 cat、echo、grep、mount、telnet 、ping、ifconfig 等）的精简工具箱，它只需要几 MB 的大小，很方便进行各种快速验证，被誉为“Linux 系统的瑞士军刀”。



BusyBox 容器镜像可以帮助我们快速测试容器网络


直接运行并进入命令行：
```bash
docker run --name <my-docker-name> -it --rm busybox sh
```
`--rm` 参数可以让我们在退出容器时自动销毁该容器


创建时指定网络
```bash
docker run -it --rm --name <my-docker-name> --network <my-net> busybox sh
```
将一个容器连接到网络
```bash
docker network connect <my-net> <my-docker-name>
```
将容器从网络断开
```bash
docker network disconnect <my-net> <my-docker-name>
```
