---
layout: post
title: "JSONModel在64位环境下解析为NSUInterge属性的异常问题"
date: 2014-04-04 08:23:50 +0800
comments: true
categories: 
---


在连续的加班中,人已经有一点吃不消了.别人都是996...我们加起班都是9-11-7...而且经常被催几个通宵完成的页面和功能,又突然来一句设计大改,全部重新调整. 
不说了,说多了都是泪啊.

今天早上在测试各种环境运行状态的时候,在64bit模拟器下突然崩了(因为公司没有买任何设备,只有自己买的苹果本跟一个iPhone5,所以只能将就用模拟器测试了).

抛出异常的位置:

![](/images/20140404/1.png)

继续执行后控制台的Reason输出:

`*** Terminating app due to uncaught exception 'JSONModelProperty type not allowed', reason: 'Property type of XXModel.XXXXX is not supported by JSONModel.'
`

奇怪的是打印出来的XXXX这个属性是我自己后来添加到model类的一个index标识(NSUInterge类型)...应该没有问题吧,我以前也添加过json数据以外的属性来扩充model.没有出现过问题....这个Reason异常估计粗问题了(原谅我这个新手不知道为森么)~~~
抱着不作就不会死的原则...我修改自定义的那个property 类型..从nsstring一直改到了nsobjec...最后MB我都注释掉了!他还是抛出异常了..这次终于打印出来罪魁祸首.

>其实在最初debug的时候我是直接在异常上面的源码断点调试的,然后直接查到了是哪个属性执行后异常.这里不断修改出错的那个属性,是写本文时的另外新鲜尝试.

就是上面NSUInteger类型的变量...氮素,这个在32bit下没点问题啊.为什么到了64bit就不行了呢..
来阅读下jsonmodel的源代码也是一个不错的学习机会.
`propertyType = valueTransformer.primitivesNames[propertyType];`
这里的propertyType是nil.也就是说他没有取到值,所以在下面直接抛出异常了.(对象类型在更上面的几行代码中会被执行完成).

propertyType是一个NSString.所以上面这行代码是从一个字典取值.我进入这个字典一探究竟.

{% highlight C++ %}
@interface JSONValueTransformer : NSObject

@property (strong, nonatomic, readonly) NSDictionary* primitivesNames;
{% endhighlight %}

果然是一个字典属性,再进入`JSONValueTransformer.m`看看他的初始化

![](/images/20140404/2.png)

恩..好像没看到NSUInteger...看了几遍..大概晓得了,没有NSUInteger.
氮素NSUInteger的key是什么?我也不知道,自己写几行小程序看看吧..

![](/images/20140404/3.png)

在64bit simulator下看到了NSUInteger的key是"Q".(在32bit下是"I").
看下NSUInteger的定义:

{% highlight C++ %}
#if __LP64__ || (TARGET_OS_EMBEDDED && !TARGET_OS_IPHONE) || TARGET_OS_WIN32 || NS_BUILD_32_LIKE_64
typedef long NSInteger;
typedef unsigned long NSUInteger;
#else
typedef int NSInteger;
typedef unsigned int NSUInteger;
#endif
{% endhighlight %}

64位下的其实就是unsigned long了.那么我们继续回到`JSONValueTransformer.m`中,在初始化的时候加上`@"Q":@"NSUInteger"`. 结果如图所示:

![](/images/20140404/4.png)

然后继续64bit模拟器下测试....没有抛出异常了~~

因为时间仓促的原因,很多细节没有细扣,如有说的不对的地方,欢迎各路大神拍砖.(我身上已经隐隐渗出冷汗了)

另外jsonmodel没有实现NSCoding协议.本来自己想写一个自动NSCoding的,后来发现网上已经有现成的轮子了.我就不重复造了.
[https://github.com/nicklockwood/AutoCoding](https://github.com/nicklockwood/AutoCoding)

在你项目使用继承于jsonmodel的BaseModel.m实现中,import下就行了.
