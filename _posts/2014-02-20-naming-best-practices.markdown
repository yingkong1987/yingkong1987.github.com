---
layout: post
title: "iOS命名的最佳实践"
date: 2014-02-20 00:11:38 +0800
comments: true
categories: 
---

贯穿整个iOS,命名约定始终都是非常重要的.在下面的部分你可以学习到如何正确的命名各种各样的名目,并且知道为什么要这样命名的原因.

<!-- more -->

####自动变量

Cocoa 是一个动态类型化的语言,而且你很容易就会被到底工作在什么类型中困惑.一些集合(数组,字典等等)和Cocoa没有类型联系,所以非常容易敲出一些会出意外的代码,比如下面:

{% highlight C++ %}
 NSArray *dates = @[@”1/1/2000”];
 NSDate *firstDate = [dates firstObject];
{% endhighlight %}
  
这段代码编译的时候不会有警告,但是当你试着使用`firstDate`,将会很有可能因为未知选择器异常而崩溃.这个错误是因为一个字符串的数组居然被叫做`dates`...这个数组应该被叫做`dateStrings`,或者将其内的对象更换成`NSDate`对象.这种谨慎的命名可以节省很多的麻烦。



####方法

方法的名称应该明确他们接受和返回类型。例如,该方法的名称就是混乱的:

{% highlight C++ %}
- (void)add;
{% endhighlight %}

看起来好像`add`应该带一个参数,但是它没有....难道他是添加一些默认的对象吗?

命名像这些,就非常清楚明白了:

{% highlight C++ %}
 - (void)addEmptyRecord;
 - (void)addRecord:(Record *)record;
{% endhighlight %}

现在可以清楚的看到` addRecord:`方法接受一个`Record`对象.假如有出现混乱的可能,那么命名就得匹配好对象的类型,下面的例子就是展示了一个常见的错误:

{% highlight C++ %}
- (void)setURL:(NSString *)URL; // Incorrect
{% endhighlight %}
这是错误的,因为所谓的`setURL:`方法应该接受一个`NSURL`类型的参数,而不是`NSString`.假如你需要一个string,你需要添加一些提示来让方法更明确:

{% highlight C++ %}
 - (void)setURLString:(NSString *)string;
 - (void)setURL:(NSURL *)URL;
{% endhighlight %}
这条不要过度使用.假如这个类型是一目了然显而易见的,就不要添加类型信息到这个变量.例如,有一个属性叫做`name`要好过一个叫`nameString`的,---只要没有一个 `Name`的类在你的系统中造成阅读混乱....

方法同样有特定的命名规则关于内存管理和键值编码(KVC).虽然有自动引用计数(ARC)使得这些规则不那么重要了,不正确的命名方法导致在ARC和MRC(包括苹果框架中也有一部分MRC)交互时产生一些非常具有挑战性的bug.

方法的命名应该遵循驼峰式的规则..并且以小写字母开头...

假如一个方法的命名是以`alloc, new, copy`, 或者 `mutableCopy, `开头的..那么调用者就对返回对象具有拥有权.(也就是说,它的引用计数+1,调用者需要自己去平衡).如果你的一个属性的命名像`newRecord`这样,那么这条规则会导致一些问题,对其重命名为`nextRecord`或其他的一些名字来避免.

方法以`get`开头的话,需要返回一个值.例如:

{% highlight C++ %}
- (void)getPerson:(Person **)person;
{% endhighlight %}

不要使用get前缀来做为属性存取器的一部分.`name`属性的getter方法应该是`-name`