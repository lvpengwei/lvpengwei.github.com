---
layout: post
title: "iOS APP的简单Web Interface应用"
date: 2016-06-13 17:28:28 +0800
comments: true
categories: 
---

      最近接了一个内部工具开发的需求：运营人员需要生成商品尺码图片，之前他们是用PS做的，现在想让我们做个程序工具化这个过程。

刚拿到这个需求，分析之后，想了两个方案：

 1. APP生成图片，导出到电脑
 2. h5生成图片save到电脑
 3. 脚本生成，因为UI比较复杂，需要获取数据，所以排除

由于对APP开发比较熟悉，所以尝试着用客户端去做，生成图片不难，关键是如何导出到电脑上，当时的思路是

 1. iOS的话保存到相册用AirDrop去传
 2. Android保存到SD卡上，用PC上的软件导出
 3. 打成zip包，客户端起一个server，把zip download下来

本着操作简洁的原则，开始探索第三种方案。

一个程序，对用户来说只要有输入和输出就行了，中间不需要什么过多的介入，这才是优化流程的意义所在。

在这个例子中，输入是商品id数组，输出是一个zip包。

设计流程是：接收一个商品id数组，开始进行处理，最终输出一张图片save到Documents的一个文件夹下，然后进行zip打包，最后进行下载。

遇到的问题：

1.iOS的`AutoLayout`，图片的模板是用`AutoLayout`实现，变高(基于`AutoLayout`中`view`的`ContentSize`概念)。问题就是截图的时候会报警告，然后出的图是空的，debug之后，发现是view赋值之后没有进行layout，所以size是`CGSizeZero`的，截图时得到的context是0x0，之前在项目中，没有出现此问题，是因为view一般都会有superview，而superview在layout的时候subview也会layout。这里的view没有superview，所以没人触发它的layout过程，加了两行代码，完成。
```
[self setNeedsUpdateConstraints];
[self layoutIfNeeded];
```

2.在提交商品id数组之后，如果处理过程是异步的，那么如何通知用户是个问题，本来打算是手机local notification通知，但是我们这个APP直接部署在局域网的Mac Pro的模拟器里面，不需要用户安装，而web页面做push操作比较麻烦，所以采用同步处理，也就是说，提交过商品id数组之后页面一直loading，直到处理完毕，给出下一个页面。在APP端，商品信息的加载和商品主图的加载也是异步的，那么怎么才能在一个请求中做同步呢，也就是收到web请求的时候，提交到一个manager中去处理这部分商品，但是这里sleep掉，等那边处理完毕之后，再唤醒接着返回response，于是就想到了iOS的`Runloop`这个机制。代码如下：

`MyHTTPConnection.m`
```
	NSPort *port = [NSMachPort port];
    [[NSRunLoop currentRunLoop] addPort:port forMode:NSDefaultRunLoopMode];
    [ExportImageManager sharedInstance].thread = [NSThread currentThread];
    [[ExportImageManager sharedInstance] createImageWithGoodsIds:goodsIdArr];
    while (![ExportImageManager sharedInstance].completed) {
        NSLog(@"runloop start......");
        [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
        NSLog(@"runloop end......");
    }
    [[NSRunLoop currentRunLoop] removePort:port forMode:NSDefaultRunLoopMode];
    CFRunLoopStop(CFRunLoopGetCurrent());
    [ExportImageManager sharedInstance].thread = nil;
```
注解：
这块代码执行的环境是在HTTPServer的队列里面，所以在哪个线程中是未知的，线程中的runloop默认是不开启的，所以在这里，做了几件事

 1. 声明一个port放在runloop中，防止runloop中没有任何输入源会直接退出
 2. 把当前所在的thread赋给manager，当处理完成的时候，在这个thread上performSelector，唤醒runloop
 2. 在while循环中启动runloop
 3. 把之前的port remove掉
 4. 把runloop停掉
 5. 之前引用的thread清空

最终实现，3个操作步骤

 1. `localhost:8081/create?id,id,id,id` 图片生成请求，response返回的时候处理结束，会在`Documents/export/`的文件夹下生成以商品id命名的图片
 2. `localhost:8081/zip` 将`Documents/export`打包成`Documents/export.zip` 并删除`Documents/export`
 3. `localhost:8081/export.zip` 下载zip

其实可以把打包操作合并在第一步中。缩减为两步

**思路来源是，QQ阅读传pdf的时候是在电脑上打开一个网页，然后把文件拖进去就可以同步到手机。实现之后发现，这个功能和`Charles`的`Web Interface`功能一样。。**

项目地址：https://github.com/lvpengwei/ExportGoodsImage

![Simulator Screen Shot Jun 13, 2016, 17.00.33.png](/images/20160613/3834834005.png =500x)

  