---
title: iOS逆向&iOS字体Fallback&游戏汉化
categories: iOS
tags:
  - iOS
  - 逆向
  - Game
updated: '2016-07-14 01:39:32'
date: 2016-06-05 20:33:54
---

> 最近偶然间发现《命运石之门：线形拘束的表征图》(STEINS;GATE 线形拘束のフェノグラム)iOS版的汉化基本完成，只缺测试，于是加入帮忙汉化组……在这中间偶然间学到一些关于iOS逆向和字体相关内容

# 字体问题

游戏是基于iOS UIKit以及Cocos游戏引擎写的（动画效果是Cocos的，文本框是定制的UITextField，而Backlog就是个UIScrollView……）本身支持UTF-8，覆盖汉化脚本后运行，却发现字体渲染不正常，出现了两种字体渲染（看图，"喵"明显是黑体-简，即iOS9之前默认的简体中文字体），更会有很多汉字变为日文汉字（"变" -> "変"，"归" -> "帰")

![](http://dreampiggy-image.test.upcdn.net/image/8/2e/d2b034dd2baab3fa03776f8ec9988.jpg)

起初并没有理解问题，最后查资料才认识到这是iOS字体Fallback所导致的。在查阅资料时候也发现一个异常好用的iOS字体对比渲染网页：[iosfonts](http://iosfonts.com)，直接输入文字，即可查看各种当前固件各种font family的效果（最好用Safari访问，OS X访问的话，相当于查看当前OS X的font family）

![](http://dreampiggy-image.test.upcdn.net/image/1/c5/a3c3555692a29541f2a45c8799323.png)

经过对比以及反编译搜索，发现渲染正常的是HiraKakuProN-W6，而其它字体会被Fallback到STHeitiSC-Light(黑体-简,细体)，原因自然的，因为HiraKakuProN并不是中文字体，只是支持部分日文汉字的显示，剩下的中文汉字将被Fallback（这是iOS UIKit的自动处理，不会出现[?][?]这种空白字块的效果）（这一些列支持中文的字体也有，就是大名鼎鼎的冬青黑体(Hiragino Sans)，在iOS 9和OS X 10.10.6之后加入）

同时，做过iOS国际化的也知道，在iOS Bundle的info.pilst里面，有一个键`CFBundleDevelopmentRegion`，这个键将决定所有文字默认字体，假如你设置为"en"，那么所有英文文字都将Fallback到Helvetica Light(iOS 9之后为San Francisco)，而不是黑体-简的英文字体（参考：[iOS Developer Library](https://developer.apple.com/library/ios/documentation/General/Reference/InfoPlistKeyReference/Articles/CoreFoundationKeys.html#//apple_ref/doc/uid/20001431-130430)）

所以汉化的时候，我们最好确认该值设置为`zh_CN`（大部分日文游戏会是`ja_JP`），防止未手动设置的字体Fallback到日文字体

这样的话，我们的目标就是逆向把字体设置为黑体-简了。

# 逆向

iOS逆向入门，大家可以参考[Reverse Engineering iOS Apps](https://www.whitehack.com.au/reverse-engineering-ios-apps/)，讲述了关于各种类型的逆向，命令行工具，Hopper Disassembler（最好的iOS(ARM)/OS X(x86)逆向工具，支持反编译Objective-C伪代码），同时，[GitHub](https://github.com/iosre/iOSAppReverseEngineering)上有个非常好的电子书，对逆向入门很有帮助。

我们的目标，自然是想办法替换字体名，大家应该都知道，二进制的Hex值你可以替换部分为0x00来无效化，但绝不能随便中间添加插入。因为一旦加入或者删除Hex，后续所有的地址跳转都将挂掉……

通过Hopper静态分析，能发现游戏将我们想要替换的字体名`HiraKakuProN-W6`写死到一个const NSString中去（实际上，Objective-C会编译为`_CFConstantStringClassReference`），遗憾的是，`STHeitiSC-Medium`是16个Byte，而前者只有15个Byte，意味着暴力Hex替换是不可能的。

那么，就没有办法了吗？不是的，就以我不懂逆向来看，也知道至少有两种办法：

### 调用过程前插入指令
对所有引用这个常量地址的过程，手动插入ARM汇编指令，暴力将目标字符串放入寄存器。同时，在调用外过程的时候，把这个寄存器传进去，相当于动态创建了新的字符串并传参……

对于ARM汇编来说，与8086汇编最大的差距就是ARM指令是三地址码(`mov r0,#0x00`)，而且旧版iOS SDK编译的的Mach-O文件更是包含了ARM32(32位指令)和ARM Thumb(16位指令)，需要单独区别。不过熟悉的pc寄存器还是在的，比如这就是进入和函数返回的指令

```nasm
push { r4, r5, r6, r7, lr }
pop { r4, r5, r6, r7, pc }
```

当然，真要做逆向工程师，需要学习大量ARM汇编知识，对我这种基本不懂的人，一般也不会采取这种方式。

### 修改常量段和地址引用
想办法把常量段(__DATA)前面不需要的常量（一般是一些Log的字符串之类）字串缩减，同时对后续所有引用的地址全部修改(减去缩短的字节数)，这样就有了更多空间来修改目标字串。

当然，对于我这种不懂的人，还是选择2方法更好……

通过Hopper全局搜索，发现一个地址为0x002fc50的字串"Not interleaved!"，再通过地址引用，找到调用过程，用伪代码分析一看，是用来做Log的，没有意义，决定用它开刀。

![](http://dreampiggy-image.test.upcdn.net/image/d/a6/20b6c740e08b4ce50c7726931039d.png)

既然我们目标是把16Byte替换15Byte，那么只需要删除一个Byte(也就选"!")了，之后，把从该地址开始，到目标地址的所有地址配合Hex Editor，把所有字符串向前一个Byte，然后更新这些常量的被引用地址值（这步骤手工很麻烦，幸好Hopper支持脚本，大工程可以脚本跑）

替换修改后的常量段：

![](http://dreampiggy-image.test.upcdn.net/image/f/b8/2fd8b75defcd753005347e05e3994.png)

检查没问题，导出可执行文件，测试，结果非常满意，全部被替换为黑体-简(粗体)，相关功能也不受影响（真要说影响，可能就是用户看不到的控制台的Log会输出少个"!"罢了2333）

最终成果（漂亮的黑体-简)：

![](http://dreampiggy-image.test.upcdn.net/image/5/6b/ba2950239095f36f16e57cd593b21.jpg)

# 感想

从中，也可以看出iOS开发的时候，对于常量值，比如一些密码，Key之类的，任何静态分析工具都能轻易找到，并且替换。因此，对敏感的信息，一定要做好相应的安全反逆向。

单纯的宏定义，虽然编译后会展开，但是还是会有一个随机分配标签的常量存在，也容易通过长度和相关函数签名（比如encrypt，key之类名称）的传入参数发现。简单处理，可以加密函数起名A_Function，通过宏定义`#define A_Function ENCRYPT_FUN`，这样将不会反编译后将不会知道标签，更进一步可以通过宏定义直接封装加密函数和解密函数，减少静态分析工具发现的可能性。

不过对于动态分析工具以及Hook系统调用来说，这些都是没有用的……归根结底还是建议要确保iOS客户端，尽量少保留敏感信息，对服务端API设置Auth，SSL验证等手段，才能减少被逆向者获取信息的可能性。

好了，也扯完了，愉快地去(tui)测(you)试(xi)《命运石之门：线形拘束的表征图》了……这作品是《命运石之门》的剧情补完和外传，而妄想科学ADV系列基本一直是我的最爱，大家有兴趣可以去[贴吧](http://tieba.baidu.com/p/4260615784)关注进度。

*El psy congroo*
