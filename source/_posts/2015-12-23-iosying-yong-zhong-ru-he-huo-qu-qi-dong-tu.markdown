---
layout: post
title: "iOS应用中如何获取启动图"
date: 2015-12-23 10:32:14 +0800
comments: true
categories: 
---

可以在文档中搜索`UILaunchImages`和`UILaunchImageFile`这两个关键字。


<!--more-->


前提:使用.xcassets配置LaunchImage。
1.设备系统为iOS7以上，可以通过遍历由`UILaunchImages`取出来的dict，根据当前尺寸、系统和方向获取名字。key为`UILaunchImageName`, `UILaunchImageMinimumOSVersion`, `UILaunchImageSize`, `UILaunchImageOrientation`。
![E5589B97-D7D4-4B6B-B648-0F9D15FEBFA2.png][1]
![5B51C674-18CC-408C-A394-53ED00E5849F.png][2]
2.设备系统为iOS6及以下，可以通过`UILaunchImageFile`取出启动图名字，在配上一些参数获取。
* iPhone5拼上`-568h`
* iPad拼上`Portrait`或者`Landscape`
* 其他直接用启动图名字
![C8D35976-AE29-4601-A4BB-45E24826BDCB.png][3]
最后，分析一下.xcassets的配置和打进bundle中素材的对应关系
![D8BBA614-CC66-4DE1-A01D-DEE798E2A549.png][4]
![053E1411-BC46-404B-8153-564749A1EBEC.png][5]

iOS8 and Later中的素材分别对应的是iPhone6和6p的尺寸，所以名字是`LaunchImage-800-667h@2x.png`和`LaunchImage-800-Portrait-736h@3x.png`(因为6p可以横向，所以带有`Portrait`，不过事例中没有勾选设置)

iOS7 and Later中`iPhone`的`Portrait`生成的素材是`LaunchImage-700@2x.png`和`LaunchImage-700-568h@2x.png`
`iPad`的`Portrait`生成的素材是`LaunchImage-700-Portrait@2x~ipad.png`和`LaunchImage-700-Portrait~ipad.png`

iOS6.0 and Later中`iPhone`的`Portrait`生成的素材是`LaunchImage-568h@2x.png`, `LaunchImage@2x.png`和`LaunchImage.png`
`iPad`的`Portrait`生成的素材是`LaunchImage-Portrait@2x~ipad.png`和`LaunchImage-Portrait~ipad.png`

[项目地址](https://github.com/lvpengwei/LVLaunchImage)

[本篇博客另外地址](http://blog.yourdream.cc/2015/03/28/119.html)

  [1]: http://blog.yourdream.cc/usr/uploads/2015/04/2453186553.png
  [2]: http://blog.yourdream.cc/usr/uploads/2015/03/453113142.png
  [3]: http://blog.yourdream.cc/usr/uploads/2015/04/2656813522.png
  [4]: http://blog.yourdream.cc/usr/uploads/2015/03/3287509698.png
  [5]: http://blog.yourdream.cc/usr/uploads/2015/03/3750565276.png