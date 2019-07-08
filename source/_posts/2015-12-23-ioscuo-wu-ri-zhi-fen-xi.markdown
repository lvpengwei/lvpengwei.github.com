---
layout: post
title: "iOS错误日志分析"
date: 2015-12-23 10:20:55 +0800
comments: true
categories: 
---
穿衣助手iOS使用umeng来做错误日志统计，下面介绍一下如何通过lldb和错误日志定位崩溃位置。


<!--more-->


1.首先，启一个终端，输入`lldb`，回车。进入lldb模式。如图
![F91A0BB9-84B4-4BAE-8F11-FF937DEF0CC0.png][2]

2.输入`target create --no-dependents --arch armv7 'path'`，此处path是指可执行文件CYZS的位置，路径大概是`~/CYZS\ 1-27-15,\ 9.49\ AM.xcarchive/dSYMs/CYZS.app.dSYM/Contents/Resources/DWARF/CYZS`,armv7根据错误日志所记录进行改变，执行即可，如图
![DC2A5FFA-6CE8-4EC2-B189-BB26BADE3A2B.png][3]

3.使用`image lookup --address '地址'`进行定位。umeng日志如图:
![1E31F2DA-9035-4C3C-BCF0-CC0DF28C4502.png][4]

然后复制那些前面标有我们APP名称的地址
![407AAE6A-E111-4171-92A8-32BF1F461881.png][5]

Update : 2015-04-23
umeng中64位crash的log中发现slide address跟以前的不一样，所以不能正确的定位crash的地址。
解决方法是在第二步之后输入`image list`看是否有与umeng中一致的地址，若没有则输入`target modules load --file 'path' __TEXT 0x429496729616`重设即可。

![asdfasdfa.png][1]

[本篇博客另外地址](http://blog.yourdream.cc/2015/03/06/107.html)

参照
[The LLDB Debugger](http://lldb.llvm.org/symbolication.html)


  [1]: http://blog.yourdream.cc/usr/uploads/2015/04/612599683.png
  [2]: http://blog.yourdream.cc/usr/uploads/2015/03/1979910195.png
  [3]: http://blog.yourdream.cc/usr/uploads/2015/03/2077915295.png
  [4]: http://blog.yourdream.cc/usr/uploads/2015/03/4132646907.png
  [5]: http://blog.yourdream.cc/usr/uploads/2015/03/1195208596.png