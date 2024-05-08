---
slug:  understanding-blending-modes-in-photoshop-layers”
title: PhotoShop 图层的混合模式是怎么回事
authors: [solidSpoon]
tags: []
---

## 前言
在修图软件中，调整混合模式就可以将两张照片用不同的风格混合在一起

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210223210643.gif)

上图就是将下面这两个图片用不同的混合模式叠加的效果，那么你有没有想过这是什么原理呢？本文就以几个经典的混合模式为例简单研究一下。

<table>
<tr>
<td>
<img src="https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210223210706.png" width="100%"/>
</td>
<td>
<img src="https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210223210720.png" width="100%"/>
</td>
</tr>
</table>




## 图像显示原理
其实各种图层混合模式的灵感就是来源于胶片相机时代。那个时代的摄影师没有先进的计算机来修图，只能拍好胶片后（当然也有其他的感光材料做底片），在暗房通过各种骚操作来给自己的照片添加「特效」，其中很多方法在数字时代就演变成了修图软件中的混合模式。


在修图软件中，图层就是一张张胶片叠在一起，而混合模式就是胶片与胶片之间的药水，不同的药水会让胶片之间产生不同的混合效果。当然要彻底理解修图软件的混合模式就必须得了解 RGB 色彩模型，因为这是混合模式的根基。


### YES!!! RGB!!
有谁能不喜欢 RGB 呢。课本上都讲过，光的三原色是红绿蓝，将这三种颜色按照不同的强度和不同的比例混合之后，就可以得到其他的颜色。

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210223211659.png)

计算机也是这样。在计算机中，红绿蓝的比例可以由一组在 [0 - 255] 之间的数据表示，数字越大对应颜色光强越大。就像下图这样，比如我们想显示纯红色，那就让红色的发光强度最大，绿色和蓝色不发光，因此红色就表示为 `RGB(255, 0, 0)`

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210223211735.png)

如果想显示黄色呢？根据上面的三原色图，只需要让蓝色不发光，红色和绿色发光强度最大，就得到了黄色，

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210223211803.png)

如果让三种颜色第比例相同，那就变成了黑白灰。神奇吧，黑白灰的三原色比例相同，只是发光强度不同。从这个角度看，黑白灰其实是一种颜色，只是亮度不同。所以就不难理解如果想看到真正的「白色」，就必须拼命提高亮度，这也是我们希望显示器亮度更高的原因之一。

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210223211815.png)

当我们把上面的那些颜色块做的非常小，就变成了显示器中像素点。每个都是由红绿蓝三原色组成的，使用程序控制每个像素点的三原色比例，就显示出了不同的图案。下图就是显示器放大的样子。

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210223211825.png)

### 修图就是计算
到这，你一定能理解，修图软件中的所有操作就是对每个像素点的 RGB 值做计算，比如想要提高一张照片的曝光，那就同时提高每个像素的 RGB 值，这样照片就会变得明亮。如果想提高对比度，那就让较亮的地方的 RGB 更大，较暗的地方 RGB 更小，如下图：

<table>
<tr>
<td>
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210223212000.png)
</td>
<td>
![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210223212013.png)
</td>
</tr>
</table>


> 对比度低的照片各像素点的 RGB 值都比较居中，提高对比度后，RGB 值较低（暗）和 RGB 值较高（亮）的像素点变多了。



混合模式也是这个原理，既然是混合，那就得有两个或两个以上的对象。在修图软件中，这样的对象就以图层为载体，我们将两个图层叠放到一起的时候，就会有一个上下对应的关系。上层的一个像素点会对应到下层的一个像素点，混合模式就是对这一组组像素点进行计算。

那么常见的混合模式是怎么计算像素点的呢？
## 计算方法
为了计算两张照片的混合，首先要将个像素点 [0 - 255] 的值映射到 [0.00 - 1.00] 的小数区间，比如：


- 0 = 0.00；
- 128 = 0.50；
- 256 = 1.00



也就是每个像素点的三原色值变成了三个小数，这么处理了之后，就比较好计算，那先来两个简单的练练手
### 「变亮」和「变暗」
这俩操作的公式很简单，分别对比两个像素点的 RGB，变亮就是取大值，变暗就是取小值。


比如有两个像素点：`a [84, 164, 109]`，`b [136, 100, 149]`

- 变亮：`c [136, 164, 149]`
- 变暗：`c [84, 100, 109]`



![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210223212124.png)
### 正片叠底 Multiply
什么是正片呢？


> 正片（英语：Positive Film）为[底片](https://zh.wikipedia.org/wiki/%E5%BA%95%E7%89%87)的分类标准之一，胶片功能类似相纸，利用[负片](https://zh.wikipedia.org/wiki/%E8%B2%A0%E7%89%87)冲印得到正像显影，但不像负片和反转片是摄影胶片；由于以反转冲洗法（Reversal Process）的反转片（Reversal film）亦采正像显影方式[[1]](https://zh.wikipedia.org/wiki/%E6%AD%A3%E7%89%87#cite_note-1)[[2]](https://zh.wikipedia.org/wiki/%E6%AD%A3%E7%89%87#cite_note-2)，“正片”遂成与负片相对的感光材料总称，可供[影片拷贝](https://zh.wikipedia.org/wiki/%E4%B8%AD%E9%96%93%E6%AD%A3%E7%89%87)、[幻灯机](https://zh.wikipedia.org/wiki/%E5%B9%BB%E7%87%88%E6%A9%9F)及灯箱观赏等用途上，也可印制照片、印刷制版。
> [https://zh.wikipedia.org/wiki/%E6%AD%A3%E7%89%87](https://zh.wikipedia.org/wiki/%E6%AD%A3%E7%89%87)



可见，正片上的色彩就是图像真实的色彩

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210223212157.jpeg)

> [图片来源](https://www.google.com/url?sa=i&url=https%3A%2F%2Fzh-cn.facebook.com%2Fhi.xikon%2Fposts%2F279712775851997%2F&psig=AOvVaw1f4r21n5CgNHNQKHowqq5b&ust=1614155052538000&source=images&cd=vfe&ved=2ahUKEwi_8LWlyv_uAhXPCIgKHRkvCTIQjB16BAgAEAg)



那正片叠底就是把两个正片叠上，由于正片亮的地方是透明的，暗的地方是不透明的，叠上之后透明的地方就会显示出另一张正片的图案。

典型示例如下图，上层图层是一个白色背景的水印，下层图层是一个风筝图片，它俩应用正片叠底之后水印的白色背景消失了。

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210223212214.png)

正片叠底的英文是 Mutiply，跟它的名字一样，用公式表示就是：$c=a \times b$


如果刚才那个扣水印的原理你没有看明白，那就用公式解释一下：白色的值是 1，如果 a 是白色，那么混合之后的结果就是 $1 \times b=b$，因此水印白色背景被扣掉了。


如果自己跟自己做正片叠底呢？
那公式就是 $c = a ^ 2$

图像如下，可见整体变暗了一些，亮度低的地方透明度低，变暗的幅度就比较大。

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210223212230.png)

理论上，如果你能在曲线工具中调出一个标准的二次曲线，那它俩效果就是一样的！

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210223212253.jpeg)

### 滤色 Screen
正片叠底是堆叠正片，滤色就是堆叠负片，负片就是正片颜色取反。较暗的场景在负片中变得较亮，较亮就意味着透明，叠上之后透明的地方就会显示出另一张负片的图案。

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210223212317.jpeg)

因此如果我们把水印的背景换成黑色，文字换成白色，它俩做滤色，就会得到白色的水印

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210223212335.png)

公式是这样：$C = 1-\left(1-a\right)\times\left(1-b\right)$
也很好理解：(1 - a) 和 (1 - b) 代表 a 和 b 的负片，它俩做堆叠（乘法），最后再冲洗成正片（1 - x）

自己叠底自己效果如下曲线

$c=1-\left(1-a\right)^2$

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210223212357.png)

可见整体偏亮了，同样可以用曲线工具模拟出来：

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210223212412.jpeg)

### 叠加 Overlay
叠加模式是「正片叠底」和「滤色」的混合模式，是个分段函数。它的公式如下：


$$
{f(a, b)}=\left\{\begin{array}{ll}2 a b, & \text { if } a<0.5 \\ 1-2(1-a)(1-b), & \text { otherwise }\end{array}\right.
$$


其中，a 是**底下的图层**，b 是上面的图层，就是说

- 当下面像素比较暗的时候，效果相当于两倍正片叠底，导致整体变暗
- 当下面像素比较亮的时候，效果相当于两倍滤色，导致整体变亮



注意观察公式：当其中一个像素某个值是 0.5 的时候（a = 0.5 或 b = 0.5），另一个像素对应值相当于没变。如果我们拿一个红绿蓝的亮度都是 0.5 的图片，也就是`RGB(128,128,128)` 和另一个图片做叠加，这个颜色就消失了，再扣个水印试试。

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210223212427.jpeg)

把水印放在**下层**，在上层应用「叠加」效果，`RGB(128,128,128)` 就被扣掉了，我来解释一下：

- 白色水印部分比较亮，应用的是两倍滤色，前面说滤色会保留白色，因此白色被保留了下来
- 黑色水印部分比较暗，应用的是两倍正片叠底，前面说正片叠底会保留黑色，因此黑色被保留了下来



`RGB(128,128,128)` 这个颜色叫做中性灰。也就是叠加模式可以扣掉中性灰，这里引申一点，常见的「中性灰修图法」原理就是上面列出的这几点，有兴趣可以去查一下。


那叠加公式这个「两倍」是干嘛的，你说「叠加」是结合了「正片叠底」和「滤色」，那为什么不直接结合，非要弄个两倍呢？

看一下图像就知道了，由于三维图不好展示，这里还是压缩成二维，也就是自己叠加自己，如果不加两倍直接结合的话会怎么样呢？

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210223212442.png)

两条曲线在 x = 0.5 处根本就是不连续的嘛，这怎么能行呢，所以加上两倍是为了让它俩的图像连续，就下面这样

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210223212500.png)
去掉多余的线条，就是下面这个，是一个增加对比度的曲线：

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210223212527.png)

注意上面这个曲线是自己叠加自己的情况，看起来还是可以用曲线工具模拟出来

![](https://ced-md-picture.oss-cn-beijing.aliyuncs.com/img/20210319193823.jpeg)

## 总结
图层的混合模式分为几种，就像曲线一样，有的应用了之后会让整体变亮，有的会让整体变暗，有的会增加对比度。

- 「正片叠底」就是一种变暗的模式，混合后最暗的黑色会被保留下来，最亮的白色会被丢弃。
- 「滤色」是一种变亮的模式，混合之后最亮的白色会被保留下来，最暗的黑色会被丢弃。
- 「叠加」是一种增加对比度的模式，它是「正片叠底」和「滤色」的结合体，会扣去中性灰。



混合模式还有很多，但只是函数的曲线略有差别，但是他们的函数复杂，因此就不展开了。如果你有兴趣可以去维基百科上看看

> [https://en.wikipedia.org/wiki/Blend_modes](https://en.wikipedia.org/wiki/Blend_modes)