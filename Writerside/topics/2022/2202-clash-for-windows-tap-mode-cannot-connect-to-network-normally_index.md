---
slug: clash-for-windows-tap-mode-cannot-connect-to-network-normally
title: ClashForWindows tap 模式无法正常连接网络
authors: [solidSpoon]
tags: [教程]
---

ClashForWindows 正常情况下只能代理通过 Http 或 Socks 代理工作。这两种协议工作在网络模型中的较高层级，可能无法代理系统全部的流量，比如对 SSH 或 WSL 等不起作用，使用时需要对这些应用单独配置。其实下面这几个选项可以让 ClashForWindows 有能力在 TCP/IP 层级工作，从而代理系统全部流量，具体的教程参见[官方文档](https://docs.cfw.lbyczf.com/contents/tun.html#windows)

![image-20211202230604161](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/2021/12/02/20211202-230605.png)

这里主要提一下通过官方文档操作之后无法正常代理的情况，这种情况 GitHub 的 issue 上已经有了解决方案，[链接](https://github.com/Fndroid/clash_for_windows_pkg/issues/1243)。如果你也连不上网，不妨排查一下网卡的驱动或相关应用

![image-20211202231142928](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/2021/12/02/20211202-231144.png)

使用上述方法代理系统全部流量时，可以关闭 ClashForWindows 的 System Proxy 开关，也会正常工作。

需要提及一下，这种方法虽然可以代理全部系统流量，看起来十分强大，但它的性能不如直接使用 Http 或 Socks 代理，所以还是要看情况使用不同的代理方案。