---
title: 接上一话，在Swift中使用代理来中转请求和响应
categories: iOS
tags:
  - iOS
  - Swift
updated: '2016-03-30 01:50:54'
date: 2016-03-30 01:37:07
---

> 这一次主要讲讲代理（Delegate）在iOS开发中的重要意义

上一次说道通过一个类的静态方法来把所有的垃圾初始化代码扔到一起，减少每次创建新的VC所带来的重复劳动问题，这次主要说一下如果通过代理，来使你的代码更为简单，调用一个API： "Could not be simple"

所谓的代理，就是意思被代理的类把自己的方法交给代理人那个类来执行

首先，为了做一个代理，你必须要定义一个协议（称作APIGetter），这里我是把所有的返回结果放在代理，发送请求通过类的静态方法来使用（类名就叫做HeraldAPI吧～）

协议：

```swift
@objc protocol APIGetter{
  func getResult(APIName:String, results:NSDictionary)
  func getError(APIName:String, statusCode:Int)
}
```
    

由于我使用的是AFNetworking的库，可以方便的进行网络通信，AFNetworking就不再介绍了，这里还通过上层封装了两个方法，还有一个对AFNetworking的代理（代理也可以传递的……）：

```swift
var httpController:HttpController//对AFNetworking的代理，暂时不要管他啦
func didReceiveDicResults(results: NSDictionary, tag: String)//返回通过POST调用API的结果
func didReceiveErrorResult(code: Int, tag: String)//返回通过POST请求调用API失败，code为HTTP状态码
```
    

然后，你便可以大展身手，设置你的代理了。首先，你当然需要一个类的成员变量，一般就叫做delegate

```swift
    var delegate:APIGetter?
```

之后，我是通过类的静态方法来发送的（因为和APIGetter的代理无关可以略过……虽然这个sendAPI用了AFNetworking的代理……所谓可以层层封装代理）

而对于接受请求结果，只需要这样：

```swift
func didReceiveDicResults(results: NSDictionary, tag: String) {
    switch tag{
    case "cardDetail":
        self.delegate?.getResult(tag, results: results["content"] as NSDictionary)
    default:break
    }
}

func didReceiveErrorResult(code: Int, tag: String) {
    switch tag{
    case "cardDetail":
        self.delegate?.getError(tag, statusCode: code)
    default:break
    }
}
```

这样，你就已经定义好了一个代理，现在只需要代理人类（就是真正的VC）只要把自己当作代理人（设置自己为delegate就好了）

在你自己的VC中首先确保自己实现了这个协议，比如：

```swift
SeuCardTableViewController: UIViewController, UITableViewDataSource, UITableViewDelegate, APIGetter {

//......

	var API = HeraldAPI()
	override func viewDidLoad() {
		self.API.delegate = self
		//......
	}

//......
}
```

然后，在实现代理方法中，干自己想干的任何事情

```swift
func getResult(APIName: String, results: NSDictionary) {
    println(results)
}

func getError(APIName: String, statusCode: Int) {
    println("Oh no!")
}
```

**你就完成了整个东西……**

你可能觉得很奇怪，自己在VC中的getResult里面的results和APIName是从哪里来的？很简单，你实现了这个协议，这个协议，作为被代理者，它不知道自己什么时候执行。然而，当我们每次通过类的静态方法发送一个请求的时候，AFNetworking会通过代理，然后调用

```swift
func didReceiveDicResults(results: NSDictionary, tag: String) {....}
```

这时候，在里面的

```swift
self.delegate?.getResult(tag, results: results["content"] as NSDictionary)
```

会把直接调用被代理者的方法，然后，由于我们在VC中是一个代理者，自然可以接收到这个代理信息（方法的内容），也就完成了数据传递。 **数据的流向：从AFNetworking，经过我们的类，然后转入APIGetter协议，最后到了VC的方法里面**

其实，主要是由了AFNetworking这个代理在这里，才让过程分析比较复杂（其实这构成了一个含有2个代理的代理链……）

现在，我任何的VC中只需要6行代码就可以搞定之前需要100行的东西，多重构才是王道……代理的作用十分明显，以后除了内部算法实现，千万千万不要写基于过程或者拿靠函数堆积的那种代码了，多用代理，多用协议，继承，才是真正意义上的iOS开发。