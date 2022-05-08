---
title: iOS开发学习-用继承，静态类封装方法
categories: iOS
tags:
 - iOS
updated: '2016-03-30 01:36:02'
date: 2016-03-30 01:32:51
---

iOS开发中，如果不进行适当的封装，使用协议或者继承类来进行开发，你就会遇到传说中的ViewController（以后简称VC） Hell的问题……

比如说，我们先声网App中为了调用接口，做简单的判断，会有如下的垃圾代码（前辈遗留下来的）：

```swift
override func viewDidLoad() {
    super.viewDidLoad()

    var color = UIColor(red: 153/255, green: 204/255, blue: 204/255, alpha: 1)
    self.navigationController?.navigationBar.barTintColor = color

    self.httpController.delegate = self

    Config.shareInstance().isNetworkRunning = CheckNetwork.doesExistenceNetwork()

    if Config.UUID == nil || Config.UUID!.isEmpty
    {
        Tool.showErrorHUD("去信息门户登录一下吧:)")
    }
    else if !Config.shareInstance().isNetworkRunning
    {
        Tool.showErrorHUD("貌似你没有联网哦")
    }
    else
    {
        Tool.showProgressHUD("正在更新校园网信息")
        sendNicAPI()
    }
    // Do any additional setup after loading the view.
}

func sendNicAPI(){
    let nicURL = "http://herald.seu.edu.cn/api/nic"
    let parameter:NSDictionary = ["uuid":Config.UUID!]

    self.httpController.postToURLAF(nicURL, parameter: parameter, tag: "nic")
}

func didReceiveDicResults(results: NSDictionary, tag: String) {
    if let content:NSDictionary = results["content"] as? NSDictionary{
        if tag == "nic"{
            firstSend = false
            Tool.showSuccessHUD("获取信息成功")
            println(content.allKeys)
        }
    }
}
```

看到了吗，每个VC开头都得这样写一发，如果我们有20多个功能呢？会变成什么样子？

![](http://dreampiggy-image.test.upcdn.net/image/9/39/e8ffcb7a5101d2e7acd62c6d72e09.png)

所以，这样下去是绝对不行的，必须对整个乱七八糟的初始化，发送请求，请求接受进行封装，这里就会用到Swift最有用的协议，代理，以及闭包了。

这个首先通过协议和代理，闭包放在下一篇。

协议，顾名思义，也就是其他语言里面的接口（C++的抽象类也差不多） 由于Swift不支持普通类型（Int之流）设置为Static，类方法如果是静态，必须加class关键字（我觉得这个很有槽点），只有Struct和Enum可以直接用Static（也有小Tip可以用Struct包裹一个普通类型，设为计算类型，然后充当一个Static成员，但是这里不讲了）

我们首先可以这样封装简单的初始化方法……

```swift
class func initNavigationAPI(VC:UIViewController,navBarColor:UIColor) -> HttpController?{
    var httpController:HttpController = HttpController()
    VC.navigationController?.navigationBar.barTintColor = navBarColor

    Config.shareInstance().isNetworkRunning = CheckNetwork.doesExistenceNetwork()

    if Config.UUID == nil || Config.UUID!.isEmpty{
        Tool.showSuccessHUD("请在边栏的个人资料中补全您的信息")
    }
    else if !Config.shareInstance().isNetworkRunning{
        Tool.showErrorHUD("貌似你没有联网哦")
    }
    else{
        Tool.showProgressHUD("正在获取信息")
        return httpController
    }
    return nil
}
```

OK？那个HttpController是另一个接口，来进行网络操作的，代理需要靠它，所以我们返回一个HttpController实例，如果失败就返回nil，在实际VC里面加一个解包判断即可。

以后，想要初始化，就只需要这样了

```swift
override func viewDidLoad() {
    super.viewDidLoad()
    var color = UIColor(red: 153/255, green: 204/255, blue: 204/255, alpha: 1)
    self.httpController = Tool.initNavigationAPI(self,navBarColor: color) ?? nil

    if (self.httpController != nil){
        self.httpController!.delegate = self
        Tool.sendAPI("cardDetail", httpController: self.httpController!)
    }
}
```

把一群乱七八糟的代码扔走。下一步就是如果用代理来代理我们所有的请求以及相应的结果了，下一篇文章补上……