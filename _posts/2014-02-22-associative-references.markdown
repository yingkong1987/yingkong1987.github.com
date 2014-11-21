---
layout: post
title: "Associative References"
date: 2014-02-22 23:29:07 +0800
comments: true
categories: 
---

Associative references(关联引用)允许你去附加key-value(键值对)数据到任意的对象上.这个功能能让你做很多的事情,但是一个共同的用途就是可以让category添加数据到property.

<!-- more -->

以`Person`类的情况来考虑,比如说你想使用category来添加一个新的属性叫`emailAddress`.
也许你在其他的程序中使用了`Person`,有时候拥有一个电子邮件地址是有意义的,但是有时候却不是.所以当你不需要它的时候一个category能很好的避开这个麻烦.或者也许你不用拥有这个`Person`类,并且维护人员也不需要为你添加属性.不管哪种情况,你要如何破解这些问题?首先,下面是基础的`Person`类:

{% highlight C++ %}
   @interface Person : NSObject
   @property (nonatomic, readwrite, copy) NSString *name;
   @end
   @implementation Person
   @end
{% endhighlight %}

现在你可以添加一个新的属性,`emailAddress`,在category里面使用
associative reference(关联引用):

{% highlight C++ %}
 #import <objc/runtime.h>
   
@interface Person (EmailAddress)
@property (nonatomic, readwrite, copy) NSString *emailAddress;
@end

@implementation Person (EmailAddress)

static char emailAddressKey;

- (NSString *)emailAddress {
 return objc_getAssociatedObject(self, &emailAddressKey);
}

- (void)setEmailAddress:(NSString *)emailAddress {
 objc_setAssociatedObject(self, &emailAddressKey,emailAddress,OBJC_ASSOCIATION_COPY);
}
@end
{% endhighlight %}
注意,associative references是基于key的地址而不是他的值.不要在意存储在`emailAddressKey`里的是什么.它只须要是唯一的,不变的地址.这就是为什么一般会使用一个未定义的`static char`来做key.

Associative references 有着很好的内存管理,可以正确的处理由`objc_setAssociatedObject`传入的copy, assign, or retain等内存类型参数.当相关联的对象被销毁时,关联引用会被释放.
事实上这就意味着,你可以使用关联对象来追踪另外一个对象的生命周期.
例如:

{% highlight C++ %}
const char kWatcherKey;

@interface Watcher : NSObject
@end

#import <objc/runtime.h>
@implementation Watcher
- (void)dealloc {
	NSLog(@"HEY! The thing I was watching is going away!");
}
@end

NSObject *something = [NSObject new]; 

objc_setAssociatedObject(something, &kWatcherKey, [Watcher new],OBJC_ASSOCIATION_RETAIN);                         
{% endhighlight %}

嘿嘿嘿,上面的技术对于debugging来说是非常有用的,但是也可以用于非debug类的普通任务,比如执行清理.

使用associative references是一个非常好的办法来附加一个有意义的对象到alert平板或者控制器上.例如你可以附加一个"描述对象"到一个alert平板,就像下面的代码(部分):

{% highlight C++ %}
ViewController.m (AssocRef)

id interestingObject = ...;

UIAlertView *alert = [[UIAlertView alloc]
                     initWithTitle:@"Alert" message:nil
                     delegate:self
                     cancelButtonTitle:@"OK"
                     otherButtonTitles:nil];
￼
objc_setAssociatedObject(alert, &kRepresentedObject,
                         interestingObject,
[alert show];
{% endhighlight %}

现在,当alert被dismissed的时候,你可以找到你所在意的:
{% highlight C++ %}
- (void)alertView:(UIAlertView *)alertView
   clickedButtonAtIndex:(NSInteger)buttonIndex {
UIButton *sender = objc_getAssociatedObject(alertView, &kRepresentedObject);
     self.buttonLabel.text = [[sender titleLabel] text];
   }
{% endhighlight %}

很多程序处理这种任务都是用调用者内的一个实例对象,但是associative references相对来说更干净,更简单一点.对于熟悉Mac开发的人来说,这些代码有点像`representedObject`,但是相对会更灵活一点.

associative references的一个限制就是它们无法与`encodeWithCoder:`集合.所以它们很难通过一个category序列化.