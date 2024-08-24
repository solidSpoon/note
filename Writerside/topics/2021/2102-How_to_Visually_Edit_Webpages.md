---
slug: how-to-visually-edit-web-page-elements
title: 怎么直观地给网页「P 图」
authors: [solidSpoon]
tags: [教程]
---

你肯定知道在浏览器中按下 F12 打开开发者工具，在其中修改源代码就可以更改网页上的任意内容。那么有没有办法可以不看源码，通过「所见即所得」的方式直接在页面上修改呢

![演示](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20201215204850.gif)

下面这两个属性可以帮助我们实现这一想法：

1. 使用 `contentEditable` 属性
1. 使用 `designMode` 属性

这两个属性用来帮助开发人员创建网页端的富文本编辑器，因此如果我们将整个网页都应用这两个属性，都可以达到可视化编辑网页的效果。

```javascript
document.body.contentEditable='true';document.designMode='on';
```

`designMode` 是 document 级别的属性

`contentEditable` 是元素级别的属性

当一个 HTML 文档被切换到 designMode 时，我们就可以使用 `document.execCommand` 方法，我们可以通过这个方法对文档中的内容添加**粗体**、*斜体*、缩进、对某段文字添加一个链接等等。

例如，下面的代码就会给选中的文本加上链接

```javascript
document.execCommand("CreateLink",false,"http://www.baidu.com");
```

使用这些方法，我们就可以很容易的用可视化的方式改网页了。当然，你肯定不想在控制台敲这么长的命令，所以找一些可以在浏览器运行脚本的软件来简化这个过程。

这儿有一个简单的示例希望可以帮助到你

https://getquicker.net/sharedaction?code=f7db1234-2681-4c6a-82a4-08d83ad18965

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20201215213916.gif)
