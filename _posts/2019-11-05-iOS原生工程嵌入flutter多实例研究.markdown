---
layout: post
title: iOS原生工程嵌入flutter多实例研究
date: 2019-11-05 11:32:24.000000000 +08:00
---


背景
-----------

​	随着跨平台开发越来越流行，flutter的横空出世，给很多人在h5、rect native之外的另一种选择。由于h5和rect native都存在着性能的问题，而谷歌开始推出flutter时宣传性能出众，更贴近原生，于是便想着用现有的iOS工程嵌入flutter，来看看效果。以下是对flutter的一些研究。


嵌入Flutter
-----------

##### 1、官方嵌入方法

​	官方提供的嵌入方法是基本引入的pod工程，详细情况可看教程https://github.com/flutter/flutter/wiki/Add-Flutter-to-existing-apps 。这种方法适用于已经引入了pod工程的项目，而我现在的项目是没有引入pod的，引入pod又会破坏现有的工程体系，所以只能另外其他嵌入方法。

##### 2、静态库方法

​	首先看到的是这篇文章<https://www.infoq.cn/article/Nizkk*FGGYuzeyM8p59p>，文章中分析了flutter编译后的产物，如下图：

![有赞Flutter混编方案](https://static001.infoq.cn/resource/image/41/88/41efea436524f2e82bc1e73a607ef688.png)

​	该文章中直接利用flutter生成的Flutter.framework、App.framwork和其他的.a文件，加入工程中，少了很多对工程的改造，只需要引入一些静态库文件，正是我需要找的方法。但.a文件管理起来不是很方便，此时又惊喜地发现pod工程设置中可以直接生成framework，只需要在Podfile首行添加一句use_frameworks!。

![WX20191105-142020.png](https://i.loli.net/2019/11/05/VYZlWfQxOasCXpy.png)

​	在flutter项目下分别编译simulator和device，编译命令:flutter build ios --simulator，flutter build ios --release --no-codesign，查看编译后产物，果然生成了framework文件，大功造成！

## Flutter使用

​	在接触flutter之前就听说闲鱼有一套很有名的flutter插件flutter_boost，也符合我的初始目的，原生工程和flutter混合开发。

##### 1、使用flutter_boost	

​	flutter_boost设计思路使用的是单engine单ViewController，内存管理和界面路由比较高效。和flutter官方一样找不到FlutterView类似的View级别的类，只能尝试使用ViewController下的view了。

​	使用flutter_boost创建的view代码类似如下：

```objective-c
FLBFlutterViewContainer *vc1 = FLBFlutterViewContainer.new;
[vc1 setName:@"first" params:@{}];
vc1.view.frame = CGRectMake(0,0,100,100);
[self.view addSubview:vc1.view];
```

​	此时发现虽然可以把vc里面的view贴到我想要的视图上去，但是却无法完成路由的操作，始终显示的是flutter的初始化界面，路由的指定就当是在push界面的时候才会完成。而且此view还存在问题，无法响应事件，flutter里面的setState操作无法完成，把flutter的view当成原生view这样创建行不通。查询资料后发现可以如此创建：

```objective-c
__weak __typeof(self)weakSelf = self;
self.ctr4.modalPresentationStyle = UIModalPresentationOverCurrentContext;
[self presentViewController:weakSelf.ctr4 animated:NO completion:^{
    [weakSelf dismissViewControllerAnimated:NO completion:^{
        [weakSelf addChildViewController:weakSelf.ctr4];
        [weakSelf.view bringSubviewToFront:weakSelf.tabbarContainer];
    }];
}];
```

​	这样解决了显示的问题和事件的问题，接着我进行了第二个界面的尝试，此时flutter_boost的应用场景限制问题就出来了，发现只有后创建的view才会显示出来，前面创建的view界面都是一片空白。这应当就是flutter_boost单例设计模式，而我却在同一个视图下创建出来两个flutter的vc，engine也只有一份，才会造成如此问题。

##### 2、使用官方创建方法

​	官方提供创建方法如下：

```objective-c
FlutterViewController* flutterViewController = [[FlutterViewController alloc] initWithEngine:self.engine nibName:nil bundle:nil];
[flutterViewController setInitialRoute:@"route1"];
```

​	然而此方法官方也埋坑了，这个方法和`initWithEngine` 搭配使用时，并没有起作用，传入到main.dart里面的`window.defaultRouteName`一直是 `/` 根目录符号，网上也有解决方案

[flutter多实例实战]: https://juejin.im/post/5c6e84156fb9a049a5718047	"flutter多实例实战"

​	我没有尝试按文章上的方法继续去改造，因为最后作者也不推送多实例方法去构造，我试着创建出来两个同时带有FlutterView的界面，发现这两个界面切换时出现闪屏的情况，这应当也是由于共用一个engine所造成的。

​	接着在github的issue中找到另外一种创建方式，一个真正多实例多engine的方式，就是使用FluttterViewController的initWithProject方法，创建方式如下：

```objective-c
flutterViewController = [[FlutterViewController alloc] initWithProject:nil nibName:nil bundle:nil];
[flutterViewController setInitialRoute:routeName];
messageChannel = [FlutterMethodChannel
                                                methodChannelWithName:@"tdxFlutterCallNativeChannel"
                                                binaryMessenger:flutterViewController];
        
        [messageChannel setMethodCallHandler:^(FlutterMethodCall* call,
                                               FlutterResult result) {
            if ([@"test" isEqualToString:call.method]) {
                result(@"flutter test");
            }
        }];
        
        flutterViewController.modalPresentationStyle = UIModalPresentationOverCurrentContext;
        [vc presentViewController:flutterViewController animated:NO completion:^{
            [vc dismissViewControllerAnimated:NO completion:^{
                weakSelf.flutterViewController.view.frame = CGRectMake(0, 0, weakSelf.frame.size.width, weakSelf.frame.size.height);
                [weakSelf addSubview:weakSelf.flutterViewController.view];
                [uimgr addChildViewController:weakSelf.flutterViewController];
                [weakSelf bringSubviewToFront:weakSelf.flutterViewController.view];
            }];
        }];
        
        NSString *channelName = @"NativeCallFlutterChannel";
        evenChannal = [FlutterEventChannel eventChannelWithName:channelName binaryMessenger:flutterViewController];
        [evenChannal setStreamHandler:self];
```

​	此方法能将一个完整的FlutterView当成UIView使用起来，需要注意的是析构的时候要释放掉messageChannel和eventChannel。

```objective-c
[evenChannal setStreamHandler:nil];
evenChannal = nil;
[messageChannel setMethodCallHandler:nil];
messageChannel = nil;
```

​	而光做这些还不够，发现页面退出的时候仍然存在内存泄露，后来查询得知是使用initWithProject创建出来的FlutterViewController每个实例自带一个engine，需要在视图释放的时候一并释放掉engine，调用engine的destroyContext函数。

```objective-c
[flutterViewController.engine destroyContext];
```

​	到此，FlutterView的构建工作全部完成，然后看起来挺美好，真正在真机上运行起来发现，像6s以下1g ram的手机，跑起来相当吃力，即使像6s这种2g ram的手机，跑起来流畅不少，但每次FlutterView初始化的过程中会有接近2s左右的白屏时间，甚至还不如网页效果。看来多实例构建FlutterView效果还有待观望，有转入Flutter混合编程的还是推荐使用Flutter单例模式吧。



