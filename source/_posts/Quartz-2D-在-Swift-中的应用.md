---
title: Quartz 2D 在 Swift 中的应用
date: 2017-09-26 12:46:44
tags: [iOS,Swift]
categories: [iOS]
---

`Quartz 2D`是一个二维绘图引擎，同时支持iOS和Mac系统。
其实 iOS 中很多控件都是通过 `Quartz 2D` 画出来的.
同时`Quartz 2D`还可以做这些事情,
```
1、剪裁图形
2、涂鸦/画板（如签名等）
3、手势解锁(连线)
4、折线图、饼状图、柱形图等绘制(虽然我都是直接用 charts)
```

使用 `Quartz 2D`绘图的核心步骤：
```
1、获得上下文
2、绘制/拼接绘图路径
3、将路径添加到上下文
4、渲染上下文
记住：所有的绘图，都是这个步骤，即使使用贝塞尔路径，也只是对这个步骤进行了封装。
对于绘图而言，拿到上下文很关键。
```
其中图形上下文有五种
![WechatIMG2009.jpeg](https://user-gold-cdn.xitu.io/2018/2/27/161d76bcf0650dc8?w=670&h=646&f=jpeg&s=22776)

```
Bitmap Graphics Context
PDF Graphics Context
Window Graphics Context
Layer Graphics Context
Printer Graphics Context
```

使用`Quartz 2D`自定义 UI 控件绘图的方法
```swift
1.新建一个类，继承自UIView
2.必须实现- (void)drawRect:(CGRect)rect方法，然后在这个方法中，可以：
取得跟当前view相关联的图形上下文
绘制相应的图形内容，绘制时产生的线条称为路径。 路径由一个或多个直线段或曲线段组成。
利用图形上下文将绘制的所有内容渲染显示到view上面
也可以：
利用UIKit封装的绘图函数直接绘图
```

关于drawRect:
```swift
为什么要实现drawRect:方法才能绘图到view上？
因为在drawRect:方法中才能取得跟view相关联的图形上下文

drawRect:方法在什么时候被调用？
当view第一次显示到屏幕上时（被加到UIWindow上显示出来）
调用view的setNeedsDisplay或者setNeedsDisplayInRect:时

在drawRect:方法中取得上下文后，就可以绘制东西到view上

View内部有个layer（图层）属性，drawRect:方法中取得的是一个Layer Graphics Context，因此，绘制的东西其实是绘制到view的layer上去了

View之所以能显示东西，完全是因为它内部的layer
```

```swift
//一些常用方法
//获取上下文对象
let context = UIGraphicsGetCurrentContext()

//线条颜色
context?.setStrokeColor(UIColor.red.cgColor)

//线条宽度
context?.setLineWidth(1.0)

//移动画笔到某一点
context?.move(to: CGPoint(x: 10, y: 10))

//画线
context?.addLine(to: CGPoint(x: gameSize , y: 10))

//画弧线 （方法1 ，这个方法如果画一个完整的圆的话，
//有起始点的时候，只会连接起始点和重点，不会出现圆的轨迹!）
context?.addArc(center: CGPoint.init(x: 150, y: 100), radius: 30, startAngle: 0, endAngle: CGFloat(M_PI*2), clockwise: true);
```
![使用这个方法画完整圆.jpeg](https://user-gold-cdn.xitu.io/2018/2/27/161d76bcf0f78f32?w=272&h=274&f=jpeg&s=1671)

```swift
context?.addArc(center: CGPoint.init(x: 150, y: 100), radius: 30, startAngle: 0, endAngle: CGFloat(M_PI), clockwise: true);
```

![如果弧度为180度就没有这个问题.jpeg](https://user-gold-cdn.xitu.io/2018/2/27/161d76bcf0ebb77c?w=276&h=272&f=jpeg&s=1877)




```swift
//画圆弧 （方法2）
context?.addArc(tangent1End: CGPoint(x: 50, y: 50), tangent2End: CGPoint(x: 100, y: 50), radius: 50)


//二次贝塞尔曲线
context?.addQuadCurve(to: CGPoint.init(x: 150, y: 150), control: CGPoint.init(x: 50, y: 100))

//三次贝塞尔曲线
context?.addCurve(to: CGPoint.init(x: 250, y: 150), control1: CGPoint.init(x: 50, y: 100), control2: CGPoint.init(x: 100, y: 150))

//设置填充颜色
context?.setFillColor(UIColor.brown.cgColor);
//绘制矩形
context?.fill(CGRect.init(x: 50, y: 50, width: 100, height: 50));

//绘制椭圆
context?.strokeEllipse(in: CGRect.init(x: 50, y: 50, width: 100, height: 50));

//旋转
context?.rotate(by: CGFloat.pi * 0.3)

//如果是需要使用矩阵变换(如平移，缩放，旋转等)，需要把添加路径放到后面
//也就是需要把路径添加进加入上下文之前进行,
context?.addArc(center: CGPoint(x: 50, y: 50), radius: 50, startAngle: 0, endAngle: CGFloat.pi, clockwise: true)

//绘制
context?.drawPath(using: .stroke)

```

上下文栈
```swift
1.入栈context?.saveGState()
2.出栈context?.restoreGState()
3.如果出战的次数大于入栈，就会奔溃
4.什么是入栈？就是拷贝当前图形上下文，然后放到栈中， 只使用一个CGContextRef的话，需要很多修改上下文属性（颜色，线宽等）的重复代码，所以可以保存当前上下文属性进入栈中
5.如何理解拷贝图形上下文，我们操作的图形上下文成为A,此时如果图形上下文就设置了context?.setStrokeColor(UIColor.green.cgColor)这一个属性，入栈（我们把拷贝后入栈的成为B），然后我们继续操作当前上下文(现在称为 B)，但是我们出栈 A， 当前上下文的样式都是只有一个context?.setStrokeColor(UIColor.green.cgColor)的状态！说白了，就是保存某种图形上下文的状态！
5.出栈的上下文，将样式赋值给当前样式，然后释放

```

出栈入栈参考自 [王鑫20111](https://www.jianshu.com/u/aea30b9a5690) 的
https://www.jianshu.com/p/604b386d0468

