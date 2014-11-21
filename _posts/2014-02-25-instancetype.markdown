---
layout: post
title: "instancetype"
date: 2014-02-25 23:11:33 +0800
comments: true
categories: 
---

Objective-C 一直有一些敏感的子类化问题. 思考下面的情况:

{% highlight C %}
@interface Foo : NSObject
+ (Foo *)fooWithInt:(int)x; @end
@interface SpecialFoo : Foo
@end
...
SpecialFoo *sf = [SpecialFoo fooWithInt:1];

{% endhighlight %}

<!-- more -->

这段代码会产生一个警告:` Incompatible pointer types initializing ’SpecialFoo *’ with an expression of type ’Foo *`. 这个问题是因为`fooWithInt`方法返回了一个`Foo`对象.所以编译器不知道这个返回的类型实际是一个更明确的类(SpecialFoo).这是一个相当普遍的情形. 思考:`[NSMutableArray array]`. 假如返回的是一个`NSArray`,编译器会不允许你分配到一个子类上(NSMutableArray).

这个情况有几种可能解决的办法.首先,你可以重载`fooWithInt:`这个方法:

{% highlight C %}
@interface SpecialFoo : Foo
+ (SpecialFoo *)fooWithInt:(int)x; 
@end

@implementation SpecialFoo
+ (SpecialFoo *)fooWithInt:(int)x {
 return (SpecialFoo *)[super fooWithInt:x];
}
{% endhighlight %}
这个方法虽然有效,但是很不方便.你必须要覆盖很多方法来添加类型转换.
你也可以执行这个类型转对在调用中:

{% highlight C %}
SpecialFoo *sf = (SpecialFoo *)[SpecialFoo fooWithInt:1];
{% endhighlight %}
这样的话..在`SpecialFoo`中是更方便了,但是对于调用者有产生了许多书写工作了.而且添加大量的强制类型转换会消除了类型检查,所以它也更容易出错.

大多数的解决方法是让他的返回类型为`id`:

{% highlight C %}
@interface Foo : NSObject 
+ (id)fooWithInt:(int)x; 
@end
  
@interface SpecialFoo : Foo
@end
...
SpecialFoo *sf = [SpecialFoo fooWithInt:1];
{% endhighlight %}

这个方法相当方便,还消除了类型检查.可用的解决方法中,它是最好的选择,直到最近.这就是为什么cocoa中大多数的构造函数返回`id`.

Cocoa有着非常一致的命名习惯.任何以`init`开头的方法都要假定返回一个那个类型的对象.能让编译器来做这些吗?答案是肯定的,并且最新版本的Clang就可以做到.所以现在,假如你有一个方法叫做`initWithFoo: `,并返回`id`,编译器会假设返回的类型是实际上这个类的对象,并且在你类型不匹配的时候给出一个警告.

这个自动转换对于`init`方法是极好的,但是这个例子是一个简单的构造函数,`+fooWithInt:`.这种情况下编译器还能帮忙吗?当然可以,但是不是自动的.这个便利构造函数的命名习惯没有`init`方法一样的强大.`SpecialFoo`可以有一个便利构造函数像` +fooWithInt:specialThing:`.编译器没有更好的办法去从这个命名中发现这是要返回一个`SpecialFoo`,所以它不会尝试.相反,Clang添加了一个新的类型,`instancetype`.作为返回类型,`instancetype`表示着"当前的类".所以现在,你可以声明你的方法如下:

{% highlight C++ %}
@interface Foo : NSObject
+ (instancetype)fooWithInt:(int)x; 
@end

@interface SpecialFoo : Foo
@end
...
SpecialFoo *sf = [SpecialFoo fooWithInt:1];
{% endhighlight %}

为了一致性,最好在`init`方法和遍历构造函数内都使用`convenience constructors`做返回类型.