---
layout: post
title: "关于使用 NSNotification, NSOperationQueue and GCD 的百慕大三角"
date: 2014-01-24 21:18:26 +0800
comments: true
categories: 
---

<!-- more -->

本文翻译自:[http://wangling.me/2013/11/surprising-corner-case-of-using-nsnotification-nsoperationqueue-and-gcd.html](http://wangling.me/2013/11/surprising-corner-case-of-using-nsnotification-nsoperationqueue-and-gcd.html)

先不要运行下面的代码,告诉我哪一个会先执行,block1 还是block2?

{% highlight C++ %}
// Dispatch from main thread.
dispatch_async(_backgroundQueue, ^{
    [[NSOperationQueue mainQueue] addOperationWithBlock:^{
        // block 1
        NSLog(@"First Operation");
    }];

    __block id observer = [[NSNotificationCenter defaultCenter] addObserverForName:@"MyNotif" object:nil queue:[NSOperationQueue mainQueue] usingBlock:^(NSNotification *note) {
        // block 2
        NSLog(@"Receive Notif");
        [[NSNotificationCenter defaultCenter] removeObserver:observer];
    }];

    [[NSNotificationCenter defaultCenter] postNotificationName:@"MyNotif" object:self];
});

{% endhighlight %}



同样不运行代码的情况下,告诉我哪一个会先执行,block1 还是block2?
假如你没有注意到区别,我直接告诉你,区别就一点:这次我使用`dispatch_sync`来代替`dispatch_async`.

{% highlight C++ %}
// Dispatch from main thread.
dispatch_sync(_backgroundQueue, ^{
    [[NSOperationQueue mainQueue] addOperationWithBlock:^{
        // block 1
        NSLog(@"First Operation");
    }];

    __block id observer = [[NSNotificationCenter defaultCenter] addObserverForName:@"MyNotif" object:nil queue:[NSOperationQueue mainQueue] usingBlock:^(NSNotification *note) {
        // block 2
        NSLog(@"Receive Notif");
        [[NSNotificationCenter defaultCenter] removeObserver:observer];
    }];

    [[NSNotificationCenter defaultCenter] postNotificationName:@"MyNotif" object:self];
});
{% endhighlight %}

假如你回答错误也完全OK,因为第二次情况是非常难以捉摸的.这到底揭露了什么呢?我插入了几个`NSLog`到第二种情况的代码中.这次我们一起来运行下代码,看看到底是怎么回事.

{% highlight C++ %}
// Dispatch from main thread.
dispatch_sync(_backgroundQueue, ^{
    NSLog(@"%@", [NSThread isMainThread] ? @"In Main Thread" : @"In Background Thread");
    [[NSOperationQueue mainQueue] addOperationWithBlock:^{
        // block 1
        NSLog(@"First Operation");
    }];

    __block id observer = [[NSNotificationCenter defaultCenter] addObserverForName:@"MyNotif" object:nil queue:[NSOperationQueue mainQueue] usingBlock:^(NSNotification *note) {
        // block 2
        NSLog(@"Operations in Queue: %@", [[NSOperationQueue currentQueue] operations]);
        NSLog(@"Receive Notif");
        [[NSNotificationCenter defaultCenter] removeObserver:observer];
    }];

    [[NSNotificationCenter defaultCenter] postNotificationName:@"MyNotif" object:self];
});

{% endhighlight %}

如果你依然不太了解,我给你看下相关的文档:

```
dispatch_sync 
As an optimization, this function invokes the block on the current thread when possible.

addObserverForName:object:queue:usingBlock: 
queue 
The operation queue to which block should be added. 
If you pass nil, the block is run synchronously on the posting thread.
```

OK.如果关于`queue`参数的解释不太明了的话,我诠释为:
```
queue 
The operation queue to which block should be added. 
If you pass nil, the target thread to run the block is the posting thread. If the target thread is the posting thread the block is run synchronously without being enqueued.
```
