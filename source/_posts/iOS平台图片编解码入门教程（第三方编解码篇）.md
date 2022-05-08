---
title: iOS平台图片编解码入门教程（第三方编解码篇）
date: 2017-10-30 18:05:14
categories: iOS
tags:
  - iOS
  - Image
---
> 这篇教程，是系列教程的第二篇，前篇名为《iOS平台图片编解码入门教程（Image/IO篇）》。这篇主要讲第三方解码器如何在iOS平台上处理（和Image/IO的几大要点一一对应），更会介绍一些基本的Bitmap概念，总结通用的处理方法，毕竟授人以鱼不如授人以渔

# 第三方编解码
对于图片编解码来说，Apple自带的Image/IO确实非常的易用，但是对于Image/IO不支持的图像格式就能无能为力了。截止到iOS 11，Image/IO不支持WebP，BPG，对于一些需要依赖WebP的业务就比较麻烦了（WebP的优点就不再介绍了）。不过我们可以自己集成第三方的图片解码器，去支持这些需要的的格式。

一般来说，我们需要根据自己想要支持的图像格式，选择相对应的编解码器，进行编解码。这里我们以WebP的解码库[libwebp](https://developers.google.com/speed/webp/docs/api)为例子，其他解码器需要根据对应解码器的API处理，基本概念类似。

# 解码
不像Image/IO那样封装了整套流程，第三方解码的关键之处，就是在于获取到图像的[Bitmap](https://en.wikipedia.org/wiki/Raster_graphics)数据，通常情况就是[RGBA](https://en.wikipedia.org/wiki/RGBA_color_space)的矢量表示。

简单解释一下，Bitmap可以理解为连续排列像素（Pixel）的二维数组，一个像素包括4个通道(Components)的点，每个点的位数叫做色深（常见的32位色，指的就是1个像素有4通道，每个通道8位），而像素的通道排列顺序按照对应的RGBA Mode顺序排列，比如说RGBA8888（大端序），就是这样一串连续的值：

```text
{uint8_t red, uint8_t green, uint8_t blue, uint8_t alpha, uint8_t red, uint8_t green, uint8_t blue, uint8_t alpha...}
```

这样的话，在内存中，一般就可以用`uint8_t bitmap[width * components * 8][height]`来表示。

有了这样的知识，对照着就能看懂CGImage的BitmapInfo所表示的信息了。

![Bitmap的表示](https://developer.apple.com/library/content/documentation/GraphicsImaging/Conceptual/drawingwithquartz2d/Art/colorformatrgba32.gif)

## 静态图
只要Bitmap数据到手，后面的过程其实都大同小异。第三方解码器主要处理图像编码数据到原始Bitmap的解码过程，后续就可以通过固定的方式来得到CGImage或者UIImage，用于上层UI组件的渲染。

1. 第三方解码器获取图像Bitmap
2. 通过Bitmap创建CGImage
3. CGImage重绘（可选）
4. 生成上层的UIImage，清理

### 1. 第三方解码器获取图像Bitmap
根据自己的需要，可以选择对应的编码器，来获取图像Bitmap。这里我们以WebP的解码库[libwebp](https://developers.google.com/speed/webp/docs/api)为例子，其他解码器需要根据对应解码器的API处理，基本概念类似。

示例代码：

```objectivec
NSData *data; // 待解码的图像二进制数据

WebPData webpData;
WebPDataInit(&webpData);
webpData.bytes = data.bytes;
webpData.size = data.length;

// 初始化Config，用于存储图像的Bitmap，大小等信息
WebPDecoderConfig config;
if (!WebPInitDecoderConfig(&config)) {
    return nil;
}

// 判断是否能够解码
if (WebPGetFeatures(webpData.bytes, webpData.size, &config.input) != VP8_STATUS_OK) {
    return nil;
}

// 设定输出的Bitmap色彩空间，有Alpha通道选择RGBA，没有选择RGB 
bool has_alpha = config.input.has_alpha;
config.output.colorspace = has_alpha ? MODE_rgbA : MODE_RGB;
config.options.use_threads = 1;

// 真正开始解码，输出RGBA数据到Config的output中
if (WebPDecode(webpData.bytes, webpData.size, &config) != VP8_STATUS_OK) {
    return nil;
}

// 获取图像的大小
int width = config.input.width;
int height = config.input.height;
if (config.options.use_scaling) {
    width = config.options.scaled_width;
    height = config.options.scaled_height;
}

//RGBA矢量和对应的大小
uint8_t *rgba = config.output.u.RGBA.rgb;
size_t rgbaSize = config.output.u.RGBA.size;
```

截止到这里，我们基本上调用通过第三方解码库的接口，就完成了获取Bitmap的工作。一般来说，以RGBA来说，最少需要知道以下信息：图像RGBA数组，数组大小，图像宽度、高度、是否含有Alpha通道这几个，以便开始下一步的创建CGImage的过程

### 2. 通过Bitmap创建CGImage
有了图像的Bitmap数据之后，可以通过`CGImageCreate`来生成CGImage。对于RGBA的输入，需要的参数基本比较固定，以下代码基本上可以参考来复用。（需要注意，iOS上只支持[premultiplied-alpha](https://segmentfault.com/a/1190000002990030)，macOS可以支持非premultiplied-alpha）

示例代码：

```objectivec
// 通过RGBA数组，创建一个DataProvider。最后一个参数是一个函数指针，用来在创建完成后清理内存用的
CGDataProviderRef provider = CGDataProviderCreateWithData(NULL, rgba, rgbaSize, FreeImageData);
// 目标色彩空间，我们这里用的就是RGBA
CGColorSpaceRef colorSpaceRef = CGColorSpaceCreateDeviceRGB();
// Bitmap数据，如果含有Alpha通道，设置Premultiplied Alpha。没有就忽略Alpha通道
CGBitmapInfo bitmapInfo = has_alpha ? kCGBitmapByteOrder32Big | kCGImageAlphaPremultipliedLast : kCGBitmapByteOrder32Big | kCGImageAlphaNoneSkipLast;
// 通道数，有alpha就是4，没有就是3
size_t components = has_alpha ? 4 : 3;
// 这个是用来做色彩空间变换的指示，如果超出色彩空间，比如P3转RGBA，默认会进行兼容转换
CGColorRenderingIntent renderingIntent = kCGRenderingIntentDefault;
// 每行字节数（RGBA数组就是连续排列的多维数组，一行就是宽度*通道数），又叫做stride，因为Bitmap本质就是Pixel(uint_8)的二维数组，需要知道何时分行
size_t bytesPerRow = components * width;
// 创建CGImage，参数分别意义为：宽度，高度，每通道的Bit数（RGBA自然是256，对应8Bit），每行字节数，色彩空间，Bitmap信息，数据Provider，解码数组（这个传NULL即可，其他值的话，会将经过变换比如premultiplied-alpha之后的Bitmap写回这个数组），是否过滤插值（这个一般不用开，可以在专门的图像锐化里面搞），色彩空间变换指示
CGImageRef imageRef = CGImageCreate(width, height, 8, components * 8, bytesPerRow, colorSpaceRef, bitmapInfo, provider, NULL, NO, renderingIntent);
// 别忘了清理DataProvider，此时会调用之前传入的清理函数
CGDataProviderRelease(provider);

static void FreeImageData(void *info, const void *data, size_t size) {
    free((void *)data);
}
```

### 3. CGImage重绘
这一步其实是可选的，但是建议都加上这一步骤。虽然我们之前通过RGBA创建了CGImage，但是实际上，CALayer和上层的UIImageView这些渲染的时候，要求的色彩是限定的，不然会有额外的内存和渲染消耗，我们解码出来的rgba的格式可能并不是按照这样的色彩空间排列，因此建议进行一次重绘，即将CGImage重绘到一个CGBitmapContext之上。这个代码比较简单。

示例代码：

```objectivec
CGImageRef imageRef;
size_t canvasWidth = CGImageGetWidth(imageRef)
size_t canvasHeight = CGImageGetHeight(imageRef)
CGBitmapInfo bitmapInfo;
if (! has_alpha) {
    bitmapInfo = kCGBitmapByteOrder32Big | kCGImageAlphaNoneSkipLast;
} else {
    bitmapInfo = kCGBitmapByteOrder32Big | kCGImageAlphaPremultipliedLast;
}

// 参数意思很简单，由于没有数据，对于RGBA可以自动计算bytesPerRow
CGContextRef canvas = CGBitmapContextCreate(NULL, canvasWidth, canvasHeight, 8, 0, CGColorSpaceCreateDeviceRGB(), bitmapInfo);
if (!canvas) {
    return nil;
}
    
CGImageRef imageRef = image.CGImage;
size_t width = CGImageGetWidth(imageRef);
size_t height = CGImageGetHeight(imageRef);
// 画CGImage上去
CGContextDrawImage(canvas, CGRectMake(0, 0, width, height), imageRef);
CGImageRef newImageRef = CGBitmapContextCreateImage(canvas);
```

### 4. 生成上层的UIImage，清理
有了最终的用于显示CGImage，那么我们就可以生成一个UIImage来给UI组件显示了。注意如果需要有特殊的scale，orientation处理（比如说图像可能有额外的EXIF Orientation信息），需要在这一步加上。
由于是C接口，需要手动清理内存，除了CGImage相关的，也需要清理第三方库自己的内存分配。对于错误提前返回的清理内存，灵活运用`__attribute__((cleanup))`，设置一个返回函数前清理的Block，可以减少犯错的可能性

示例代码：

```objectivec
UIImage *image = [UIImage imageWithCGImage:imageRef scale:1 orientation:UIImageOrientationUp];

// 各种清理，省略
CGImageRelease(newImageRef);
CGContextRelease(canvas);
```

## 动态图
动态图的解码过程，其实很直观的想，我们目标就是需要对所有动图帧，都拿到Bitmap，解码到CGImage和UIImage就行了。这样想的话，其实步骤就比较明确了。

步骤：

1. 第三方解码器生成每帧的Bitmap
2. 重复静态图的2-3
3. 生成动图UIImage（参考Image/IO）

### 1. 第三方解码器生成每帧的Bitmap
不同解码器可能对于动图有特殊的解码过程，拿libwebp举例来说，libwebp的动图，需要用到它的demux模块，其他解码器自行参考对应的API。

同时，这里需要额外介绍一些概念。一般来说，动图格式的话不会直接将每帧原始的Bitmap都编码到文件中，这样得到的文件过于庞大（帧数 * 每帧Bitmap）。因此，会有Dispose Method的方式（可以参考WebP规范[Disposal method (D): 1 bit](https://developers.google.com/speed/webp/docs/riff_container)，[移动端图片格式调研](https://blog.ibireme.com/2015/11/02/mobile_image_benchmark/)）。简单点来说，对于动图来说，每一帧有一个参考画布，在前一帧画完以后，后一帧可以利用前一帧已画好的图像，仅仅改变前后变化的部分，从而减小整体大小。因此我们创建动图时，需要准备好一个CGBitmapContext当作画布，根据Disposal Method（如果为None，不清空canvas；如果为Background，清空为Background Color，一般就是直接清空成透明）

有了所有帧的Bitmap后，转成CGImage，UIImage，最后生成动图UIImage，这个在系列前篇已经介绍过了，不再赘述

示例代码：

```objectivec
- (void)decodeWebP {
	// 前期准备代码，直接略过，假设已经创建好canvas
	WebPDemuxer *demuxer;
	WebPIterator iter;
	if (!WebPDemuxGetFrame(demuxer, 1, &iter)) {
	    return nil;
	}
	    
	NSMutableArray<UIImage *> *images = [NSMutableArray array];
	double durations[frameCount];
	    
	do {
	    @autoreleasepool {
	        UIImage *image = [self drawnWebpImageWithCanvas:canvas iterator:iter];
	        int duration = iter.duration;
	        [images addObject:image];
	        durations[frame_num] = duration;
	    }
	    
	} while (WebPDemuxNextFrame(&iter));
	
	// 创建UIImage动图，和Image/IO的相同，这里就直接封装成方法，略过
	UIImage *animatedImage = [self animatedImageWithImages:images durations:durations];
	    
	// 清理……略
	WebPDemuxReleaseIterator(&iter);
}

- (nullable UIImage *)drawnWebpImageWithCanvas:(CGContextRef)canvas iterator:(WebPIterator)iter {
    // 这里是调用的前面静态图绘制的方法
    UIImage *image = [self rawWebpImageWithData:iter.fragment];
    if (!image) {
        return nil;
    }
    
    size_t canvasWidth = CGBitmapContextGetWidth(canvas);
    size_t canvasHeight = CGBitmapContextGetHeight(canvas);
    CGSize size = CGSizeMake(canvasWidth, canvasHeight);
    CGFloat tmpX = iter.x_offset;
    CGFloat tmpY = size.height - iter.height - iter.y_offset;
    CGRect imageRect = CGRectMake(tmpX, tmpY, iter.width, iter.height);
    // Blend
    BOOL shouldBlend = iter.blend_method == WEBP_MUX_BLEND;
    
    // 如果BlendMode开启，该帧应当混合画画布上，否则，应该覆盖，也就是清空指定范围后再重画
    if (!shouldBlend) {
        CGContextClearRect(canvas, imageRect);
    }
    CGContextDrawImage(canvas, imageRect, image.CGImage);
    CGImageRef newImageRef = CGBitmapContextCreateImage(canvas);
    
    image = [UIImage imageWithCGImage:newImageRef];
    CGImageRelease(newImageRef);
    
    // Dispose如果是Background，表示解码下一帧需要清空画布
    if (iter.dispose_method == WEBP_MUX_DISPOSE_BACKGROUND) {
        CGContextClearRect(canvas, imageRect);
    }
    
    return image;
}
```


## 渐进式解码
渐进式解码的概念，在系列前篇中已经介绍过了，一般来说，第三方解码器支持渐进式解码的接口都比较类似，通过提供二进制流不断进行Update，每次能够得到当前解码的部分的Bitmap，最后可以拿到完整的Bitmap。之后只需要参考静态图对应步骤即可。

这里还是以libwebp的接口为例，libwebp需要使用它的WebPIDecoder接口，来专门进行渐进式解码。注意，libwebp渐进式解码出来的Bitmap不会将未解码的部分自动填空，会保留随机的内存地址置，要么手动清空，要么画的时候仅仅画解码出来的高度部分。

示例代码：

```objectivec
NSData *data; // 输入的原始图像格式的二进制数据
UIImage *image;
WebPIDecoder *idec = WebPINewRGB(MODE_rgbA, NULL, 0, 0);
if (!idec) {
    return nil;
}

// 这里需要更新全部的数据，当然libwebp也有仅仅更新新增数据（非全部）的接口
VP8StatusCode status = WebPIUpdate(idec, data.bytes, data.length);
if (status != VP8_STATUS_OK && status != VP8_STATUS_SUSPENDED) {
    return nil;
}
    
int width = 0;
int height = 0;
int last_y = 0; // 已经解码出来的Bitmap数据的高度，即对应有效Bitmap的行数
int stride = 0;
// 然后可以拿到Bitmap数据和相应的图像信息了
uint8_t *rgba = WebPIDecGetRGB(_idec, &last_y, &width, &height, &stride);

// 和静图解码的过程类似，构造一个DataProvider，略过
CGDataProviderRef provider;
CGBitmapInfo bitmapInfo = kCGBitmapByteOrder32Big | kCGImageAlphaPremultipliedLast;
size_t components = 4;

//这里说一个坑，libwebp不能保证last_y以下的数据是全空的，所以一定注意，仅仅解码last_y范围内的Bitmap
CGImageRef imageRef = CGImageCreate(width, last_y, 8, components * 8, components * width, CGColorSpaceCreateDeviceRGB(), bitmapInfo, provider, NULL, NO, kCGRenderingIntentDefault);

// 为了能得到完整的图片高度，创建一个canvas来画图，不画的部分保持透明状态即可
CGContextRef canvas = CGBitmapContextCreate(NULL, width, height, 8, 0, CGColorSpaceCreateDeviceRGB(), bitmapInfo);

// 仅仅画last_y高度的图像（不是全部），注意CoreGraphics的坐标系统是右手系的，与UIKit的坐标相反
CGContextDrawImage(canvas, CGRectMake(0, height - last_y, width, last_y), imageRef);

// 拿到CGImage
CGImageRef newImageRef = CGBitmapContextCreateImage(canvas);
// 创建UIImage
image = [[UIImage alloc] initWithCGImage:newImageRef];
// 各种清理，省略
```

# 编码

编码过程其实比解码过程要简单得多，因为实际上，我们可以通过自带的接口，直接拿到当前UIImage的Bitmap数据，因此只要将Bitmap交给第三方编码库来进行编码，最后输出数据即可。

## 静态图
静态图的过程其实就可以直接分为两步：

1. UIImage获取Bitmap
2. 调用编码器进行编码

### 1. UIImage获取Bitmap
UIImage本身能够直接通过方法拿到对应的CGImage，这样只需要调用`CGImageGetDataProvider`就可以拿到对应的Bitmap数据的DataProvider了，直接上代码吧。

示例代码：

```objectivec
UIImage *image;
NSData *webpData; // Bitmap数据的容器
CGImageRef imageRef = image.CGImage;
    
size_t width = CGImageGetWidth(imageRef);
size_t height = CGImageGetHeight(imageRef);    
size_t bytesPerRow = CGImageGetBytesPerRow(imageRef); // 大部分编码器需要知道bytesPerRow，或者叫做stride
CGDataProviderRef dataProvider = CGImageGetDataProvider(imageRef);
CFDataRef dataRef = CGDataProviderCopyData(dataProvider);
uint8_t *rgba = (uint8_t *)CFDataGetBytePtr(dataRef);
```

### 2. 调用编码器进行编码
我们还是以libwebp来对WebP进行编码，libwebp对于静态图片的编码非常简单（动态图片需要调用另一套mux的API，在动图章节讲）

示例代码：

```objectivec
uint8_t *data = NULL; //编码输出的二进制数据
float quality = 100.0; // libwebp可以选择编码质量，影响输出文件大小和编码速度
size_t size = WebPEncodeRGBA(rgba, (int)width, (int)height, (int)bytesPerRow, quality, &data);
CFRelease(dataRef); // 编码后清理Bitmap数据
rgba = NULL;
    
if (size) {
    // success
    webpData = [NSData dataWithBytes:data length:size];
}
if (data) {
    WebPFree(data);
}
```

## 动态图

对于动态图来说，也就是将多帧的Bitmap输入到编码器即可。对于libwebp的动态图编码，需要利用到它的mux模块，它能够将多个编码成WebP的二进制流，最后mux合并一次，最终得到了动态WebP。因此我们需要利用之前的静态图编码的步骤，只需要依次遍历取图并编码，最后使用mux处理即可。

步骤：

1. 遍历每帧Bitmap，编码

### 1. 遍历每帧Bitmap，编码

示例代码：

```objectivec
NSData *data;
NSArray<UIImgae *> *images;
double durations[frameCount];
int loopCount;
// 创建mux
WebPMux *mux = WebPMuxNew();
if (!mux) {
    return nil;
}
// 遍历带编码的每帧
for (size_t i = 0; i < framesCount; i++) {
    NSData *webpData = [self encodedWebpDataWithImage:images[i]]; // 单帧编码后数据
    int duration = (int)durations[i] * 1000; // 单帧持续时长
    // 设置WebP每帧属性，包括Data，Blend，Disposal等
    WebPMuxFrameInfo frame = {.bitstream.bytes = webpData.bytes //省略}
    if (WebPMuxPushFrame(mux, &frame, 0) != WEBP_MUX_OK) {
        WebPMuxDelete(mux);
        return nil;
    }
}

// 设置动图本身的属性
WebPMuxAnimParams params;
params.bgcolor = 0, params.loop_count = loopCount;
WebPMuxSetAnimationParams(mux, &params);

WebPData outputData;
// 最后进行编码，拿到输出的二进制
WebPMuxError error = WebPMuxAssemble(mux, &outputData);
WebPMuxDelete(mux);
if (error != WEBP_MUX_OK) {
    return nil;
}
data = [NSData dataWithBytes:outputData.bytes length:outputData.size];
WebPDataClear(&outputData);
```

# 总结
第三方编解码其实相对于Image/IO来说，主要难度其实在于需要获取的Bitmap。开发者需要一点基本的图像知识，再者就是要能会用第三方编解码器的接口（一般来说第三方编解码器就是C或者C++写的，与OC和Swift交互也非常方便，至少不用像Java JNI那样调用）。之后只要按照通用的步骤，去编码和解码即可。

到这里的话，一般的大部分格式的编解码就基本没有问题了。当然，关于进阶的方面，比如图像的编解码性能优化，进阶的图像处理（Bitmap的几何变化，Alpha合成，位数转换等等）这就需要用到更低层的库vImage了，会在之后的系列教程中进行介绍。