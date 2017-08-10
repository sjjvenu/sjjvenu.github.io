---
layout: post
title: iOS静态库中的Category
date: 2017-08-10 17:32:24.000000000 +08:00
---


&emsp;&emsp;一个项目中使用了一个包含 category 的静态库，但是此项目在运行过程中，该静态库调用category增加的方法处，却报selector not recognized异常。

产生的原因
---

&emsp;&emsp;苹果官方文档中的这个 Q&A QA1490:Building Objective-C static libraries with categories 已经说明了这个问题产生的原因：

    这个异常是因为标准 UNIX 静态库、linker 以及 Objective-C 的动态性三者之间的实现导致的，Objective-C 不会为方法定义 linker symbols，它只会为每一个类定义 linker symbols。如果你使用 category 扩展了一个已经存在的类，那么 linker 不会将已有类的实现跟 category 的实现连接起来，这就导致了调用静态库中 category 中新增加的方法时抛出 selector not recognized 的异常。

&emsp;&emsp;当你组建（build）一个动态库（.dylib）、一个framework、一个可加载的bundle（bundle）或者一个可执行的二进制文件的时候，这些文件被链接器链接到一起来生成一些操作系统认为“可用的”的东西，例如一些可以直接载入指定内存地址的东西。
       
&emsp;&emsp;然而，当你组建静态库（static library）的时候，所有的对象文件被简单的添加到一个大的归档文件，这就是.a（archive）文件的由来。所以.a文件就是一些对象文件的归档（想象一下没有经过压缩的tar归档或者zip归档）。拷贝单个.a文件要比拷贝一堆.o文件简单的多（Java也是一样，为了使用方便你可以把一些.class文件放到一个.jar归档里）。
       
&emsp;&emsp;在把二进制链接到静态库（=archive）的时候，编译器会获得一张含有在归档里所有符号的表然后检查哪些符号被这些二进制文件引用了。只有包含被引用的符号的对象文件会被链接器真正的载入，并被链接进程处理。例如，如果你的文档中有50个对象文件，但是只有20个包含被二进制使用的符号，那么只有这20个文件会被链接器载入，另外30个会被链接进程完全忽略。
       
&emsp;&emsp;在C和C++代码里，这种机制能很好地工作，因为这些语言尽可能在编译时（C++也有一些runtime特性）去做这些事。然而Objective-C是一种不同的语言。OC非常依赖runtime特性，而且很多OC的特性都是只支持runtime的。OC类里面实际上也有类似于C函数或全局C变量的符号表（至少现在的OC runtime是这样）。编译器可以识别出一个类有没有被引用，从而确定这个类是不是被使用了。如果你使用了静态库里的对象文件中的类，这个对象文件就会被链接器载入，因为链接器发现它的一个符号被使用了。
       
&emsp;&emsp;类别是runtime下特有的特性，类别并不会像类或者函数一样被符号化，也就是说，编译器并不能检测到类别是不是被使用了。
       
&emsp;&emsp;如果链接器载入了一个包含OC代码的对象文件，这些OC代码的所有部分都是编译阶段的一部分。所以当一个包含类别的对象文件因为任何符号被认为“在使用”（可能是一个类，可能是一个函数，也可能是一个全局变量），它的类别也会被载入并在runtime中变得可用。一个只包含类别的对象文件里没有被编译器认为“在使用”的符号，所以是不会被载入的。

解决方案
---
1、将category文件跟静态库一起导入到工程中。

2、用xcode创建lib时，-ObjC这个flag默认是有的，但是最终程序还是会抛这个异常，这是因为linker的bug，对于64位osx程序和iOS程序，这个bug导致只包含category而不包含class的文件没法从静态库中加载。

&emsp;&emsp;所以，apple建议我们为要最终程序的linker加上-all_load或者-force_load参数。

&emsp;&emsp;-all_load选项强制linker加载所有包中的所有对象文件，即使文件中没有Objective-C代码也加载。-force_load是从Xcode3.2开始有的，它使得linker获取包加载的控制权，每个-force_load参数后面都必须跟上一个包的路径，然后这个包的所有对象文件都会被加载。
```objc
-force_load $(BUILT_PRODUCTS_DIR)/<library_name.a>
```
3、解决linker的bug

* 参考facebook的解决方案
```objc
/**
 * Add this macro before each category implementation, so we don't have to use
 * -all_load or -force_load to load object files from static libraries that only contain
 * categories and no classes.
 * See http://developer.apple.com/library/mac/#qa/qa2006/qa1490.html for more info.
 */
#define TT_FIX_CATEGORY_BUG(name) @interface TT_FIX_CATEGORY_BUG_##name @end \
                                  @implementation TT_FIX_CATEGORY_BUG_##name @end

```
&emsp;&emsp;上面的宏定义在 TTCorePreprocessorMacros.h 文件中，在每个 category 的实现文件开头加上：TT_FIX_CATEGORY_BUG({cateory名字}) ，这样就能避免在 iOS 中使用 -ObjC 的 linker 的 bug，但是记住，还是需要把使用静态库的 Target 中的 Building Setting 的 Other Linker Flags 设置成 -ObjC 。

&emsp;&emsp;每个category的.m文件中调用这个宏
```objc
/**
 * Additions.
 */
TT_FIX_CATEGORY_BUG(NSArrayAdditions)

@implementation NSArray (TTCategory)

```

* 利用全局函数

&emsp;&emsp;在只有category的源文件里添加Fakesymbol。如果你想在runtime里使用category，一定要确保你以某种方法在编译时引用了fake symbol，这会使得对象文件以及它里面的OC代码被载入。例如，它可以是一个有空函数体的函数，也可以是一个被访问的全局变量（例如一个全局的int变量，只要它被读或者写了一次就足够了）。和上面其他的解决方法不一样，这种解决方法可以控制哪些category可以在runtime里被编译后的代码使用（可以通过使用这个符号，使它们被链接并变得可用；也可以不使用这个符号，这样链接器就会忽略它）。
 
&emsp;&emsp;如果你有一个包含上百个对象的静态库，但是你的二进制文件只使用了其中的几个，你应该避免使用1~4的方法。否则你会获得一个非常大的包含了所有类（可能这些类根本没被用到）的二进制文件。对于一个类别，你通常不用做特殊的处理；而对于一个类别，考虑一下第5中解决方案，它可以让你只包含你想要的类别。
例如：如果你想使用NSData的类别，添加两个方法compressionData和decompressionData，你可以创建一个这样的头文件。
```objc
// NSData+Compress.h  
@interface NSData (Compression)  
    - (NSData *)compressedData;  
    - (NSData *)decompressedData;  
@end  
```
&emsp;&emsp;再添加一个这样的实现文件
```objc
// NSData+Compress  
@implementation NSData (Compression)  
    - (NSData *)compressedData   
    {  
        // ... magic ...  
    }  
   
    - (NSData *)decompressedData  
    {  
        // ... magic ...  
    }  
@end  
   
void import_NSData_Compression ( ) { } //注意这里***************  
```
&emsp;&emsp;然后确保在你的代码里调用了import_NSData_Compression这个方法。是不是真的调用或者调用的次数并不重要，实际上你并不需要真的去调用它，只需要让编译器认为你调用了这个方法就行了。你可以吧这段代码添加到你的工程的任何地方。
```objc
__attribute__((used)) static void importCategories ()  
{  
    import_NSData_Compression();  
    // add more import calls here  
}  
```
&emsp;&emsp;你并不需要在你的代码里调用importCategories ()，attribute将会让编译器认为它被调用了。

本文参考:

[怎样让静态库（static library)中的category变得可用](http://blog.csdn.net/sonnefish/article/details/49950825)

[使用一个包含category的静态库](http://zhenby.com/blog/2012/08/13/zai-jing-tai-ku-zhong-shi-yong-category/)