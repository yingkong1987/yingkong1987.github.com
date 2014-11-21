---
layout: post
title: "Property 和 Ivar 的最佳实践"
date: 2014-02-20 20:14:40 +0800
comments: true
categories: 
---

在文章的开头,先看下Objective-C runtime 内关于objc的几个基本类型
![](/images/20140220/1.png)

<!-- more -->

####Method
An opaque type that represents a method in a class definition.
(一个不可见的类型,表示一个类定义中的方法)下面是他的结构体.

```
struct objc_method {
    SEL method_name                                          OBJC2_UNAVAILABLE;
    char *method_types                                       OBJC2_UNAVAILABLE;
    IMP method_imp                                           OBJC2_UNAVAILABLE;
}                                                            OBJC2_UNAVAILABLE;
```
####Ivar
An opaque type that represents an instance variable.
(一个不可见的类型,表示一个实例变量)下面是他的结构体.

```
struct objc_ivar {
    char *ivar_name                                          OBJC2_UNAVAILABLE;
    char *ivar_type                                          OBJC2_UNAVAILABLE;
    int ivar_offset                                          OBJC2_UNAVAILABLE;
#ifdef __LP64__
    int space                                                OBJC2_UNAVAILABLE;
#endif
}                                                            OBJC2_UNAVAILABLE;
```

####Category
An opaque type that represents a category.
(一个不可见类型,表示一个类别)下面是他的结构体.

```
struct objc_category {
    char *category_name                                      OBJC2_UNAVAILABLE;
    char *class_name                                         OBJC2_UNAVAILABLE;
    struct objc_method_list *instance_methods                OBJC2_UNAVAILABLE;
    struct objc_method_list *class_methods                   OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols                     OBJC2_UNAVAILABLE;
}                                                            OBJC2_UNAVAILABLE;
```

####objc_property_t
An opaque type that represents an Objective-C declared property.(一个不可见类型,表示一个Objective-C的公开属性)下面是他的结构体.

```
typedef struct objc_property {
    const char *name;
    const char *attributes;
} property_t;
```

属性应该表示一个对象的状态. getter方法在外部应该没有明显的side effect(副作用,后文皆直接书为副作用).它们可能会有一些内部的副作用,比如缓存,但是这些对于调用者应该是不可见的.一般来说,它们应该有要高效的调用并且不应该阻塞.---side effect,不太好怎么解释.原意为副作用,但是这里是中性,意思为边际效应,指的是这个函数还做了基本需求以外的其他事情.(ps.感谢@唐巧 http://weibo.com/tangqiaoboy 还有其他大神对于翻译这词的帮助)

我们应该避免直接访问Ivar(实例变量).要使用访问器来代替直接访问.我讨论了一会儿免责条款,但是现在,我想首先讨论下使用访问器的原因.

在ARC之前,BUG最常见的一个原因就是直接访问了Ivar.开发者没有正确的retain或者release他们的实例变量,所以他们的程序产生了内存泄露或者崩溃.因为ARC自动管理了retain和release,一些开发者可能认为这个规则已经不重要了,但是还有几个其他的理由来建议使用访问器.

* KVO(键值观察)---最关键的原因可能是使用访问器可以让属性被监听到.假如你不使用访问器,你每次修改属性所关联的实例变量时,需要去调用`willChangeValueForKey:` 和 `didChangeValueForKey: `.这个访问器可以自动让这两个方法在他们需要的时候调用.
* 副作用---这个类或这类的一个子类可能在setter方法里包含了副作用.可能发送了Notifications或者用NSUndoManager注册了事件.你不应该避开这些副作用,除非它是有必要去绕开的.同样的,这类或者它的子类可以添加一个缓存到getter中,使得可以绕开getter直接访问实例变量.
* Lazy instantiation(惰性初始化/延迟初始化)---假如一个属性是惰性初始化,你就必须使用访问器来确保这个属性被正确的初始化了.
* Locking(锁定)---假如你采用了锁定属性来管理多线程代码,那么直接访问实例变量将会违反你的锁并且很可能让你的程序崩溃.
* Consistency(一致性)---有人会说,你应该使用存取器,当你知道你在前面几个原因其中的一种情况会需要使用.但是这使得代码非常不好维护.最好的做法是让每一个直接访问的实例变量是可疑的和阐明的.而不是要你不得不经常记住这些实例变量哪些需要访问器,哪些不需要.这使得代码更容易审核,复查和维护.访问器,特别是合成访问器,它们在Objective-C中是高度优化的,所以他们是值得的开销.

刚才说了有几个地方不需要用访问器:

* 访问器内部---显然,你不能在一个访问器本身的内部使用它.一般来说,你也不想在setter里面使用getter(这在某些模式下会产生死循环).一个访问器应该访问自己的实例变量.
* Dealloc---ARC已经大大降低了对`dealloc`的需求,不过仍然会在一些情况中出现.最好不要在dealloc内调用外部的对象.因为对象可能是处于一个不一致的状态,并且很有可能造成观察者接受的几个通知被混淆,那样可能发生属性被修改的时候,实际上整个对象已经被销毁了.
* Initialization(初始化)---和`dealloc`类似,对象可能在初始化期间处于不一致状态,并且通常在此期间你不应该启动通知或者其他副作用.这同样是一个初始化只读变量的常见地方,例如`NSMutableArray`.这样的话你可以避免声明一个读写属性,正是这样,你才能初始化它.

访问器在Objective-C中是高度优化的.并且为维护性和灵活性提供文本特征.一般来说,适用于所有的属性,即使是你在你自己写的文件中,最好也使用他们的访问器.