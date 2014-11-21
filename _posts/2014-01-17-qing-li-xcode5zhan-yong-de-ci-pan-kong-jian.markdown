---
layout: post
title: "清理Xcode所占用的磁盘空间"
date: 2014-01-17 13:22:16 +0800
comments: true
categories: 
---

<!-- more -->

[本文源自:http://blog.favo.org/post/31649090293/xcode-5-places-to-save-some-disk-space,仅供学习使用](http://blog.favo.org/post/31649090293/xcode-5-places-to-save-some-disk-space)

我的SSD已经空间不足了.所以我检查并删除了一些不再需要的数据资料.

你可以清理Xcode在下面几个方面:

<h1>1) ARCHIVES</h1>

打开`Organizer `(⇧+⌘+2),切换到`Archives `

![](/images/20140117/2.png)

然后删除所有这些随着日子一天天产生的旧archives.

![](/images/20140117/1.png)

假如你觉得操作太麻烦了~还有另外一种so easy的办法(想起来田雨橙:爸爸再也不用担心我的学习). 就是随便选中其中的一个archives,然后右键.点击`Show in finder`

![](/images/20140117/3.png)

或者使用finder的前往文件夹,输入路径`~/Library/Developer/Xcode/Archives`.删除里面的旧档案.一个archive要比一个.IPA文件大得多,因为archive还包含了很多额外数据,比如debug 信息.


<h1>2) DERIVEDDATA</h1>

这个是每个工程都会产生的,并且包含了工程索引,编译输出还有日志.他可以在删除后重新生成,并在第一次重新打开的的时候消耗一点时间.

打开`Organizer `(Command-Shift- 2),选择`Projects `,点击每一个工程的`delete `.会有一个弹出警告框,上面对这个操作有更详细的说明,点击`delete `确认删除.

![](/images/20140117/4.png)


<h1>3) BASICS (FIRMWARE & LOGS)</h1>

还是在`Organizer `里面(快捷键我就不写了).选择Devices ,看到左上角的Device logs 和Screnshots 没..里面都是调试的时候产生的日志和截图. 特别是截图可是占用比较多的东西..删除很简单,选中-然后`delete`.

![](/images/20140117/5.png)

<h1>4) OLD DEVICE INFORMATION</h1>

你可以删除一些旧的iOS信息,
在`~/Library/Developer/Xcode/iOS DeviceSupport/`路径下可以看到那些大概~400MB的文件夹.(快捷方式:打开finder,⇧+⌘+G,粘贴上面的路径,回车)
它们可以自动重新生成这些数据,前提是你插入了一个设备,并且在这个文件夹内没有该设备所运行的iOS版本.

<h1>5) SIMULATOR-APPS</h1>

假如你测试一个很大的APP项目,你可能也会想清理掉模拟器.让我们来看`~/Library/Application Support/iPhone Simulator`路径.
里面的Application文件夹都包含有这个版本对应模拟器的所有APP.

—————————————————————————————————————————————

总的来说,我努力清理回来了大概20GB的空间,就是从刚才检查Xcode的那些步骤中 (10GB 在 archives,4GB 在 DerivedData, 2GB 在 basics 还有 2GB DeviceSupport … 最后 2GB 是 删除了这个  “Install Xcode.app” )