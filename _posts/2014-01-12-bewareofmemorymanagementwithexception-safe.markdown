---
layout: post
title: "异常安全代码的内存管理需谨慎!"
date: 2014-01-12 02:35:06 +0800
comments: true
categories: 
---

<!-- more -->

异常(Exceptions)是许多现代语言所提供的语言特性. 
异常在纯C中不存在,但是存在于Objective-C和C++中.事实上,在现代runtime,C++ 和 Objective-C的异常是兼容的,意思就是说,其中一个语言抛出的异常可以被另外一种语言用处理程序捕捉到.

就内存管理而言.异常引入了一个有趣的问题.  在@try的块里面,假如有一个对象被持有(强引用),并且在这个对象被释放之前就抛出了异常,那么这个对象会产生内存泄露(leak). 除非你在@catch块中对其进行了处理.  一般情况在Objective-C的异常处理中运行C++析构函数.这对于C++来说是非常重要的.因为一个异常抛出并需要销毁的时候,任何对象的生命周期都被缩短了.否则,他所使用的内存就会发生泄露,更不用说其他所有的系统资源,例如 文件句柄就可能没有被适当的清理.

常规异常处理中用MRC环境来完成对象的销毁是非常棘手的. 对下面使用MRC的 Objective-C 代码仔细思考:

{% highlight C++ %}
@try {
 EOCSomeClass *object = [[EOCSomeClass alloc] init];
 [object doSomethingThatMayThrow];
 [object release];
}
@catch (...) {
 NSLog(@"Whoops, there was an error. Oh well...");
}
{% endhighlight %}

乍一看,这似乎是正确的. 但是假如doSomethingThatMayThrow碰巧抛出一个异常呢?
在下面的release方法将不会被运行,因为这个异常将停止继续执行并且跳入@catch块内.
所以在这种情况下,这个对象将在抛出异常的时候发生泄漏.这当然不是理想的.解决的办法是使用@finally块.这个块保证会运行,并且不管是否抛出异常仅且运行一次.例如刚才的代码可以写成这样:

{% highlight C++ %}
EOCSomeClass *object;
@try {
 object = [[EOCSomeClass alloc] init];
 [object doSomethingThatMayThrow];}
@catch (...) {
 NSLog(@"Whoops, there was an error. Oh well...");
}
@finally {
 [object release];
}
{% endhighlight %}

注意,对象必须要在 @try外面声明,因为它还将需要在@finally块中引用. 如果你对所有需要释放的对象这样操作,那么这可是非常乏味的.同样,如果在@try块中有多个声明使得逻辑太过复杂,那么这可能就容易疏忽而产生内存泄露.如果泄露的对象是非常重要的(或者负责管理的),比如文件描述符 或者 数据库链接,那么这个泄露可能就是灾难性的.因为程序到最后不必要的保留了系统的所有资源而被结束..

在ARC中,情况会更为严重.最开始的那段代码在ARC的情况中如下:

{% highlight C++ %}
@try {
 EOCSomeClass *object = [[EOCSomeClass alloc] init];
 [object doSomethingThatMayThrow];
}
@catch (...) {
 NSLog(@"Whoops, there was an error. Oh well...");
}
{% endhighlight %}


现在有了更大的问题.你不能使用 在@finally 块中执行release的设计了. 因为ARC中无法调用release方法.但是,你大概会想,ARC当然能自己处理这种情况. 不过,在默认情况,它没有处理. 假如需要一个对象,在抛出异常时进行清理的话,那么就需要添加大量的用例代码来跟踪对象, 这些代码在没有抛出异常的运行时会严重的降低性能. 也会增加程序的体积. 出现了这些副作用都不是理想的.

虽然默认不是打开,但是ARC还是支持添加这些额外代码来处理异常安全.打开这个功能的代码就是使用编译器标签-fobjc-arc-exception. 在Objective-C编程中,默认不把它打开来背后的原因是.异常处理仅仅被用来处理一个能导致应用程序结束的异常.因此,如果应用程序终止了,潜在的内存泄漏是无关紧要的.假如程序被终结,那么这些为了使得异常安全而必须添加的代码将是毫无意义的.

假如编译器是Objective-C++类型,-fobjc-arc-exceptions标记又是默认打开的话.因为C++已经有类似ARC将实现异常安全功能的代码,所以这对性能的影响并不大.并且,C++大量的使用了异常,所以很可能有的开发者使用Objective-C++ 来操作异常.


如果你使用的MRC又必须捕获异常,记得确保代码编写正确的方式来清理. 如果你正在使用ARC又必须捕获异常,你将需要打开-fobjc-arc-exceptions标记.但是最重要的是,如果你发现你捕获了大量的异常,那么最好通过使用NSError类型error来重构,代替异常.

<h3>记住下面两点</h3>
<ol>
<li><strong>当捕获到异常,应该注意确保@try中创建的对象被清理完成.<strong></li>
<li><strong>ARC在默认情况下不会清理抛出异常时的代码,但是可以通过打开一个编译器标记来完成.不过会产生大量的代码和运行时的成本.<strong></li>
</ol>