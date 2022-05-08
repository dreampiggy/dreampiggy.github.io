---
title: 发现iOS SDK的Bug - Hopper使用教程向
date: 2019-06-13 21:39:38
categories: iOS
tags:
- iOS
- Objective-C
- Hopper
- 逆向
---

# Hopper简介

Hopper，全称Hopper Disassembler，是一个macOS和Linux平台上的反汇编IDE。提供了诸如伪代码，子程序，脚本，Debugger，Hex编辑等等一些列工具。相比于其他知名的反汇编工具如[IDA](https://www.hex-rays.com/products/ida/)，最大的好处是对平台特性，也就是Objective-C的反汇编有优化，提供非常贴近原始代码的伪代码（IDA目前则会是保留诸如objc_msgSend的伪代码），并且新版本也对Swift提供了一定的反汇编符号优化，因此作为探究iOS平台上的SDK实现，可以说是一利器。

# Hopper安装

Hopper本身目前是收费的软件，提供了免费的使用（30分钟）。官方下载地址为：[https://www.hopperapp.com](https://www.hopperapp.com/)。Mac版本后解压，拖到Application下即可使用。

对于个人使用，价格不菲，有两种方案，个人比较推荐第一种

+ Per User：收费为¥700，允许同一时间唯一激活，不绑定机器硬件
+ Per Computer，收费¥900，和一台电脑的机器硬件绑定

对于只是尝鲜或者轻度使用，其实使用免费版即可。网上现在也有针对旧版本的Cracked版本，不过存在一些问题和崩溃。如果是在需要，可尝试[链接](https://xclient.info/s/hopper-disassembler.html)

# Hopper使用

Hopper提供了一个教程，可以参考[官方简易教程](https://www.hopperapp.com/tutorial.html)

针对我们的场景：分析iOS的SDK内实现或者问题，我这里提供了一个Step By Step的过程，教你如何查找问题。

## 获取需要反汇编的二进制文件

首先，我们需要获取一份iOS SDK的二进制Mach-O文件。最简单的方式，是通过Xcode提供的iPhone模拟器去获取它。在获取之前，我们先了解一下iOS SDK对应的二进制文件路径。

+ Xcode自带模拟器，对应系统根路径：

Xcode 11:

`/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Library/Developer/CoreSimulator/Profiles/Runtimes/iOS.simruntime/Contents/Resources/RuntimeRoot`

Xcode 10:

`/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/Library/CoreSimulator/Profiles/Runtimes/iOS.simruntime/Contents/Resources/RuntimeRoot/`

+ 已下载的历史版本固件的模拟器，对应系统根路径，自己根据版本版本修改中间的数字：

`/Library/Developer/CoreSimulator/Profiles/Runtimes/iOS\ 11.4.simruntime/Contents/Resources/RuntimeRoot/`

+ 真机的系统根路径，不用说了吧：

`/`

iOS 系统提的库和二进制，可以简单分以下几类，按照需要选择对应的相对根路径：

+ 公开Framework: `/System/Library/Framework`
+ 私有Framework：`/System/Library/PrivateFrameworks`
+ 系统App：`/Applications`
+ UNIX动态库: `/usr/lib`

这里我们以Xcode 10自带的iOS 12 SDK，UIKitCore为例（注意，UIKit从iOS 12开始，为了支持部署到macOS，将代码基本全盘移动到了私有Framework的UIKitCore.framework中，UIKit.framework只是一个外层的壳），我们就能直接去访问这个路径，获取它的Mach-O二进制：

`/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/Library/CoreSimulator/Profiles/Runtimes/iOS.simruntime/Contents/Resources/RuntimeRoot/System/Library/PrivateFrameworks/UIKitCore.framework/UIKitCore`

一般来说，iPhone模拟器提供的二进制Mach-O即可够用，虽然它实际上是x86_64架构的编译产物，但是基本上的逻辑和真机上是一致的。如果涉及到需要只能在真机可用的库，如AVFoundation的摄像机，建议可以从真机中提取（也可以从iOS的IPSW固件中提取），见下文。


### 真机获取系统库的二进制文件

### 获取dyld shared cache

在真机上，为了加快动态库的加载，并减少iOS 占用磁盘的体积，dyld采取了一个缓存，将多个Mach-O文件合并到一起，由系统启动后就预热。因此，实际上系统库（公开和私有）的路径上，只有Framework和其中的资源文件，却没有对应的Mach-O二进制。我们需要首先获取到这个cache，然后解压出来对应的二进制。参考[dyld_shared_cache](https://iphonedevwiki.net/index.php/Dyld_shared_cache)

对应dyld shared cache路径（以arm64机器为例）：`/System/Library/Caches/com.apple.dyld/dyld_shared_cache_arm64`

当然，除了使用已经越狱的真机，我们还可以通过IPSW，即iOS的固件包，来直接提取对应的dyld shared cache，并解压得到对应的Mach-O文件。

IPSW可以从这个[网页](https://ipsw.me/)上下载，选择你的设备以及iOS版本号，就可以下载对应的IPSW文件。

将下载的IPSW解压（建议使用zip命令行，或者BetterZip之类的解压软件，Mac自带的解压似乎会报错），可以找到一个最大容量的DMG文件，双击即可加载
![](http://dreampiggy-image.test.upcdn.net/image/2019/06/13/15604236336003.jpg)

加载后就是完整的iOS系统根路径了，从对应路径下找到dyld shared cache。

### 解压dyld shared cache

为了解压dyld shared cache，市面上一些工具其实早已过期，要么不支持arm64，要么存在Bug。但实际上，Apple自己开源的dyld源代码，就已经包含了这样一个命令行工具，叫做`dsc_extractor`，我们这里直接用来源码来编译一份来使用即可。

进入[opensource.apple.com](https://opensource.apple.com/，)选择最新的macOS的版本，我这里例子使用的是我写这篇文章最新公开的 [macOS 10.14.1](https://opensource.apple.com/release/macos-10141.html)

然后下载两份代码，一份是[dyld](https://opensource.apple.com/tarballs/dyld/dyld-635.2.tar.gz)，一份是[CommonCrypto](https://opensource.apple.com/tarballs/CommonCrypto/CommonCrypto-60118.220.1.tar.gz)

为了编译，需要一点小技巧，但是对于iOS开发者我觉得挺简单

1. 用Xcode，打开dyld代码中的`dyld.xcodeproj`
2. 修改Build Settings中，把对应的Base SDK，从`macosx.internal`改成公开的`macOS`
3. 进入`dsc_extractor.cpp`，看到最后有一个`test program`，把上面的`#if 0`改成`#if 1`
4. 我在编译新版本时发现依赖了一个叫做`CommonDigestSPI.h`的私有头文件，这个在下载的CommonCrypto工程中，拖进来改一下引用方式即可
5. 选择`dsc_extractor`，Archive得到一个产物，叫做`dsc_extractor.bundle`，然而他实际就是一个Mach-O二进制，直接删掉后缀，chmod+x，即可使用

![](http://dreampiggy-image.test.upcdn.net/image/2019/06/13/15604271114928.jpg)

如果上面的编译比较麻烦，可以直接下载我这里编译好的一份二进制，然后放到你的PATH路径下：[dsc_extractor](https://raw.githubusercontent.com/dreampiggy/dsc_extractor/master/bin/dsc_extractor)

然后我们可以使用`dsc_extractor`来解压我们提取到的dyld shared cache，很简单的命令

```
dsc_extractor ./dyld_shared_cache_arm64e ./output
```

会得到所有dyld shared cache中的二进制Mach-O文件，按照路径排列，然后我们就可以用自己想反编译的库，如UIKitCore，来使用Hopper了。


## 载入Hopper

现在我们已经有了一个UIKitCore的Mach-O文件了，我们打开Hopper来载入它。我们可以使用Command+Shift+O来选择一个Mach-O文件，也可以将文件拖动到Hopper界面上来打开。

![](http://dreampiggy-image.test.upcdn.net/image/2019/06/13/15604335461822.jpg)

载入Mach-O文件后，Hopper会弹出框来选择具体分析的内容，大部分情况直接确认即可。如果是分析其他类型的文件，可能有特例如下：

+ 分析一个.a或者.dylib，并且该二进制由多个.a或者.dylib合成，这时候会提示你选择具体的某个编译产物
+ 分析一个FAT Binary，这时候会提示你选择具体的某一个架构的文件

载入开始后，一般需要等待一段时间来分析（下方会有进度条），等待分析完成后，你可以将当前分析的结果，保存成一个`.hop`结尾的文件，未来就不再需要分析了，非常有用（注：免费版不可用）。

## 符号分析

左侧有一个符号框，从左到右依次表示：

+ Labels: C/C++Objective-C的符号，包括类名，方法名，全局变量等
+ Proc：子程序，对应C/C++的函数，Objective-C的方法，Block代码段等
+ Str：常量段，包括了所有C/C++Objective-C字面量，即代码中直接用`@"", ""`写的内容

![](http://dreampiggy-image.test.upcdn.net/image/2019/06/13/15592985644004.jpg)

每项内容都支持搜索，一般来说取决于我们要解决的问题，有大概几个场景

1. 分析特定方法的实现：使用Proc搜索
2. 运行时抛出的异常或者Log：使用Str搜索关键字
3. 得知一个类的所有方法：可以使用Label，但更好的方式是通过Class-dump获取头文件（见下）

## Class-dump与私有头文件

[Class-dump](https://github.com/nygard/class-dump)是一个能够解析Mach-O文件，对应的Objective-C符号，以生成一个完整的头文件的工具。得益于Objective-C运行时和符号的特点，可以方便的还原回基本接近原始的类声明代码。具体使用也很简单，参见项目的Readme，编译得到二进制，放到PATH中，然后执行：

```
class-dump UIKitCore.framework -r -o output -H
```

对于重头戏，关于iOS SDK的所有头文件，早有专人建立了一个在线网站去分析，点击跳转：[iOS Runtime Headers](http://developer.limneos.net/)

在这个网页上，可以支持Framework/类/方法级别的搜索，支持点击头文件跳转链接，非常的方便，一般的分析iOS SDK都可以采取这个网页的结果来辅助分析。


## 伪代码分析

当我们了解到需要分析的符号方法后，下一步一般就会进行伪代码分析。在Hopper中，点击到一个子程序入口，然后点击上方的这个像是`if (b)`代码的图标，即可打开伪代码分析框

![](http://dreampiggy-image.test.upcdn.net/image/2019/06/13/15592994457947.jpg)

对于简单的代码，我们基本上能够还原回100%可读的Objective-C代码，由于ARC时便一起，我们可以看到对应的Retain和Realse调用

![](http://dreampiggy-image.test.upcdn.net/image/2019/06/13/15592995793557.jpg)


### 分析调用关系

我们可以通过对应的子程序页面，右键选择"References To Selector"，来查看所有对这个Selector的调用。（由于Objective-C运行时的特点，只能是Selector级别的调用，如果有不同类的同名Selector，可以在弹出的窗口中搜索或者依次检查）

![](http://dreampiggy-image.test.upcdn.net/image/2019/06/13/15593006542887.jpg)


## 常见的分析姿势

### Block

Objective-C会使用到Block，而Block由于其实现原理，会生成对应的C方法，Hopper目前原生解析的Block语法并不是很直观，这里提供一个简单的说明。

其实Hopper反编译出来就是Block实现的原理，如果对于Block实现原理不清楚，建议可以先看一遍[《这个教程》](https://blog.devtang.com/2013/07/28/a-look-inside-blocks/)

### 简单Block

```c
dispatch_async(dispatch_get_main_queue(), ^{
    printf("%d", 1);
});
```

Hopper原生反编译如下，实际Block代码会单独在另一个C方法中，在`block implemented at:`提示对应的方法中

```c
dispatch_async([objc_retainAutoreleaseReturnValue(*__dispatch_main_q) retain], ^ {/* block implemented at ___29-[ViewController viewDidLoad]_block_invoke */ } });

void ___29-[ViewController viewDidLoad]_block_invoke(void * _block) {
    printf("%d", 0x1);
    return;
}
```

### 捕获变量

如果Block捕获了变量，那么根据Block的实现原理，可以知道这些变量在Block中可见的变量都是被值宝贝，对于NSObject就是指针

如果使用`__block`修饰，那么会保留原始的变量的指针，对于NSObject就是对象指针的指针，我们可以通过这个简单识别。

比如对于这样代码：

```c
NSObject *obj = [NSObject new];
__block NSObject *obj2 = [NSObject new];
[self testBlock:^(int value){
    NSLog(@"%@", obj);
    obj2 = nil;
}];
```

实际反编译出来的结果长这样：

```c
int ___29-[ViewController viewDidLoad]_block_invoke(int arg0, int arg1) {
    NSLog(@"%@", *(arg0 + 0x20));
    rax = objc_storeStrong(*(*(arg0 + 0x28) + 0x8) + 0x28, 0x0);
    return rax;
}
```

对应的`arg0`就是第一个参数，而最后参数对应的是`block_impl_0`实现结构体，可以忽略。



## CGRectMake等inline的C方法

一些带有inline数值计算的方法，会被苹果的clang在编译时优化，实际上并不是你看到的头文件的样子，这种就需要我们枚举出来，人肉还原回他的实现，举个例子：


这样的代码：

```c
CGRect rect = CGRectMake(0, 1, 2, 3);
NSLog(@"%@", NSStringFromCGRect(rect));
```

反编译结果：

```c
intrinsic_movsd(xmm1, *double_value_1);
intrinsic_movsd(xmm2, *double_value_2);
intrinsic_movsd(xmm3, *double_value_3);
_CGRectMake(&var_30, _cmd, rdx, rcx);
NSLog(@"%@", [NSStringFromCGRect(*(&var_30 + 0x10), *(&var_30 + 0x18)) retain]);
```

可以看到有`mov`之类的汇编命令调用，其实这就是为了压栈其实大部分场景我们只要熟悉简单的`mov` `add` `sub` `mul`几个基本的汇编命令的意义即可。

## Swift

Swift作为Apple一致力推的下一代官方编程语言，随着iOS 13的发布，现在已经可以作为第一优先的SDK支持语言了，iOS 13上出现了4个Swift Only的库，因此对于Swift相关的反编译需求，也会慢慢出现。然而，不同于动态性强的Objective-C代码，Swift天生的静态强类型语言特性，造成了相当高的反编译难度（堪比C++开O2优化），在这里基本不细讲，只是大概说一下目前的状况。

Hopper从v4开始支持了对Swift符号的符号化，我们不再需要使用swift来反解决mangled的符号名。

由于Swift支持完整的命名空间，查询符号需要带上完整的符号

![](http://dreampiggy-image.test.upcdn.net/image/2019/06/13/15604313892455.jpg)


同时，Swift由于clang的优化，会讲很多编译器检查到的频繁的代码调用，自动转换为一个以`sub`开头的函数，以减少二进制大小。

对于Swift非`@objc`和`dynamic`的属性和方法，会类似于C++的虚函数表，实际上的调用都是编译器展开的地址偏移，而不像Objective-C那样有符号可查。这种时候我们需要就是类似C++反编译那样，通过分析Swift class或者struct的属性，来对照偏移量得知调用。

对于Swift的会触发运行态的一些语法，需要你对Swift语言实现有了解，比如Protocol Extension Where子句，会生成Protocol Witness，我们可以在Hooper中搜索到它

![](http://dreampiggy-image.test.upcdn.net/image/2019/06/13/15604319451736.jpg)

可以看到，目前的Hopper对Swift有相应的支持，但受限于Swift的语言性质很难直观阅读，必要时候还是需要一些汇编，以及传统C++的反编译分析模式去对待它


# 总结

这篇教程基本上是从我个人的使用经验来介绍，以工具和流程为主，主要是为了给目标iOS平台，且不是专攻二进制安全的人来阅读。

其实对大部分iOS平台开发者，最主要的目的，其实是为在发现一些iOS SDK表现奇怪的行为，或者Crash时，能够有一定的分析和判断能力，去尝试定位原因，绕过问题，并最终能够有底气，去向Apple提交Bug Report。

反编译本身就是二进制安全中的灰色地带，而且还有类似二进制加固等攻防模式，并不是万能方式去了解一个程序运行的方式。还需要配合自己的编写代码经验，才能更好地解决问题


# 参考

+ https://github.com/bartcone/reverse-engineering-blog
+ http://stevenygard.com/projects/class-dump/
+ https://worthdoingbadly.com/dscextract/
+ http://iphonedevwiki.net/index.php/Dyld_shared_cache
+ http://www.toves.org/books/arm/

