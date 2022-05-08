---
title: 一段简单逆向之旅-绕开Xcode 13.3最低macOS 12.0限制
date: 2022-03-25 20:47:50
categories:
tags:
  - macOS
  - Xcode
  - 逆向
---

> 因为众所周知的原因，苹果的Xcode版本会不断提高自己的最低安装版本，在Xcode 13.0-13.2.1上，这个最低安装版本是macOS 11
>
> 而随着Xcode 13.3正式版放出，这个最低部署版本在最后关头被提升到了macOS 12

# Why？

一般来说，各位开发者或者众多基建，总有各种各样的原因需要暂时留在老版本的macOS系统上，但是又希望使用新Xcode版本自带的Toolchain进行一些工作开发调试，有些是主观问题，有些是客观限制：

举例子：

1. macOS 12禁止了`sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setglobalstate off`绕过GUI配置，导致一些公司采取Apple Device Management管理的电脑，无法正常关闭TCP拦截，会导致一些服务异常
2. macOS 12加强了Kernel Extension的安全性，导致GitHub Action和Circle CI截止2022年3月底，迟迟无法更新他们的虚拟化集群到macOS 12，只有11.6的最新版本

这些都是闲聊，进入正题。那么有没有办法能够绕开，或者从原理上来讲，是否这个系统绑定的最低部署版本限制是必要的？

# 先放结论

1. 可以绕过这个macOS 12的最低安装版本限制运行
2. 这个限制是非必要的，绕过以后所有功能正常可用（构建，独立工具集，调试，连接iPhone）

下面来说明具体的逆向流程，和进行绕过的简单Step-by-step手法

# 安装Xcode

先说明测试机器Mac环境和Xcode环境：

1. macOS 11.4 (20F71)
2. Xcode Version 13.2 (13C90)：需要保留一份以防万一
3. Xcode Version 13.3 (13E113)：目标安装的版本

首先，作为iOS/macOS开发者，我们肯定会使用dmg的格式，或者使用[Xcodes.app](https://github.com/RobotsAndPencils/xcodes)来安装我们的Xcode 13.3了（App Store安装Xcode曾经出的坑：[App Store version of Xcode 13.2 causing problems for developers](https://appleinsider.com/articles/21/12/14/app-store-version-of-xcode-132-causing-problems-for-developers)，我是不会再用了）

安装完毕后，我们在Finder中看到的Xcode.app是一个画着❎的样子，直接打开会提示如下：

![1648210752406_af87e2021743b9639d35f86714f127c7](http://dreampiggy-image.test.upcdn.net/2022/03/25/1648210752406_af87e2021743b9639d35f86714f127c7.png)


# 绕过GUI部分的限制

## 修改`LSMinimumSystemVersion`

作为iOS/macOS开发者，我们第一想到的就是，是否是Xcode.app对应的Info.plist中，设置了和最低部署版本相关的字段导致拒绝载入呢？

我们用另一个Xcode（或者plistutil）打开`Xcode.app/Contents/Info.plist`，果然发现了对应的字段：

![1648210752770_b314e1b23bdd01fe803ede1383e29491](http://dreampiggy-image.test.upcdn.net/2022/03/25/1648210752770_b314e1b23bdd01fe803ede1383e29491.png)

这个[LSMinimumSystemVersion](https://developer.apple.com/documentation/bundleresources/information_property_list/lsminimumsystemversion)是Mac应用标准的声明最低部署版本的方式，修改为你的机器当前OS版本之后保存，执行

```
touch /Applications/Xcode-13.3.0.app
killall Finder
```

重新尝试双击。不错，这次我们打开了，初看起来不错（直到我们正式开始编译）！

![1648210752322_3d4e9ae78eaab94de9cfc7d1f8eaecf0](http://dreampiggy-image.test.upcdn.net/2022/03/25/1648210752322_3d4e9ae78eaab94de9cfc7d1f8eaecf0.png)


# 绕过CLI部分的限制

## 神奇的xcrun

但是只要创建一下工程并执行编译，就会发现，各种命令行工具的调用是有问题的，比如我们先通过xcode-select设置为当前的Xcode 13.2，尝试执行：

![1648210752844_f21e111a081ff5dfc61dab7137109a39](http://dreampiggy-image.test.upcdn.net/2022/03/25/1648210752844_f21e111a081ff5dfc61dab7137109a39.png)
![1648210752266_e83287242af496cbbec7817b4b60f9f0](http://dreampiggy-image.test.upcdn.net/2022/03/25/1648210752266_e83287242af496cbbec7817b4b60f9f0.png)

但是我们如果直接找到，执行对应绝对路径的clang，是可以执行的

![1648210752227_fd8cd9f89407475fe62c0489dc61a232](http://dreampiggy-image.test.upcdn.net/2022/03/25/1648210752227_fd8cd9f89407475fe62c0489dc61a232.png)

并且，我们可以直接检查clang这个二进制，是否链接时设置了target，这部分可以使用otool -l读取machO Header查看到：

![1648210752133_02c1257232efb1e2adebcb51ea25ceb0](http://dreampiggy-image.test.upcdn.net/2022/03/25/1648210752133_02c1257232efb1e2adebcb51ea25ceb0.png)

好，最低部署版本是macOS 10.14.6；那现在我们有充分的证据说明，一定可以在我当前的电脑运行clang，而上述提示应该是xcrun这个调度器，添加了额外的判断。

通过搜索关键词，可以在Xcode的strings输出中找到这句“Executable requires at least”的关键字：参考仓库：[Xcode.app-strings](https://github.com/keith/Xcode.app-strings/blob/11e15fd3cd8446f2a966746caefcef5beebc8929/Xcode.app/Contents/Frameworks/libxcodebuildLoader.dylib)

## 反编译`libxcodebuildLoader`

我们定位到这个`libxcodebuildLoader.dylib`，拖进Hopper尝试反编译理解他检查的原理，伪代码如下：

```c
void _checkMinimumOSVersion(int arg0) {
    var_2C = 0x0;
    rbx = 0x0;
    rax = _NSGetExecutablePath(0x0, &var_2C);
    rdi = var_2C;
    if (rdi != 0x0) {
            rbx = malloc(rdi);
    }
    rax = _NSGetExecutablePath(rbx, &var_2C);
    if (rax != 0x0) goto loc_282c;

loc_26fc:
    rax = [NSString stringWithUTF8String:rbx];
    rax = [rax retain];
    r14 = rax;
    r15 = [[NSURL fileURLWithPath:rax] retain];
    if (rbx != 0x0) {
            free(rbx);
    }
    rbx = CFBundleCopyInfoDictionaryForURL(r15);
    CFRelease(r15);
    if (rbx == 0x0) goto loc_2867;

loc_276a:
    r13 = [CFDictionaryGetValue(rbx, @ DVTMinimumSystemVersion ) retain];
    CFRelease(rbx);
    if ((r13 == 0x0) || ([r13 length] == 0x0)) goto loc_280c;

loc_27a6:
    r12 = *_objc_msgSend;
    r15 = [[DVTVersion versionWithStringValue:r13] retain];
    rax = [DVTVersion currentSystemVersion];
    rax = [rax retain];
    rbx = r12;
    r12 = rax;
    rdx = r15;
    if ([rax isEqualToOrNewerThanVersion:rdx] == 0x0) goto loc_288c;

loc_27fb:
    [r12 release];
    [r15 release];
    goto loc_280c;

loc_280c:
    [r13 release];
    [r14 release];
    return;

loc_288c:
    r15 = [(rbx)(r15, @selector(stringValue), rdx) retain];
    rax = (rbx)(r12, @selector(stringValue), rdx);
    rax = [rax retain];
    r14 = [(rbx)(@class(NSString), @selector(stringWithFormat:), @ Executable requires at least macOS %@, but is being run on macOS %@, and so is exiting. , r15, rax) retain];
    [rax release];
    [r15 release];
    fprintf(**___stderrp,  %s\n , (rbx)(objc_retainAutorelease(r14), @selector(UTF8String), @ Executable requires at least macOS %@, but is being run on macOS %@, and so is exiting. ));
    goto loc_292b;

loc_292b:
    exit(0x1);
    return;

loc_2867:
    fwrite( Unable to open executable info dictionary; xcodebuild may be corrupt and should be reinstalled.\n , 0x60, 0x1, **___stderrp);
    goto loc_292b;

loc_282c:
    _DVTAssertionFailureHandler(*_self, *__cmd,  void checkMinimumOSVersion() ,  /Library/Caches/com.apple.xbs/Sources/IDETools/IDETools-20008/xcodebuildLoader/xcodebuildLoader.m , 0x69, @ 0 , @ Couldn't get executable path to self! );
    return;
}
```

好，阅读伪代码以及查阅资料可知：

`xcrun`会先一步调用到`xcodebuild`，检查`DVTMinimumSystemVersion`这个变量的值是否和当前OS版本匹配。

而这个变量，竟然是通过[CFBundleCopyInfoDictionaryForURL](https://developer.apple.com/documentation/corefoundation/1537134-cfbundlecopyinfodictionaryforurl)打开的。

参考苹果的函数说明，它除了常规的打开一个.bundle的文件夹，解析为NSBundle.infoDictionary以外，竟然能打开存在于二进制`__TEXT,__info_plist`中的数据来解析为一个字典。所以我们接下来去找`xcodebuild`的二进制看看。

![1648210752247_62087827de34d7b56eea1e48208b261e](http://dreampiggy-image.test.upcdn.net/2022/03/25/1648210752247_62087827de34d7b56eea1e48208b261e.png)

参考：

1.  `_NSGetExecutablePath`：[函数说明](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man3/dyld.3.html)，大概理解获取当前程序的可执行路径

## 反编译`xcodebuild`

同时，出于好奇，我们可以再把`xcodebuild`拖进Hopper去尝试理解，发现它整个程序竟然只有一个main函数，逻辑其实都在`libxcodebuildLoader.dylib`中

```c
void _main(int arg0, int arg1) {
    r14 = arg1;
    rbx = arg0;
    rax = dlopen( @rpath/libxcodebuildLoader.dylib , 0x1);
    if (rax == 0x0) goto loc_100002b7f;

loc_100002b5c:
    rax = dlsym(rax,  XcodeBuildMain );
    if (rax == 0x0) goto loc_100002bd8;

loc_100002b70:
    (rax)(rbx, r14);
    return;

loc_100002bd8:
    rax = dlerror();
    rax = [NSString stringWithUTF8String:rax];
    rax = [rax retain];
    rax = objc_retainAutorelease(rax);
    r15 = rax;
    rax = [rax UTF8String];
    rsi =  Error loading symbol: %s\n ;
    goto loc_100002c2f;

loc_100002c2f:
    fprintf(**___stderrp, rsi);
    rbx = [_prunedErrorMessage() retain];
    [r15 release];
    if (rbx != 0x0) {
            _main.cold.1(rbx, @selector(UTF8String));
    }
    return;

loc_100002b7f:
    rax = dlerror();
    rax = [NSString stringWithUTF8String:rax];
    rax = [rax retain];
    rax = objc_retainAutorelease(rax);
    r15 = rax;
    rax = [rax UTF8String];
    rsi =  Error loading required libraries. If there is an ongoing installation please wait for it to complete. Otherwise reinstall. (%s)\n ;
    goto loc_100002c2f;
}
```

## 修改`DVTMinimumSystemVersion`

其实大家也发现了，`xcodebuild`二进制本身竟然内嵌了一段XML！我使用`llvm-objdump`把它直接提取了出来：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
<key>BuildMachineOSBuild</key>
<string>21E160</string>
<key>CFBundleDevelopmentRegion</key>
<string>English</string>
<key>CFBundleDisplayName</key>
<string>xcodebuild</string>
<key>CFBundleIdentifier</key>
<string>com.apple.dt.xcodebuild</string>
<key>CFBundleInfoDictionaryVersion</key>
<string>6.0</string>
<key>CFBundleName</key>
<string>xcodebuild</string>
<key>CFBundleShortVersionString</key>
<string>13.3</string>
<key>CFBundleSupportedPlatforms</key>
<array>
<string>MacOSX</string>
</array>
<key>CFBundleVersion</key>
<string>20008</string>
<key>DTCompiler</key>
<string>com.apple.compilers.llvm.clang.1_0</string>
<key>DTPlatformBuild</key>
<string>21E185d</string>
<key>DTPlatformName</key>
<string>macosx</string>
<key>DTPlatformVersion</key>
<string>12.3</string>
<key>DTSDKBuild</key>
<string>21E185d</string>
<key>DTSDKName</key>
<string>macosx12.3.internal</string>
<key>DTXcode</key>
<string>1330</string>
<key>DTXcodeBuild</key>
<string>13E112a</string>
<key>DVTMinimumSystemVersion</key>
<string>12.0</string>
<key>LSMinimumSystemVersion</key>
<string>11.0</string>
</dict>
</plist>
```

看到我们关心的`DVTMinimumSystemVersion`和`LSMinimumSystemVersion`都在里面。其实也侧面证明了，真正的最低部署版本是macOS 11.0，而不是macOS 12.0（12.0只是苹果为了间接Push Developer去频繁更新macOS的阴谋罢了😂）

那下一步，要做的事情就是用魔改`xcodebuild`并重新codesign。修改的方式多种多样，你暴力使用Hex Editor也是最简单。但是我更好奇的是这个`__TEXT,__info_plist`的machO段和节的相关说明。

在网上搜索了一下相关资料，很容易就找到了感兴趣的资料：

1. [The Power Of Plist](https://redsweater.com/blog/2083/the-power-of-plist)：解释Info.plist可以内嵌在二进制中
2. [Gimmedebugah: how to embedded a Info.plist into arbitrary binaries](https://reverse.put.as/2013/05/28/gimmedebugah-how-to-embedded-a-info-plist-into-arbitrary-binaries)：对任意已有二进制注入Info.plist
3. [llvm-objcopy](https://llvm.org/docs/CommandGuide/llvm-objcopy.html)：拷贝修改machO结构到新machO

基本解释得很明确清晰，如果你有源码，可以直接利用ld64的参数`  --sectcreate __TEXT,__info_plist path_to/Info.plist `来注入你的Info.plist信息。没有源码可以手动修改machO结构并签名即可。

对于我此次跑Xcode 13.3来说，我选择最傻瓜最直观的Hex Editor修改（我用的是开源小工具[HexFiend](https://github.com/HexFiend/HexFiend)），只需要把`12.0`修改为`11.0`即可满足我的需要，并重新codesign一波。

![1648210752222_811d0d800592f601f309e48b732f1194](http://dreampiggy-image.test.upcdn.net/2022/03/25/1648210752222_811d0d800592f601f309e48b732f1194.png)

codesign：

```
codesign --remove-signature /Applications/Xcode-13.3.0.app/Contents/Developer/usr/bin/xcodebuild
sudo /Applications/Xcode-13.3.0.app/Contents/Developer/usr/bin/xcodebuild -license
```

测试一下CLI，很正常

![1648210752374_0a5d6ef92b0e270d6164a05153565572](http://dreampiggy-image.test.upcdn.net/2022/03/25/1648210752374_0a5d6ef92b0e270d6164a05153565572.png)

# 最终替换步骤

1. 修改`Xcode-13.3.0.app/Contents/Info.plist`中的`LSMinimumSystemVersion`的值为`11.0`
2. 替换`Xcode-13.3.0.app/Contents/Developer/usr/bin/xcodebuild`中的`DVTMinimumSystemVersion`的二进制为`11.0`，或者使用我这个已经替换好的（建议还是手动参考上面步骤[修改DVTMinimumSystemVersion]替换，授人以渔而不是授人以鱼）