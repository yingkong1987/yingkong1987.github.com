---
layout: post
title: "在APi设计中的 Delegate VS Block"
date: 2014-01-24 20:03:47 +0800
comments: true
categories: 
---

<!-- more -->

本文翻译自:[http://wangling.me/2014/01/delegate-vs-block-in-api-design.html](http://wangling.me/2014/01/delegate-vs-block-in-api-design.html)

当被问到在API的设计中`delegate`或`block`哪一个更好的时候,我没有足够的时间来彻底思考这个问题,所以我头顶马上就会冒出一个答案"无论什么情况下delegate能工作的话,block也同样适用".

我知道这并不是一个好的答案,所以我一直在思考着.现在,我有了一个更明确的选择并且更系统的论证了我的选择.
分别为下面两种情况


####1. 一次性回调(One-Off Callbacks)

例如:`[NSArray enumerateObjectsUsingBlock:]`.

用block就非常的方便.block的调用者负责在使用完后处理释放.这就意味着,即使假如这里出现了block提供方和block之间的强引用循环,这个循环也是临时的,我等码农不需要在block内去写更复杂的代码来使用弱引用去解决这个循环.

####2. 终生回调(Lifetime Callbacks)

例如:`UITableViewDataSource` & `UITableViewDelegate`.

Delegate 是更加友好的.block在这里的问题就是循环引用是比较常见的并且没有人可以帮你打破这个循环.APi的使用者就被迫在一开始的地方或者结尾的地方写更多笨拙的代码来解决这个循环问题.虽然优秀的代码可以处理这种情况,但是作为一个API设计者,我不应该这样要求我的使用者.我应该让他们尽可能的避免伤脑筋,并且给他们尽可能少的机会来滥用我的API.

选择使用deleagte而不是block的另一个原因是更易于减少错误的出现。生命周期的回调函数数量通常很大。相比使用大量block，组织delegate的回调API和创建一个简单地delegate会更加简洁

>虽然`无论什么情况下delegate能做的工作,block也同样适用`这句话是对的,但是有时候delegate仍然是API设计中的更好选择.