---
layout: post
title: "iOS NV12转SkImage颜色不正常的问题"
date: 2020-03-07 11:01:32 +0800
comments: true
categories: skia VideoToolbox skimage nv12 ios yuv
---

# 环境
设备：iPhone 6s  
系统：13.1  
Skia版本：m62  
视频的YUV ColorSpace：ITU-R BT.601  

# 现象
VideoToolbox 配置的 pixelFormat 是`kCVPixelFormatType_420YpCbCr8BiPlanarVideoRange`，然后把输出的 pixelBuffer 用下面的代码片段1转成 NV12，再使用代码片段2转成 SkImage，在 SkCanvas 上 draw 出来如图1，视频原图如图2。

```objc
uint32_t pixelFormatType = kCVPixelFormatType_420YpCbCr8BiPlanarVideoRange;
```
```objc
// 代码片段1
// Y 数据
CVOpenGLESTextureCacheCreateTextureFromImage(kCFAllocatorDefault,
                                             textCache,
                                             pixelBuffer,
                                             NULL,
                                             GL_TEXTURE_2D,
                                             GL_LUMINANCE,
                                             width,
                                             height,
                                             GL_LUMINANCE,
                                             GL_UNSIGNED_BYTE,
                                             0,
                                             &outputTextureLuma);
// UV 数据
CVOpenGLESTextureCacheCreateTextureFromImage(kCFAllocatorDefault,
                                             textCache,
                                             pixelBuffer,
                                             NULL,
                                             GL_TEXTURE_2D,
                                             GL_LUMINANCE_ALPHA,
                                             width / 2,
                                             height / 2,
                                             GL_LUMINANCE_ALPHA,
                                             GL_UNSIGNED_BYTE,
                                             1,
                                             &outputTextureChroma);
```

```cpp
// 代码片段2
GrGLTextureInfo textureInfo1 = {videoImage->textureTarget(), videoImage->getTextureID(0)};
GrGLTextureInfo textureInfo2 = {videoImage->textureTarget(), videoImage->getTextureID(1)};
GrBackendObject nv12TextureHandles[] = {reinterpret_cast<GrBackendObject>(&textureInfo1),
                                        reinterpret_cast<GrBackendObject>(&textureInfo2)};
SkISize nv12Sizes[] = \{\{videoImage->width(), videoImage->height()\},
                       \{videoImage->width(), videoImage->height()\}\};
skImage = SkImage::MakeFromNV12TexturesCopy(grContext,
                                            kRec601_SkYUVColorSpace,
                                            nv12TextureHandles,
                                            nv12Sizes,
                                            kTopLeft_GrSurfaceOrigin,
                                            nullptr);
```
![图1](/images/20200307/nv12_ra.png)
![图2](/images/20200307/nv12_rg.png)


# 查问题
#### 1. 查视频的 YUV ColorSpace 是否和 SkImage 对应
是一致的，但输出的图像还是有问题。

### 2.试试把 VideoToolbox 的输出格式换成 RGBA
配置 VideoToolbox 的 pixelFormat 为 `kCVPixelFormatType_32BGRA`，使用代码片段3把 pixelBuffer 转成 RGBA 纹理，然后使用代码片段4转成 SkImage，图像是正常的。
```objc
uint32_t pixelFormatType = kCVPixelFormatType_32BGRA;
```
```objc
// 代码片段3
CVOpenGLESTextureCacheCreateTextureFromImage(kCFAllocatorDefault,
                                             textCache,
                                             pixelBuffer,
                                             NULL,
                                             GL_TEXTURE_2D,
                                             GL_RGBA,
                                             width,
                                             height,
                                             GL_BGRA,
                                             GL_UNSIGNED_BYTE,
                                             0,
                                             &outputTextureLuma);
```
```cpp
// 代码片段4
GrGLTextureInfo textureInfo = {videoImage->textureTarget(), videoImage->getTextureID(0)};
GrBackendTexture backendTexture(videoImage->width(), videoImage->height(), kRGBA_8888_GrPixelConfig,
                                textureInfo);
skImage =  SkImage::MakeFromTexture(grContext, backendTexture, kTopLeft_GrSurfaceOrigin,
                                    kPremul_SkAlphaType, nullptr);
```

### 3.查 Skia 源码
```cpp
// SkImage_Gpu.cpp
// SkImage::MakeFromNV12TexturesCopy -> make_from_yuv_textures_copy
// GrYUVEffect.cpp
// GrYUVEffect::MakeYUVToRGB -> YUVtoRGBEffect::Make -> YUVtoRGBEffect() -> onCreateGLSLInstance() -> GLSLProcessor -> shader '.rg'
```
从 Skia 的源码中一直跟下去，发现最后 shader 使用的是 `rg` 通道，而因为我们是用 `GL_LUMINANCE_ALPHA` 来获取 `UV` 数据，在 GLSL 中应该使用 `ra` 通道，所以出现了不一致。当使用`GL_RG`获取`UV`数据的时候（代码片段5），SkImage 输出的图片就正常了。
```objc
// 代码片段5
// UV 数据
CVOpenGLESTextureCacheCreateTextureFromImage(kCFAllocatorDefault,
                                             textCache,
                                             pixelBuffer,
                                             NULL,
                                             GL_TEXTURE_2D,
                                             GL_RG,
                                             width / 2,
                                             height / 2,
                                             GL_RG,
                                             GL_UNSIGNED_BYTE,
                                             1,
                                             &outputTextureChroma);
```

# Link
[skia](https://github.com/google/skia)  
[GL 移植到 Metal 的小细节](https://juejin.im/entry/5cbac68c6fb9a0688c039ebc)
