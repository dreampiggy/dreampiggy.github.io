---
title: Swift之AnyClass与动态类型
categories: iOS
tags:
  - Swift
updated: '2016-03-31 14:14:53'
date: 2016-03-30 02:38:56
---

> 这次写一下关于Swift中AnyClass的使用以及动态类型的实例化和使用场景

# AnyClass与AnyObject
Swift中，任何自定义的对象都是AnyClass的子类，类似于Java的Object类（但注意，这和Objective-C的NSObject不同，后者在Swift中是专门的UIKit或者AppKit框架里面定义的类型，而非语言所规定的类型）

> AnyClass
The protocol to which all class types implicitly conform.

Declaration
`typealias AnyClass = AnyObject.Type`

> AnyObject

protocol AnyObject { ... }
`The protocol to which all classes implicitly conform.`

但是注意，这个AnyObject.Type是一个毕竟是一个接口的Property，所以只能在函数的参数里面使用，如果想要直接获取某个类（而非实例）的类型，使用类名.self即可；如果想要获取一个实例的类的类型，使用.dynamicType；对了，如果对象是NSObject的实例（iOS开发中常用），用classForCoder也是一个选择

```swift
class Test {}
func f(s: AnyClass) {
    print(s)
}
let a = Test.self
let b = Test().dynamicType
let c:AnyClass = "test".classForCoder //注意此时加了classForCoder的方法调用，编译器会推导出""是一个NSString的实例而不是String
f(a) //Test
f(b) //Test
f(c) //NSString
```

# 动态实例化

既然我们有了某个AnyObject的Type，这样我们就可以直接构造出一个类型的实例。使用AnyObject都有的.init方法即可，当然，AnyObject本身并没有init的构造方法……

```swift
let d = a.init() // d is an instance of Test
let e = (c as! NSString.Type).init(stringLiteral: "test") // e is "test"
```

当然，你也许肯定奇怪那个required的init是什么意思，其实这是为了避免出现你使用的这个Test.Type有继承的子类，然后子类的构造函数中使用了这个Type来构造父类这种边界情况出现（考虑的真细……）

> Use an initializer expression to construct an instance of a type from that type’s metatype value. For class instances, the initializer that’s called must be marked with the required keyword or the entire class marked with the final keyword.

而且对于一个Protocol，可以用.Protocol来获取这个Protocol的类型（还是AnyClass），也可以用self来统一处理，因为实际上

`metatype-type(.self) -> type.Type | type.Protocol`

这样的话，有了动态就可以开始干活了

# Swift的反射

或许你也想，既然我有了动态的类型实例，那么是不是能通过类似Java的反射，获取某个类型的所有Property，然后直接访问这个Property呢？答案也是有的，不过在Swift中很少用到，这里用到到了Mirror
> struct Mirror { ... }
Representation of the sub-structure and optional "display style" of any arbitrary subject instance.

```swift
class Person {
    var whatThisProperty:String
    init(name:String) {
        self.whatThisProperty = name
    }
}
let p = Person(name: "Bob")
let mirror = Mirror(reflecting: p)
mirror.children.forEach {
    print("\($0.label!): \($0.value)")
}
```

这里的children返回的是一个AnyForwardCollection，也是可以像数组一般用index来访问或者forEach遍历的，不过索引顺是序按照你的Property的声明顺序

# 最后的小应用

由于我也不怎么会写iOS，有时候遇到这样一种情况：
我提供了一个3D Touch的按钮，四个按钮会对应四种ViewController，iOS提供的API可以获取到用户点击的那个按钮对应的设置的一个字符串值，那么，我可以这样来玩……

```swift
// 3D Touch 传入的字符串来判断返回某个ViewController
static func shortcutToViewController(type:String) -> UIViewController.Type {
    switch type {
    case "pe":
        return RunningViewController.self
    case "curriculum":
        return CurriculumViewController.self
    case "card":
        return SeuCardViewController.self
    case "nic":
        return NicViewController.self
    default: return UIViewController.self
    }
}
```

这是用来判断字符串来产生对应的ViewController.Type的，然后，在真正需要实例化一个新的ViewController来显示一个View的时候，再实例化
```swift
func pushToViewController(vc: UIViewController.Type) {
    // 确保要显示的ViewController不是顶层显示的ViewController
    guard let duplication = navigationController?.topViewController?.isKindOfClass(vc) else { return }
    if duplication { // 检查失败，重复的ViewController，不需要跳转
        return
    }
    let viewController = vc.init(nibName: "\(vc.classForCoder())", bundle: nil) // 初始化ViewController
    navigationController?.pushViewController(viewController, animated: true)
}
```

这也是一种比较奇怪的方式……不过如果不这样，就会导致在主类和上层里面引入过多的Switch Case或者导致代码中出现纯字符串定义的nibName，这对以后重构非常不利。

# 结束语

嗯，好久没写东西了……主要最近在学编译原理，之后会有几篇讲解通过Java实现一个简单的支持CFG的Yacc(误)的东西……顺便就当复习编译原理的Parser部分了。

Swift如今开源了：[Swift](https://github.com/apple/swift)，不到几天Star就超过了2年出头的Rust(lol)……现如今Star也到了2W5的程度

虽然UIKit和AppKit这种宝贵的财富Apple肯定不会开源，不过Swift标准库的实现也是越来越完善，而且也有Linux的版本，很多第三方的库开始加入了对Linux的支持（非面向iOS和OS X开发的，比如对SQLite，Redis的wrapper）

Swift作为一个Rust的对手，一个完全抛弃了C的现代语言，之后在除客户端开发外，更可能的领域就是系统编程和服务端编程了吧。希望能够在在安全性，易用性，效率上达到一个更大的高度，让我们这种开发者也能用的爽，用的顺。