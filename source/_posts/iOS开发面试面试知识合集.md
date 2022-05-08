---
title: iOS开发面试面试知识合集
categories: iOS
tags:
  - iOS
  - Objective-C
  - 面试
updated: '2016-09-26 21:53:48'
date: 2016-09-25 19:27:00
---

> 最近一段时间到南京某公司实习之后，就一直没有更新博客，而校招季节到来，也是投了几家公司，之前不发面经是担心有泄题之类的风险，而且自己也需要调整一下心态。而现在，终于可以谈谈iOS开发的面试了

# 面试技术栈

技术岗位面试，基本上离不开这三方面的东西：

+ 数据结构与算法
+ 语言／框架方面基础知识
+ 通用知识，项目

所以今天专门整理了一些最近的面试中遇到的问题，对自己的能力也正确的评价一下

#数据结构与算法

对于国内的校招来说，《剑指offer》确实是一个特别好的书，同时也有用Web版的[在线答题](http://www.nowcoder.com/ta/coding-interviews?page=1)来做，里面的题基本上面试都会见到，而且并没有超出一般程序员的水平（ACMer请忽略），基本上是人人必备宝典，推荐认真读一遍，[LeetCode](https://leetcode.com/)的话，难度是Easy和Normal的题目尽量做（当然，一些特别复杂比如迷宫搜索，就不必强求了）

常见的题目，基本上分两类：数据结构类（比如什么`反转链表`、`各种遍历`、`二叉搜索树插入删除`、`最大堆实现`）和算法类（`ATOI`、`2sum`、`TopK问题`（很常见）、DP问题如`最长公共子串`等），基本面试的题目不会特别难（要考虑手写出来的时间），保证基础功好一点，临场不断提出新的解题方法（暴力->递归->DP），总会有方法的

数据结构：

+ 链表（反转，合并，找环，排序）
+ 二叉树（检查，遍历，高度，反转）
+ 堆（最大堆，最小堆，TopK问题）
+ 哈希表（再散列法，拉链法，冲突处理，简单Hash算法）
+ 字典树（实现，配合TopK问题）

算法：

+ 排序（插入，选择，冒泡，归并，快排，堆排，基数，计数）
+ 搜索（二分、二叉搜索树、DFS/BFS、图的最小生成树，最短路径（Dijkstra））
+ 递归（树的各种问题，包括迭代求法（stack/queue实现），LCA之类）
+ 贪心（出现的比较少，比如股票问题之类）
+ 动态规划（楼梯、硬币、LCSequence、LCString、回文）

#语言方面

iOS的话，基本上面试只会考虑Objective-C，毕竟是国内公司现状，产品线和项目基本都是纯Obejctive-C写的，短期无法改变。Swift可能会作为加分项目，但其实如果熟悉C或者C++的话加分的概率会更大（实际项目中，多是Objective-C和C结合用的）。对于不知道问题的方向的，可以先参考这篇[《招聘一个靠谱的 iOS》](https://github.com/ChenYilong/iOSInterviewQuestions)，权当自我检测一下

对于Objective-C的语言知识，基本上下面的都问道过，建议应该打好基础，了解清楚：

+ 内存管理（MRC，ARC，assign，weak，strong，copy，unsafe_unretain）
+ Block（循环引用，\_\_block关键字，block类型，block_copy）
+ 多线程（GCD，NSOperationQueue， NSThread），了解复杂同步异步混合逻辑如何处理，线程安全
+ 消息转发机制（objc_msgSend），五个对应消息转发方法的顺序，isa指针，self/super，method swizzling
+ RunLoop（Source，Observer，Timer，NSTimer滚动回调问题），AutoreleasePool在RunLoop作用，source源和UIKit事件
+ Runtime（AssociatedType实现Category属性，消息，swizzling）
+ AutoreleasePool（UI事件会隐式创建AutoreleasePool，循环体内手动创建避免内存峰值），autorelease消息，Pool drain的时机（Runloop一次周期的休眠前）
+ KVC/KVO原理（isa_swizzling），和其他方式对比
+ 属性观察（NSNotification、Delegate、KVO）的对比，线程安全
+ 设计模式，观察者模式，代理模式，工厂模式等
+ 响应者链，HitTest，第一响应者概念
+ ViewController/View生命周期，依次调用方法，手动布局步骤
+ 性能优化、离屏渲染（drawRect和UIBezierPath）
+ UIView，CALayer的关系，动画处理（Core Animation）
+ CoreData使用，SQLite对比优/缺点，managedObject跨线程问题
+ MVC，MVP，MVVM区别

#通用知识，项目

通用知识，一般是本科的课程基础知识，基本上以下对于移动开发都是重要的，因此一定要注意。而且面试也会问到（以HTTP/TCP最多）：

+ HTTP协议具体请求/响应格式，HTTPS原理
+ TCP/UDP区别，TCP/UDP头，TCP三次握手四次挥手，滑动窗口机制
+ 线程和进程的区别，调度方式
+ 操作系统时间片法，多级队列，内存LRU，磁盘SCAN等
+ 数据库范式、索引、数据库锁


项目，基本上是你自己做的一些应用，你需要了解自己做的项目的大体架构，同时对一些组件、复用、网络层的功能实现要知晓，使用的开源库的源代码如果也看过或者了解过最好不过了，比如：

+ 自动布局/手动布局，解释Constrains，手动布局步骤（initWithFrame:，addSubviews:，layoutSubviews:，sizeThatFits:等之类）
+ 动态TableView高度，下拉刷新控件实现，进度条实现
+ SDWebImage源码，NSCache原理简介
+ AFNetworking对RunLoop的利用
+ JSPatch/wxPatch之类的原理简介
+ 异步请求回调，对象线程安全问题
+ 优化问题，圆角图片等

#其他问题

其他问题，一般就是类似HR面的内容，可能会有一些对项目管理的看法，人际交往能力、对待加班态度。注意面试官问到你“你还有什么要问我的吗”一定要主动交谈，可以问一些关于对应事业部的环境、技术氛围、是否可以实习等等问题。但是不必谈薪资或者录用与否，毕竟这主要是最终企业还有多方共同决定的。到这一步的话，面试主要的目的就是一个双方互相了解的过程，所以你也应当主动去问各种问题，去了解公司、团队以及职位等相应内容。

总体来说，基本上原则还是那句话：“真话挑着说，假话绝不说”，保持自信，主动提问，基本上就没问题。如果真到最后却还是失之交臂，一般都是自己前几面表现有些问题，毕竟招聘不是过关制度，最后决定是否录用的时候，前几面的面试官意见都会参考，所以只能说每一面都需要做好准备，尽量表现自己

#学习福利
对于我这种自学iOS开发的，很多基本知识都是后来才补的，在这过程中参考了很多资料，这里就列举出其中最有用的吧：

[GitHub-必看面试问题](https://github.com/ChenYilong/iOSInterviewQuestions)

[objc期刊-各种iOS进阶知识，架构](https://objccn.io/issues/)

[NSHipster-包括KVC/KVO，Swizzling等，非常全面](http://nshipster.com/)

[NSHipster-中文版](http://nshipster.cn/)

[iOS设计模式](https://www.raywenderlich.com/46988/ios-design-patterns)

[iOS多线程](http://www.jianshu.com/p/0b0d9b1f1f19)

[Core Data教程](https://www.raywenderlich.com/115695/getting-started-with-core-data-tutorial)

[ibireme博客-主要是Runtime，RunLoop](http://blog.ibireme.com)

[sunnyxx博客-主要是高级iOS知识和优化](http://blog.sunnyxx.com)

[唐巧的博客-主要是iOS开发工具](http://blog.devtang.com)

[SDWebImage原理](http://www.jianshu.com/p/ba4cbf8dfe49)

[AFNetworking原理](http://www.jianshu.com/p/358dc280fb33)

[JSPatch原理](https://github.com/bang590/JSPatch/wiki/JSPatch-实现原理详解)

#结语
经历了某公司实习，还有一些面试的碰壁，也慢慢认清自己的实力，自己在iOS开发这个方向上自己还有很长很长的路要走，现在也应该继续努力提升自己。在祝愿自己的同时，也希望处于就职路途中的iOS程序猿们能够一起共勉，找到心仪的offer