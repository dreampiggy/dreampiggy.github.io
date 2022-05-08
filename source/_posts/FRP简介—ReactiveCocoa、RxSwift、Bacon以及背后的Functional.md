---
title: FRP对比—ReactiveCocoa、RxSwift、Bacon以及背后的Functional
categories: iOS
tags:
 - Functional
 - Swift
 - JavaScript
 - iOS
date: 2016-11-17 14:57:20
---

# ReactiveCocoa和RxSwift

iOS的开发上，Objective-C可以说既是一个巨大的成功，也是一个巨大的限制。Cocoa Touch提供的原生API本身就是目标当年的事件驱动和消息派发的GUI编程模型，并且专门为Objective-C这门类smalltalk的消息式OO语言设计的，更为尴尬的是iOS上没有OS X上自带的[Data Binding](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CocoaBindings/CocoaBindings.html)。种种原因，导致Target-Acion，KVO，Notification，Apple式MVC架构才会一直成为iOS开发的主流。然而，做过开发的都知道，这套架构在大型App，尤其复杂是网络请求和人机交互特别多的情况下，非常容易让整个App架构变得难以维护。

Apple式的MVC，又称为Massive View Controller，会让你整个业务代码和UI绑定代码充斥同一个文件中，并且导致很多人经常会在View中，直接#include一个Modeld的头文件，然后起一个`configureInfo:
`的方法，直接在里面把Model的数据拿来绑定到View的属性上。不信？试试搜一遍你所有的View，把Model的头文件删掉，看看能否编译通过。

```swift
var userCell = tableView.dequeueReusableCellWithIdentifier("identifier") as UserCell
userCell.configureWithUser(user)
```

MVP架构或许是你的救星，不过实际上，MVP只是一个工程化的解决问题，把Massive View Controller变成Massive View Presenter，带来相对明确的架构分层的副作用就是近乎两倍的代码量。而在这种情况下，MVVM的架构就是一个非常大的突破，和MVP一样把View/ViewController扔到一起，但是引入单独的ViewModel，通过View到ViewModel的单向绑定，ViewModel对Model的订阅，既避免了MVC造成的代码混乱，又减少了MVP的造成的重复代码。而实践上，提到MVVM，就得 提到[ReactiveCocoa](https://github.com/ReactiveCocoa/ReactiveCocoa)或者[RxSwift](https://github.com/ReactiveX/RxSwift)，这两者都是FRP的GUI框架实现。

## ReactiveCocoa

> 为了统一术语，ReactiveCocoa中的概念这里都描述成Rx中类似的概念，本质上都是一样的东西

ReactiveCocoa把事件流的接口，定义为`RACStream`。而实际上，通常的事件流实现都是`RACSignal`对象，这个Signal是一个冷事件流（也可以叫做push-driven），即有订阅者订阅后，才会`开始从头依次发送事件`。而对应的冷事件流接口叫做`RACMulticastConnection`，即没有订阅者也会发送事件流。热事件可以通过`publish`和`multicast`转换到热事件流，这对于很多请求，比如WebSocket这种不需要重入的事件流来说很有用。

另外，为了支持Objective-C语言上对泛函性的缺乏，提供了另一个事件流的实现`RACSequence`对象，用来处理集合类型的事件流。

一旦订阅之后，事件流就可以解包，拿到不同状态下的数据，Objective-C的接口就是和Rx类似的三种：
`void (^next)(id) `：拿到事件本身，事件流本身继续流动
`void (^error)(NSError *)`：处理错误事件，error和completed后事件流均结束，两种状态必局其一
`void (^completed)()`：事件流正常结束的处理

```objectivec
[[self.usernameTextField.rac_textSignal
filter:^BOOL(id value){
   NSString*text = value;
   return text.length > 3;
}]
subscribeNext:^(id x){
   NSLog(@"%@", x);
  }];
```

这里，`rac_textSignal`就是一个事件源，而后面的filter，是一个操作符，对事件流的事件变换到真正订阅者关心的数据，最后的`subscribeNext`是一个便捷方法，订阅并生命next状态的处理方式。整个流程模拟的是一个TextFiled的用户输入事件流的走向，用户的所有输入，一旦超过3个文本，就会流动并且打印出来，注意冷事件流是**整个流从头开始的**。

```
hel
hell
hello
```

就如上一篇简介中提到的那样，我们可以不断添加新的操作符，来灵活处理我们的关心的事件流。虽然Objective-C本身没有任何泛函性的接口，但是ReactiveCocoa封装的`RACSequence`本身提供了相当丰富的操作符，包括常见的`map`,`flatmap`,`filter`,`combine`,`switch`等，比如你可以把用户名和密码框的检验事件应用`combineLastest`来确保二者永远同时满足才允许登陆。

```objectivec
RACSignal *signUpActiveSignal =
  [RACSignal combineLatest:@[validUsernameSignal, validPasswordSignal]
                    reduce:^id(NSNumber *usernameValid, NSNumber *passwordValid) {
                      return @([usernameValid boolValue] && [passwordValid boolValue]);
                    }];
```

为了和非Reactive代码和谐相处，ReactiveCocoa提供了一个`RACSubject`类型，是用来处理有副作用的流的，即这个流是可变的。你可以手动创造一个新的流，并不断调用`sendNext:`来手动发送事件给其他订阅者，这就类似了传统的消息事件绑定机制。这个对于一些条件下，比如类似连续加载页面的信号，视图跳转等等有一定的作用，不过对于网络请求等，应当使用`RACSignal`。

```objectivec
[[_button rac_signalForControlEvents:UIControlEventTouchUpInside] subscribeNext:^(id x) {  
    [self.loggerSubject sendNext:@"pop"];  
    [self.navigationController popViewControllerAnimated:YES];  
      
}];  
```

如前面所说，ReactiveCocoa是一个方便打造MVVM架构的框架，提供的`RAC宏`可以方便的进行单向绑定，把事件结果同你的UI对象属性绑定起来，避免了繁琐的代码处理，达到Reactive Programming

```objectivec
RAC(self.passwordTextField, backgroundColor) =
  [validPasswordSignal
    map:^id(NSNumber *passwordValid) {
      return [passwordValid boolValue] ? [UIColor clearColor] : [UIColor yellowColor];
    }];
```

![](https://koenig-media.raywenderlich.com/uploads/2014/01/CombinePipeline.png)


简单来看，ReactiveCocoa真不愧是Cocoa，所有的设计围绕Cocoa的设计模式，提供了方便的宏，并且弱化了泛函概念，提供了很多副作用处理的方式，不像Rx那样纯粹。然而随着Objective-C语言的慢慢淡化，整个项目之后也转为依赖ReactiveSwift的实现。在当前iOS开发的情况下，如果使用Objective-C语言，那么这就是不二的FRP之选。但是如果使用Swift，最好使用正统的RxSwift。

## RxSwift

![](https://pic3.zhimg.com/v2-4b572f2ac5bc28905f043c7999c83c2a_r.jpg)

[ReactiveX](http://reactivex.io/)，也就是Rx，是一个大的语言无关的FRP架构设计，只要你了解了一个语言下的用法，那么就可以达到`learn once, write everywhere`（跑……）

在Rx中，事件流定义为一个`Observable`，而订阅者对应的是`Disposable`接口（RxJava里面对应的就是`Subscriber`），事件流可以通过subscribe来订阅，也是对应了三个状态`onNext`，`onError`，`onCompleted`。

这里，就以RxSwift为准，介绍一下简单的区别。首先，得益于Swift的语法，ReactiveCocoa的Data Binding也变得更简单了，不需要宏包裹和提前声明，看起来更为清晰

```swift
let subscription = primeTextField.rx.text           // Observable<String>
	.map { WolframAlphaIsPrime(Int($0) ?? 0) }      // Observable<Observable<Prime>>
	.concat()                                       // Observable<Prime>
	.filter { $0.isPrime }                          // Observable<Prime>
	.map { "number \($0.n) is prime" }              // Observable<String>
	.bindTo(resultLabel.rx.text)                    // bind to label text
```

另外，Rx中，不同于ReactiveCocoa，事件流本身都是`Observable`，至于是冷还是热，是通过`publish`和`connect`操作得到的，不同于ReactiveCocoa中的`RACSignal`和`RACMulticastConnection`这种分开的设计，导致必须用对应的操作符，在Rx中，所有的操作符都是一致的表现，这点是一个非常大的改进。

同时，Rx的操作符也是最丰富的，什么`lift`，`switch`这种常用的，在ReactiveCocoa中就得自己组合一套。当然，Rx的自定义操作符也很多简单，你只需要一个`T -> Observable<T>`类型的函数来定义

```swift
extension ObservableType {
    func replaceWith<R>(value: R) -> Observable<R> {
        return map { _ in value }
    }
}
```

整体上看，Rx是如今比较有名，并且成套的FRP解决方案，并且迁移到不同平台上的学习成本非常低。ReactiveCocoa本身如今已经分离为Swift版和Objective-C版，并且后者不再继续维护，因此对于混合Objective-C和Swift，或者纯Swift项目，RxSwift是一个构建MVVM和FRP架构的不二选择。

# Promise
为什么这里要提到Promise呢，因为Reactive Programming需要处理的很多，就是对异步请求和频繁事件响应的处理。而Promise是一个比较流行的JavaScript平台异步解决方案。在和FRP的配合上面，可以通过不断的then组合成需要的Promise事件，并且Promise的超集，也就是Future，本身就有搭配不同的Future操作符来达到类似于Rx的组合效果。

不过，Promise的目的，在于对异步请求流程的控制，而本身并没有对事件流的管理。原始的Promise虽然有着类似Rx的事件流类似特点：`不可变性`、`可组合性`，但是关键区别在于Promise自身是单次流动，数据流只会从then开始走到结束或者catch掉，无法多次重新流动；不支持流程中断取消；需要配合其他框架层面的东西，来达到完整事件流和GUI数据绑定，这里就得提到[Bacon](https://github.com/baconjs/bacon.js/)

# Bacon

![](https://baconjs.github.io/logo.png)

Bacon是JavaScript上的一个FRP框架，借鉴于知名的[EventStream](https://github.com/dominictarr/event-stream)所实现的事件流，Bacon在这之上完成了FRP所需要的一切：事件流，变换，数据绑定，比起正统的RxJS来说，提供了更适合Web前端应用的的`EventStream`和`Property`，不需要被RxJS的Hot/Cold Observerable烦扰。并且原生支持了所有惰性求值，在benchmark上比起RxJS有着不错的性能优势。

+ 示例——计数器

```javascript
let plus = $("#plus").asEventStream("click").map(1)
let minus = $("#minus").asEventStream("click").map(-1)
let both = plus.merge(minus)
	.scan(0, add) // add +1 or -1 base on click eventstream
	.onValue(sum => $("#sum").text(sum))
	.onError(e => console.log(e))
	.onEnd(() => alert('total: ' + $("#sum").text));
```

除了专门提供的EventStream和Propery的两种Observable，并且提供了更好的事件源支持，你可以从原生的DOM事件来触发事件源，可以从Promise来触发（这是一个大的优势），甚至从callback或者自定义的binder都可以。在RxJS的基础上有了比较大的提升。不过具体工程上讲两者都是Rx实现的FRP，取舍还要看自己的特定选择（幸好我不做前端）

# Functional

> 由于自己也不是Haskell Guy，仅仅接触过一点点JS、Closure和Swift这些有泛函编程思想的语言 ，如果想具体了解函数式编程中，关于`Functor`、`Applicative`以及`Monad`的知识，推荐花上10分钟看一下简单的图文教程：分别有[原文(推荐)](http://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html)、[Swift版](http://www.mokacoding.com/blog/functor-applicative-monads-in-pictures/)和[JS版](https://medium.com/@tzehsiang/javascript-functor-applicative-monads-in-pictures-b567c6415221#.5upsphilw)


![](http://adit.io/imgs/functors/bind_def.png)

下面这些内容，默认为已经掌握了上述简单理解，如果看不太懂可以回过头重新看一下对应的Functional知识

## ReactiveX

Rx的`Observable`的本质就是一个`Event Monad`，即上下文（就是图文教程中包裹的盒子）为Event的一个Monad，这里的Event定义，可以对应语言的struct或者enum，包括了`next`、`error`和`complete`三个上下文即可。这里截取的是Swift语言的实现，`map`方法实现拆装箱（类似Optional，即Haskell的Maybe）

```swift
public enum Event<Element> {
    /// Next element is produced.
    case next(Element)

    /// Sequence terminated with an error.
    case error(Swift.Error)

    /// Sequence completed successfully.
    case completed
}

extension Event {
    /// Maps sequence elements using transform. If error happens during the transform .error
    /// will be returned as value
    public func map<Result>(_ transform: (Element) throws -> Result) -> Event<Result> {
        do {
            switch self {
            case let .next(element):
                return .next(try transform(element))
            case let .error(error):
                return .error(error)
            case .completed:
                return .completed
            }
        }
        catch let e {
            return .error(e)
        }
    }
}
```


而Rx的`subscribe`方法就是一个解包，也就是`Monad<Event>.map()`，接收一个`(Event) -> void`的参数。或者使用更一般直观的三个参数`onNext: (Element) -> Void`、`onError: (Error) -> Void`、`onCompleted: (Void) -> Void`方法（在其他语言实践上，RxJS就是三个function参数，而RxJava为了支持Java7可以使用匿名内部类）

理论：

```haskell
Monad Event <$> subscribe
```

示例：

```swift
let subscription = Observable<Int>.interval(0.3)
	.subscribe { event in
		print(event) // unwraped event
	}

let cancel = searchWikipedia("me")
	.subscribe(onNext: { results in
		print(results)
	}, onError: { error in
		print(error)
	})

```


Rx的Operator是`Functor`，也就是说`(Event) -> Event`，因此可以通过Monad不断`bind`你想要的组合子，直到最终符合UI控件需要的数据

理论：

```haskell
Monad Event >>= map >>= concat >>= filter >>= map <$> subscribe
```

示例：

```swift
let subscription = primeTextField.rx.text           // Observable<String>
	.map { WolframAlphaIsPrime(Int($0) ?? 0) }      // Observable<Observable<Prime>>
	.concat()                                       // Observable<Prime>
	.filter { $0.isPrime }                          // Observable<Prime>
	.map { $0.intValue }                            // Observable<Int>
```


## Promise / Future
Promise本质上也是一个`Monad`，包裹的上下文就是`resolve`和`reject`。
你可能反驳说`Promise.then(f)`中的`f`，可以是`value => value`，而并不是一个被Promise包裹的类型啊。但是实际上，由于JavaScript类型的动态性，Promise.then中直接返回value类型是个语法糖罢了，实际上会处理为`value => Promise.resolve(value)`

```javascript
Promise.resolve(1)
.then(v => v+1) //便捷写法罢了，返回的是resolved状态的Promise对象
.then(v => Promise.resolve(v+1)) //完整写法
.then(v => Promise.reject('error ' + v)) //想要返回rejected状态，无便捷方法
.catch(e => console.log(e)) // error 3
```

原理：

```haskell
Monad Promise >>= then >>= then >>= catch >>= then
```

示例：

```javascript
Promise.resolve(1)
  .then(v => {
    return v + 1; // 1
  }.then(v =>  {
    throw new Error('error'); //reject
  }.catch(e => {
    console.log(e); // error
    return Promise.resolve(0);
  }.then(v => {
    console.log('end', v); // end 0
  }
```

# 总结
FRP本身发展时间并不长，主要是因为当年的GUI程序的复杂度和需求变化成都，和现如今相比有着明显的差距。传统的事件驱动在构件原型和简单交互的App确实非常简单，但随着架构的发展和业务增多，到了连MVP都无法承担的地步，MVVM的提出和相应的FRP框架就是一个救命稻草。

虽然现如今来说，FRP的主要问题在于入门门槛相对高一点，不过在我看来，这就和当年Web走向Angular和React一样，都是需要一段时间过渡的。在Android平台上，RxJava已经获得了相当大的成功和推广，ReactiveCocoa可能在国内并不如RxJava那样出名，但估计在日后，FRP＋MVVM＋Reactive Native＋Redux这种混合App架构将会得到更大推广和发展，如果Apple或者Google再加一把推手，到那时候才可以说Reactive Programming的时代真正到来了吧。

#参考资料

+ [RxObjc](https://github.com/ReactiveCocoa/ReactiveObjC)
+ [RxSwift](https://github.com/ReactiveCocoa/ReactiveSwift)
+ [ReactiveX](http://reactivex.io/)
+ [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)
+ [Bacon](https://github.com/baconjs/bacon.js/)
+ [Monad](http://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html)
+ [NSHipster](http://nshipster.cn/reactivecocoa/)
+ [ReactiveCocoa Tutorial](https://www.raywenderlich.com/62699/reactivecocoa-tutorial-pt1)
+ [ReactiveCocoa Monad](http://blog.leichunfeng.com/blog/2015/11/08/functor-applicative-and-monad/)