---
title: Google 搜索技巧 1
date: 2021-02-16 21:00:55
updated: 
tags: 效率
categories: Google 搜索技巧
---

### 几个网址
可以通过搜索关键字来访问这些页面

1. **Google 主页面，大多数搜索的入口**
    - <a href="https://www.google.com" target="blank">https://www.google.com</a>
1. **Google 网上论坛**
    - <a href="https://groups.google.com" target="blank">https://www.google.com</a>
3. **用 Google 搜索图片和图表**
    - <a href="https://images.google.com" target="blank">https://www.google.com</a>
4. **从 Google 搜索视频文件**
    - <a href="https://video.google.com" target="blank">https://www.google.com</a>
6. **高级搜索列表**
    - <a href="http://www.google.com/advanced_search" target="blank">https://www.google.com</a>
7. **Google 搜索设置**
    - <a href="https://www.google.com/preferences" target="blank">https://www.google.com</a>
8. **Google 翻译**
    - <a href="https://translate.google.com" target="blank">https://www.google.com</a>

<!-- more -->
	
### Google 搜索的基本规则

1.  Google 查询不区分大小写
2.  Google 通配符
    - **`*`** 代表一个单词
3.  Google 在一个搜索中会忽略某些通用单词、字母和单个数字
    - *eg*.  where 和 how 在某些时候会被忽略
4.  Google 搜索最多限制 32 个单词（包含搜索项和高级运算符）

### 基础搜索

1. **关键词搜索**
    - *eg*. Hacker
    - *eg*. FBI hacker Mitnick
    - *eg*. Mad hacker dpak
2. **短语搜索**
    - 短语是包含在双引号中的一组的单词，Google 会严格按照你给定的顺序对短语中所有单词进行搜索
        - *eg*. "Google hacker"
        - *eg*. "adult huor"
        - *eg*. "Carolina gets pwnt"
    

### 使用布尔运算符和特殊字符搜索

Google 的布尔运算符包括 `AND`, `OR` 和 `NOT`

1. **`AND` & `+`：强制搜索某个单词**
    - Google 默认会自动搜索你查询的所有元素，所以 `AND` 在 Google 中通常是多余的
    - 由于 Google 会忽略一些常用词，如果你要强制搜索他们，可用 `+`，其后不加空格
        - *eg*. +and just 
2. **`NOT` & `-`：把一个单词排除在搜索之外**
    - 用 `-` 可以缩小搜索范围,其后不加空格
        - *eg*. hacker -jacket
3. **`OR` & `|`：查询只包含其中一个的**
    - *eg*. "evil cybercriminal" OR hacker

### Google URL

- Google URL 就是一个指向搜索结果页面的连接，它是动态的，每次访问的结果可能不同。每个 Google 查询都能表示成一个这样的 url，即地址栏显示的那串字符。
    - *eg*. 搜索 <u>hacking to zhe gate</u>
    
        `https://www.google.com/search?q=hacking+to+zhe+gate&oq=hacking+to+zhe+gate&...` 此处省略部分字符
    - 其中 `www.google.com/search` 是 Google 搜索脚本的位置，`?` 表示紧接着的参数将要被传递到搜索脚本中去
    - 参数之间通过与号 `&` 分离,由变量（Variable）组成，这个变量带着 `=` 以及赋给它的值
    
        `www.google.com/search?variable1=value&variable2=value`
    - 如果搜索中包含特殊字符，它们会被表示成**等价的十六进制编码**
- 你可以直接修改 URL 来搜索你想要的东西或者通过访问 Google 高级搜索界面 `www.google.com/advanced_search` 来设置各种参数

#### 关于 URL 编码
- 通常如果一样东西需要编码，说明这样东西并不适合传输。原因多种多样，如 Size 过大，包含隐私数据，**对于Url来说，之所以要进行编码，是因为Url中有些字符会引起歧义**。
- URL 编码的原则就是使用安全的字符（没有特殊用途或者特殊意义的可打印字符）去表示那些不安全的字符。
    - Url中只允许包含英文字母（a-zA-Z）、数字（0-9）、`-_.~` 4 个特殊字符以及所有保留字符。
- URL 的编码方式非常简单，使用 % 百分号加上两位十六进制字符（因此也称百分号编码），Url编码默认使用的字符集是 US-ASCII 
    - *eg*. `空格` » `%20`

### 总结
Google 简单的外表下提供了很多强大的功能选项，你可以用它来实现强有力的搜索。你可以搜索很多类型的内容，比如 Web 页面、新闻论坛、图片、视频等等。可以用 `-`, `|` 等代替布尔运算符，Google 自动包含一个搜索里的所有搜索项，所以 AND 运算符通常是被忽略的。可以通过访问高级搜索页面填写高级搜索项，可以使用高级搜索来快速缩小搜索范围。随着经验的增多，你就能更快的找到你想要的。