---
layout: post
title: "避免block引用他的持有者产生循环引用"
date: 2014-01-22 23:46:21 +0800
comments: true
categories: 
---

<!-- more -->


###  避免block引用他的持有者产生循环引用

   假如你没有非常细致认真的思考, Blocks 会很容易就产生循环引用.
   例如,下面的类提供了一个接口来下载某一个URL. 在完成请求后会,会调用一个完成处理的回调block.为了让这个完成处理程序在请求完成后依然有效,它需要被存储在一个实例变量中.

{% highlight C++ %}

// EOCNetworkFetcher.h

#import <Foundation/Foundation.h>

typedef void(^EOCNetworkFetcherCompletionHandler)(NSData *data);

@interface EOCNetworkFetcher : NSObject

@property (nonatomic, strong, readonly) NSURL *url;

- (id)initWithURL:(NSURL*)url;

- (void)startWithCompletionHandler:(EOCNetworkFetcherCompletionHandler)completion;

@end
	
{% endhighlight %}


{% highlight C++ %}
// EOCNetworkFetcher.m
#import "EOCNetworkFetcher.h"

@interface EOCNetworkFetcher ()
@property (nonatomic, strong, readwrite) NSURL *url;
@property (nonatomic, copy) EOCNetworkFetcherCompletionHandler completionHandler;
@property (nonatomic, strong) NSData *downloadedData;
@end


@implementation EOCNetworkFetcher

- (id)initWithURL:(NSURL*)url
{
		if ((self = [super init])) {
				_url = url;
		}
		return self;
}

- (void)startWithCompletionHandler:(EOCNetworkFetcherCompletionHandler)completion
{
		self.completionHandler = completion;
		// Start the request
		// Request sets downloadedData property
		// When request is finished, p_requestCompleted is called
}

- (void)p_requestCompleted {
		if (_completionHandler) {
				_completionHandler(_downloadedData);
		}
}

@end
{% endhighlight %}

另外一个类会创建一个这些 network fetcher 的对象,并且用这个对象来下载URL上的数据. 如下面:

{% highlight C++ %}
@implementation EOCClass
{
    EOCNetworkFetcher* _networkFetcher;
    NSData* _fetchedData;
}

- (void)downloadData
{
    NSURL* url = [[NSURL alloc] initWithString:@"http://www.example.com/something.dat"];
		
    _networkFetcher = [[EOCNetworkFetcher alloc] initWithURL:url];

		[_networkFetcher startWithCompletionHandler:^(NSData *data){

				NSLog(@"Request URL %@ finished", _networkFetcher.url);
				_fetchedData = data;

		}];
}
@end
{% endhighlight %}

这段代码看起来很正常,但是你可能没有意识到一个循环引用(retain cycle)已经出现.
事情是这样的:完成事件处理程序block引用了self变量. 因为self包含_fetchedData实例变量.
这个意味着EOCClass实例创建的这个network fetcher对象已经被block retain了. 这个block又被network fetcher对象 retain.
这样network fetcher 就轮流被block 和 EOCClass的实例 retain了.因为这个network fetcher对象是一个strong属性的实例变量.
下图就阐明了什么是循环引用.

![循环引用在network fetcher和class之间互相持有](/images/20140123/1.png)

这个_networkFetcher实例变量和completionHandler互相引用所产生的循环引用可以很简单的被修复.
这个完成回调如果改成如下的样子:

{% highlight C++ %}
[_networkFetcher startWithCompletionHandler:^(NSData *data){
 NSLog(@"Request for URL %@ finished", _networkFetcher.url);
 _fetchedData = data;
 _networkFetcher = nil;
}
{% endhighlight %}

在编程接口中使用完成回调Block,产生循环引用问题是很普遍的.因此更加需要去理解Block. 通常,可以在恰当时机清理掉引用的其中一个可以解决这个问题.
但是循环引用并不保证总是发生,在示例中,循环引用只有在完成处理程序运行才会被打破.假如这个完成处理程序没有运行,那么循环引用将永不会被打破.造成内存泄露..
另外一个可能潜在的循环引用是发生在完成处理程序block结束时引用并持有了这个对象.例如,继续扩展之前的示例,当network fetcher运行时,代替用户来持有对它的引用.原理是要保持他自己的生命周期.
这个network fetcher对象可以将自己添加到一个全局集合中,比如在开始的时候添加到集合中,在完成后从集合中删除.
用户可以修改代码如下:
{% highlight C++ %}
- (void)downloadData {
 NSURL *url = [[NSURL alloc] initWithString:
 @"http://www.example.com/something.dat"];
 EOCNetworkFetcher *networkFetcher =
 [[EOCNetworkFetcher alloc] initWithURL:url];
 [networkFetcher startWithCompletionHandler:^(NSData *data){
 NSLog(@"Request URL %@ finished", networkFetcher.url);
 _fetchedData = data;
 }];
}
{% endhighlight %}

大部分的网络库都使用的这种方法,因为自己来持有fetcher对象是让人厌烦的,例如Twitter框架中的TWRequest对象. 然而EOCNetworkFetcher的代码中残留了一个循环引用.事实上是这个完成处理block引用了请求本身.这个block因此retain了fetcher.反过来fetcher又通过completionHandler属性retain了block.
幸运的是修复起来很简单.回想一下这个完成处理程序仅被保存在一个属性中,这样就可以以便在将来使用.问题是一旦完成处理程序被运行,它就不在需要被block持有了.所以,简单的修复办法就如下面的修改:
{% highlight C++ %}
- (void)p_requestCompleted {
 if (_completionHandler) {
 _completionHandler(_downloadedData);
 }
 self
{% endhighlight %}

一旦请求完成循环引用就会被打破,并且这个fetcher 对象必要时会释放.
注意,这是一个好理由来通过完成处理程序在开始方法中.假如用暴露一个公开属性来代替这个完成处理程序,你不可能只在请求完成时去清理.
在这种情况下明显打破循环引用的唯一办法是在在处理内强制执行清理这个completionHandler属性.
但是这样不是很明智,因为你不能假设用户(使用网络库的开发者)会这样做而不是在责怪这个网络库有内存泄露.

这两个场景并不少见,都很容易在使用block的时候悄悄产生bug.同样也能简单的避免,假如你比较注意这些方面.关键是要考虑这个对象会不会被block捕获(使用)而产生retain.
如果有任何一个可以retain block的对象,不管直接还是间接,你都需要好好思考怎么在合适的时机来打破循环引用.


###需要记住的事情

	1.意识到block 捕获的对象又直接或者间接的retain了这个block会造成潜在的循环引用问题
	2.确保在最适当的时候打破循环引用，但不要将打破循环的责任交给你API的调用者。