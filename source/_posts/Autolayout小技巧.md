---
title: Autolayout小技巧
categories: iOS
tags:
  - iOS
updated: '2016-03-30 01:44:01'
date: 2016-03-30 01:43:27
---

iOS开发UI一直是一个问题，当年用代码画UI一度成为流行趋势，相信代码能万能解决问题，而且十分简单。

然而，现在由于iOS设备的不断迭代，市场上常见的iPhone设备就会有：

1.  iPhone4/4S 960*640 (480*320 @2x)

2.  iPhone5/5S 1136*640 (568*320 @2x)

3.  iPhone6 1334\*750(667\*375 @2x)

4.  iPhone6 Plus 1920\*1080(736\*414 @3x)->(2208*1242)

加上iPad以后，还会有一个1024*768(@1x) 和 2048*1536(@2x) 原来想要做一个自适应的，同时支持iPhone和iPad的应用，就算用代码来画UI，也是十分简单的。而现在，在这总共3类，5种，7状态的iOS设备面前，就会有点力不从心了，更别说以后想要做WatchOS的开发就会遇到很多问题，而Autolayout的解决方法的提出大大简化了这一过程

> Autolayout，就是通过一系列的约束条件来控制一个UIView在视图中的位置，同时还要配合Size Classes(兼容iOS8之后的设备)

1、对于一个TableView，我们只需要设置它的Leading、Trailing、Top、Bottom临接到根View即可让它永远全屏显示，无论设备像素。而且，重要的一点，就是在Attribute Inspector中，要把这些距离设置为Standard（或者是0），这样才能在不同设备中获得推荐的显示效果（如果不是Standard或者0的话，就要小心了，这些就是所谓的魔法数字，很可能不同尺寸设备上显示效果会有所差别）

2、对于有事想要使两个View沿着一条中线水平对齐在两侧，这个时候就需要一点点小技巧，比如说，你可以拿一个空的View放在中线上，设置Hidden，宽度为0，然后两边两个View跟这个空View对齐即可。同样的方法也很适合于想要调整两个View的比例，可以选中对应的约束条件，在multiplier中设置比例。

3、有时候实在解决不了，就需要我们使用Size Classes来根据不同Size来设置了。默认的Size Classes是Any Any，对于iPhone来说，除了iPhone 6 Plus的横屏模式，其他情况下都是长宽紧凑的，所以很好设置，对iPhone 6 Plus如果想优化的话，就选择长正常宽紧凑的模式，然后单独设置。iPad由于都是长宽正常的，所以一般单独就做一个Size Classes就好。

4、学会用Storyboard，传统的Xib固然很不错，但是Storyboard也可以作为很好的团队开发助力，不要把所有的视图放在一个Storyboard，可以一个Storyboard一两个视图，把逻辑相关的非常精简的视图放在一个Stoyboard中更能提供开发效率，而总是一个Xib一个Xib关联反而会很凌乱。