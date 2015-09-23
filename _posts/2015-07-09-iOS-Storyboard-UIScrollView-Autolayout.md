---
layout: post
title: Storyboard中怎么使用UIScrollView + AutoLayout
date:   2015-07-09 18:21:00
categories: iOS Storyboard AutoLayout
---

我们先用一个 Demo 来一步一步阐述并解决这个问题。

首先新建一个Xcode Single View Application工程，命名为ScrollViewDemo，设备选择iPhone。
![Alt text](http://ww1.sinaimg.cn/large/dd869288jw1ewcbb5jlwoj20kh0bxgmt.jpg)

然后，打开该工程的Storyboard，确保选中AutoLayout和SizeClasses。
![Alt text](http://ww2.sinaimg.cn/large/dd869288jw1ewcbbkznzgj207206smxt.jpg)

为了方便，我们可以点击Storyboard里的ViewController，在属性页面把Size改为iPhone 4 inch。
![Alt text](http://ww1.sinaimg.cn/large/dd869288jw1ewcbc2e8j5j2074055t94.jpg)

假设我们现在要实现一个表单页面，也就是我们需要填写用户名，密码，邮箱等等信息，我们可能会考虑新建很多UITextField，然后添加到这个ViewController的View上，但是，如果信息项目太多，iPhone4的480高的屏幕很可能把这些项目显示完全，解决的方法是使用一个UIScrollView。

下面我们把UIScrollView加到ViewController的View上，并设置好上下左右的约束（这里都是0，也就是全屏的ScrollView）
![Alt text](http://ww4.sinaimg.cn/large/dd869288jw1ewcbdhov57g212g0m87wh.gif)

接下来要把UITextField加到scrollView上面，第一直觉肯定是直接往scrollView上拖UITextField，我们可以试试这样做行不行。

为了能显示scrollView的全貌，我们最好把ViewController的Size设为freeform，然后把对应的高度设为一个值（这里假设是1000）。
![Alt text](http://ww2.sinaimg.cn/large/dd869288jw1ewcbe68wooj207804nq31.jpg)

然后直接往ScrollView上拖入几个TextField，并设置好约束
![Alt text](http://ww4.sinaimg.cn/large/dd869288jw1ewcbim78llg212g0m84qs.gif)

然后就会发现，出错了，这里我们并没有指定ScrollView的contentSize，所以会出现这样的问题：
![Alt text](http://ww4.sinaimg.cn/large/dd869288jw1ewcbjgwgukj209j05tgm0.jpg)

下面说下正确的处理方法：**我们不应该直接在ScrollView上添加实际要用到的View（这里是UITextField），而是需要在UIScrollView上添加一个空白的View（姑且命名为ContentView），然后再将UITextField添加到ContentView上**，下面演示一下整个过程：

先把原来除了scrollView之外的几个UITextField删掉，然后拖入一个UIView，约束是铺满整个scrollView：
![Alt text](http://ww4.sinaimg.cn/large/dd869288jw1ewcbkcffbgg212g0m8qv5.gif)

然后又发现有错了吧，这里我们还要添加另外两个约束，contentView的宽和高，这也是ScrollView和一般的View不同的地方（ScrollView除了frame之外还有ContentSize）。

这里，我们的ScrollView是需要垂直滚动的，所以宽度需要等于设备的宽度。方法是在左侧的View列表里面直接把contentView拖向ViewController的View，然后选择Equal Widths。

而高度就得看实际需求了，如果UITextField较多，就需要设置大一些（相当与代码中设置contentSize），这里我们设为1000。设完之后，终于没有错误了。
![Alt text](http://ww3.sinaimg.cn/large/dd869288jw1ewcbkypwj1g212g0m8k7k.gif)

之后的步骤就比较简单了，把UITextField往contentView上拖，设置好约束，之后运行一下吧，是不是现在已经实现之前的需求了？
![Alt text](http://ww1.sinaimg.cn/large/dd869288jw1ewcblfl5gmg212g0m87n0.gif)