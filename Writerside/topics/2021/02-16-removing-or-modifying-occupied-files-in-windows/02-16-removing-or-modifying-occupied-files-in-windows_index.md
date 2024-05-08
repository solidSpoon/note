---
slug: removing-or-modifying-occupied-files-in-windows
title: 删除或修改 Windows 中被占用的文件
authors: [solidSpoon]
tags: []
---

有时候我们想要在 Windows 中修改或删除一个文件，会收到一个「文件正在使用」的消息，而我们又不知道是哪个程序在使用，这时候该怎么办呢？

当某一运行中的进程持有一个资源的句柄时，我们就不能修改该资源。解决办法是结束所有对资源有句柄的进程。我们可以在「资源监视器」中找到持有该句柄的所有进程，资源监视器的打开方式如下

- 方法 1：

\<Win + R\>，运行 `resmon.exe`

- 方法 2：

开始菜单 👉 所有程序 👉 Windiws 管理工具 👉 资源监视器

打开后选中就 CPU ，在「关联的句柄」中输入要释放的文件，即可找到持有该文件按句柄的所有进程，右键结束即可

![image](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210216212626.png)
