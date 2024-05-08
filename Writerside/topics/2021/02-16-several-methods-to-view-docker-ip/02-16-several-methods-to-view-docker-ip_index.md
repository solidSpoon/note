---
slug: several-methods-to-view-docker-ip
title: 查看 Docker IP 的几种方法
authors: [solidSpoon]
tags: [Docker]
---

使用 Docker 时常常需要知道某一容器的 IP，这是个挺烦人的事儿，本文介绍几种查看 Docker IP 的方法

### 在容器内部查看
```bash
cat /etc/hosts
```
会显示自己以及 `–link` 软连接的容器 IP
### 使用 docker inspect 命令
`inspect` 会列出容器详细信息


下面的命令任选其一
```bash
docker inspect --format '{{ .NetworkSettings.IPAddress }}' <container-ID>
```
上面这个我在 Win10-WSL2-Docker 上用不了
```bash
docker inspect <container id>
```
```
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container_name_or_id
```
### 在 ~/.bashrc 中写一个 bash 函数
```bash
function docker_ip() {
    sudo docker inspect --format '{{ .NetworkSettings.IPAddress }}' $1
}
```
`source ~/.bashrc` 然后：
```bash
$ docker_ip <container-ID>
```
### 显示所有容器名称及其 IP 地址
```bash
docker inspect -f '{{.Name}} - {{.NetworkSettings.IPAddress }}' $(docker ps -aq)
```
如果使用 docker-compose 命令将是
```bash
docker inspect -f '{{.Name}} - {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $(docker ps -aq)
```
### 显示所有容器 IP 地址
```
docker inspect --format='{{.Name}} - {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $(docker ps -aq)
```

---

- https://blog.csdn.net/sannerlittle/article/details/77063800

