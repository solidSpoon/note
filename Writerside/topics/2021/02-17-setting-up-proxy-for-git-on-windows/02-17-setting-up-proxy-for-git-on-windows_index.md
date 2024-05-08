---
slug:  setting-up-proxy-for-git-on-windows
title: Windows 下为 Git 设置代理
authors: [solidSpoon]
tags: []
---

## Windows 下为 Git 设置代理

命令行上 GitHub 真是龟速，偶尔体验一下国内的 Gitee 就感觉爽爆了，还是快给 Git 整个代理吧！
Git 支持两种协议，SSH 和 HTTPS，配置的方式不一样，这两种方式平时都得用，下面分别介绍一下
![image.png](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210217132147.png)
## 设置 SSH 代理
打开用户目录下 `.ssh` 文件夹，也就是 `C:\Users\<用户名>\.ssh`  ，在这个目录下新建一个叫做 config 的文件，注意没有后缀名，内容如下：
```
## -S表示通过 sock5 协议连接
## 按照情况修改代理地址
ProxyCommand connect.exe -S 127.0.0.1:7890 %h %p

Host github.com
  User git
  Port 22
  Hostname github.com
  # 注意将用户名替换成自己的
  IdentityFile "C:\Users\<用户名>\.ssh\id_rsa"
  TCPKeepAlive yes
  IdentitiesOnly yes

Host ssh.github.com
  User git
  Port 443
  Hostname ssh.github.com
  # 注意将用户名替换成自己的
  IdentityFile "C:\Users\<用户名>\.ssh\id_rsa"
  TCPKeepAlive yes
  IdentitiesOnly yes
```
### 检验
```
Cloning into 'solidSpoon.github.io'...
remote: Enumerating objects: 91, done.
remote: Counting objects: 100% (91/91), done.
remote: Compressing objects: 100% (40/40), done.
remote: Total 1532 (delta 40), reused 77 (delta 26), pack-reused 1441 eceiving objects:  99% (1517/1532), 39.85 MiB | 2.16 MiB/
Receiving objects: 100% (1532/1532), 41.13 MiB | 1.78 MiB/s, done.
Resolving deltas: 100% (593/593), done.
```
嗯，有效果
## 设置 http/https 代理
命令如下，也有可能不用加单引号，自己试试吧
```bash
git config --global http.proxy 'socks5://127.0.0.1:7890'
git config --global https.proxy 'socks5://127.0.0.1:7890'
```
### 取消 http/https 代理
```bash
git config --global --unset http.proxy
git config --global --unset https.proxy
```

---

参考资料：

- [https://upupming.site/2019/05/09/git-ssh-socks-proxy/](https://upupming.site/2019/05/09/git-ssh-socks-proxy/)