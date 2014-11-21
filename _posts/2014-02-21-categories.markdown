---
layout: post
title: "Categories(类别)"
date: 2014-02-21 15:16:58 +0800
comments: true
categories: 
---

category允许你在运行时添加一些方法到一个现有的类中.任何类,甚至苹果提供的Cocoa类,都可以用category来扩展.并且这些新添加的方法对类中的所有实例都是有效的.声明一个类别是非常容易的.看上去就像声明一个类接口声明后面加上一个括号,括号内写上类别的名字. 例如:

{% highlight C++ %}
 @interface NSMutableString (PTLCapitalize)
   - (void)ptl_capitalize;
   @end
{% endhighlight %}

`PTLCapitalize`就是类别的名字.注意,这里没有实例变量的声明.类别是不能声明实例变量的,也不可以合成实例变量的属性(其实都是一回事).
<!-- more -->

>类别可以声明属性,因为这只是另外一种方式来声明方法.但是他们不能合成属性因为没办法创建实例变量.这个`PTLCapitalize`类别不需要`ptl_capitalize`在任何地方被实现.假如`ptl_capitalize`没有被实现,并且有一个调用者企图去调用它,系统会抛出一个异常.这里编译器没有给你任何提示.假如你去实现`ptl_capitalize`,那么按照"惯例",这将会看起来像下面这样:

{% highlight C++ %}
 @implementation NSMutableString (PTLCapitalize)
   - (void)ptl_capitalize {
    [self setString:[self capitalizedString]];
   }
@end
{% endhighlight %}

我说的"按照惯例",是因为并没有要求要在类别的实现中定义它,或者说类别的实现必须和类别接口有相同的名字.然而,假如你提供一个`@implementation`的block叫做`PTLCapitalize`,那就必须所有的方法都来自于叫做`PTLCapitalize`的`@interface`block.

技术上,一个类别可以重载掉方法,但是这么做是非常危险的,而且也不推荐这样做.假如,两个category实现了相同的方法,哪一个会被使用是无法确定的.假如一个class在以后分割到类别来进行
维护,你重载的方法将会成为无法确定的行为,这将为造成让人疯狂的bug追踪..并且,使用这种特性会让代码难以理解.category重载还规定无法去调用原来的方法.对于debugg的话,我还是建议使用`swizzling`.

>因为有可能造成冲突,所以你应该对你的category方法添加一个前缀,然后是下划线,如示例`ptl_capitalize`所示的一样.Cocoa一般不会使用嵌入的下划线,但是对于这种情况的话,他是可选方案中最清晰的.

category正确的用法就是给现有的class提供实用的方法.当你这么做的时候,我建议命名头文件和实现文件的时候用原始class的名字加上扩展的名字.例如,你可以创建一个简单的NSDate分类`PTLExtensions`,如下:

{% highlight C++ %}
NSDate+PTLExtensions.h
   @interface NSDate (PTLExtensions)
   - (NSTimeInterval)ptl_timeIntervalUntilNow;
   @end
NSDate+PTLExtensions.m
   @implementation NSDate (PTLExtensions)
   - (NSTimeInterval)ptl_timeIntervalUntilNow {
    return -[self timeIntervalSinceNow];
   }
@end
{% endhighlight %}

####+load
category在运行时依附于class的.有可能这个库定义一个category是动态加载的,所以category可以被很晚加载.(虽然你不可以在iOS中写入你自己的动态库,但是系统的框架是动态加载的,并且包含category).Objective-C 提供一个钩子叫`+load`,这个钩子在category第一次被依附的时候运行.正如`+initialize`,你可以使用这个方法来实现category特性来设置,比如初始化静态变量.你不能在一个category中安全的使用`+initialize`方法.因为class已经实现了.

我希望你准备问一些明显的问题:"假如category不能使用`+initialize`,因为他们可能和其他category冲突,如果多个category实现了`+load`呢?"这是Objective-C runtime为数不多的真正神奇部分.这个`+load`方法是runtime中是一个特别封装,所以每一个category都可以实现它并且所有的实现都可以运行.不过无法保证顺序.并且你不应该试图去手动调用`+load`.

`+load`被调用是不管category是被静态加载还是动态加载的.只要category被添加到runtime中,`+Load`就会被调用,这常常是在程序启动之后,main之前,但是可能会很晚很晚.

Classes可以有自己的`+load`方法(不是category中定义的),并且他们也是在class被添加到runtime的时候被调用.这种方法很少有用的,除非你要动态的添加一个类.

