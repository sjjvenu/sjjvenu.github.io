---
layout: post
title: WebView与JS交互(转)
date: 2017-08-29 17:32:24.000000000 +08:00
---

iOS中调用HTML
---
1、加载网页
```objc
    NSURL *url = [[NSBundle mainBundle] URLForResource:@"index"
        withExtension:@"html"];
    NSURLRequest *request = [NSURLRequest requestWithURL:url];
    [self.webView loadRequest:request];
```
2、删除
```objc
    NSString *str1 = @"var word =
        document.getElementById('word');";
    NSString *str2 = @"word.remove();";
    [webView  stringByEvaluatingJavaScriptFromString:str1];
    [webView  stringByEvaluatingJavaScriptFromString:str2];
```
3、更改
```objc
    NSString *str3 = @"var change =
        document.getElementsByClassName('change')[0];" "change.innerHTML = '更改内容!';";
    [webView stringByEvaluatingJavaScriptFromString:str3];
```
4、插入
```objc
    NSString *str4 =@"var img = document.createElement('img');"
                     "img.src = 'img_01.jpg';"
                     "img.width = '160';"
                     "img.height = '80';"
                     "document.body.appendChild(img);";
    [webView stringByEvaluatingJavaScriptFromString:str4];
```
5、改变标题
```objc
    NSString *str1 = @"var h1 =
        document.getElementsByTagName('h1')[0];" "h1.innerHTML='改变标题';";
    [webView stringByEvaluatingJavaScriptFromString:str1];
```
6、删除尾部
```objc
    NSString *str2
        =@"document.getElementById('footer').remove();";
    [webView stringByEvaluatingJavaScriptFromString:str2];
```
7、拿出所有的网页和内容
```objc
    NSString *str3 = @"document.body.outerHTML"; 
    NSString *html = [webView stringByEvaluatingJavaScriptFromString:str3];
    NSLog(@"%@", html);
```

在HTML中调用OC
---
```objc
-(BOOL)webView:(UIWebView *)webView shouldStartLoadWithRequest:
(NSURLRequest *)request navigationType:
(UIWebViewNavigationType)navigationType{
    NSString *str = request.URL.absoluteString;
    NSRange range = [str rangeOfString:@"ZJY://"];
    if (range.location != NSNotFound) {
        NSString *method = [str substringFromIndex:range.location + range.length];
        SEL sel = NSSelectorFromString(method);
        [self performSelector:sel];
    }
    return YES; 
}
//     
- (void)getImage{
    UIImagePickerController *pickerImg = [[UIImagePickerController alloc]init];
    pickerImg.sourceType = UIImagePickerControllerSourceTypePhotoLibrary;
    [self presentViewController:pickerImg animated:YES completion:nil];
}
```

JavaScriptCore的使用
---
>:JavaScriptCore是webkit的一个重要组成部分，主要是对JS进行解析和提供执行环境。代码是开源的，可以下下来看看（源码）。iOS7后苹果在iPhone平台推出，极大的方便了我们对js的操作。我们可以脱离webview直接运行我们的js。iOS7以前我们对JS的操作只有webview里面一个函数 stringByEvaluatingJavaScriptFromString，JS对OC的回调都是基于URL的拦截进行的操作。大家用得比较多的是WebViewJavascriptBridge和EasyJSWebView这两个开源库，很多混合都采用的这种方式。

&emsp;&emsp;JavaScriptCore和我们相关的类不是很多，使用起来也非常简单。
```objc
#import "JSContext.h"
#import "JSValue.h"
#import "JSManagedValue.h"
#import "JSVirtualMachine.h"
#import "JSExport.h"
```
#### JSContext
&emsp;&emsp;JS执行的环境，同时也通过JSVirtualMachine管理着所有对象的生命周期，每个JSValue都和JSContext相关联并且强引用context。
#### JSValue
&emsp;&emsp;JS对象在JSVirtualMachine中的一个强引用，其实就是Hybird对象。我们对JS的操作都是通过它。并且每个JSValue都是强引用一个context。同时，OC和JS对象之间的转换也是通过它，相应的类型转换如下：
```
Objective-C type  |   JavaScript type
 --------------------+---------------------
         nil         |     undefined
        NSNull       |        null
       NSString      |       string
       NSNumber      |   number, boolean
     NSDictionary    |   Object object
       NSArray       |    Array object
        NSDate       |     Date object
       NSBlock (1)   |   Function object (1)
          id (2)     |   Wrapper object (2)
        Class (3)    | Constructor object (3)

```
#### JSManagedValue
&emsp;&emsp;JS和OC对象的内存管理辅助对象。由于JS内存管理是垃圾回收，并且JS中的对象都是强引用，而OC是引用计数。如果双方相互引用，势必会造成循环引用，而导致内存泄露。我们可以用JSManagedValue保存JSValue来避免。
#### JSExport
&emsp;&emsp;一个协议，如果JS对象想直接调用OC对象里面的方法和属性，那么这个OC对象只要实现这个JSExport协议就可以了。

## Objective-C -> JavaScript  
```objc
    self.context = [[JSContext alloc] init];

    NSString *js = @"function add(a,b) {return a+b}";

    [self.context evaluateScript:js];

    JSValue *n = [self.context[@"add"] callWithArguments:@[@2, @3]];

    NSLog(@"---%@", @([n toInt32]));//---5

```
&emsp;&emsp;步骤很简单，创建一个JSContext对象，然后将JS代码加载到context里面，最后取到这个函数对象，调用callWithArguments这个方法进行参数传值。（JS里面函数也是对象）
## JavaScript -> Objective-C
&emsp;&emsp;JS调用OC有两个方法：block和JSExport protocol。
#### block(JS function):
```objc
    self.context = [[JSContext alloc] init];

    self.context[@"add"] = ^(NSInteger a, NSInteger b) {
        NSLog(@"---%@", @(a + b));
    };

    [self.context evaluateScript:@"add(2,3)"];

```
&emsp;&emsp;我们定义一个block，然后保存到context里面，其实就是转换成了JS的function。然后我们直接执行这个function，调用的就是我们的block里面的内容了。
#### JSExport protocol:
```objc
//定义一个JSExport protocol
@protocol JSExportTest <JSExport>

- (NSInteger)add:(NSInteger)a b:(NSInteger)b;

@property (nonatomic, assign) NSInteger sum;

@end

//建一个对象去实现这个协议：

@interface JSProtocolObj : NSObject<JSExportTest>
@end

@implementation JSProtocolObj
@synthesize sum = _sum;
//实现协议方法
- (NSInteger)add:(NSInteger)a b:(NSInteger)b
{
    return a+b;
}
//重写setter方法方便打印信息，
- (void)setSum:(NSInteger)sum
{
    NSLog(@"--%@", @(sum));
    _sum = sum;
}

@end

//在VC中进行测试
@interface ViewController () <JSExportTest>

@property (nonatomic, strong) JSProtocolObj *obj;
@property (nonatomic, strong) JSContext *context;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    //创建context
    self.context = [[JSContext alloc] init];
    //设置异常处理
    self.context.exceptionHandler = ^(JSContext *context, JSValue *exception) {
        [JSContext currentContext].exception = exception;
        NSLog(@"exception:%@",exception);
    };
    //将obj添加到context中
    self.context[@"OCObj"] = self.obj;
    //JS里面调用Obj方法，并将结果赋值给Obj的sum属性
    [self.context evaluateScript:@"OCObj.sum = OCObj.addB(2,3)"];

}
```

&emsp;&emsp;demo定义了一个两个数相加的方法，还有一个保存结果的变量。在JS中进行调用这个对象的方法，并将结果赋值sum。唯一要注意的是OC的函数命名和JS函数命名规则问题。协议中定义的是add: b:，但是JS里面方法名字是addB(a,b)。可以通过JSExportAs这个宏转换成JS的函数名字。

&emsp;&emsp;修改下代码：
```objc
@protocol JSExportTest <JSExport>
//用宏转换下，将JS函数名字指定为add；
JSExportAs(add, - (NSInteger)add:(NSInteger)a b:(NSInteger)b);

@property (nonatomic, assign) NSInteger sum;

@end

//调用
[self.context evaluateScript:@"OCObj.sum = OCObj.add(2,3)"];

```
&emsp;&emsp;我们可以定义自己的异常捕获，可以把context，异常block改为自己的：
```objc
    self.context.exceptionHandler = ^(JSContext *context, JSValue *exception) {
            [JSContext currentContext].exception = exception;
            NSLog(@"exception:%@",exception);
        };
```
内存管理
---
&emsp;&emsp;现在来说说内存管理的注意点，OC使用的ARC，JS使用的是垃圾回收机制，并且所有的引用是都强引用，不过JS的循环引用，垃圾回收会帮它们打破。JavaScriptCore里面提供的API，正常情况下，OC和JS对象之间内存管理都无需我们去关心。不过还是有几个注意点需要我们去留意下。
#### 1、不要在block里面直接使用context，或者使用外部的JSValue对象。
```
    //错误代码：
    self.context[@"block"] = ^(){
        JSValue *value = [JSValue valueWithObject:@"aaa" inContext:self.context];
    };
```
```
    //一个比较隐蔽的
     JSValue *value = [JSValue valueWithObject:@"ssss" inContext:self.context];

    self.context[@"log"] = ^(){
        NSLog(@"%@",value);
    };
```
&emsp;&emsp;这个是block里面使用了外部的value，value对context和它管理的JS对象都是强引用。这个value被block所捕获，这边同样也会内存泄露，context是销毁不掉的。
```objc
//正确的做法，str对象是JS那边传递过来。
self.context[@"log"] = ^(NSString *str){
        NSLog(@"%@",str);
    };
```
#### 2、OC对象不要用属性直接保存JSValue对象，因为这样太容易循环引用了
```objc
//定义一个JSExport protocol
@protocol JSExportTest <JSExport>
//用来保存JS的对象
@property (nonatomic, strong) JSvalue *jsValue;

@end

//建一个对象去实现这个协议：

@interface JSProtocolObj : NSObject<JSExportTest>
@end

@implementation JSProtocolObj

@synthesize jsValue = _jsValue;

@end

//在VC中进行测试
@interface ViewController () <JSExportTest>

@property (nonatomic, strong) JSProtocolObj *obj;
@property (nonatomic, strong) JSContext *context;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    //创建context
    self.context = [[JSContext alloc] init];
    //设置异常处理
    self.context.exceptionHandler = ^(JSContext *context, JSValue *exception) {
        [JSContext currentContext].exception = exception;
        NSLog(@"exception:%@",exception);
    };
   //加载JS代码到context中
   [self.context evaluateScript:
   @"function callback (){};

    function setObj(obj) {
    this.obj = obj;
    obj.jsValue=callback;
}"];
   //调用JS方法
   [self.context[@"setObj"] callWithArguments:@[self.obj]];  
}

```
&emsp;&emsp;上面的例子调用JS方法，进行赋值，JS对象保留了传进来的obj，最后，JS将自己的回调callback赋值给了obj，方便obj下次回调给JS；由于JS那边保存了obj，而且obj这边也保留了JS的回调。这样就形成了循环引用。

&emsp;&emsp;怎么解决这个问题？我们只需要打破obj对JSValue对象的引用即可。当然，不是我们OC中的weak。而是之前说的内存管理辅助对象JSManagedValue。

&emsp;&emsp;JSManagedValue 本身就是我们需要的弱引用。用官方的话来说叫garbage collection weak reference。但是它帮助我们持有JSValue对象必须同时满足一下两个条件（不翻译了，翻译了怪怪的！）：
>:The JSManagedValue's JavaScript value is reachable from JavaScript

>:The owner of the managed reference is reachable in Objective-C. Manually adding or removing the managed reference in the JSVirtualMachine determines reachability.

&emsp;&emsp;JSManagedValue 帮助我们保存JSValue，那里面保存的JS对象必须在JS中存在，同时 JSManagedValue 的owner在OC中也存在。我们可以通过它提供的方法
```objc
+ (JSManagedValue )managedValueWithValue:(JSValue )value;

+ (JSManagedValue )managedValueWithValue:(JSValue )value andOwner:(id)owner 
//创建JSManagedValue对象。
- (void)addManagedReference:(id)object withOwner:(id)owner
//通过JSVirtualMachine的方法来建立这个弱引用关系
- (void)removeManagedReference:(id)object withOwner:(id)owner 
//手动移除他们之间的联系
```
&emsp;&emsp;把刚刚的代码改下：
```objc
//定义一个JSExport protocol
@protocol JSExportTest <JSExport>
//用来保存JS的对象
@property (nonatomic, strong) JSValue *jsValue;

@end

//建一个对象去实现这个协议：

@interface JSProtocolObj : NSObject<JSExportTest>
//添加一个JSManagedValue用来保存JSValue
@property (nonatomic, strong) JSManagedValue *managedValue;

@end

@implementation JSProtocolObj

@synthesize jsValue = _jsValue;
//重写setter方法
- (void)setJsValue:(JSValue *)jsValue
{
    _managedValue = [JSManagedValue managedValueWithValue:jsValue];

    [[[JSContext currentContext] virtualMachine] addManagedReference:_managedValue 
    withOwner:self];
}

@end

//在VC中进行测试
@interface ViewController () <JSExportTest>

@property (nonatomic, strong) JSProtocolObj *obj;
@property (nonatomic, strong) JSContext *context;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    //创建context
    self.context = [[JSContext alloc] init];
    //设置异常处理
    self.context.exceptionHandler = ^(JSContext *context, JSValue *exception) {
        [JSContext currentContext].exception = exception;
        NSLog(@"exception:%@",exception);
    };
   //加载JS代码到context中
   [self.context evaluateScript:
   @"function callback (){}; 

   function setObj(obj) {
   this.obj = obj;
   obj.jsValue=callback;
   }"];
   //调用JS方法
   [self.context[@"setObj"] callWithArguments:@[self.obj]];  
}

```
&emsp;&emsp;注:以上代码只是为了突出用 JSManagedValue来保存JSValue，所以重写了setter方法。实际不会写这么搓的姿势。应该根据回调方法传进来参数，进行保存JSValue。
#### 3、不要在不同的 JSVirtualMachine 之间进行传递JS对象。
&emsp;&emsp;一个 JSVirtualMachine可以运行多个context，由于都是在同一个堆内存和同一个垃圾回收下，所以相互之间传值是没问题的。但是如果在不同的 JSVirtualMachine传值，垃圾回收就不知道他们之间的关系了，可能会引起异常。

线程
---
&emsp;&emsp;JavaScriptCore 线程是安全的，每个context运行的时候通过lock关联的JSVirtualMachine。如果要进行并发操作，可以创建多个JSVirtualMachine实例进行操作。

与UIWebView的操作
---
&emsp;&emsp;通过上面的demo，应该差不多了解OC如何和JS进行通信。下面我们看看如何对 UIWebView 进行操作，我们不再通过URL拦截，我们直接取 UIWebView 的 context，然后进行对JS操作。

&emsp;&emsp;在UIWebView的finish的回调中进行获取
```objc
- (void)webViewDidFinishLoad:(UIWebView *)webView
{
    self.context = [webView valueForKeyPath:@"documentView.webView.mainFrame.javaScriptContext"];

}
```
&emsp;&emsp;上面用了私有属性，可能会被苹果给拒了。这边要注意的是每个页面加载完都是一个新的context，但是都是同一个JSVirtualMachine。如果JS调用OC方法进行操作UI的时候，请注意线程是不是主线程。

参考：

http://blog.csdn.net/lizhongfu2013/article/details/9232129

https://developer.apple.com/videos/play/wwdc2013-615/
