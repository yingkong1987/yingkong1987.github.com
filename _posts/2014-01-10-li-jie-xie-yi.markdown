---
layout: post
title: "理解 NSCopying 协议"
date: 2014-01-10 01:58:22 +0800
comments: true
categories: 
---

<!-- more -->

如果你想复制一个对象.在 Objective-C中,一般是通过 copy 方法. 这个方法对于你自定义的对象,需要实现 NSCopying 协议包含的一个简单方法:

{% highlight C++ %}
(id)copyWithZone:(NSZone *)zone;
{% endhighlight %}
在很久很久以前..当时还在用NSZone申请内存区域,并在区域里面创建对象.而现在,每一个 app 都有一个区域:默认区域.所以即使你需要实现这个方法,你也不需要担心这个NSZone参数是什么东西了.
要支持复制的话,你需要做的就是实现NSCopying 协议和协议里面的一个方法.举个例子,设想一个类表示一个人.在接口定义中,你需要申明你的类实现NSCopying:

{% highlight C++ %}
#import <Foundation/Foundation.h>
@interface EOCPerson : NSObject <NSCopying>
@property (nonatomic, copy, readonly) NSString *firstName;@property (nonatomic, copy, readonly) NSString *lastName;
- (id)initWithFirstName:(NSString*)firstName
 andLastName:(NSString*)lastName;
@end
{% endhighlight %}

那么你将实现这个协议中所需的一个方法

{% highlight C++ %}
- (id)copyWithZone:(NSZone*)zone {
 EOCPerson *copy = [[[self class] allocWithZone:zone]
 initWithFirstName:_firstName
 andLastName:_lastName];
 return copy;
}
{% endhighlight %}

这个简单的例子通过所有的初始化工作都到指定的初始化程序.有时,你可能需要进一步的复制工作,像其他类的数据结构,没有在初始化方法中建立.比如,EOCPerson包含一系列用来操作与另外一个EOCPerson”建立好友”和”取消好友”的方法.在这种情况下,你会想copy这个好友数组(_friends).以下是如何工作的完整示例:

{% highlight C++ %}
#import <Foundation/Foundation.h>
@interface EOCPerson : NSObject <NSCopying>
@property (nonatomic, copy, readonly) NSString *firstName;
@property (nonatomic, copy, readonly) NSString *lastName;
- (id)initWithFirstName:(NSString*)firstName
 andLastName:(NSString*)lastName;
- (void)addFriend:(EOCPerson*)person;
- (void)removeFriend:(EOCPerson*)person;
@end


@implementation EOCPerson {
 NSMutableSet *_friends;
}
- (id)initWithFirstName:(NSString*)firstName
 andLastName:(NSString*)lastName {
 if ((self = [super init])) {
 _firstName = [firstName copy];
 _lastName = [lastName copy];
 _friends = [NSMutableSet new];
 }
 return self;}
- (void)addFriend:(EOCPerson*)person {
 [_friends addObject:person];
}
- (void)removeFriend:(EOCPerson*)person {
 [_friends removeObject:person];
}
- (id)copyWithZone:(NSZone*)zone {
 EOCPerson *copy = [[[self class] allocWithZone:zone]
 initWithFirstName:_firstName
 andLastName:_lastName];
 copy->_friends = [_friends mutableCopy];
 return copy;
}
@end
{% endhighlight %}

这次\_friends是一个内部实例方法 所以会先自己copy一份副本 然后再将值传出去.要注意 ->方法在这里被调用 是因为\_friends是一个内部实例变量 ,本来可以为它定义一个属性 但是既然它不会被外部使用 就不必这样做了.
对于这个例子提出了一个有趣的问题:为什么\_friends是一个实例变量的副本?你不能轻易的复制,并且每个对象将共享相同的可变集合.但是,这意味着假如一个朋友添加到原来的对象,那么他会神奇的和这个副本成为了朋友.这显然不是你想要的情节.但是如何集合是不可变的,你可以选择不创建副本,因为这个集合无论如何不能被修改. 这样将会在内存中存在两份完全相同的集合.
你一般应该使用指定的初始化程序来初始化副本,就像这个例子一样.但是你不想去指定初始化程序来复制的时候产生副作用,比如设置复杂的内部数据结构,你将会立即覆盖.
如果你回顾下copyWithZone:方法,你会发现朋友集合(_friends)是用mutableCopy方法复制的.这来自另外一个协议,叫做NSMutableCopying.他跟NSCopying类似,不过定义了下面的方法:

{% highlight C++ %}
- (id)mutableCopyWithZone:(NSZone*)zone
{% endhighlight %}
mutableCopy助手就像copy一样,用默认空间(zone)调用前面的方法.你需要实现NSMutableCopying,假如你的类中有可变和不可变的变体.当你使用这个模式的时候,你不能在你的可变类中覆写copyWithZone:并返回一个可变副本.相反,你应该用mutableCopy.类似的,如果你需要一个不可变的副本,你应该使用copy.
以下是NSArray和NSMutableArray的示例:

{% highlight C++ %}
-[NSMutableArray copy] => NSArray
-[NSArray mutableCopy] => NSMutableArray
{% endhighlight %}
注意,对一个可变对象调用copy会得到另外一个不可变的变体.这样做很容易在可变和不可变之间转换.换句话说,这可能实现的有三种方法: copy, immutableCopy, and mutableCopy, copy是总返回同样类型,但是其他的两种是返回指定变体.无论怎么样,当你不知道你的实例是否不可变,这是不好的情况. 在一些时候你可以决定去调用copy来返回一个NSArray,但是实际上它是一个NSMutableArray.这种情况下,你认为你返回的是一个不可变数组,但是它本身是一个可变的.
你可以好好想想(参考条目14)来确定你的实例是什么类型,但是这将会在创建副本的时候增加复杂度.所以，从安全方面考虑，最终只有两种方法，immutableCopy或者 mutableCopy.这就完全绕回到(前面的)两种方法，copy 和 mutableCopy.调用copy而不是immutableCopy的优点是 NSCopying的设计不仅适用于  带mutable和immutable变量的类；还适用于没有区别的场合。所以immutableCopy可能是个很差的命名方法.
你的复制方法的另外一个决定是执行深拷贝还是浅拷贝. 一个深拷贝副本的所有备份数据也是如此. 默认对Foundation中集合类的拷贝都是浅拷贝,意思就是说,仅仅只拷贝了容器本身的内存,而不是(拷贝)容器内存储的数据.这主要是因为,容器内的对象有可能是无法被复制的;此外,复制每一个对象通常也是不值得的.和浅拷贝的区别如图3.2所示:

!["图示3.2 [浅拷贝 与 深拷贝]"](/images/20140110_4.png)

浅拷贝的内容指向相同的对象.深拷贝的内容指向原内容的副本.
通常情况,在系统框架中使用copyWithZone:执行一个浅拷贝时,你会希望自己的类能遵循一个相同的模式.但是如果需要,可以添加一个执行深拷贝的方法.比如NSSet的情况下,提供的一个初始化方法

{% highlight C++ %}
- (id)initWithSet:(NSArray*)array copyItems:(BOOL)copyItems
{% endhighlight %}

假如copyItems设置成YES,那么这个数组里面的所有元素将调用copy方法来构造出一个新的集合返回.
在这个EOCPerson 类的例子中,这个集合内包含的”好友”是用copyWithZone:拷贝的,但如前所述,这并不是集合自己拷贝的这些元素.假如你需要深拷贝,你可以如下所示提供一个深拷贝方法

{% highlight C++ %}
- (id)deepCopy {
 EOCPerson *copy = [[[self class] alloc]
 initWithFirstName:_firstName
 andLastName:_lastName];
 copy->_friends = [[NSMutableSet alloc] initWithSet:_friends
 copyItems:YES];
 return copy;}
{% endhighlight %}

没有协议定义了深拷贝,所以留给了每一个类去自己定制深拷贝. 你仅仅只需要决定是否提供一个深拷贝方法.同样,你不应该假设一个遵循了NSCopying协议的对象将会执行一个深拷贝,绝对多数的时候,这只是一个浅拷贝.如果你需要一个深拷贝的对象,要么找到相关的方法,或假设你已经自己创建了一个,除非文档标明其实现的NSCopying已经提供了深拷贝方法.




<h3>需要记住</h3>
<ol>
<li><strong>假如你的对象需要被拷贝,那么要实现NSCopying协议.</strong></li>
<li><strong>假如你的对象有可变和不可变的差异,那么你需要都实现NSCopying 和 NSMutableCopying协议.</strong></li>
<li><strong>决定是需要深拷贝还是浅拷贝.一般情况的普通拷贝都是浅拷贝.</strong></li>
<li><strong>考虑添加一个深拷贝的方法,如果你的对象可能有这个需求.</strong></li>
</ol>