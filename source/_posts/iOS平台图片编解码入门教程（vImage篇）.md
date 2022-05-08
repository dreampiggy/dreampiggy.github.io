---
title: iOS平台图片编解码入门教程（vImage篇）
date: 2017-11-12 10:32:45
categories: iOS
tags:
  - iOS
  - Image
---
> 这篇教程，是系列教程的第三篇，前篇名为iOS平台图片编解码入门教程（第三方编解码篇）。由于vImage已经属于较为底层框架，这一篇将不会特别着重图片封装格式的编解码，会介绍一些Bitmap级别的操作，包括了图像的色彩转换，Alpha合成、基本几何变换等实际用法。由于教程侧重是图像格式，所以不会介绍vImage强大的Convolution等知识，这方面涉及到数字图像处理的复杂知识，不是教程的目标

# 基本介绍

[vImage](https://developer.apple.com/library/content/documentation/Performance/Conceptual/vImage/Introduction/Introduction.html)是Apple的Accelerate库的一部分，侧重于高性能的图像Bitmap级别的处理。库本身全部是C的接口，而且不同于Core系列的（Core Graphics/Core Foundation）C接口，是比较贴近传统C语言的接口，不会有XXXRef这种贴心的定义，而且很多接口需要自己手动分配内存。

vImage按照功能，可以分为Alpha Compositing（Alpha合成）、Geometry（几何变换）、Conversion（色彩转换）、Convolution（卷积，用于图像滤镜）Morphology（形态学处理）等。这里主要介绍的，就是色彩转换，Alpha合成，以及几何变换的内容。

首先需要对vImage的基本接口有所了解，有这么几个概念：

+ `vImage_Buffer`: 对应Bitmap的数据，只有最基本的width、height、rowBytes(stride)以及data
+ `vImage_CGImageFormat`: 每个vImage的功能，会提供不同色彩格式的类似接口，比如会有ARGB8888，Planar8的同样功能。这里ARGB8888指的是ARGB排列，每通道占8个Bit，也就是一个Piexel占32Bit。而vImage还有一个常见的色彩格式Plane8，指的是只有一个通道（平面），按照顺序排列，比如`{R, R, R, R}`这样，更方便进行计算
+ `vImage_Flags`: 每个vImage接口，都会有一个`flags `参数来控制一些选项，比如说可以自己定义内存分配，背景色填充策略，重采样策略等，默认的是`kvImageNoFlags`
+ `vImage_Error`: 每个vImage的接口，都会返回这个result，来让用户确认是否成功，以及失败的原因，在Debug下比较有帮助

为了统一期间，以下的内容，都是基于ARGB8888色彩格式的输入来说明的。其他的情况处理，参考同名接口的不同格式即可。

# 色彩转换

色彩转换指的是将图像的Bitmap格式，从一个色彩格式，比如ARGB8888，转换到另一个色彩格式，比如说RGB888的功能。对于RGB来说，一般来说就是通道的增加和减少。当然还有RGB转为Planar8的情况。

vImage对这些色彩转换的功能，统一提供了方法`vImageConvert_AtoB`，比如ARGB8888转RGB888，就可以用下面的代码来处理。顺便通过这个代码，来简单了解vImage的API的基本用法。

先来定义几个简单的结构体，方便后续使用：

```objectivec
// 为了方便，我们首先直接定义好ARGB8888的format结构体，后续需要多次使用
static vImage_CGImageFormat vImageFormatARGB8888 = (vImage_CGImageFormat) {
    .bitsPerComponent = 8, // 8位
    .bitsPerPixel = 32, // ARGB4通道，4*8
    .colorSpace = NULL, // 默认就是sRGB
    .bitmapInfo = kCGImageAlphaFirst | kCGBitmapByteOrderDefault, // 表示ARGB
    .version = 0, // 或许以后会有版本区分，现在都是0
    .decode = NULL, // 和`CGImageCreate`的decode参数一样，可以用来做色彩范围映射的，NULL就是[0, 1.0]
    .renderingIntent = kCGRenderingIntentDefault, // 和`CGImageCreate`的intent参数一样，当色彩空间超过后如何处理
};
// RGB888的format结构体
static vImage_CGImageFormat vImageFormatRGB888 = (vImage_CGImageFormat) {
    .bitsPerComponent = 8, // 8位
    .bitsPerPixel = 24, // RGB3通道，3*8
    .colorSpace = NULL,
    .bitmapInfo = kCGImageAlphaNone | kCGBitmapByteOrderDefault, // 表示RGB
    .version = 0,
    .decode = NULL,
    .renderingIntent = kCGRenderingIntentDefault,
};
// 字节对齐使用，vImage如果不是64字节对齐的，会有额外开销
static inline size_t vImageByteAlign(size_t size, size_t alignment) {
    return ((size + (alignment - 1)) / alignment) * alignment;
}
```

接着，就是完整的转换代码：

```objectivec
+ (CGImageRef)nonAlphaImageWithImage:(CGImageRef)aImage
{
    // 首先，我们声明input和output的buffer
    __block vImage_Buffer a_buffer = {}, output_buffer = {};
    @onExit {
        // 由于vImage的API需要手动管理内存，避免内存泄漏
        // 为了方便错误处理清理内存，可以使用clang attibute的cleanup（这里是libextobjc的宏）
        // 如果不这样，还有一种方式，就是使用goto，定义一个fail:的label，所有return NULL改成`goto fail`;
        if (a_buffer.data) free(a_buffer.data);
        if (output_buffer.data) free(output_buffer.data);
    };
    
    // 首先，创建一个buffer，可以用vImage提供的CGImage的便携构造方法，里面需要传入原始数据所需要的format，这里就是ARGB8888
    vImage_Error a_ret = vImageBuffer_InitWithCGImage(&a_buffer, &vImageFormatARGB8888, NULL, aImage, kvImageNoFlags);
    // 所有vImage的方法一般都有一个result，判断是否成功
    if (a_ret != kvImageNoError) return NULL;
    // 接着，我们需要对output buffer开辟内存，这里由于是RGB888，对应的rowBytes是3 * width，注意还需要64字节对齐，否则vImage处理会有额外的开销。
    output_buffer.width = a_buffer.width;
    output_buffer.height = a_buffer.height;
    output_buffer.rowBytes = vImageByteAlign(output_buffer.width * 3, 64);
    output_buffer.data = malloc(output_buffer.rowBytes * output_buffer.height);
    // 这里使用vImage的convert方法，转换色彩格式
    vImage_Error ret = vImageConvert_ARGB8888toRGB888(&a_buffer, &output_buffer, kvImageNoFlags);
    if (ret != kvImageNoError) return NULL;
    // 此时已经output buffer已经转换完成，输出回CGImage
    CGImageRef outputImage = vImageCreateCGImageFromBuffer(&output_buffer, &vImageFormatRGB888, NULL, NULL, kvImageNoFlags, &ret);
    if (ret != kvImageNoError) return NULL;
    
    return outputImage;
}
```

## 任意色彩格式转换

除了一系列`vImageConvert_AtoB`的转换，vImage还提供了一个非常抽象的接口，叫做`vImageConvert_AnyToAny`，只需要你提供一个input format，一个output format，就可以直接转换。这个接口比较强大，不仅能够handler所有支持的色彩格式，而且还能支持`CVImageBuffer`（通过这个`vImageConverter`来构造）。所以一般如果做库封装，做一些色彩转换的case的时候，就可以试着用这个接口。

因此，我们之前的ARGB8888ToRGB888的色彩转换，可以这样写，更为通用。示例代码：

```objectivec
    vImageConverterRef converter = vImageConverter_CreateWithCGImageFormat(&vImageFormatARGB8888, &vImageFormatRGB888, NULL, kvImageNoFlags, &ret);
    if (ret != kvImageNoError) return NULL;
    ret = vImageConvert_AnyToAny(converter, &a_buffer, &output_buffer, NULL, kvImageNoFlags);
```


# Alpha合成

![Alpha合成](https://raw.githubusercontent.com/dreampiggy/vImageProcessor/master/Example/Screenshot/Screenshot1.png)


[Alpha合成](https://en.wikipedia.org/wiki/Alpha_compositing)指的是将两张含有Alpha通道的图（被Blend的叫做bottom，Blend的叫做top），通过一定的公式合成成为一张新的含Alpha通道的图，一般来说用于给图像添加遮罩、覆盖等，常见的图像处理软件都有这个功能。其实本质上来说，Alpha合成，就是对图像的每一个像素值，进行这样一个计算：

```c
resultAlpha = (topAlpha * 255 + (255 - topAlpha)
                 * bottomAlpha + 127) / 255
resultColor = (topAlpha * topColor + (((255 - topAlpha)
                 * bottomAlpha + 127) / 255) * bottomColor +  127)
                    / resultAlpha
```

公式看起来比较复杂，因此这里顺便可以介绍一下关于[premultiplied-alpha](https://segmentfault.com/a/1190000002990030)的概念，直观地说，就是将`(r, g, b, a)`预先乘以了对应的alpha通道的值，成为`(r * a, g * a, b * a, a)`。这个带来的好处，就是Alpha合成的时候，可以少一次乘法，而且简化了计算，成为这样子：

```c
resultColor = (topColor + (((255 - topAlpha)
                 * bottomAlpha + 127) / 255) * bottomColor +  127)
```

在vImage中，已经提供了一个接口来专门处理Alpha合成，针对nonpremultiplied的，是`vImageAlphaBlend_ARGB8888`，而针对premultiplied，是`vImagePremultipliedAlphaBlend_ARGB8888`。需要注意的是，这个接口要求的两个buffer，宽度和高度必须相等，因此，我们对于Color和Image的遮罩，需要进行处理，保证这两个buffer满足要求。

## Alpha Blend Color

这个用处，一般是用来做图像的遮罩的，可以对图像整体盖一层有透明度的颜色，比如说夜间模式，纯色滤镜等。根据上面说的，如果需要对一个Bitmap使用vImage进行Alpha Blend，我们需要保证两个buffer的宽度和高度相同，因此可以使用`vImageBufferFill_ARGB8888`填充整个Color来构造一个与输入图像Buffer相同宽高的新buffer，然后用它来进行Alpha Blend。

代码示例：

```objectivec
    CGImageRef aImage; // 输入的bottom Image
    CGColorRef color; // 输入的color
    __block vImage_Buffer a_buffer = {}, b_buffer = {}, output_buffer = {}; // 分别是bottom buffer，top buffer和最后的output buffer
    Pixel_8888 pixel_color = {0};
    const double *components = CGColorGetComponents(color);
    const size_t components_size = CGColorGetNumberOfComponents(color);
    // 对CGColor进行转换到Pixel_8888
    if (components_size == 2) {
        // white, alpha
        pixel_color[0] = components[1] * 255;
    } else {
        // red, green, blue, (alpha)
        pixel_color[0] = components_size == 3 ? 255 : components[3] * 255;
        pixel_color[1] = components[0] * 255;
        pixel_color[2] = components[1] * 255;
        pixel_color[3] = components[2] * 255;
    }
    // 填充color到top buffer
    vImage_Error b_ret = vImageBufferFill_ARGB8888(&b_buffer, pixel_color , kvImageNoFlags);
    if (b_ret != kvImageNoError) return NULL;
    // Alpha Blend
    vImage_Error ret = vImageAlphaBlend_ARGB8888(&b_buffer, &a_buffer, &output_buffer, kvImageNoFlags);
```


## Alpha Blend Image

上面说到了关于Color的Alpha Blend，不同于Color这种需要填充全部宽度，如果对于一个Image需要进行Alpha Blend，我们大部分情况都是需要制定一个起始点的，因为不能保证所有输入的两个Image的宽高相同。因此设计的时候，可以给用户提供一个point参数，以这个坐标点开始来绘制Alpha Blend，类似于很多图像编辑软件提供的图层功能。

由于vImage的Alpha Blend需要两个等宽高的Buffer，因此我们需要对用户提供的Top Image进行处理，通过平移变换移动到指定的Point以后，填充其余部分为Clear Color。最后进行Alpha Blend即可。

```objectivec
    CGImageRef aImage, bImage; // 输入的bottom Image和top Image
    __block vImage_Buffer a_buffer = {}, b_buffer = {}, c_buffer = {}, output_buffer = {};
    //c buffer指的是将top Image进行处理后的临时buffer，使得宽高同bottom image相同
    // 这里我们使用到了线性变换的平移变换，以(0,0)放置top image，然后偏移point个像素点，其余部分填充clear color，即可得到这个处理后的c buffer
    CGAffineTransform transform = CGAffineTransformMakeTranslation(point.x, point.y);
    vImage_CGAffineTransform cg_transform = *((vImage_CGAffineTransform *)&transform);
    Pixel_8888 clear_color = {0};
    vImage_Error c_ret = vImageAffineWarpCG_ARGB8888(&b_buffer, &c_buffer, NULL, &cg_transform, clear_color, kvImageBackgroundColorFill);
    if (c_ret != kvImageNoError) return NULL;
    // 略过output buffer初始化
    // 将bottom image和处理后的c buffer进行Alpha Blend
    vImage_Error ret = vImageAlphaBlend_ARGB8888(&c_buffer, &a_buffer, &output_buffer, kvImageNoFlags);
```

# 几何变换

几何变换，指的是将一个原始的Bitmap，通过线性方法进行处理，实现比如平移、缩放、旋转、错切等操作的图像处理技术。

可能大部分人已经知道了（之前也说过），Core Graphics的坐标系统，和UIKit的坐标系统，在Y坐标上是相反的。UIKit的使用的是Y轴正向垂直向下的左手系，而Core Graphics和普通的右手系直角坐标系相同。vImage也遵守了右手系，因此之后介绍的变换都是按照右手系的，如果想处理UIKit的坐标系，自己转换一下即可（一般就是取`image.height - offsetY`即可）

关于要介绍的的这些几何变换，虽然都最后可以统一到到线性变换上，只不过效率上可能相比单独的方法来说有所损耗，因此单独对每个功能所需要的vImage接口进行了介绍。关于线性变换不太理解的，可以参考一下之前的一篇教程：[Core Graphics仿射变换知识](http://dreampiggy.com/2016/09/27/core-graphicsfang-she-bian-huan/)

## 缩放

![缩放](https://raw.githubusercontent.com/dreampiggy/vImageProcessor/master/Example/Screenshot/Screenshot2.png)

缩放是最简单的一个处理过程，但是由于缩放之后，之前的同一个像素点，现在可能会映射到4个或者更多像素点，或者是原本4个像素点，现在需要映射到1个像素点。这就会涉及到一个叫做[图像重采样](https://en.wikipedia.org/wiki/Image_scaling)的过程。具体来说，就是对每一个像素，所在的Bitmap的子矩阵（比如3x3），通过一定的算法计算，得到对应的缩放以后的中心像素的值。同时，这个像素值可能变成浮点数，还需要进行处理，最后填到采样后的Bitmap相应的位置上。常见的简单处理有最邻近算法、双线性算法、双立方算法等。

vImage默认使用的是[Lanczos Algorithm](https://en.wikipedia.org/wiki/Lanczos_resampling)，具体的介绍可以参考Wikipedia和DSP相关的书籍。这里有一个直观的对比表现[网页](https://clouard.users.greyc.fr/Pantheon/experiments/rescaling/index-en.html)。如果想要更高画质的算法，可以提供`kvImageHighQualityResampling`参数，来使用`Lanczos5`算法。或者可以使用之后要谈的相对底层一点的错切API，来自定义你的重采样过程。

vImage提供了自带的`vImageScale_ARGB8888`方法，这里就简单举个例子（之前重复代码的都略过）：

```objectivec
    CGSize size; // 目标大小
    output_buffer.width = MAX(size.width, 0);
    output_buffer.height = MAX(size.height, 0);
    output_buffer.rowBytes = vImageByteAlign(output_buffer.width * 4, 64);
    output_buffer.data = malloc(output_buffer.rowBytes * output_buffer.height);
    if (!output_buffer.data) return NULL;
    // 进行缩放，输出到output buffer中
    vImage_Error ret = vImageScale_ARGB8888(&a_buffer, &output_buffer, NULL, kvImageHighQualityResampling);
```

## 裁剪

裁剪是指的将原始Bitmap，只裁出来指定矩形大小的部分，其余部分直接丢弃的过程。虽然vImage没有提供直接的API来处理这个流程（当然你是可以用vecLib的方法，直接对Bitmap进行矩阵操作，但是有点过于小题大做了）。但是实际上，这就是一个平移变换能够搞定的事情。我们只需要对输入目标的坐标的`CGRect`进行转换，将原始图像平移之后，再限制输出的Bitmap的大小，这样平移超出部分就会自动被裁掉。不需额外的处理，示例代码如下：

```objectivec
    CGRect rect; // 输入的目标rect
    output_buffer.width = MAX(CGRectGetWidth(rect), 0); // 输出宽度
    output_buffer.height = MAX(CGRectGetHeight(rect), 0); // 输出高度
    output_buffer.rowBytes = vImageByteAlign(output_buffer.width * 4, 64);
    output_buffer.data = malloc(output_buffer.rowBytes * output_buffer.height);
    if (!output_buffer.data) return NULL;
    
    // 使用平移来处理，X轴Y轴分别平移负向的minX，minY即可
    CGFloat tx = CGRectGetMinX(rect);
    CGFloat ty = CGRectGetMinY(rect);
    CGAffineTransform transform = CGAffineTransformMakeTranslation(-tx, -ty);
    vImage_CGAffineTransform cg_transform = *((vImage_CGAffineTransform *)&transform);
    Pixel_8888 clear_color = {0};
    vImage_Error ret = vImageAffineWarpCG_ARGB8888(&a_buffer, &output_buffer, NULL, &cg_transform, clear_color, kvImageBackgroundColorFill);
```

## 镜像

镜像顾名思义，就是将图像沿着某个轴进行翻转，比如沿X轴就是水平镜像，同一个像素点，对应的X坐标不变，Y坐标变为高度减去本身的Y坐标即可。

vImage对应的API，是`vImageVerticalReflect_ARGB8888`和`vImageHorizontalReflect_ARGB8888`，使用起来也比较简单。直接上一个简单的示例：


```objectivec
    BOOL horizontal;
    __block vImage_Buffer a_buffer = {}, output_buffer = {};
    // 省略
    vImage_Error ret;
    if (horizontal) {
        // 水平镜像
        ret = vImageHorizontalReflect_ARGB8888(&a_buffer, &output_buffer, kvImageHighQualityResampling);
    } else {
        // 垂直镜像
        ret = vImageVerticalReflect_ARGB8888(&a_buffer, &output_buffer, kvImageHighQualityResampling);
    }
```



## 旋转

旋转也是非常常见一个图像几何几何变化。具体坐标的变化就是对旋转的角度，求对应三角函数到X轴和Y轴的投影结果，比较直观。

vImage对旋转也提供了一个非常方便的API，角度是弧度值，按照顺时针方向进行。另外，由于输出的Buffer的大小会限制图像大小，而旋转后可能超出原图大小，我们需要对输出的大小也计算出对应的新的大小。示例代码：

```objectivec
    CGFloat radians; //旋转的弧度
    CGSize size = CGSizeMake(a_buffer.width, a_buffer.height);
    // 这里直接借用CG的方法来计算旋转后的大小，方便
    CGAffineTransform transform = CGAffineTransformMakeRotation(radians);
    size = CGSizeApplyAffineTransform(size, transform);    output_buffer.width = ABS(size.width);
    output_buffer.height = ABS(size.height);
    output_buffer.rowBytes = vImageByteAlign(output_buffer.width * 4, 64);
    output_buffer.data = malloc(output_buffer.rowBytes * output_buffer.height);
    if (!output_buffer.data) return NULL;
    Pixel_8888 clear_color = {0};
    // 旋转操作，多余部分填充Clear Color
    vImage_Error ret = vImageRotate_ARGB8888(&a_buffer, &output_buffer, NULL, radians, clear_color, kvImageBackgroundColorFill | kvImageHighQualityResampling);
```

## 错切

![错切](https://raw.githubusercontent.com/dreampiggy/vImageProcessor/master/Example/Screenshot/Screenshot3.png)

[错切](https://en.wikipedia.org/wiki/Shear_mapping)是一种特殊的线性变换，直观的介绍可以从Wikipedia上看，也可以参考之前的另一篇教程。主要的参数有一个m值，表示对应参考坐标的缩放倍数。

在vImage中，错切变换是相对底层的接口，实际上，线性变换是通过这三个接口（错切、旋转、镜像）来实现的。错切的接口，比如水平错切对应的是`vImageHorizontalShear_ARGB8888`，参数算是最多的一个，稍微详细介绍一下：

+ `srcOffsetToROI_X`: 错切定位点水平偏移量，具体指的就是左上角那个像素点，在经过旋转的映射后，水平偏移的距离，会影响最后图像（除去Buffer的宽度限制）的整体宽度
+ `srcOffsetToROI_Y`: 错切定位点的垂直偏移量，类似水平值
+ `xTranslate`: 错切完成后的水平平移距离
+ `shearSlope`: 错切的弧度值，顺时针
+ `filter`: 用来自定义重采样的方法，一般用自带的`vImageNewResamplingFilter`，或者也可以提供一个函数指针构造对应的重采样过程。会用到一个scale参数，表示这个重采样对应的缩放倍数，也就是错切的m值
+ `backgroundColor`: 背景填充色

对应的示例代码：


```objectivec
    CGVector offset; // 定位点偏移量
    CGFloat translation; // 水平平移量
    CGFloat slope; // 旋转弧度
    CGFloat scale; // 对应错切的m值
    output_buffer.width = MAX(a_buffer.width - offset.dx, 0); //这里需要同时减去水平定位点的偏移
    output_buffer.height = MAX(a_buffer.height - offset.dy, 0); // 同理
    output_buffer.rowBytes = vImageByteAlign(output_buffer.width * 4, 64);
    output_buffer.data = malloc(output_buffer.rowBytes * output_buffer.height);
    if (!output_buffer.data) return NULL;
    
    Pixel_8888 clear_color = {0};
    // 这里示例就用默认的重采样方法
    ResamplingFilter resampling_filter = vImageNewResamplingFilter(scale, kvImageHighQualityResampling);
    vImage_Error ret;
    if (horizontal) {
        // 水平错切
        ret = vImageHorizontalShear_ARGB8888(&a_buffer, &output_buffer, offset.dx, offset.dy, translation, slope, resampling_filter, clear_color, kvImageBackgroundColorFill);
    } else {
        // 垂直错切
        ret = vImageVerticalShear_ARGB8888(&a_buffer, &output_buffer, offset.dx, offset.dy, translation, slope, resampling_filter, clear_color, kvImageBackgroundColorFill);
    }
    vImageDestroyResamplingFilter(resampling_filter);
```

## 线性变换

最后再来说通用的线性变换吧，这个其实在之前的功能中已经用到过了，vImage有兼容Core Graphics的`CGAffineTransform`的结构体`vImage_CGAffineTransform`，两个结构体对应的内存布局是一样的，直接强制转换过去就可以了，不需要单独赋一遍。关于通用线性变换的内容就不再赘述了，有兴趣可以查看相关资料，或者之前的教程：[Core Graphics仿射变换知识](http://dreampiggy.com/2016/09/27/core-graphicsfang-she-bian-huan/)

示例代码：

```objectivec
    CGAffineTransform transform; // 输入的CG变换矩阵
    vImage_CGAffineTransform cg_transform = *((vImage_CGAffineTransform *)&transform); // 结构一样，直接强转
    Pixel_8888 clear_color = {0};
    // 线性变换
    vImage_Error ret = vImageAffineWarpCG_ARGB8888(&a_buffer, &output_buffer, NULL, &cg_transform, clear_color, kvImageBackgroundColorFill | kvImageHighQualityResampling);
```


# 总结

vImage是一个比较底层的图像Bitmap处理的库，在这里介绍了关于色彩转换、Alpha合成、几何变换等基本知识。相比于简单的Core Graphics的处理，能够提供更为复杂的参数控制，并且带来较高的性能。对于很多图像密集处理软件处理来说，用Core Graphics显的比较低效，因此可以考虑vImage。

但是vImage强大之处远不在这里，里面还包含了类似图像卷积，形态处理等，可以对复杂滤镜进行支持，类似于[GPUImage](https://github.com/BradLarson/GPUImage)。这些功能都需要数字图像处理相关知识，在这种教程系列就不会介绍了。

对于这篇教程的示例代码，其实我写了个非常简单的库，放到GitHub上了：[vImageProcessor](https://github.com/dreampiggy/vImageProcessor)，有兴趣的可以去参考一下，希望能够用于自己的图片处理相关框架中。

由于自己完全是业余兴趣，工作和图像处理基本不相关，并不打算深入学习数字图像处理的知识，因此这个教程可能就会暂时告一段落了。最后，之所以写这篇教程，是因为自己想要参考一下vImage的教程，却发现只会搜出来一堆互相抄袭的内容，而且大部分都是关于图像滤镜的，对于图像处理本身不会太多介绍。我希望这系列教程，能给同样对图像编解码、图像处理有一点兴趣的人，提供一个相对简单且清晰的入门概览吧。