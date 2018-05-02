---
title: iOS Runtime的理解
date: 2018-04-13 10:25:35
tags: [iOS,Objective-C]
categories: [iOS]
---

```
字数5000+,预计阅读时间 30分钟
```

主要参考自:

1. [iOS运行时(Runtime)详解+Demo](https://www.jianshu.com/p/adf0d566c887)
2. [Objective-C Runtime](http://yulingtianxia.com/blog/2014/11/05/objective-c-runtime/ "Objective-C Runtime")
3. [神经病院Objective-C Runtime出院第三天——如何正确使用Runtime](https://www.jianshu.com/p/db6dc23834e3)


## 运行时简介
`Objective-C`语言是一门动态语言，它将很多静态语言在编译和链接时期做的事放到了运行时来处理。
对于`Objective-C`来说，这个运行时系统就像一个操作系统一样：它让所有的工作可以正常的运行。`Runtime`基本上是用`C`和`汇编`写的，这个库使得`C语言`有了面向对象的能力。
在`Runtime`中，对象可以用`C语言`中的结构体表示，而方法可以用`C`函数来实现，另外再加上了一些额外的特性。这些结构体和函数被runtime函数封装后，让`OC`的面向对象编程变为可能。
找出方法的最终执行代码：当程序执行`[object doSomething]`时，会向消息接收者`(object)`发送一条消息`(doSomething)`，`Runtime`会根据消息接收者是否能响应该消息而做出不同的反应。

## 一、类与对象基础数据结构
__1、object_class 类__
`Objective-C`类是由Class类型来表示的，它实际上是一个指
向`objc_class`结构体的指针。
```OC
typedef struct object_class *Class
```
它的定义如下：
```OC
struct object_class{
Class isa OBJC_ISA_AVAILABILITY;
#if !__OBJC2__
    Class super_class                        OBJC2_UNAVAILABLE;       // 父类
    const char *name                         OBJC2_UNAVAILABLE;  // 类名
    long version                             OBJC2_UNAVAILABLE;  // 类的版本信息，默认为0
    long info                                OBJC2_UNAVAILABLE;  // 类信息，供运行期使用的一些位标识
    long instance_size                       OBJC2_UNAVAILABLE;  // 该类的实例变量大小
    struct objc_ivar_list *ivars             OBJC2_UNAVAILABLE;  // 该类的成员变量链表
    struct objc_method_list *methodLists     OBJC2_UNAVAILABLE;  // 方法定义的链表
    struct objc_cache *cache                 OBJC2_UNAVAILABLE;  // 方法缓存
    struct objc_protocol_list *protocols     OBJC2_UNAVAILABLE;  // 协议链表
#endif
}OBJC2_UNAVAILABLE;

```
__2、objc_object 实例__
`objc_object`是表示一个类的实例的结构体
它的定义如下：
```OC
struct objc_object{
    Class isa OBJC_ISA_AVAILABILITY;
};
typedef struct objc_object *id;
```
可以看到，这个结构体只有一个字体，即指向其类的`isa`指针。这
样，当我们向一个`Objective-C`对象发送消息时，运行时库会根据
实例对象的isa指针找到这个实例对象所属的类。`Runtime`库会在类
的方法列表及父类的方法列表中去寻找与消息对应的`selector`指向
的方法，找到后即运行这个方法。

__3、元类(Meta Class)__
`meta-class`是一个类对象的类。
在上面我们提到，所有的类自身也是一个对象，我们可以向这个对象发送消息(即调用类方法)。
既然是对象，那么它也是一个`objc_object`指针，它包含一个指向其类的一个`isa`指针。那么，这个`isa`指针指向什么呢？

答案是，为了调用类方法，这个类的`isa`指针必须指向一个包含这些类方法的一个`objc_class`结构体。这就引出了`meta-class`的概念，`meta-class`中存储着一个类的所有类方法。

所以，调用类方法的这个类对象的`isa`指针指向的就是`meta-class`
当我们向一个对象发送消息时，`runtime`会在这个对象所属的这个类的方法列表中查找方法；而向一个类发送消息时，会在这个类的`meta-class`的方法列表中查找。

再深入一下，`meta-class`也是一个类，也可以向它发送一个消息，那么它的`isa`又是指向什么呢？为了不让这种结构无限延伸下去，`Objective-C`的设计者让所有的`meta-class`的`isa`指向基类的`meta-class`，以此作为它们的所属类。

即，任何`NSObject`继承体系下的`meta-class`都使用`NSObject`的`meta-class`作为自己的所属类，而基类的`meta-class`的`isa`指针是指向它自己。

通过上面的描述，再加上对`objc_class`结构体中`super_class`指针的分析，我们就可以描绘出类及相应`meta-class`类的一个继承体系了，如下：
![ OC 中的继承体系.png](https://user-gold-cdn.xitu.io/2018/2/27/161d746b4071dbde?w=663&h=679&f=png&s=141254)

__4、Category__
`Category`是表示一个指向分类的结构体的指针，其定义如下：
```OC
typedef struct objc_category *Category
struct objc_category{
    char *category_name                         OBJC2_UNAVAILABLE; // 分类名
    char *class_name                            OBJC2_UNAVAILABLE;  // 分类所属的类名
    struct objc_method_list *instance_methods   OBJC2_UNAVAILABLE;  // 实例方法列表
    struct objc_method_list *class_methods      OBJC2_UNAVAILABLE; // 类方法列表
    struct objc_protocol_list *protocols        OBJC2_UNAVAILABLE; // 分类所实现的协议列表
}

```
这个结构体主要包含了分类定义的实例方法与类方法，其中`instance_methods`列表是`objc_class`中方法列表的一个子集，而`class_methods`列表是元类方法列表的一个子集。
可发现，类别中没有`ivar`成员变量指针，也就意味着：类别中不能够添加实例变量和属性,__(!除非使用`关联对象`,而且`Category`中的属性，只会生成setter和getter方法，不会生成成员变量)例子如下__
```OC
#import "UIButton+ClickBlock.h"
#import static const void *associatedKey = "associatedKey";
@implementation UIButton (ClickBlock)
//Category中的属性，只会生成setter和getter方法，不会生成成员变量
-(void)setClick:(clickBlock)click{
    objc_setAssociatedObject(self, associatedKey, click, OBJC_ASSOCIATION_COPY_NONATOMIC);
    [self removeTarget:self action:@selector(buttonClick) forControlEvents:UIControlEventTouchUpInside];
    if (click) {
        [self addTarget:self action:@selector(buttonClick) forControlEvents:UIControlEventTouchUpInside];
    }
}
-(clickBlock)click{
    return objc_getAssociatedObject(self, associatedKey);
}
-(void)buttonClick{
    if (self.click) {
        self.click();
    }
}
@end
```
然后在代码中，就可以使用 `UIButton` 的属性来监听单击事件了：
```OC
UIButton *button = [UIButton buttonWithType:UIButtonTypeCustom];
button.frame = self.view.bounds;
[self.view addSubview:button];
button.click = ^{
NSLog(@"buttonClicked");
};
```
__关联对象的使用__
```OC
1.设置关联值
参数说明：
object：与谁关联，通常是传self
key：唯一键，在获取值时通过该键获取，通常是使用static
const void *来声明
value：关联所设置的值
policy：内存管理策略，比如使用copy

void objc_setAssociatedObject(id object, const void *key, id value, objc _AssociationPolicy policy)
```

```OC
2.获取关联值
参数说明：
object：与谁关联，通常是传self，在设置关联时所指定的与哪个对象关联的那个对象
key：唯一键，在设置关联时所指定的键
id objc_getAssociatedObject(id object, const void *key)
```

```OC
3.取消关联
void objc_removeAssociatedObjects(id object)
```

```OC
关联策略
使用场景：
可以在类别中添加属性

typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy){
    OBJC_ASSOCIATION_ASSIGN = 0,             // 表示弱引用关联，通常是基本数据类型
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1,   // 表示强引用关联对象，是线程安全的
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,     // 表示关联对象copy，是线程安全的
    OBJC_ASSOCIATION_RETAIN = 01401,         // 表示强引用关联对象，不是线程安全的
    OBJC_ASSOCIATION_COPY = 01403            // 表示关联对象copy，不是线程安全的
};

```

## 二、方法与消息
__1、SEL__

SEL又叫选择器，是表示一个方法的`selector`的指针，其定义如下：
```OC
typedef struct objc_selector *SEL；
```
方法的`selector`用于表示运行时方法的名字。`Objective-C`在编译时，会依据每一个方法的名字、参数序列，生成一个唯一的整型标识`(Int类型的地址)`，这个标识就是`SEL`。
两个类之间，只要方法名相同，那么方法的`SEL`就是一样的，每一个方法都对应着一个`SEL`。所以在`Objective-C`同一个类(及类的继承体系)中，不能存在2个同名的方法，即使参数类型不同也不行
如在某一个类中定义以下两个方法: 错误

```OC
- (void)setWidth:(int)width;
- (void)setWidth:(double)width;
```
当然，不同的类可以拥有相同的`selector`，这个没有问题。不同类的实例对象执行相同的`selector`时，会在各自的方法列表中去根据`selector`去寻找自己对应的`IMP`。
工程中的所有的`SEL`组成一个`Set`集合，如果我们想到这个方法集合中查找某个方法时，只需要去找到这个方法对应的`SEL`就行了，`SEL`实际上就是根据方法名`hash`化了的一个字符串，而对于字符串的比较仅仅需要比较他们的地址就可以了，可以说速度上无语伦比！
本质上，`SEL`只是一个指向方法的指针（准确的说，只是一个根据方法名`hash`化了的`KEY`值，能唯一代表一个方法），它的存在只是为了加快方法的查询速度。
```OC
通过下面三种方法可以获取SEL:
a、sel_registerName函数
b、Objective-C编译器提供的@selector()
c、NSSelectorFromString()方法
```
__2、IMP__

`IMP`实际上是一个函数指针，指向方法实现的地址。
其定义如下:
```OC
id (*IMP)(id, SEL,...)
```
第一个参数：是指向`self`的指针(如果是实例方法，则是类实例的内存地址；如果是类方法，则是指向元类的指针)
第二个参数：是方法选择器`(selector)`
接下来的参数：方法的参数列表。

前面介绍过的`SEL`就是为了查找方法的最终实现`IMP`的。由于每个方法对应唯一的`SEL`，因此我们可以通过`SEL`方便快速准确地获得它所对应的`IMP`，查找过程将在下面讨论。取得`IMP`后，我们就获得了执行这个方法代码的入口点，此时，我们就可以像调用普通的`C语言`函数一样来使用这个函数指针了。

__3、Method__

Method用于表示类定义中的方法，则定义如下：

```OC
typedef struct objc_method *Method
struct objc_method{
    SEL method_name      OBJC2_UNAVAILABLE; // 方法名
    char *method_types   OBJC2_UNAVAILABLE;
    IMP method_imp       OBJC2_UNAVAILABLE; // 方法实现
}
```
我们可以看到该结构体中包含一个`SEL`和`IMP`，实际上相当于在`SEL`和`IMP`之间作了一个映射。有了`SEL`，我们便可以找到对应的`IMP`，从而调用方法的实现代码。

__4、消息__

Objc 中发送消息是用中括号（`[]`）把接收者和消息括起来，而直到运行时才会把消息与方法实现绑定。

**有关消息发送和消息转发机制的原理，可以查看[这篇文章](http://yulingtianxia.com/blog/2016/06/15/Objective-C-Message-Sending-and-Forwarding/)。**

____
面对着 `Cocoa` 中大量 `API`，只知道简单的查文档和调用。还记得初学 `Objective-C` 时把 `[receiver message]` 当成简单的方法调用，而无视了“发送消息”这句话的深刻含义。其实 `[receiver message]` 会被编译器转化为：
```OC
objc_msgSend(receiver, selector)
```
如果消息含有参数，则为：
```OC
objc_msgSend(receiver, selector, arg1, arg2, ...)
```
如果消息的接收者能够找到对应的 `selector`，那么就相当于直接执行了接收者这个对象的特定方法；否则，消息要么被转发，或是临时向接收者动态添加这个 `selector` 对应的实现内容，要么就干脆玩完崩溃掉。

现在可以看出` [receiver message] `真的不是一个简简单单的方法调用。因为这只是在编译阶段确定了要向接收者发送 `message `这条消息，而 `receiver` 将要如何响应这条消息，那就要看运行时发生的情况来决定了。

这里看起来像是`objc_msgSend`返回了数据，其实`objc_msgSend`从不返回数据而是你的方法被调用后返回了数据。下面详细叙述下消息发送步骤：
```OC
1、检测这个 selector 是不是要忽略的。比如 Mac OS X 开发，有了垃圾回收就不理会 retain, release 这些函数了。
2、检测这个 target 是不是 nil 对象。ObjC 的特性是允许对一个 nil 对象执行任何一个方法不会 Crash，因为会被忽略掉。
3、如果上面两个都过了，那就开始查找这个类的 IMP，先从 cache 里面找，完了找得到就跳到对应的函数去执行。
4、如果 cache 找不到就找一下方法分发表。
5、如果分发表找不到就到超类的分发表去找，一直找，直到找到NSObject类为止。
6、如果还找不到就要开始进入动态方法解析了，后面会提到。
PS:这里说的分发表其实就是Class中的方法列表，它将方法选择器和方法实现地址联系起来。
```
![方法调用流程.gif](https://user-gold-cdn.xitu.io/2018/2/27/161d746b40b47f61?w=332&h=544&f=gif&s=11991)

其实编译器会根据情况在`objc_msgSend`, `objc_msgSend_stret`, `objc_msgSendSuper`, 或 `objc_msgSendSuper_stret`四个方法中选择一个来调用。如果消息是传递给超类，那么会调用名字带有`”Super”`的函数；如果消息返回值是数据结构而不是简单值时，那么会调用名字带有`”stret”`的函数。排列组合正好四个方法。

值得一提的是在` i386` 平台处理返回类型为浮点数的消息时，需要用到`objc_msgSend_fpret`函数来进行处理，这是因为返回类型为浮点数的函数对应的` ABI(Application Binary Interface) `与返回整型的函数的` ABI `不兼容。此时`objc_msgSend`不再适用，于是`objc_msgSend_fpret`被派上用场，它会对浮点数寄存器做特殊处理。不过在` PPC` 或 `PPC64` 平台是不需要麻烦它的。

PS：有木有发现这些函数的命名规律哦？带`“Super”`的是消息传递给超类；`“stret”`可分为`“st”+“ret”`两部分，分别代表`“struct”`和`“return”`；`“fpret”`就是`“fp”+“ret”`，分别代表`“floating-point”`和`“return”`。

__5、动态方法解析__

你可以动态地提供一个方法的实现。例如我们可以用`@dynamic`关键字在类的实现文件中修饰一个属性：
```OC
@dynamic propertyName;
```
这表明我们会为这个属性动态提供存取方法，也就是说编译器不会再默认为我们生成`setPropertyName:`和`propertyName`方法，而需要我们动态提供。我们可以通过分别重载`resolveInstanceMethod:`和`resolveClassMethod:`方法分别添加实例方法实现和类方法实现。因为当` Runtime `系统在`Cache`和方法分发表中（包括超类）找不到要执行的方法时，`Runtime`会调用`resolveInstanceMethod:`或`resolveClassMethod:`来给程序员一次动态添加方法实现的机会。我们需要用`class_addMethod`函数完成向特定类添加特定方法实现的操作：
```OC
void dynamicMethodIMP(id self, SEL _cmd) {
// implementation ....
}
@implementation MyClass
+ (BOOL)resolveInstanceMethod:(SEL)aSEL{
    if (aSEL == @selector(resolveThisMethodDynamically)) {
        class_addMethod([self class], aSEL, (IMP) dynamicMethodIMP, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:aSEL];
}
@end
```
上面的例子为`resolveThisMethodDynamically`方法添加了实现内容，也就是`dynamicMethodIMP`方法中的代码。其中 “`v@:`” 表示返回值和参数，这个符号涉及 [Type Encoding](https://developer.apple.com/library/mac/DOCUMENTATION/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html)

PS：动态方法解析会在消息转发机制浸入前执行。如果 `respondsToSelector:` 或 `instancesRespondToSelector:`方法被执行，动态方法解析器将会被首先给予一个提供该方法选择器对应的`IMP`的机会。如果你想让该方法选择器被传送到转发机制，那么就让`resolveInstanceMethod:`返回`NO`。

**备注:** 解析类方法等具体做法 例：
.h
```OC
#import <Foundation/Foundation.h>

@interface Student : NSObject
+ (void)learnClass:(NSString *) string;
- (void)goToSchool:(NSString *) name;
@end
```
.m
```OC
#import "Student.h"
#import <objc/runtime.h>

@implementation Student
+ (BOOL)resolveClassMethod:(SEL)sel {
    if (sel == @selector(learnClass:)) {
        class_addMethod(object_getClass(self), sel, class_getMethodImplementation(object_getClass(self), @selector(myClassMethod:)), "v@:");
        return YES;
    }
    return [class_getSuperclass(self) resolveClassMethod:sel];
}

+ (BOOL)resolveInstanceMethod:(SEL)aSEL
{
    if (aSEL == @selector(goToSchool:)) {
        class_addMethod([self class], aSEL, class_getMethodImplementation([self class], @selector(myInstanceMethod:)), "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:aSEL];
}

+ (void)myClassMethod:(NSString *)string {
    NSLog(@"myClassMethod = %@", string);
}

- (void)myInstanceMethod:(NSString *)string {
    NSLog(@"myInstanceMethod = %@", string);
}
@end
```
需要深刻理解` [self class]` 与 `object_getClass(self)` 甚至 `object_getClass([self class])` 的关系，其实并不难，重点在于 `self` 的类型：

```OC
1、当 self 为实例对象时，[self class] 与 object_getClass(self) 等价，因为前者会调用后者。
object_getClass([self class]) 得到元类。

2、当 self 为类对象时，[self class] 返回值为自身，还是 self。
object_getClass(self) 与 object_getClass([self class]) 等价。
```

__6、消息转发__

![消息转发](https://user-gold-cdn.xitu.io/2018/2/27/161d746b4067b792?w=800&h=618&f=png&s=21960)

__重定向__

在消息转发机制执行前，`Runtime` 系统会再给我们一次偷梁换柱的机会，即通过重载`- (id)forwardingTargetForSelector:(SEL)aSelector`方法替换消息的接受者为其他对象：
```OC
- (id)forwardingTargetForSelector:(SEL)aSelector{
    if(aSelector == @selector(mysteriousMethod:)){
        return alternateObject;
    }
    return [super forwardingTargetForSelector:aSelector];
}
```
毕竟消息转发要耗费更多时间，抓住这次机会将消息重定向给别人是个不错的选择，如果此方法返回`nil`或`self`,则会进入消息转发机制`(forwardInvocation:);`否则将向返回的对象重新发送消息。

如果想替换类方法的接受者，需要覆写 `+ (id)forwardingTargetForSelector:(SEL)aSelector `方法，并返回类对象：
```OC
+ (id)forwardingTargetForSelector:(SEL)aSelector {
    if(aSelector == @selector(xxx)) {
        return NSClassFromString(@"Class name");
    }
    return [super forwardingTargetForSelector:aSelector];
}
```
__转发__

当动态方法解析不作处理返回`NO`时，消息转发机制会被触发。在这时`forwardInvocation:`方法会被执行，我们可以重写这个方法来定义我们的转发逻辑：
```OC
- (void)forwardInvocation:(NSInvocation *)anInvocation
{
    if ([someOtherObject respondsToSelector:[anInvocation selector]]){
        [anInvocation invokeWithTarget:someOtherObject];
    }else{
    [super forwardInvocation:anInvocation];
    }
}
```
该消息的唯一参数是个`NSInvocation`类型的对象——该对象封装了原始的消息和消息的参数。我们可以实现`forwardInvocation:`方法来对不能处理的消息做一些默认的处理，也可以将消息转发给其他对象来处理，而不抛出错误。

这里需要注意的是参数`anInvocation`是从哪的来的呢？其实在`forwardInvocation:`消息发送前，`Runtime`系统会向对象发送`methodSignatureForSelector:`消息，并取到返回的方法签名用于生成`NSInvocation`对象。所以我们在重写`forwardInvocation:`的同时也要重写`methodSignatureForSelector:`方法，否则会抛异常。

当一个对象由于没有相应的方法实现而无法响应某消息时，运行时系统将通过`forwardInvocation:`消息通知该对象。每个对象都从`NSObject`类中继承了`forwardInvocation:`方法。然而，`NSObject`中的方法实现只是简单地调用了`doesNotRecognizeSelector:`。通过实现我们自己的`forwardInvocation:`方法，我们可以在该方法实现中将消息转发给其它对象。

`forwardInvocation:`方法就像一个不能识别的消息的分发中心，将这些消息转发给不同接收对象。或者它也可以象一个运输站将所有的消息都发送给同一个接收对象。它可以将一个消息翻译成另外一个消息，或者简单的”吃掉“某些消息，因此没有响应也没有错误。`forwardInvocation:`方法也可以对不同的消息提供同样的响应，这一切都取决于方法的具体实现。该方法所提供是将不同的对象链接到消息链的能力。

注意： `forwardInvocation:`方法只有在消息接收对象中无法正常响应消息时才会被调用。 所以，如果我们希望一个对象将`negotiate`消息转发给其它对象，则这个对象不能有`negotiate`方法。否则，`forwardInvocation:`将不可能会被调用。

__转发与多继承__

转发和继承相似，可以用于为Objc编程添加一些多继承的效果。就像下图那样，一个对象把消息转发出去，就好似它把另一个对象中的方法借过来或是“继承”过来一样。

![forwarding.gif](https://user-gold-cdn.xitu.io/2018/2/27/161d746b407e15c4?w=376&h=241&f=gif&s=6597)


这使得不同继承体系分支下的两个类可以“继承”对方的方法，在上图中`Warrior`和`Diplomat`没有继承关系，但是`Warrior`将`negotiate`消息转发给了`Diplomat`后，就好似`Diplomat`是`Warrior`的超类一样。

消息转发弥补了 Objc 不支持多继承的性质，也避免了因为多继承导致单个类变得臃肿复杂。它将问题分解得很细，只针对想要借鉴的方法才转发，而且转发机制是透明的。

__转发与继承__

尽管转发很像继承，但是`NSObject`类不会将两者混淆。像`respondsToSelector: `和 `isKindOfClass:`这类方法只会考虑继承体系，不会考虑转发链。比如上图中一个`Warrior`对象如果被问到是否能响应`negotiate`消息：
```OC
if ( [aWarrior respondsToSelector:@selector(negotiate)] )
...
```
结果是`NO`，尽管它能够接受`negotiate`消息而不报错，因为它靠转发消息给`Diplomat`类来响应消息。

如果你为了某些意图偏要“弄虚作假”让别人以为`Warrior`继承到了`Diplomat`的`negotiate`方法，你得重新实现 `respondsToSelector: `和 `isKindOfClass:`来加入你的转发算法：
```OC
- (BOOL)respondsToSelector:(SEL)aSelector
{
    if ( [super respondsToSelector:aSelector] ){
        return YES;
    }else {
/* Here, test whether the aSelector message can     *
* be forwarded to another object and whether that  *
* object can respond to it. Return YES if it can.  */
    }
    return NO;
}
```
除了`respondsToSelector:` 和` isKindOfClass:`之外，`instancesRespondToSelector:`中也应该写一份转发算法。如果使用了协议，`conformsToProtocol:`同样也要加入到这一行列中。类似地，如果一个对象转发它接受的任何远程消息，它得给出一个`methodSignatureForSelector:`来返回准确的方法描述，这个方法会最终响应被转发的消息。比如一个对象能给它的替代者对象转发消息，它需要像下面这样实现`methodSignatureForSelector:：`
```OC
- (NSMethodSignature*)methodSignatureForSelector:(SEL)selector{
    NSMethodSignature* signature = [super methodSignatureForSelector:selector];
    if (!signature) {
        signature = [surrogate methodSignatureForSelector:selector];
    }
    return signature;
}
```
____

__7、方法交换 Method Swizzling__

之前所说的消息转发虽然功能强大，但需要我们了解并且能更改对应类的源代码，因为我们需要实现自己的转发逻辑。当我们无法触碰到某个类的源代码，却想更改这个类某个方法的实现时，该怎么办呢？可能继承类并重写方法是一种想法，但是有时无法达到目的。这里介绍的是 `Method Swizzling` ，它通过重新映射方法对应的实现来达到“偷天换日”的目的。跟消息转发相比，`Method Swizzling` 的做法更为隐蔽，甚至有些冒险，也增大了`debug`的难度。

```OC
Swizzling原理
在Objective-C中调用一个方法，其实是向一个对象发送消息，而查找消息的唯一依据是selector的名字。
所以，我们可以利用Objective-C的runtime机制，实现在运行时交换selector对应的方法实现以达到我们的目的。

每个类都有一个方法列表，存放着selector的名字和方法实现的映射关系。
IMP有点类似函数指针，指向具体的Method实现

```

这是参考Mattt大神在NSHipster上的文章自己写的代码。
```OC
#import <objc/runtime.h> 

@implementation UIViewController (Tracking) 

+ (void)load { 
    static dispatch_once_t onceToken; 
    dispatch_once(&onceToken, ^{ 
        Class aClass = [self class]; 
        // When swizzling a class method, use the following:
        // Class aClass = object_getClass((id)self);

        SEL originalSelector = @selector(viewWillAppear:); 
        SEL swizzledSelector = @selector(xxx_viewWillAppear:); 

        Method originalMethod = class_getInstanceMethod(aClass, originalSelector); 
        Method swizzledMethod = class_getInstanceMethod(aClass, swizzledSelector); 

        BOOL didAddMethod = class_addMethod(aClass, originalSelector, method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod)); 

        if (didAddMethod) { 
            class_replaceMethod(aClass, swizzledSelector, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod)); 
        } else { 
            method_exchangeImplementations(originalMethod, swizzledMethod); 
        } 
    }); 
} 

#pragma mark - Method Swizzling 

- (void)xxx_viewWillAppear:(BOOL)animated { 
    [self xxx_viewWillAppear:animated]; 
    NSLog(@"viewWillAppear: %@", self); 
} 

@end
```
上面的代码通过添加一个`Tracking`类别到`UIViewController`类中，将`UIViewController`类的`viewWillAppear:`方法和`Tracking`类别中`xxx_viewWillAppear:`方法的实现相互调换。`Swizzling` 应该在`+load`方法中实现，因为`+load`是在一个类最开始加载时调用。`dispatch_once`是GCD中的一个方法，它保证了代码块只执行一次，并让其为一个原子操作，线程安全是很重要的。

如果类中不存在要替换的方法，那就先用`class_addMethod`和`class_replaceMethod`函数添加和替换两个方法的实现；如果类中已经有了想要替换的方法，那么就调用`method_exchangeImplementations`函数交换了两个方法的 `IMP`，这是苹果提供给我们用于实现` Method Swizzling `的便捷方法。

可能有人注意到了这行:
```OC
// When swizzling a class method, use the following:
// Class aClass = object_getClass((id)self);
// ...
// Method originalMethod = class_getClassMethod(aClass, originalSelector);
// Method swizzledMethod = class_getClassMethod(aClass, swizzledSelector);

```
`object_getClass((id)self)` 与 `[self class] `返回的结果类型都是 `Class`,但前者为元类,后者为其本身,因为此时 `self` 为 `Class` 而不是实例.注意 `[NSObject class]` 与 `[object class]` 的区别：
```OC
+ (Class)class {
    return self;
}

- (Class)class {
    return object_getClass(self);
}
```
__PS:__ 如果类中没有想被替换实现的原方法时，`class_replaceMethod`相当于直接调用`class_addMethod`向类中添加该方法的实现；否则调用`method_setImplementation`方法，`types`参数会被忽略。`method_exchangeImplementations`方法做的事情与如下的原子操作等价：
```OC
IMP imp1 = method_getImplementation(m1);
IMP imp2 = method_getImplementation(m2);
method_setImplementation(m1, imp2);
method_setImplementation(m2, imp1);

```
最后`xxx_viewWillAppear:`方法的定义看似是递归调用引发死循环，其实不会的。因为`[self xxx_viewWillAppear:animated]`消息会动态找到`xxx_viewWillAppear:`方法的实现，而它的实现已经被我们与`viewWillAppear:`方法实现进行了互换，所以这段代码不仅不会死循环，如果你把`[self xxx_viewWillAppear:animated]`换成`[self viewWillAppear:animated]`反而会引发死循环。
__PS:__ 看到有人说+load方法本身就是线程安全的，因为它在程序刚开始就被调用，很少会碰到并发问题，于是 stackoverflow 上也有大神去掉了dispatch_once 部分。


扩展阅读:
1、[iOS---防止UIButton重复点击的三种实现方式](https://link.jianshu.com/?t=http://icetime17.github.io/2016/06/29/2016-06/iOS-%E9%98%B2%E6%AD%A2UIButton%E9%87%8D%E5%A4%8D%E7%82%B9%E5%87%BB%E7%9A%84%E4%B8%89%E7%A7%8D%E5%AE%9E%E7%8E%B0%E6%96%B9%E5%BC%8F/)
2、[Swift Runtime分析：还像OC Runtime一样吗？](https://www.jianshu.com/p/adf0d566c887)
