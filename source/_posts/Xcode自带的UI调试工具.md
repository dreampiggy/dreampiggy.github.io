---
title: Xcode自带的UI调试工具
categories: iOS
tags:
  - iOS
  - Xcode
  - Debug
updated: '2016-05-30 22:43:18'
date: 2016-05-30 15:03:46
---

## Xcode or Reveal

在iOS开发中，UI调试总是一个大问题……原始的暴力调试（View强行着色然后看，或者在@Selector中直接把`viewForLastBaselineLayout`或者`viewForFirstBaselineLayout`）利用`Swizzle` Hook，然后Debug输出相应的内容（比如View的属性，描述符等等）。

这是一个好方法……但是过于暴力，而且非常不直观。虽然iOS开发者都知道的[Reveal](http://revealapp.com)是好东西，然而价格并不是人人都买得起……（不过对iOS专业开发者确实是一个必备的工具）。其实Xcode 7自带了一套UI调试的工具，熟悉使用之后也是非常好用。

## 使用

使用方法很简单，首先自然得运行项目，然后Run(Command R)，然后切换到Debug栏(Command 6)，点击左边栏最右侧的图表，选择`View UI Hierarchy`即可。

![](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-07-14/image/d/b8/3ee5a4bd1289d16d36db566f3bf1e.png)

### UI列表

进入UI Debugger视图后，App会被lldb暂停，左侧为所有当前的UI对象，以及它们的继承顺序。注意，此时的UIView对象按照由上到下，对应了UIView的层级顺序（即UIView.subviews)，最上方为最底层的View（对应subviews atIndex 0)，最下方即当前的顶层视图。点击对应的UIView可以在右侧的图形化视图中看到。

![](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-07-14/image/a/72/a576918da1fc5658bccaab351b669.png)

### 层级预览

右侧的图形化视图，默认下是显示了所有视图以及它们的外边框，并且以实际显示(非3D层次)。点击下方的立方体按钮切换到3D预览（像不像Reveal？）

![](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-07-14/image/5/8b/c05da0308cc5f1b90e1ee80e14756.png)

### 配合检查器

点击控件也可在左侧标记出对象，可以在右侧的Object Inspector来查看它们的属性，还能看到不同View的约束在真实运行时候的表现状态，对于使用大量Autolayout和约束的应用来说非常方便。

![](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-07-14/image/0/4a/c340caefd7bd60c6deb7dd07eef43.png)

## 总结

怎么样，虽然相比Reveal虽然不能实时Debug，但可以用Debug的`Continue exec`来使App恢复再继续Debug……而且有着非常详细的图形化和属性检测，还有官方支持……对于大部分开发者来说基本足够了，也不必真心追求昂贵的Reveal来支持UI Debug。

> 补充：Xcode 7自带的UI Debugger由于在对UIView绘图时，使用了在iOS 9才加入的`viewForFirstBaselineLayout`
而iOS 8上使用的`viewForBaselineLayout`被deprecated，因此默认无法正确调试，如果需要在iOS 8及以下的模拟器或者真机上进行UI Debug，可以参考别人提供一个Swizzle替换

```objectivec
#ifdef DEBUG

#import <UIKit/UIKit.h>
#import <objc/runtime.h>

@implementation UIView (FixViewDebugging)

+ (void)load
{
    Method original = class_getInstanceMethod(self, @selector(viewForBaselineLayout));
    class_addMethod(self, @selector(viewForFirstBaselineLayout), method_getImplementation(original), method_getTypeEncoding(original));
    class_addMethod(self, @selector(viewForLastBaselineLayout), method_getImplementation(original), method_getTypeEncoding(original));
}

@end

#endif
```