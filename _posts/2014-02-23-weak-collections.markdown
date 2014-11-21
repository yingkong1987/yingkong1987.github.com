---
layout: post
title: "Weak Collections & NSCache"
date: 2014-02-23 23:38:15 +0800
comments: true
categories: 
---


最常见最优秀的Cocoa集合是`NSArray`,`NSSet`和`NSDictionary`.但是在某些情况时是不适合的.`NSArray`和`NSSet`会对里面的对象retain.`NSDictionary`不仅会retain存入的值,而且还会copy他的key.这些行为通常正是你想要的,但是对于某些问题来说,这样对你是不利的.幸运的是,其他的集合已经在iOS6中生效了:`NSPointerArray`,`NSHashTable`和`NSMapTable`.它们在苹果的文档内被统称为指针集合类,并且有时使用NSPointerFunctions类来配置.

<!-- more -->

`NSPointerArray`类似于`NSArray`,`NSHashTable`类似于`NSSet`,还有`NSMapTable`类似于`NSDictionary`.不管是这些中的哪一种新集合类都可以被配置成弱引用,非对象指针,或者其他的非常用状况.`NSPointerArray`额外的好处就能够存储`NULL`值,这个在`NSArray`时代是一个普遍的问题.

>苹果文件关于指针集合类的通常是指的是垃圾回收,因为这些类最初被开发来垃圾回收是在10.5中.这些类现在使用ARC弱引用编译.这些并不是总是清晰的在主类引用中,但是结果表明在`NSPointerFunctions`类引用中.

指针集合类能广泛的使用一个`NSPointerFunctions`对象来配置,但是大多数情况下,它类似于通过一个`NSPointerFunctionsOptions`标记到`–initWithOptions:`方法.最常见的情况如:`+weakObjectsPointerArray`,有它自己的构造函数.

有关更多信息,请参见相应的类文档Collections Programming Topics.还有个NSHipster的文章[“NSHashTable & NSMapTable”](http://nshipster.com/nshashtable-and-nsmaptable/).
[网上找的中文翻译版](http://www.yuzhongleixueren.com/blog/2014/02/14/nshashtable-and-nsmaptable/); 


###NSCache
一个最常见使用弱集合的原因就是实现一个缓存功能.不过,在大多数情况下,你可以使用基础缓存对象`NSCache`来代替.大多数情况使用起来就像`NSDictionary`..调用`objectForKey:`,`setObject:forKey: `和`removeObjectForKey:`.

`NSCache`有许多被忽视的特性,比如它是线程安全的.你可以在任何线程修改`NSCache`而不需要锁.`NSCache`的目的也是在于集成对象并符合`<NSDiscardableContent>`协议.最常见的符合`<NSDiscardableContent>`协议的一种类型是`NSPurgeableData`.当调用`beginContentAccess`和`endContentAccess`时,你可以控制它安全的丢弃这个对象.这不仅在你APP运行的时候提供了自动缓存管理,它甚至能在APP暂停的时候帮忙.通常情况下,当内存紧张和内存警告后没有释放出足够的空间时,iOS开始杀死后台暂停程序.在这种情况下,你的APP不会收到代理消息.它就是被这么干掉了.但是假如你使用了`NSPurgeableData`,iOS为你释放这个内存,甚至当你被暂停时.

更多消息,请查看`NSCache`,`<NSDiscardableContent>,`和`NSPurgeableData`的相关文档.
