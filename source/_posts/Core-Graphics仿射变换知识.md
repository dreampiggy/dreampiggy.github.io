---
title: Core Graphics仿射变换知识
categories: iOS
tags:
  - iOS
  - Objective-C
  - CoreGraphics
updated: '2016-11-01 23:36:06'
date: 2016-09-27 09:49:02
mathjax: true
---

> 这是补充记录关于CG的几何变换的一些知识，涉及到简单的矩阵变换

# 变换矩阵

在Core Graphics进行图层缩放、旋转、平移的时候，本质的操作就是使用`CGAffineTransform`这个3x2矩阵对象，与我们的`CGPoint`这个1x2的矩阵（其实就是对应就是[x,y]这个向量）进行矩阵相乘操作，得到的新矩阵就是变换后的新向量。一般通过CALayer得到的图层都是矢量，因此可以把整个图层进行相应的缩放、旋转、平移。

```objectivec
typedef struct CGAffineTransform { 
  CGFloat a; 
  CGFloat b; 
  CGFloat c; 
  CGFloat d; 
  CGFloat tx; 
  CGFloat ty; 
} CGAffineTransform;
```

这个结构体对应的矩阵如下（看不到LaTeX公式的请看[Apple Developer Document](https://developer.apple.com/reference/coregraphics/1455865-cgaffinetransformmake)）：

$ \begin{bmatrix} a & b & 0 \\\ c & d & 0 \\\ t\_{x} & t\_{y} & 1 \end{bmatrix} $

Apple采用了用[1]补齐1x3的向量，和用[0,0,1]的转置补齐的3x3的变换矩阵相乘来做仿射变换。虽然可能觉理论上可以直接用2x3变换矩阵和3x1的向量([x,y,1]的转置)运算，得到一个2x1的向量，省3个`CGFloat`的空间。但是由于这种变换操作叠加次数特别多，与其每次得到的向量结果再补齐[1]，还不如一次性就用一个1x3和3x3运算，用空间换取时间，这也许是QuartzCore的实现者的考虑吧。

$ \begin{bmatrix} x & y & 1 \end{bmatrix} \times \begin{bmatrix} a & b & 0 \\\ c & d & 0 \\\ t\_{x} & t\_{y} & 1 \end{bmatrix} = \begin{bmatrix} ax+cy+t\_{x} & bx+dy+t\_{y} & 1 \end{bmatrix} $


注意：iOS上坐标原点从屏幕左上方起，x轴指向右方，y轴指向下方。macOS的原点在屏幕左下方，x轴指向右方，y轴指向上方，要注意区别

# 平移

变换矩阵第三行的t\_x和t\_y对应的就是x、y的平移量，因此矩阵变换很简单，比如将[x,y]向量平移到[x+a,y+b]：

平移矩阵：
$ \begin{bmatrix} x & y & 1 \end{bmatrix} \times \begin{bmatrix} 1 & 0 & 0 \\\ 0 & 1 & 0 \\\ a & b & 1 \end{bmatrix} = \begin{bmatrix} x+a & y+b & 1 \end{bmatrix} $

对应API：

```objectivec
CGAffineTransform CGAffineTransformMakeTranslation(CGFloat tx, CGFloat ty);
```

# 缩放
缩放的本质，就是对向量[x,y]通过同时乘以相同的系数a，得到[ax,ay]，那么矩阵很简单，只需要一个对角矩阵，系数都为a就行

缩放矩阵：

$\begin{bmatrix} x & y & 1 \end{bmatrix} \times \begin{bmatrix} a & 0 & 0 \\\ 0 & a & 0 \\\ 0 & 0 & 1 \end{bmatrix} = \begin{bmatrix} ax & by & 1 \end{bmatrix} $

这个可以使用CA的API来简单构造（也可以直接自己初始化）：

```objectivec
CGAffineTransform CGAffineTransformMakeScale(CGFloat sx, CGFloat sy);
```

# 旋转
旋转的变换矩阵初看上去好像难以理解（各种cos、sin），其实就是一个简单的解方程的出来的结果，注意这里需要的是弧度，而且iOS上旋转的正弧度代表顺时针（macOS上就是正弧度是顺时针），需要注意

旋转矩阵：

$ \begin{bmatrix} x & y & 1 \end{bmatrix} \times \begin{bmatrix} \cos \theta & \sin \theta & 0 \\\ - \sin \theta & \cos \theta & 0 \\\ 0 & 0 & 1 \end{bmatrix} = \begin{bmatrix} x \cos \theta - y \sin \theta & x \sin \theta + y \cos \theta & 1 \end{bmatrix} $

推导过程：

![](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2022-07-14/image/4/ef/04946d0b15517c28f52f8aaf7a850.png)

$ P = (x,y) = (r\cos A , r\sin A) \\\ P^{'} = (r \cos B, r \sin B) = (r\cos(A + \theta), r\sin(A + \theta)) \\\ r\cos(A + \theta) = r\cos A \cos \theta - r\sin A \sin \theta = x \cos \theta - y \sin \theta \\\ r\sin(A + \theta) = r\sin A \cos \theta + r\cos A \sin \theta = y \cos \theta + x \sin \theta \\\ \therefore M = \begin{bmatrix} \cos \theta & \sin \theta & 0 \\\ - \sin \theta & \cos \theta & 0 \\\ 0 & 0 & 1 \end{bmatrix} $

对应API：

```objectivec
CGAffineTransform CGAffineTransformMakeRotation(CGFloat angle);
```

弧度可以用自带的定义，比如`M_PI_4`这些，也可以手动转换，比如用弧度、角度转换的宏：

```objectivec
#define RADIANS_TO_DEGREES(x) ((x)/M_PI*180.0) 
#define DEGREES_TO_RADIANS(x) ((x)/180.0*M_PI)
```

# 错切

错切，就是一种特殊的线性变换（不平移），指的是某一个坐标轴依赖不变，另一个轴线性变换，参考Wiki上的图片：
![](https://upload.wikimedia.org/wikipedia/commons/0/08/Eigen.jpg)

错切矩阵：

$ \begin{bmatrix} x & y & 1 \end{bmatrix} \times \begin{bmatrix} 1 & 0 & 0 \\\ m & 1 & 0 \\\ 0 & 0 & 1 \end{bmatrix} = \begin{bmatrix} x+my & y & 1 \end{bmatrix} $

这种变换没有提供专用的API，我们自己可以用`CGAffineTransformIdentity`创建一个矩阵，然后赋值矩阵中c或者b的值就行

# 叠加
既然了解了这些图层操作的矩阵变换，我们也可以自己定义矩阵，比如非线性变换(得到的向量不平行）、也可以把几个连续的变换串起来，这时候就要注意矩阵的运算顺序，比如先缩放50%，再旋转`M_PI_4`，再平移到[x+100,y]，那么等价于沿着45度平移50的距离（对应结果坐标就成了[x\*sqrt(2)/4, y\*sqrt(2)/4]）

对应API：

```objectivec
CGAffineTransform CGAffineTransformConcat(CGAffineTransform t1, CGAffineTransform t2); // 最通用
CGAffineTransform CGAffineTransformScale(CGAffineTransform t, CGFloat sx, CGFloat sy);
// 缩放
CGAffineTransform CGAffineTransformMakeRotation(CGFloat angle); //旋转
CGAffineTransform CGAffineTransformTranslate(CGAffineTransform t, CGFloat tx, CGFloat ty); // 平移
```

# 参考资料
[iOS Core Animation Advanced Techniques](https://github.com/AttackOnDobby/iOS-Core-Animation-Advanced-Techniques/blob/master/5-%E5%8F%98%E6%8D%A2/%E5%8F%98%E6%8D%A2.md)