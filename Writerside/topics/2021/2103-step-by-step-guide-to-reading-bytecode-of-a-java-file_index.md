---
slug: step-by-step-guide-to-reading-bytecode-of-a-java-file
title: 手把手教你读一个 Java 文件的字节码
authors: [solidSpoon]
tags: [字节码, Java]
---

想要读懂 Java 的字节码其实没那么难。当然，如果你有汇编语言的经验就会更好上手。本文手把手教你阅读一个简单 Java 文件的字节码。

## 如何得到字节码？

以下面这段示例代码为例，他存放在一个包中：

```java
package demo.a
public class B{
    ...
}
```

通过下面这几个方法就可以查看代码的字节码：

### 方法 1 、命令行

相关命令如下

```java
javac demo/a/B.java // 编译
jvavp -c demo.a.B   // 输出字节码
javap -c -verbose demo.a.B // 详细输出
```

### 方法 2 、idea 插件

下载个插件：「jclasslib Bytecode Viewer」，网址如下


> [https://plugins.jetbrains.com/plugin/9248-jclasslib-bytecode-viewer](https://plugins.jetbrains.com/plugin/9248-jclasslib-bytecode-viewer)



安装该插件后，首先编译代码，然后
菜单 👉 「view」 👉 「Show Bytecode With jclasslib」
结果如下：
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210402103711.png)


## 实验代码

我们使用下面这段代码，你可以将其输入 IDE 中

```java
import java.util.ArrayList;
import java.util.List;

public class Hello {
    public static void main(String[] args) {
        int num1 = 1;
        int num2 = 130;
        int num3 = num1 + num2;
        int num4 = num2 - num1;
        int num5 = num1 * num2;
        int num6 = num2 / num1;

        final int num7 = 5;
        Integer num88 = 6;

        //看装箱指令
        if(num88 == 0){
            System.out.println(num1);
        }

        List<Integer> nums = new ArrayList<>();
        nums.add(1);
        nums.add(2);

        for (int num : nums){
            System.out.println(num);
        }

        if (nums.size() == num2) {
            System.out.println(num2);
        }
    }
}
```

下面是由 idea 反编译得到的代码，可以观察到 `for` 循环被改成了 `while`

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by FernFlower decompiler)
//

import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;

public class Hello {
    public Hello() {
    }

    public static void main(String[] args) {
        int num1 = 1;
        int num2 = 130;
        int var10000 = num1 + num2;
        var10000 = num2 - num1;
        var10000 = num1 * num2;
        var10000 = num2 / num1;
        int num7 = true;
        Integer num88 = 6;
        if (num88 == 0) {
            System.out.println(num1);
        }

        List<Integer> nums = new ArrayList();
        nums.add(1);
        nums.add(2);
        Iterator var10 = nums.iterator();

        while(var10.hasNext()) {
            int num = (Integer)var10.next();
            System.out.println(num);
        }

        if (nums.size() == num2) {
            System.out.println(num2);
        }

    }
}
```

## 阅读字节码

为了方便解释，我将字节码文件拆成小段，首先使用下面这个命令输出字节码

```bash
PS C:\Users\cedar\Desktop\ReadBytecode\code\target\classes> javap -c .\Hello.class
```

一开始就说明了这是「Hello.java」的字节码

```java
Compiled from "Hello.java"
public class Hello {
```

紧接着自动创建了无参构造方法，调用了父类 `Object` 的初始化函数。 `aload_0` 是说把本地变亮表位置 0 的对象加载出来，而这个位置保存的是对自身的引用。


你会发现字节码每条命令前面也有一个数字，比如 `0: aload_0` 前面有一个 `0` ，它代表 `aload_0` 这条指令在第 0 个位置。接着观察 `4: return`，它的位置怎么突然变成 4 了？那是因为 `invokespecial` 这个指令还有两个输入参数，一共占用三个字节

```java
-- 字节码
 public Hello();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return
```

`1: invokespecial #1` 的 `#1`，代表常量池位置 1.常量池通过 `javap -c -verbose demo.a.B` 就可以显示出来，如下所示

```bash
Constant pool:
   #1 = Methodref          #15.#48        // java/lang/Object."<init>":()V
   #2 = Methodref          #12.#49        // java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
   #3 = Methodref          #12.#50        // java/lang/Integer.intValue:()I
   ......
```

接下来就是 `main` 方法了，还记得我们在 `main` 方法中干了什么吗

```java
// 源码
        int num1 = 1;
        int num2 = 130;
        int num3 = num1 + num2;
        int num4 = num2 - num1;
        int num5 = num1 * num2;
        int num6 = num2 / num1;

        final int num7 = 5;
        Integer num88 = 6;
```

它对应的字节码是下面这样的，具体内容我已经标注出来了，稍微解释一下 `iconst_1` ，代表常量 `int 1` ，也就是代码中有个常量 「1」加载到栈顶

```java
  public static void main(java.lang.String[]);
    Code:

    -- 初始化 num1 = 1;保存到变量表 1
       0: iconst_1
       1: istore_1

    -- 初始化 num2 = 130; 保存到 变量表2，以下同理
       2: sipush        130
       5: istore_2

    -- 计算 num3(匿名了) = num1 + num2;
       6: iload_1
       7: iload_2
       8: iadd
       9: istore_3

    -- 计算 num4(匿名了) = num2 - num1;  
      10: iload_2
      11: iload_1
      12: isub
      13: istore        4

    -- 计算 num5(匿名了) = num1 * num2; 
      15: iload_1
      16: iload_2
      17: imul
      18: istore        5

    -- 计算 num6(匿名了) = num2 / num1;
      20: iload_2
      21: iload_1
      22: idiv
      23: istore        6

    -- final int num7 = 5;
      25: iconst_5
      26: istore        7

    -- Integer num88 = 6;
      28: bipush        6
      30: invokestatic  #2                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
      33: astore        8
```

然后是这个 `if` 语句

```java
        if (num88 == 0) {
            System.out.println(num1);
        }
```

注意上文 `num88` 被保存到变量表位置 8，所以此处把位置 8 加载出来

```java
-- 字节码
      35: aload         8
      37: invokevirtual #3                  // Method java/lang/Integer.intValue:()I
      40: ifne          50 -- 如果不等于 0 就跳转到 50
      43: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
      46: iload_1          -- 存储 num1 的地方
      47: invokevirtual #5                  // Method java/io/PrintStream.println:(I)V
```

然后我们操作了一个 `List`

```java
// 源码
        List<Integer> nums = new ArrayList<>();
        nums.add(1);
        nums.add(2);
```

```java
    -- 初始化 List 对象
      50: new           #6                  // class java/util/ArrayList
      53: dup              -- 把栈顶的值复制一份再压回去，此时栈顶有两份一样的值，分别被 54 和 57 指令消耗了
      54: invokespecial #7                  // Method java/util/ArrayList."<init>":()V
      57: astore        9 -- 将初始化的对象存到寄存器 9

    -- list -> add(1);
      59: aload         9
      61: iconst_1
      62: invokestatic  #2                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
      65: invokeinterface #8,  2            // InterfaceMethod java/util/List.add:(Ljava/lang/Object;)Z
      70: pop           -- 丢弃了 add 返回值

    -- list -> add(2)
      71: aload         9
      73: iconst_2
      74: invokestatic  #2                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
      77: invokeinterface #8,  2            // InterfaceMethod java/util/List.add:(Ljava/lang/Object;)Z
      82: pop           -- 丢弃了 add 返回值
```

遍历 `List` ，这里 JVM 把 `for` 改成了 `while`

```java
// 源代码
    for (int num : nums){
        System.out.println(num);
    }

//被 JVM 该成如下代码
    Iterator var10 = nums.iterator();
    while(var11.hasNext()) {
        int num = (Integer)var11.next();
        System.out.println(num);
    }
```

```java
    -- 获取迭代器
      83: aload         9
      85: invokeinterface #9,  1            // InterfaceMethod java/util/List.iterator:()Ljava/util/Iterator;
      90: astore        10

    -- 
      92: aload         10
      94: invokeinterface #10,  1           // InterfaceMethod java/util/Iterator.hasNext:()Z
      99: ifeq          128 -- 如果等于 0，跳转到 128

    -- 获取 next() 并打印
     102: aload         10
     104: invokeinterface #11,  1           // InterfaceMethod java/util/Iterator.next:()Ljava/lang/Object;
     109: checkcast     #12                 // class java/lang/Integer  -- 检查对象是否为给定类型
     112: invokevirtual #3                  // Method java/lang/Integer.intValue:()I
     115: istore        11
     117: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
     120: iload         11
     122: invokevirtual #5                  // Method java/io/PrintStream.println:(I)V
     125: goto          92
```

最后我们写了个 if

```java
// 源码
        if (nums.size() == num2) {
            System.out.println(num2);
        }
```

```java
    -- 如果 list.size() == num2; 打印 num2
     128: aload         9
     130: invokeinterface #13,  1           // InterfaceMethod java/util/List.size:()I
     135: iload_2
     136: if_icmpne     146
     139: getstatic     #4                  // Field java/lang/System.out:Ljava/io/PrintStream;
     142: iload_2
     143: invokevirtual #5                  // Method java/io/PrintStream.println:(I)V
     146: return
}
```

## 小结

Java 的字节码还是要比汇编简单一些。


这里再提一点，当要初始化一个 int 时（在 JVM 中：bool，byte，char，short 都是 int），根据不同的数字所占的位数不同，分别需要如下几个命令，方括号中给出了命令适用的范围

- iconst: [-1, 5]
- bipush: [-128, 127]
- sipush: [-32768, 32767]
- idc: any int value

---

- [https://tech.meituan.com/2019/09/05/java-bytecode-enhancement.html](https://tech.meituan.com/2019/09/05/java-bytecode-enhancement.html)