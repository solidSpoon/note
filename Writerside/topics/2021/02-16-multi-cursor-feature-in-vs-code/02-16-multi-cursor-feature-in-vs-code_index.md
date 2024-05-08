---
slug: multi-cursor-feature-in-vs-code
title: VsCode 多光标特性
authors: [solidSpoon]
tags: [效率]
---

# VsCode 多光标特性

使用 VsCode 或者其他编辑器的时候，经常会碰到如下场景:

```
.foo {
  padding: 5;
  margin: 5;
  font-size: 5;
}
```

如何将上面的三个 `5` 改成 `5px` ？答案是创建多个光标，以下给出了创建多光标的几种方法：

## 创建多个光标

### 1. 使用鼠标

1. 先将将光标置于第一个『5』之后

2. 按住键盘上的 『alt 』,然后鼠标点在第二个“5”之后。那么第二个光标就创建好了。

3. 同样的方法创建第三个光标

![1](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20200701163849.gif)

### 2. 使用键盘

1. 将光标置于第一个『5』的那一行
2. 然后按下『Ctrl + Alt + 下方向键』在当前光标下创建新的光标，同样的方法创建的三个光标
3. 按下『Fn + 右方向键』即可切换到行尾（即『End』键）
4. 此时，按下『右方向键』，三个光标都到达了指定位置

注：想要以单词为单位跳转光标，只需按下『Ctrl + 方向键』

![2](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20200701170211.gif)

## 相关快捷键

编辑器中还有很多快捷键可以帮助我们快速地创建多光标

### 1. 『Ctrl + D』

这个命令的作用是，第一次按下时，它会选中光标附近的单词；第二次按下时，它会找到这个单词第二次出现的位置，创建一个新的光标，并且选中它。

这样，我们只需将光标置于 `5` 附近，连续按下三次『Ctrl + D』即可选中所有的 `5` 。此时再按下『右方向键』，输入“px”，即可完成任务。

![3](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20200701173116.gif)

### 2. 『Ctrl + Shift + L』

这个命令会选择所有匹配项

1. 选择一个 `5`
2. 按下『Ctrl + Shift + L』
3. 按下『右方向键』

![5](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20200701174651.gif)

### 3. 『Alt + Shift + I』

1. 选择多行代码
2. 然后按下『Alt + Shift + i』，这样操作的结果是：每一行的最后都会创建一个新的光标。

![4](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20200701173845.gif)