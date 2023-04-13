---
layout: post
title: "PAG 支持 web 做了哪些事"
date: 2023-04-13 22:51:19 +0800
comments: true
categories: PAG web wasm webassembly emscripten
styles: data-table
---

## 思路
PAG 是纯 C++ 的项目，所以我们可以尝试通过 WebAssembly 在浏览器中运行。

首先我们的目标是先跑通一个纯矢量的 PAG 文件。

### 1. 用 freetype 跑通矢量绘制
我们需要用 emscripten 把 PAG 打成 wasm，PAG 的依赖库有很多，比如 ffmpeg、libpng、libjpeg、libwebp、zlib、pathkit、freetype、opengl 等，要跑通纯矢量的绘制，我们需要一个 OpenGL ES 的环境，再链接 pathkit 和 freetype 这两个库，其他的可以先不管。

寻找 OpenGL ES 的过程绕了一些弯路，不过万幸找到 emscripten 提供了 OpenGL ES 的 API，背后是 webgl 的实现。

wasm 链接第三方库也是 .a 的后缀，不过要用`emcamke`来生成 makefile，它会带入 emscripten 的环境变量，再去 build 就可以得到 wasm 支持的 .a。

把这两个库编译完，还需要一个 binding 文件来桥接 js 和 c++ 的代码，最后用`emcc`把 libpag.a、pathkit.a、freetype.a、binding.cpp 链接在一起生成 wasm 文件。

### 2. 视频序列帧
PAG 在其他平台是通过解码器来解码视频，web 平台不提供视频解码器，所以我们把 PAG 里面的裸 h264 流封装成 mp4 再放到 video 标签中播放，通过 seek 来控制进度，通过 [txtImage2D](https://developer.mozilla.org/zh-CN/docs/Web/API/WebGLRenderingContext/texImage2D)来上传[HTMLVideoElement](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLVideoElement)的内容。

HTMLVideoElement 的 seek 是真的 seek，它没有做任何优化，所以当时间在一个 GOP 结尾的时候，HTMLVideoElement 的 seek 耗时非常久。因为 web 端的 PAG 只用于播放，不发生导出，所以这里画面没有对上也没关系，我们采用让它 play 起来，当下次请求时判断它当前的时间和我们请求的时间是否超过一个阈值，没有超过就不发生 seek；当有一段时间没有发生请求，就会自动暂停。

### 3. 图片解码
web 端对包大小很敏感，所以要尽量减少第三方库的依赖，像 png、jpg 的解码，web 的 image 标签就可以做到，然后通过`txtImage2D`上传到纹理，而 webp 因为平台的原因，浏览器不一定支持，这个库就不能去掉。

### 4. 文字
一开始我们用 freetype 来适配 web 端的文字渲染，搞完之后发现，web 没法提供系统字体的路径，如果用 freetype 来处理字体，需要在服务器上配置字体文件，web 端去下载注册到 freetype 中，而中文字体文件一般比较大，macOS 的苹方字体有 100M+，显然用户体验不会很好。我们去查了 flutter-web 的实现，它用的是 skia 的 web 版本，叫 [CanvasKit](https://skia.org/docs/user/modules/canvaskit/)，他们也是先下载字体然后注册之后使用。
我们调研了一下，可以用 web-font 来加载系统字体，用 web-canvas 来渲染，path 也可以用 web-canvas 渲染，这样 freetype 依赖就可以去掉，包又小了一点。

### 5. 包大小
做完以上这些，PAG 适配 web 端基本完成了，测试了一下包大小  
<table>
   <tr>
      <td></td>
      <td>size</td>
      <td>gzip</td>
   </tr>
    <tr>
      <td>CanvasKit</td>
      <td>6.6M</td>
      <td>2.7M</td>
   </tr>
   <tr>
      <td>pag</td>
      <td>2.2M</td>
      <td>643K</td>
   </tr>
</table>

### 6. 性能
上面的弄完之后，发现每帧耗时都比较高，要 30ms+，通过浏览器的 Performance 工具发现是 OpenGL 调用耗时比较高，查看了 emscripten 的文章 [Optimizing WebGL](https://emscripten.org/docs/optimizing/Optimizing-WebGL.html)，按照上面的建议逐个排查，去掉 `glGet*`、`glGetError`、`glCheckFramebufferStatus`之后，每帧耗时明显降低。

### 7. PixiJS 
之前的封装是基于 canvas 的，从 web 的 canvas 中创建`webgl`的 context，然后 PAG 在这个 context 中渲染。但是业务方是 web 端的视频编辑场景，可能加载很多个 PAG，webgl context 超出了浏览器的上限。

因为业务方使用的 [PixiJS](https://pixijs.com/) 本身就有一个 context，所以我们想直接共用一个 context，不再重新创建。

通过调查，[PIXI.Resource](https://pixijs.download/release/docs/PIXI.Resource.html)可以做到这件事，在回调方法[`upload`](https://pixijs.download/release/docs/PIXI.Resource.html#upload)中，PixiJS 会传入`PIXI.Renderer`和`PIXI.GLTexture`，通过`PIXI.Renderer`我们可以拿到 webgl 的 context，通过`PIXI.GLTexture`我们可以拿到 webgl 的 texture，我们再把 context 和 texture 注册到 emscripten 的 GL 中，再用注册后的 texture 去创建 PAGSurface，就可以完成渲染。

这里要注意的是，`upload`传进来的 texture 可能会发生改变，所以在发现 texture 改变的时候，要从 emscripten 的 GL 中解注册，重新注册一个新的 texture，再创建一个新的 PAGSurface 去渲染。

调整进度接口直接写在这个`PIXI.Resource`的子类里面，再调用一下`update`方法，等 PixiJS 回调`upload`。

示例代码如下
```ts
import { Resource } from 'pixi.js';  
  
class PAGResource extends Resource {  
  static async create(PAG, pagFile) {  
    const width = await pagFile.width();  
    const height = await pagFile.height();  
    const pagResource = new PAGResource(width, height);  
    pagResource.pagPlayer = await PAG.PAGPlayer.create();  
    await pagResource.pagPlayer.setComposition(pagFile);  
    pagResource.module = PAG;  
    return pagResource;  
  }  
  
  private module;  
  private contextID = null;  
  private textureID = null;  
  private pagPlayer = null;  
  private pagSurface = null;  
  
  constructor(width, height) {  
    super(width, height);  
  }  
  
  async upload(renderer, baseTexture, glTexture) {  
    const { width } = this;  
    const { height } = this;  
    glTexture.width = width;  
    glTexture.height = height;  
  
    const { gl } = renderer;  
  
    // 注册 context  
    if (this.contextID === null) {  
      this.contextID = this.module.GL.registerContext(gl, { majorVersion: 2, minorVersion: 0 });  
    }  
    
    if (glTexture.texture.name !== this.textureID) {  
      // texture 变化  
      if (this.textureID !== null) {  
        // 销毁旧的 surface  
        this.module.GL.textures[this.textureID] = null;  
        this.pagSurface.destroy();  
      }  
      // 分配内存不然绑定 frameBuffer 会失败  
    gl.texImage2D(  
        baseTexture.target,  
        0,  
        baseTexture.format,  
        width,  
        height,  
        0,  
        baseTexture.format,  
        baseTexture.type,  
        null,  
      );  
      // 注册  
      this.textureID = this.module.GL.getNewId(this.module.GL.textures);  
      glTexture.texture.name = this.textureID;  
      this.module.GL.textures[this.textureID] = glTexture.texture;  
      // 生成 surface  
      this.module.GL.makeContextCurrent(this.contextID);  
      this.pagSurface = await this.module._PAGSurface.FromTexture(this.textureID, width, height, false);  
      await this.pagPlayer.setSurface(this.pagSurface);  
    }  
    await this.pagPlayer.flush();  
    renderer.reset();  
    return true;  
  }  
  
  public async setProgress(progress) {  
    await this.pagPlayer.setProgress(progress);  
    this.update();  
  }  
}
```

## 链接
[WebAssembly](https://developer.mozilla.org/zh-CN/docs/WebAssembly)  
[emscripten](https://emscripten.org/)  
[PixiJS](https://pixijs.com/)  
