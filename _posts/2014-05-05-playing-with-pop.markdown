---
layout: post
title: "玩转Facebook开源的Pop库"
date: 2014-05-05 20:50:16 +0800
comments: true
categories: 
---

原文链接:["http://codeplease.io/playing-with-pop-i/"](http://codeplease.io/playing-with-pop-i/)


Facebook在向外界开源一些三方库上,一直都是做的非常好的. 最近在github发布的["Pop"](https://github.com/facebook/pop)库在不到24小时之内就已经收到了3.5k个stars.

引用Facebook的原话:

'pop是一个适用于iOS和OSX上的可扩展的动画引擎.除了基础的静态动画之外,还支持动态的弹性和衰减动画,可用于构建逼真的,基于物理的交互.它的API可以快速集成到现有的Objective-C代码库中,并且能够让任意对象的任何属性具有动画效果.它是一个成熟的并且经过良好测试的框架 在paper中驱动所有的动画和过渡效果.'

我用它创建一个["简单的示例"](https://www.dropbox.com/s/p63g7y8sduvc97h/Weee.m4v)开始.我的目的是想看一下它在给用户输入框加上阴影时,能达到什么样子的效果;它做的太棒了.我也想要获取第一手的体验,用一些东西快速去构建.这个特定的案例中使用的`POPSpringAnimation`代码,感觉与["我做过的其他东西"](https://github.com/RuiAAPeres/RPDynamicWarningView)很类似.

为了研究这个库,我的方法是直接进入他们的.h文件一探究竟(顺便提下,它们都是由objective-c++写的):

{% highlight C++ %}
POPBasicAnimation
POPDecayAnimation
POPPropertyAnimation
POPSpringAnimation
POPCustomAnimation
POPAnimation
POPAnimatableProperty
{% endhighlight %}
关于Pop有一些非常酷的东西,当你添加一些动画的时候,他的表现层和模型层是同步的. 这个 Sam Page(@sampage)写的 ["圆圈示例"](http://subjc.com/spark-camera/) 就使用了pop. 有点类似于 ["这个"](https://gist.github.com/RuiAAPeres/11402456). 由于`strokeEnd` 和 `strokeStart` 并不是["默认属性"](https://github.com/facebook/pop/blob/master/pop/POPAnimatableProperty.h#L90L141)的一部分,你需要创建你自己的自定义属性,比如这样:

{% highlight C++ %}
[POPAnimatableProperty propertyWithName:@"strokeStart" initializer:^(POPMutableAnimatableProperty *prop) {
        prop.readBlock = ^(id obj, CGFloat values[]) {
            values[0] = [obj strokeStart];
        };
        prop.writeBlock = ^(id obj, const CGFloat values[]) {
            [obj setStrokeStart:values[0]];
        };
    }];
{% endhighlight %}

必须的说POP库是非常强大的,正如之前所说:表示层和模型层是同步的!


[" Facebook's Paper Tech Talk "](https://www.youtube.com/watch?v=OiY1cheLpmI)里面Brian Amerige(@brianamerige)的话一直困扰在我的脑海里,特别是这一部分,他展示了如何让我们把手势融合进动画中(视频47:40部分).

所以直接从视频中看到:

![](/images/20140506/1.png)

基于手势的速率,可以旋转一个`UIView`

{% highlight C++ %}
- (void)rotate:(UIPanGestureRecognizer*)recognizer
{
    CGPoint velocity = [recognizer velocityInView:self.view];

    POPSpringAnimation *spring = [POPSpringAnimation animationWithPropertyNamed:kPOPLayerRotation];
    spring.velocity = [NSValue valueWithCGPoint:velocity];

    [_outletView.layer pop_addAnimation:spring forKey:@"rotationAnimation"];
}
{% endhighlight %}

现在,当你开始接触这些坏东西的时候,这一切又开始变得有趣了.
![](/images/20140506/2.png)

那么,如果我们同时对`position` 和 `size` 加上一些"dynamics",会发生什么?

![](/images/20140506/3.gif)

见代码:

{% highlight C++ %}
- (void)rotate:(UIPanGestureRecognizer*)recognizer
{
    CGPoint velocity = [recognizer velocityInView:self.view];

POPSpringAnimation *positionAnimation = [POPSpringAnimation animationWithPropertyNamed:kPOPLayerPosition];  
positionAnimation.velocity = [NSValue valueWithCGPoint:velocity];  
positionAnimation.dynamicsTension = 5;  
positionAnimation.dynamicsFriction = 5.0f;  
positionAnimation.springBounciness = 20.0f;  
[_outletView.layer pop_addAnimation:positionAnimation forKey:@"positionAnimation"];


POPSpringAnimation *sizeAnimation = [POPSpringAnimation animationWithPropertyNamed:kPOPLayerSize];  
sizeAnimation.velocity = [NSValue valueWithCGPoint:velocity];  
sizeAnimation.springBounciness = 1.0f;  
sizeAnimation.dynamicsFriction = 1.0f;  
[_outletView.layer pop_addAnimation:sizeAnimation forKey:@"sizeAnimation"];
{% endhighlight %}

在删除所有与dynamics相关的代码后:

![](/images/20140506/4.gif)

你依然可以看到一个轻微的弹力效果,但是这些是`POPSpringAnimation`的默认值起的效果.



至于POP,或者其他用于娱乐用途的三方库,扪心自问都涉及到一点:<strong>我现在应该使用它(这个library)来做吗?</strong>?
 Brian Lovin (@brian_lovin) 所写的["Design Details: Paper by Facebook(设计细节:Facebook的paper)"](http://blog.brianlovin.com/design-details-paper-by-facebook)给了你大量的资料去思考这个问题.
 下面来自于Brian Lovin的博客:

 ![](/images/20140506/5.gif)

 我想创建一些内容,允许我展示一个小的弹出窗口,并且有一些震动感.(好吧,是因为我喜欢这样的效果). 上图并不是我想要的效果(甚至差得很远).但是我从这里获得了灵感,所以:

![](/images/20140506/6.gif)

是的,我知道.没有什么奇特的,但是,可以使用POP很轻松实现这个效果.同样,就动画而言,没有能比POP带来"更甜美味道"的感觉了.

{% highlight C++ %}
- (void)hidePopup
{
    _isMenuOpen = NO;
    POPBasicAnimation *opacityAnimation = [POPBasicAnimation animationWithPropertyNamed:kPOPLayerOpacity];
    opacityAnimation.fromValue = @(1);
    opacityAnimation.toValue = @(0);
    [_popUp.layer pop_addAnimation:opacityAnimation forKey:@"opacityAnimation"];

    POPBasicAnimation *positionAnimation = [POPBasicAnimation animationWithPropertyNamed:kPOPLayerPosition];
    positionAnimation.fromValue = [NSValue valueWithCGPoint:VisiblePosition];
    positionAnimation.toValue = [NSValue valueWithCGPoint:HiddenPosition];
    [_popUp.layer pop_addAnimation:positionAnimation forKey:@"positionAnimation"];

    POPSpringAnimation *scaleAnimation = [POPSpringAnimation animationWithPropertyNamed:kPOPLayerScaleXY];

    scaleAnimation.fromValue  = [NSValue valueWithCGSize:CGSizeMake(1.0f, 1.0f)];
    scaleAnimation.toValue  = [NSValue valueWithCGSize:CGSizeMake(0.5f, 0.5f)];
    [_popUp.layer pop_addAnimation:scaleAnimation forKey:@"scaleAnimation"];
}
{% endhighlight %}

还有显示出效果的代码:

{% highlight C++ %}
- (void)showPopup
{
    _isMenuOpen = YES;

    POPBasicAnimation *opacityAnimation = [POPBasicAnimation animationWithPropertyNamed:kPOPLayerOpacity];
    opacityAnimation.fromValue = @(0);
    opacityAnimation.toValue = @(1);
    opacityAnimation.beginTime = CACurrentMediaTime() + 0.1;
    [_popUp.layer pop_addAnimation:opacityAnimation forKey:@"opacityAnimation"];

    POPBasicAnimation *positionAnimation = [POPBasicAnimation animationWithPropertyNamed:kPOPLayerPosition];
    positionAnimation.fromValue = [NSValue valueWithCGPoint:VisibleReadyPosition];
    positionAnimation.toValue = [NSValue valueWithCGPoint:VisiblePosition];
    [_popUp.layer pop_addAnimation:positionAnimation forKey:@"positionAnimation"];


    POPSpringAnimation *scaleAnimation = [POPSpringAnimation animationWithPropertyNamed:kPOPLayerScaleXY];
    scaleAnimation.fromValue  = [NSValue valueWithCGSize:CGSizeMake(0.5, 0.5f)];
    scaleAnimation.toValue  = [NSValue valueWithCGSize:CGSizeMake(1.0f, 1.0f)];//@(0.0f);
    scaleAnimation.springBounciness = 20.0f;
    scaleAnimation.springSpeed = 20.0f;
    [_popUp.layer pop_addAnimation:scaleAnimation forKey:@"scaleAnimation"];
}
{% endhighlight %}

这里补充三个注意事项:
1.当你添加一个动画类似`[myView pop_addAnimation:animation forKey:@"animationKey"];` 要记住,假如你添加其他的动画使用了相同的key,那么将会替换掉之前的那一个.

2.当你想一个动画用`CACurrentMediaTime()`并使用`beginTime`来指定. 因此,看起来就像这样:`animation.beginTimee = CACurrentMediaTime() + delayInSeconds;`. 我添加了一个简单的延迟,当然,他不会工作.感谢Kimon(@kimon)所提出的["警告"](https://github.com/facebook/pop/issues/32#issuecomment-41823023);

3.当你看到一个属性类似于`kPOPLayerScaleXY`,它会期望两个值. 在这种情况下是一个`CGSize`.现在可能是有意义的,但是我传了一个`NSNumber`(一个单一值)并且期待它可以同时设置X和Y的值.


这是一位天才(via @_tiagoalmeida):

![](/images/20140506/7.png)

它起效果了:
{% highlight C++ %}
POPAnimatableProperty *constantProperty = [POPAnimatableProperty propertyWithName:@"constant" initializer:^(POPMutableAnimatableProperty *prop){  
        prop.readBlock = ^(NSLayoutConstraint *layoutConstraint, CGFloat values[]) {
            values[0] = [layoutConstraint constant];
        };
        prop.writeBlock = ^(NSLayoutConstraint *layoutConstraint, const CGFloat values[]) {
            [layoutConstraint setConstant:values[0]];
        };
    }];

POPSpringAnimation *constantAnimation = [POPSpringAnimation animation];  
constantAnimation.property = constantProperty;  
constantAnimation.fromValue = @(_layoutConstraint.constant);  
constantAnimation.toValue = @(200);  
[_layoutConstraint pop_addAnimation:constantAnimation forKey:@"constantAnimation"];
{% endhighlight %}

非常感谢 Jake Marsh (@jakemarsh).

仅仅只是一条小笔记,我没有注意到`kPOPLayoutConstraintConstant`.所以你不需要自己去创建一个自定义的`POPAnimatableProperty`.


在玩了几天的POP后,现在已经进入到了除了享受以外,我还可以做出共享的时刻了.

在我第一篇文章中(本文开头部分) ,我创建了一个自定义的属性给`strokeStart`和 `strokeEnd` (两个都属于`CAShapeLayer`):

{% highlight C++ %}
[POPAnimatableProperty propertyWithName:@"strokeStart" initializer:^(POPMutableAnimatableProperty *prop) {
        prop.readBlock = ^(id obj, CGFloat values[]) {
            values[0] = [obj strokeStart];
        };
        prop.writeBlock = ^(id obj, const CGFloat values[]) {
            [obj setStrokeStart:values[0]];
        };
}];
{% endhighlight %}

这就几行代码的工作,但是不再害怕了.["我第一次提交的代码请求"](https://github.com/facebook/pop/pull/38) (希望不是最后一次) 已经被批准通过 并且已经合并到POP的主仓库.这意味着,我现在可以在`CAShapeLayer`上使用这两个属性而不用添加任何附加逻辑.

添加一个属性到POP只有三个简单的步骤(也许是4步或者5步).
假如有人想试一试:

>1.在`POPAnimatableProperty.h`内给你的属性名称添加一个`extern NSString`修饰,努力保持一个好的命名习惯.就像`kPOP<class name witout prefix><propertyName>`这样.假如他是由超过一个值组成的话,可以这样`kPOP<class name witout prefix><propertyName>XY`.
在这之后去`POPAnimatableProperty.m`添加真实的值,就像这样:`NSString * const kPop<class name witout prefix><propertyName> = @"<propertyName>"`.显然这并不是成文的规定,所以你不确定如何命名的话,去看看其他的属性是如何命名的.

>2.通过这种方法添加的 读/写的block 将会可以运行,增加`threshold`值.你可以看到["其他的属性是如何做到的"](https://github.com/facebook/pop/blob/master/pop/POPAnimatableProperty.mm#L86L432)

>3. 我不需要这样做,我添加的属性很简单.我想说,当你在读写一个新值时,使用辅助方法对你来说会更好.例如,你可以看一下`kPOPViewBackgroundColor`是如何做的:

{% highlight C++ %}
{kPOPViewBackgroundColor,
    ^(UIView *obj, CGFloat values[]) {
      POPUIColorGetRGBAComponents(obj.backgroundColor, values);
    },
    ^(UIView *obj, const CGFloat values[]) {
      obj.backgroundColor = POPUIColorRGBACreate(values);
    },
    1.0
  },
{% endhighlight %}

这种情况下要确保`POPUIColorGetRGBAComponents`和`POPUIColorRGBACreate`:

{% highlight C++ %}
void POPUIColorGetRGBAComponents(UIColor *color, CGFloat components[])  
{
  return POPCGColorGetRGBAComponents(color.CGColor, components);
}

UIColor *POPUIColorRGBACreate(const CGFloat components[])  
{
  CGColorRef colorRef = POPCGColorRGBACreate(components);
  UIColor *color = [[UIColor alloc] initWithCGColor:colorRef];
  CGColorRelease(colorRef);
  return color;
}
{% endhighlight %}
这些辅助方法位于`POPCGUtils`,尽管在`POPLayerExtras`有更多.作为一个好市民,你可以创建其他的方法,这样人们就可以使用它们来模拟属性的行为.

>1.添加的你属性到测试组!由于这些属性非常的天真,我仅仅添加它到`POPAnimatablePropertyTests.m`的`testProvidedExistence`,来保证它的实现方法实际存在.

>2.尽可能更多测试,假如你做一些与众不同的,并且没有被默认的组覆盖.

正如我需要成长,我将会贡献更多的内容到这个仓库.


这儿有些关于旋转的实验:
![](/images/20140506/8.gif)

下面是实现的代码:
{% highlight C++ %}
POPSpringAnimation *anim = [POPSpringAnimation animationWithPropertyNamed:kPOPLayerRotationX];  
anim.springBounciness = 20.0f;  
anim.dynamicsMass = 5.0f;  
anim.fromValue = @(degreesToRadians(degrees));  
anim.toValue = @(M_PI);  
[self.bundledLayers pop_addAnimation:anim forKey:@"rotationXAnimation"];
{% endhighlight %}

由于 Florien Kluger (@floriankugler) 对我之前的那个["示例"](http://codeplease.io/playing-with-pop-vi/)给出的建议,修改了一点儿地方.除了现有的之外,我并且可以在动画的时候抓住模块的顶部.这对于用户来说是一个很愉快的体验.

![](/images/20140506/9.gif)

最开始的时候我禁用并且移除了手势,在下面的条件时:

>1.`UIGestureRecognizerStateEnded`,在交互结束时,很快到动画结束的位置并禁用手势.

>2.假如这个view被点击过,我会仅仅只让动画到结束位置.

我用了第二条,它依然是有意义的.第一条已经被修改了.所以,我现在做的就是:

当`UIGestureRecognizerStateEnded`状态已经完成,我就设置下面的完成块:

{% highlight C++ %}
^(POPAnimation *anim, BOOL finished) {
        if (finished) [self disableGestures];
    }
{% endhighlight %}

所以假如动画已经完成了,我将禁用手势.更有趣的部分在`UIGestureRecognizerStateBegan`状态:

{% highlight C++ %}
if ([self.bundledLayers pop_animationForKey:RPPopRotationAnimation])  
{
  POPSpringAnimation *animation = [POPSpringAnimation animationWithPropertyNamed:RPPopRotationAnimation];
  [animation setCompletionBlock:^(POPAnimation *anim, BOOL finished) {

  CGFloat currentDegrees = [(NSNumber *)[self.bundledLayers valueForKeyPath:@"transform.rotation.x"] floatValue];
  [self applyRotationAnimationOnLayer:self.bundledLayers initialDegrees:currentDegrees toFinalDegrees:degrees completionBlock:nil];
            }];
}
else  
{
  [self applyRotationAnimationOnLayer:self.bundledLayers initialDegrees:0.0f toFinalDegrees:degrees completionBlock:nil];
}
{% endhighlight %}


1. 检查是否有动画正在进行(可能由于`UIGestureRecognizerStateEnded`)
    1.1假如是这样,就让我们"劫持"这个完成事件`completionBlock` 并且替换成一个新的转换效果,基于我触摸的位置.
2. 假如没有动画的话,我会让消息视图仍然呈现折叠并且在应用一个转换效果从`0.0f`开始到我触摸的地方.

除了代替劫持掉`completionBlock`,你还能:

{% highlight C++ %}
[animation setCompletionBlock:nil];
animation.toValue = @(degreesToRadians(degrees)); 
{% endhighlight %}

带来了不同的效果,但是依然是一个不错的. 首先第一个区别是,我会让动画完成并且应用一个新的转换效果. 第二个区别是,我不会允许动画到达原始的`toValue`位置.

PS:我的数学非常糟糕,所以基于我触摸点位置计算角度的逻辑在 ["这里"](http://math.stackexchange.com/questions/788286/interpotaliton-in-degrees).