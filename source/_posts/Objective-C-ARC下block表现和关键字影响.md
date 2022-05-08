---
title: Objective-C ARC下block表现和关键字影响
categories: iOS
tags:
  - iOS
  - Objective-C
updated: '2016-06-17 01:34:07'
date: 2016-06-17 00:58:03
---

> Objective-C中的block是一种特别的结构，block与普通的instance不同的地方，不止更在于它的语法，更在于它的不同表现以及内存分配。

虽然block对于Objective-C来说已经早不新鲜了，但现如今很多博文讲述的block行为是基于MRC的，这与ARC下的表现是不同的。现代Objective-C也应该渐渐淘汰MRC和GC（其实GC已经淘汰了，在`macOS Sierra已经无法使用，iOS从来不支持`）本文所提及情况均限于ARC


# ARC下不同类型的block表现

很多博文都提到过，block通过llvm编译后，会生成对应的三种Class的实例变量，分别是：`NSStackBlock`、`NSGlobalBlock`、`NSMallocBlock`，分配区域分别位于进程的栈，TEXT段，堆。ARC下为了简化block的内存管理，以及性能优化，llvm会对不同情形下的block进行不同的类型变化，

```objectivec
int a = 1;
NSString *string = @"";

NSLog(@"%@", NSStringFromClass([^(){} class]));

NSLog(@"%@", NSStringFromClass([^(){
	int b = a;
} class]));

void (^block1)(void) = ^{
	int b = 2;
};
NSLog(@"%@", NSStringFromClass([block1 class]));

void (^block2)(void) = ^{
	NSString *b = string;
};
NSLog(@"%@", NSStringFromClass([block2 class]));
```

猜猜输出是什么？

```
__NSGlobalBlock__
__NSStackBlock__
__NSGlobalBlock__
__NSMallocBlock__
```

从这里也可以总结出规律：

1. 如果block不捕获任何外部变量（包括了`Primitives`（基本类型）），既没有对外部任何对象retain，也没有copy基本类型，那么这个block不存在任何内存泄漏的风险，也不需要引用计数，所以类型为`__NSGlobalBlock__`
2. 如果block捕获了外部变量（包括基本类型），但并没有被任何对象所引用（retian），而是直接被用于直接执行或者发送消息，那么它不会有任何引用计数问题，类型为`__NSStackBlock__`。由于位于栈区，这个block在函数返回后将被销毁，不过请放心，在ARC下，因为没有被任何对象引用，所以它始终是安全的（一旦之后被引用，立即会由Runtime负责通过`Block_copy()`转换为`__NSMallocBlock__`）
3. 通常情况下，如果block捕获了外部变量，且只要有对象持有（注意，无论引用是`__strong` 还是`__weak`还是`__copy`，参考[llvm-blocks](http://clang.llvm.org/docs/AutomaticReferenceCounting.html#blocks)），都会通过Runtime的`Block_copy()`和`Block_release()`，由编译器自动地将原本在栈的block拷贝到堆上，因此会像普通对象一样，交由ARC自动管理引用计数


# __block的影响

`__block`的关键字的作用大家都知道，默认情况下block是无法修改外部实例变量的（能读，也就是捕获），而经过__block修饰的实例变量可以通过block外修改。
但是的表现是否单纯可以概括为"捕获了一份实例变量到堆上，并修改了原来的引用"呢？

看看这个：

```objectivec
__block NSMutableArray *array1 = [[NSMutableArray alloc] initWithCapacity:10];
NSLog(@"object addr: %p, pointer addr: %p", array1, &array1);
void (^block1)(void) = ^{
	NSLog(@"object addr: %p, pointer addr: %p", array1, &array1);
};
block1();
NSLog(@"object addr: %p, pointer addr: %p", array1, &array1);

__block NSMutableArray *array2 = [[NSMutableArray alloc] initWithCapacity:10];
NSLog(@"object addr: %p, pointer addr: %p", array2, &array2);
void (^block2)(void) = ^{
	NSLog(@"object addr: %p, pointer addr: %p", array2, &array2);
};
block2();
NSLog(@"object addr: %p, pointer addr: %p", array2, &array2);
```

输出结果：

```
object addr: 0x7ffe8a60d800, pointer addr: 0x7fff5548c988
object addr: 0x7ffe8a60d800, pointer addr: 0x7ffe8a501db0
object addr: 0x7ffe8a60d800, pointer addr: 0x7fff5548c988

object addr: 0x7ffe8a401bf0, pointer addr: 0x7fff5548c950
object addr: 0x7ffe8a401bf0, pointer addr: 0x7ffe8a401e98
object addr: 0x7ffe8a401bf0, pointer addr: 0x7ffe8a401e98
```

从中可以看出，由于Objective-C所有的实例变量都分配在堆上，而对于ARC下的block，如果不加`__block`关键字，那么在捕获后，外部的引用（Objective-C的指针，其实就是一个对象的引用，类似于Java）不会受到任何影响（只是对引用进行了拷贝）。而如果使用`__block`的话，那么会将原来的引用修改（注意到地址值的变化）。

当然，实际上的`__block`捕获的实例变量，会额外追加一些字段，用于Runtime进行内存管理和处理引用（参考[block- marked-variables](http://clang.llvm.org/docs/Block-ABI-Apple.html#layout-of-block-marked-variables)）

```c
struct _block_byref_foo {
    void *isa; //isa指针
    struct Block_byref *forwarding; // block_byref结构体指针
    int flags;   //引用计数数,retianCount
    int size;   //分配大小
    typeof(marked_variable) marked_variable;  //实例变量的引用
};
```

因此可以知道，`__block`是好，但每个捕获变量都会多出至少20字节……虽然llvm的优化能力很好，盲目的标记`__block`也并不是一件好事（还会增加Runtime的开销和少量内存开销）


# 其他

顺便一说，最近在补iOS开发基础知识，发现这个[《招聘一个靠谱的iOS》答案-38题](https://github.com/ChenYilong/iOSInterviewQuestions/blob/master/01《招聘一个靠谱的iOS》面试题参考答案/《招聘一个靠谱的iOS》面试题参考答案（下）.md#38-在block内如何修改block外部变量)的说法是有问题的，不存在什么"block的变量copy到堆区"，只要你的block被引用，那么这个block一定在堆区，而且并不是所谓的"加入__block后才copy"，，真正变化的，只是那个引用的地址变了罢了。大家希望看到后不要被误导……


PS：
1. 如果想了解更多Runtime实现block的方式和具体block的内存分布，可以参考[llvm-block](http://clang.llvm.org/docs/Block-ABI-Apple.html)
2. 如果你真的需要MRC，可以参考这篇文章，附带一个小题目测试一下你的掌握情况[MRC-block-quiz](http://blog.parse.com/learn/engineering/objective-c-blocks-quiz/)