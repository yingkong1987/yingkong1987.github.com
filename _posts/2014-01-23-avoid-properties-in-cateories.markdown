---
layout: post
title: "避免在类别(category)中定义属性(@property)"
date: 2014-01-23 20:31:47 +0800
comments: true
categories: 
---

<!-- more -->

property 是包装数据的一种办法.尽管技术上可以实现在category里面声明一个property,但是应该尽量避免这样做.理由是,除了class延续类别外,是不可能用一个category对class添加一个实例变量.
因此对于category同样也不可能合成一个实例变量去支持property.
我们来切割下本来是实现person的class.你可能需要一个关于友谊的category声明方法,来操作关于这个person朋友的列表.在不知道描述的问题前,你也可以把property的friends列表放到友谊category里面.
想这样:

{% highlight C++ %}
#import <Foundation/Foundation.h>
@interface EOCPerson : NSObject
@property (nonatomic, copy, readonly) NSString *firstName;
@property (nonatomic, copy, readonly) NSString *lastName;
- (id)initWithFirstName:(NSString*)firstName andLastName:(NSString*)lastName;
@end
{% endhighlight %}

{% highlight C++ %}
@implementation EOCPerson
// Methods
@end

@interface EOCPerson (Friendship)
@property (nonatomic, strong) NSArray *friends;
- (BOOL)isFriendsWith:(EOCPerson*)person;
@end

@implementation EOCPerson (Friendship)
// Methods
@end
{% endhighlight %}

假如你进行编译,可是你将会收到编译器警告后结束
	`warning: property 'friends' requires method 'friends' to be
	defined - use @dynamic or provide a method implementation in
	this category [-Wobjc-property-implementation]
	warning: property 'friends' requires method 'setFriends:' to be
	defined - use @dynamic or provide a method implementation in
	this category [-Wobjc-property-implementation]
	`
这个稍微有点含糊的警告意思是说不能用category合成一个实例变量,并且因此property需要有一个accessor方法来实现在category中.或者,可以声明accessor方法@dynamic,意味着你声明的变量将在运行时有效,但是编译时是看不到的.
这可能发生的情况是如果你使用消息转发机制来拦截这个方法并且在运行时提供实现.

为了避免category不能合成实例变量的问题,你可以使用关联的对象.例如,你可能需要去实现下面category内的关联:

{% highlight C++ %}
#import <objc/runtime.h>
static const char *kFriendsPropertyKey = "kFriendsPropertyKey";
@implementation EOCPerson (Friendship)

- (NSArray*)friends {
 return objc_getAssociatedObject(self, kFriendsPropertyKey);
}

- (void)setFriends:(NSArray*)friends {
 objc_setAssociatedObject(self,kFriendsPropertyKey, friends,OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}
@end
{% endhighlight %}

这样不是很完美的解决办法.是有大量的引用和容易出现内存管理错误.因为很容易就会忘记这个property是像这样实现的.例如,你可以用改变property的属性来修改内存管理语义.但是你同样需要去记得改变内存管理语义在setter内关联的对象.
尽管这是一个不错的解决方案,但是我不推荐.

同样,你可能希望这个实例变量可以支持这个friends数组是一个可变数组. 你可以带一个可变拷贝,但是这样又会是另外一种无尽的混乱开始进入你的代码基础.因此property在主要接口定义比在category里面定义要干净的多.

在这个示例中,正确的解决办法就是把所有的property定义放到主接口声明.所有的数据封装在一个主接口定义的类中,主接口是唯一可以声明实例变量(数据)的地方.因为他们仅仅是定义一个实例变量和相关语法糖访问器方法,property受到相同的规则.category应该被想到用做一个类的方法扩展来扩展功能,而不是封装数据.
也就是说,有些时候只读属性可以被成功的用在category中.例如,你可能想建立一个NSCalendar的category来返回一个包含月份字符串的数组.因为这个方法访问任何数据,并且这个property不支持一个实例变量,你可以实现这个category像这样:
{% highlight C++ %}
@interface NSCalendar (EOC_Additions)
@property (nonatomic, strong, readonly) NSArray *eoc_allMonths;
@end
@implementation NSCalendar (EOC_Additions)
- (NSArray*)eoc_allMonths {
 if ([self.calendarIdentifier
 isEqualToString:NSGregorianCalendar])
 {
 return @[@"January", @"February",
 @"March", @"April",
 @"May", @"June",
 @"July", @"August",
 @"September", @"October",
 @"November", @"December"];
 } else if ( /* other calendar identifiers */ ) {
 /* return months for other calendars */
 }}
@end
{% endhighlight %}

property所支持的实例变量将不会自动合成,因为所有所需的方法(这里只有一个只读方法)需要被实现.因此,编译器会发出警告.然而,即使在这种情况下,一般最好避免使用property. property的用意就在于依赖class对数据的支持.property是封装数据,在本例中,你在category内将替换声明方法为检索列表中的月份.
{% highlight C++ %}
@interface NSCalendar (EOC_Additions)
- (NSArray*)eoc_allMonths;
@end
{% endhighlight %}

需要记住的事情
	1.把所有封装数据的property声明都在住接口中定义.
	2.在category中希望使用访问器方法来声明property,除非他是一个类延续(class-continuation)category