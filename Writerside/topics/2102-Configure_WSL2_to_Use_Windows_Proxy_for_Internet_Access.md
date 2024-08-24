---
title: 配置 WSL2 使用 Windows 代理上网
date: 2021-02-17 01:41:45
updated: 
tags: 教程
categories: 
---

# WSL 2 配置代理

在 Windows 上设置好代理，连上了谷歌开开心心，但是 WSL 2 不能共享 Windows 的代理策略，如果在 WSL 上再装一个代理软件那可太麻烦了，所以得想想办法。

其实办法还挺简单的，可能有的同学不知道，在一个局域网下如果有一台机器配置好了代理，那么这个代理是可以共享给这个局域网下的其他设备的，比较类似软路由哈！


具体方法如下：
## Windows 端
这里以 Clash 为例，打开 Allow LAN 选项，如下图所示。如果你使用其他软件，那可能是叫「网关模式」、「允许来自局域网的链接」或者其它的什么，都是一个东西，打开就好了，注意打开这个选项后你的电脑就可以代理整个局域网内的机器了，虽然其他的机器还需要额外的配置，但也还是注意安全。

![image.png](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210217014320.png)
对于 Clash 来说，这个选项是一次性的，下次开机它就关了，不过可以在配置文件里改，通常文件的开头就是。如下图，改成 true 就行。
![image.png](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210217014312.png)
开启这个选项后，仔细找找，你会找到一个 IP 地址和一个端口号，IP 其实就是本机 IP 啦，这两个数一会有用。 

![image.png](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210217014306.png)
Clash 这个端口 http 和 socks 通用 


注意如果后文配置后没有效果，那可能是 Windows  防火墙的锅，快去配置防火墙放行 Clash
## WSL 2 端
说是 WSL 2，其实其他的手机电脑都能连上，就在网络设置或者 WiFi 设置那有个配置代理，把上面得到的 IP 和端口填上就行。

下面就说说在 WSL 2 下怎么操作吧！

```bash
## 获取主机 IP
## 主机 IP 保存在 /etc/resolv.conf 中
export hostip=$(cat /etc/resolv.conf |grep -oP '(?<=nameserver\ ).*')
```
> Q: 以上似乎会定位到默认网关 `192.168.3.1`
> A: 切换到 WSL2 

```bash
export https_proxy="http://${hostip}:7890";
export http_proxy="http://${hostip}:7890";
```
注意修改成你的端口


如果是 socket5 协议的话
```bash
export http_proxy="socks5://${hostip}:7890"
export https_proxy="socks5://${hostip}:7890"
```
如果端口一样就可以合并成一句话，http 的同理
```bash
export all_proxy="socks5://${hostip}:7890"
```
使用 `curl` 即可验证是否代理成功，如下有返回值说明成功
```bash
➜  ~cedar curl google.com
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
```
可以将上面命令选择你需要的添加到 `.bashrc` ，这样会让代理一直开启。
使用 zsh 应该保存到  `~/.zshrc`


更新配置
```bash
source ~/.zshrc
```


或者添加如下，需要代理的时候输入 `setss` 即可设置代理，取消代理就 `unsetss` ，或者新开一个窗口。
下面第二条的长命令你好像得根据情况删掉一部分。
```bash
export hostip=$(cat /etc/resolv.conf |grep -oP '(?<=nameserver\ ).*')
alias setss='export https_proxy="http://${hostip}:7890";export http_proxy="http://${hostip}:7890";export all_proxy="socks5://${hostip}:7890";'
alias unsetss='unset all_proxy'
```
如下是我在 `~/.zshrc` 中添加的配置文件
```bash
export hostip=$(cat /etc/resolv.conf |grep -oP '(?<=nameserver\ ).*')
alias setss='export all_proxy="socks5://${hostip}:7890";'
alias unsetss='unset all_proxy'
```
## 验证：
```bash
➜  ~cedar setss
➜  ~cedar curl google.com
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
➜  ~cedar unsetss
➜  ~cedar curl google.com
curl: (28) Failed to connect to google.com port 80: Connection timed out
```
