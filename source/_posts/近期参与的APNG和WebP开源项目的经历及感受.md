---
title: 近期参与的APNG和WebP开源项目的经历及感受
date: 2017-07-25 22:44:10
categories: Code
tags:
  - iOS
  - Web
  - JavaScript
  - C++
  - Image
  - 开源活动
---

> 这篇文章讲的是有关近期自己参与的几个开源项目的经历以及感受，不过巧合的是内容都和APNG和WebP这两种图像格式相关，阅读前建议先简单略读一下之前写的一篇文章：[客户端上动态图格式对比和解决方案](http://dreampiggy.com/2017/03/06/%E5%AE%A2%E6%88%B7%E7%AB%AF%E4%B8%8A%E5%8A%A8%E6%80%81%E5%9B%BE%E6%A0%BC%E5%BC%8F%E5%AF%B9%E6%AF%94%E5%92%8C%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88/)

# SDWebImage

[SDWebImage](https://github.com/rs/SDWebImage)是iOS平台上非常著名的图片下载、缓存库，而今年发布的SDWebImage 4.0在架构、接口变动并带来性能优化的同时，还支持了Animated WebP，因此我就高兴地去实验了一下，本想着可以替代之前使用的[YYImage](https://github.com/ibireme/YYImage)。但是一测试就发现渲染不正常，追回去看源码，发现SDWebImage的实现可以说是Too naive，压根没有按照WebP规范实现，大部分Animated WebP动图渲染都挂了，完全不可用（连测试都过不了，更别说生产环境了）。演示Demo在此：[AnimatedWebPDemo](https://github.com/dreampiggy/AnimatedWebPDemo)

总结出来的具体问题有以下几个：

1. SD绘制每帧的canvas大小不正确，在代码中，直接取得当前帧frame的大小，而非整个canvas的大小。这就导致最后生成的所有帧图片的数组中，每帧的图像大小不一致。这样渲染就会出现Bug（把所有帧拉伸到最大的那个图像大小上）。
2. SD的实现没有考虑过[WebP Disposal Method](https://developers.google.com/speed/webp/docs/riff_container#animation)，这个在很多动图中都会用到，因为能够重复利用前一帧的画布，来大幅减少最后生成动图的体积。常见的动图格式如GIF、APNG生成工具一般都采用这种Disposal，不然最终文件体积较大（但Google提供的WebP工具暂时没有自带这种优化的方式，一般使用第三方工具处理）。
3. UIKit自带的`UIImage.animatedImages`是非常弱的，SD并没有提供额外的抽象，而是直接用的这个接口。这带来的最大的问题，是UIImage需要提供一个图片数组和总时长，但是会对数组中每个图片平均分配时长。这与Animated WebP的规范就是不同的，后者允许对每帧设置一个不同的持续时长。
4. UIImageView直接设置`image`属性，是不支持设置循环次数的，会默认无限循环播放。而有些Animated WebP图片需要有循环次数。


既然知道这么多坑，想着SD毕竟是主流框架，就赶紧提了[Issue](https://github.com/rs/SDWebImage/issues/1951)，但是过了一周多，SD社区依然没有任何回应。于是尝试自己一个个解决。最后的成果也比较好，上述4个问题都得到了解决。

## Canvas大小问题
这个问题，可以直接通过libwebp的API，修改来使用canvas大小而不是frame大小，确保每帧最后的图像大小相同。其中，为了优化性能，对于透明的且frame比canvas要小的帧，绘制出来等价于将frame平移，然后所有剩余部分填充透明值。在使用CGBitmapContext的时候，可以直接在要传入的Bitmap矢量数据上做变换，减少绘制带来的开销（不过CGBitmapContext本身应该有优化，对于这个开销影响不大，但参考YYImage里面有这一步处理）

## Disposal Method支持
在绘制每帧时，按照Animated WebP规范，共享一个全局的CGContext当作canvas，根据每帧不同的Disposal Method，如果为Disposal Background，则在绘制完当前帧后清空CGContext，否则的话不处理，保留到下一帧继续绘制，最终测试和YYImage行为一致。

## 每帧持续时长相等问题

这个问题相对比较麻烦，因为你无法改动UIKit实现方式。最后想了一个比较Trick的方式。思路也简单，考虑这样的情况：第1帧持续时间：50ms，第2帧持续时间：100ms，第3帧持续时间：150ms，总共时长300ms。在依然使用UIImage的接口情况下（即数组每帧时长平均分配），那就可以提供一个[1, 2, 2, 3, 3, 3]（元素表示帧的编号）的图像数组，总时长300ms。这样的话平均分到每个元素是50ms，表面上看是6帧但实际渲染是3帧，也能达到最后的显示效果。这样实现的话，只要求一个所有帧持续时间的gcd，然后对每帧图像，按该帧所占的比例重复添加多次就可以了。

## 循环次数问题

由于SD的接口问题（用到了UIImageView的`sd_setImageWithURL`），是直接设置到`UIImageView.image`上的，而不是`animationImages`。而直接设置`image`会无视掉`animationRepeatCount`这个本来用于设置循环次数的属性。但如果SD框架自动设置`animationImages `属性的话，可能对使用者现有代码有影响（因为使用者还是用的`image`属性而不是`animationImages`属性），因此最后的解决方案，是在UIImage的扩展中，单独提供了一个`sd_webpLoopCount`的属性来获取循环次数，使用者可以自行设置UIImageView的属性，来实现指定循环次数。

举个例子，一般情形下（显示的动图超过循环次数后停到最后一帧上）就可以这样子用。

```objectivec
[imageView sd_setImageWithURL:webpURL completed:^(UIImage * _Nullable image, NSError * _Nullable error, SDImageCacheType cacheType, NSURL * _Nullable imageURL) {
    imageView.image = image.images.lastObject;
    imageView.animationDuration = image.duration;
    imageView.animationRepeatCount = image.sd_webpLoopCount;
    imageView.animationImages = image.images;
    [imageView startAnimating];
}];
```


这也算是一个解决方式吧。

## 感受

在写完这些，跑过单元测试，提交了[Pull request](https://github.com/rs/SDWebImage/pull/1952)之后，回头来看，才能真正感到YYImage的实力。

YYImage通过一个抽象层YYImageFrame，来把GIF、APNG和Animated WebP三种格式统一到一起，并且提供了Encoder和Decoder可以在三种格式来互相转换（这是重点）。关于绘制部分，还使用到了[Accelerate Framework](https://developer.apple.com/documentation/accelerate)，通过vImage的GPU加速的Bitmap变换来替代部分CGBitmapContext绘制。在缓存上，由于SD的抽象层存在，他使用了[ImageIO](https://developer.apple.com/documentation/imageio)来直接缓存CGImageSource（SD采用的是缓存了WebP的rawData），效率提升了很高也减少缓存大小（速度对比的话，可以从那个Demo工程看到，checkout到`fix_sd_animated_webp_canvas_size`分支上运行）。想想还是挺佩服ibireme这个人的，看来以后还要多使用YYKit并多学习。

# apng2webp

[apng2webp](https://github.com/Benny-/apng2webp)是一个转换APNG到Animated WebP图片的命令行工具，使用Python脚本 + 外部命令行工具来实现。在之前的工作需求中，使用到来优化APNG的大小，并且产出Animated WebP来让客户端使用。

为什么要转换APNG到Animated WebP呢，其实是因为APNG这个规范由于没有进入到PNG标准规范中，一直处于一个不温不火的地步，网上的APNG动图数量也不多，很多网页的PNG图片上传也不支持。虽然如今各大浏览器都对APNG提供了支持（Chrome 59正式支持了APNG，iOS很早从8.0支持，FireFox就是亲爹一直推动），但是客户端上，Android端没有相对靠谱的解码和渲染组件能够使用。反倒是Animated WebP借助Google亲爹推动，成为Android天生支持的图像格式，并且iOS上也有YYImage来提供支持。随着WebP的流行，越来越多设备估计都会支持WebP和Animated WebP，甚至最终超越GIF这个广为流行，但是已有30年历史，只支持256色和1位alpha通道的古老动图格式。

这次对apng2webp项目，主要是贡献了两个功能。

1. Windows的支持，即现在三大桌面端命令行均可使用
2. CI自动Build和Test

## Windows的支持

由于整个外部命令行工具(有四个工具，其中`cwebp`和`webpmux`是Google官方提供的，有Windows Build，另两个是源码编译）都是UNIX工具链下的，依赖几个C++库也挺常见，但是尝试过使用VS 2015源码编译跪了，使用[vcpkg](https://github.com/Microsoft/vcpkg)这个非常新的Windows上的C++包管理工具，又爆了一堆[link error](https://www.zhihu.com/question/62158323/answer/196189709)。对于我这种C++菜鸟来说，最后只好选择了直接上[Mysys2](http://www.msys2.org/)和MinGW-w64，一键`pacman -S`安装依赖，cmake makefile可用，跑了一遍测试也没问题，确实非常方便。由于MinGW-w64的编译产物，会依赖于libgcc，winpthreads，为了使最后的分发方便，于是在Windows上改用静态链接。

## CI和单元测试

关于Python的单元测试，由于这是一个简单的命令行工具，最后就通过引入pytest，直接对main函数和外部工具进行了测试，写起来也特别简单（自动匹配文件名和类名这点挺好）。用起来感觉比起Objective-C和Java的工具要好用多了。

在CI Build上，对于Linux和macOS的话，一般都会使用GitHub官方合作的[Travis CI](travis-ci.org)，配置使用yml语法，再加上一系列的Bash命令。而Windows上使用的[Appveyor](https://ci.appveyor.com/)也非常好用，自带了`VS 2012,2015,2017`，`Msys2`，`MinGW-w64`，`cmake`等一系列工具，上手开箱即用。配置的话注意要使用CMD或者PowerShell，如果不熟悉，甚至可以用Msys2装一些UNIX工具来搞定（好处之一）。

## 感受

总体来说，这个项目主要是苦力活，不过也算熟悉了一下UNIX工具在Windows上移植的一种手段，而且还学习到了pytest和开源项目的CI Build方式，也算有点意思吧。

# iSparta

[iSparta](https://github.com/iSparta/iSparta)是一个图形化的APNG和WebP转换工具，包含了很多功能（APNG合成，WebP转换，图片压缩等），虽说是开源项目，但是上一次提交已经是三年前了。而我最希望的APNG转换Animated WebP功能却没有实现（这也难怪，三年前Animated WebP规范还没出来）。大概看了一眼，使用的是[NW.js](https://nwjs.io/)（其实用的是改名前叫做`node-webkit`的东西），是一个和[Electron](https://electron.atom.io/)类似的，使用前端技术栈来构建跨平台应用的框架，本质上都是一个Chromium的运行环境来提供渲染，再加上node.js来提供JS Runtime。上手相对容易。

基本上的目标，是为了提供更好的GUI工具，因此主要就参考了一下iSparta的Issue，解决这几个问题：

1. 支持APNG转换Animated WebP
2. 支持i18n国际化

由于我并不是专业前端出身（大二学过一段时间前端基本知识和Node.js简单应用，也接触过React Native），经过近两天的奋斗，才终于磕磕碰碰完成。期间遇到过各种问题（NW.js的问题，node第三方库的问题，跨平台行为不一致的问题等等），不过在这里略过说一下重点吧。

## APNG支持Animated WebP

关于这个功能，自然可以想到上面的apng2webp命令行工具，不过由于apng2webp本身是Python写的脚本来调用外部工具，没必要在NW.js里打包一个Python环境。因此最后就决定直接在JS里，实现了相同逻辑的脚本来完成。不过实话说这部分花费的时间不长，在GUI布局上才是重头。大体框架参考了项目中的已有写法，但CSS的部分由于实在生疏（原项目有一些布局Hack），最后使用了[flexbox](https://css-tricks.com/snippets/css/a-guide-to-flexbox/)布局来搞定的。

## i18n国际化

在网页端支持i18n国际化，这是确实是以前未接触过的地方。考虑到这个项目有大量散落的HTML文本中硬编码了中文文字，而又没有使用类似于Angular、React这种先进的技术来支持模板，因此就需要自行解决。最开始思考了使用服务端渲染的解决方案（即NW.js当作浏览器，本地起node使用express当作服务端，来返回渲染好对应国际化后的HTML），但是遇到了问题，当作纯浏览器后，NW.js无法再使用node端的本地包，这也就意味着无法调用外部的命令行工具（相当于RPC了）。因此这种方案不可行。

再经过尝试后，最后使用的解决方案，是引入了[node-i18n](https://github.com/mashpie/i18n-node)和模板引擎（这里用的是[doT](olado.github.io/doT/)）。在项目目录下准备好i18n的文本资源（框架支持的是JSON格式）。然后在NW.js应用启动时加载一个空body的页面，执行JS来获取i18n后的字符串，再将这些字符串渲染到只有body的模板中，最后把国际化完成后的HTML body插入到原始的页面的body中。整个过程没有多余的开销（避免了模板未渲染前被显示出来，而且可以缓存模板结果，因为实际上给定一种locale，模板生成的HTML是固定的）。

## 感受

其实现在看看自己平时用到的应用，`Atom`、`VS Code`、`GitKraken`、`钉钉`，这些看起来已经足够复杂，也都能够用这种前端技术栈构建起来了。以前自己如果提到跨平台桌面客户端应用，第一反应就是Qt，不过现在看来，如果对前端技术栈有所了解，对性能和实时性要求不高，是可以使用Electron或者NW.js这种框架来构建。虽然曾经见过有人批判这些框架（体积庞大-打包了Chromium和Node；内存占用高，效率低下-WebKit渲染而不是原生UI组件），[reddit](https://www.reddit.com/r/programming/comments/64oqaq/electron_is_flash_for_the_desktop/)上甚至有讨论说这是新一代的Adobe Flash。

但我个人看来，不排斥这样的框架，只是感觉如今的解决方案并不是十分完美，这些前端栈技术写的客户端最大的问题其实是代码复用问题，基本上是各家有自己的一套组件，而且很多解决方案很Trick。我觉得更为理想的情况，是能够提供一套完整的解决方案，包含了开箱即用的UI组件（并非指Bootstrap这种通用Web UI组件，而是专门针对桌面客户端优化的，符合客户端的交互方式），能够开发，构建，测试，打包一站式自动处理，足够多的Native桥接（这也是一大痛点，见过一些应用又回过头在Electron里面使用Flash），更多的优化，比如共享Chromium容器-不必每个应用的带上200MB的运行环境。

总体来说，Electron或者NW.js这些框架的前途还是比较光明的，毕竟传统意义上的桌面应用开发成本还是太高，尤其是互联网公司的产品，追求跨平台的情况下，在成本，人力还有技术难点考虑来看，也是一个不错的选择。

# 总结

其实，这三个开源项目都是属于一时兴起才去贡献的，并不是为了而去专门寻找的，至于为什么都是WebP相关，或许真的是巧合吧。参与这些开源项目，虽然花费了一定的时间精力，但是获得的知识面上的提升确实非常大，包括但不限于：`WebP规范`、`Accelerate Framework`、`跨平台C++移植`、`Python单元测试`、`CI配置`、`NW.js`和`前端i18n`。

说实话，参与开源项目的时候，你会发现一些社区是很有意思的，你能够和不认识的人去合作，还能够直观感受到其他人对项目的关注，更能够接触很多你之前从没有接触过的技术栈。我不能说自己是一个愿意花费大量个人时间去贡献开源事业的人，但是其实很多项目参与门槛不是那么高，无论是你自己平时用到的软件、类库，甚至是一个小工具、脚本、翻译、教程，都可以试着参与一下。我觉得程序员的知识，并不是为了单纯为了打工搬砖，能够把自己的想法与他人分享也是一个相当大的乐趣，不是吗？

