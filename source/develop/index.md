---
title: 开发
updated: '2016-06-02 02:27:27'
date: 2016-05-30 18:46:16
type: "develop"
comments: false
---

> 这里提供了一些iOS开发相关知识、还有各种好玩的
代码小片段、开发软件、各种脚本、各种实用网站等等


# iOS开发
+ Objective-C
  + [Runtime](http://blog.devtang.com/2013/10/15/objective-c-object-model/)
  + [Runloop & AutoreleasePool](http://blog.ibireme.com/2015/05/18/runloop/)
  + [Blocks](http://tanqisen.github.io/blog/2013/04/19/gcd-block-cycle-retain/)
  + [Associated Objects](http://nshipster.com/associated-objects/)
  + [KVO](http://nshipster.cn/key-value-observing/)
  + [Method Swizzling](http://nshipster.com/method-swizzling/)
+ Swift
  + [Reflect & MirrorType](http://swifter.tips/reflect/)
  + [Functional](http://blog.callmewhy.com/2014/09/11/functional-swift-apis/)
+ Develop
  + [知识点](http://hit-alibaba.github.io/interview/index.html)
  + [面试问题](https://github.com/ChenYilong/iOSInterviewQuestions)
  + [性能优化](http://www.reviewcode.cn/article.html?reviewId=7)
  + [防逆向](http://tanqisen.github.io/blog/2014/06/06/how-to-prevent-app-crack/)
  + [自定义URL Scheme](http://objcio.com/blog/2014/05/21/the-complete-tutorial-on-ios-slash-iphone-custom-url-schemes/)


# 代码片段
+ [deep-copy-js](https://gist.github.com/lizhuoli1126/daff71c295dafc6e7b47)


# 实用网站
+ [visualgo](http://visualgo.net) 各种算法图形化解释和伪代码执行过程，学习、复习必备
+ [shields.io](http://shields.io)
生成各种GitHub上README.mk上常见的构建，版本，作者，Licence图标
+ [regextester](http://www.regextester.com) 比较全面好用的正则表达式测试网站
+ [oschina在线工具集](http://tool.oschina.net) 各种Web、移动、测试、对照表等工具集合
+ [codelf](http://unbug.github.io/codelf/) 不知道怎么命名变量、函数、类？不知道英文名怎么写？参考这个，输入中英文，会从GitHub各开源项目中寻找相关命名，相当好用
+ [NSHipster](http://nshipster.com) [中文](http://nshipster.cn) iOS开发，Cocoa，Swift，Objective-C技巧合集


# OS X 必备软件

### GUI
+ [Homebrew](http://brew.sh)
最好的OS X包管理工具，brew install解万难，不需要root权限
+ [ShadowSocks](https://github.com/shadowsocks)
必备代理工具……
+ [aria2](http://brewformulas.org/Aria2) + [百度云](https://github.com/acgotaku/BaiduExporter) + [迅雷离线](https://github.com/binux/ThunderLixianExporter)
超级好用下载工具，OS X上配合插件满速百度云、迅雷，不用担心各种管家。有[GUI封装版](https://github.com/yangshun1029/aria2gui)
+ [Sublime](https://www.sublimetext.com) 大家都知道的轻巧快速的编辑器，当然还得有[Package Control](https://packagecontrol.io)，可以命令行启动
+ [Atom](https://atom.io) 不说了吧，Hackable Editor，各种插件，除了速度和内存外完爆Sublime（其实这就是Sublime的优势……）
+ [iTerm](https://www.iterm2.com) OS X必备的Shell模拟器，完爆自带的Terminal
+ [Go2Shell](http://zipzapmac.com/go2shell) Finder上一键在当前路径下打开终端
+ [Dash](https://kapeli.com/dash) 最好的API DocSet，作为开发者必备的一类工具，快速搜索各种API，类，函数，参数介绍等，无论何时何平台都得有一个以便随时查阅（总不能每次Google或者上Apple Developer、MSDN吧……）
+ [SourceTree](https://www.sourcetreeapp.com) 超好用Git客户端，比起GitHub Desktop更好用，所有命令都可以图形化实现(比如很好用的rebase)
+ [Paw](https://luckymarmot.com/zh-hans/paw) 原生HTTP API开发工具，Web开发必备，免费的话也可以选择[Postman](https://chrome.google.com/webstore/detail/postman/fhbjgbiflinjbdggehcddcbncdddomop)
+ [WWDC Desktop](https://github.com/insidegui/WWDC) 一起搞iOS/OS X开发吧，各种发布会、教程录像合集
+ [StarUML](http://staruml.io) 画UML的超好用工具，和(ProcessOn)[]差不多，更直观方便
+ [MAMP](https://www.mamp.info) + [Postgres](http://postgresapp.com) 开发PHP/MySQL/PostgreSQL/MongoDB等各种Web和数据库图形化管理工具 
+ [bitbar](https://github.com/matryer/bitbar)
把各种小脚本放到通知栏上，配合下面很多CLI有奇效（原来曾经有个[TodayScript](https://github.com/SamRothCA/Today-Scripts)不过在10.11似乎有问题）
+ [Tickeys](http://www.yingdev.com/projects/tickeys)
让你享受机械键盘的快感，个人喜欢"打字机"的声音
+ [AirServer](https://www.airserver.com) 把iOS设备屏幕投射到电脑上，开发iOS应用或者玩耍都是相当好用，价格还算能接受
+ [SmoothScroll](https://www.smoothscroll.net) OS X对非Apple的鼠标很不友好，忍受不了反向滚动以及无平滑滚动的人终于有救了……然而这软件价格惊人(138/Year)……有钱的支持吧
+ [f.lux](https://justgetflux.com) 护眼专用，自动改变色温，防止青光眼
+ [Charles](http://www.charlesproxy.com) 高级代理调试抓包工具，HTTP(S)，Map local等等，可以配合下载一些缓慢资源，还有iOS 模拟器的代理抓包等，[参考Charles从入门到精通...](http://blog.devtang.com/2015/11/14/charles-introduction/)
+ [Genymotion](https://www.genymotion.com) 超快的Android模拟器，调试工具，比起Android Studio自带的好很多……如果是Android开发者的话很推荐，当然，也能用来在电脑玩Android游戏（其实还有一个Chrome跑Android的[Chrome-ARChon](https://github.com/vladikoff/chromeos-apk)也行）
+ [CrossOver](https://www.codeweavers.com/products) 图形化封装版Wine……对于OS X这种非主流平台，跑Windows软件（尤其是游戏）是一个无奈的现实（其实一般的话放个WinRar，FlashFXP，VC6.0，Microsoft Access什么的就差不多了……）虽然其实用这个主要是跑很多游戏的（Steam、GOG大法好）
+ [awesome-osx](https://github.com/iCHAIT/awesome-osx) 里面有各种类型的软件，工具集合（主要面向开发者），时不时可以看看


### CLI
+ [oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh)
不得不用的Shell，完爆Bash，必备……
+ [vim-spf13](http://vim.spf13.com)
超爽的Vim一件配置工具……适用于各类开发者(C系,Shell系,Go,Python,Ruby,JS等)
+ [axel](http://brewformulas.org/Axel)
超快速下载工具， `axel -n100 "{url}"` 满速下Amazon S3（GitHub上全部资源都是S3的），MEGA等国外资源，还慢的话可以 `axel -S {url}` 搜镜像站点
+ [hub](https://hub.github.com) 顾名思义，git + hub = github，git扩展
+ [tree](http://brewformulas.org/Tree)
必备的代替 `ls cd xxx ls` 的东西
+ [trash](http://brewformulas.org/Trash)
再不怕 `rm -rf` 手残了……
+ [tldr](https://github.com/tldr-pages/tldr) Too long, don't read，还怕man手册的长文介绍？试试这个傻瓜命令行帮助手册
+ [istats](https://github.com/Chris911/iStats)
一键查看各硬件温度，风扇转速，电池容量等，免费，不需要安装不需要root权限，比iStat Menus好用多了


# 小脚本

+ [awesome-osx-command-line](https://github.com/herrbischoff/awesome-osx-command-line)
真是各种OS X的命令行和脚本工具集合，你想要的都能找得到……
+ [mama-bookmark](http://zythum.sinaapp.com/youkuhtml5playerbookmark/) 支持国内各大视频网站直接HTML5观看，拒绝Flash

# 其它

+ [multcloud](https://multcloud.com/home) 网盘迁移助手
