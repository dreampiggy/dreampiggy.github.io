---
title: iOS平台图片编解码入门教程（Image/IO篇）
date: 2017-10-30 17:32:29
categories: iOS
tags:
  - iOS
  - Image
---
> 这篇教程是系列教程的第一篇，主要是面向于没有怎么接触过iOS平台上图像编解码的人的，不会涉及到多媒体处理中的数字信号处理、图像编码的深入知识。这是系列最简单的一篇，之后会有关于第三方编解码，以及vImage的另两篇教程。

# Image/IO
Image/IO是Apple提供的一套用于图片编码解码的系统库，对外是一层非常直观易用的C的接口。上层的UIKit，Core Image，还有Core Graphics中的CGImage处理，都是依赖Image/IO库的。因此，掌握Image/IO的基本编解码操作，对一些图像相关的数据处理是非常必要的。这篇教程就主要从简单的用法，说明Image/IO的用法，完整的文档，可以参考[Apple Image/IO](https://developer.apple.com/documentation/imageio)

# 解码
解码，指的是讲已经编码过的图像封装格式的数据，转换为可以进行渲染的图像数据。具体来说，iOS平台上就指的是将一个输入的二进制Data，转换为上层UI组件渲染所用的UIImage对象。

Image/IO的解码，支持了常见的图像格式，包括PNG（包括APNG）、JPEG、GIF、BMP、TIFF（具体的，可以通过`CGImageSourceCopyTypeIdentifiers`来打印出来，不同平台不完全一致）。在iOS 11之后另外支持了HEIC（即使用了HEVC编码的HEIF格式）。

对于解码操作，我们可以分为静态图（比如JPEG，PNG）和动态图（比如GIF，APNG）的两种，分别进行说明一下解码的过程。

## 静态图
静态图的解码，基本可以分为以下步骤：

1. 创建CGImageSource
2. 读取图像格式元数据（可选）
3. 解码得到CGImage
4. 生成上层的UIImage，清理

### 1. 创建ImageSource
CGImageSouce，表示的是一个待解码数据的输入。之后的一系列操作（读取元数据，解码）都需要到这个Source，与解码流程一一对应。

CGImageSource可以通过不同的几个接口构造（这里先忽略渐进式解码的接口）：

+ `CGImageSourceCreateWithData`： 从一个内存中的二进制数据（CGData）中创建ImageSource，相对来说最为常用的一个
+ `CGImageSourceCreateWithURL`： 从一个URL（支持网络图的HTTP URL，或者是文件系统的fileURL）创建ImageSource，
+ `CGImageSourceCreateWithDataProvider`：从一个DataProvide中创建ImageSource，DataProvider提供了很多种输入，包括内存，文件，网络，流等。很多CG的接口会用到这个来避免多个额外的接口。

示例代码：

```objectivec
CGImageSourceRef source = CGImageSourceCreateWithData((__bridge CFDataRef)data, NULL);
if (!source) { // 一般这时候都是输入图像数据的格式不支持
    return nil;
}
```

### 2. 读取图像格式元数据
创建好CGImageSource之后，我们是可以立即解码。但是很多情况下，我们需要获取一些相关的图像信息，包括图像的格式，图像数量，[EXIF元数据](https://en.wikipedia.org/wiki/Exif)等。在真正解码之前，我们可以拿到这些数据，进行一些处理，之后再开始解码过程。

其中，这些信息可以直接在CGImageSource上获取：

+ 图像格式：`CGImageSourceGetType`
+ 图像数量（动图）：`CGImageSourceGetCount`

其他的，需要通过获取属性列表来查询。对于图像容器的属性（EXIF等），我们需要使用`CGImageSourceCopyProperties`即可，然后根据不同的Key去获取对应的信息。

其实苹果还有一套`CGImageSourceCopyMetadataAtIndex`，对应的数据不是字典，而是一个`CGImageMetadata`，再通过其他方法去取。这套API使用起来也是可以的，读取数据和前者是完全兼容一致的，优点是能够进行自定义扩展（比如说你有非标准的图像信息想自己添加和删除）。一般来说使用前者就足够了。

示例代码：

```objectivec
CGImageSourceRef source;
NSDictionary *properties = (__bridge NSDictionary *)CGImageSourceCopyProperties(source, NULL);
NSUInteger fileSize = [properties[kCGImagePropertyFileSize] unsignedIntegerValue]; // 没什么用的文件大小

NSDictionary *exifProperties = properties[(__bridge NSString *)kCGImagePropertyExifDictionary]; // EXIF信息
NSString *exifCreateTime = exirProperties[(__bridge NSString *)kCGImagePropertyExifDateTimeOriginal]; // EXIF拍摄时间
```

当然，前面这个指的是图像容器的属性，而真正的获取图像的元信息，需要使用`CGImageSourceCopyPropertiesAtIndex`，对于静态图来说，index始终传0即可。

示例代码：

```objectivec
NSDictionary *imageProperties = (__bridge NSDictionary *) CGImageSourceCopyPropertiesAtIndex(source, 0, NULL);
NSUInteger width = [imageProperties[(__bridge NSString *)kCGImagePropertyPixelWidth] unsignedIntegerValue]; //宽度，像素值
NSUInteger height = [imageProperties[(__bridge NSString *)kCGImagePropertyPixelHeight] unsignedIntegerValue]; //高度，像素值
BOOL hasAlpha = [imageProperties[(__bridge NSString *)kCGImagePropertyHasAlpha] boolValue]; //是否含有Alpha通道
CGImagePropertyOrientation exifOrientation = [imageProperties[(__bridge NSString *)kCGImagePropertyOrientation] integerValue]; // 这里也能直接拿到EXIF方向信息，和前面的一样。如果是iOS 7，就用NSInteger取吧 :)
```

### 3. 解码得到CGImage
通过Image/IO解码到CGImage确实非常简单，整个解码只需要一个方法`CGImageSourceCreateImageAtIndex`。对于静态图来说，index始终是0，调用之后会立即开始解码，直到解码完成。

值得注意的是，Image/IO所有的方法都是线程安全的，而且基本上也都是同步的，因此确保大图像文件的解码最好不要放到主线程。

示例代码：

```objectivec
CGImageRef imageRef = CGImageSourceCreateImageAtIndex(source, 0, NULL);
```

### 4. 生成上层的UIImage，清理
解码得到CGImage后，就基本完成了，我们可以直接构造对应的UIImage用于UI组件渲染。其中UIImage的orientation，可以通过之前的EXIF元信息获得（注意，需要转换EXIF的方向，到UIImageOrientation的方向）。然后就完成了，比较简单。

示例代码：

```objectivec
// UIImageOrientation和CGImagePropertyOrientation枚举定义顺序不同，封装一个方法搞一个switch case就行
UIImageOrientation imageOrientation = [self imageOrientationFromExifOrientation:exifOrientation];
UIImage *image = [[UIImage imageWithCGImage:imageRef scale:[UIScreen mainScreen].scale orientation:imageOrientation];

// 清理，都是C指针，避免内存泄漏
CGImageRelease(imageRef);
CFRelease(source)
```

## 动态图
前面的情况，主要介绍了是静态图（也就是说，取的index都是0的情况 ）。对于动态图来说，我们可以通过`CGImageSourceGetCount`来获取动图的帧数，之后就比较简单了，通过循环遍历每一帧，重复2-4步骤生成对应的UIImage，最后通过UIImage自带的`animatedImageWithImages:duration:`来生成一张动图即可。但是关于这里有坑，在下面说明。

步骤：

1. 静态图的步骤1
2. 遍历所有图像帧，重复静态图的步骤2-4
3. 生成动图UIImage

### 1. 生成动图UIImage
由于遍历很简单，就不重复了，这里我们以一个GIF为例，简单说明一下解码过程，直观易懂。

示例代码：

```objectivec
NSUInteger frameCount = CGImageSourceGetCount(source); //帧数
NSMutableArray <UIImage *> *images = [NSMutableArray array];
double totalDuration = 0;
for (size_t i = 0; i < frameCount; i++) {
    NSDictionary *frameProperties = (__bridge NSDictionary *) CGImageSourceCopyPropertiesAtIndex(source, i, NULL);
    NSDictionary *gifProperties = frameProperties[(NSString *)kCGImagePropertyGIFDictionary]; // GIF属性字典
    double duration = [gifProperties[(NSString *)kCGImagePropertyGIFUnclampedDelayTime] doubleValue]; // GIF原始的帧持续时长，秒数
    CGImagePropertyOrientation exifOrientation = [frameProperties[(__bridge NSString *)kCGImagePropertyOrientation] integerValue]; // 方向
    CGImageRef imageRef = CGImageSourceCreateImageAtIndex(source, i, NULL); // CGImage
    UIImageOrientation imageOrientation = [self imageOrientationFromExifOrientation:exifOrientation];
    UIImage *image = [[UIImage imageWithCGImage:imageRef scale:[UIScreen mainScreen].scale orientation:imageOrientation];
    totalDuration += duration;
    [images addObject:image];
}

// 最后生成动图
UIImage *animatedImage = [UIImage animatedImageWithImages:images duration:totalDuration];
```

这样处理的话，大部分情况下基本是可以接受的。但是这里有一个坑：UIImage这个animatedImages的接口，只会根据你传入的images的数量，平均分配传入的totalDuration的展示时长。但是大部分动图格式（GIF，APNG，WebP等等），都是不同帧不同时长的，这就会导致最后看到的动图每帧时长乱掉。

对于这个的解决方式也有。简单来说，就是通过对特定图像帧重复特定次数，以填充满整个应该播放的时长。其实实现也比较简单，我们可以对所有帧的时长，求一个最大公约数`gcd`，这样的话，只需要每帧重复播放`duration / gcd`次数，最终的总时长各帧`repeat * duraion`的和，就可以实现这个了，有兴趣可以看看我参与维护的[SDWebImage的代码](https://github.com/rs/SDWebImage/blob/4.1.2/SDWebImage/UIImage%2BWebP.m#L281=L294)。

示例代码：

```objectivec
NSUInteger durations[frameCount];
NSUInteger const gcd = gcdArray(frameCount, durations);
for (size_t i = 0; i < frameCount; i++) {
    NSUInteger duration = durations[i];
    NSUInteger repeatCount = duration / gcd;
    for (size_t j = 0; j < repeatCount; j++) {
        [animatedImages addObject:image];
    }
}
```

## 渐进式解码
渐进式解码（Progressive Decoding），即不需要完整的图像流数据，允许解码部分帧（大部分情况下，会是图像的部分区域），对部分使用了渐进式编码的格式（参考：[渐进式编码](https://en.wikipedia.org/wiki/Interlacing_\(bitmaps)），则更可以解码出相对模糊但完整的图像。

比如说，JPEG支持三种方式的渐进式编码，包括Baseline，interlaced，以及progressive（参考：[iOS 处理图片的一些小 Tip](https://blog.ibireme.com/2015/11/02/ios_image_tips/))

Baseline | Interlaced | Progressive
:-: | :-: | :-:
![](https://blog.ibireme.com/wp-content/uploads/2015/11/image_baseline.gif) | ![](https://blog.ibireme.com/wp-content/uploads/2015/11/image_interlaced.gif) | ![](https://blog.ibireme.com/wp-content/uploads/2015/11/image_progressive.gif)

对于Image/IO的渐进式解码，其实和静态图解码的过程类似。但是第一步创建CGImageSource时，需要使用专门的`CGImageSourceCreateIncremental`方法，之后每次有新的数据（下载或者其他流输入）输入后，需要使用`CGImageSourceUpdateData`（或者`CGImageSourceUpdateDataProvider`）来更新数据。注意这个方法需要每次传入所有至今为止解码的数据，不仅仅是当前更新的数据。

之后的过程，就和普通的解码一致，就不再说明了。

示例代码：

```objectivec
NSData *data;
bool finished = data.length == totalLength;
CGImageSourceRef source;
// 更新数据
CGImageSourceUpdateData(source, (__bridge CFDataRef)data, finished);

// 和普通解码过程一样
CGImageRef imageRef = CGImageSourceCreateImageAtIndex(source, 0, NULL);
```

# 编码
编码过程，这里指的就是将一个UIImage表示的图像，编码为对应图像格式的数据，输出一个NSData的过程。Image/IO提供的对应概念，叫做CGImageDestination，表示一个输出。之后的编码相关的操作，和这个Destination一一对应。

## 静态图
静态图的编码，基本可以分为以下步骤：

1. 创建CGImageDestination
2. 添加图像格式元数据（可选）和CGImage
3. 编码得到NSData，清理

### 1. 创建CGImageDestination
CGImageDestination的创建也有三个接口，你需要提供一个输出的目标来输出解码后的数据。同时，由于编码需要提供文件格式，你需要指明对应编码的文件格式，用的是[UTI Type](https://developer.apple.com/library/content/documentation/FileManagement/Conceptual/understanding_utis/understand_utis_intro/understand_utis_intro.html#//apple_ref/doc/uid/TP40001319)。对于静态图来说，第三个参数的数量都写1即可。

+ `CGImageDestinationCreateWithData`：指定一个可变二进制数据作为输出
+ `CGImageDestinationCreateWithURL`：指定一个文件路径作为输出
+ `CGImageDestinationCreateWithDataConsumer`：指定一个DataConsumer作为输出

示例代码：

```objectivec
CFStringRef imageUTType; //目标格式，比如kUTTypeJPEG
// 创建一个CGImageDestination
CGImageDestinationRef destination = CGImageDestinationCreateWithData((__bridge CFMutableDataRef)imageData, imageUTType, 1, NULL);
if (! destination) {
    // 无法编码，基本上是因为目标格式不支持
    return nil;
}
```

### 2. 添加图像格式元数据（可选）和CGImage
接下来就是添加图像了，由于CGImage只是包含基本的图像信息，很多额外信息比如说EXIF都已经丢失了，如果我们需要，可以添加对应的元信息。不像解码那样提供了两个API分别获取元信息和图像。使用的接口是`CGImageDestinationAddImage`。

当然，如果有自定义的元信息，可以通过另外的`CGImageDestinationAddImageAndMetadata`来添加`CGImageMetadata`，这个上面解码也说到过，这里就不解释了。

此外，还有一个ImageIO最强大的功能，叫做`CGImageDestinationAddImageFromSource `（这个东西可以媲美`vImageConvert_AnyToAny`，后续教程会谈到），这个能够从一个任意的CGImageSource，添加一个图像帧到任意一个CGImageDestination。这个一般的用途，就是专门给图像转换器用的，比如说从图像格式A，转换到图像格式B。我们不需要先解码到A的UIImage，再通过编码到B的NSData，直接在中间就进行了转换。能够极大地提升转换效率（Image/IO底层就是通过vImage，传的是Bitmap的引用，没有额外的消耗）。不过这篇教程侧重于Image/IO的编码和解码，转换可以自行参考处理，不再详细说明了。

示例代码：

```objectivec
CGImageRef imageRef = image.CGImage; // 待编码的CGImage

// 可选元信息，比如EXIF方向
CGImagePropertyOrientation exifOrientation = kCGImagePropertyOrientationDown;
NSMutableDictionary *frameProperties = [NSMutableDictionary dictionary];
imageProperties[(__bridge_transfer NSString *) kCGImagePropertyExifDictionary] = @(exifOrientation);
// 添加图像和元信息
CGImageDestinationAddImage(destination, imageRef, (__bridge CFDictionaryRef)frameProperties);
```

### 3. 编码得到NSData，清理
当添加完成所有需要编码的CGImage之后，最后一步，就是进行编码，得到图像格式的数据。这里直接用一个方法`CGImageDestinationFinalize`即可，编码得到的数据，会写入最早初始化时提供的Data或者DataConsumer。

示例代码：

```objectivec
if (CGImageDestinationFinalize(destination) == NO) {
    // 编码失败
    imageData = nil;
}
// 编码成功，清理……
CFRelease(destination);
```

## 动态图
动态图的编码，其实不像解码那样困难。只需要准备好所有的动态图的帧，按照帧的顺序进行一一添加即可。基本步骤可以概括为：

1. 静态图的步骤1，提供帧数
2. 遍历所有图像帧，重复静态图的步骤2
3. 静态图的步骤3


### 1. 提供帧数，遍历图像帧
在进行动态图编码时，创建CGImageDestination的时候需要提供动态图的张数。即在`CGImageDestinationCreateWithData`的参数中，将`count`设置为需要编码的总张数。

另外，在遍历图像帧的过程，其实只需要不断地按顺序添加就行了，如果需要设置额外元信息，也需要按顺序设置到当前帧上。相对于解码来说简单多了。其他的没有什么大的区别。我们这里还是以GIF为例，简单说明一下。

示例代码：

```objectivec
NSArray<UIImage *> *images;
float durations[frameCount];
for (size_t i = 0; i < frameCount; i++) {
    float frameDuration = durations[i];
    CGImageRef frameImageRef = images[i].CGImage;
    NSDictionary *frameProperties = @{(__bridge_transfer NSString *)kCGImagePropertyGIFDictionary : @{(__bridge_transfer NSString *)kCGImagePropertyGIFUnclampedDelayTime : @(frameDuration)}};
    CGImageDestinationAddImage(imageDestination, frameImageRef, (__bridge CFDictionaryRef)frameProperties);
}
```

# 总结
Image/IO封装了非常简单直观的接口来处理图像编解码，对于任何开发者来说都能轻易上手。而且性能方面很多格式都有Apple自己的硬件解码器来做保证。另外，对于图像转换，Image/IO所提供的这种Source-Destination的操作能够非常方便地在不同格式之间转换，有兴趣的人务必可以试试。

不过遗憾的是，Image/IO的接口设计并没有提供可以扩展或者插件化的地方，不支持的图像格式就比较无能为力了。关于这个问题，请期待系列教程第二篇——第三方编解码教程。
