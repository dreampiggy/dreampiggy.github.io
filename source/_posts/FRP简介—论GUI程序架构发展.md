---
title: FRP简介—论GUI程序架构发展
categories: Code
tags:
 - Functional
date: 2016-11-16 22:31:10
---

> 熟悉做端GUI程序（客户端，Web前端）的同学一定会知道，做UI最大的问题就是模型和视图对象的绑定，视图对象的状态管理，以及事件消息的处理。

# 背景

传统的GUI编程的一大核心，就是使用了事件驱动编程模型。UI对象的布局、状态等，通过外部的消息事件（点击，触摸，网络请求响应等等）来触发。这是由于GUI程序的人机交互的天生性质决定的（当然，这里的GUI不包括游戏，游戏一般采用立即的帧驱动而非事件）。对于GUI编程的架构方面，从发现到现在，不外乎这几种：

# 传统的事件监听，消息派发机制：

![](http://www.codeproject.com/KB/java/677591/EventModel.jpg)

这是最常见，也是最贴近GUI程序底层实现的模型。一般来说，GUI程序的框架入口就是一个大的while(true)循环，通过在循环内不断向窗口管理器请求消息（比如点击事件等用户输入），通过把底层的消息回调函数回调或者IPC机制，封装成一个个对开发者友好的事件对象来派发出来。

因此，传统的这种模型，在GUI开发的时候，通过把UI对象绑定指定的事件监听器，在监听器的代码中手动改变状态，来达到人机交互。当然，这一点还不够，因为我们没法手动来触发多个UI对象的关联关系，也很难处理非输入类型事件，比如网络请求，文件读写。因此就需要引入消息的机制，通过派发消息，UI对象可以选择是否处理该消息，或者重新派发消息给其他UI对象，对于网络请求等，既可以用高层的消息处理，也可以手动通过回调函数来处理。这样整套机制就是传统的GUI程序的核心机制。

传统的GUI事件驱动模型，一直伴随着历史的发展，诞生了无数的解决方案和GUI框架，从早期的暴力的函数指针来绑定事件回调，到如今各种面向对象的消息-事件机制。基本你在各种GUI框架中都能找到。不过，遗憾的事，一般的Event都是单个消息或者事件对象，你可以再派发给其他UI对象来处理，但整个流程不是非常响应变化的，假如需要新的消息处理，就得在各处的监听器上手动修改代码，这一点也不Reactive

举例：

+ [iOS Target-Action](https://developer.apple.com/library/content/documentation/General/Conceptual/CocoaEncyclopedia/Target-Action/Target-Action.html)
+ [Android Event](https://developer.android.com/guide/topics/ui/ui-events.html)
+ [Web EventListener](https://developer.mozilla.org/en-US/docs/Web/API/EventListener)
+ [Qt Singal Slot](http://doc.qt.io/qt-4.8/signalsandslots.html)
+ [WPF Events](https://msdn.microsoft.com/en-us/library/ms753115.aspx)

# 数据绑定和状态管理

![](http://i.stack.imgur.com/KZFfe.png)

随着GUI程序的发展，尤其是Web前端领域的发展，传统的事件消息机制也越来越难以方便应对大型GUI应用。随着你的GUI程序交互越来越多样，网络请求越来越多，不同UI对象之间关联越来复杂，一个事件过来，也许你需要修改十多个相关UI对象的属性和布局；同时网络请求的回调，你又得分发封装成多种消息发送出去。最终，你写的UI逻辑代码，将达到：`消息对应事件数 * 事件绑定的UI对象数 * UI对象需要修改的属性数`这样一种地步。这就对GUI开发带来了一个非常大的挑战。

因此，这就带来一个GUI开发的新模式。我们可以重新思考一下，假如我们把消息源，通过一定的策略，直接同UI的属性绑定起来，这就是数据绑定。可以通过建立一套框架封装消息和事件，并自动化事件到UI对象属性这一流程。同时，为了正确修改UI对象的属性，传统的事件消息机制一般会在事件监听器上计算UI对象的当前状态，并手动修改需要修改的属性。因此，数据绑定的时候也需要引入状态管理。在这套框架中，UI对象本身不需要存储状态，需要有一层来处理不同状态对应的UI对象绑定方式，整个Data Flow从数据模型出发，触发状态改变，然后同步到UI对象对应状态下的绑定方式，最终改变UI对象的属性。

当然，从上面的说法也能看出，最简单的实现，至少要达到事件->UI对象的单向绑定，同时也可以存在事件<\->对象的双向绑定。数据绑定常见于使用类XML布局的GUI框架，因为纯XML无法存储状态。比如Vue.js的XML模版，React的JSX，Android的XML布局，WPF的XAML等等。而对于iOS应用而言，除了搭配Storyboard来简化状态，代码布局中一般采用MVVM架构，将View和ViewController这个与View紧耦合的模块放在一起当做View层，其中ViewController专门负责ViewModel的数据绑定到UI对象上，把所有View产生的事件派发回ViewModel（比如按钮的点击，Target为ViewModel），本身不负责任何业务逻辑。而ViewModel就是真正业务逻辑的地方，负责管理View的状态、触发的事件来更新Model，Model更新得到的数据和状态变化则代理给View。不过实践上一般都直接采用ReactiveCocoa了（当然，它数据绑定只是小部分，真正重要地方在FRP上）

举例：

+ [React.js](https://facebook.github.io/react/)
+ [React Native](https://facebook.github.io/react-native/)
+ [Redux](https://github.com/reactjs/redux)
+ [Vue.js](https://vuejs.org/)
+ [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa)
+ [Android Data Binding](https://developer.android.com/topic/libraries/data-binding/index.html)
+ [WPF](https://msdn.microsoft.com/en-us/library/ms752347.aspx)

# FRP——Functional Reactive Programming
![](https://camo.githubusercontent.com/995c301de2f566db10748042a5a67cc5d9ac45d9/687474703a2f2f692e696d6775722e636f6d2f484d47574e4f352e706e67)

数据绑定看起来很直观很美好，但是状态管理却不是。随着UI对象的增加，GUI应用的状态就会再次面临`组件状态数=关联的组件状态数求积`这样一个累乘关系。而且更糟糕的是，因为状态这种东西，不同于具体的UI对象属性，要改变只能重新触发，所以当状态流从数据源开始向下传递的时候，假如某些UI对象想要修改并继续传递，就只能再触发新的状态，这更加重了状态管理的压力。

这时候，函数式编程的思想又发挥了功力。不同于传统事件消息机制的繁琐和复杂，也不需要面对复杂状态时管理，FRP的思想，在于把不定期的事件触发，当做一个事件流，让不同的订阅者来订阅，并绑定事件流的数据到UI对象的属性上。

借由函数式编程的思想，事件流本身是不可修改的，但订阅者可以通过组合无副作用的函数来得到一个属于自己定制的新的事件流，不同订阅者可以重用其他订阅者已经组合过的事件流。事件流的流动方向就是时间轴方向，而订阅者可以组合得到新的事件流的副本，某时刻原事件的状态，该订阅者就能得到该时刻事件对应变化后的状态，用来绑定UI对象。

比如你需要做一个点击监测的功能，需要给一个文本框显示在250ms间隔内连续点击两次以上的次数。如果换做传统事件消息机制，那么就得写两个函数，一个捕获事件，一个计时器，还需要一个全局状态量记录当前这250ms点击的次数。换做数据绑定的方式稍微简化了一点，一个绑定处理函数，但是得引入两个额外状态：当前轮次数增加状态，和切换下一轮的状态。而换做FRP，就如上图所示，把点击事件流，直接通过运算符组合到真正的数据流，绑定到UI对象的即可。

FRP的核心，在于事件流可多次触发，以及各种操作符用来作事件流变换，最终交到订阅者手上的，就是真正UI对象想要的数据流，这样我就可以把这个数据流绑定到UI对象上，达到整个Data Flow的完整性。

举例：

+ [ReactiveX](http://reactivex.io/)
+ [ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa)
+ [EventStream](https://github.com/dominictarr/event-stream)

# SO?
传统的事件驱动将永远是GUI框架的基础，因为最贴近实现层，而且可扩展性强。但是实际开发中，事件消息驱动将导致你的事件监听器遍布各处，也会强行把View层和Model层绑定在一起，并且不利于修改。而数据绑定和FRP的架构能够将GUI程序的UI对象，和数据相对分离开，View不需要管什么事件，只需要自己关系的，为了渲染的属性数据即可。

在现在看来，FRP是在数据绑定的基础上，避免了过重的状态管理，并且能够大大简化代码量，想对容易达到MVVM架构，对于大型应用构建是一个不错的选择。之后的会简单介绍几个FRP框架和比较，同时可以科普一下FRP背后的Functional简单原理。期待今后的MVVM和FRP，在移动和Web平台能够得到更大的推广，解放广大人民生产力。