---
title: CocoaPods资源管理—Data Asset最低部署版本的坑
date: 2021-07-16 16:16:18
categories: iOS
tags:
  - iOS
  - CocoaPods
---

# 背景

自己很早之前曾经写过一些CocoaPods管理Resource资源的文章：[CocoaPods的资源管理和Asset Catalog优化](https://bytedance.feishu.cn/wiki/wikcnJUiDWMmCbkSnfFvYpcMT0f) ，当时列举了对普通图片类型的管理方式和一些用法，也普及了一下UIImage获取Bundle去加载不在mainBundle图像的方式。

但是苹果早在iOS 9，Xcode 7时代，苹果就已经推出了Data Asset的概念，并在随后的Xcode，尤其是Xcode 10中，为Data Asset提供了App Slicing的能力（即App Store提审包会根据选择的不同设备/内存/分辨率/GPU/CPU，最终下载到唯一匹配的一份文件），这个功能渐渐地开始被一些国内开发者使用。

在NSHipster这里，有一篇专门的文章介绍：《[NSDataAsset](https://nshipster.cn/nsdataasset/)》

不过，这篇文章主要的内容是，最近有同事踩到一个关于Data Asset和最低部署版本的坑，这里单独列举一下以防后人重复踩坑。

# Data Asset初见

标准的配置下，我们可以直接在Xcode里创建一个Asset Catalog，然后拖入想要的文件。注意我们可以在右侧针对不同的配置设置不同的文件内容。

![1625559957403_29bd9b59c2bbaa1f363122a8276779b6](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2021-07-16/image/1625559957403_29bd9b59c2bbaa1f363122a8276779b6.png)

最终一个Data Asset的输入大概的形式是这样子的：

```
Image.xcassets

- A.dataset

-- Contents.json

-- 1.zip

-- 2.webp
```

可以看到除了后缀名以外，其他的结构和普通的imageset保持一致。

# Data Asset产物

在执行Xcode标准的`Copy Bundle Resources`的Build Phase之后，可以看到我们的Data Asset会被编译为一个Assets.car文件，这个格式也是老熟人了。

![1625559957281_7b787078bbae747abaf28cde1a513955](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2021-07-16/image/1625559957281_7b787078bbae747abaf28cde1a513955.png)

# Data Asset获取代码

类似于图像，由于Data Asset最终会编译到Car中，无法直接获取文件路径（Flutter/H5等跨平台库又需要使用Bridge方案来调用Native接口）

在运行时，我们需要使用Fondation提供的专门类[NSDataAsset](https://developer.apple.com/documentation/uikit/nsdataasset?language=objc)相关接口，来获取真正的NSData，接口比较简单直观：

```
/** 如果是非Main Bundle，要获取Bundle

NSString *bundlePath = [[NSBundle bundleForClass:self.class].resourcePath stringByAppendingPathComponent:@"A.bundle"];

NSBundle *bundle = [NSBundle bundleWithPath:bundlePath];

*/

NSBundle *bundle = [NSBundle mainBundle];

NSDataAsset *asset = [[NSDataAsset alloc] initWithName:@"TestImageAnimated" bundle:bundle];

NSData *data = asset.data;
```

看起来比UIImage的相关接口简单理解多了，对吧。

# 坑-最低部署版本影响行为

然而最近有同事发现，他们的一个SDK，使用了Data Asset，在不同的宿主App中行为不一致。某个宿主中可以能访问到数据，另一个一直访问不到。前来咨询（？）了我，因此做了一番排查，发现了一个坑：

**先说结论：Data Asset的编译单元，在最低部署版本iOS 9以下时，不会产出Asset.car而是直接拷贝了文件到原Bundle路径下；只有iOS 9及以上才会产出Asset.car**

如图，这是SDK的资源。SDK使用了CocoaPods进行托管，Podspec里面使用了`resource_bundles`来提供对外的资源。这里的Data Asset里面内容是一个WebP文件。

![1625559957352_30cb4a561799eec3da92fa1c607c101e](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2021-07-16/image/1625559957352_30cb4a561799eec3da92fa1c607c101e.png)


```
s.ios.deployment_target = "8.0"

s.subspec 'Core' do |ss|

  ss.resource_bundle     =  {'splashResourceCore' => ['TTAdSplashSDK/Assets/splashResource/CoreImage.xcassets', 'TTAdSplashSDK/Assets/splashResource/ShakeMusic.mp3']}

end
```

看起来非常正常，但是实际上行为就是有所不同。于是简单开始从源头排查差异。

## 宿主A

我们搜索查看Xcode最终编译的命令。负责编译xcassets的命令是actool。我们可以看到，在`com.apple.actool.compilation-results`这里有打印所有的输出，是符合预期的。

```
CompileAssetCatalog /Users/bytedance/Library/Developer/Xcode/DerivedData/TTAdSplashSDK-bouxjwktwlrwfthejbcmzvcqddie/Build/Products/Debug-iphonesimulator/TTAdSplashSDK-Core-Interactive-Privacy/splashResourceCore.bundle /Users/bytedance/TTiOS/subs/tt_splash_sdk/TTAdSplashSDK/Assets/splashResource/CoreImage.xcassets (in target 'TTAdSplashSDK-Core-Interactive-Privacy-splashResourceCore' from project 'TTAdSplashSDK')

    cd /Users/bytedance/TTiOS/subs/tt_splash_sdk/Example/Pods

    /Applications/Xcode.app/Contents/Developer/usr/bin/actool --output-format human-readable-text --notices --warnings --export-dependency-info /Users/bytedance/Library/Developer/Xcode/DerivedData/TTAdSplashSDK-bouxjwktwlrwfthejbcmzvcqddie/Build/Intermediates.noindex/TTAdSplashSDK.build/Debug-iphonesimulator/TTAdSplashSDK-Core-Interactive-Privacy-splashResourceCore.build/assetcatalog_dependencies --output-partial-info-plist /Users/bytedance/Library/Developer/Xcode/DerivedData/TTAdSplashSDK-bouxjwktwlrwfthejbcmzvcqddie/Build/Intermediates.noindex/TTAdSplashSDK.build/Debug-iphonesimulator/TTAdSplashSDK-Core-Interactive-Privacy-splashResourceCore.build/assetcatalog_generated_info.plist --compress-pngs --enable-on-demand-resources NO --optimization space --filter-for-device-model iPhone13,2 --filter-for-device-os-version 14.5 --development-region en --target-device iphone --target-device ipad --minimum-deployment-target 10.0 --platform iphonesimulator --compile /Users/bytedance/Library/Developer/Xcode/DerivedData/TTAdSplashSDK-bouxjwktwlrwfthejbcmzvcqddie/Build/Products/Debug-iphonesimulator/TTAdSplashSDK-Core-Interactive-Privacy/splashResourceCore.bundle /Users/bytedance/TTiOS/subs/tt_splash_sdk/TTAdSplashSDK/Assets/splashResource/CoreImage.xcassets

    

/* com.apple.actool.compilation-results */

/Users/bytedance/Library/Developer/Xcode/DerivedData/TTAdSplashSDK-bouxjwktwlrwfthejbcmzvcqddie/Build/Intermediates.noindex/TTAdSplashSDK.build/Debug-iphonesimulator/TTAdSplashSDK-Core-Interactive-Privacy-splashResourceCore.build/assetcatalog_generated_info.plist

/Users/bytedance/Library/Developer/Xcode/DerivedData/TTAdSplashSDK-bouxjwktwlrwfthejbcmzvcqddie/Build/Products/Debug-iphonesimulator/TTAdSplashSDK-Core-Interactive-Privacy/splashResourceCore.bundle/Assets.car
```

检索产物Assets.car，也符合预期：

![1625559957323_ceced1da15007185b48893a6eda48754](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2021-07-16/image/1625559957323_ceced1da15007185b48893a6eda48754.png)

## 宿主B

同样的，我们查看编译命令：

```
CompileAssetCatalog /Users/bytedance/Library/Developer/Xcode/DerivedData/TTAdSplashSDK-bouxjwktwlrwfthejbcmzvcqddie/Build/Products/Debug-iphonesimulator/TTAdSplashSDK-Core-Interactive-Privacy/splashResourceCore.bundle /Users/bytedance/TTiOS/subs/tt_splash_sdk/TTAdSplashSDK/Assets/splashResource/CoreImage.xcassets (in target 'TTAdSplashSDK-Core-Interactive-Privacy-splashResourceCore' from project 'TTAdSplashSDK')

    cd /Users/bytedance/TTiOS/subs/tt_splash_sdk/Example/Pods

    /Applications/Xcode.app/Contents/Developer/usr/bin/actool --output-format human-readable-text --notices --warnings --export-dependency-info /Users/bytedance/Library/Developer/Xcode/DerivedData/TTAdSplashSDK-bouxjwktwlrwfthejbcmzvcqddie/Build/Intermediates.noindex/TTAdSplashSDK.build/Debug-iphonesimulator/TTAdSplashSDK-Core-Interactive-Privacy-splashResourceCore.build/assetcatalog_dependencies --output-partial-info-plist /Users/bytedance/Library/Developer/Xcode/DerivedData/TTAdSplashSDK-bouxjwktwlrwfthejbcmzvcqddie/Build/Intermediates.noindex/TTAdSplashSDK.build/Debug-iphonesimulator/TTAdSplashSDK-Core-Interactive-Privacy-splashResourceCore.build/assetcatalog_generated_info.plist --compress-pngs --enable-on-demand-resources NO --optimization space --filter-for-device-model iPhone13,2 --filter-for-device-os-version 14.5 --development-region en --target-device iphone --target-device ipad --minimum-deployment-target 8.0 --platform iphonesimulator --compile /Users/bytedance/Library/Developer/Xcode/DerivedData/TTAdSplashSDK-bouxjwktwlrwfthejbcmzvcqddie/Build/Products/Debug-iphonesimulator/TTAdSplashSDK-Core-Interactive-Privacy/splashResourceCore.bundle /Users/bytedance/TTiOS/subs/tt_splash_sdk/TTAdSplashSDK/Assets/splashResource/CoreImage.xcassets



/* com.apple.actool.compilation-results */

/Users/bytedance/Library/Developer/Xcode/DerivedData/TTAdSplashSDK-bouxjwktwlrwfthejbcmzvcqddie/Build/Intermediates.noindex/TTAdSplashSDK.build/Debug-iphonesimulator/TTAdSplashSDK-Core-Interactive-Privacy-splashResourceCore.build/assetcatalog_generated_info.plist

/Users/bytedance/Library/Developer/Xcode/DerivedData/TTAdSplashSDK-bouxjwktwlrwfthejbcmzvcqddie/Build/Products/Debug-iphonesimulator/TTAdSplashSDK-Core-Interactive-Privacy/splashResourceCore.bundle/Assets.car

/Users/bytedance/Library/Developer/Xcode/DerivedData/TTAdSplashSDK-bouxjwktwlrwfthejbcmzvcqddie/Build/Products/Debug-iphonesimulator/TTAdSplashSDK-Core-Interactive-Privacy/splashResourceCore.bundle/ad_btn_hand.webp

/Users/bytedance/Library/Developer/Xcode/DerivedData/TTAdSplashSDK-bouxjwktwlrwfthejbcmzvcqddie/Build/Products/Debug-iphonesimulator/TTAdSplashSDK-Core-Interactive-Privacy/splashResourceCore.bundle/ad_btn_triangle.webp
```

此时，在actool的编译结果中，我们发现，原本预期应该在Data Asset的`ad_btn_hand.webp`和`ad_btn_triangle.webp`两个文件，竟然直接拷贝到了.bundle的根路径，而不是Assets.car中！

![1625559957272_fa17b69e9dd37090291bc0a6952baa38](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2021-07-16/image/1625559957272_fa17b69e9dd37090291bc0a6952baa38.png)

对比两者的命令，只有`--minimum-deployment-target`这一项有差距，宿主A是iOS 10.0，宿主B是iOS 8.0。

经过再次Demo验证，确定了是这个导致了行为的差异！

## SDK调用代码

SDK运行时需要获取这些代码，经过查看，这里的代码是假设按照.bundle根路径存在Data Asset的文件名的方式去取的，因此在宿主A中会出现异常：

```
// 伪代码

NSString *bundlePath = [[NSBundle bundleForClass:TTAdSplashManager.class].resourcePath stringByAppendingPathComponent:@"splashResourceCore.bundle"];

NSbundle* bundle = [NSBundle bundleWithPath:bundlePath];



NSString *trianglePath = [bundle.resourcePath stringByAppendingPathComponent:@"ad_btn_triangle.webp"];

NSData *triangleData = [NSData dataWithContentsOfFile:trianglePath];

self.imageView.image = [UIImage imageWithData:triangleData];
```

## 进一步排查最低部署版本变化

本质原因了解清楚后，进一步排查这个疑问：

> 为什么宿主A和宿主B，对于一个SDK的Pod，最低部署版本不一致？

因为SDK的Podspec的最低部署版本已经指明了iOS 8，按理说在哪个宿主集成都应该走的是路径的逻辑，而不应该受限于宿主iOS App自己的编译最低部署版本。

查看宿主A，发现宿主A使用了CocoaPods的插件，在Pod Project Generate的时候，强制修改了所有Pod，伪代码如下：

```
all_targets.each do |target|

  target.set_build_settings('IPHONEOS_DEPLOYMENT_TARGET') do |_, old|

    old.to_f < 10.0 ? '10.0' : old

  end

  target.set_build_settings('ASSETCATALOG_COMPILER_OPTIMIZATION') do |_, old|

    definitions = 'space'

    definitions

  end

end
```

导致SDK的编译Assets.car时，`--minimum-deployment-target`传入了iOS 10.0，Data Asset编译到Assets.car里

而宿主B，并没有这个逻辑，按照iOS 8.0传入，Data Asset散落在Bundle根路径。

# 结论

从这个坑可以看到，最低部署版本，这个编译配置，设置时需要谨慎。由于iOS App不会针对不同的部署版本，单独打一份独立的ipa包（类似PC等平台），所以很多工具链对针对最低部署版本，有着可能不同的兼容性行为，iOS系统快速迭代的节奏下尤其是这样。

这里有两个改进方案：

1.  对于宿主，除非你清楚知道改变最低部署版本的副作用，否则要慎重处理外部Pod的最低部署版本，建议在修改后进行一定的回归测试，或者针对白名单来进行修改。
1.  对于SDK作者，如果没有用到Data Asset的特性（App Slicing），可以考虑直接不用Data Asset而直接放到Bundle中，省去踩坑的问题。如果需要利用Data Asset，并且你无法保证引入方宿主会对你的Pod做额外的修改，可以考虑这种兼容代码来判断：

```
NSString *bundlePath = [[NSBundle bundleForClass:self.class].resourcePath stringByAppendingPathComponent:@"Image.bundle"];

NSBundle *bundle = [NSBundle bundleWithPath:bundlePath];

// 如果编译时的最低部署版本iOS 9以上，Data Asset需要用NSDataAsset类获取，否则用直接取路径

NSDataAsset *asset = [[NSDataAsset alloc] initWithName:@"TestImageAnimated" bundle:bundle]; // 此处是Asset名，不是文件名！

NSData *data = asset.data;

if (!data) {

    // Fallback到路径

    data = [NSData dataWithContentsOfFile:[bundlePath stringByAppendingPathComponent:"TestImageAnimated.webp"]]; // 此处是文件名，注意！

}
```