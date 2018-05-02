---
title: 优化APP启动时间实践 iOS
date: 2018-04-12 18:27:40
tags: [iOS]
categories: [iOS]
---

## 前言

当用户按下home键的时候，iOS的App并不会马上被kill掉，还会继续存活若干时间。理想情况下，用户点击App的图标再次回来的时候，App几乎不需要做什么，就可以还原到退出前的状态，继续为用户服务。这种持续存活的情况下启动App，我们称为热启动，相对而言冷启动就是App被kill掉以后一切从头开始启动的过程。我们这里只讨论App冷启动的情况。

对于冷启动来说，启动时间是指从用户点击 APP 那一刻开始到用户看到第一个界面这中间的时间。我们进行优化的时候，我们将启动时间分为 `pre-main` 时间和 `main` 函数到第一个界面渲染完成时间这两个部分。
因为 APP 的入口在 `main` 函数 ，在 main 函数之后我们的代码才会执行。

这里有两个阶段
```swift
1. pre-main阶段

1.1. 加载应用的可执行文件

1.2. 加载动态链接库加载器dyld（dynamic loader）

1.3. dyld递归加载应用所有依赖的dylib（dynamic library 动态链接库）

2. main()阶段

2.1. dyld调用main() 

2.2. 调用UIApplicationMain() 

2.3. 调用applicationWillFinishLaunching

2.4. 调用didFinishLaunchingWithOptions
```
我们把 `pre-main`阶段称为 `t1`，`main()`阶段一直到__首个页面加载完成__称为 `t2`。
## t1 时间的优化分析
`t1`部分主要参考自[APP启动优化的一次实践](https://icetime17.github.io/2018/01/01/2018-01/APP%E5%90%AF%E5%8A%A8%E4%BC%98%E5%8C%96%E7%9A%84%E4%B8%80%E6%AC%A1%E5%AE%9E%E8%B7%B5/)

其中 `t1`苹果提供了内建的测量方法, Xcode 中 `Edit scheme -> Run -> Auguments 将环境变量 DYLD_PRINT_STATISTICS 设为 1`
```swift
//结果为
Total pre-main time: 1.4 seconds (100.0%)
dylib loading time: 1.3 seconds (89.4%)
rebase/binding time:  36.75 milliseconds (2.5%)
ObjC setup time:  35.65 milliseconds (2.4%)
initializer time:  80.97 milliseconds (5.5%)
slowest intializers :
libSystem.B.dylib :  12.63 milliseconds (0.8%)
//解读
1、main()函数之前总共使用了1.4s

2、在94.33ms中，加载动态库用了1.3s，指针重定位使用了36.75ms，ObjC类初始化使用了35.65ms，各种初始化使用了80.97ms。

3、在初始化耗费的80.97ms中，用时最多的初始化是libSystem.B.dylib。
```
可以看到,我的 `dylib loading time` 花费了 `1.3s`时间，

其中各部分的作用是
```swift
加载dylib
分析每个dylib（大部分是iOS系统的），找到其Mach-O文件，
打开并读取验证有效性，找到代码签名注册到内核，
最后对dylib的每个segment调用mmap()。
```

```swift
rebase/bind
dylib加载完成之后，它们处于相互独立的状态，需要绑定起来。

在dylib的加载过程中，系统为了安全考虑，引入了ASLR（Address Space Layout Randomization）技术和代码签名。
由于ASLR的存在，镜像（Image，包括可执行文件、dylib和bundle）会在随机的地址上加载，和之前指针指向的地址（preferred_address）会有一个偏差（slide），dyld需要修正这个偏差，来指向正确的地址。
Rebase在前，Bind在后，Rebase做的是将镜像读入内存，修正镜像内部的指针，性能消耗主要在IO。
Bind做的是查询符号表，设置指向镜像外部的指针，性能消耗主要在CPU计算。
```

```swift
OC setup
OC的runtime需要维护一张类名与类的方法列表的全局表。
dyld做了如下操作：

对所有声明过的OC类，将其注册到这个全局表中（class registration）
将category的方法插入到类的方法列表中（category registration）
检查每个selector的唯一性（selector uniquing）
```

```swift
如果在各个 OC 类别的 ‘load’方法里做了不少事情(如在里面使用 Method swizzle),那么会耗时不少。dyld运行APP的初始化函数，调用每个OC类的+load方法，调用C++的构造器函数（attribute((constructor))修饰），创建非基本类型的C++静态全局变量，然后执行main函数。
```


优化思路是
```swift
1. 移除不需要用到的动态库
2. 移除不需要用到的类
3. 合并功能类似的类和扩展
4. 尽量避免在+load方法里执行的操作，可以推迟到+initialize方法中。
```
____
## t2 时间的优化分析
`t2`使用了来自[NewPan](https://www.jianshu.com/u/e2f2d779c022)大大 的打点计时器[BLStopwatch](https://github.com/beiliao-mobile/BLStopwatch)

![检测耗时](https://user-gold-cdn.xitu.io/2018/2/28/161daa6d6d0359f6?w=640&h=1136&f=jpeg&s=101076)

可以看到，我的 APP 加载时间并没有很慢，但是也想看一看有没有优化的空间。

在 `didFinishLaunchingWithOptions` 方法里我们一般都有以下的逻辑：
```swift
初始化第三方 SDK
配置 APP 运行需要的环境
自己的一些工具类的初始化
...
```
这里主要参考[[iOS]一次立竿见影的启动时间优化](https://www.jianshu.com/p/c1734cbdf39b) 
从优化图可以看到，我的应用的跳转逻辑是 `打开` -> `广告页` -> `首页`，首页的UI 架构是:
![UITabBarC管理一堆 UINavigationC](https://user-gold-cdn.xitu.io/2018/2/28/161daa6d6d6768cc?w=700&h=320&f=png&s=67159)


但是如果 UI 架构如上，并且在`didFinishLaunchingWithOptions `里面设置了根视图
```swift
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
NSLog(@"didFinishLaunchingWithOptions 开始执行");

self.window = [[UIWindow alloc] initWithFrame:[UIScreen mainScreen].bounds];
TestTabBarController *tabBarVc = [TestTabBarController new];
self.window.rootViewController = tabBarVc;
[self.window makeKeyAndVisible];

NSLog(@"didFinishLaunchingWithOptions 跑完了");

return YES;
}

```
然后我们来到 `TestTabBarController` 里的 `viewDidLoad `方法里进行它的 `viewControllers` 的设置，然后再进入到每个 `viewController` 的 `viewDidLoad` 方法里进行更多的初始化操作。那么你觉得从 `didFinishLaunchingWithOptions` 到最后显示展示的 `viewController` 的  `viewDidLoad` 这些方法的执行顺序是怎么样的呢？
```swift
didFinishLaunchingWithOptions 开始执行 
开始加载 TestTabBarController 的 viewDidLoad
didFinishLaunchingWithOptions 跑完了
开始加载 TestViewController 的 viewDidLoad, 然后执行一堆初始化的操作

```

在`TestTabBarController` 中操作了 `TestViewController` 的 `view` 的话，那么调用顺序将会是这样：
```swift
didFinishLaunchingWithOptions 开始执行 
开始加载 TestTabBarController 的 viewDidLoad
开始加载 TestViewController 的 viewDidLoad, 然后执行一堆初始化的操作
didFinishLaunchingWithOptions 跑完了
```
这样的问题就是当我们把界面的初始化、网络请求、数据解析、视图渲染等操作放在了`viewDidLoad` 方法里，这样一来每次启动 APP 的时候，在用户看到第一个页面之前，我们要把这些事件全部都处理完，才会进入到视图渲染阶段。

一般来说，我们放到`didFinishLaunchingWithOptions `执行的代码，有很多**初始化操作，如日志，统计，SDK配置**等。尽量做到只放必需的，其他的可以延迟到`MainViewController`展示完成`viewDidAppear`以后。
```
* 日志、统计等必须在 APP 一启动就最先配置的事件
* 项目配置、环境配置、用户信息的初始化 、推送、IM等事件
* 其他 SDK 和配置事件
```

* 第一类，必须第一时间启动，仍然把它留在 `didFinishLaunchingWithOptions` 里启动。
* 第二类，这些功能在用户进入 APP 主体的之前是必须要加载完的，我把他放到广告页面的`viewDidAppear`启动。
* 第三类，由于启动时间不是必须的，所以我们可以放在第一个界面的 `viewDidAppear` 方法里，这里完全不会影响到启动时间。

![优化后](https://user-gold-cdn.xitu.io/2018/2/28/161daa6d6defd0ba?w=640&h=1136&f=jpeg&s=110438)

这是优化后的启动时间

### 优化思路
```
梳理各个三方库，找到可以延迟加载的库，做延迟加载处理，比如放到首页控制器的viewDidAppear方法里。

梳理业务逻辑，把可以延迟执行的逻辑，做延迟执行处理。比如检查新版本、注册推送通知等逻辑。

避免复杂/多余的计算。

避免在首页控制器的viewDidLoad和viewWillAppear做太多事情，这2个方法执行完，首页控制器才能显示，部分可以延迟创建的视图应做延迟创建/懒加载处理。

采用性能更好的API。

首页控制器用纯代码方式来构建。
```

**另**：[[iOS]一次立竿见影的启动时间优化](https://www.jianshu.com/p/c1734cbdf39b) 提到了使用一个工具类来管理的方法，可以比较方便的管理优化。

#### 总结
性价比最高的优化阶段就是`t2`的一些逻辑整理，尽量将不需要的耗时操作延迟到首屏展示之后执行。
同时一般来说，优化应该在项目完成稳定之后进行，避免过早优化.

参考：
1. [App Startup Time: Past, Present, and Future](https://link.jianshu.com/?t=https%3A%2F%2Fdeveloper.apple.com%2Fvideos%2Fplay%2Fwwdc2017%2F413%2F)
2. [[iOS]一次立竿见影的启动时间优化](https://www.jianshu.com/p/c1734cbdf39b)
3. [iOS App 启动性能优化](https://mp.weixin.qq.com/s?__biz=MzA3NTYzODYzMg==&mid=2653579242&idx=1&sn=8f2313711f96a62e7a80d63851653639&chksm=84b3b5edb3c43cfb08e30f2affb1895e8bac8c5c433a50e74e18cda794dcc5bd53287ba69025&mpshare=1&scene=1&srcid=081075Vt9sYPaGgAWwb7xd1x&key=4b95006583a3cb388791057645bf19a825b73affa9d3c1303dbc0040c75548ef548be21acce6a577731a08112119a29dfa75505399bba67497ad729187c6a98469674924c7b447788c7370f6c2003fb4&ascene=0&uin=NDA2NTgwNjc1&devicetype=iMac16%2C2+OSX+OSX+10.12.6+build(16G29))
4. [APP启动优化的一次实践](https://icetime17.github.io/2018/01/01/2018-01/APP%E5%90%AF%E5%8A%A8%E4%BC%98%E5%8C%96%E7%9A%84%E4%B8%80%E6%AC%A1%E5%AE%9E%E8%B7%B5/)
5. [阿里数据iOS端启动速度优化的一些经验](http://www.cocoachina.com/ios/20180202/22120.html)
