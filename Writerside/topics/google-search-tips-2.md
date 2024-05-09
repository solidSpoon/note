---
title: Google 搜索技巧 2
date: 2021-02-16 21:03:10
updated: 
tags: 效率
categories: Google 搜索技巧
---

### ﹚运算符语法

为了让查询结果更加准确，我们可以在其中加入高级运算符。其基本形式为 `operator:search_term` ，当你使用高级运算符时，请记住下面的原则：

- 运算符、冒号和搜索项之间没有空格
- search_term 部分可能是一个单词或者是一个带引号的短语
- 可以在高级查找中使用布尔运算符（如：OR，+）和其他特殊字符，但注意不要与 `:` 的功能发生冲突。
- 以 All 开头的运算符通常在每个查询中只能使用一次

<!-- more -->

下面我们来看一些利用高级运算符查询的例子：

- *eg*. `intitle:Google` 将得到标题中包含单词 Google 的页面
- *eg*. `intitle:"index of"` 将得到标题中包含短语 index of 的页面
  - 也可以写成 `intitle:index.of`，因为休止符 `.` 可以充当任何字符
- *eg*. `intitle:"index of" private` 将得到标题中包含短语 index of 和在任意地方出现单词 privae 的页面。
  - 任意地方包括 URL、标题、文本等地
  - `intitle` 只对 index of 有效，不包括 private
- *eg*. `intitle:"index of" "backup files"` 将得到标题中包含短语 index of 和在任意地方出现短语 backup file 的页面
  - 同上

### ﹚语法排错

Google 会提醒我们在使用高级运算符时犯的语法错误

````
小提示： 仅限搜索简体中文结果。您可以在设置中指定搜索语言
````

当然，有时候 Google 也会无能为力

````
找不到和您查询的“allintitle:food inurl:food”相符的内容或信息。

建议：

• 请检查输入字词有无错误。
• 请尝试其他查询词。
• 请改用较常见的字词。
• 请减少查询字词的数量。
````

有时候 Google 没有识别出语法错误，依然会尝试搜索。当你发现搜索结果页中你输入的运算符被以粗体显示，那很可能就是你犯了语法错误，因为 Google 把他们当成了关键词。

### ﹚高级运算符

#### ◈ `intitle` 和 `allintitle`：在页面标题搜索

一个页面的标题大多显示在浏览器的顶部，技术上，它可能是一份 HTML 文档中的 TITLE 标签部分。

**二者区别**：

- `intitle` 遵守前面提到的语法规则
  - `intitle:"index of" "backup files"` 将得到标题中包含短语 index of 和在任意地方出现短语 backup file 的页面
- `allintitle` 的作用范围则包括它后边接的全部字符，不要和其他高级运算符混用

建议使用多个 `intitle` 代替 `allintitle`，且 `intitle` 与其他运算符混合使用效果佳


#### ◈ `allintext` 在页面文本中查找字符串

`allintext` 的作用可以概括为「在一个网页文本中找到一个搜索项」或者「除了在标题、URL 和链接里以外，在其他任何地方找到这个字符串」。

由于其作用范围包括它后边的每一个字符，所以 **`allintext` 运算符最好不要和别的高级运算符混用，其实就最好别用这个运算符了**。

#### ◈ `inurl` 和 `allinurl` 在一个 URL 中查找文本

- 同样：建议使用多个 `inurl` 代替 `allinurl`

##### URL

URL 即统一资源定位器（Uniform Resource Locatir）的缩写，一个 URL 就是一个网页的地址。一个普通的 URL 类似俩面这样：

    https://blog.solidspoon.xyz/54/KaPianLiu/KaPianLiu.html

- 其开头通常是 http、ftp 等协议
  - *eg*. `http://`、`https://`、`ftp://`。
- 协议后面是一个路径名地址
  - *eg*. `blog.solidspoon.xyz/54/KaPianLiu/`
- 最后是一个可选的路径名
  - *eg*. `KaPianLiu.html`

所以上面示例 URL 的含义：`http` 表示这基本是一个 web 服务器，服务器位于 `blog.solidspoon.xyz`，被请求的文件 `KaPianLiu.html` 能在服务器的 `/54/KaPianLiu/` 目录里找到。

#### ◈ `site` 把搜索精确到特定的网站

- 使用 `site` 可以搜索位于一个特定服务器或者一个特定域名里的页面。Google 会从右向左读取 Web 服务器的名字
  - *eg*. `site:google.com` 会找出所有结尾是 google.com 的网站
- 如果你建设了自己的网站而不确定是否被搜索引擎收录的话，就用这条命令吧，如果显示如下信息，那你最好去搜索引擎提交一下你的网站。
- 可以与其他运算符或搜索项一起使用：是

````
找不到和您查询的“site:shanlin257.coding.me”相符的内容或信息。

建议：

• 请检查输入字词有无错误。
• 请尝试其他查询词。
• 请改用较常见的字词。
````

#### ◈ `filetype` 搜索特定类型的文件

- Google 可以搜索很多不同类型的文件，例如 PDF、微软 Office 文档等。`filetype` 能够搜索以特定文件名扩展名结尾的页面。
- 可以与其他运算符和搜索项一起使用：是

*eg*. `filetype:doc pirate` 点击搜索结果会直接下载文件

#### ◈ `link` 搜索一个网页的连接

- `link` 后接一个 URL 或服务器名，它会搜索那些连接到这个 URL 或服务器的网页
  - *eg*. `link:https://mp.weixin.qq.com` 会搜索到很多涉及公众号的网页
- 如果语法错误 *eg*. `link:linux` Google 会将他作为一个带有冒号的短语来搜索，正确形式为 `link:linux.org`
- 可以与其他运算符或搜索项一起使用：否

#### ◈ `inanchor` 在连接描述性文字中寻找文本

*eg*. 在Google 搜索 `inanchor:点击这里` 返回的结果页面本身并不一定包含"点击这里"这四个字，而是这些页面里的链接锚文字中出现了"点击这里"这四个字。

我用 HTML 解释一下：

    <a  href="blog.solidspoon.xyz">点击这里</a>

#### ◈ `cache` 显示页面缓存文本

一个网站必须被搜索引擎爬取过后才会被收录，而搜索引擎在爬取的时候会保存网页快照，即搜索结果页显示的网站摘要。由于互联网上的网站太多了，所以快照不会实时更新，如果你想查看那个版本，就用这个命令把。

*eg*. `cache:blog.solidspoon.xyz` 会返回 Google 上一次爬取我的博客时的快照。

#### ◈ `numrange` 搜索一个数字

- `numrange` 命令可以寻找某一范围内的数字,当这个运算符被不坏好意的 Google 骇客利用时，它强大又危险。
- 可以与其他运算符或搜索项一起使用：是

*eg*. `numrange:13-15` 会返回含有 13、14 或 15 的网页，你也可以把这个运算符简写为 `13..15`

#### ◈ `daterange` 搜索在特定日期范围内被抓取过的页面

- 一个网页每被抓取一次，这个日期就会改变
- `daterange` 接受一个日期范围，必须符合 Julian dates 格式，即一个日期自公元前 4713 年 1 月 1 日起经过的天数。
  - *eg*. `daterange:2452164-2452164 "osma bin laden"` 会返回被 Google 在 2001 年 9 月 11 日索引过并包含短语 Osma Bin Laden 的网页 
- 可以直接在 Google 高级搜索页面 `https://www.google.com/advanced_search` 限制日期
- 可以通过修改 URL 来限制日期
  - *eg*. `https://www.google.com/search?q=apple&as_qdr=m3` 会返回过去三个月内被 Google 抓取过的关于 apple 的网页
- 可以与其他运算符或搜索项一起使用：必须

#### ◈ `info` 显示 Google 的总结信息

实测还是推荐直接搜索网址比较好

- 可以与其他运算符或搜索项一起使用：是

#### ◈ `related` 显示相关站点

- 其后接一个完整的 URL 或主机名
- 与搜索结果页面上的「类似网页」或者高级搜索中的「查找类似网页或相应网页」功能相同。

#### ◈ `stocks` 显示股票信息

- 参数为一个有效股票的缩写
  - *eg*. `stocks:csc`
- 可以与其他运算符或搜索项一起使用：否

#### ◈ `defone` 显示一个术语的定义

-参数可能是一个单词或短语

### ﹚总结

- 除了修改 URL，你还可以使用高级运算符进行查询。高级运算符是 Google 骇客手中强有力的武器。所以你也应该基于这一武器保护你的信息安全。
- 应该避开 all 开头的运算符组合使用
- `intitle`，`inurl`，`link` 分别在标题，URL，网页连接中寻找字符串
- `allintext` 在文档文本中寻找，它看起来最没用
- `filetype` 和 `site` 能够搜索特定的网站或特定的文件类型
- `daterange` 搜索在特定时间框架内被索引过的文件
- `cache`、`info`、`related` 可以搜索 Google 提供的网页快照，信息摘要，相关网站列表
- `stocks` 查询某个股票信息
- `define` 返回一个单词或简单短语的定义