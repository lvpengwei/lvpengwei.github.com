---
layout: post
title: "导出迪斯尼乐拍通里的照片"
date: 2016-12-31 20:42:21 +0800
comments: true
categories: 
---
> 迪斯尼乐拍通App里面的照片导出收费，而且费用很高，我通过抓包拿到App里面的图片链接然后使用脚本下载解析得到高清的图片（非原图）
> 代码：[Disneyphotopass-export](https://github.com/lvpengwei/Disneyphotopass-export)

###过程

1. **抓包**：使用Charles拿到某张图片的链接，下载下来之后发现是另外一个图片，如下图，这时候就蛋疼了，那我需要的那张图去哪了，而且这张图片的分辨率不高，按理说不会这么大，那么猜测我需要的那张图就藏在这返回的数据里面     
![enter image description here](http://www.disneyphotopass.com.cn:4000/media/d14acc0d4a274d3154486fe481c634dbfff6cffcf8681e56bfb8fd8f03648f0616a3ce5b8a0ee8d8890cb2c68dfe6a466d20010501d7d081d21c2941b053a89d72d589b23d5caffdd71cf7564126e5db538a3998bf98b4cc39ae265aa5676705)
2. **隐写术**：之前不知道这个概念，所以我就进行了各种尝试去找
	- 是否是gif
	- 使用iOS ImageIO的API去看是不是在图片的exif里面
	- 然后问了伟哥，是否有方法把一张图片放在另外一张图片里，伟哥说“有好多图片隐写的方法你可以搜一下”     
算是找到了这种方案的一个术语-**隐写**，然后发现其中一种隐写方法的解析工具[binwork](https://github.com/devttys0/binwalk)，拿出之前下载的图片试一下，果然里面放了两张图片，如下图
![enter image description here](/images/20161231/E7997AA4-F8CF-41E5-A10E-CF3C205B45F1.png =500x)     
3. **解析**：使用命令可以解出想要的图片```dd if=image-download/1 bs=1 skip=10103 of=image/1.jpg```
![enter image description here](/images/20161231/80F75B59-CAC7-427B-9E19-2C31B19BA0DE.png =500x)

###脚本
然后把这个过程写成一个脚本，输入是一行一个链接的文本，输出是想要的图片，脚本来做下载和解析。

###tokenId获取
脚本可以自动收集链接，需要抓包获取tokenId

1. 使用浏览器(Chrome)：登录网页版[乐拍通](https://disneyphotopass.com.cn/)，然后`检查(Inspect)`去`网络(Network)`里找
2. 乐拍通App抓包

###链接的收集(Deprecated)
给手机设上代理，打开乐拍通，进入图片详情，然后一张一张滑过去，然后过滤一下，全选，右击，Copy URLs，然后粘在一个`disneyphotopass`文档里，执行脚本，就可以了。
![enter image description here](/images/20161231/8E02B64D-EC99-438C-B354-3F87C3DCB6BE.png =500x)
