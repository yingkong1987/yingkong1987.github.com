---
layout: post
title: "NSURLComponents CFStringTransform"
date: 2014-02-24 23:50:06 +0800
comments: true
categories:
---

有时候苹果会悄悄的添加一些有趣的类.在iOS7中,苹果添加了`NSURLComponents`,但是没有一个类参考文档.在“What’s New in iOS 7”中的iOS 7 release notes(iOS7发行说明)部分中有提到,但是你还是要去阅读` NSURL.h`的文档.

<!-- more -->

`NSURLComponents`很容易把URL切割成一部分.例如:

{% highlight C++ %}
NSString *URLString =
     @"http://en.wikipedia.org/wiki/Special:Search?search=ios";
   NSURLComponents *components = [NSURLComponents
     componentsWithString:URLString];
NSString *host = components.host;
{% endhighlight %}

你依然可以使用`NSURLComponents`来构造或者修改URL:
{% highlight C++ %}
   components.host = @"es.wikipedia.org";
   NSURL *esURL = [components URL];
{% endhighlight %}

在iOS7中,`NSURL.h `添加了几个有效的处理URL的category.例如,你可以使用`[NSCharacterSet URLPathAllowedCharacterSet]`来取得一个URL合法字符的集合.`NSURL.h `也添加了`[NSString stringByAddingPercentEncodingWithAllowedCharacters:]`,让你控制字符字符的百分比编码.之前要做到这个功能只能使用 Core Foundation中的`CFURLCreateStringByReplacingPercentEscapes`.

在`NSURL.h`中搜索`7_0`(快捷键command+F),找出所有新方法和它们的文档.


###CFStringTransform
`CFStringTransform`是一个函数----当你发现它后,你自己都无法相信你之前居然不知道它.他可以从很多方面来翻译字符串,比如简化正常,索引和搜索.例如,他可以使用选项`kCFStringTransformStripCombiningMarks`来删除重音符号.

{% highlight C++ %}
CFMutableStringRef string = CFStringCreateMutableCopy(NULL, 0, CFSTR("Schläger"));
   CFStringTransform(string, NULL, kCFStringTransformStripCombiningMarks,
     false);
... => string is now “Schlager” CFRelease(string);
{% endhighlight %}

`CFStringTransform`对待非拉丁书写系统更加强大,例如阿拉伯语或者中文.它可以转化许多书写系统为拉丁脚本,让标准化更简单.例如,你可以转化中文脚本为拉丁脚本,代码如下:

{% highlight C++ %}
CFMutableStringRef string = CFStringCreateMutableCopy(NULL, 0, CFSTR("你好"));
CFStringTransform(string, NULL, kCFStringTransformToLatin, false);
... => string is now “nˇı hˇao”
CFStringTransform(string, NULL, kCFStringTransformStripCombiningMarks,
false);
... => string is now “ni hao” CFRelease(string);
{% endhighlight %}

注意选项只是`kCFStringTransformToLatin`.源语言不是必须的.你几乎可以把任何字符串先转换成这个而不需要知道是什么语言.`CFStringTransform`也可以把拉丁文字转换成其他的书写系统,比如阿拉伯语,韩语,希伯来语和泰语.



#####中文字符在日本
中文字符总是音译成普通话,即使他们被使用在另外一个书写系统.不过对于日本可能特别棘手,因为汉字可能被包括在日本的字符串.例如一个三个字的日文,如:`白い月`..他的第一个字和最后一个字被音译成中国的普通话(bái 和 yuè),中间的字被音译为日语(i).于是就创建了一个荒诞的字符串:bái i yuè.

尽管`CFStringTransform`可以处理平假名和片假名,但是他不能处理日文汉字.假如你需要音译复杂的日文文本,看`"NSString-Japanese"`,由00StevenG在[https://github.com/00StevenG/NSString-Japanese](https://github.com/00StevenG/NSString-Japanese)上发表.它可以处理日本汉字注音和平假名和片假名。

`NSString-Japanese`是基于`CFStringTokenizer`的.提供了很多的语言意识转换,但是也增加了使用的复杂度.


更多的信息,可以看 CFMutableString Reference和CFStringTokenizer Reference的相关内容.
