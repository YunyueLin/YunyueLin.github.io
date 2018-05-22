---
title: Swift 派发机制
date: 2018-05-22 16:04:36
tags: [iOS,Swift]
categories: [iOS]
---

> 此篇博客用来自我学习，来源[戴铭大佬的这篇博客](https://ming1016.github.io/2018/01/24/why-swift/#more)
## Swift 派发机制

派发目的是让 CPU 知道被调用的函数在哪里。`Swift` 语言是支持编译型语言的直接派发，函数表派发和消息机制派发三种派发方式的，下面分别对这三种派发方式说明下。

### 直接派发

C++ 默认使用的是直接派发，加上 `virtual `修饰符可以改成函数表派发。直接派发是最快的，原因是调用指令会少，还可以通过编译器进行比如内联等方式的优化。缺点是由于缺少动态性而不支持继承。

```swift
struct DragonFirePosition {
var x:Int64
var y:Int32
func land() {}
}
func DragonWillFire(_ position:DragonFirePosition) {
position.land()
}
let position = DragonFirePosition(x: 342, y: 213)
DragonWillFire(position)
```

编译 `inline` 后 `DragonWillFire(DragonFirePosition(x: 342, y: 213)) `会直接跳到方法实现的地方，结果就变成 `position.land()`。

### 函数表派发

Java 默认就是使用的函数表派发，通过 `final` 修饰符改成直接派发。函数表派发是有动态性的，在 Swift 里函数表叫 `witness table`，大部分语言叫 `virtual table`。一个类里会用数组来存储里面的函数指针，`override` 父类的函数会替代以前的函数，子类添加的函数会被加到这个数组里。举个例子：

```swift
class Fish {
func swim() {}
func eat() {
//normal eat
}
}
class FlyingFish: Fish {
override func eat() {
//flying fish eat
}
func fly() {}
}
```

编译器会给 `Fish` 类和 `FlyingFish` 类分别创建 `witness table`。在 `Fish` 的函数表里有 `swim` 和 `eat` 函数，在 `FlyingFish` 函数表里有父类 `Fish` 的 `swim`，覆盖了父类的 `eat` 和新增加的函数 `fly`。

一个函数被调用时会先去读取对象的函数表，再根据类的地址加上该的函数的偏移量得到函数地址，然后跳到那个地址上去。从编译后的字节码这方面来看就是两次读取一次跳转，比直接派发还是慢了些。

### 消息机制派发

这种机制是在运行时可以改变函数的行为，`KVO` 和 `CoreData` 都是这种机制的运用。`OC` 默认就是使用的消息机制派发，使用 `C` 来直接派发获取高性能。`Swift` 可以通过 `dynamic` 修饰来支持消息机制派发。

当一个消息被派发，运行时就会按照继承关系向上查找被调用的函数。但是这样效率不高，所以需要通过缓存来提高效率，这样查找性能就能和函数派发差不多了。

### 具体派发

#### 声明

值类型都会采用直接派发。无论是 `class` 还是`协议` 的 `extension` 也都是直接派发。`class` 和`协议`是函数表派发。

#### 指定派发方式

*   final：让类里的函数使用直接派发，这样该函数将会没有动态性，运行时也没法取到这个函数。
*   dynamic：可以让类里的函数使用消息机制派发，可以让 extension 里的函数被 override。

#### 派发优化

Swift 会在这上面做优化，比如一个函数没有 `override`，Swift 就可能会使用直接派发的方式，所以如果属性绑定了 `KVO` 它的 `getter `和 setter 方法可能会被优化成直接派发而导致 `KVO` 的失效，所以记得加上 `dynamic` 的修饰来保证有效。后面 `Swift` 应该会在这个优化上去做更多的处理。

