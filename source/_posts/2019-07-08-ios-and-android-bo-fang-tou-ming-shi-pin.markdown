---
layout: post
title: "iOS&amp;Android 播放透明视频"
date: 2019-07-08 21:49:54 +0800
comments: true
categories: 
---

在 iOS & Android 上播放带 alpha 的 mp4 视频，用来代替序列帧动画(png/webp)，素材小，播放也流畅。

用滤镜就可以完成，Demo 地址：  
[iOS](https://github.com/lvpengwei/AlphaVideoiOS)  
[Android](https://github.com/lvpengwei/ExoPlayerFilter)

[如何使用 AE 制作带 alpha 的 mp4](https://felgo.com/doc/felgo-alphavideo/#how-to-create-an-alpha-video)

原理：因为视频只有 rgb，没有 a，所以制作两个视频，第一个是原视频，第二个是纯黑白的边界视频，然后把两个视频合成一个，目的是保证帧对应。第一个视频的 rgb 是 0~1 的 float 值，第二个视频的 rgb 要么全是 1，要么全是 0。在客户端进行渲染的时候，写一个滤镜，目标像素 rgba 在取值的时候，rgb 使用第一个视频的 rgb，a 使用第二个视频的 r，就可以完成播放透明视频。  
```
gl_FragColor = vec4(color1.rgb, color2.r);
```
