---
layout: post
title: "You May Not Know"
date: 2014-02-26 23:40:47 +0800
comments: true
categories: 
---

###Base64 and Percent Encoding
Cocoa 一直以来都有需求去方便的访问Base64编码和解码. Base64也是很多web协议的标准并且在你需要存储任意数据到一个字符串时是非常有用的.

Percent编码也是一个重要的web协议.特别是URs.你现在可以用` [NSString stringByRemovingPercentEncoding]`解码percent-encoded(百分比编码)字符串.虽然一直以来可能是用` stringByAddingPercentEscapesUsingEncoding:`来做percent编码,但是iOS7添加了`stringByAddingPercentEncodingWithAllowedCharacters:`方法,它可以让你用来控制字符为 percent-encoded(百分比编码).

<!-- more -->

###-[NSArray firstObject]
这是一个很微小的变化,但是我又不得不提到它,因为这么长时间以来,我们一直在等待:经过多年的开发者实现他们自己的category来获取数组中的第一个对象,苹果终于添加了这个方法`firstObject`.就像`lastObject`,假如数组是空的,`firstObject`方法会返回一个`nil`而不是像用了`objectAtIndex:0`一样crash掉.(当数组中没有元素时.使用objectAtIndex:0取值会导致崩溃).

####摘要
Cocoa有着悠久的历史,传统和习俗. Cocoa也是一个不断发展,积极发展的框架.在这一章你所学到的一些最佳实践已经经过了几十年的Objective-C开发验证.你已经了解如何选择最好的名字去命名类.方法和变量.您还学习了如何最好的使用一些不太知名的功能比如关联引用,还有一些新特性比如`NSURLComponents`.即使对经验丰富的objective - c开发者,你也有希望能学到一些Cocoa的小技巧和一些以前不知道的事情.


####延伸阅读
#####苹果文档
下列文档可以在iOS开发者文库[ developer.apple.com]( developer.apple.com)或者通过Xcode文档和API参考中见到

>CFMutableString Reference
CFStringTokenizer Reference
Collections Programming Topics
Collections Programming Topics, “Pointer Function Options” Programming with Objective-C

#####其他资源
Thompson, Matt. NSHipster.
 Matt Thompson的博客是一个非常奇妙的周刊.会更新一些鲜为人知的Cocoa特性.
 [nshipster.com](nshipster.com)
 00StevenG.NSString-Japanese.假如你需要规范日语文本,这是一个非常有用的category来处理多个复杂的书写系统.
 [https://github.com/00StevenG/NSString-Japanese](https://github.com/00StevenG/NSString-Japanese)