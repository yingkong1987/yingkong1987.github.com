---
layout: post
title: "重构的技巧"
date: 2014-05-25 21:00:42 +0800
comments: true
categories: 
---

本人翻译自:["http://www.merowing.info/2014/03/refactoring-tricks/#.U4Hn2Ba-zW0"](http://www.merowing.info/2014/03/refactoring-tricks/#.U4Hn2Ba-zW0).

我想一条童子军的军规:“始终保持露营地比你发现它的时候还要干净”。如果你在地上发现了一点脏东西，不管是谁弄的，都清理掉它。要为了下一拨来露营的人改善环境。（实际上，那条规矩的早期版本，出自Robert Stephenson Smyth Bden-Powell，童子军活动之父，说的是“努力使世界比你发现它时变得更好”。）
这就是为什么我不断的重构我的代码让它干净整洁.
当谈到编程的质量时,代码的可读性是我最关注的部分. 我想分享一些大部分人不知道的我用来简化代码的重构技巧.

####字典的映射
假如你有一个switch语句来分配一些值:

{% highlight C++ %}
switch(condition) {
          case value1:
              result = @"valueFor1";
          break;
          case value2:
              result = @"valueFor2";
          break;
          case value3:
              result = @"valueFor3";
          break;
          case value4:
              result = @"valueFor4";
          break;
          case value5:
              result = @"valueFor5";
          break;
          default:
              result = @"valueForDefault";
          break;
      }
      return result;
{% endhighlight %}

记住这儿仅仅就 switch 5个值, 想象一下一对多. 让我们使用字典映射来简化这段代码:

{% highlight C++ %}
static NSDictionary *mapping = nil;
      if(!mapping) {
          mapping = @{
              @(value1) : @"valueFor1",
              @(value2) : @"valueFor2",
              @(value3) : @"valueFor3",
              @(value4) : @"valueFor4"
              @(value5) : @"valueFor5"
          };
      }
      
      return mapping[@value] ?: @"valueForDefault";
{% endhighlight %}

####Pro's(赞成的原因)
1.更方便阅读,即使在原来的switch里面有着更多的值也可以变得更容易阅读.
2.更快,映射仅仅构造一次,然后我们只需要快速的查找值.
3.更不易出错,因为这儿并不需要写`break`或者`return`.
4.基于代码映射的性质可以非常轻松的将这个映射转成某种静态数据,例如`JSON`,`PLIST`文件,.

####用block动态映射
关于更加复杂的switch,怎么样真正动态的做一些事情?我们可以用block来简化这段代码.

最近,我重构了一些代码,字符串格式化来针对不同的类型:

{% highlight C++ %}
if ([title isEqualToString:@"Scratches"])
  {
      title = [NSString stringWithFormat:(self.vehicle.numberOfScratches == 1 ? @"%d Scratch" : @"%d Scratches"), self.vehicle.numberOfScratches];
  }
     else if ([title isEqualToString:@"Dents"])
  {
      title = [NSString stringWithFormat:(self.vehicle.numberOfDents == 1 ? @"%d Dent" : @"%d Dents"), self.vehicle.numberOfDents];
  }
  else if ([title isEqualToString:@"Painted Panels"])
  {
      title = [NSString stringWithFormat:(self.vehicle.numberOfPaintedPanels == 1 ? @"%d Painted Panel" : @"%d Painted Panels"), self.vehicle.numberOfPaintedPanels];
  }
  else if ([title isEqualToString:@"Chips"])
  {
      title = [NSString stringWithFormat:(self.vehicle.numberOfChips == 1 ? @"%d Chip" : @"%d Chips"), self.vehicle.numberOfChips];
  }
  else if ([title isEqualToString:@"Tires"])
  {
      title = [NSString stringWithFormat:(self.vehicle.numberOfTires == 1 ? @"%d Tire" : @"%d Tires"), self.vehicle.numberOfTires];
  }
  else title = nil;
{% endhighlight %}

通过使用block映射,我们可以重构这段代码为下面这样:

{% highlight C++ %}
static NSDictionary *titleMapping = nil;
if (!titleMapping) {
 NSString *(^const format)(NSUInteger, NSString *, NSString *) = ^(NSUInteger value, NSString *singular, NSString *plural) {
    return [NSString stringWithFormat:@"%d %@", value, (value == 1 ? singular : plural)];
  };

  titleMapping = @{
    @"Scratches" : ^(MyClass *target) {
      return format([target numberOfScratches], @"Scratch", @"Scratches");
    },
    @"Dents" : ^(MyClass *target) {
      return format([target numberOfDents], @"Dent", @"Dents");
    },
    @"Painted Panels" : ^(MyClass *target) {
      return format([target numberOfPaintedPanels], @"Painted Panel", @"Painted Panels");
    },
    @"Chips" : ^(MyClass *target) {
      return format([target numberOfChips], @"Chip", @"Chips");
    },
    @"Tires" : ^(MyClass *target) {
      return format([target numberOfTires], @"Tire", @"Tires");
    }
  };
}

  NSString *(^getTitle)(MyClass *target) = titleMapping[title];
  return getTitle ? getTitle(self) : nil;

{% endhighlight %}

Pro’s

1.非常安全,因为没有办法弄错if-else链.
2.缓存了映射,因为我们使用的是静态的变量.
3.我们可以很容易的添加一些宏来使得代码更精简,并且因此更容易来扩展.

PS.我可以用字符串匹配来实现它,甚至使用的代码更少. 但是我不认为它能使可读性更好.


####更早使用return和取反if的判断条件来简化的流程
现在你大概可能发现我不太喜欢太多的if条件句并且我`讨厌`太长的if-esle链. 反而我比较喜欢让if条件语句尽可能的简单并且更早使用return来返回.

原始代码:

{% highlight C++ %}
if(!error) {
  //! success code
} else {
  //! failure code
}
{% endhighlight %} 

我比较喜欢的写法:

{% highlight C++ %}
if(error) {
  //! failure code
  return;
}

//! success code
{% endhighlight %}

Pro's
1.我不需要阅读更多的代码.假如我仅仅只是对error的情况感兴趣.
2.在大段代码的情况我不需要去记住所有的流程,因为我可以非常清楚看到前面的return/break.


####使用动态方法解决
有时,我们可能看到类似下面的情况:
{% highlight C++ %}
if([type isEqualToString:@"videoWidget"]) {
  [self parseVideoWidget:dictionary];
} else
if([type isEqualToString:@"imageWidget"]) {
  [self parseImageWidget:dictionary];
} else
if([type isEqualToString:@"textWidget"]) {
  [self parseTextWidget:dictionary];
} else
if([type isEqualToString:@"twitterWidget"]) {
  [self parseTwitterWidget:dictionary];
}
{% endhighlight %}

首先,我可以把上面讲的所有重构技巧应用到此,但是这段代码看起来主要等待解决的问题是将来的可扩展问题,让我们一起看看如何能够让它变得更好.

{% highlight C++ %}
SEL dynamicSelector = NSSelectorFromString([NSString stringWithFormat:@"parse%@:", type]);
if(![self respondsToSelector:dynamicSelector]) {
  DDLogWarning(@"Unsupported widget type %@", type);
  return;
}
[self performSelector:dynamicSelector withObject:dictionary];
{% endhighlight %}

Pro's
1.非常容易阅读并且理解
2.比if条件语句更安全
3.更容易扩展,例如我可以添加新的工具类型到分类中.当开发一个衍生APP的时候我甚至都不需要去接触它的基础类.


####结论
我时常重构我的代码,这儿有一些知道人不多的技巧,但是有助于让你的代码更简化.我通常在["AppCode"](http://www.jetbrains.com/objc/)上写我的代码.`AppCode`是一个很棒的IDE,它有着大量的关于重构的函数,我每天都在使用这些,点击链接下载.