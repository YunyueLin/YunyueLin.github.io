---
title: Swift 类的构造方法
date: 2017-09-13 12:47:47
tags: [iOS,Swift]
categories: [iOS]
---

**这篇文章讲解了在 Swift 中的类的构造规则。**

Swift 类或结构体如果所有的属性都有默认值，同时没有自定义构造器，那么 Swift 会给这些结构体或类提供一个默认构造器，它后简单的创建一个所有属性值都设置为默认值的实例。
```swift
class Test {
var title = "Test"
}

Test()
```

类的继承和构造过程:
类里面的所有存储型属性(包括所有继承自父类的属性），都必须在构造器中设置默认值。类类型有两种构造器来保证所有存储型属性都获得默认值，他们是指定构造器和便利构造器。

Swift 采用以下三条规则来限制构造器之间的代理调用:
```
规则 1 指定构造器必须调用其直接父类的的指定构造器。
规则 2 便利构造器必须调用同类中定义的其它构造器。
规则 3 便利构造器必须最终导致一个指定构造器被调用。

一个更方便记忆的方法是:
• 指定构造器必须总是向上代理 
• 便利构造器必须总是横向代理
```
如图所示：![20160126152938443.png](https://user-gold-cdn.xitu.io/2018/2/27/161d7704c50d5f5d?w=896&h=429&f=png&s=9396)

两段式构造过程：
Swift 中类的构造过程包含两个阶段。第一个阶段，每个存储型属性被引入它们的类指定一个初始值。当每个存 储型属性的初始值被确定后，第二阶段开始，它给每个类一次机会，在新实例准备使用之前进一步定制它们的存储型属性。

Swift 编译器执行 4 种有效的安全检查。
```
安全检查 1 指定构造器必须保证它所在类引入的所有属性都必须先初始化完成，之后才能将其它构造任务向上代理给父类中 的构造器。

安全检查 2 指定构造器必须先向上代理调用父类构造器，然后再为继承的属性设置新值。如果没这么做，指定构造器赋予的 新值将被父类中的构造器所覆盖。

安全检查 3 便利构造器必须先代理调用同一类中的其它构造器，然后再为任意属性赋新值。如果没这么做，便利构造器赋予 的新值将被同一类中其它指定构造器所覆盖。

安全检查 4 构造器在第一阶段构造完成之前，不能调用任何实例方法，不能读取任何实例属性的值，不能引用 self 作为一个 值。

//对于 1-3
class SuperClass {
var title : String
init() {
title = "title"
}
}

class SomeClass: SuperClass {
var content : String
//重写父类的指定构造器
override init() {
//先初始化自己的属性
content = "testContent"
super.init()
//再初始化继承来的属性
title = "subClass title"
}

//便利
convenience init(content:String){
self.init()
//先调用指定构造器，再修改属性
self.content = content
}


}

//对于4 刚好回答了很久之前一个点击按钮不走方法的坑
var testBtn = { () -> UIButton in

let testBtn = UIButton()

//这样不行，因为在执行 Block 的时候，self 还没有初始化完成， 如果需要在这里写，可以用懒加载的方式
testBtn.addTarget(self, action: #selector(someMethod), for: .touchUpInside)

return testBtn
}()


```
以下是构造流程展示
```swift
阶段 1
• 某个指定构造器或便利构造器被调用。
• 完成新实例内存的分配，但此时内存还没有被初始化。
• 指定构造器确保其所在类引入的所有存储型属性都已赋初值。存储型属性所属的内存完成初始化。
• 指定构造器将调用父类的构造器，完成父类属性的初始化。
• 这个调用父类构造器的过程沿着构造器链一直往上执行，直到到达构造器链的最顶部。
• 当到达了构造器链最顶部，且已确保所有实例包含的存储型属性都已经赋值，这个实例的内存被认为已经完 全初始化。此时阶段 1 完成。
阶段 2
• 从顶部构造器链一直往下，每个构造器链中类的指定构造器都有机会进一步定制实例。构造器此时可以访问 self 、修改它的属性并调用实例方法等等。
• 最终，任意构造器链中的便利构造器可以有机会定制实例和使用 self 。
```

构造器的继承和重写:

跟 Objective-C 中的子类不同，Swift 中的子类默认情况下不会继承父类的构造器。Swift 的这种机制可以防止 一个父类的简单构造器被一个更精细的子类继承，并被错误地用来创建子类的实例。
```
父类的构造器仅会在 你为子类中引入的所有新属性都提供了默认值 情况下被继承。

规则 1 
如果子类没有定义任何指定构造器，它将自动继承所有父类的指定构造器。

规则 2
如果子类提供了所有父类指定构造器的实现——无论是通过规则 1 继承过来的，还是提供了自定义实现——它将 自动继承所有父类的便利构造器。
即使你在子类中添加了更多的便利构造器，这两条规则仍然适用。

```

这里有个好玩的事情，子类可以把父类的指定构造器重写为便利构造器



```swift
在类的构造器前添加 required 修饰符表明所有该类的子类都必须实现该构造器:

class Car {
var name: String
init(name: String) {
self.name = name
}
convenience init() {
self.init(name: "[Unnamed]")
}
}

class Audi: Car {
var displacement: Int
init(name: String, displacement: Int) {
self. displacement = displacement
super.init(name: name)
}

override convenience init(name: String) {
self.init(name: name, displacement: 1)
}

}

其中 Audi 的便利构造器 ‘override convenience init(name: String)’ 使用了和 Car 中的指定构造器‘init(name: String)’相同的参数，所以这个便利构造器实际是重写了父类的指定构造器。
而且尽管 Audi 将父类的指定构造器重写为便利构造器，它依然提供了父类的所有指定构造器的实现，所以 Audi 还有自动继承父类的所有便利构造器

```
必要构造器:

```swift
class SomeClass {
required init() {
// 构造器的实现代码

} }


在子类重写父类的必要构造器时，必须在子类的构造器前也添加 required 修饰符，表明该构造器要求也应用于继 承链后面的子类。在重写父类中必要的指定构造器时，不需要添加 override 修饰符:
class SomeSubclass: SomeClass {
required init() {
// 构造器的实现代码 }
}

```

PS：开始使用 Swift都一年多时间了，觉得开始好好整理整理博客了，毕竟写出来比只是自己学有更好的学习效果，还能当个笔记。
先从简单的基础开始 ，然后再慢慢写别的东西。欢迎大家交流哦~

