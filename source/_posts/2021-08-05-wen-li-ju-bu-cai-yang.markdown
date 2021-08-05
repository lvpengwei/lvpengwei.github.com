---
layout: post
title: "纹理局部采样"
date: 2021-08-05 17:21:42 +0800
comments: true
categories: 纹理 局部 采样 subregion texture sampling 裁剪 crop
---

## 现象
在使用 MediaCodec 解码视频获取到纹理时，它会给出一个 cropRect 来裁剪多余的绿色像素。当拿着这个纹理和对应的 cropRect 去上屏的时候，发现在边缘的地方有一像素绿边。

如图所示，解码出的纹理大小是 `1920*1088`，有 8 像素的绿边；裁剪后大小是 `1920*1080`，有 1 像素绿边。

#### 解码图片
![](/images/20210805/decoded_image.png =480x)

#### 裁剪后
![](/images/20210805/crop_image.png =480x)

## 纹素和像素的映射关系
一开始怀疑是纹素和像素的坐标系不一致的问题，对纹理坐标减了 0.5，发现还是有绿边。然后还找到坐标系不一致的问题只存在于 D3D9，后续的 D3D10 修改了坐标系的对应关系，而且 OpenGL 的坐标系一直没这个问题。

## 收缩 0.5 纹素
在[OpenGL ES Texture Coordinates Slightly Off](https://stackoverflow.com/questions/6023400/opengl-es-texture-coordinates-slightly-off)上看到说只有当采样的点在纹素中心，才返回准确的颜色，否则就是插值出来的。也就是当采样的点在纹素中心和边界之间时，可能就会采到超出边界的颜色。

## 查 Android 源码
同时也发现使用`SurfaceTexture.getTransformMatrix`得到的 matrix 时，画面是正常的，所以去查看了 Android 的源码，想知道这个 matrix 是怎么生成的。

生成的逻辑就是下面这段代码，可以看到注释说为了防止双线性采样超过裁剪边缘，普通纹理需要收缩 0.5 纹素，YUV420的要收缩 1.0 纹素。

```cpp
......
......
void SurfaceTexture::computeTransformMatrix(float outTransform[16], const sp<GraphicBuffer>& buf,
                                            const Rect& cropRect, uint32_t transform,
                                            bool filtering) {
	......
    if (!cropRect.isEmpty() && buf.get()) {
        float tx = 0.0f, ty = 0.0f, sx = 1.0f, sy = 1.0f;
        float bufferWidth = buf->getWidth();
        float bufferHeight = buf->getHeight();
        float shrinkAmount = 0.0f;
        if (filtering) {
            // In order to prevent bilinear sampling beyond the edge of the
            // crop rectangle we may need to shrink it by 2 texels in each
            // dimension.  Normally this would just need to take 1/2 a texel
            // off each end, but because the chroma channels of YUV420 images
            // are subsampled we may need to shrink the crop region by a whole
            // texel on each side.
            switch (buf->getPixelFormat()) {
                case PIXEL_FORMAT_RGBA_8888:
                case PIXEL_FORMAT_RGBX_8888:
                case PIXEL_FORMAT_RGBA_FP16:
                case PIXEL_FORMAT_RGBA_1010102:
                case PIXEL_FORMAT_RGB_888:
                case PIXEL_FORMAT_RGB_565:
                case PIXEL_FORMAT_BGRA_8888:
                    // We know there's no subsampling of any channels, so we
                    // only need to shrink by a half a pixel.
                    shrinkAmount = 0.5;
                    break;

                default:
                    // If we don't recognize the format, we must assume the
                    // worst case (that we care about), which is YUV420.
                    shrinkAmount = 1.0;
                    break;
            }
        }

        // Only shrink the dimensions that are not the size of the buffer.
        if (cropRect.width() < bufferWidth) {
            tx = (float(cropRect.left) + shrinkAmount) / bufferWidth;
            sx = (float(cropRect.width()) - (2.0f * shrinkAmount)) / bufferWidth;
        }
        if (cropRect.height() < bufferHeight) {
            ty = (float(bufferHeight - cropRect.bottom) + shrinkAmount) / bufferHeight;
            sy = (float(cropRect.height()) - (2.0f * shrinkAmount)) / bufferHeight;
        }

        mat4 crop(sx, 0, 0, 0, 0, sy, 0, 0, 0, 0, 1, 0, tx, ty, 0, 1);
        xform = crop * xform;
    }
	......
}
......
......
```
再看一遍[SurfaceTexture.getTransformMatrix](https://developer.android.com/reference/android/graphics/SurfaceTexture#getTransformMatrix(float[]))发现也有说明。

![](/images/20210805/getTransformMatrix.png =600x)

## 链接
[OpenGL ES Texture Coordinates Slightly Off](https://stackoverflow.com/questions/6023400/opengl-es-texture-coordinates-slightly-off)  
[SurfaceTexture::computeTransformMatrix](https://cs.android.com/android/platform/superproject/+/master:frameworks/native/libs/nativedisplay/surfacetexture/SurfaceTexture.cpp;l=275;drc=master;bpv=0;bpt=1)  
[SurfaceTexture.getTransformMatrix](https://developer.android.com/reference/android/graphics/SurfaceTexture#getTransformMatrix(float[]))  
[图形学底层探秘 - 纹理采样、环绕、过滤与Mipmap的那些事](https://zhuanlan.zhihu.com/p/143377682)  
[Directly Mapping Texels to Pixels (Direct3D 9)](https://docs.microsoft.com/en-us/windows/win32/direct3d9/directly-mapping-texels-to-pixels)  
