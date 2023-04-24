---
layout: post
title: "PAG 去掉 skia 后如何渲染 path"
date: 2023-04-24 21:37:16 +0800
comments: true
categories: pathkit skia path-render 2d
styles: data-table
---


## 问题
在 2D 绘图库中，path 绘制是很重要的一块。

## 思路
当我们要实现这一块的时候，优先想到的方案是封装一层接口，然后用平台侧的 API 去实现 path 的编辑和绘制，我们再提交到 OpenGL 去上屏。

但是经过调研发现，iOS 和 Mac 的 CGPath、web 的 Path2D 都没有实现类似 skia 的 [path-ops](https://api.skia.org/SkPathOps_8h.html#a3db3691847c3c9828e0c141e275b8ec2)  功能；Android 因为背后就是 skia，所以它的 API 和 skia 基本保持一致；在 linux 平台，没有找到合适的 path 编辑库。

然后发现 skia 有一个 module 叫 [PathKit](https://skia.org/docs/user/modules/pathkit/)，可以把 skia 的 Path 编辑部分编译成 
wasm 提供给 web 使用，渲染时把 SkPath 转成 Path2D 或者直接调用 web-canvas 的 path 接口。因为之前看过 rlottie 使用 freetype 来渲染 path，所以如果 SkPath 能转成 FTOutline，那么全平台的 path 渲染都可以用 freetype 来实现。 

调研之后得出结论是 OK 的。

因为 freetype 会增加包大小，而且在 iOS 和 web 平台，用 freetype 没法使用系统字体，所以我们决定 path 编辑部分都使用 SkPath，渲染部分 iOS 和 Mac 用 CoreGraphics，web 平台使用 canvas，其他平台因为文字解析需要使用 freetype，就统一用 freetype 来渲染 path。

<table>
   <tr>
      <td>平台</td>
      <td>path 渲染</td>
   </tr>
    <tr>
      <td>iOS & Mac</td>
      <td>CoreGraphics</td>
   </tr>
   <tr>
      <td>Android</td>
      <td>freetype</td>
   </tr>
   <tr>
      <td>linux</td>
      <td>freetype</td>
   </tr>
   <tr>
      <td>windows</td>
      <td>freetype</td>
   </tr>
   <tr>
      <td>web</td>
      <td>canvas</td>
   </tr>
</table>


在把 `SkPath` 转成 `FT_Outline` 时，要注意 `FT_Outline` 的 `n_contours` 和 `n_points` 是 `short` 类型，所以一个 `SkPath` 可能会转成多个 `FT_Outline`。

## pathkit
提取出来的代码我们作为第三方库 [pathkit](https://github.com/libpag/pathkit) 来引入。

## GPU 加速
skia 的 path 绘制有一部分是使用 GPU 加速的，而我们实现的是 CPU 绘制再提交到 GPU。

我们去调研了一下，这么多年以来，path 的绘制都是依赖 CPU 去实现的，GPU 在发展的过程中并没有考虑这个问题。Khronos 有一个硬件加速矢量绘制的标准 API，叫 [OpenVG](https://zh.wikipedia.org/wiki/OpenVG) ，但是没有普及起来。Nvidia 提供了 [GPU Accelerated Path Rendering](https://developer.nvidia.cn/gpu-accelerated-path-rendering) ，也没有普及起来。而且不是所有的 path 都适合用 GPU 加速。

所以我们现阶段的方案是简单图形比如矩形、椭圆、圆等等，用 GPU 加速，复杂 path 比较少用，实现起来也比较麻烦，可以后续参考 skia 再支持 GPU 渲染。

## 链接
[skia](https://github.com/google/skia)  
[path-ops](https://api.skia.org/SkPathOps_8h.html#a3db3691847c3c9828e0c141e275b8ec2)  
[PathKit](https://skia.org/docs/user/modules/pathkit/)  
[rlottie](https://github.com/Samsung/rlottie)  
[CoreGraphics-CGPath](https://developer.apple.com/documentation/coregraphics/cgpath)  
[CoreGraphics-CGContext](https://developer.apple.com/documentation/coregraphics/cgcontext)  
[web-Path2D](https://developer.mozilla.org/zh-CN/docs/Web/API/Path2D)  
[web-Canvas](https://developer.mozilla.org/zh-CN/docs/Web/API/Canvas_API/Tutorial)  
[GPU-Accelerated-Path-Rendering](https://on-demand.gputechconf.com/gtc/2012/presentations/S0024-GPU-Accelerated-Path-Rendering.pdf)  
[GPU-accelerated Path Rendering (2012) [pdf]](https://news.ycombinator.com/item?id=9259910)  
[why have hardware accelerated vector graphics not taken off](https://softwareengineering.stackexchange.com/questions/191472/why-have-hardware-accelerated-vector-graphics-not-taken-off)  
