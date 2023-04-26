---
layout: post
title: "mp3精准seek与比特池技术"
date: 2020-04-11 16:00:20 +0800
comments: true
categories: ffmpeg seek 精准 audio mp3
---

> ffmpeg 的 seek flag AVSEEK_FLAG_ANY 并不精准。

### 起因
最近在做音频剪辑的功能，有下面的场景

一段音频，一个时间区间将它分成三段，前段和后段速度保持不变，中间一段变速2倍。  

实现上，我分成了三个不同的 segment 来处理，segment.start 不等于 0 的，会执行一下 seek，使用的是 ffmpeg 的 `AVSEEK_FLAG_ANY | AVSEEK_FLAG_BACKWARD`，来精准 seek，完成之后发现段与段交接的地方声音并不连贯。

#### 剪映
研究了竞品，发现也有这个问题，[剪映编辑音乐录屏](/images/20200411/luping.MP4)  
分析了一下视频的波形，可以看到在 11s 的位置有缝。

<div align=center>
<img src="/images/20200411/luping_boxing.png" width="300"/>
</div>


#### 裁剪 frame
我已经做了一个处理，在段结尾的时候，裁掉多余的bytes，在段开始的时候也裁掉，保证段与段之间解码后的数据连续。但是声音还是不连续。

```cpp
std::shared_ptr<SampleData> AudioSegmentReader::copyNextSample() {  
    if (currentLength >= endLength) {  
        return nullptr;  
    }  
    auto data = copyNextSampleInternal();  
    if (data == nullptr) {  
        return nullptr;  
    }  
    // 裁掉结尾多余的 bytes
    data->length = std::min(data->length, endLength -   currentLength);  
    currentLength += data->length;  
    return data;  
}

// 解码出的数据判断是否需要裁掉开头的 bytes
data = decoder->onRenderFrame();
auto time = decoder->currentPresentationTime();
if (0 <= time && time < startTime) {
    auto delta = startLength - SampleTimeToLength(time, outputSetting.get());
    if (delta < data->length) {
        data->data += delta;
        data->length -= delta;
    } else {
        data->data = nullptr;
        data->length = 0;
    }
}
```
#### 排查 packet 和 frame
打印了一下段与段连接地方的 packet 的 packetData 和 frameData，发现 packetData 正常，seek 之后的 frameData 中前面大部分是 0，和上一段结尾解出的 frameData 不一样。记得音频帧可以独立解码，不需要参考前面的帧数据，那问题出现在哪里？
> 一个测试：解封装连续，解码之前 flush 一下 decoder，会发现 frameData 前面都有0，和不 flush decoder 的情况不一样。

<div align=center>
<img src="/images/20200411/decode_data.png" width="500"/>
</div>

#### 了解 mp3 帧头格式
很多规则，但是没卵用。

#### 比特池技术(bit reservoir)
最后去查 mp3 的解码过程实现，发现 mp3 使用了比特池技术，当前帧的主数据可能放在上一帧。。。。也就是要实现精准 seek，得往前多 seek 几帧，然后把前面的 frame 丢掉。
试了一下，结果如预期。

<div align=center>
<img src="/images/20200411/mp3_bitstream.png" width="300"/>
</div>

#### 结尾
最后放一个修复前后的波形对比图。
第一条波形是一个 Segment 时候的波形；
第二条是从中间剪开，两个 Segment 的波形，会发现中间有缝；
第三条是修复后的波形。

<div align=center>
<img src="/images/20200411/wave_compare.png" width="300"/>
</div>

### 参考
[mp3比特池技术](https://blog.csdn.net/jgdu1981/article/details/6757498)  
[功耗高集成度MP3解码器IP核设计](http://journal2.cqupt.edu.cn/jcuptnse/html/2013/1673-825X-25-4-494.html)  
[维基百科-MP3](https://en.wikipedia.org/wiki/MP3)  
[维基百科-AAC](https://en.wikipedia.org/wiki/Advanced_Audio_Coding)

