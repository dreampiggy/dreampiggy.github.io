---
title: iOS端矢量图解决方案汇总（SVG篇）
date: 2020-03-30 19:01:33
categories: iOS
tags:
 - iOS
 - SVG
 - Image
 - Swift
---

# iOS端矢量图解决方案汇总（SVG篇）
## 简介

[矢量图](https://en.wikipedia.org/wiki/Vector_graphics)，指的是通过一系列数学描述，能够进行无损级别的变化和缩放的一种图像。相比于标量图（如JPEG等标量图压缩格式），能够在绘制时进行任意大小伸缩而不产生模糊，甚至能够实现动态着色，动画等等一系列交互。

![intro_raster_to_vecto](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2020-03-30/image/intro_raster_to_vector.png)


在当今移动端设备尺寸越来越复杂，各种操作系统级别的夜间主题（或者Dark Mode）越来越提倡的场景下，如果依旧使用标量图，我们需要针对不同的屏幕大小（如2x，3x），和对应主题场景（Light/Dark），提供NxM数量级的标量图，对于App大小开销是很大的。因此，使用矢量图是一个非常有效的解决方案。这个系列文章，就是主要侧重讲解iOS端上的矢量图解决方案。

第一章是关于[SVG](https://en.wikipedia.org/wiki/Scalable_Vector_Graphics)及其相应衍生方案的解决方案，后续会有其他矢量图相关的PDF章节，Lottie等。他们各自有不同的细节场景区分和优缺点。

SVG作为目前在Web上最流行的矢量格式，在iOS端的支持可以说是一言难尽。在这里，我从各个方向上总结了截至目前已有的实现（公开的方案，企业内部实现无从得知），方便对比选择最适合自己场景的选择。

## Symbol Image

[Symbol Image](https://developer.apple.com/videos/play/wwdc2019/206/)，是Apple在WWDC 2019和iOS 13上提供的矢量图解析方案。

之所以名称叫做Symbol Image，源自于这个技术方案的实现细节，它最早诞生于SVG字体规范：[OpenType-SVG](https://helpx.adobe.com/fonts/using/ot-svg-color-fonts.html)。这个规范是Adobe提出的，并且得到了包括Microsoft在内的多家公司支持。Apple自己的CoreText字体框架，其实早早就在[iOS 11时代](https://developer.apple.com/documentation/coretext/1524658-anonymous/kctfonttablesvg?language=objc)内部支持了SVG类型的font table。

### 制作Symbol Image

Symbol Image的整体API设计，其实不像是图像，更像是一种字体（和Icon Font类似）。

对于同一个Symbol Image，它可以看作是一个SVG Path的集合。前面提到，Symbol Image基于OpenType-SVG字体，对于字体来说，我们都知道字重的概念，用来决定渲染时候的线条粗细程度。

因此Symbol Image也有9个字重：Ultralight，Thin，Light，Regular，Medium，Semibold，Bold，Heavy，Black。与此同时，Symbol Image对每一个字重，支持了3种大小，分别是Small，Medium和Large。这也就是说，一个Symbol Image最多可以有27种大小字重的样式选择。

一般来说，从头构建一个Symbol Image会非常复杂，Apple推荐的方式，是通过使用[SF Symbols App](https://developer.apple.com/design/human-interface-guidelines/sf-symbols/overview/)，来导出一个SVG模版，再通过Sketch来进行图层编辑。

![Sketch2](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2020-03-30/image/Sketch2.png)


从原始的SVG数据来看，每一个Symbol Image包含的所有样式都是一个单独的Path节点，对应了图标的绘制。如果要新建一个Symbol Image，需要完全删除Path节点，重新绘制矢量路径。

```xml
<svg version="1.1" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" width="3300" height="2200">
 <!--glyph: "uni100665.medium", point size: 100.000000, font version: "Version 15.0d7e11", template writer version: "5"-->
 <g id="Notes">
 </g>
 <g id="Guides">
  <g id="H-reference" style="fill:#27AAE1;stroke:none;" transform="matrix(1 0 0 1 339 696)">
<path d="M 54.9316 0 L 57.666 0 L 30.5664 -70.459 L 28.0762 -70.459 L 0.976562 0 L 3.66211 0 L 12.9395 -24.4629 L 45.7031 -24.4629 Z M 29.1992 -67.0898 L 29.4434 -67.0898 L 44.8242 -26.709 L 13.8184 -26.709 Z"/>
 </g>
 <g id="Symbols">
  <g id="Medium-M" transform="matrix(1 0 0 1 1682.22 1126)">
   <path d="M 64.3555 -18.6035 C 67.8223 -18.6035 69.9219 -20.752 70.0195 -24.3164 C 70.166 -40.2832 70.5078 -59.7656 70.6543 -75.1953 C 70.6543 -78.7598 67.9688 -81.3477 64.3555 -81.3477 C 60.6934 -81.3477 58.0078 -78.7598 58.0078 -75.1953 C 58.1543 -59.7656 58.4961 -40.2832 58.6914 -24.3164 C 58.7891 -20.752 60.8887 -18.6035 64.3555 -18.6035 Z M 17.1875 -41.0645 C 18.1641 -40.0391 19.7266 -40.0879 20.7031 -41.1621 C 29.2969 -50.2441 39.9414 -56.0547 51.8066 -58.3984 L 51.6113 -73.3398 C 34.8145 -70.4102 19.5801 -61.7676 10.498 -50.7812 C 9.76562 -49.9023 9.76562 -48.6816 10.6445 -47.7539 Z M 108.057 -41.1133 C 108.984 -40.1367 110.498 -40.1367 111.523 -41.2109 L 117.969 -47.7539 C 118.896 -48.6816 118.896 -49.9023 118.164 -50.7812 C 108.984 -61.6699 93.7988 -70.3613 77.0508 -73.291 L 76.9043 -58.3496 C 88.7695 -56.0059 99.3164 -50.0488 108.057 -41.1133 Z M 36.6699 -21.5332 C 37.793 -20.4102 39.2578 -20.5078 40.2832 -21.7285 C 43.457 -25.1465 47.7051 -28.0273 52.3926 -29.7852 L 51.9531 -45.4102 C 42.4805 -43.0664 34.375 -37.9883 29.1992 -31.8359 C 28.3691 -30.8594 28.4668 -29.6875 29.3457 -28.8086 Z M 88.5254 -21.5332 C 89.5508 -20.459 90.8691 -20.5078 91.9922 -21.582 L 99.2676 -28.8086 C 100.195 -29.6875 100.293 -30.8594 99.4629 -31.8359 C 94.2871 -37.9395 86.1816 -43.0176 76.709 -45.4102 L 76.3184 -29.6875 C 81.0547 -27.8809 85.3516 -24.9512 88.5254 -21.5332 Z M 64.3555 6.25 C 69.043 6.25 72.8516 2.53906 72.8516 -2.09961 C 72.8516 -6.73828 69.043 -10.4492 64.3555 -10.4492 C 59.668 -10.4492 55.8105 -6.73828 55.8105 -2.09961 C 55.8105 2.53906 59.668 6.25 64.3555 6.25 Z"/>
  </g>
 </g>
</svg>
```


### 导入Symbol Image

导入Symbol Image的方式非常简单，你只需要将制作好的Symbol Image，向Xcode的Asset Catalog窗口拖动，就可以集成。Xcode可以会展示对应的预览效果。

![截屏2020-03-30下午6.08.56](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2020-03-30/image/%E6%88%AA%E5%B1%8F2020-03-30%E4%B8%8B%E5%8D%886.08.56.png)


另外，实际上产生的文件夹后缀为`.symbolset`，这个不同于普通的Asset Image（后缀名`.imageset`），也就意味着你可以同时引入一个同名的Symbol Image和普通Image。

![截屏2020-03-30下午6.09.18](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2020-03-30/image/%E6%88%AA%E5%B1%8F2020-03-30%E4%B8%8B%E5%8D%886.09.18.png)

### 使用Symbol Image

对于iOS 13系统提供的自带Symbol Image，UIKit提供了[init(systemName:)](https://developer.apple.com/documentation/uikit/uiimage/3294233-init)方法来获取，对于App自行提供的Symbol Image，我们使用[init(named:)](https://developer.apple.com/documentation/uikit/uiimage/1624146-init)方法。

注意，你可以同时包含一个Symbol Image和普通的Asset Image，共享一个Name。这样设计的好处，在WWDC上有介绍，是为了兼容iOS 12等低系统版本，在iOS 13上，Symbol Image优先级永远高于普通Asset Image，在iOS 12会自动fallback。

```swift
let imageView = UIImageView()
let symbolImage = UIImage(named: "my.symbol.image")
// 默认配置下，这个symbol image是template的，意味着他不会含有颜色，颜色由UIView级别tintColor决定
imageView.image = symbolImage

// 如果确定要获取系统Symbol Image
let systemSymbolImage = UIImage(systemName: "wifi.exclamationmark")

// 如果要指定颜色
let redSymbolImage = symbolImage.withTintColor(.red, renderingMode: .alwaysOrigin)
imageView.image = redSymbolImage
```

对于Symbol Image来说，我们可以指定在运行时需要的字重

```swift
let regularSymbolImage = UIImage(named: "my.symbol.image")
// 指定你想要的字号，字重，这里是18号，Bold 字重，Large 大小
let symbolConfiguration = UImage.SymbolConfiguration(pointSize: 18, weight: .large, scale: .large)
let boldSymbolImage = regularSymbolImage.applyingSymbolConfiguration(symbolConfiguration)
imageView.image = boldSymbolImage
```

另外，我们还可以配合AttributedString使用，只要使用TextAttachment传入对应的Symbol Image即可。

```swift
let textView = UITextView()
// 可以微调Symbol Image与文字的对齐
let baselineSymbolImage = symbolImage.withBaselineOffset(fromBottom: 1.0)
let imageAttachment = NSTextAttachment(image: baselineSymbolImage)
let imageString = NSAttributedString(attachment: imageAttachment)
textView.attributedText = imageString
```

### 优缺点

优点：

+ iOS原生支持，工具链完善
+ SwiftUI原生支持，截止目前Image能唯一使用的矢量方案（排除UIViewRepresentable）
+ 支持和AttributedString无缝混合，类似Icon Font

缺点：

+ iOS 13+ Only
+ 通过字体属性控制大小，取决于UI场景，做到Pixel级别的拉伸会是一个问题
+ 需要单独制作Symbol Image，跨平台，Web使用痛点

## CoreSVG

CoreSVG是iOS 13支持Symbol Image的背后的底层SVG渲染引擎，使用C++编写。

截至目前，CoreSVG依然属于Private Framework，社区也有很多人向Apple提了反馈并建议开放出来，可能在之后的WWDC 2020我们能够得知更多的消息。

注意！以下方法均为使用了CoreSVG的Private API，可能随着操作系统变动会有改变，并且有审核风险，如果需要线上使用，请自行进行代码混淆等方案。

### 通过Asset Catalog使用SVG

目前Xcode不支持直接拖动SVG文件来集成到Asset Catalog，因为拖动SVG默认会当作Symbol Image处理。

但是我们可以通过一个取巧的方式来实现，Xcode支持PDF矢量图（从iOS 11与Xcode 9开始支持，PDF章会讲解）。因此，我们可以将SVG后缀改成PDF，然后拖动到Xcode中，最后再修改回SVG后缀名，并且同步`.imageset/Contents.json`里面的文件名即可，如下：

![EUR_hKSUwAA1-65](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2020-03-30/image/EUR_hKSUwAA1-65.png)

当你添加好SVG图像后，可以通过Name，以和PDF矢量图一样的方式来引入和使用，如下

```objectivec
UIImageView *imageView = [UIImageView new];
UIImage *svgImage = [UIImage imageNamed:@"my_svg"];
imageView.image = svgImage;
// 然后我们可以自由缩放ImageView的大小，会自动触发矢量绘制
imageView.frame = CGRectMake(0, 0, 1000, 1000);
```

从运行时来看，加入Asset Catalog的SVG矢量图的UIImage，含有对应的CGSVGDocumentRef对象，并且也包含了一个标量图的缩略图，可以供缩略图或者其他系统API来调用。并且在Xcode的Interface Builder上也会有明显的SVG标识（类似PDF）

![EUU_DLPU8AM5KHD](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2020-03-30/image/EUU_DLPU8AM5KHD.jpeg)


### 加载任意SVG数据（网络）

除了能够通过Asset Catalog添加SVG图像，通过CoreSVG，我们可以在运行时去解析网络数据下载得到的SVG数据，为此能提供更为广阔的应用场景。

```objectivec
UIImageView *imageView = [UIImageView new];
NSData *data;
CGSVGDocumentRef document = CGSVGDocumentCreateFromData((__bridge CFDataRef)data, NULL);
UIImage *svgImage = [UIImage _imageWithCGSVGDocument:document];
imageView.image = svgImage;
```

### 渲染SVG矢量图到标量图

一些UIKit的视图，或者一些图像处理，对矢量图支持并没有考虑，或者是我们在做性能优化时，需要将矢量图光栅化得到对应的标量图。CoreSVG提供了和CoreGraphics的PDF类似的接口，允许你去绘制得到对应的标量图。

```objectivec
CGSVGDocumentRef document; // 原始SVG Document
CGSize targetSize; // 指定标量图大小
BOOL preserveAspectRatio; // 是否保持宽高比

// 获取SVG的canvas大小，本质上是按照SVG规范，将viewPort和viewBox计算得出的
CGSize size = CGSVGDocumentGetCanvasSize(document);
// 计算Transform
CGFloat xRatio = targetSize.width / size.width;
CGFloat yRatio = targetSize.height / size.height;
CGFloat xScale = preserveAspectRatio ? MIN(xRatio, yRatio) : xRatio;
CGFloat yScale = preserveAspectRatio ? MIN(xRatio, yRatio) : yRatio;
    
CGAffineTransform scaleTransform = CGAffineTransformMakeScale(xScale, yScale);
CGSize scaledSize = CGSizeApplyAffineTransform(size, scaleTransform);
CGAffineTransform translationTransform = CGAffineTransformMakeTranslation(targetSize.width / 2 - scaledSize.width / 2, targetSize.height / 2 - scaledSize.height / 2);
// 开始CGContext绘制
UIGraphicsBeginImageContextWithOptions(targetSize, NO, 0);
CGContextRef context = UIGraphicsGetCurrentContext();
// UIKit坐标系和CG坐标系转换
CGContextTranslateCTM(context, 0, targetSize.height);
CGContextScaleCTM(context, 1, -1);
// 应用Transform   
CGContextConcatCTM(context, translationTransform);
CGContextConcatCTM(context, scaleTransform);
// 绘制SVG Document
CGContextDrawSVGDocument(context, document);
// 获取标量图
image = UIGraphicsGetImageFromCurrentImageContext();
UIGraphicsEndImageContext();
```

### SVG导出

目前，CoreSVG没有提供类似于PDF的修改元素的接口，我们只能直接对SVGDocument进行导出。或许随着未来框架的开放，会有类似于目前CoreGraphics对PDF进行编辑的高级接口。

```objectivec
// 获取SVG Document
UIImage *svgImage;
CGSVGDocumentRef document = [svgImage _CGSVGDocument];
NSURL *url = [NSURL fileURLWithPath:@"/tmp/output.svg"];
NSMutableData *data = [NSMutableData data];
// 导出到Data
CGSVGDocumentWriteToData(document, (__bridge CFMutableDataRef)data, NULL);
// 或者文件
CGSVGDocumentWriteToURL(document, (__bridge CFURLRef)url, NULL);
```

### 优缺点 

优点

+ 能够支持目前已有的大量SVG，在Android和Web端复用
+ Apple原生支持，稳定性有一定保证，并且随系统升级会持续优化
+ 性能高，CoreSVG利用了CoreGraphics系统库和内部的SPI做矢量绘制，目前性能最好

缺点
+ 目前是私有Framework，有审核和使用风险
+ 可能存在一些SVG元素兼容问题，需要不断摸索
+ SwiftUI不支持，需要使用UIViewRepresentable

## 三方SVG库

### [SVGKit](https://github.com/SVGKit/SVGKit)

SVGKit是最早的iOS上开源SVG渲染方案，已经有8年之久。SVGKit内部支持两种渲染模式，一种是通过CPU渲染（CoreGraphics重绘制），一种是通过GPU渲染（CALayer树组合）。有着不同的兼容性和性能。

示例

```objectivec
// CPU渲染
SVGKImageView *imageView = [SVGKFastImageView new];
// GPU渲染
imageView = [SVGKLayeredImageView new];
SVGKImage *svgImage = [[SVGKImage alloc] initWithData:data];
imageView.image = svgImage;
```

优点

+ 支持纯Objective-C
+ 如果是支持的图像，性能相对较高（1000个级别的Path可在1秒内渲染）

缺点

+ 社区不再维护，大量Issue无人跟进解决
+ 不遵循语义版本号，用分支发布更新，下游无法依赖
+ 部分SVG特性虽然声明支持，但存在问题，如Gradient等，缺少单测
+ 不支持SVG动画

### [Macaw](https://github.com/exyte/Macaw)

Macaw是一个矢量绘制框架，提供了非常简单的DSL语法来描述矢量路径绘制的场景。它本身不是和SVG强绑定的，但是对SVG格式提供了兼容和支持

示例

```swift
let node = try! SVGParser.parse(path: "/path/to/svg")
let imageView = SVGView()
imageView.node = node
```

优点

+ 目前最活跃和成熟的iOS端SVG开源框架（在GitHub上）
+ 支持DSL去直接生成矢量图，修改节点等，非常强大
+ 支持SVG动画（部分特性）

缺点

+ 部分SVG特性特性声明不支持
+ SVG性能渲染差（相对于SVGKit），依赖大量的的CPU绘制操作（非CALayer组合），可能需要结合异步绘制框架

### [SwiftSVG](https://github.com/mchoe/SwiftSVG)

SwiftSVG是一个专门针对SVG Path等常见特性的矢量图解析框架，他不侧重于完整的SVG/1.1规范支持，而是保证了基本的绘制实现的正确性，并且支持导出SVG的Path到UIBezierPath

示例

```swift
let svgURL = URL(string: "https://openclipart.org/download/181651/manhammock.svg")!
let hammock = UIView(SVGURL: svgURL) { (svgLayer) in
    svgLayer.fillColor = UIColor(red:0.52, green:0.16, blue:0.32, alpha:1.00).cgColor
    svgLayer.resizeToFit(self.view.bounds)
}
self.view.addSubview(hammock)
```

优点

+ 性能相对MacPaw较好
+ 对Path，Circle等常见元素，有着良好的兼容性和完整单测，基本上只用这些特性的SVG不存在问题
+ 支持导出UIBezierPath，可以用作一些描边的交互
+ 提供了便携方法，能直接读取Xcode的Data Asset，URL等

缺点

+ 基本上只针对Path，Circle等元素有良好的支持，其他的Gradient，Text等均不支持
+ 不支持SVG动画

## VectorDrawable

[VectorDrawable](https://developer.android.com/guide/topics/graphics/vector-drawable-resources)是Android平台上官方提供的一套矢量图解决方案，他是以一个类似SVG的XML表达形式，来描述矢量图的绘制方式。

![截屏2020-03-30下午5.44.59](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2020-03-30/image/%E6%88%AA%E5%B1%8F2020-03-30%E4%B8%8B%E5%8D%885.44.59.png)


从整体设计上看，VectorDrawable基本上是对SVG的精简和二次改造，大部分的元素在SVG中都有对应的概念，并且样式属性也一一对应。甚至，Android Studio支持直接将SVG导出成VectorDrawable文件并直接集成。

在iOS上平台上，Uber内部开源了一套自己在用的VectorDrawable实现：[Cyborg](https://github.com/uber/cyborg)，通过利用CoreGraphics和CoreAnimation来渲染VectorDrawable文件。

### 使用VectorDrawable渲染

VectorDrawable提供了一个专门用于矢量图的View，并且能够制定对应的Theme（Theme是用来支持不同资源的Dark Mode切换的）。

```swift
// Bundle加载
let vectorView = VectorView(theme: myTheme)
vectorView.drawable = VectorDrawable.named("MyDrawable")

// Data加载
vectorView.drawable = VectorDrawable.create(from: data)
```

如果这个不满足，你也可以通过CALayer来做渲染，做更为细致的调节。并且VectorDrawable也提供了一些定制项（如设置tintColor）

### 优缺点

优点

+ 能够和Android端复用，并且由于可由SVG生成，意味着Web端也可复用设计资源
+ 性能良好，无论官方还是Example测试，除去CoreSVG外都是最快的渲染速度

缺点

+ 目前iOS实现不支持动画（AnimatedVectorDrawable）
+ 部分SVG实现VectorDrawable不支持，需要设计资源修改
+ Uber内部开源，可能存在未来持续社区建设和维护成本，需要评估

## SVG-Native

[SVG-Native](https://svgwg.org/specs/svg-native/)是由Adobe主导提出的一个W3C规范，目前处于Draft Stage，不过由于Apple，Google的赞同，大概率会在2020年内通过，并且正式规范定稿。

SVG-Native基于目前的SVG/1.1版本，是SVG/1.1的真子集（即一个SVG-Native图一定可以被浏览器正确渲染）。

注：曾经W3C有一个SVG Tiny的规范，但是它是针对移动浏览器场景的，和SVG-Native解决的问题是不一样的。

它针对移动平台，桌面平台等非浏览器场景做了针对性定制，废弃了一些Native端非常困难实现的功能，包括：

+ scripting: 不依赖JavaScript环境
+ animations: 不支持动画
+ filters: 不支持滤镜，部分效果（如文字滤镜）依赖实现复杂
+ masks: 不支持蒙层
+ patterns: 不支持仿制图章，Color Pattern
+ texts: 不内嵌文字，文字使用Path绘制
+ events: 点击事件等，因为没有Script交互自然不需要
+ CSS3：CSS3是一个完整布局系统，大量属性远远超过SVG的功能，如Flexbox，Media-Query，都是不必要的，只有基本的渲染属性

可以看出，这些剥离的功能都是和浏览器场景完全绑定的，不适用于通用的App内渲染矢量图的用途。SVG-Native更适合桌面/移动的App，渲染器实现也会精简很多，容易单元测试，并且可供操作系统内嵌集成。

### 使用

Adobe提供了一个目前Draft规范的渲染实现[SVG Native Viewer](https://github.com/adobe/svg-native-viewer)，目前提供了多种渲染引擎的桥接，包括我们熟悉的CoreGraphics和Skia。

SVG-Native解码器，能够以标量图的方式，渲染SVG到一个指定大小的CGContext上，性能目前看足够快（和CoreSVG对比）。目前一般是通过重写drawRect来让View大小变化时进行重绘。

```objectivec
- (void)drawRect:(NSRect)dirtyRect {
    [super drawRect:dirtyRect];
    
    Document* d = [[[self window] windowController] document];
    SVGNative::SVGDocument* doc = [d getSVGDocument];
    if (!doc)
        return;
    NSGraphicsContext* nsGraphicsContext = [NSGraphicsContext currentContext];
    CGContextRef ctx = (CGContextRef) [nsGraphicsContext CGContext];
    SVGNative::CGSVGRenderer* renderer = static_cast<SVGNative::CGSVGRenderer*>(doc->Renderer());
    CGRect r(dirtyRect);
    CGAffineTransform m = {1.0, 0.0, 0.0, -1.0, 0.0, r.size.height};
    CGContextConcatCTM(ctx, m);
    renderer->SetGraphicsContext(ctx);
    doc->Render(r.size.width, r.size.height);
    renderer->ReleaseGraphicsContext();
}
```

### 优缺点

优点

+ W3C规范，可以确保未来规范的准确性，并且操作系统提供商，如Apple更容易集成
+ SVG-Native是SVG1.1的真子集，意味者可以复用到Web上
+ SVG-Native会是未来的OpenType-SVG实现，意味着Adobe字体或者设计师群体更容易接受

缺点

+ SVG-Native是SVG真子集，意味着目前的SVG设计资源，需要适配修改才可支持
+ 截至目前，SVG-Native依然处于Draft阶段，稳定，推广普及需要较长时间
+ SVG-Native目前只有Adobe的解析器实现，部分特性在CoreGraphics上工作并不良好
+ 目前没有看到动画的支持

## 总结

总结一下关于SVG的相关解决方案，可以看出，没有一种Case能够涵盖所有场景，当然，这和Apple本身对矢量图支持的建设有一定关系，大部分建设依赖于开源社区。因此，通常情况下需要根据自己具体的实际需要来选择，比如：

+ 只考虑Path，Circle等矢量路径：使用SwiftSVG、Macaw即可
+ 考虑和Android复用：使用VectorDrawable
+ 不考虑iOS 13以下兼容：优先用Symbol Image和CoreSVG
+ 考虑SVG动画：Macaw
+ 面向未来：SVG-Native

## 参考资料

+ [解读 WWDC19 - SF Symbols 内置图标库](https://swiftcafe.io/post/sf-symbol)
+ [SF Symbols: The benefits and how to use them guide](https://www.avanderlee.com/swift/sf-symbols-guide/)
+ [SVG: Scalable Vector Graphics](https://developer.mozilla.org/en-US/docs/Web/SVG)
+ [SDWebImageSVGCoder](https://github.com/SDWebImage/SDWebImageSVGCoder)
+ [Vector drawables overview](https://developer.android.com/guide/topics/graphics/vector-drawable-resources)
+ [Introducing Cyborg, an Open Source iOS Implementation of Android VectorDrawable](https://eng.uber.com/cyborg/)
+ [OpenType-SVG color fonts](https://helpx.adobe.com/fonts/using/ot-svg-color-fonts.html)
+ [SVG Native: Open Sourcing SVG Native Viewer](https://medium.com/adobetech/svg-native-open-sourcing-svg-native-viewer-988125328a07)