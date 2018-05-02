---
title: Swift与JavaScript的交互(通过JavaScriptCore)
date: 2018-02-27 10:43:58
tags: [Swift,iOS]
categories: [iOS]
---

# __前言__ #
此篇参考自[iOS开发 - Swift使用JavaScriptCore与JS交互](https://www.jianshu.com/p/c11f9766f8d5)
作者 ：[天秤vs永恒](https://www.jianshu.com/u/4f3505a8ff68)
```swift
实践通过 JavaScriotCore 来实现 Swift 和 JS 的交互。
```

## 一、JavaScriptCore中的类
```
JSContext：JSContext是JS的执行环境，通过evaluateScript()方法可以执行JS代码
JSValue：  JSValue封装了JS与ObjC中的对应的类型，以及调用JS的API等
JSExport： JSExport是一个协议，遵守此协议，就可以定义我们自己的协议，
在协议中声明的API都会在JS中暴露出来，这样JS才能调用原生的API
```

## 二、在 Swift 中调用 JS 有两种方法。
### 1、直接通过 JSContext 执行 JS 代码。
```swift
import JavaScriptCore    //记得导入JavaScriptCore


let context: JSContext = JSContext()
let result1: JSValue = context.evaluateScript("1 + 3")
print(result1)  // 输出4

// 定义js变量和函数
context.evaluateScript("var num1 = 10; var num2 = 20;")
context.evaluateScript("function multiply(param1, param2) { return param1 * param2; }")

// 通过js方法名调用方法
let result2 = context.evaluateScript("multiply(num1, num2)")
print(result2 ?? "result2 = nil")  // 输出200

// 通过下标来获取js方法并调用方法
let squareFunc = context.objectForKeyedSubscript("multiply")
let result3 = squareFunc?.call(withArguments: [10, 20]).toString()
print(result3 ?? "result3 = nil")  // 输出200

```

### 2、 在Swift中通过JSContext注入模型，然后调用模型的方法

1. 首先定义一个协议SwiftJavaScriptDelegate 该协议必须遵守JSExport协议
```swift
// 定义协议SwiftJavaScriptDelegate 该协议必须遵守JSExport协议
@objc protocol SwiftJavaScriptDelegate: JSExport {

// js调用App的返回方法
func popVC()

// js调用App的showDic。传递Dict 参数
func showDic(_ dict: [String: AnyObject])

// js调用App方法时传递多个参数 并弹出对话框 注意js调用时的函数名
func showDialog(_ title: String, message: String)

// js调用App的功能后 App再调用js函数执行回调
func callHandler(_ handleFuncName: String)

}
```
2. 然后定义一个模型 该模型实现SwiftJavaScriptDelegate协议
__(这里注意，如果有更改 UI 的需求，那么需要回到主线程。因为调用不在主线程)__
```swift
// 定义一个模型 该模型实现SwiftJavaScriptDelegate协议
@objc class SwiftJavaScriptModel: NSObject, SwiftJavaScriptDelegate {

weak var controller: UIViewController?
weak var jsContext: JSContext?

func popVC() {
if let vc = controller {
DispatchQueue.main.async {
vc.navigationController?.popViewController(animated: true)
}

}
}

func showDic(_ dict: [String: AnyObject]) {

print("展示信息：", dict,"= = ")

// 调起微信分享逻辑
}

func showDialog(_ title: String, message: String) {

let alert = UIAlertController(title: title, message: message, preferredStyle: .alert)
alert.addAction(UIAlertAction(title: "确定", style: .default, handler: nil))
self.controller?.present(alert, animated: true, completion: nil)
}

func callHandler(_ handleFuncName: String) {

let jsHandlerFunc = self.jsContext?.objectForKeyedSubscript("\(handleFuncName)")
let dict = ["name": "sean", "age": 18] as [String : Any]
let _ = jsHandlerFunc?.call(withArguments: [dict])

}
}
```
3. 然后使用WebView加载对应的网页，这里加载例子中的demo.html文件
```swift
func setWebView(){
webView = UIWebView(frame: self.view.bounds)
view.addSubview(webView)
webView.delegate = self
webView.scalesPageToFit = true

// 测试加载本地Html页面
let url = Bundle.main.url(forResource: "demo", withExtension: "html")
let request = URLRequest(url: url!)

// 加载网络Html页面 请设置允许Http请求
//        let url = URL(string: "https://www.jianshu.com/u/50bd017bb4ba")
//        let request = URLRequest(url: url!)
webView.loadRequest(request)
}
```

4. 在webViewDidFinishLoad代理中将我们定义的模型注入到网页中，暴露给JS
```swift
func webViewDidFinishLoad(_ webView: UIWebView) {

setContext()
}

func setContext(){

let context = webView.value(forKeyPath: "documentView.webView.mainFrame.javaScriptContext") as! JSContext

let model = SwiftJavaScriptModel()
model.controller = self
model.jsContext = context

// 这一步是将SwiftJavaScriptModel模型注入到JS中，在JS就可以通过WebViewJavascriptBridge调用我们暴露的方法了。
context.setObject(model, forKeyedSubscript: "WebViewJavascriptBridge" as NSCopying & NSObjectProtocol)

// 注册到网络Html页面 请设置允许Http请求
let curUrl = self.webView.request?.url?.absoluteString  //WebView当前访问页面的链接 可动态注册
context.evaluateScript(curUrl)

context.exceptionHandler = { (context, exception) in
print("exception：", exception as Any)
}

}
```

5. Swift与JS方法互相调用

**JS调用Swift方法**
```JavaScript
WebViewJavascriptBridge.showDic({
'title' : '字典传值，PierceDark 的博客',
'description' : '欢迎交流学习',
'url' : 'https://www.jianshu.com/u/50bd017bb4ba'
})
```
**Swift调用JS方法并传参**
```swift
func callHandler(handleFuncName: String) {

let jsHandlerFunc = self.jsContext?.objectForKeyedSubscript("\(handleFuncName)")
let dict = ["name": "sean", "age": 18]
jsHandlerFunc?.callWithArguments([dict])
}

```

#### 下载地址：

Github：[https://github.com/YunyueLin/SwiftJavaScriptCore](https://link.jianshu.com/?t=https://github.com/YunyueLin/SwiftJavaScriptCore)。
