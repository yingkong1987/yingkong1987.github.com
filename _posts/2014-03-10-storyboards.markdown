---
layout: post
title: "Storyboards"
date: 2014-03-10 00:05:55 +0800
comments: true
categories:
---

在iOS5之前,界面基础和试图都是使用`Interface Builder `(IB)来创建的,并用nib文件来保存.Storyboards是另外一种创建界面的方式,并且添加了一些内容创建界面基础,你可以在两个界面之间指定导航(叫做segues).这样可以让你在开始写代码之前不用做什么事情.你可以把storyboards想象成一个图表,上面所有view controllers用segues连接,还可以指定之间的转换效果.

storyboards带来的好处还不止于此. 它同样可以很轻松让开发者创建一个静态的table view而不用data source. 有多少次你想去创建一个table view而不需要去绑定一个真正的data source -- 例如,一个table来展示一个选项列表代替数据.一般用来app中的设置页面.
Storyboards 也可以帮助合作开发者或者客户了解app完整的工作流程.

Storyboards 并不都是浪漫的,在我看来,它依然有着严重的缺陷,不过在以后的章节中,我们会学习如何使用故事版而不受那些缺点的影响.

<!-- more -->

>Storyboards是构建你用户界面的未来.假如你一直在逃避Storyboards,那么现在是时候来学习它了.事实上,当你使用新工程模版创建一个工程的时候,Xcode5都不提供方法来关闭Storyboards(以前创建viewbase模版的是偶有一个选项可以关闭Storyboards).

你首先学习如何开始使用storyboards ,和使用storyboards做一些事情.在此之前你可能用的是nib文件来处理这些事情.比如controllers间的通信.在本章稍后的“Static Tables” 段落中,你会找到如何创建一个静态table view 而不要data source..最后你会发现storyboards很多有趣的方面:写入你自己自定义的过渡动画.尽管这些很酷的过渡动画听起来很复杂, 但是iOS的SDK可以让你很轻松的去实现起来.

###开始使用Storyboards
你可以在一个新的工程使用Storyboards或者在一个还没有Storyboards的项目中添加它们.对于现有的项目,你可以添加Storyboards像添加一个新文件到工程中一样.在接下来的“Instantiating a Storyboard.”部分中你将学到如何在storyboard中实例化一个view controllers.

当你创建一个使用storyboards新工程
,这个`info.plist`中包含的一个key叫做`UIMainStoryboardFile`.这个key替代了iOS5之前使用的`NSMainNibFile`.假如你的主 window是从一个nib文件中加载的,那么你可以继续使用NSMainNibFile来代替storyboard.然而,你不能在一个app中同时使用`UIMainStoryboardFile`和`NSMainNibFile`.优先使用`UIMainStoryboardFile`,并且你在`NSMainNibFile`中指定的nib文件将不会被加载.

>你的应用程序可以完整的储存在一个文件中,并且IB自动构建成单独的分离文件去做最优化的加载.简而言之,你不必担心使用storyboards时的加载时间或者性能.


###实例化一个Storyboard
当你的'UIMainStoryboardFile'已经设置好了后,随着程序的启动窗口,编译器在自动生成代码时实例化Storyboard并且加载.假如你已经添加了storyboards到你现有的APP中,你就开始编码吧.storyboard实例化view controllers的方法都定义在`UIStoryboard`类中.

当你想在storyboard中指定显示一个 view controller的时候,你用这个方法加载storyboard:
{% highlight C++ %}
+ storyboardWithName:bundle:
{% endhighlight %}

###在Storyboard中加载View Controllers
在storyboard中加载View Controllers的方法类似于nib的加载方法,并且搭配`UIStoryboard`对象.你可以使用下面的一些方法实例化View Controllers:

{% highlight C++ %}

 - instantiateInitialViewController
 - instantiateViewControllerWithIdentifier:

{% endhighlight %}

###Segues
Segues是在你storyboard文件中的过渡定义.UIKit提供了两个默认的过渡风格: Push
和 Modal. 他的表现形式类似于`pushViewController:animated:completion:`和`presentViewController:animated:completion`方法.

在新增内容中,你可以创造一个自定义segues和创造一个新的view controllers过渡种类.你可以稍后在“自定义过度效果”中看看这些内容.

你在storyboard文件中通过连接view controllers的某个事件到其他的view controllers来创建segues. 你可以拖动一个button到一个 view controller上,或者拖动一个手势识别到一个view controller,还有很多,就不一一列举了.IB在两个VC间创建了一个segue后,你还可以选中segue,并使用attributes inspector面板来修改这个过渡风格.

![](/images/20140311/1.png)

假如你选择了一个自定义的过度风格,attributes inspector 面板依然可以允许你设置一个自定义的类.你可以认为,segue是专门负责关联过渡的行为.按钮的点击事件行为,静态table上row(cell)的被选中,一个手势,或者甚至是声音事件,都可以引发segues.编译器会自动生成所需要的代码来执行segue的行为,当你已经连接了segue后.

当segue去执行被源VC调用的`prepareForSegue:sender:`方法.传递给他的是一个`UIStoryboardSegue`类型的对象.你可以覆写这个方法来传递数据给目标view controller.下一节
将解释如何完成这一任务.

当一个 view controller 执行多个segue,相同的` prepareForSegue:sender: `方法会被每个segue调用.要鉴别出执行的segue的话,使用segue identifier来检查是否是准备执行的segue,并传递相应的数据.
作为防御性编程实践,我建议也执行这个检查事件,即使你的viewcontroller执行的只有一个segue.这样的话,你可以保证以后你添加一个新的segue,你的app也会继续运行而不是崩溃.


###传递数据
在你使用storyboards时,viewcontroller 实例化并自动的呈现给用户.你有机会通过覆写` prepareForSegue:sender: `方法来填充数据.通过覆写这个方法,你可以获得指针到目标viewcontroller,并用来设置初始化的值.

framework 会调用相同的方法在你使用之前.比如 `viewDidLoad`, `initWithCoder:`,和` NSObject awakeFromNib `方法,这就意味着在你没有使用storyboard的时候你可以继续写你的viewcontroller的初始化代码.


###返回数据
使用storyboards传递数据返回到父级(或者叫做上层)viewcontroller上就跟你使用nib文件或者手动编码用户接口一样.你在View上展示的那些用户创建或者输入的数据,可以通过delegate或block的形式返回上级.唯一不同的是,在你的父级viewcontroller,你需要去在` prepareForSeque:sender: `方法中设置delegate为`self`.

###实例化其他的viewcontroller
`UIViewController`有一个`storyboard`属性,持有一个来自于被实例化的storyboard对象的指针(UIStoryBoard) .假如你的viewcontroller创建于nib文件或者手动编码那么这个属性将会是nil.换句话说,你可以实例化其他定义在你storyboard中的viewcontroller. 你通过使用viewcontroller的标示符做这些.下面这个`UIStoryBoard`的方法可以让你使用表示符实例化一个viewcontroller.

{% highlight C++ %}
 – instantiateViewControllerWithIdentifier:
{% endhighlight %}

 因此,在没有通过segues链接任何viewcontroller的时候你依然可以拥有viewcontroller在你的storyboard里.并且这些viewcontroller依然可以被初始化并使用.

 ###手动执行Segues
 虽然storyboards可以自动触发segues基于actions.在一些情况,你可能需要去执行一个segues编程.你可能需要去解决actions不能被storyboard文件处理的问题.去执行一个segue,你调用viewcontroller的这个` performSegueWithIdentifier:sender: `方法.当你手动执行segues时,你可以通过sender参数内的调用者和上下文对象.这个sender参数将会在稍后发送`prepareForSegue:sender: `方法.

###解开Segues
 Storyboards本来是允许你去实例化并且有导航图标的viewcontroller.iOS6在`UIViewController`引进的方法允许你解开一个解开Segues. 解开Segues后,你可以实现导航器"back"的方法,而不用创建附加的viewcontroller.

 你可以在你的viewcontroller内通过实现一个`IBAction`来添加解开支持Segues,取得`UIStoryboardSegue`参数,就像下面一样:

{% highlight C++ %}
 -(IBAction)unwindMethod:(UIStoryboardSegue*)sender {
   }
{% endhighlight %}

 你现在可以在一个viewcontroller内连接事件到他的`Exit`对象.Xcode自动计算在storyboard内所有可能的解开事件(任何返回为`IBAction`的方法都可以接受一个`UIStoryboardSegue`作为参数),并且允许你去连接他们.如图所示:

 ![](/images/20140311/2.png)

###通过storyboard构建tableview
storyboard一个重要的优势就是从IB中构建静态tableview的能力.在storyboard上,你可以构建两种类型的tableview:一个类型是不需要指定一个类做datasource的静态tableview,另外一个是包含一个原型cell来约束model内数据(类似iOS4中的custom tableview cells).


####静态tableview
你可以在你的storyboard里面创建一个静态tableview,首先拖动一个table到storyboard上,并选中.然后在 attributes inspector 上选择`Static Cells`..如图所示:
 ![](/images/20140311/3.png)

 Static cells 对于创建设置页面是非常不错的选择,例如苹果的设置页面就是Static cells做的.(或者说内容不是从coredata model,网络服务器,或者任何诸如此类的datasource来的页面).

 >Static cells的tableview仅仅只能从一个`UITableViewController`中创建.你不能创建static cells的tableview,假如这个tableview是作为`UIViewController`的子视图被添加的.

###Prototype Cells(原型cell)
 Prototype Cells类似于tableview的custom cell.但是并不是创建Prototype Cells在单独的nib文件上和加载在datasource的方法`cellForRowAtIndexPath:`内.
 你创建他们在storyboard的IB上,并且仅仅只需要在你的datasource方法内设置数据.

 >你所有的Prototype Cells标示符都是使用一个自定义的标示符.来确保在tableview的队列方法中适当的运行.假如你storyboard中的Prototype Cells没有一个cell标示符,那么Xcode会给你提出警告.


###自定义转场(过渡)

 Storyboards 会更早的执行一个自定义的转场效果在一个viewcontroller到另一个viewcontroller的时候.
 当segues执行的时候,编译器会基于你在storyboard中设置的自定义转场风格生成必要的代码来present或push目标viewcontroller.你可以看到有两种转场风格,push和model,这两个是iOS原生支持的.当然还会有第三个风格---自定义.选择自定义并且提供一个你自己的`UIStoryboardSegue`子类来处理你自定义的转场效果.

 创建一个`UIStoryBoardSegue`的子类并覆写`perform`方法.在这个`perform`内,访问来源viewcontroller的主view的layer指针,并且实现你自定义专场动画(使用core animation).当动画执行完成的时候,push或者present你的目标viewcontroller.(你可以从segue对象中获得他的指针).这总的来说是很简单的.

 举例说明的话,我将展示如何创建一个过渡效果,关于主视图push到详情视图显示在底部.

 创建一个新的工程使用` Master-Details `模版.打开`MainStoryboard`.点击segue并修改类型为custom. 添加一个`UIStoryboardSegue`子类.覆写`perform`方法,再粘贴下面的代码;

{% highlight C++ %}

Custom Transition Using a Storyboard Segue (CustomSegue.m)
   - (void) perform {
     UIViewController *src = (UIViewController *)self.sourceViewController;
     UIViewController *dest = (UIViewController *)self.destinationViewController;
     CGRect f = src.view.frame;
     CGRect originalSourceRect = src.view.frame;
     f.origin.y = f.size.height;
     [UIView animateWithDuration:0.3 animations:^{
       src.view.frame = f;
     } completion:^(BOOL finished){
       src.view.alpha = 0;
       dest.view.frame = f;
       dest.view.alpha = 0.0f;
       [[src.view superview] addSubview:dest.view];
       [UIView animateWithDuration:0.3 animations:^{
         dest.view.frame = originalSourceRect;
         dest.view.alpha = 1.0f;
       } completion:^(BOOL finished) {
         [dest.view removeFromSuperview];
         src.view.alpha = 1.0f;
         [src.navigationController pushViewController:dest animated:NO];
	}]; }];
}
 
{% endhighlight %}


这样的话,你就可以用目标viewcontroller的layer指针做出许多疯狂的事情来..Justin Mecham在github开源了一个关于doorway转场效果非常不错的例子.[(https://github.com/jsmecham/DoorwaySegue)]((https://github.com/jsmecham/DoorwaySegue))..你也可以创建你自己的过渡效果,通过控制源layer的指针和目标viewcontroller.

>iOS7介绍了不少复杂的API来执行自定义的转场效果. 集合视图布局转场,和交互师转场效果.例如navigation controller中普遍的pop手势.


###优势
当你使用storyboard的时候,合作开发者(或者客户/项目经理)更容易理解这个APP的流程.而不是通过大量的nib文件和穿插的实例化代码来理解流程. 合作开发者可以打开storyboard来完整的理解流程.这仅是一个给力的理由让人们使用storyboard.

###劣势 地狱般的合并冲突
storyboard对于整个团队有一个让人非常烦恼的问题.Xcode默认的应用程序模版会有一个对整个程序的storyboard.这就意味着当两个开发者同时在UI上工作时.合并冲突将无法避免.并且由于storyboard内部自动生成XML,造成这些合并冲突修复起来将会非常复杂的.甚至用一个新的XML格式化文件来计划减少合并冲突,但是storyboard还是常常获得冲突信息.

最简单解决这个问题的办法就是首先避免发生这种情况. 我的建议是你拆分你的storyboard为多个文件,每种使用情况对应一个文件.大多数情况,一个开发者只会在一种情况上工作,这样最后会导致合并冲突的概率降低很多.

Twitter客户端使用基于情况分类的storyboard的例子是`Login.Storyboard, Tweets. Storyboard, Settings.Storyboard, and Profile.Storyboard. `
而不是把storyboard变成多个nib文件(又绕回起点了).使用多个storyboard来帮助解决这个合并冲突的问题,又同时保留了优雅.像之前我提到过的,一个开发者不会在开发登录的同时又开发消息.
