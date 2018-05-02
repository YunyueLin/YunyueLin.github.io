---
title: Swift GCD 的一些高级用法
date: 2017-10-13 12:44:04
tags: [iOS,Swift]
categories: [iOS]
---

## 信号量
之前遇到一个问题，一个请求需要在另一个请求获得的参数。这个时候最开始的办法是把第二个请求写在第一个请求的回调里，但是这样的话，两个请求就很紧密的耦合在一起了。这个时候可以使用`信号量`来使他们分离开来。
先看下相关的3个方法：

`dispatch_semaphore_t dispatch_semaphore_create(long value)：`方法接收一个long类型的参数, 返回一个`dispatch_semaphore_t`类型的信号量，值为传入的参数
`long dispatch_semaphore_wait(dispatch_semaphore_t dsema, dispatch_time_t timeout)：`接收一个信号和时间值，若信号的信号量为0，则会阻塞当前线程，直到信号量大于0或者经过输入的时间值；若信号量大于0，则会使信号量减1并返回，程序继续住下执行
`long dispatch_semaphore_signal(dispatch_semaphore_t dsema)：`使信号量加1并返回

下面看几种使用方法
保持线程同步
```swift
let semaphore = DispatchSemaphore.init(value: 0)
var i = 10
DispatchQueue.global().async {
i = 100

semaphore.signal()
}
semaphore.wait()
print("i = \(i)")
```

输出` i ＝ 100`
如果注掉`semaphore.wait()`这一行，则 `i ＝ 10`
__注释__： 由于是将`block`异步添加到一个并行队列里面，所以程序在主线程跃过`block`直接到`semaphore.wait()`这一行，因为`semaphore`信号量为`0`，所以当前线程会一直阻塞，直到`block`在子线程执行到`semaphore.signal()`，使信号量`+1`，此时`semaphore`信号量为`1`了，所以程序继续往下执行。这就保证了线程间同步了。

为线程加锁(同时可以控制最大并发数量 , value 的值等于几就是最多几个并发)
```swift
let semaphore = DispatchSemaphore.init(value: 1)
for i in 0..<100 {
DispatchQueue.global().async {
semaphore.wait()
print("i = \(i)")
semaphore.signal()
}

}
```
__注释__：当线程1执行到`semaphore.wait()`这一行时，`semaphore`的信号量为`1`，所以使信号量`-1`变为`0`，并且线程`1`继续往下执行；如果当在线程`1`的`print`这一行代码还没执行完的时候，又有线程2来访问，执行`semaphore.wait()`时由于此时信号量为`0`(`.wait()方法默认时间是 OC 的DISPATCH_TIME_FOREVER`)，所以会一直阻塞线程2（此时线程2处于等待状态），直到线程1执行完`print`并执行完`semaphore.signal()`使信号量为1后，线程`2`才能解除阻塞继续住下执行。以上可以保证同时只有一个线程执行`print`这一行代码。

## 栅栏函数(barrier)
等待异步执行多个任务后, 再执行下一个任务，一般使用`barrier函数`
```swift
//创建串行队列
//        let queue = DispatchQueue.init(label: "test", qos: .default, attributes: .init(rawValue: 0), autoreleaseFrequency: .workItem, target: nil)
//创建并行队列
let queue = DispatchQueue.init(label: "test", qos: .default, attributes: .concurrent, autoreleaseFrequency: .workItem, target: nil)

queue.async {//任务一
for _ in 0...3 {
print("......")
}
}
queue.async {//任务二
for _ in 0...3 {
print("++++++");
}
}

queue.async(group: nil, qos: .default, flags: .barrier) {
print("group")
}

queue.async {
print("finish")
}
最后打印
......
++++++
++++++
++++++
++++++
......
......
......
group
finish

```

__注释__:使用`barrier`函数可以做到先让前面的任务执行完毕，再执行之后的任务，会阻塞当前的线程

## 延时任务
```swift
let queue = DispatchQueue.init(label: "test", qos: .default, attributes: .concurrent, autoreleaseFrequency: .workItem, target: nil)
queue.async {//任务一
for _ in 0...3 {
print("......")
}
}
print("0")
queue.asyncAfter(deadline: DispatchTime.now() + 10, execute: {
print("延时提交的任务")
})

queue.async {//任务二
for _ in 0...3 {
print("++++++");
}
}

打印：

```
_注释_：`10s`后提交。并且不会阻碍当前线程

## 组的用法(Group)
__notify(依赖任务)__
```swift
let queue = DispatchQueue.init(label: "test", qos: .default, attributes: .concurrent, autoreleaseFrequency: .workItem, target: nil)

let group = DispatchGroup()
queue.async(group: group, qos: .default, flags: [], execute: {
for _ in 0...10 {

print("耗时任务1")
}
})
queue.async(group: group, qos: .default, flags: [], execute: {
for _ in 0...10 {

print("耗时任务2")
}
})
//执行完上面的两个耗时操作, 回到myQueue队列中执行下一步的任务
group.notify(queue: queue) {
print("回到该队列中执行")
}
queue.async {
print("完成")
}

打印：
耗时任务2
完成
耗时任务1
耗时任务2
耗时任务2
耗时任务2
耗时任务2
耗时任务1
耗时任务2
耗时任务1
耗时任务1
耗时任务1
耗时任务1
回到该队列中执行
```
__注释__：使用`group`+`notify`的话，也会等待之前的任务先执行完，和`barrier`的区别是不会阻碍当前的线程

__wait(等待任务)__
```swift
let queue = DispatchQueue.init(label: "test", qos: .default, attributes: .concurrent, autoreleaseFrequency: .workItem, target: nil)

let group = DispatchGroup()
queue.async(group: group, qos: .default, flags: [], execute: {
for _ in 0...5 {

print("耗时任务1")
}
})
queue.async(group: group, qos: .default, flags: [], execute: {
for _ in 0...5 {

print("耗时任务2")
}
})
//等待上面任务执行，会阻塞当前线程，超时就执行下面的，上面的继续执行。可以无限等待 .distantFuture
let result = group.wait(timeout: .now() + 2.0)
switch result {
case .success:
print("不超时, 上面的两个任务都执行完")
case .timedOut:
print("超时了, 上面的任务还没执行完执行这了")
}

print("接下来的操作")

打印：
耗时任务1
耗时任务2
耗时任务1
耗时任务2
耗时任务1
耗时任务2
耗时任务1
耗时任务2
耗时任务1
耗时任务2
耗时任务1
耗时任务2
不超时, 上面的两个任务都执行完
接下来的操作
```

__注释__:使用`wait`+`group`的话，如果设置`timeout = .distantFuture`的话，那么就和`barrier`函数一样了，会等待之前的完成，否则就是等待之前的完成或者等待设置的时间到了，就会执行接下来的任务了，会阻塞当前现场。

__参考__:
[iOS GCD之dispatch_semaphore学习](https://www.jianshu.com/p/a84c2bf0d77b) 来自[萌小菜](https://www.jianshu.com/u/9ea2ce7958f7)

[Swift 3.0 GCD的常用方法](https://www.jianshu.com/p/be5a277e1f96) 来自[床前明月_光](https://www.jianshu.com/u/a7dee4527c8b)

