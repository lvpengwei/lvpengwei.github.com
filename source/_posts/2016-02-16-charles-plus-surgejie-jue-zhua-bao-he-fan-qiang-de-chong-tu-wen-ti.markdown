---
layout: post
title: "Charles+Surge解决抓包和翻墙的冲突问题"
date: 2016-02-16 23:34:49 +0800
comments: true
categories: 
---

平常做iOS开发的时候经常使用`ShadowsocksX`来翻墙查资料, 使用`Charles`来抓包debug. 但是两个软件不能同时开, 一直想不到什么好的解决办法.

`Surge`特点是: 支持翻墙, 但是抓包的request和response不够详细.
Mac版的Surge还要自己配置`Web Proxy(HTTP)`和`Secure Web Proxy(HTTPS)`.

然后我就查了一下`Charles`的菜单, 发现了`External Proxy Setting`, 然后发现刚好支持`Web Proxy(HTTP)`和`Secure Web Proxy(HTTPS)`, 配好调试, 果然成功. 然后手机上再设置成`Charles`的`Web Proxy(HTTP)`, 也是可以抓包和翻墙的. 

`Charles`特点是: 抓包的request和response很详细, 但是`External Proxy Setting`支持的protocol比较少.

`Charles`和`Surge`结合起来就很完美的解决了我这个问题.

再简化一步, 就直接用手机上的`Surge`生成一个`Web Proxy(HTTP)`的config, 这样就不用每次去系统设置里手动设置了(每次敲好麻烦), 这样每次进入公司, 电脑上开着`Charles`和Mac版的`Surge`, 手机上起着`Surge`, 两个设备都可以被抓包和翻墙, 妈妈再也不用担心我在`ShadowsocksX`和`Charles`这两个软件之间来回切换了.

附几张比较重要的图:
`Charles`的`External Proxy Setting`
![51F279D2-6924-4BB8-A230-48C693F2CE96.png](/images/20160216/51F279D2-6924-4BB8-A230-48C693F2CE96.png)

手机上的Surge的`Web Proxy(HTTP)`的config
![IMG_1261.png](/images/20160216/IMG_1261.png =500x)
