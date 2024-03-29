---
title: 分享一下Java作业－Swing开发的计算器
categories: Java
tags:
  - Java
updated: '2016-03-30 00:37:07'
date: 2016-03-29 23:40:53
---

上周我们Java课程老师要求我们用Swing来开发一个计算器。

说起来图形化的开发，我最早用的应该就是MFC了，当时就只会拖一下控件，然后双击一下写函数……改改属性什么的……

其次接触到了Qt，是在我们大二上实训的时候。那个时候要求我们做一个Qt跨平台的聊天工具，我们最后的效果差不多就是这样，代码放在了GitHub：

https://github.com/lizhuoli1126/SEU-Chat

接下来，由于要加入到学校的一个组织（[先声网](http://herald.seu.edu.cn/index/)），于是又做了一款简单的Android下的天气应用，接触到了Android开发使用的XML来布局的方法。（为什么Qt不允许手动修改XML里面的内容啊啊啊！）

之后，又做起了iOS开发，发现Xcode的图形化真是赞。无论是Storyboard还是Xib，都既能很好的支持图形化空间，又能手动编辑，而且那种按Control拖拽关联代码和图形空间的方式很有意思，也很有效率。于是就按照教程写了一个小的游戏……（之后会有文章说明）

于是，问题来了，我为什么要说前面这么多呢，那是因为：Swing必须得手动写代码布局啊啊啊！（回声～）

最开始用的是Eclipse，按照老师的方法，手写Java代码来进行界面布局，于是，你就得写几行代码跑一下，看看布局，再改再跑………………折腾了几次之后崩溃，想着必须得找解决办法。

百度一下，就发现了Netbeans这款神器，建Java应用－添加JFrame，然后一个熟悉的GUI空间界面就出现了T.T（拖控件其实相当于在Java代码中自动写进去布局代码）终于可以愉快的玩耍了……

![](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-07-14/image/a/3c/3bfdef0a075c8130da1ba2fafd3b4.png)

最终的成品就是这个样子（做得不好别打我，代码也不上传了）：

![](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-07-14/image/0/31/6e31b3c293f2be8bd75da3c977780.png)

说这么多，其实我就是想说Java的Swing并没有提供一个把代码和布局通过灵活方式分开的方法（GUI之所以叫GUI，我认为这是一种交互方式，如果用CLI的交互方式开发GUI程序，这绝对产生不出完美的产品）。如果你们有什么想法或者吐槽的地方，尽情说出来吧！评论就有礼品送。:-P