---
title: iOS开发中使用Swift来完成异步事件
categories: iOS
tags:
  - iOS
  - Swift
updated: '2016-03-30 02:44:10'
date: 2016-03-30 01:44:12
---

异步事件编程，其实并不是什么新东西了，基本所有涉及到GUI的，网络请求的，数据库读写的，都会有它的身影。

异步事件，就是说这一个代码或者代码块，并不会阻塞程序的运行，程序会立即执行下一条语句，而这条语句，会在相应的方法调用结束之后，执行它自身的回调函数发送一些信号，来表明这个异步事件完成。就像你约会提前1小时到见面地点，先去买点东西踩点什么的（……），等GF/BF到了之后短信通知你，你就立即回来。而不是一直在原地等到对方过来（……）

最早使用异步开发，是在使用JavaScript来开发Web前端的时候，**XMLHttpRequest**或者**jQuery**的**$.ajax**中，都会用到回调函数，来指明成功或者失败之后的处理方法。当对应的网络请求得到响应之后，会调用响应的成功或者失败的回调函数，然后执行里面相应的方法，这大大提升了前端的效率，不会在网络请求时整个页面卡住，而且也不需要一次次轮询看是否有响应，简化了代码的复杂性。

这点Node.js中更为常见，不过也更能表现中滥用异步事件编程的问题。新人使用Node.js总会发现基本任何东西都是异步的，数据库是异步的，IO文件操作是异步的，Session读写是异步的，甚至获得Request对象都是异步的。这就导致很多人一直在嵌套回调函数，导致了著名的[Callback Hell][1]

在Node.js中，解决方案有非常成熟的[Async][2]，更有号称能用同步思维写异步的[Promises][3]，都是非常棒的解决方案。前者的本质就是一个自动生成回调的封装……，后者则是一个真正意义上的全新的解决方案。

而在Swift和iOS开发中，也有必须用到异步事件编程的地方。除了View层的简单UI和Controller之间的交互以外（这部分一般不需要手写代码处理异步交互或者顺序），其他很多地方需要这些知识。例如网络请求的异步调用，请求队列的处理（虽然可以一个网络请求就是一个线程，但这种方法的效率不高，而且容易导致线程间冲突），SQLite数据库大量数据的读写，本地存储的大量数据读写，复杂UI的渲染顺序等等……这些都是需要进行异步编程的，而不能让同步的代码阻塞住整个应用或者UI。

举个例子，这里是一个UI顺序加载的动画……

```swift
func schoolLifeClicked()
{
    var mydrawerController = self.mm_drawerController //一个用TableView实现的应用侧边栏抽屉View
    let schoolLifeViewController:SchoolLifeViewController = SchoolLifeViewController(nibName: "SchoolLifeViewController", bundle: nil)
    let navSchoolLifeViewController = CommonNavViewController(rootViewController: schoolLifeViewController)

    self.mm_drawerController.toggleDrawerSide(MMDrawerSide.Left, animated: true, completion:{(complete) in
        if complete{//如果成功拉出抽屉
            mydrawerController.setCenterViewController(navSchoolLifeViewController, withCloseAnimation: true, completion: nil)//设置主视图
            mydrawerController.closeDrawerAnimated(true, completion:nil)//关闭抽屉
        }
    })//一个闭包，成功后调用
}
```

可以看到，Swift很多时候也可以依靠回调函数，把一个闭包扔进去当参数，然后执行，从而控制这种异步事件的流程……

但是，这种方法写起来，就会回到和JS那种匿名函数闭包扔进去当参数一样，小范围用还可以，一旦你要进行复杂的流程控制，比如一系列异步事件，AB同时执行，AB同时完成后执行C，C执行完成后执行D……这种控制下写出来的代码和JavaScript的callback hell是一样的，难以维护。

怎么办呢？其实自己实现一个语法糖或者函数队列来执行也不难，不过这里可以推荐一下GitHub上非常厉害的库，大家有兴趣也要认真看看源码（虽然源码是Objective-C的……但是慢慢来） 链接： [Async][4]这个利用了OS X 10.10和iOS8的[GCD][5]技术，只能在这个平台以上 [Async.legacy][6]兼容OS X 10.9和iOS7

怎么使用呢？参考人家的Readme，用语法糖可以很简单的使用：

```swift
Async.userInitiated {
    println("start")
}.main {
    println("1")
}.background {
    println("2")
}.background {
    println("2 all the same")
}.main {
    println("stop")
}
```

由于异步事件的特点，所以整个输出可能就会是

```text
start
1
2
stop
2 all the same
```

不要大惊小怪哦。利用这个就可以从繁重的callback中解放出来，简单的处理异步事件的顺序，并且获得很高的性能，这也是网络请求和数据库访问等必须要考虑的地方……

最后，我还是多看看关于异步事件，闭包的知识，对这些知识有了更深的了解，不仅对iOS开发，对Web开发，客户端开发，并行计算算法的实现等都会十分有帮助。

 [1]: http://www.tuicool.com/articles/Ur2EfmZ
 [2]: https://github.com/caolan/async
 [3]: https://github.com/then/promise
 [4]: https://github.com/duemunk/Async
 [5]: http://en.wikipedia.org/wiki/Grand_Central_Dispatch
 [6]: https://github.com/josephlord/Async.legacy