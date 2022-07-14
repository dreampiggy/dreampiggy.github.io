---
title: 一步步带你开发macOS QuickLook Plugin
date: 2019-04-16 15:02:11
categories:
tags:
- macOS
- QuickLook
- AVIF
---

## QuickLook简介

QuickLook 是macOS上提供的一项快速展示文档预览的功能，只需要按下空格就可以快速查看各种文件格式的信息，包括文本，代码，图片，音频，视频等等。

由于QuickLook需要支持不断扩展的文件格式，因此macOS专门提供了一个QuickLook Plugin，能让开发者对自己的文件格式提供一个自定义的完整的UI显示，不必依赖macOS系统更新来支持缤纷复杂的格式。

之前一段时间，出于兴趣做了一个[AVIF (AV1 Image File Format)](https://aomediacodec.github.io/av1-avif/)的解码器封装，AV1作为现在流行的HEVC(H.265)潜在未来竞争者，有着开源，无专利限制，更高的压缩比等等优势，比起HEVC晚诞生了5年。

目前AVIF虽然发布了第一版规范，但是缺少相应的周边工具链的支持，在macOS上想要找一个简单的Image Viewer都没找到，调试起来异常困难，因此抽空顺便做了一个简单的Quick Look Plugin，来让自己能直接空格预览AVIF图像。

在做QuickLook Plugin的过程中，感觉有一些小坑需要记下来，因此这篇文章，目标就是一个简单的入门教程，讲解如何做一个QuickLook Plugin，来对自己喜爱但又不被系统支持的文件格式，提供更好的用户体验支持。

## QuickLook Plugin工程

虽然苹果提供了完善的QuickLook Plugin开发文档，参考：[Quick Look Programming Guide](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/Quicklook_Programming_Guide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40005020-CH1-SW1)

但是文档已经稍显过时，遇到的一个坑点也没有提示，因此这里更详细直观的介绍一下QuickLook开发的流程。


+ 新建Xcode工程，选择这个`Quick Look Plug-In`模板

![屏幕快照 2019-04-16 上午11.45.25](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-07-14/image/2019/04/16/屏幕快照 2019-04-16 上午11.45.25.png)


+ 打开你的模版，你会发现如下的结构

```
Project
- GenerateThumbnailForURL.c // 用来提供Finder缩略图的代码
- GeneratePreviewForURL.c // 用来生成Preview用的绘制代码
- main.c // 插件入口文件，不要修改它
- Info.plist // 描述插件支持的UTI类型的，后面会讲
```

QuickLook Plugin支持两种情形的功能展示：一个是对文件，按下空格来展示的窗口预览，在使用Option+空格进行全屏预览时候也会展示，后面都称作Preview

另一个是用来给Finder，来提供一个缩略图展示，这样一些图像格式，视频格式，在Finder中就能直接看到对应的缩略图，而不是一个僵硬的默认图标。后文都称作Thumbnail

由于QuickLook的核心，是希望对**指定的文件格式**，提供一个展示的UI和缩略图。那么在继续进一步写代码之前，我们必须得首先清楚自己需要的文件格式是什么，并了解UTI的概念。如果这一步骤处理的有问题，你的QuickLook Plugin是无法按预期的想法，被调用的。

## 绑定文件格式和UTI

在继续下一步之前，你需要对你想支持的文件格式，选择一个UTI ([Uniform Type Identifiers](https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/understanding_utis/understand_utis_intro/understand_utis_intro.html#//apple_ref/doc/uid/TP40001319)).

QuickLook，在用户按下空格开始Preview的时候，会根据每个QuickLook Plugin注册的UTI，依次去询问，直到找到第一个返回成功的，最后来判定选择哪个Plugin进行展示。

建立好模版之后，打开`Info.plist`，在顶层的`LSItemContentTypes`项里面，添加你的Plugin所能支持的UTI，是一个数组，会按照先后顺序匹配，一般建议只写自己能准确识别的UTI，如果是一个通配的Plugin（如通用图片预览，通用代码预览），可以使用UTI继承关系的父级(`public.image`, `public.source-code`等）

```xml
<key>CFBundleDocumentTypes</key>
<array>
	<dict>
		<key>CFBundleTypeRole</key>
		<string>QLGenerator</string>
		<key>LSItemContentTypes</key>
		<array>
			<string>public.avif</string> <!--Here!-->
		</array>
	</dict>
</array>
```

在配置好Plugin支持的UTI之后，你还需要根据具体UTI的分配来源，来使用导入或者导出。

### 找出已有的UTI

你可以通过使用如下命令，查看一个文件对应的UTL

```
mdls test.avif
```

查看输出的`kMDItemContentType`，如果是以`dyn`开头，表明没有被注册过，而是系统分配的一个动态UTI（用于任意不支持的类型和代码兼容，参考[Dynamic Type Identifiers](https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/understanding_utis/understand_utis_conc/understand_utis_conc.html#//apple_ref/doc/uid/TP40001319-CH202-BCGCDHIJ)）

否则，形如`public.png`这种，标示是一个已有的UTI，可以导入来直接使用

```
kMDItemContentType ="dyn.ah62d4rv4ge80c7xmq2"
```

如果你是一个比较执着的人，想了解具体的每一个UTI，是由系统或者还是某个第三方App注册的，你可以使用如下命令，导出完整的系统UTI报表，来进行搜索。

```
/System/Library/Frameworks/CoreServices.framework/Versions/A/Frameworks/LaunchServices.framework/Versions/A/Support/lsregister -dump
```

### UTI定义

一个UTI对应一段XML的定义，其中声明了它的类型（继承关系），UTI字符串，简介名称，扩展名，标准链接等等，基本的格式如下，很容易理解。这里是自己定义的一个AVIF格式的描述

```xml
<dict>
    <key>UTTypeConformsTo</key>
    <array>
        <string>public.image</string>
    </array>
    <key>UTTypeDescription</key>
    <string>AVIF image</string>
    <key>UTTypeIdentifier</key>
    <string>public.avif</string>
    <key>UTTypeReferenceURL</key>
    <string>https://aomediacodec.github.io/av1-avif/</string>
    <key>UTTypeTagSpecification</key>
    <dict>
        <key>public.filename-extension</key>
        <array>
            <string>avif</string>
        </array>
        <key>public.mime-type</key>
        <string>image/avif</string>
    </dict>
</dict>
```

### 导入UTI

如果你想支持QuickLook的文件格式，已经有了系统分配的UTI，或者第三方App定义好的UTI，那么你要做的，就是导入一个UTI。

如果要导入UTI，你需要在`Info.plist`中，使用`UTImportedTypeDeclarations`这个项，来导入对应的UTI描述内容，值是一个数组，数组每项都是上面提到的UTI定义。

PS：对于导入UTI来说，你其实并不需要完整的把别人的声明抄过来，只要存在`UTTypeIdentifier`项即可，但是这样写能更清晰了解对应的格式描述。

```xml
<key>UTImportedTypeDeclarations</key>
<array>
    <dict>
    	<key>UTTypeIdentifier</key>
    	<string>public.png</string>
    	<!--...-->
    </dict>
</array>
```

### 导出UTI

反之，如果你想支持的QuickLook的文件格式，不存在已有的UTI，那么你需要新增一个并导出。

如果要导出UTI，你需要在`Info.plist`中，使用`UTExportedTypeDeclarations`这个项，来导出对应的UTI描述内容，值是一个数组，数组每项都是上面提到的UTI定义。

```xml
<key>UTExportedTypeDeclarations</key>
<array>
    <dict>
    	<key>UTTypeIdentifier</key>
    	<string>public.avif</string>
    	<!--...-->
    </dict>
</array>
```

### QuickLook Plugin和导出UTI

值得注意的一个坑点，macOS系统注册UTI规则，会注册当前硬盘上所有的`.app`后缀的App包，里面所含有的导出UTI，而遗憾的是，作为QuickLook Plugin，最后编译得到的产物，不是以`.app`为后缀名的，而是一个`.qlgenerator`。

因此，这就导致，如果你新增了一个UTI，但是你的QuickLook Plugin，没有任何宿主App来提供导出UTI，最终macOS会不认这个UTI，因此你的QuickLook Plugin不会被调用。这可能是苹果早期认为，QuickLook Plugin是和一个App绑定的（如Keynote和Keynote QuickLook插件的关系），独立存在的QuickLook Plugin并没有特别处理……

这个坑花费了一些时间，经过一番StackOverflow和GitHub搜索，最终找到了一个非常聪明（Trick）的解决方案：

构造一个临时占位的`Dummy.app`包，专门用于导出UTI，在打包的时候直接将这个`Dummy.app`拷贝到对应QuickLook Plugin的包中即可

我们可以使用macOS自带的`Script Editor.app`，来创建一个空壳App：

1. 打开`Script Editor`，创建一个新文档
2. 直接Save，类型选择`Application`，名称随便写一个`Dummy.app`，导出
3. 用文本编辑器，打开`Dummy.app/Contents/Info.plist`
4. 参考上文提到的UTI导出方式，添加对应的`UTExportedTypeDeclarations`项目
5. 将这个`Dummy.app`，放到工程下，直接拖进来当作资源，添加到`Copy Bundle Resource`过程中


![未命名3](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-07-14/image/2019/04/16/未命名3.png)

这样一波操作以后，你最后构建得到的QuickLook Plugin，就能自带一个导出的UTI，然后被系统识别，最终被真正加载。


## 用于Preview的代码绘制实现

准备好上述UTI的配置后，现在再来看看代码。首先我们侧重看一下用于提供Preview的UI的代码。

对应的文件是`GeneratePreviewForURL.c`。如果要使用Objective-C，或者C++代码，你可以更改对应的文件名为`.m`或者`.cpp`即可，以下示例是以Objective-C代码为主

入口调用函数原型为下：

```c
GeneratePreviewForURL(void *thisInterface, QLPreviewRequestRef preview, CFURLRef url, CFStringRef contentTypeUTI, CFDictionaryRef options)
```

其实对于大多数QuickLook插件，我们关注的基本上只有这个`url`参数，他对应的是文件的File URL，可以拿到对应被选中的文件Data Buffer。

```objectivec
NSString *path = [(__bridge NSURL *)url path];
NSData *data = [NSData dataWithContentsOfFile:path];
```

下一步就是绘制和渲染我们的UI，QuickLook支持两种方式渲染：

+ 使用Core Graphics自定义绘制
+ 使用预置支持的数据格式，动态生成Data

### 使用Core Graphics绘制

这里假设已经了解Core Graphics绘制的基本知识，如果有不了解请提前查阅苹果的教程：[Quartz 2D Programming Guide](https://developer.apple.com/library/archive/documentation/GraphicsImaging/Conceptual/drawingwithquartz2d/Introduction/Introduction.html).

在拿到Data以后，该怎么绘制取决于你的QuickLook插件的功能，比如说，我想做的一个AVIF图像预览Quick Look插件，那么就希望触发解码，以拿到CGImage和Bitmap Buffer来绘制。

```objectivec
CGImageRef cgImgRef = [AVIFDecoder createAVIFImageWithData:data];
```

下一步，我们需要获取一个CGContext来绘制，使用[QLPreviewRequestCreateContext](https://developer.apple.com/documentation/quicklook/1402613-qlpreviewrequestcreatecontext?language=objc)，传入入口函数透传进来的`preview`，会得到一个CGContext，来作为上下文进行绘制。同时，还需要了解绘制的大小，标题等等选项，来提供合适的渲染UI。

```objectivec
CGFloat width = CGImageGetWidth(cgImgRef);
CGFloat height = CGImageGetHeight(cgImgRef);

// Add image dimensions to title
NSString *newTitle = [NSString stringWithFormat:@"%@ (%d x %d)", [path lastPathComponent], (int)width, (int)height];
    
NSDictionary *newOpt = @{(NSString *)kQLPreviewPropertyDisplayNameKey : newTitle,
    (NSString *)kQLPreviewPropertyWidthKey : @(width),
    (NSString *)kQLPreviewPropertyHeightKey : @(height)};

// Draw image
CGContextRef ctx = QLPreviewRequestCreateContext(preview, CGSizeMake(width, height), YES, NULL);
CGContextDrawImage(ctx, CGRectMake(0,0,width,height), cgImgRef);
QLPreviewRequestFlushContext(preview, ctx);

// Cleanup
CGImageRelease(cgImgRef);
CGContextRelease(ctx);
```

这样基本就完成了，我们绘制了一个完整的图像到CGContext上，QuickLook会渲染到屏幕上，大小是我们指定的图像大小。

![](https://raw.githubusercontent.com/dreampiggy/AVIFQuickLook/master/Screenshot/Preview.png)

如果你的QuickLook插件，需要有一个异步的处理和等待，同时可以实现这个取消的入口函数，来减少CPU占用，优化一下用户体验

```
void CancelPreviewGeneration(void *thisInterface, QLPreviewRequestRef preview)
```

比如说，对于大图像解吗，可以中断解码提前释放内存。

### 使用预置类型生成数据渲染

QuickLook Preview还有另一种渲染方式，就是使用QuickLook预置的文件类型支持，来提供相应的数据。对应文档：[Dynamically Generating Previews](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/Quicklook_Programming_Guide/Articles/QLDynamicGeneration.html#//apple_ref/doc/uid/TP40005020-CH15-SW5)


我们需要使用[QLPreviewRequestSetDataRepresentation](https://developer.apple.com/documentation/quicklook/1402661-qlpreviewrequestsetdatarepresent?language=objc)，来提供一个预置支持格式的Data Buffer给QuickLook。

支持的格式有：

+ Image: 系统Image/IO解码库支持的图像压缩格式
+ PDF：PDF数据
+ HTML：WebKit支持的HTML字符串，注意如果有本地的CSS，需要使用`kQLPreviewPropertyAttachmentDataKey`带上CSS的数据
+ XML：WebKit支持的XML字符串
+ RTF：macOS支持的富文本格式（NSAttributedString可以转换的到）
+ Text：纯文本字符串
+ Movie：系统CoreVideo库支持的视频压缩格式
+ Audio：系统CoreAudio库支持的音频压缩格式

```objectivec
NSImage *image;
NSData *imageData = [image TIFFRepresentation];
QLPreviewRequestSetDataRepresentation(preview, (__bridge CFDataRef)data, kUTTypeImage, NULL);
```

### 在其他App中使用Preview

值得一提的是，得益于macOS完整的软件生态，你的QuickLook Plugin的Preview UI，不仅仅会出现在Finder中空格弹出的预览，甚至于Xcode和一些第三方App内置的预览（即用到了[QLPreviewPanel](https://developer.apple.com/documentation/quartz/qlpreviewpanel)来展示UI的地方），都能触发你的插件，所以可以说是非常舒服。

在Xcode中缩略图如下：

![屏幕快照 2019-04-16 下午1.49.04](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-07-14/image/2019/04/16/屏幕快照 2019-04-16 下午1.49.04.png)

## 用于Thumbnail的代码绘制实现

说完了关于Preview的实现代码，现在再来看看关于如何生成Finder用到的文件缩略图

Thumbnail也支持两种模式

1. 使用同Preview的，基于Core Graphics绘制逻辑
2. 更为简单的API，使用CGImage或者Image Data

第一种方式，和上文一模一样，这里就不再赘述了。我们可以看看第二种方式。我们只需要提供一个CGImage，或者一个Image/IO支持的图像格式的Image Data即可

```objectivec
// 如果是原生支持的格式，使用QLThumbnailRequestSetImageWithData

// 否则，自己解码器输出一个CGImage，然后传进去
CGImageRef cgImgRef;
if (cgImgRef) {
    QLThumbnailRequestSetImage(thumbnail, cgImgRef, nil);
    CGImageRelease(cgImgRef);
} else {
    QLThumbnailRequestSetImageAtURL(thumbnail, url, nil);
}
```

对应在Finder中缩略图如下：

![](https://raw.githubusercontent.com/dreampiggy/AVIFQuickLook/master/Screenshot/Thumbnails.png)


## 调试QuickLook插件

作为一个插件，要调试起来比起一般的App要麻烦一些。不过好在macOS提供了一个专门的QuickLook调试命令，苹果也有专门文档介绍

我们可以使用如下的命令，以`public.avif`的UTI，对`test.avif`文件，触发一次Quick Look的Preview，来查看渲染是否正确。

```
qlmanage -d2 -p test.avif -c public.avif
```

同时，为了能够Debug单步调试，我们使用Xcode的Debug Scheme，通过将`Execulable`改成`/usr/bin/qlmanage`，在Arguments中填写成上述的参数。

![未命名](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-07-14/image/2019/04/16/未命名.png)

![未命名2](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-07-14/image/2019/04/16/未命名2.png)

这样，你可以给你的对应代码下上断点，当你再次点击Run来运行时，会自动触发单步调试，检查存在的问题。

## 总结

整体看下来，QuickLook Plugin的开发流程并没有多么复杂，其实你要做的就是用已有的Core Graphics绘制知识，并不涉及到AppKit相关概念，对于iOS开发者也能快速上手。

其中的坑，主要在于没有文档说明新增UTI，需要绑定一个App，而不是QuickLook Plugin本身能够声明的，对应也介绍了一个聪明的方式绕过这一限制。希望能帮助到有同样需求的人。

自己的AVIF QuickLook Plugin也终于完工，欢迎有兴趣的人尝试，并且给一点Star：

+ [AVIFQuickLook](https://github.com/dreampiggy/AVIFQuickLook)：预览AVIF图像

这里还有一些推荐和自己用到的QuickLook Plugin，也列举出来，能大大提升日常使用效率哦

+ [qlImageSize](https://github.com/Nyx0uf/qlImageSize)：预览WebP和BPG图像，显示图像元信息
+ [qlmarkdown](https://github.com/toland/qlmarkdown)：预览Markdown
+ [ProvisionQL](https://github.com/ealeksandrov/ProvisionQL)：预览ipa包，以及mobileprovision证书信息

