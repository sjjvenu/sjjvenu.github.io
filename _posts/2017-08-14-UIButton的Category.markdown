---
layout: post
title: UIButton的Category
date: 2017-08-14 17:32:24.000000000 +08:00
---


&emsp;&emsp;曾在网上看到过一个这样的文章[iOS防止UIButton重复点击的三种实现方式](http://www.jianshu.com/p/7bca987976bd),其中利用runtime来解决问题的思路很高端，但是使用过程中会碰到unrecognized selector sent to instance的问题。下面来解释产生这个问题的原因。

&emsp;&emsp;源代码:

UIButton+TdxNoRepeatButton.h
```objc
#import <UIKit/UIKit.h>

@interface UIButton (TdxNoRepeatButton)

@property (nonatomic, assign) NSTimeInterval tdx_acceptEventInterval;       // 重复点击的间隔

@end
```
UIButton+TdxNoRepeatButton.m
```objc
#import "UIButton+TdxNoRepeatButton.h"
#import <objc/runtime.h>


// 因category不能添加属性，只能通过关联对象的方式。
static const char *UIControl_acceptEventInterval = "UIControl_acceptEventInterval";
static const char *UIControl_acceptEventTime = "UIControl_acceptEventTime";

@interface UIButton (TdxNoRepeatButton)

@property (nonatomic, assign,) NSTimeInterval tdx_acceptEventTime;           //当前点击时间

@end

@implementation UIButton (TdxNoRepeatButton)

- (NSTimeInterval)tdx_acceptEventInterval {
    return  [objc_getAssociatedObject(self, UIControl_acceptEventInterval) doubleValue];
}

- (void)setTdx_acceptEventInterval:(NSTimeInterval)tdx_acceptEventInterval {
    objc_setAssociatedObject(self, UIControl_acceptEventInterval, @(tdx_acceptEventInterval), OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}


- (NSTimeInterval)tdx_acceptEventTime {
    return  [objc_getAssociatedObject(self, UIControl_acceptEventTime) doubleValue];
}

- (void)setTdx_acceptEventTime:(NSTimeInterval)tdx_acceptEventTime {
    objc_setAssociatedObject(self, UIControl_acceptEventTime, @(tdx_acceptEventTime), OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}


// 在load时执行hook
+ (void)load {
    Method before   = class_getInstanceMethod(self, @selector(sendAction:to:forEvent:));
    Method after    = class_getInstanceMethod(self, @selector(tdx_sendAction:to:forEvent:));
    method_exchangeImplementations(before, after);
}

- (void)tdx_sendAction:(SEL)action to:(id)target forEvent:(UIEvent *)event {
    if ([NSDate date].timeIntervalSince1970 - self.tdx_acceptEventTime < self.tdx_acceptEventInterval) {
        return;
    }
    
    if (self.tdx_acceptEventInterval > 0) {
        self.tdx_acceptEventTime = [NSDate date].timeIntervalSince1970;
    }
    
    [self tdx_sendAction:action to:target forEvent:event];
}


@end
```


unrecognized selector产生的原因
---

&emsp;&emsp;我们在利用这个UIButton的Category时，如若遇到UITabBarButton、UISegmentedControl继承自UIControl的类就会产生崩溃，因为这些类并没有实现tdx_sendAction:to:forEvent函数。为什么会出现这种情况呢，我们明明是对UIButton进行Category扩展。首先我们来看看class_getInstanceMethod函数说明:
```objc
/** 
 * Returns a specified instance method for a given class.
 * 
 * @param cls The class you want to inspect.
 * @param name The selector of the method you want to retrieve.
 * 
 * @return The method that corresponds to the implementation of the selector specified by 
 *  \e name for the class specified by \e cls, or \c NULL if the specified class or its 
 *  superclasses do not contain an instance method with the specified selector.
 *
 * @note This function searches superclasses for implementations, whereas \c class_copyMethodList does not.
 */
OBJC_EXPORT Method class_getInstanceMethod(Class cls, SEL name)
    OBJC_AVAILABLE(10.0, 2.0, 9.0, 1.0);
```
&emsp;&emsp;注意note里面的话，这个函数会搜索父类的实现，而我们需要hook的函数正好是在UIControl中实现的，而不是UIButton,所以此处会hook所有的UIControl的sendAction:to:forEvent:函数。继而UITabBarButton、UISegmentedControl继承自UIControl的类都会被hook到，因此才会产生unrecognized selector错误。

解决方案
---
&emsp;&emsp;既然知道产生问题的原因，那么下面我们来解决这个问题，有两种解决方案。

1、从UIControl入手。我们要hook的函数来自UIControl，那么我们可以为UIControl添加Category，而不是UIButton，这样所有继承自UIControl的类都会添加这个Category。

2、修改hook实现。修改load函数后的实现如下：
```objc
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];
        
        SEL originalSelector = @selector(sendAction:to:forEvent:);
        SEL swizzledSelector = @selector(tdx_sendAction:to:forEvent:);
        
        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
        
        BOOL success = class_addMethod(class, originalSelector, method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod));
        if (success) {
            class_replaceMethod(class, swizzledSelector, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod));
        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}
```
&emsp;&emsp;简单说下上述实现的思路。首先取到class、originalSelector、swizzledSelector、originalMethod、swizzledMethod，然后利用class_addMethod这个函数，此函数文档如下：
```objc
/** 
 * Adds a new method to a class with a given name and implementation.
 * 
 * @param cls The class to which to add a method.
 * @param name A selector that specifies the name of the method being added.
 * @param imp A function which is the implementation of the new method. The function must take at least two arguments—self and _cmd.
 * @param types An array of characters that describe the types of the arguments to the method. 
 * 
 * @return YES if the method was added successfully, otherwise NO 
 *  (for example, the class already contains a method implementation with that name).
 *
 * @note class_addMethod will add an override of a superclass's implementation, 
 *  but will not replace an existing implementation in this class. 
 *  To change an existing implementation, use method_setImplementation.
 */
OBJC_EXPORT BOOL class_addMethod(Class cls, SEL name, IMP imp, 
                                 const char *types) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);
```
&emsp;&emsp;这个函数会在当前类上添加一个实例方法，并覆盖父类的方法返回YES。如果当前类有这个方法则无法覆盖返回NO。而我们hook的实现返回NO才会使用method_exchangeImplementations交换两者实现，这样有效的避免了替换掉当前类并未实现而在父类实现的方法，而所引发的unrecognized selector问题。
