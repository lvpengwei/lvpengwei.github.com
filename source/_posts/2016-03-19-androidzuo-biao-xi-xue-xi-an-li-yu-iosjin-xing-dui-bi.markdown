---
layout: post
title: "Android坐标系学习案例(与iOS进行对比)"
date: 2016-03-19 01:27:38 +0800
comments: true
categories: 
---

问题: 判断parent view中的手势是否出现在旋转之后的subview的范围中的解决方法(iOS&Android)

解决思路: 取出点击位置的point, 然后translate到subview的坐标系中, 判断是否在subview的矩形中即可.

##iOS:
现象:

![7DBEE162-CB37-44E7-8560-4E50556532DA.png](/images/20160319/7DBEE162-CB37-44E7-8560-4E50556532DA.png =500x)

代码:

![0FDD8A27-5C7F-48EB-A0DD-853FDA4B846C.png](/images/20160319/0FDD8A27-5C7F-48EB-A0DD-853FDA4B846C.png =500x)

结果:

![78067F60-E64B-4FFB-AA67-68783D824B32.png](/images/20160319/78067F60-E64B-4FFB-AA67-68783D824B32.png =500x)

部分解释: iOS中的view有frame/bounds/center/transform等坐标属性, 

###四者的区别(具体可查看文档, 非常详细)
 - `frame`: 相对于parent view的坐标系
 - `bounds`: 相对于自己的坐标系
 - `center`: view的中心点(跟layer的anchorPoint有关, anchorPoint默认值是(0.5, 0.5), 所以是中心点)
 - `transform`: 相对于center的缩放/旋转等二维操作. layer中有个transform是三维的.

三者之间的关系是-bounds/center/transform会影响frame的值. 

大部分情况, 我们使用的`frame`(x, y, width, height)只影响到了`bounds`和`center`, 此时`transform`是初始值`CGAffineTransformIdentity`; 而当我们要设置`transform`的时候, frame的值就会变得很奇怪. 其实`frame`还是那个矩形, 不过它代表的含义是能包含这个view的最小矩形而已. 因为`transform`涉及到scale/rotation等操作, 所以`frame`看起来和我们真正想要的值不太一样, 而这时候就是需要分开使用center/bounds/transform的时候, 不能单独设置frame.

`layer`的相关属性不介绍, 有兴趣可以直接查看API文档.

##Android:
类似的, 我开始在Android中寻找相应的解决方案. 首先需要明白的几个点

 - Android的每一个view也有自己的坐标系, 左上角是原点(0, 0)
 - Android的view的基础坐标属性是left/top/right/bottom. width和height是由前四个值推算得来.
 - Android的view的matrix作用对应于iOS的view的transform.

解决思路还是上面所述, 但是取出点击的point容易, translate到subview的坐标系中遇到了一些问题, iOS中此api存在于UIView中, 而Android的View却没有. 搜索一圈, 还是在Android的事件分发的源码中获取答案. 
现象:

![A5809894-B554-433E-B603-172DC01AEB93.png](/images/20160319/A5809894-B554-433E-B603-172DC01AEB93.png =500x)

代码:

![B7B8A792-4D22-4EDF-B672-A96C1A5FCCA4.png](/images/20160319/B7B8A792-4D22-4EDF-B672-A96C1A5FCCA4.png =500x)

布局代码:

![007A1AED-36E2-4A51-B5CA-7970D56DC40C.png](/images/20160319/007A1AED-36E2-4A51-B5CA-7970D56DC40C.png =500x)

结果:

![2C8C9B9F-FD20-4986-BE8B-BAAA346943FD.png](/images/20160319/2C8C9B9F-FD20-4986-BE8B-BAAA346943FD.png =500x)

部分解释:
代码部分借鉴了Android源码. 

![069605CE-BF37-4BDA-B054-0514E4090EA4.png](/images/20160319/069605CE-BF37-4BDA-B054-0514E4090EA4.png =500x)

 1. 先deep copy出一份新的event.
 2. 设置event的offsetLocation.
 3. 给event应用subview的inverse matrix.
 4. 还有很多类似对应的属性(后边继续学习, 本次没有用到)
 

 然后event就被成功的转换到旋转后的view的坐标系中(斜着的), 原点是左上角.

遇到的问题:

 - ev.offsetLocation: 因为我是在activity中override onTouchEvent, 所以在计算deltaX和deltaY的时候先减去了relativeLayout距离window的x和y, 再减去了demoTextView的left和top. 1.为什么不直接减去demoTextView距离window的x和y? 因为我发现, demoTextView被旋转之后, 它的left/top/right/bottom并没有改变, 从现象的截图里我们也可以看出. 而demoTextView.getLocationInWindow获取的point却是旋转之后的值, 从log中可以看出. 2.为什么不减去relativeLayout的left和top, 而是减去距离window的x和y? 因为relativeLayout只是activity的contentView, 外层还有多少ViewGroup是未知, 所以直接减去距离window的x和y.
 - ev.transform: 源码里面的写法是`transformedEvent.transform(child.getInverseMatrix());`, 而我自己去调用`demoTextView.getInverseMatrix()`的时候, 一直报错, 而且API文档中并没有这个方法, 所以就直接去Matrix的类里面去搜关键词`inverse`, 发现有这样的方法可以使用. 
