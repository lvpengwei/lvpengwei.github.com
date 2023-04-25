---
layout: post
title: "tgfx 实现 Blend"
date: 2023-04-25 22:24:31 +0800
comments: true
categories: skia blend 混合模式 tgfx
styles: data-table
---

## 问题
PAG 文件里的混合模式是从 AE 中导出的，然后使用 skia 内置的混合模式去实现，去掉 skia 之后，需要用原生的 OpenGL 来实现。

## 思路
既然 skia 内置的混合模式可以满足需求，那我们用 OpenGL 实现 skia 支持的就好，先看一下 skia 里面都有哪些混合模式。
```cpp
enum class SkBlendMode {  
  kClear, //!< replaces destination with zero: fully transparent  
  kSrc, //!< replaces destination  
  kDst, //!< preserves destination  
  kSrcOver, //!< source over destination  
  kDstOver, //!< destination over source  
  kSrcIn, //!< source trimmed inside destination  
  kDstIn, //!< destination trimmed by source  
  kSrcOut, //!< source trimmed outside destination  
  kDstOut, //!< destination trimmed outside source  
  kSrcATop, //!< source inside destination blended with destination  
  kDstATop, //!< destination inside source blended with source  
  kXor, //!< each of source and destination trimmed outside the other  
  kPlus, //!< sum of colors  
  kModulate, //!< product of premultiplied colors; darkens destination  
  kScreen, //!< multiply inverse of pixels, inverting result; brightens destination  
  kLastCoeffMode = kScreen, //!< last porter duff blend mode  
  kOverlay, //!< multiply or screen, depending on destination  
  kDarken, //!< darker of source and destination  
  kLighten, //!< lighter of source and destination  
  kColorDodge, //!< brighten destination to reflect source  
  kColorBurn, //!< darken destination to reflect source  
  kHardLight, //!< multiply or screen, depending on source  
  kSoftLight, //!< lighten or darken, depending on source  
  kDifference, //!< subtract darker from lighter with higher contrast  
  kExclusion, //!< subtract darker from lighter with lower contrast  
  kMultiply, //!< multiply source with destination, darkening image  
  kLastSeparableMode = kMultiply, //!< last blend mode operating separately on components  
  kHue, //!< hue of source with saturation and luminosity of destination  
  kSaturation, //!< saturation of source with hue and luminosity of destination  
  kColor, //!< hue and saturation of source with luminosity of destination  
  kLuminosity, //!< luminosity of source with hue and saturation of destination  
  kLastMode = kLuminosity, //!< last valid value  
};
```
![blend mode](/images/20230425/blend.png)

从实现方案上来说，这些混合模式都可以用 shader 来完成。其中`kLastCoeffMode`以上的混合模式也叫 [PorterDuff 混合模式](https://zh.wikipedia.org/wiki/Alpha%E5%90%88%E6%88%90)，它们还可以用 OpenGL 提供的 `glBlendFunc`来实现 。

## 解决
### coeff blend mode 
`kLastCoeffMode`以上的比较容易，在渲染之前把对应的参数设置好就行。
```cpp
static constexpr std::pair<Blend, std::pair<unsigned, unsigned>> kBlendCoeffMap[] = {  
  {Blend::Clear, {GL_ZERO, GL_ZERO}},  
  {Blend::Src, {GL_ONE, GL_ZERO}},  
  {Blend::Dst, {GL_ZERO, GL_ONE}},  
  {Blend::SrcOver, {GL_ONE, GL_ONE_MINUS_SRC_ALPHA}},  
  {Blend::DstOver, {GL_ONE_MINUS_DST_ALPHA, GL_ONE}},  
  {Blend::SrcIn, {GL_DST_ALPHA, GL_ZERO}},  
  {Blend::DstIn, {GL_ZERO, GL_SRC_ALPHA}},  
  {Blend::SrcOut, {GL_ONE_MINUS_DST_ALPHA, GL_ZERO}},  
  {Blend::DstOut, {GL_ZERO, GL_ONE_MINUS_SRC_ALPHA}},  
  {Blend::SrcATop, {GL_DST_ALPHA, GL_ONE_MINUS_SRC_ALPHA}},  
  {Blend::DstATop, {GL_ONE_MINUS_DST_ALPHA, GL_SRC_ALPHA}},  
  {Blend::Xor, {GL_ONE_MINUS_DST_ALPHA, GL_ONE_MINUS_SRC_ALPHA}},  
  {Blend::Plus, {GL_ONE, GL_ONE}},  
  {Blend::Modulate, {GL_ZERO, GL_SRC_COLOR}},  
  {Blend::Screen, {GL_ONE, GL_ONE_MINUS_SRC_COLOR}}};

glEnable(GL_BLEND);  
glBlendFunc(first, second);  
glBlendEquation(GL_FUNC_ADD);
```

### shader blend mode
用 shader 实现混合模式，我们需要在 shader 中访问当前  frame buffer 上的颜色分量，OpenGL 有一些 extension 提供了 frame buffer fetch 的功能，如下表所示。
<table>
   <tr>
      <td>extension</td>
      <td>color name</td>
   </tr>
    <tr>
      <td>GL_EXT_shader_framebuffer_fetch</td>
      <td>gl_LastFragData[0]</td>
   </tr>
   <tr>
      <td>GL_NV_shader_framebuffer_fetch</td>
      <td>gl_LastFragData[0]</td>
   </tr>
   <tr>
      <td>GL_ARM_shader_framebuffer_fetch</td>
      <td>gl_LastFragColorARM</td>
   </tr>
</table>
<br/>

如果当前的 OpenGL 没有提供这些 extension，我们还有一个兜底措施，把这个 frame buffer 的内容复制到一个纹理(`dstTexture`)上，再把纹理传入 shader。

![](/images/20230425/blend_dst_texture.png)

当然我们不需要把完整的 frame buffer 内容复制一份，因为我们的绘制区域可能只是局部。

复制局部 frame buffer 内容到纹理上，我们使用的是`glCopyTexSubImage2D`。这里还有一些其他的方式，比如`glBlitFramebuffer`，用这个的话，需要多创建一个 frame buffer，没有前一个方便和高效。

如果当前 frame buffer 已经绑定了一个纹理，而且当前的 OpenGL 也支持 `glTextureBarrier`，可以直接把这个绑定的纹理传入 shader，不过在绘制之前要调用一下`glTextureBarrier`。

## shader 公式
skia 的 shader 公式来源是 [w3c - Advanced compositing features](https://www.w3.org/TR/compositing-1/#advancedcompositing) 的文档。
> 注意：公式里的 RGB 是 Premultiplied 还是 Unpremultiplied。

D2D 也有一份 [blend](https://docs.microsoft.com/en-us/windows/win32/direct2d/blend) 公式。这两份基本是一样的，w3c 的更全一点。

## 总结
实现混合模式的整个过程，主要就是用 shader 实现的那部分比较复杂，因为需要考虑 OpenGL 的兼容性。

## 链接
[skia](https://github.com/google/skia)  
[best method to copy texture to texture](https://stackoverflow.com/questions/23981016/best-method-to-copy-texture-to-texture)  
[OpenGL Reading from a texture unit currently bound to a framebuffer](https://stackoverflow.com/questions/38534694/opengl-reading-from-a-texture-unit-currently-bound-to-a-framebuffer)  
[SkBlendMode Overview](https://skia.org/docs/user/api/skblendmode_overview/)  
[w3c - Advanced compositing features](https://www.w3.org/TR/compositing-1/#advancedcompositing)  
[D2D - blend](https://docs.microsoft.com/en-us/windows/win32/direct2d/blend)  
