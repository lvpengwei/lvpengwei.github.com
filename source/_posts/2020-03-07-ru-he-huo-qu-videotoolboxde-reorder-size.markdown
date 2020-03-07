---
layout: post
title: "如何获取VideoToolbox的reorder size"
date: 2020-03-07 11:17:28 +0800
comments: true
categories: VideoToolbox h264 h265 hevc reorder
---
# Decoder 的区别
FFmpeg 和 MediaCodec 解码的时候，送数据的顺序是 dts，出数据的顺序是 pts，而 VideoToolbox 是送一个出一个，没有按照 pts 来出数据，需要我们自己排序。

去网上查资料的时候，发现有很多不同的方式

 1. sps.max_num_ref_frames
 2. sps.vui.max_num_reorder_frames
 3. 通过 sps.level 计算
 4. 直接设置为4

通过测试几个文件的 sps 发现 `max_num_ref_frames` 不是很准

 1. `max_num_ref_frames=0; max_num_reorder_frames=2`
 2. `max_num_ref_frames=9; max_num_reorder_frames=2`

# sps.max_num_ref_frames
取 `max_num_ref_frames` 的有两个播放器，ijkplayer 和 ThumbPlayer
## ijkplayer
ijkplayer 的逻辑是先取 `sps.max_num_ref_frames`，然后再取最小值2，最大值5。
```objc
fmt_desc->max_ref_frames = FFMAX(fmt_desc->max_ref_frames, 2);

fmt_desc->max_ref_frames = FFMIN(fmt_desc->max_ref_frames, 5);
```
主要代码在下面两个文件。  
[IJKVideoToolBoxAsync.m](https://github.com/bilibili/ijkplayer/blob/master/ios/IJKMediaPlayer/IJKMediaPlayer/ijkmedia/ijkplayer/ios/pipeline/IJKVideoToolBoxAsync.m#L1136)  
[h264_sps_parser.h](https://github.com/bilibili/ijkplayer/blob/cced91e3ae3730f5c63f3605b00d25eafcf5b97b/ios/IJKMediaPlayer/IJKMediaPlayer/ijkmedia/ijkplayer/ios/pipeline/h264_sps_parser.h#L267)

## ThumbPlayer
ThumbPlayer 的逻辑是取 `sps.max_num_ref_frames`，如果没有设置为  10。

# sps.vui.max_num_reorder_frames
取 `max_num_reorder_frames` 的也有两个，[Chrome](https://cs.chromium.org/) 和 [vlc](https://github.com/videolan/vlc)。

## Chrome
Chrome 的主要代码如下，代码文件在[vt_video_decode_accelerator_mac.cc](https://cs.chromium.org/chromium/src/media/gpu/mac/vt_video_decode_accelerator_mac.cc?dr=CSs&q=vt_video_decode_accelerator_mac&g=0&l=262) 。

逻辑是先判断 `pocType`，为 2 直接返回不需要排序
再判断是否有 `vuiParameters`，取 `max_num_reorder_frames`
最后是特定的 profile，不需要排序
```c
int32_t ComputeReorderWindow(const H264SPS* sps) {
  // When |pic_order_cnt_type| == 2, decode order always matches presentation
  // order.
  // TODO(sandersd): For |pic_order_cnt_type| == 1, analyze the delta cycle to
  // find the minimum required reorder window.
  if (sps->pic_order_cnt_type == 2)
    return 0;

  // TODO(sandersd): Compute MaxDpbFrames.
  int32_t max_dpb_frames = 16;

  // See AVC spec section E.2.1 definition of |max_num_reorder_frames|.
  if (sps->vui_parameters_present_flag && sps->bitstream_restriction_flag) {
    return std::min(sps->max_num_reorder_frames, max_dpb_frames);
  } else if (sps->constraint_set3_flag) {
    if (sps->profile_idc == 44 || sps->profile_idc == 86 ||
        sps->profile_idc == 100 || sps->profile_idc == 110 ||
        sps->profile_idc == 122 || sps->profile_idc == 244) {
      return 0;
    }
  }
  return max_dpb_frames;
}
```

## vlc
vlc 的逻辑和 chrome 类似，多了一个根据 level 计算
代码文件在[h264_nal.c](https://github.com/videolan/vlc/blob/d7ff28e96eb2cb64c5b1a502443a24229532a449/modules/packetizer/h264_nal.c#L735)
```c
static uint8_t h264_get_max_dpb_frames( const h264_sequence_parameter_set_t *p_sps )
{
    const h264_level_limits_t *limits = h264_get_level_limits( p_sps );
    if( limits )
    {
        unsigned i_frame_height_in_mbs = ( p_sps->pic_height_in_map_units_minus1 + 1 ) *
                                         ( 2 - p_sps->frame_mbs_only_flag );
        unsigned i_den = ( p_sps->pic_width_in_mbs_minus1 + 1 ) * i_frame_height_in_mbs;
        uint8_t i_max_dpb_frames = limits->i_max_dpb_mbs / i_den;
        if( i_max_dpb_frames < 16 )
            return i_max_dpb_frames;
    }
    return 16;
}

bool h264_get_dpb_values( const h264_sequence_parameter_set_t *p_sps,
                          uint8_t *pi_depth, unsigned *pi_delay )
{
    uint8_t i_max_num_reorder_frames = p_sps->vui.i_max_num_reorder_frames;
    if( !p_sps->vui.b_bitstream_restriction_flag )
    {
        switch( p_sps->i_profile ) /* E-2.1 */
        {
            case PROFILE_H264_BASELINE:
                i_max_num_reorder_frames = 0; /* only I & P */
                break;
            case PROFILE_H264_CAVLC_INTRA:
            case PROFILE_H264_SVC_HIGH:
            case PROFILE_H264_HIGH:
            case PROFILE_H264_HIGH_10:
            case PROFILE_H264_HIGH_422:
            case PROFILE_H264_HIGH_444_PREDICTIVE:
                if( p_sps->i_constraint_set_flags & H264_CONSTRAINT_SET_FLAG(3) )
                {
                    i_max_num_reorder_frames = 0; /* all IDR */
                    break;
                }
                /* fallthrough */
            default:
                i_max_num_reorder_frames = h264_get_max_dpb_frames( p_sps );
                break;
        }
    }

    *pi_depth = i_max_num_reorder_frames;
    *pi_delay = 0;

    return true;
}
```

[gst-plugins-bad](https://github.com/GStreamer/gst-plugins-bad/blob/bc128d610063a266a1b715e5a696ca252f2d5a74/sys/applemedia/vtdec.c#L962) 也是通过 level 计算。


# 设置为4
[iOS解码关于视频中带B帧排序问题](https://juejin.im/post/5d1385c751882532531bca11)


# HEVC
vlc 中还有 `HEVC(H265)` 视频获取 `max_num_reorder` 的方式，代码文件在[hevc_nal.c](https://github.com/videolan/vlc/blob/b525b27e85f1f2cec0fe9b38e08f5dee698a893e/modules/packetizer/hevc_nal.c#L1111)

# Link
[ijkplayer](https://github.com/bilibili/ijkplayer/blob/cced91e3ae3730f5c63f3605b00d25eafcf5b97b/ios/IJKMediaPlayer/IJKMediaPlayer/ijkmedia/ijkplayer/ios/pipeline/h264_sps_parser.h#L267)  
[Chrome](https://cs.chromium.org/)  
[vlc](https://github.com/videolan/vlc)  
[iOS解码关于视频中带B帧排序问题](https://juejin.im/post/5d1385c751882532531bca11)  