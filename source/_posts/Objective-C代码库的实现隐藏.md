---
title: Objective-C代码库的实现隐藏
date: 2017-06-04 17:46:41
categories: iOS
tags:
  - iOS
  - Objective-C
---

> 虽然Swift现在是开发iOS推荐入手的最佳语言，但是对于代码库而言，最大的一个问题是Swift ABI仍然没有定下（今年发布的的Swift 4.0，依然放弃ABI稳定性，而注重于Swift源代码3.x->4.0的兼容性）。所以这就意味着Swift 3.x编译的二进制库，在Swift 4.0将无法链接，只能重新代码编译。看来这又将是Objective-C这门古老的语法，能够作为一些framework首选开发语言的一年。

对于一个代码库来说，有时候我们为了隐藏一些实现的细节，或者内部处理流程，需要编译到二进制进行分发，并提供Public Header来供其他开发者调用。

因此，开发代码库的时候，需要明确哪些API是对外公开的，可以由其他开发者调用。那些是库内部之间互相调用的，不应该由外部使用者调用。而Objective-C不像C++提供了private关键字来限制直接访问成员变量和成员方法。因此，就需要尽量避免私有属性和私有方法的定义出现在头文件中。只要不引入私有的头文件，那就无法直接访问这些属性和方法。

# 隐藏内部属性
私有属性，可以分成两种，一种是希望放到类内部而纯粹不想暴露给任何人的，可以叫做内部属性。一种是希望暴露到Private Header中，只限于引入该头文件的地方进行访问。

内部属性的声明非常简单，我们可以直接使用类扩展声明属性，而编译器会自动生成getter和setter，不需要任何额外工作。

```objectivec
// Person.m
@interface Person ()

@property (nonatomic, strong) NSObject *internalObject;

@end
```

## 改变属性修饰符
对于很多情况，我们需要对外暴露属性是readonly的，以防止使用者手动修改，但是内部流程的时候也需要这个属性，并且希望是readwrite的，这个在类扩展中直接可以重新声明已有的属性，并修改属性修饰符。

```objectivec
// Person.h
@interface Person

@property (nonatomic, strong, readonly) NSNumber *number;

@end
// Person.m
@interface Person ()

@property (nonatomic, strong, readwrite) NSNumber *number;

@end
```

注意，由于类扩展是可以在任何地方声明的（不限于.m实现文件），我们也可以把属性修饰符的修改，放到Private Header（可以用`+Private`后缀，也可以参考UIKit等框架起名为`UIKitInternal.h`）中，这样引入了Private Header的地方可以readwrite，没有引入的地方是readonly。

```objectivec
// Person+Private.h
@interface Person ()

@property (nonatomic, strong, readwrite) NSNumber *number;

@end
```

# 隐藏私有属性
但是很多时候，我们希望一些属性是私有的，即类实现处和引入了Private Header的地方才可以访问。这种时候就需要采取别的方式了。常见的方法是通过类扩展（主要针对类的实现文件可见）或者使用关联对象（主要针对类的实现文件不可见，如其他第三方库的类）两种方式。

## 类扩展（Class Extension）
### 通常情形
类扩展，不同于Category，最大的优势在于可以直接添加实例变量ivar到类的本身实现中，而Category是无法添加实例变量的。而在类扩展中声明的属性，也可以自动在编译期合成，同普通类声明属性的方式相同，不了解的参见：[CustomizingExistingClasses](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/CustomizingExistingClasses/CustomizingExistingClasses.html)。因此，实际上类扩展非常适合隐藏私有属性。

```objectivec
// Person+Private.h
@interface Person ()

@property (nonatomic, strong) NSString *privateID;

@end
```


### 自定义存取方法

对于通常case来说，这是非常好的解决方法（不用任何额外代码）。但是有一个问题，如果你想**自定义这个属性的存取方法**（比如，实例变量的惰性初始化），那就会遇到问题。因为属性合成的ivar，是只在类本身实现中创建的，在Category中无法创建，而且类的实现只能实现一次（在原始的`Person.m`中实现）。试想一下这样子的情况，就会出现编译错误：

```objectivec
// Person+Private.m
@implementation Person (Private)

- (NSString *)privateID
{
    if (!_privateID) { //Compile Error: undeclared identifier:_privateID
        _privateID = @"foo";
    }
    
    return _privateID;
}
```

**第一种解决方案：**

最简单的方式，就是直接把自定义的存取方法写在类本身实现文件中，然后在Category中暴露头文件，并用`@dynamic`来标记这个属性（否则由于Category看不到编译器自动生成的getter和setter会报warning）。自定义存取方式就和普通的写法一模一样。这相当于是一种把内部属性暴露出来的方法。不过容易导致耦合（因为其实我们的私有属性目标是用于和外部类交互的，不希望放到Private Category以外）。

```objectivec
//Person.m

@interface Person ()

@property (nonatomic, strong) NSString *privateID;

@end

@implementation Person

- (NSString *)privateID
{
    if (!_privateID) {
        _privateID = @"foo";
    }
    
    return _privateID;
}

@end

//Person+Private.h
@interface Person (Private)

@property (nonatomic, strong) NSString *privateID;

@end

//Person+Private.m
@implementation Person (Private)
@dynamic privateID;

@end
```

**第二种解决方案：**

当然，聪明的你自然会想到，既然Category没法定义ivar，那直接在类扩展中声明一个ivar不就行了。于是你可以这样写，但是这会出现一个编译警告：

```objectivec
// Person+Private.h
@interface Person () {
    NSString *_privateID;
}

@property (nonatomic, strong) NSString *privateID;

@end

// Person+Private.m
@implementation Person (Private)

// Compile Warning: category override method from class
- (NSString *)privateID
{
    if (!_privateID) {
        _privateID = @"foo";
    }
    
    return _privateID;
}
```

由于在类扩展中已经定义了属性，那么这个类在编译期间会自动合成存取方法，而在Private Category中覆盖就会覆盖本身合成的方法（虽然我们确实需要这样），但由于可以在多处定义Category，并且方法覆盖的顺序不定，无法保证你的存取方法就是真实想要的，所以这是编译警告。对于这种需要自定义存取方法的私有属性的case，应该在类扩展中定义ivar，在Private Category中定义属性并实现。注意由于在类扩展定义了ivar，不会自动生成getter+setter，**需要自行同时定义setter和getter**，注意对不同属性修饰符，比如`copy`的话setter需要用`[-copy]`，`weak`的话ivar要标注`__weak`等。

```objectivec
// Person+Private.h
@interface Person () {
    NSString *_privateID;
}

@end

@interface Person (Private)

@property (nonatomic, strong) NSString *privateID;

@end

// Person+Private.m
@implementation Person (Private)

- (NSString *)privateID
{
    if (!_privateID) {
        _privateID = @"foo";
    }
    
    return _privateID;
}

- (void)setPrivateID:(NSString *)privateID
{
    _privateID = privateID;
}
```

## 分类（Category）和关联对象
由于Objective-C的属性，其实就是ivar+getter方法+setter方法，我们可以在使用的地方通过Runtime来获取ivar。但是这种方式实际上来说是用的人非常少。第一个是复杂，第二个是不好使用一个通用的宏进行转换（因为ivar需要计算offset，根据不同类型的type encoding还不同……），而且对于这种需求来说优点大材小用了。因此我们一般都是使用关联对象（不了解的参见：[Associated Object](http://nshipster.com/associated-objects/)）

使用了关联对象后，为了方便不必要繁琐地书写`objc_getAssociatedObject`、`objc_setAssociatedObject`，我们可以定义一些宏来方便使用。由于属性是包括了语义和引用计数相关内容的，因此针对不同的属性修饰符，需要采用不同的宏来保证属性的语义。

属性修饰符的语义，可以参考clang官网的说明：[Objective-C Automatic Reference Counting](http://clang.llvm.org/docs/AutomaticReferenceCounting.html#property-declarations)，如下：

> `assign` implies `__unsafe_unretained` ownership.  
> `copy` implies `__strong` ownership, as well as the usual behavior of copy semantics on the setter.  
> `retain` implies `__strong` ownership.  
> `strong` implies `__strong` ownership.  
> `unsafe_unretained` implies `__unsafe_unretained` ownership.  
> `weak` implies `__weak` ownership.

由于属性修饰符只会影响setter，而不是getter，我们可以定义一个通用宏。对应的setter就需要单独根据情况编写。

```objectivec
#import <objc/runtime.h>
#define __GET_PROPERTY(property) objc_getAssociatedObject(self, @selector(property));
```

### strong(retain)

`strong`或者`retain`，就是所有对象的默认属性存取行为，隐含着对对象进行retain而使引用计数+1。这个可直接通过关联对象的行为设置。

宏：

```objectivec
#define __SET_STRONG(property) objc_setAssociatedObject(self, @selector(property), property, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
```

示例：

```objectivec
@property (nonatomic, strong) NSNumber *number;

- (NSNumber *)number
{
    return __GET_PROPERTY(number);
}

- (void)setNumber:(NSNumber *)number
{
    __SET_STRONG(number)
}
```


### copy
`copy`属性修饰，表示在调用setter的时候，首先需要对对象进行`copy`操作，然后再表示`strong`，在Objective-C中其实就是发送了`copyWithZone:`消息。这个可直接通过关联对象的行为设置。

宏：

```objectivec
#define __SET_COPY(property) objc_setAssociatedObject(self, @selector(property), property, OBJC_ASSOCIATION_COPY_NONATOMIC);
```

示例：

```objectivec
@property (nonatomic, copy) NSString *name;

- (NSString *)name
{
    return __GET_PROPERTY(name);
}

- (void)setName:(NSString *)name
{
    __SET_COPY(name);
}
```

### unsafe_unretained
`unsafe_unretained`和`assign`的语义是相同的，前者是ARC下加入的，而后者从MRC开始存在。一般来说，对于原始类型（`int`、`double`、`BOOL`、`NSInteger`)这些，由于本身就是copy by value，而且不存在对象和引用计数管理，因此属性声明用`assign`（很少见写`unsafe_unretained`，虽然允许）。

而对于对象而言，一般如果想表示不改变任何引用计数的弱引用，现在都用的是`weak`，因为`unsafe_unretained`不会像`weak`那样，在对象引用计数降到0被销毁后，自动置nil，而会保持指向的地址，因此可能随时都成为野指针而不安全。但是由于历史代码缘故，还有很少的代码库在用，姑且暂时保留。

这里我们定义一个宏，仅用于表示对象的`unsafe_unretained`和`assign`。这个可直接通过关联对象的行为设置。而对于原始类型的属性，参见下面的`assign`

宏：

```objectivec
#define __SET_UNSAFE_UNRETAINED(property) objc_setAssociatedObject(self, @selector(property), property, OBJC_ASSOCIATION_ASSIGN);
```

示例：

```objectivec
@property (nonatomic, unsafe_unretained) NSObject *unsafeObject;

- (NSObject *)unsafeObject
{
    return __GET_PROPERTY(unsafeObject);
}

- (void)setUnsafeObject:(NSObject *)unsafeObject
{
    __SET_UNSAFE_UNRETAINED(unsafeObject);
}
```

### assign
区别于上面针对对象的`unsafe_unretained `和`assign`语义，这里的`assign`特指对原始类型的属性修饰符。由于Runtime的Associated Object一定是一个Object，因此我们需要把原始类型进行装箱，封装为一个Object，在getter中拆箱，拿到真实的原始数据。这个过程由于我们一定是一个Object箱子，只装一个真实的原始数据，因此没有必要进行copy（箱子是唯一的，但是内容的原始数据来源是copy by value）。可以用`strong`来修饰。

对于不同的原始类型，装箱的方式不同，一般来说，对于数值类型（int、double、NSInteger），可以使用NSNumber来装箱。对于其他类型，比如结构体，可以使用NSValue来进行装箱（比如CGRect，NSRange, Pointer）。对于不同的装箱来说方式不同，因此不好在宏里面进行处理，直接接收一个装好箱的value就可以了。

宏：

```objectivec
#define __SET_ASSIGN(property, value) objc_setAssociatedObject(self, @selector(property), value, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
```

由于装箱方式不同，拆箱方式肯定不同。不过只要拿到箱子之后，自己根据类型来进行相应拆箱即可。

示例：

```objectivec
@property (nonatomic, assign) int age;
@property (nonatomic, assign) CGRect frame;

- (int)age
{
    NSNumber *value = __GET_PROPERTY(age);
    return value.intValue;
}

- (void)setAge:(int)age
{
    __SET_ASSIGN(age, @(age));
}

- (CGRect)frame
{
    NSValue *value = __GET_PROPERTY(frame);
    return value.CGRectValue;
}

- (void)setFrame:(CGRect)frame
{
    NSValue *value = [NSValue valueWithCGRect:frame];
    __SET_ASSIGN(frame, value);
}
```

### weak
`weak`属性指的是一个弱引用，不改变对象的引用计数，同时和`assign`和`unsafe_unretained`的最大区别，在于有着自动置nil的安全性质。一旦weak对象被销毁，该引用不会成为一个野指针，而会被立即置为nil，保证了安全。对于如今的现代Objective-C，能表示弱引用全部使用weak，应当避免使用`assign`和`unsafe_unretained`表示一个弱引用（就算考虑上性能问题，weak立即置nil采用了一个全局的weak表，由Runtime管理，开销和手动release基本一致，不太可能成为性能问题）。

由于`weak`的特殊性（全局weak表），关联对象本身就没有提供weak的语义行为，但是我们可以来模拟一个等价的行为。

**第一种解决方案：**
我们使用一个WeakContainer，只包含一个weak的属性，来存放真实的weak引用对象。这样，通过关联对象把整个WeakContainer关联到Category的属性上，然后存取使用的时候进行装箱和拆箱，解决方案即可。不过唯一的缺点是由于需要引入一个WeakContainer类，无法做到Header Only。

```objectivec
@interface WeakObjectContainer : NSObject

@property (nonatomic, weak) id object;

+ (instancetype)containerWithObject:(id)object;

@end

@implementation WeakObjectContainer

+ (instancetype)containerWithObject:(id)object
{
    WeakObjectContainer *container = [[WeakObjectContainer alloc] init];
    container.object = object;
    return container;
}

@end
```

宏：

```objectivec
#import "WeakObjectContainer.h"
#define __SET_WEAK(property) objc_setAssociatedObject(self, @selector(property), [WeakObjectContainer containerWithObject:property], OBJC_ASSOCIATION_RETAIN_NONATOMIC);
#define __GET_WEAK(property) [objc_getAssociatedObject(self, @selector(property)) object];
```


**第二种解决方案：**

为了做到Header only，我们需要借助一个匿名的block，首先定义一个weak引用指向属性值，然后block捕获它。这样子，只要把block关联到对象上，那么在getter的时候，通过直接执行block返回这个weak对象，就可以拿到真正的弱引用（实现时，block要用copy，而且要判空）。

宏：

```objectivec
#define __SET_WEAK(property) id __weak __weak_object = property; \
  id (^__weak_block)() = ^{ return __weak_object; }; \
  objc_setAssociatedObject(self, @selector(property), __weak_block, OBJC_ASSOCIATION_COPY);

#define __GET_WEAK(property) objc_getAssociatedObject(self, @selector(property)) ? ((id (^)())objc_getAssociatedObject(self, @selector(property)))() : nil;
```

示例：

```objectivec
@property (nonatomic, weak) id delegate;

- (id)delegate
{
    return __GET_WEAK(delegate);
}

- (void)setDelegate:(id)delegate
{
    __SET_WEAK(delegate);
}
```

### 自定义存取方法
自定义存取方法一般类的属性写法类似。比如说想要惰性初始化（即只有在第一次调用getter的时候，才会初始化属性）这里就不用`_name`来操作ivar，而是通过setter（当然也能用`__SET_* `宏来直接操作关联对象）就可以了。

示例：

```objectivec
- (NSString *)name
{
    NSString *name = __GET_PROPERTY(name);
    if (!name) {
        name = @"foo";
        [self setName:name];
    }
    return name;
}

- (void)setName:(NSString *)name
{
    __SET_COPY(name);
}
```

# 隐藏内部方法

## 类扩展实现类的内部方法
Objective-C没有真正意义上的私有方法，毕竟是C语言的超集嘛。但是Objective-C提供了一个类扩展语法，允许定义方法的接口。因此，只要我们在.m实现文件中定义了一些内部方法，就可以对外隐藏（当然，class-dump selector这些是可以直接调用的）

```objectivec
// Person.m

@interface Person ()

- (void)internalMethod;

@end

@implementation Person

- (void)internalMethod
{
    //...
}

@end
```

# 隐藏私有方法

## 分类实现类的私有方法

但一些情况下，我们需要很多库内部使用的类的私有方法（私有方法和内部方法虽然都不对外可见，但是其实目标不一样，私有方法一般是一些可以直接设置实例的状态，内部数据的危险方法，用于库内部的一些类之间，互相调用来使用。而内部方法一般放一些复杂流程处理，工具方法，是为了简化代码逻辑而使用的）这些方法需要和公开头文件的方法分开，保持对外隐藏。这时候就得用到Category。

我们可以把想要隐藏的私有方法，全部放到一个Private Category里面，库内部其他需要操作的地方，引用这个头文件即可。

```objectivec
// Person+Private.h

@interface Person (Private)

- (void)privateMethod;

@end

// Person+Private.m
@implementation Person (Private)

- (void)privateMethod
{
    //...
}

@end
```

## 暴露公开类的内部方法
对于公开类，我们有可能在实现中定义很多内部的方法，这些方法可能依赖一些上下文，或者是只在类扩展里面定义的属性（而不是在我们的Private分类里面）。当我们在库的其他地方，也想使用这些内部方法时，但是方法定义不在Private Header中（虽然实际上在类内部已经实现了）。我们需要一种方式来暴露类的内部方法。

```objectivec
//Person.m

- (void)publicMethod
{
    //...
    [self internalMethod];
}
//我们想暴露这个方法给其他引用了Private Header的地方使用
- (void)internalMethod
{
    //...
}
```

**第一种解决方案（错误示范）：**

使用一个Private Category，在头文件中暴露这个方法。但是由于是类本身而不是Category的方法，编译器会报找不到`internalMethod`的实现的warning（虽然它确实在本身的类中实现了）。我们是可以警告编译器，忽略warning，因为你知道实际上这个方法已经有了实现，只不过头文件没有暴露罢了。但是这种方法忽略警告，会忽略所有Private Category的方法检查，假如Person+Private.h中定义的方法真的没有在Person+Private.m中实现，也不会有任何警告，所以非常不推荐。

```objectivec
//Person+Private.h

@interface Person (Private)

- (void)internalMethod; //在类本身实现中的内部方法，想要暴露出去
- (void)privateMethod;

@end

//Person+Private.m

#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wincomplete-implementation"
@implementation Person (Private)
//...
@end
#pragma clang diagnostic pop
```

**第二种解决方案：**

使用类拓展（而不是Private Category）来暴露一个内部方法，实际上这才是最佳的方式，因为类扩展并不局限于任何地方，而且可以在任何.h或者.m中进行声明。实际上，类扩展只有@interface而不能有@implementation，是方法的接口而不是实现，不会出现方法重定义或者覆盖的问题。这样，我们在类扩展中加入实际类的内部方法即可。

```objectivec
//Person+Private.h

@interface Person ()

- (void)internalMethod;

@end

@interface Person (Private)

- (void)privateMethod;

@end

//Person+Private.m
@implementation Person (Private)
//...
@end
```

因为类扩展在编译器检查时，是需要对类本身实现的方法进行检查的，因此假如Person类本身没有实现internalMethod，编译器会报warning，这也保证了正确性。


# 总结

Objective-C毕竟已经几十年的语言了，语法层面上对抽象隐藏支持的就不好，不像Swift提供了四种访问控制关键字：`public`、`internal`、`fileprivate`、`private`，而且支持Module，再也不用担心命名和重定义问题了。不过Swift的现状，在Swift 4.0 ABI还不能稳定的情况下，代码库分发就只能使用源代码，这点对于很多开发者还有企业的影响确实比较大。不过了解Objective-C的实现也不是什么坏事，毕竟谁不定总会有需要写的的时候。希望这些代码库的接口与实现隐藏的方法，能够帮到一些平时没有接触过代码库开发的人吧。

# 资料
1. [完整Category属性宏](https://gist.github.com/dreampiggy/2f2da443874b329a2f5d12f546a7a0cf)