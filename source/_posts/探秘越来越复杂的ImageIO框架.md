---
title: 探秘越来越复杂的ImageIO框架
date: 2022-11-07 23:45:00
categories: iOS
tags:
- Image
- ImageIO
- AVIF
- WebP
---

> ImageIO是Apple提供的上层框架，用于处理常见图像格式的编解码支持。这篇文章主要讲述了三个子话题：WebP/AVIF的支持进展，IOSurafce和硬件解码优化50%内存开销，以及CGImageSource机制变化导致的线程安全问题

ImageIO的定位是上层的支持框架，其封装了诸多的苹果的底层解码器，开源编解码器，硬件HEVC/ProRes加速器等等底层细节，致力于提供和上层UI框架（如UIKit/CoreGraphics）的可交互性。

在早些年的时候，我写过一系列文章，介绍了其API使用的基本流程（参考：[《iOS平台图片编解码入门教程（Image/IO篇）》](https://dreampiggy.com/2017/10/30/iOS%E5%B9%B3%E5%8F%B0%E5%9B%BE%E7%89%87%E7%BC%96%E8%A7%A3%E7%A0%81%E5%85%A5%E9%97%A8%E6%95%99%E7%A8%8B%EF%BC%88Image:IO%E7%AF%87%EF%BC%89)），以及有关其惰性解码的机制（参考：[《主流图片加载库所使用的预解码究竟干了什么》](https://dreampiggy.com/2019/01/18/%E4%B8%BB%E6%B5%81%E5%9B%BE%E7%89%87%E5%8A%A0%E8%BD%BD%E5%BA%93%E6%89%80%E4%BD%BF%E7%94%A8%E7%9A%84%E9%A2%84%E8%A7%A3%E7%A0%81%E7%A9%B6%E7%AB%9F%E5%B9%B2%E4%BA%86%E4%BB%80%E4%B9%88)）。

实话说，自从重心从iOS开发，转移到做LLVM工具链相关工作之后，我本以为不会再写这些上层iOS框架的文章了，但是[SDWebImage](https://github.com/SDWebImage/SDWebImage)这个开源库依旧没有合我预期的新Maintainer，来作为交接，因此现在还是忍不住先写这一篇吐槽和说明文章。

这篇文章会介绍，自iOS 13时代之后，苹果在ImageIO上做的一系列优化（“机制变化”），以及对开发者生态带来的影响。

### WebP/AVIF新兴图像格式支持

> 自从HEVC/HEIF在苹果高调提供支持之后，由于硬件解码器的加持，本以为苹果会对其他竞争的媒体格式不再抱有兴趣，但实际上并非如此

#### WebP

WebP作为Google主导的无专利费的图像格式，其诞生后就一直跟随Chrome推广到各大Web站点，如今已经占据了互联网的一大部分（虽然其兄弟的WebM视频编码并没有这么热门）。

早在iOS 11时代，我就呼吁并提Radar希望Apple的ImageIO能够支持原生的WebP，而最终，时隔3年，在iOS 14上，ImageIO终于迎来了其内置的WebP支持，并且能够在Mac，iPhone上的各种原生系统应用中，预览WebP图像了。

那么，ImageIO对WebP的支持到底如何呢？答案其实很简单，ImageIO直接内置了开源的libwebp的一份源码和VP8的支持，并且去掉了编码的能力支持，所以能够以软件解码的形式支持WebP，不支持硬件解码。

![screenshot-20221107-210603](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-11-07/media/screenshot-20221107-210603.png)

换言之，使用这个ImageIO的系统解码器解码WebP，和使用我写的[SDWebImageWebPCoder](https://github.com/SDWebImage/SDWebImageWebPCoder)没有本质上的巨大差异（最多是一些编译器优化导致的差异），而后者还支持WebP编码（虽然耗时很慢）

#### AVIF

[AVIF](https://aomediacodec.github.io/av1-avif/v1.1.0.html)是基于AV1视频编码的新兴图像格式，作为HEVC的无专利费的竞争对手。AVIF与AV1，HEIF和HEVC，这两大阵营的关系一直是在相互竞争中不断发展的。而各大视频站如YouTube，Netflix，以及国内的Bilibli都在积极的推广这一视频格式，减少CDN带宽和专利费的成本。

而随着Apple在2018年加入[AOM-Alliance for Open Media](https://aomedia.org/)之后，我就预测有朝一日能够看到苹果拥抱这一开源标准。在2021年WebKit的开源部分曾经接受了[PR并支持AVIF软件解码](https://bugs.webkit.org/show_bug.cgi?id=207750)。而在2022的今年，iOS 16/macOS 13搭载的Safari 16，已经正式宣布[支持了AVIF](https://webkit.org/blog/13152/webkit-features-in-safari-16-0/)

虽然目前没有在其他系统应用中可以直接预览AVIF，但是我们已经看到这一趋势。在ImageIO的反编译结果中也看到了对`.avif`的处理和UTI的识别，虽然目前其本身只是会fallback到AVCI（[AVC编码](https://en.wikipedia.org/wiki/H.264/MPEG-4_AVC)的HEIF，并不是AV1），但是我相信，后续OS版本一定会带来其对应的原生SDK和应用层的整体支持，甚至未来可以看到新iPhone搭载AV1的硬件解码器。

![screenshot-20221107-211812](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-11-07/media/screenshot-20221107-211812.png)

PS：广告时间，我之前也尝试过一些利用开源AV1解码器实现的[AVIF解码库](https://github.com/SDWebImage/SDWebImageAVIFCoder)，以及macOS专用的[Finder QuickLook插件](https://github.com/dreampiggy/AVIFQuickLook)，在未来到来之前，依旧可以发挥其最后的功用：）

```
brew install avifquicklook
```

### IOSurface和硬件解码优化

> IOSurface，作为iOS平台上古老的一套在多进程，CPU与GPU之间共享内存的方案，在早期iOS 4时代就已经诞生，但是一直仅仅作为系统私有的底层XPC通信用的数据格式

而从iOS 13之后，苹果对硬件解码的支持的图像格式的上屏渲染，大量使用了IOSurface，抛弃了原有的“主线程触发CGImage的惰性解码”的模式。

也就是说，《主流图片加载库所使用的预解码究竟干了什么》这篇文章关于ImageIO的部分已经彻底过时了，至少对于JPEG/HEIF而言是这样。

如何验证这一点呢？可以从一个简单的Demo，我们这里有一个`4912*7360`分辨率的JPEG和HEIC图（[链接](https://p11.douyinpic.com/img/douyin-admin-obj/a67f70f5b8c681b25e768cf5ecde0b9b~noop.heic)），使用UIImageView渲染上屏，开启Instruments，对比内存占用

IOSurface：

```swift
// JPEG/HEIF格式限定，iOS 13，arm64真机限定
let data = try! Data(contentsOf: largeJpegUrl)
let image = UIImage(data: data)!
self.imageView.image = image
```

CGImage：

```swift
// JPEG/HEIF格式限定，iOS 13，arm64真机限定
let data = try! Data(contentsOf: largeJpegUrl)
let source = CGImageSourceCreateWithData(data as CFData, nil)!
let cgImage = CGImageSourceCreateImageAtIndex(source, 0, nil)!
let image = UIImage(cgImage: cgImage)
self.imageView.image = image
```

数据较多，直接看IOSurface的结果，可以发现，除了峰值上HEIC出现了翻倍，最终稳定占用都为**51.72MB**

而直接用CGImage（或者你换用模拟器而不是真机），则结果为**137.9MB**（RGBA8888）

+ JPEG（IOSurface）

![screenshot-20221107-220103.png](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-11-07/media/screenshot-20221107-220103.png)


+ HEIC（IOSurface）

![screenshot-20221107-220104.png](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-11-07/media/screenshot-20221107-220104.png)


#### 50%内存开销的奥秘

反编译可以发现，苹果系统库的内部流程，已经废弃了CGImage来传递这种硬件解码器的数据Buffer，而直接使用IOSurface，以换取更小的内存开销，达到同分辨率下RGBA8888的内存占用的**37.5%（即3/8）**，同分辨率下RGB888的内存占用的**50%（即1/2）**

你可能会表示很震惊，因为数学公式告诉我们，一个Bitmap Buffer的内存占用为：

```
Bytes = BytesPerPixel * Width * Height
```

而要实现这个无Alpha通道的50%内存占用，简单计算就知道，意味着`BytesPerPixel`只有1.5，也就是说12个Bit，存储了3个256（2\^8）色彩信息，换句话说0-255的数字用4个Bit表示！

你觉得数学上可能吗？答案是否定的，因为实际上是用了[色度采样](https://en.wikipedia.org/wiki/Chroma_subsampling)，并不是完整的0-255的数字，学过数字图像处理的同学都应该有所了解。

打开调试器，给IOSurface的`initWithProperties:`下断点，发现这个创建的IOSurface很有意思，`PixelFormat = 875704438('420v')`，即`kCVPixelFormatType_420YpCbCr8BiPlanarVideoRange`，看来使用了[YUV 4:2:0的采样方式](https://markrepo.github.io/avcodec/2018/06/28/YUV/)

![screenshot-20221107-215148](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-11-07/media/screenshot-20221107-215148.png)

因此，这里应该对应有两个Plane，分别对应了Y和U两个采样的平面，最终由GPU渲染时进行处理。这里不采取YUV 4:4:4的原因是，大多数JPEG/HEIF的无透明度的图像，在肉眼来看，采样损失的色度人眼差异不大，这一优化能节省50%内存占用，无疑是值得的。

值得注意的是，这里苹果处理具体采样的逻辑也是和原图像编码有关的，如果YUV 4:4:4编码的，则最终CMPhoto可能依旧会采取YUV 4:4:4进行解码并直接上屏，苹果专门的策略类来进行处理。


#### IOSurface和跨进程Buffer

不过，除了这一点，为什么只有真机能支持色度采样呢？答案和Core Animation的跨进程上屏有关。

之前有文章分享过之前，iOS的UI渲染是依赖于SpringBoard进程的中的CARenderServer子线程来处理的，因此这就有一个问题，我们如何才能将在App进程的Bitmap Buffer传给另一个进程的CARenderServer呢？

在iOS 13之前我们的方案，就是利用mmap，直接分配内存。但是mmap的问题在于，在最终Metal渲染管线传输时，我们依旧要经过一次额外的把Bitmap Buffer转为Texture并拷贝到显存的流程，因此这一套历史工作的横竖还有一些局限性。

在A12+真机的设备上，这一步借助IOSurface来实现跨CPU内存和GPU显存的高效沟通。

参考苹果的文档以及一些相关资料
+ [Transferring Data Between Connected GPUs](https://developer.apple.com/documentation/metal/resource_fundamentals/transferring_data_between_connected_gpus)
+ [Cross-process Rendering](http://www.russbishop.net/cross-process-rendering)

IOSurface的资源管理本质上是Kernel-level而不是User-level的mmap的buffer，Kernel已经实现了一套高效的传输模型，借助Lock/Unlock来避免多个进程或者CPU/GPU之间发生资源冲突，因此这是上述优化的一个必要条件。

```swift
let surface: IOSurface
surface.lock(options: .readOnly, seed: nil)
defer { surface.unlock(options: .readOnly, seed: nil) }

// Use surface.baseAddress to read the pixel data
// Make sure to step by bytesPerRow for each row
```

#### 开发者的痛，我的Public API呢？

现在揭秘了苹果优化JPEG和HEIF硬件解码内存开销之后，下一个问题是：

> 作为开发者，我如果加载一个JPEG/HEIF网络图，有办法也利用这个优化吗？

答：可以也不可以。

可以是由于，如果JPEG/HEIF网络图下载到本地，并通过`UIImage imageWithContentsOfFile:`加载，那么我们依旧可以利用到这个优化，节省内存占用。

不可以的原因是，假如像SDWebImage这种图片库，老的接口仅仅把Data存在内存中的话，就会有问题。原因是ImageIO和UIKit并没有提供公开API来加载内存中的Data，只有其内部使用了以下私有接口：

+ `-[UIImage initWithIOSurface:]`
+ `CGImageSourceCreateIOSurfaceAtIndex`

诚然，我们都知道能够直接调用任意的Objective-C/C API的姿势，这里也不再展开，只是需要注意，上文提到的这一优化，存在特定iPhone硬件（A12+）和格式（JPEG/HEIF无Alpha通道）的限定，需要注意检查可用性。

对于SDWebImage来说，如果我还继续维护下去的话，未来也许会提供基于URLSessionDownloadTask以及文件路径模式的解码方案，或许就能直接支持这一点。

### 不再安全的ImageIO

> 曾经，在我的最佳设计模式观念里，一个Producer，产出的Product，永远不应该反向持有Producer本身。但是这个想法被ImageIO团队打破了

#### 从一个崩溃说起

在iOS 15放出后的很长一段时间里，SDWebImage遇到一个奇怪的崩溃问题[#3273](https://github.com/SDWebImage/SDWebImage/issues/3273)，从堆栈来看是典型的多线程同时访问了CFData（CFDataGetBytes）导致的野指针。起初我对此并没有在意，以为又是小概率问题，并且@kinarobin提了一个可能的CGImageSource过度释放的修复后，我就关闭了这个问题。但是随后越来越多用户依旧反馈这个崩溃，因此重新打开仔细看了一下，发现了其背后的玄机。

玄机在于，iOS 15之后，Core Animation在主线程渲染CGImage时，会调用一个新增的奇怪的接口`CGImageGetImageSource`。如果带着疑问进一步追踪调用堆栈，发现在调用`CGImageSourceCreateImageAtIndex`时，ImageIO会通过`CGImageSetImageSource`绑定一个CGImageSource实例，到CGImage本身的成员变量（实际来说，是绑定到了其结构体指针存储的CGImageProperty字典）。随后，Core Animation会通过获取到这个CGImageSource，后续在渲染时间接调用CGImageSource的相关接口。持有链条为 UIImage -> UIImageCGContent -> CGImageSource

![screenshot-20221107-185536](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-11-07/media/screenshot-20221107-185536.png)


#### 崩溃的背后

这一机制改变，同时带来了一个隐患是：ImageIO它不再线程安全了。而且开发者不能修改Core Animation代码来强制加锁。

主要原因是，CGImageSource支持渐进式解码，而第三方自定义UIImage的子类时，有可能自己创建并持有这个渐进式解码的CGImageSource，并不断更新数据。在SDWebImage本身的设计中，我们通过加锁来保证，所有的对渐进式解码的调用，以及更新数据的方法，均能被同一把锁保护。

而当我们产出的CGImage，传递给了Core Animation，它无法访问这一把锁，而直接获取CGImageSource，并调用其相关的解码调用，就会出现多线程不安全的崩溃问题。

总而言之，这一设计模式的打破，即把Product和本不应该关心的Producer一起交给了外部用户，但是外部用户无法保障Producer的生命周期和调用，最终导致了这样的问题。

#### Workaround方案

最终，针对这个问题，SDWebImage提供了两套解决思路，第一个思路是直接通过CGContext提取得到自己的Bitmap Buffer，得到一个新的CGImage，切断整个持有链，最简单粗暴的修复，代价是全量关闭惰性解码无法用户控制，可能带来更高的内存占用（[#3387](https://github.com/SDWebImage/SDWebImage/pull/3387)，修复在5.13.4版本上）

第二个思路是，通过抹除掉CGImage持有的这些额外信息，采取通过CGImageCreate重新创建一个复制的CGImage，但是依旧保留了惰性解码的可选能力（[#3425](https://github.com/SDWebImage/SDWebImage/pull/3425)，方案在5.14.0版本上）。顺便提一句，通常动图（GIF/AWebP）都不支持硬件解码且切换帧频率较高，关闭惰性解码依旧是小动图的最佳实践。

PS：对感兴趣的小伙伴详细解释一下，第二个解决思路利用了CGImageProperty（类似于CGImage上存储的一个字典，按Key-Value形式存取）的时机特性，使用`CGImageCreate`重建CGImage时会完全丢失所有CGImageProperty（只有`CGImageCreateCopy`能够保留）。

而上文提到的`CGImageGetImageSource/CGImageSetImageSource`这些私有接口，本质上是操作这个`com.apple.ImageIO.imageSourceReadRef`的Key（全局变量`kImageIO_imageSourceReadRef`），Value存储了ImageIO的C++对象，并可以还原回一个CGImageSourceRef指针。一旦我们把CGImageProperty丢失掉，那么就能打断这个持有链条。

![screenshot-20221107-202706](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-11-07/media/screenshot-20221107-202706.png)


总结起来，ImageIO Team做出如此重大的设计模式改变，并没有在任何公开渠道同步过开发者，也没有提供公开接口能够控制这个行为，或者至少，没有暴露对应的`CGImageSetImageSource`接口，导致第三方开发者不得不采取曲线救国的解决方案去Workaround，这一点很值得让人吐槽。

### 总结

这篇文章看似讲了三个话题，其实背后有着一贯的缘由背景：

早期的ImageIO和各种上层框架的设计，是针对iPhone的低内存的机型做了深入优化，希望能尽量利用惰性解码，mmap缓存，换取较低内存开消，并且对各种无硬件解码的开源格式完全不感兴趣。

而最近几年，随着苹果芯片团队的努力，高内存，M1的统一内存，以及高性能芯片的诞生，苹果已经有充足的能力能够通过软件解码，共享内存，越来越多硬件解码器技术来满足主流的多媒体图像支持，本身这是一件好事。

不过问题在于历史遗下来的API，依旧保持了之前的设计缺陷，Apple团队却一直在，通过越来越Trick和Hack的方式解决问题，并没有给开发者可感知的新机制和手段来跟进优化（除开这一点吐槽，AppKit上的NSImage的NSImageRep这种代理对象设计，比UIImage的私有类UIImageContent设计要适宜的多，也灵活的多）

个人看法：软硬件一体加之闭源，会导致开源社区的实现，永远无法及时跟上其一体的私有集成，最终会捆绑到开发者和用户（开发者越强依赖苹果API和SDK，就会越强迫用户更新OS版本，进而捆绑硬件换代销售），这并不是一个好的现象🙃

### 招募

[SDWebImage](https://github.com/SDWebImage/SDWebImage)开源项目如今缺少长久维护的Maintainer，如果你对iOS/macOS框架开发感兴趣，对图像渲染和Apple平台有所涉猎，对Swift/Objective-C大型开源项目贡献有所期待，可以在[我的GitHub](https://github.com/dreampiggy)上，以Email，Twitter私信等方式联系我。