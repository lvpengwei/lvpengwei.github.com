---
layout: post
title: "纹理边缘抗锯齿 CoverageAA"
date: 2021-12-11 12:30:34 +0800
comments: true
categories: aa 抗锯齿 纹理 边缘 skia
---

## 问题
用 OpenGL 旋转图片的时候，图片边缘会出现锯齿。

图 1 是没有做抗锯齿的时候，可以明显看到边缘的锯齿。

<div align=center>
<img src="https://s2.loli.net/2021/12/11/QsFLYTfqxGlEWBv.png" width="500"/>
<br/>
图 1
</div>

## 思路
首先想到的是 OpenGL 提供的 MSAA，但是 MSAA 占用内存比较多。然后去查了下 skia 的抗锯齿是如何实现的，发现它只是对图片边缘的 1px 做一个 alpha 从 1->0 渐变的遮罩。

如图 2 所示，矩形 abcd 是我们要绘制的区域，根据矩形的坐标向内缩 0.5px 得到矩形 P0_P1_P3_P2，向外扩 0.5px 得到矩形 P4_P5_P7_P6。内矩形里面 alpha 都是 1，外矩形边缘 alpha 都是 0，内矩形和外矩形之间 alpha 从 1->0 渐变。这样我们就对边缘做了一个逐渐消失的效果，从视觉上看，边缘的锯齿就没那么明显了。

<div align=center>
<img src="https://s2.loli.net/2021/12/05/KtvNZQPdw8SLHTq.png" width="500"/>
<br/>
图 2
</div>

## 解决
### 没有抗锯齿
在没有使用抗锯齿时，我们绘制一个矩形，提交的是 cdba 4 个顶点，2 个三角形。
```cpp
auto bounds = args.rectToDraw;  
auto normalBounds = Rect::MakeLTRB(0, 0, 1, 1);
return {  
  bounds.right, bounds.bottom, normalBounds.right, normalBounds.bottom,  
  bounds.right, bounds.top, normalBounds.right, normalBounds.top,  
  bounds.left, bounds.bottom, normalBounds.left, normalBounds.bottom,  
  bounds.left, bounds.top, normalBounds.left, normalBounds.top,  
};
```
对应的绘制命令是
```cpp
gl->drawArrays(GL_TRIANGLE_STRIP, 0, 4);
```

### 抗锯齿 CoverageAA
在使用 CoverageAA 抗锯齿时，我们绘制一个矩形，提交的是内矩形 P0P1P2P3 和外矩形 P4P5P6P7 的 8 个顶点：
```cpp
auto bounds = args.rectToDraw;
auto normalBounds = Rect::MakeLTRB(0, 0, 1, 1);

auto padding = 0.5f;  
auto insetBounds = bounds.makeInset(padding, padding);  
auto outsetBounds = bounds.makeOutset(padding, padding);  
  
auto normalPadding = Point::Make(padding / bounds.width(), padding / bounds.height());  
auto normalInset = normalBounds.makeInset(normalPadding.x, normalPadding.y);  
auto normalOutset = normalBounds.makeOutset(normalPadding.x, normalPadding.y);
return {  
  insetBounds.left, insetBounds.top, 1.0f, normalInset.left, normalInset.top,  
  insetBounds.left, insetBounds.bottom, 1.0f, normalInset.left, normalInset.bottom,  
  insetBounds.right, insetBounds.top, 1.0f, normalInset.right, normalInset.top,  
  insetBounds.right, insetBounds.bottom, 1.0f, normalInset.right, normalInset.bottom,  
  outsetBounds.left, outsetBounds.top, 0.0f, normalOutset.left, normalOutset.top,  
  outsetBounds.left, outsetBounds.bottom, 0.0f, normalOutset.left, normalOutset.bottom,  
  outsetBounds.right, outsetBounds.top, 0.0f, normalOutset.right, normalOutset.top,  
  outsetBounds.right, outsetBounds.bottom, 0.0f, normalOutset.right, normalOutset.bottom,  
};
```
转换成三角形是 30 个顶点，下面是三角形的 index 数据
```cpp
static constexpr int kIndicesPerAAFillRect = 30;
static constexpr uint16_t gFillAARectIdx[] = {  
  0, 1, 2, 1, 3, 2,  
  0, 4, 1, 4, 5, 1,  
  0, 6, 4, 0, 2, 6,  
  2, 3, 6, 3, 7, 6,  
  1, 5, 3, 3, 5, 7,  
};
```
绘制命令是
```cpp
glDrawElements(GL_TRIANGLES, kIndicesPerAAFillRect, GL_UNSIGNED_SHORT, 0);
```

## 结果

图 3 是做完抗锯齿的效果，可以看到边缘的锯齿已经没有了。

<div align=center>
<img src="https://s2.loli.net/2021/12/11/gquEYZ2hc1JM57H.png" width="500"/>
<br/>
图 3
</div>

图 4 是图 1 和 图 3 边缘对比的细节，可以看到边缘像素的过渡圆滑了很多。

<div align=center>
<img src="https://s2.loli.net/2021/12/11/hHuSMsJ8zZbvrVo.png" width="500"/>
<br/>
图 4
</div>

## 链接
[skia](https://github.com/google/skia)

