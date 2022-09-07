---
title: Xcode-LLDB耗时监控统计方案
date: 2022-09-07 20:12:00
categories: LLVM
tags:
- swift
- lldb
- llvm
---

# 声明

此篇文章在字节跳动的技术公众号已经刊登：[《字节跳动DanceCC工具链系列之Xcode LLDB耗时监控统计方案》](https://mp.weixin.qq.com/s/4DgbZosBit-kTVhYMwRlHw)

# 背景介绍

在[《Swift 调试性能的优化方案 》](https://dreampiggy.com/2022/05/07/Swift-%E8%B0%83%E8%AF%95%E6%80%A7%E8%83%BD%E7%9A%84%E4%BC%98%E5%8C%96%E6%96%B9%E6%A1%88/)一文中，我们介绍了如何使用自定义的工具链，来针对性优化调试器的性能，解决大型Swift项目的调试痛点。

在经过内部项目的接入以及一段时间的试用之后，为了精确测量经过优化后的LLDB调试Xcode项目效率提升效果，衡量项目收益，需要开发一套能够同时获取Xcode官方工具链与DanceCC工具链调试耗时的耗时监控方案。

一般来说，LLDB内置的工作耗时，可以通过输入log timers dump来获取粗略的累计耗时，但是这个耗时只包括了源代码中插入了LLDB_SCOPED_TIMER()宏的函数，并不代表完整的真实耗时。并且这个耗时统计需要用户手动触发，如果要单独获取某次操作的耗时还需要先进行reset操作清空之前的耗时记录；对于我们目前的需求而言不够精确也不够自动。

因此DanceCC提出了一套专门的方案。方案原理基于[LLDB Plugin](https://reviews.llvm.org/rG4272cc7d4c1e1a8cb39595cfe691e2d6985f7161)，利用[Fishhook](https://github.com/facebook/fishhook)，从LLDB的[Script Bridge API](https://lldb.llvm.org/design/sbapi.html)层面拦截Xcode对LLDB调用，以此来进行耗时监控统计。

注：LLDB论坛也有贡献者，讨论另一套内置的[LLDB metries方案](https://discourse.llvm.org/t/rfc-lldb-telemetry-metrics/64588)，但是目标侧重点和我们略有不同，并且截至发稿日未有完整的结论，因此仅在引用链接提及供读者延伸阅读。

# 方案原理

## LLDB Plugin
Apple在其LLDB和早期Xcode集成中，为了不侵入一些容易改动的上层逻辑，引入了LLDB Plugin的设计和支持。

每个Plugin是一个动态链接库，需要实现特定的C++/C入口函数，由LLDB主进程在运行时通过dladdr找到函数入口并加载进内存。目前有两种Plugin的接口形式（网上常见第一种）

- 新Plugin接口：

```cpp
namespace lldb {
bool PluginInitialize(SBDebugger debugger);
}
```

这种Plugin，需要用户在脚本中手动按需加载，并常驻在内存中：
plugin load /path/to/plugin.dylib

- 老Plugin接口：

```c
extern "C" bool LLDBPluginInitialize(void);
extern "C" void LLDBPluginTerminate(void);
```

将编译的动态库放入以下两个目录，即可自动被加载，无法手动控制时机，在当前调试Session结束时卸载：

```
/path/to/LLDB.framework/Resources/Plugins
~/Library/Application Support/LLDB/PlugIns
```

## 注入动态库
![1](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-09-07/images/1.png)

正常流程中，Xcode开始调试时会启动一个lldb-rpc-server的进程，这个进程会加载Xcode默认工具链，或指定工具链中的LLDB.framework，并且通过这个动态库中暴露出的Script Bridge API调用LLDB的各功能。

![2](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-09-07/images/2.png)

监控流程中，我们向lldbinit文件中添加了`command script import ~/.dancecc/dancecc_lldb.py`，用于在LLDB启动时加载脚本，脚本内会执行`plugin load ~/.dancecc/libLLDBStatistics.dylib`，加载监控动态库。

监控动态库在被加载时，因为被加载的动态库和LLDB.framework不在一个MachO Image中，我们能够通过Fishhook方案，对LLDB.framework暴露出的我们关心的Script Bridge API进行hook。

hook成功之后，每次Xcode对Script Bridge API进行调用都会先进入我们的监控逻辑。此时我们记录时间戳来计时，然后再进入LLDB.framework中的逻辑，获取结果后返回给lldb-rpc-server，并在Xcode的GUI中展示。

## Hook SB API
Hook SB API时，需要一份含有要部署的LLDB.framework的头文件（Xcode并未内置）。由于上述的流程使用了动态链接的LLDB.framework，我们选择了Swift 5.6的产物，并tbd化避免仓库膨胀。

由于LLDB Script Bridge API相对稳定，因此可以使用一个动态库实现，通过运行时来应对不同版本的API变化（极少出现，截止发文调研5.5~5.7之间Xcode并没有改变调用接口）。

对于hook C++函数的方式，这里借用了Fishhook进行替换。原C++的函数地址，可通过dlsym调用得到。注意C++函数名使用mangled后的名称（在tbd文件中可找到）。

```cpp
///
/// Hook a SB API using the stub method defined with the macros above
///
#define LLDB_HOOK_METHOD(MANGLED, CLASS, METHOD) \
Logger::Log("Hook "#CLASS"::"#METHOD" started!"); \
ptr_##MANGLED.pvoid = dlsym(RTLD_DEFAULT, #MANGLED); \
if (!ptr_##MANGLED.pvoid) { \
    Logger::Log(dlerror()); \
    return; \
} \
if (rebind_symbols((struct rebinding[1]){{#MANGLED, (void *) hook_##MANGLED, (void **) & ptr_##MANGLED.pvoid }}, 1) < 0) { \
    Logger::Log(dlerror()); \
    return; \
} \
Logger::Log("Hook "#CLASS"::"#METHOD" succeed!");
```

C++的成员函数的函数指针第一个应该是this指针，这里用self命名。也可以调用原实现先获取结果，再根据结果进行相关的统计逻辑。
```cpp
///
/// Call the original implementation for member function
///
#define LLDB_CALL_HOOKED_METHOD(MANGLED, SELF, ...)  (SELF->*(ptr_##MANGLED.pmember))(__VA_ARGS__)
最终整体代码中Hook一个API就可以写为：
// 假设期望Hook方法为：char * ClassA::MethodB(int foo, double bar)
// 这里写被Hook的方法实现
LLDB_GEN_HOOKED_METHOD(mangled, char *, ClassA, MethodB, int foo, double bar) {
  return LLDB_CALL_HOOKED_METHOD(mangled, self, 1, 2.0);
}
// 这里是执行Hook（只执行一次）
LLDB_HOOK_METHOD(mangled, ClassA, MethodB);
```

# 耗时监控场景

目前耗时监控包含下列场景：
- 展示frame变量
- 展开变量的子变量
- 输入expr命令（p, po命令也是expr命令的alias）
- Attach进程耗时
- Launch进程耗时

## 展示frame变量场景
经过观察，我们发现当在Xcode中进入断点，GUI显示当前frame的变量时，lldb-rpc-server调用SB API的流程为先调用`SBFrame::GetVariables`方法，返回一个表示当前frame中所有变量的SBValueList对象，然后再调用一系列方法获取它们的详细信息，最后调用`SBListener::GetNextEvent`等待下一个event出现。

因此我们计算展示frame变量的流程为，当`SBFrame::GetVariables`方法被调用时记录当前时间戳，等待直至`SBListener::GetNextEvent`方法被调用，再记录此时时间戳算出耗时。

## 展示子变量场景
经过观察，我们发现当在Xcode中展开变量，需要显示当前变量的子变量时，lldb-rpc-server调用SB API的流程为先调用SBValue::GetNumChildren方法，返回表示当前变量中子变量的数目，然后再调用`SBValue::GetChildAtIndex`获取这些子变量以及它们的的详细信息，最后调用`SBListener::GetNextEvent`等待下一个event出现。

因此我们计算展示frame变量的流程为，当`SBValue::GetNumChildren`方法被调用时记录当前时间戳，等待直至`SBListener::GetNextEvent`方法被调用，再记录此时时间戳算出耗时。

## 输入expr命令场景
Xcode中用户直接从debug console中输入LLDB命令的方式是不走SB API的，因此无法直接通过hook的方式获取耗时。我们发现大多数开发者，都习惯在debug console中使用po/expr等命令而不是GUI点击输入框。因此我们专门做了支持，通过SB API的OverrideCallback方法进行了拦截。

LLDB.framework暴露了一个用于注册在LLDB命令前调用自定义callback的接口：`SBCommandInterpreter::SetCommandOverrideCallback`；我们利用了这个接口注册了一个用于拦截并获取用户输入命令的callback函数，这个callback会记录当前耗时，然后调用`SBDebugger::HandleCommand`来处理用户输入的命令。但是当`SBDebugger::HandleCommand`被调用时，我们注册的callback一样会生效，并再次进入我们拦截的callback流程中。

为了解决这个递归调用自己的问题，我们通过一个`static bool isTrapped`变量表示当前进入的expr命令是否被OverrideCallback拦截过。如果未被拦截，将isTrapped置true表示expr命令已经被拦截，则调用HandleCommand方法重新处理expr命令，此时进入的HandleCommand方法同样会被OverrideCallback拦截到，但是此时isTrapped已经被置true，因此callback返回false不再进入拦截分支，而是走原有逻辑正常执行expr命令
![3](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-09-07/images/3.png)


## Attach进程场景
Attach进程时，lldb-rpc-server会调用SBTarget::Attach方法，常见于真机调试的场景。
这里在调用前后记录时间戳，计算出耗时即可。

## Launch进程场景
Launch进程时，lldb-rpc-server会调用SBTarget::Launch方法，常见于模拟器启动并调试的场景。
这里在调用前后记录时间戳，计算出耗时即可。

# 上报部分

## 数据上报
为了进一步还原耗时的细节，除了标记场景的类型以外，我们还会统一记录这些非敏感信息：

- 正在调试的进程名，用于区分多调试Session并存的场景
- 正在调试的App的Bundle ID
- 当前断点位置在哪个文件
- 当前断点位置在哪一行
- 当前断点位置在哪个函数
- 当前断点位置在哪个Module
- 表示当前使用的工具链是Xcode的还是DanceCC的
- 表示当前使用的Swift版本（与Xcode版本一一对应）

在内网提供的版本中，也通过外部环境变量，得知对应的App的仓库标识，用于在内网的数据统计平台上展示和区分。

如图，这是内网大型Swift工程，飞书iOS App接入DanceCC工具链之后，某时间的耗时数据，可以明显看出，DanceCC相比于Xcode的变量显示耗时，优化了接近一个数量级。
![4](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-09-07/images/4.png)
![5](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-09-07/images/5.png)

## 极端耗时场景堆栈收集

除了基本的耗时时间收集以外，我们还希望能够及时发现新增的极端耗时场景和新问题，因此设计了一套极端耗时情况下的调试器堆栈收集机制，目前只要发现，展示变量场景和输入expr命令耗时超过10秒种，则会记录LLDB.framework的当前调用堆栈的每个函数耗时，并将数据上报到后台进行统计和人工分析。

堆栈收集使用了log timers dump所产出的堆栈和耗时信息，本质上是LLDB代码中通过`LLDB_SCOPED_TIMER()`宏记录的函数，其会使用编译器的`__PRETTY_FUNCTION__`能力来在运行时得到一个用于人类可读的函数名。

在获取到调用前和调用后的两条堆栈后，我们会对每个函数进行Diff计算和排序，将最耗时的前10条进行了采样记录，使用字符串一同上传到统计后台中。
![6](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-09-07/images/6.png)

# 总结
无论是App还是工具链，在做性能优化的同时，数据指标建设是必不可少的。这篇文章讲述的监控方案，在后续迭代DanceCC工具链的时候，能够明确相关的优化对实际的调试体验有所帮助，能避免了主观和片面的测试来评估调试器的可用性。

除了调试器之外，DanceCC工具链还包括诸如链接器，编译器，LLVM子工具（如dsymutil）等相关优化，系列文章也会进一步进行相关的分享，敬请期待。

# 引用链接
1. https://mp.weixin.qq.com/s/MTt3Igy7fu7hU0ooE8vZog
2. https://reviews.llvm.org/rG4272cc7d4c1e1a8cb39595cfe691e2d6985f7161
3. https://github.com/facebook/fishhook
4. https://lldb.llvm.org/design/sbapi.html
5. https://discourse.llvm.org/t/rfc-lldb-telemetry-metrics/64588
