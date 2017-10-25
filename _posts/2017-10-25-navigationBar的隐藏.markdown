---
layout: post
title: navigationBar的隐藏
date: 2017-10-25 15:32:24.000000000 +08:00
---


通用方法
-----------

&emsp;&emsp;一般情况下使用setNavigationBarHidden来隐藏导航栏，但在多个ViewController之间切换时来显示隐藏导航栏，会出现闪动的情况，以下来介绍另外一种方式来隐藏。
```objc
- (void)setNavigationBarHidden:(BOOL)hidden animated:(BOOL)animated;
```
setTranslucent设置透明
-----------

&emsp;&emsp;函数解释如下

```
@property(nonatomic, assign, getter=isTranslucent) BOOL translucent;
Description	
A Boolean value indicating whether the navigation bar is translucent (YES) or not (NO).
The default value is YES. If the navigation bar has a custom background image, the default is YES if any pixel of the image has an alpha value of less than 1.0, and NO otherwise.
If you set this property to YES on a navigation bar with an opaque custom background image, the navigation bar applies a system-defined opacity of less than 1.0 to the image.
If you set this property to NO on a navigation bar with a translucent custom background image, the navigation bar provides an opaque background for the image using black if the navigation bar has UIBarStyleBlack style, white if the navigation bar has UIBarStyleDefault, or the navigation bar’s barTintColor if a custom value is defined.
```

&emsp;&emsp;设置导航栏为透明

```objc
        [self.navigationController.navigationBar setTranslucent:YES];
        [[[self.navigationController.navigationBar subviews] objectAtIndex:0] setAlpha:0.0];
        //根据需要设置titleView属性
        [self.navigationItem.titleView setAlpha:0.0];
        [self.navigationItem.titleView setHidden:YES];
        //iOS11需要设置以下属性，否则在pushViewController的时候会出现闪屏现象
        [self.navigationController.navigationBar setBackgroundImage:[UIImage new]
                                                      forBarMetrics:UIBarMetricsDefault];
        self.navigationController.navigationBar.shadowImage = [UIImage new];
```

&emsp;&emsp;还原

```objc
        [self.navigationController.navigationBar setTranslucent:NO];
        [[[self.navigationController.navigationBar subviews] objectAtIndex:0] setAlpha:1.0];
        [self.navigationItem.titleView setAlpha:1.0];
        //如果切换到其他界面需要将translucent设置为NO，在本界面做显示隐藏操作不需要
        [self.navigationItem.titleView setHidden:NO];
        [self.navigationController.navigationBar setBackgroundImage:[self imageWithColor:backColor]
                                                      forBarMetrics:UIBarMetricsDefault];
        self.navigationController.navigationBar.shadowImage = [self imageWithColor:backColor];
```

同一界面动态显示与隐藏
----------

&emsp;&emsp;利用上述设置透明的方法可以同一个界面实现导航栏的动态显示与隐藏，但是在显示导航栏的时候不宜将translucent设置为NO，这样会造成在这个界面出现闪动的问题。


Demo参考:[Demo](https://github.com/sjjvenu/NavBarDemo)
