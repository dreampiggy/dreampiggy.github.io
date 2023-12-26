---
title: LLDB调试信息裁剪传输方案
date: 2023-04-04 17:01:01
categories: LLVM
tags:
- swift
- lldb
- llvm
- toolchain
---

## 声明

此篇文章原作者就是我，版权所有。预计未来会刊登在《字节跳动终端技术》

公众号链接：
![](https://mp.weixin.qq.com/mp/qrcode?scene=10000005&size=102&__biz=Mzg2NTYyMjYxNg==&mid=2247486840&idx=1&sn=43b8a41875f86f7b62356ff2a3c064ab&send_time=)

## 背景介绍
在如今，越来越多应用采取分布式构建系统，以及一些云IDE的兴起，在这种场景下，如何保证跨机器的编译产物，能够正常的在另一台机器进行正常的开发调试，是一个常见的问题。

传统的单机编译和链接流程中，编译器会在产物中嵌入当前编译单元的单机的路径，中间产物的路径；链接器在链接时，也会尝试写入链接器输入的所有Object File和Archive File的路径。在随后的调试器工作时，会通过读取MachO Executable的Section中，编码的调试信息和路径，以进行行断点的匹配，源码信息的展示等等能力。自然的，如果编译器或者链接器在处理时全部以当前机器的绝对路径进行编码，则跨机器的产物传输后，就不能正常的实现调试功能。

对此，大部分分布式构建解决方案提供了避免绝对路径，或者绝对路径对相对路径对映射方案，其依赖编译器或者链接器的特定参数注入，也可能会依赖dSYM Bundle这种二次链接产物来进行调试信息传输。但是前者其存在一定的项目接入成本，需要依赖其构建时所有二进制（尤其是外部引入的三方预编译好的二进制）都进行了相对路径的处理。而后者的dSYM Bundle对增量不友好，会严重影响开发-调试周期的平均耗时。

当然，解决思路有很多。我们曾经使用了分布式进行编译，单机进行链接（以保证编编码进MachO Executable的Section中的路径都在当前机器可访问），随后在调试器启动时设置Source-Map来映射预二进制的源码路径。但是在更复杂的分布式构建场景下，链接阶段也会进行分布式处理。因此，为了保障开发阶段的应用，在用户设备上也能正常安装，调试，我们提供了一系列的解决能力支持，这篇文章主要用于分享相关的解决方案思路。

## OSO和dSYM Bundle
对于C/C++/Objc和Swift编译器，其会将调试信息（如编译单元的路径，函数和变量名，变量的寄存器/栈信息等），按照DWARF规范进行编码。DWARF规范得到的编码数据是二进制的，需要找到文件来实际存储。

在macOS/iOS等类UNIX系统的历史中，这个调试信息会写入到编译器输出的MachO Object File中，其中编译单元的源码路径会写入Symble Table的SO Symbol中，并编码到最终的MachO Executable中。但是这一设计会造成Debug Build的二进制过于庞大（相当于DWARF同时编码在Object File和Executable中并重复占用），对于无论是磁盘存储，还是移动应用分发这种场景都是一大痛点。

因此，在2005年，Apple的ld64链接器，不再直接编码DWARF到最终的MachO Executable中，而是引入了一个中间映射关系，称为Debug Map。其像指针一样记录了MachO Object File的路径，以及修改时间戳（防止用户重编译了Object File但是没有重链接Executable）。这样以来，调试器会从直接访问巨大二进制里的DWARF，转为先打开编译单元产物的DWARF计算偏移，随后读取，解决了这重复一倍的磁盘占用。

而这一设计类似SO Symbol指向源码路径，因此称这些MachO Object File为OSO（SO for Object）。

随之诞生的，还有dSYM Bundle。因为上述改动后，一个MachO Executable不再“内嵌”所有调试信息了，意味着你将一个MachO Executable传输到另一台机器上，需要同时带上所有的OSO，并且每一个路径都放置正确才行，和当时的很多构建流程，以及开发者的习惯不兼容。

因此，Apple开发了一套能够重新把调试信息聚合到一起的工具，也就是dsymutil。dsymutil会根据OSO的指引，打开所有的Object File解析DWARF，并修正地址偏移，去重，“链接”到最终的一个大的DWARF文件，并用MachO格式封装。这也是如今常见的分发调试信息的方式。

当然，凡事都有代价。dsymutil从工作流程上来看，就是一个类似“链接器”的工作，其也有类似的修正地址的rebase和bind动作，是严重的单进程CPU密集型应用，在大型项目中，对于上万个OSO文件，dsymutil会执行超过5分钟才可生成完毕，并且目前是不可增量的(*)。意味着就算改动1行代码，也需要额外等5分钟开销才能开始调试流程，因此主要用于最终发布阶段的调试信息分发和长期存储。

## 我们的解决方案
回到正题，在分布式场景下的调试能力，只有两种选择：
### 调试器运行在编译/链接器所在机器上
假设分布式的编译器不产生绝对路径（或者使用类似LTO的流程），我们保证链接器和调试器在同一台机器上。通过远程调试（从Remote Host启动LLDB，Attach到一个Local Process上）的能力，即可达到正确的效果，但是这存在一定的实践局限性：
1. 调试器依赖古老的gbd-remote协议，在两个机器上以“同步单工（串行）”来传输信息，包括寄存器信息，线程信息，写入内存等等操作，其中不乏大量二进制的压缩数据，对带宽要求较高。在我们测试的大型应用上，一次的断点陷入和Frame Variable变量查看，需要传输接近100MB的信息。传统Xcode和iPhone真机调试，是通过usbmux来进行传输的，但是一旦我们使用TCP Socket替代来进行在我们测试的大型应用上，在带宽受限的环境下可能需要等待10-20秒才能完成。
![](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2023-12-26/media/17035879759023.jpg)
2. 各种调试指令的输入和输出，完全依赖网络传输（就算打印一个变量的值也是），导致网络间歇性中断/离线情况下会完全不可用，对云IDE场景会影响用户体验

综上，在实际的落地场景中，在测试效果达不到预期后，我们并没有沿着这条路继续探索，转而使用下文的方案。

### 调试器不运行在编译/链接器所在机器上
另一种场景就是，编译/链接的机器，和调试器所在机器完全分离。这部分在传统构建中，通常会采取dSYM Bundle + 二进制包来进行分发，随后进行调试的方案来处理，以保证调试产物的可迁移性。但还有痛点：
1. 但是dSYM Bundle由于上文提到，无法实现“增量生成”，大型项目构建需要等待5分钟的生成，对于工程角度是不可接受的。
2. OSO的设计导致其很难人工进行跨机器的传输（涉及到生成时间戳，路径收集，路径映射），且大型项目会有较多的预二进制对象文件，其为FAT Binary含有多个架构，会造成比较高的无用带宽开销

在实际的落地场景中，我们最后选择了在此方案的基础上，大幅度优化OSO的传输开销，“等待耗时”等，最终实现在大型项目中，全调试链路启动从6-8分钟（dSYM Bundle + 穿行传输），优化为2分钟（OSO + 并行传输）的优化效果。

## 跨机器OSO传输处理
我们的解决方案主要侧重于解决开发-调试周期的问题，因此尽量希望从整体视角来看，调试信息的传输能够更快。这可以细分为两个优化：
1. 调试产物本身大小更小（假设带宽一定）
2. 调试产物生成的时间更快，或者说能够“并行生成”来提前传输

### OSO路径映射
上文也提到，最开始尝试了直接利用dSYM Bundle来进行产物传输，也参考了上游和业内的一些实践，包括New DWARFLinker，但是实践下来结果都不够理想。

因此，最后的落脚点放在采取OSO来存储调试信息，并进行优化。首先我们需要保证直接原封不动从编译机器A，传输OSO到用户机器B能够正常工作，根据前文的知识，首先就需要将编码OSO从绝对路径，转为相对路径。

我们尝试在不修改工具链的情况下进行调研，但是结果是令人沮丧的：
1. ld64写入OSO时，虽然支持一个-oso_prefix参数，但是其作用是对所有OSO路径删除一个统一前缀，不能像clang编译器的-fdebug-prefix-map=A=B那样自定义替换为相对路径
![](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2023-12-26/media/17035879956265.jpg)
2. lldb读取解析OSO时，会当作绝对路径去读取，并没有提供直接的路径映射方案
![](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2023-12-26/media/17035880084243.jpg)

既然没有办法直接用相对路径，我们还有另一个思路，就是通过绝对路径来进行映射（避免跨机器的前缀路径问题）。在这方面，我们同时提供了两个实现方案（供复杂系统选择）：

1. 针对能自定义DAP（Debugger Adapt Protool）的场景（云IDE场景）：我们提供了一套VFS机制，能够对任意的绝对路径虚拟映射到本机的某个路径上。可以兼容Apple的LLDB.framework
具体实践是利用fishhook，因为LLDB会以动态库形式加载到DAP进程内存，因此能够通过fishhook解决方案来重定向其文件系统的访问，从而走一层我们自己的映射表：
![](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2023-12-26/media/17035880202585.jpg)
2. 针对不能自定义DAP的场景（分布式构建场景）：我们利用历史文章介绍过我们有自定义的LLDB.framework，在其内部集成了原生的VFS实现，能够读取和LLVM一样的vfsoverlay.yaml文件来映射目录
具体实践是利用LLVM提供的工具类，通过settings set来设置vfsoverlay.yaml路径，提前生成好并在读取OSO相关逻辑时调用。

### OSO大小优化
在解决了传输的路径映射问题之后，另一个优化重点就是如何缩小OSO的大小。我们采取了一个朴素且保守的方案：将OSO（本身是MachO Object）的所有非Debug Info相关的Segment和Section全部清空，并调整符号表和偏移量，让这个MachO Object成为“仅供调试使用的Object”。

此外，当然还有针对FAT Binary的处理，整体功能利用llvm-objcopy，我们实现了不同的裁剪策略（见下），减少了约60-70%的大小原开销。

这样设计的好处是，能够尽量减少对LLDB原生解析逻辑影响（实际LLDB仅改动1行代码），因此为了兼容性我们提供了两个不同的开关，具体行为如下：
- --extract-oso-zero-fill：删除所有Reolcation，对所有非__DWARF Section都进行填0操作，增加压缩率，可搭配--arch
- --extract-oso-strip：删除所有Relocation，所有非__DWARF Section都裁剪，并调整segment.filesize和segment.offset，指定segment.flags为ZERO_FILL，需要搭配内网的LLDB.framework才可以正确解读，可搭配--arch

除了裁剪以外，还自动进行了MachO Universal Binary的Slicing（保留单架构），也不用调用方自己唤起lipo（比较慢）

在大型项目的实践中，整体的OSO传输大小，从优化前的15GB左右，优化为最终的10GB大小，减少幅度高达1/3（取决于项目的预二进制的对象文件多少）

### OSO并行传输

解决了OSO的传输的大小开销后，我们又产生了另一个优化方案：现有的流程提取OSO依赖链接器链接完成，但是实际上，OSO是编译器产出的结果，链接器仅仅做的是“收集并写入路径”。我们能不能自己做一个“仿造链接器”来完成一样的能力，达到并行提前裁剪和传输OSO呢？

答案是肯定的，我们利用Apple开源的ld64代码，结合一些构建系统提供的Build System监控（如Bazel的BEP，Xcode的XCBBuildService），在链接阶段开始的瞬间，并行唤起我们的仿造链接器进程，处理裁剪OSO和触发传输的逻辑。
![screenshot-20231226-185102](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2023-12-26/media/screenshot-20231226-185102.png)

在这样的优化之后，原始需要串行等待的2分钟（链接时间）+2分钟（传输OSO时间），被优化为纯粹的2分钟，优化幅度高达50%。

## 总结

现代构建系统和工具链的日益不断的结合，我们会越来越多涉及到这种类似双向配合才能达到的收益。在这个方案中，我们介绍了如何让调试器，与构建系统的分布式处理，能够协调合一，达到接近本地单机调试的开发体验（但是拥有更高的编译/链接构建速度）。

DanceCC工具链也会后续在更多领域，如编译器、链接器、调试器、LLVM子工具上进行更多的尝试，提供针对移动平台的全套解决方案。
引用链接
1. Apple's Linker & Deterministic Builds：milen.me — Apple’s Linker & Deterministic Builds
2. Apple's Lazy DWARF Scheme：https://wiki.dwarfstd.org/Apple's_%22Lazy%22_DWARF_Scheme.md
3. gdb-remote：https://developer.apple.com/library/archive/documentation/DeveloperTools/gdb/gdb/gdb_33.html
4. ld64：https://github.com/apple-opensource/ld64
5. LLVM VFS：https://llvm.org/doxygen/classllvm_1_1vfs_1_1OverlayFileSystem.html