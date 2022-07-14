---
title: WatchKit渲染原理以及SwiftUI桥接
date: 2019-12-10 17:04:16
categories: iOS
tags:
  - iOS
  - watchOS
  - SwiftUI
  - WatchKit
---

# WatchKit渲染原理以及SwiftUI桥接

## 背景

Apple Watch作为苹果智能穿戴设备领域的重头，自从第一代发布已经经历了6次换代产品，操作系统的迭代也已经更新到了watchOS 6。

不同于iPhone的App，watchOS上的大部分App都侧重于健康管理，并且UI交互以直观，快速为基准。在2015年WWDC上，苹果发布的watchOS的同时，面向开发者发布了WatchKit，以用于构建watchOS App。

![watchkit-app.jpg](https://help.apple.com/assets/61BCCC0198613862887EA61B/61BCCC2A98613862887EA62C/en_US/091b3af363505482da15ec78aa547a4d.png)

这篇主要讲了关于WatchOS上的App的架构介绍，基本概念，并深入分析了WatchKit的UI渲染逻辑，也谈了一些WatchOS和SwiftUI相关的问题。

其实写这个文章的最主要的原因，是在于自己前段时间写库时候，在SwiftUI与watchOS的集成中，遇到了相当多的问题，迫使我对WatchKit进行了一些探索和逆向分析，这里共享出来，主要原因有多个：

1. 能够了解WatchKit的背后实现细节，回答诸如这种问题：“为什么WatchKit使用Interface Object的概念，而不能叫做View“
2. 能够理解WatchKit的架构设计，作为库开发者提升自己的分层抽象，架构能力，甚至可以自己做一套类似WatchKit的实现（上层封装布局框架或者DSL）
3. 了解到SwiftUI和WatchKit之间的坑点在于什么，在开发时候遇到奇怪问题能够进行分析归因
4. 实在被逼无奈的时候，可以考虑利用渲染机制走UIKit（注意私有API风险）

## WatchKit架构介绍

一个标准WatchKit App，可以分为至少两个部分：

+ Watch App Target：只有Storyboard和资源，用来提供静态的UI层级，你不允许动态构建View树（可以隐藏和恢复）
+ Watch Extension：管理所有逻辑代码，Interface Controller转场，更新UI

如果没有接触过WatchKit，推荐参考这篇文章快速概览了解一下：[NSHipster - Watch​Kit](https://nshipster.cn/watchkit/)。只需要知道，我们的核心的UI构造单元，是Interface Object和Interface Controller，类似于UIKit的View和ViewController。

Interface Controller用于管理页面展示元素的生命周期，而Interface Object是管理Storyboard上UI元素的单元，且只能触发更新，无法获取当前的UI状态（setter-only）。

![](https://trymakedo.files.wordpress.com/2014/11/watch_app_lifecycle_simple_2x.png)

在watchOS 1时代，WatchKit采取的架构是WatchKit Extension代码，运行在iPhone设备上，于Apple Watch使用无线通信来更新UI，并且由于运行在iPhone上，可以直接访问到App的共享沙盒和UserDefaults。这受当时早期的Apple Watch硬件和定位导致的一种局限性。

在watchOS 2时代，为了解决1时候的更新UI延迟问题，WatchKit进行了改造，将Extension代码放到Apple Watch中执行，就在同样的进程当中，避免额外的传输。为了解决和iPhone的存储同步问题，与此同时推出了WatchConnectivity框架，可以与iPhone App进行通信。

![](https://developer.apple.com/library/archive/documentation/General/Conceptual/AppleWatch2TransitionGuide/Art/architecture_compared_2x.png)

## WatchKit UI布局原理

WatchKit本身设计的是一个完整的客户端-服务端架构，在watchOS 1时代，由于我们的Extension进程在iPhone手机上，而App进程在Apple Watch上，因此通信方式必定是真正的网络传输，苹果采取了WiFi-Direct+私有协议，来传输对应的数据。

watchOS 1时代的App性能表现很糟糕，一旦iPhone和Apple Watch距离较远，整个watchOS App功能基本是无法使用，只能重新连接。

在watchOS 2上，苹果取巧的把Extension进程放到了Apple Watch本身，而上层已有的WatchKit代码不需要大幅改变。但是，Apple并没有因为这个架构改变，而提供真正的UIKit给开发者。类似的，一些贯穿于iOS/macOS/tvOS的基本框架，Apple依旧把它保留为私有，包括：

+ CoreAnimation
+ Metal
+ OpenGL/ES
+ GLKit

开发者在watchOS上，除了使用WatchKit以外，只能采取SceneKit或者SpriteKit这种高级游戏引擎，来开发你的watchOS App。

虽然苹果这样做，有很多具体的原因，比如说兼容代码，比如性能考量，甚至还有从技术层面上强迫统一UI风格等等。不过随着watchOS 6的发布，watchOS终于有真正的UI框架了。

### 客户端

WatchKit的客户端，指的是Apple Watch App自带的WatchKit Extension部分。

在watchOS 1上，客户端的进程位于iPhone当中，而不是和Apple Watch在一起。之间的传输需要走网络协议。在watchOS 2中，之间的传输依旧保持了一层抽象，但是实际上最终等价于同进程代码的调用。

由Storyboard创建的WKInterfaceObject，一定会有与之绑定的WKInterfaceController，这些Controller会保留一个viewControllerID，用于向服务端定位具体的UIKit ViewController（后面提到）

WKInterfaceObject的**所有**公开API相关属性设置，比如width height，alpha, image等，均会最终转发到一个`_sendValueChanged:forProperty:`方法上。Value是对应的对象（CGFloat会转换为NSNumber，部分属性会使用字典），Property是这些属性对应的名称（如width，height，image，text等）。

根据是否WatchKit 2，会做不同的处理。WatchKit 2会经过Main Queue Dispatch分发，而Watch 1采取的是自定义的一个通信协议，通过和iPhone直连的WiFi和私有协议传输。

简单来说，等价于如下伪代码：

```objectivec
@implementation WKInterfaceObject
- (void)setWidth:(CGFloat)width {
  [self _sendValueChanged:@(width) forProperty:@"width"];
}

- (void)_sendValueChanged:(id<NSCoding>)value forProperty:(NSString *)property {
    NSDictionary *message = @{
      @"viewController": self.viewControllerID,
      @"key": "wkInterfaceObject",
      @"value": value,
      @"property": property,
      @"interfaceProperty": self.interfaceProperty
    };
    [[SPExtensionConnection remoteObjectProxy] sendMessage:message];
}
@end
```

### 服务端

这里的提到服务端，在watchOS 1时代其实就是Apple Watch上单独跑的进程，而在watchOS 2上，它和Extension都是在Apple Watch上，也实际上运行在同一个进程中。

对于每个watchOS App，它实际可以当作一个UIKit App。它的main函数入口是一个叫做WKExtensionMain的方法，里面做了一些Extension的初始化以后，就直接调用了
有UIApplicationMain。watchOS App有AppDelegate（类名为[SPApplicationDelegate](https://github.com/LeoNatan/Apple-Runtime-Headers/blob/master/watchOS/Frameworks/WatchKit.framework/SPApplicationDelegate.h)），会有一个全屏的root UIWindow当作key window。

![watchkit1](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2019-12-10/image/watchkit1.jpg)

#### UI初始化

在服务端启动后，它会加载Storyboard中的UI。对每一个客户端的Interface Controller，实际上服务端对应会创建一个View Controller，对应UIViewController的生命周期，会转发到客户端，触发对应的Interface Controller的willActivate/didAppear方法。

因此，watchOS创建了一个[SPInterfaceViewController](https://github.com/LeoNatan/Apple-Runtime-Headers/blob/master/watchOS/Frameworks/WatchKit.framework/SPInterfaceViewController.h)子类来统一做这个事情，它继承自[SPViewController](https://github.com/LeoNatan/Apple-Runtime-Headers/blob/master/watchOS/Frameworks/WatchKit.framework/SPViewController.h)，父类又继承自UIViewController，使用客户端传来的Interface Controller ID来绑定起来。

对于UI来说，每一种WKInterfaceObject，其实都会有一个原生的继承自UIView的类去做真正的渲染，比如：

+ WKInterfaceButton: [SPInterfaceButton](https://github.com/LeoNatan/Apple-Runtime-Headers/blob/master/watchOS/Frameworks/WatchKit.framework/SPInterfaceButton.h)，继承自`UIControl`
+ WKInterfaceImage: [SPInterfaceImageView](https://github.com/LeoNatan/Apple-Runtime-Headers/blob/master/watchOS/Frameworks/WatchKit.framework/SPInterfaceImageView.h)，继承自`UIImageView`
+ WKInterfaceGroup: [SPInterfaceGroupView](https://github.com/LeoNatan/Apple-Runtime-Headers/blob/master/watchOS/Frameworks/WatchKit.framework/SPInterfaceGroupView.h)，继承自`UIImageView`
+ WKInterfaceMap: [SPInterfaceMapView](https://github.com/LeoNatan/Apple-Runtime-Headers/blob/master/watchOS/Frameworks/WatchKit.framework/SPInterfaceMapView.h)，继承自`MKMapView`
+ WKInterfaceSwitch: [SPInterfaceSwitch](https://github.com/LeoNatan/Apple-Runtime-Headers/blob/master/watchOS/Frameworks/WatchKit.framework/SPInterfaceSwitch.h)，继承自`UIControl`
+ WKInterfaceTable: [SPInterfaceListView](https://github.com/LeoNatan/Apple-Runtime-Headers/blob/master/watchOS/Frameworks/WatchKit.framework/SPInterfaceListView.h)，继承自`UIView`

SPInterfaceViewController的主要功能，就是根据Storyboard提供的信息，构造出对应这些UIView的树结构，并且初始化对应的值渲染到UI上（比如说，Image有初始化的Name，Label有初始的Text）。实际上，这些具体的初始化值，都存储在Storyboard中，比如说，这里是一个简单的包含Table，每个TableRow是一个居中的Label，它对应的结构化数据如下：

```json
{
    controllerClass = "InterfaceController";
    items =     (
                {
            property = interfaceTable;
            rows =             {
                default =                 {
                    color = EFF1FB24;
                    controllerClass = "ElementRowController";
                    items =                     (
                                                {
                            alignment = center;
                            fontScale = 1;
                            property = elementLabel;
                            text = Label;
                            type = label;
                            verticalAlignment = center;
                        }
                    );
                    type = group;
                    width = 1;
                };
            };
            type = table;
        }
    );
    title = Catalog;
}
```

这些信息会在运行时用于构建真正的View Tree。

值得注意的是，watchOS由于本身的UI，这些SPInterfaceViewController的rootView，一定是一个容器的View。比如说一般的多种控件平铺的Storyboard会自带`SPInterfaceGroupView`，一个可滚动的Storyboard会自带一个`SPCollectionView`，等等。这里是简单的伪代码：

```objectivec
@implementation SPInterfaceViewController
- (void)loadView {
  Class rootViewClass;
  UIView *rootView = [[rootViewClass alloc] initWithItemDescription:self.rootItemDescription bundle:self.bundle stringsFileName:self.stringsFileName];
  self.view = rootView;
}
@end
```

#### UI更新

UI创建好以后，实际上我们的Extension代码会触发很多Interface object的刷新，比如说更新Label的文案，Image的图片等等，这些会从客户端触发消息，然后在服务端统一由AppDelegate接收到，来根据viewControllerID找到对应先前创建的SPInterfaceViewController。

```objectivec
@interface SPApplicationDelegate : NSObject <SPExtensionConnectionDelegate, UIApplicationDelegate>
@end

@implementation SPApplicationDelegate
- (void)extensionConnection:(SPExtensionConnection *)connection interfaceViewController:(NSString *)viewControllerID setValue:(id)value forKey:(NSString *)key property:(NSString *)key {
    if ([key isEqualToString:@"wkInterfaceObject"]) {
        SPInterfaceViewController *vc = [SPInterfaceViewController viewControllerForIdentifier:viewControllerID];
        [vc setInterfaceValue:value forKey:key property:property];
    }
}
@end
```

因此，拿到UIViewController以后，WatchKit会根据前面传来的interfaceProperty来定位，找到一个需要更新的View。然后向对应的UIView对象，发送对应的property和value，以更新UI。

```objectivec
@interface SPInterfaceImageView : UIImageView
@end
@implementation SPInterfaceImageView
- (void)setInterfaceItemValue:(id)value property:(NSString *)property {
    if ([property isEqualToString:@"width"]) {
        self.width = value.doubleValue;
    }
    if ([property isEqualToString:@"image"]) {
        self.image = value;
    }
    // ...
}
@end
```

后续的流程，就完全交给UIKit和CALayer来进行渲染了。

## 总结流程

![watchkit2](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2019-12-10/image/watchkit2.jpg)


通过这张图，其实完整的流程，我们可以通过调用栈清晰看到，如图各个阶段：

1. 开发者调用WKInterfaceObject的UI方法
2. 客户端的WKInterfaceObject统一封装发送消息
3. 传输层传输消息（watchOS 1走网络，watchOS 2实际上就Dispatch到main queue）
4. 服务端接收到消息，消息分发给对应的ViewController
5. ViewController分发消息给rootView（会递归处理）
6. View解码消息，得到对应的需要设置的UIKit属性和值
7. 调用UIKit的UI更新方法

可以看出来，其实WatchKit这边主要的工作就是抽象了一层Interface Object而不让开发者直接更新UIView。在watchOS 1时代这是一个非常好的设计，因为Extension进程在iPhone中，而App进程在Apple Watch上。但是到了watchOS 2以后，依然保留了这一套设计方案，实际上开发者能自定义的UI很有限。

## WatchKit与Long-Look Notification

watchOS除了本身的App功能外，还有一些其他特性，比如这里提到的Long-Look Notification。这是在Apple Watch收到推送通知时候展示的页面，它实际上类似于iOS上的Notification Extension，可以进行自定义的UI。

![](https://docs-assets.developer.apple.com/published/2d2f6b930c/52360133-b314-48c5-aa59-f2cb6c5e4e8f.png)

苹果这里面对Notification提供了3种类型，根据能不能动态更新UI/能不能响应用户点击可以分为：

+ Static Notification（固定UI，点击后关闭）
+ Dynamic Notification（可以更新UI，点击后关闭）
+ Dynamic Interactive Notification（可以更新UI，可以响应交互，不默认关闭）

和普通的WatchKit UI一样，Notification依然使用Storyboard构建。并且有单独的Storyboard Entry Point。在代码里面通过WKUserNotificationInterfaceController的方法`didReceive(_:)`，来处理接收到通知后的UI刷新，存储同步等等逻辑。

![](https://docs-assets.developer.apple.com/published/0599a724ac/8487cfa2-b872-481d-b0ae-409d1aaea6d1.png)

如图所示，整体的生命周期比较简单，可以参考苹果的文档即可：[Customizing Your Long-Look Interface](https://developer.apple.com/documentation/watchkit/enhancing_your_watchos_app_with_notifications/customizing_your_long-look_interface)

### Long-Look Notification原理

按照之前说的，WatchOS的Native App中，使用了SPApplicationDelegate作为它的AppDelegate，也直接实现了UNUserNotificationCenterDelegate相关方法。

当有推送通知出现时，如果watchOS App正处于前台，会触发一系列UserNotification的通知。类似于UIKit的逻辑，就不再赘述。

如果watchOS App未启动，那么会被后台启动（且不触发UserNotification的通知），对应Storyboard中的WKUserNotificationInterfaceController实例会被初始化。加载完成UI后，会调用`willActivate()`方法并自动弹起。

![watchkit4](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2019-12-10/image/watchkit4.jpg)


其实可以看出来，WatchKit主要做的事情，是在于watchOS App未启动时，需要对用户提供的WKUserNotificationInterfaceController，桥接对应的UserNotification接口和生命周期。

1. 当SPApplicationDelegate的`userNotificationCenter:willPresentNotification:withCompletionHandler:`被调用，它会向客户端发送消息，触发WKUserNotificationInterfaceController的`didReceive(_:)`方法
2. 当用户点击了Notification上面的按钮时，SPApplicationDelegate的`userNotificationCenter:didReceiveNotificationResponse:withCompletionHandler:`被调用，如果App不支持dynamic interactive notification，它会直接关闭通知，并唤起watchOS App到前台
3. 如果支持dynamic interactive notification(watchOS 5/iOS 12)，那么用户点击的Button/Slider之类，会调用WKUserNotificationInterfaceController上绑定的Target-Action，开发者需要手动在交互完毕后调用`performNotificationDefaultAction`，`performDismissAction`关闭通知（系统不再自动关闭通知），另外，系统给通知的最下方提供了一个默认的Dismiss按钮，点击后会强制关闭。

个人见解：之所以watchOS非要封装一层，主要原因是watchOS 1时代，不支持自定义通知；在watchOS 2时代，UserNotification这个框架还不存在，UIKit和AppKit都各自有一套接收Notification的实现，而WatchKit也照猫画虎搞了一套（当时就用的UILocalNotification）。UserNotification这个跨平台的通知库，是伴随着watchOS 3才出现的，但是已经晚了，因此WatchKit继续在已有的这个WKUserNotificationInterfaceController上新增功能。

其实可以看到，WKUserNotificationInterfaceController实际上提供的接口，基本完全等价于UserNotifications + UserNotificationsUI，方法名类似，有兴趣的话自行参考官方文档对比一下[watchOS Custom Notification Tutorial](https://developer.apple.com/documentation/watchkit/enhancing_your_watchos_app_with_notifications) 和 [iOS Custom Notification Tutorial](https://developer.apple.com/documentation/usernotificationsui/customizing_the_appearance_of_notifications)

## WatchKit和SwiftUI

在WWDC 2019上，苹果发布了新的全平台UI框架，SwiftUI。SwiftUI是一个声明式的UI框架，大量使用了Swift语法特性和API接口设计，提倡Single Source of Truth而不是UIKit一直以来的View State Mutation。

为什么专门要讲SwiftUI，因为实际上，SwiftUI才是Apple Watch上真正的完整UI框架，而WatchKit由于设计上的问题，无法实现**Owning Every Pixel**这一点，在我心中它的定位更类似于TVML的级别。

![swiftui](https://developer.apple.com/assets/elements/icons/swiftui/swiftui-96x96_2x.png)

关于SwiftUI在watchOS上的快速上手，没有什么比Apple官方文档要直观的了，有兴趣参考：[SwiftUI Tutorials - Creating a watchOS App](https://developer.apple.com/tutorials/swiftui/creating-a-watchos-app)

这里不会专门介绍SwiftUI的基础知识，后续我可能也会写一篇SwiftUI原理性介绍的文章。但是这篇文章，主要侧重一些SwiftUI在watchOS的独有特性和注意点，以及一些自己发现的坑。

### SwiftUI与WatchKit桥接

SwiftUI，允许桥接目前已有的WatchKit的Interface Object，就如在iOS上允许桥接UIKit一样。但是它能做的事情和概念其实完全不一样。

在iOS上，你能通过代码/Storyboard来构建你自己的UIView子类，并且你能构造自己的ViewController管理生命周期事件。这些都能通过SwiftUI的[UIViewRepresentable](https://developer.apple.com/documentation/swiftui/uiviewrepresentable)来桥接而来。与此同时，你还可以在你的UIKit代码中，来引入SwiftUI的View。你可以使用UIHostingController当作Child VC，甚至是对应的UIView（`UIHostingController.view`是一个私有类`_UIHostingView`，继承自UIView），是一种双向的桥接。

但是，正如之前提到，WatchKit设计是严重Storyboard Based，你不允许继承Interface Object。你不能使用SwiftUI来引入Storyboard自己构建好的Interface Object/Controller层级。不过相反的是，你可以使用WKHostingController，在Storyboard中去present或者push一个新的SwiftUI页面，实际是一种单向的桥接。

SwiftUI提供的[WKInterfaceObjectRepresentable](https://developer.apple.com/documentation/swiftui/wkinterfaceobjectrepresentable)，实际上它只允许你去绑定一些已有的系统UI到SwiftUI中（因为SwiftUI目前还不支持这些控件，比如InlineMovie，MapKit，不排除以后有原生实现）。这些对应的WatchKit Interface Object，在watchOS 6上面都加入了对应的init初始化方法，允许你代码中动态创建，这里是全部的列表：

+ WKInterfaceActivityRing
+ WKInterfaceHMCamera
+ WKInterfaceInlineMovie
+ WKInterfaceMap
+ WKInterfaceMovie
+ WKInterfaceSCNScene
+ WKInterfaceSKScene

桥接了Interface Object的View可以像普通的SwiftUI View一样使用，常见的SwiftUI的modifier（比如`.frame`, `.background`）也可以正常work。但是有一些系统UI有着自己提供的最小布局（比如MapKit），超过这个限制会导致渲染异常，建议采取scaleTransform处理。另外，请不要同时调用Interface Object的setWidth等概念等价的布局方法，这会导致更多的问题。

### 桥接原理

上文提到的所有可动态创建的Interface Object，根据我们之前的探索，它现在是没有绑定任何viewControllerID的，具体SwiftUI是怎么做的呢？

答案是，SwiftUI会对这些init创建的interfaceObject，手动通过UUID构造一个单独的新字符串，然后用这个UUID，创建一个新ViewController到WatchKit App中，插入到对应HostingController的视图栈里面。

它的初始化UI状态，通过一个单独的属性拿到（由每个子类实现，比如MapView，默认的经纬度是0,0）。整体伪代码如下：

```objectivec
@implementation WKInterfaceMap
- (instancetype)init {
    NSString *UUID = [NSUUID UUID].UUIDString;
    NSString *property = [NSString stringWithFormat:@"%@_%@", [self class], UUID];
    return [self _initForDynamicCreationWithInterfaceProperty:property];
}

- (NSDictionary *)interfaceDescriptionForDynamicCreation {
    return @{
        @"type" : @"map",
        @"property" : self.interfaceProperty,
    };
}
@end
```


另外，这种使用init注册的WKInterfaceObject，会保留一个对应UIView的weak引用，可以在运行时通过私有的`_interfaceView`拿到。SwiftUI内部在布局的时候也用到了这个Native UIView来实现。

![watchkit-swiftui2](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2019-12-10/image/watchkit-swiftui2.png)

### SwiftUI与watchOS Native App

通过从Native watchOS App的布局分析上来看，SwiftUI参考iOS上的方案，依旧是用了一个单独的UIHostingView来插入到Native App的视图层级中，也有对应的UIHostingController。

但是不同于iOS的是，SwiftUI会对每一个Push/Present出来的新View（与是否用了上面提到的WKInterfaceObjectRepresentable无关，这样设计的原因见下），额外套了一个叫做[SPHostingViewController](https://github.com/LeoNatan/Apple-Runtime-Headers/blob/master/watchOS/Frameworks/WatchKit.framework/SPHostingViewController.h)的类，它继承自上文提到的SPViewController。

每个UIHostingController套在了SPHostingViewController的Child VC中，对应View通过约束定成一样的frame，可以看作是一个容器的关系。

![watchkit3](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2019-12-10/image/watchkit3.jpg)

当你的SwiftUI View，含有至少一个WatchKit Interface Object之后，这个SPHostingViewController就起到了很大作用。它需要调度和处理上文提到的WatchKit消息。SPHostingViewController内部存储了所有interface的property，Native UIView列表，通过遍历来进行分发，走普通的WatchKit流程。它相当于起到一个转发代理的作用，让这些WatchKit的Interface Object实现不需要修改代码能正常使用。

![watchkit-swiftui1](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2019-12-10/image/watchkit-swiftui1.jpeg)


### SwiftUI与Long-Look Notification

到这里其实事情还算简单，但是还有一种更为复杂的情形。SwiftUI支持创建自定义的watchOS Long-Look UI。它提供了一个对应的WKUserNotificationHostingController（继承自WKUserNotificationInterfaceController），就像WatchOS App一样。

但是，试想一下：既然SwiftUI支持桥接系统Interface Object，如果我在这里的HostingView中，再放一个WatchKit Interface Object，会怎么样呢？答案依然是支持。

SPHostingViewController这个类兼容了这种极端Case，它转发所有收到的Remote/Local Notification，承担了原本WatchKit的WKUserNotificationInterfaceController的一部分责任（因为继承链的关系，它不是WKUserNotificationInterfaceController子类，但是实现了类似的功能）。因此实际上，SPHostingViewController内部除了上面提到的property, Native UIView列表外，还存储了对应Notification Action的列表，用于转发用户点击在通知上的动作来刷新UI。

## Independent watchOS App

在历史上，所有的watchOS App，都必须Bundle在一个iOS App中，换句话说，就算你的watchOS App是一个简单的计算器，不需要任何iPhone的联动和同步功能，你也必须创建一个能够在iOS上的App Store审核通过的App。因此制作一个watchOS App的前提变得更复杂，它需要一个iOS App。而且以这里的计算器来说，你不可以直接套一个简单空壳的iOS App，引导用户只使用Apple Watch，因为iOS App Store的审核将不会通过。这也是造成watchOS App匮乏的一个问题。

从watchOS 6之后，由于上述的一系列开发工具上和模式上的改动，苹果听取了开发者的意见，能够允许你创造一个独立的watchOS App，它不再不需要任何iOS App，直接从Apple Watch上安装，下载，运行。watchOS App也不再必须和iOS App有所关联。

### 开发配置

将一个已有的非独立watchOS App转变为独立App比较简单，你只需要在Xcode中选中的watchOS Extension Target，勾选`Supports Running Without iOS App Installation`即可。

注意，独立watchOS App目前并不意味着你不能使用WatchConnectivity来同步iPhone的数据。你依然可以在你的Extension Target中声明你对应的iOS App的Bundle ID。

注意，如果用户没有下载这个watchOS App对应的iOS App，那么WatchConnectivity的`WCSession.companionAppInstalled`的方法会直接返回NO，就算强制调用`sendMessage:`，也会返回不可用的Error，在代码里面需要对此提前判断。

### App Slicing

独立watchOS App会利用App Slicing，而非独立App不会。Apple Watch从Series 4开始采取了64位的CPU，而与此同时，由于用户的iPhone的CPU架构和Apple Watch的CPU架构是无关的（你可以在iPhone 11上配对一个Apple Watch Series 3，对吧），而watchOS App又是捆绑在ipa中的，这就导致你的ipa包中，始终会含有两份watchOS的二进制（armv7k arm64_32），用户下载完成后，在同步手表时只会用到一份，并且原始ipa中依旧会保留这份二进制。这是一种带宽和存储浪费。

对于独立watchOS App，可以直接从watchOS App Store下载，那么将只下载Slicing之后的部分，节省近一半的带宽/存储。值得注意的是，就算是独立watchOS App，依然可以从iPhone手机上操作，来直接安装到Apple Watch中，因为在Apple Watch小屏幕上的App Store搜索文本和语音输入的体验并不是很好。

## 总结

通过上面完整的原理分析，可以看到，WatchKit这一个UI框架，通过一种客户端/服务端的方案，由于抽象了连接，即使watchOS 1到watchOS 2产生了如此大的架构变化，对上层的API基本保持了相对不变。这一点对于库开发者值得参考，通过良好的架构设计能够平滑迁移。

不过实际从各个社交渠道的反馈，开发者对于WatchKit的态度并不是那么乐观，由于隐藏了所有真正能够操作屏幕像素的方案（无法使用Metal这种底层接口，也没有UIKit这种上层接口），导致WatchOS App的生态环境实际上并不是那么理想，很多App都是非常简单和玩具级别的项目。虽然这是可以归因于Apple Watch本身硬件性能的限制，但是和WatchKit提供的接口也脱离不了关系。

如果让我来重新设计WatchKit，可能在watchOS 2时代，就会彻底Deprecate目前的WatchKit，而是取而代之采取公开精简的UIKit实现来让开发者最大化利用硬件（类似于目前的UIKit在tvOS上的现状），同时，提供一个新的WatchUIKit来提供所有专为Apple Watch设计的UI和功能，比如Digital Crown，比如Activity Ring。

![watchkit-twitte](https://lf3-client-infra.bytetos.com/obj/client-infra-images/lizhuoli/f7dac35688c54f2e9ac1a605b4295a39/2019-12-10/image/watchkit-twitter.jpg)

SwiftUI为watchOS App提供了一个新的出路，它可以说是真正的能够发挥开发者能力来实现精致的App，而不再受限于系统提供的基本控件。而WatchKit，也已经完成了它的使命。相信之后的SwiftUI Native App将会为watchOS创造一片新的生态，Apple Watch也能真正摆脱“iPhone外设”这一个尴尬的局面。

## 参考资料

+ [App Programming Guide for watchOS](https://developer.apple.com/library/archive/documentation/General/Conceptual/WatchKitProgrammingGuide/index.html)
+ [WatchKit Catalog Example](https://developer.apple.com/library/archive/samplecode/WKInterfaceCatalog/Introduction/Intro.html)
+ [NSHipster - WatchKit](https://nshipster.com/watchkit/)
+ [WWDC - SwiftUI on watchOS](https://developer.apple.com/videos/play/wwdc2019/219/)
+ [SwiftUI Tutorials - Creating a watchOS App](https://developer.apple.com/tutorials/swiftui/creating-a-watchos-app)
+ [iOS Runtime Headers](http://developer.limneos.net)