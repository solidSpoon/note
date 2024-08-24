---
slug: getting-started-with-jshell
title: 初识 JShell
authors: [solidSpoon]
tags: [Java]
---

## 初识 JShell

升级到 Java 11 后，有了 JShell 这个工具（其实 Java 9 就有了），它让 Java 可以像脚本语言一样直接在命令行交互，听起来好神奇，快来体验一下！！

## 启动与退出

保险起见，得先弄明白启动与退出

直接在命令行输入 `jshell` 就启动了

```bash
➜  ~cedar jshell
|  Welcome to JShell -- Version 11.0.9.1
|  For an introduction type: /help intro

jshell>
```

退出方式稍微有一些特别，命令是 `/exit`

```bash
jshell> /exit
|  Goodbye
```

`jshell -h` 可以发现提供了几个选项，这仨比较有意思

```bash
    -q                    Quiet feedback.  Same as: --feedback concise
    -s                    Really quiet feedback.  Same as: --feedback silent
    -v                    Verbose feedback.  Same as: --feedback verbose
```

试了一下 `-s` 非常安静的反馈，看起来真的清爽

```bash
➜  ~cedar jshell -s
-> int a = 1;
-> int b = 2;
```

初学者还是别整这么安静了，使用 `-v` 开启详细反馈吧

```bash
➜  ~cedar jshell -v
|  Welcome to JShell -- Version 11.0.9.1
|  For an introduction type: /help intro

jshell>
```

## 简单使用

### 变量赋值

赋几个值看看

```bash
jshell> int a = 1
a ==> 1
|  created variable a : int

jshell> a + 1
$2 ==> 2
|  created scratch variable $2 : int

jshell> $2 + a
$3 ==> 3
|  created scratch variable $3 : int
```

可见：没有指定变量的数字会自动赋值给临时变量，我们也可以使用这个临时变量

### 方法与类

那创建方法呢？

```bash
jshell> String addMark(Word word) {
   ...> return word.val + "!";
   ...> }
|  created method addMark(Word), however, it cannot be referenced until class Word is declared
```

这里方法传入了一个不存在的类，他告诉我们要定义这个类才能使用这个方法，那定义一下吧

```bash

jshell> class Word {
   ...> String val;
   ...> public Word() {
   ...> val = "hello word";
   ...> }
   ...> }
|  created class Word
|    update replaced method addMark(Word)
```

创建个对象调用一下

```bash

jshell> Word words = new Word()
words ==> Word@2ef1e4fa
|  created variable words : Word

jshell> addMark(words)
$4 ==> "hello word!"
|  created scratch variable $4 : String
```

### 内置命令

输入 `/help` 就能看到所有可以使用的命令，例如列出所有变量

```bash

jshell> /vars
|    Word words = Word@2ef1e4fa
|    String $4 = "hello word!"
```

## 外部编辑器

有没有觉得在命令行定义类或者方法啥的太费事了，其实 JShell 支持使用编辑器

### 使用默认编辑器

先定义一个类

```bash
jshell> class Friend{}
|  已创建 类 Friend
```

调用自带的编辑器

```bash
jshell> /edit Friend
```

如下图，点击 Accept 就行

![img](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210223013703.png)

注意一定是之前定义好的片段，如下：

```bash
jshell> /list

   1 : int a = 1;
   2 : int b = 2;
   3 : int c = 1;
   6 : class Friend{
       String val = "No Friend !!!";
       }
```

否则会报错

```bash
jshell> /edit Dog
|  没有此类片段: Dog
```

### 自定义编辑器

如果想自定义编辑器呢，自带的太不好用

```bash
jshell> /set editor vim
|  编辑器设置为: vim
```

```bash
jshell> /set editor "C:\\Users\\cedar\\AppData\\Local\\Programs\\Microsoft VS Code\\code" -w
|  编辑器设置为: C:\Users\cedar\AppData\Local\Programs\Microsoft VS Code\code -w
```

该 `-w` 选项设置等待文件关闭后再返回

上述设置是一次性的，想永久设置的话，使用 `-retain` 选项

```bash
jshell> /set editor -retain vim
```
