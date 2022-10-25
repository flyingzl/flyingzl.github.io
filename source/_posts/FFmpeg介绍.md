---
title: FFmpeg介绍
date: 2020-08-14 17:00:00
categories:
- 系统软件
tags: 
- 媒体处理
---


A complete, cross-platform solution to record, convert and stream audio and video.

FFmpeg是一个音视频流录制、转换的跨平台的完整开源解决方案

![](https://g.asyncoder.com/images/20200816205600-gTO9QB.jpg)



## 安装

访问 [FFmpeg官网](https://ffmpeg.org/download.html) 进行安装


### FFmpeg家族工具

- ffmpeg: 音视频编解码命令行工具

```bash
ffmpeg -i input.avi output.mp4
```

ffmpeg通过-i参数将input.avi作为输入源，然后进行转码与转封装操作，输出到output.mp4中。ffmpeg转码工作流程如下图。

![](https://pic1.zhimg.com/v2-91ae26021ae6e983befb45064669e61f_r.jpg)


- ffplay: 基于FFmpeg的播放器，可以播放本地或者网络视频流

```bash
ffplay http://ivi.bupt.edu.cn/hls/cctv1hd.m3u8
```
- ffprobe: 强大的多媒体分析工具

`ffprobe`可以从媒体文件或者媒体流程中获得媒体信息，比如音频的参数、视频的参数、媒体容器的参数信息等。

```bash
# ffprobe sheep.mp4 -hide_banner

Input #0, mov,mp4,m4a,3gp,3g2,mj2, from 'out.mp4':
  Metadata:
    major_brand     : isom
    minor_version   : 512
    compatible_brands: isomiso2avc1mp41
    encoder         : Lavf58.20.100
  Duration: 00:00:05.63, start: 0.000000, bitrate: 1845 kb/s
    Stream #0:0(und): Video: h264 (High) (avc1 / 0x31637661), yuv420p, 3360x2100, 1781 kb/s, 29.81 fps, 30 tbr, 16k tbn, 2000k tbc (default)
    Metadata:
      handler_name    : VideoHandler
    Stream #0:1(und): Audio: vorbis (mp4a / 0x6134706D), 48000 Hz, mono, fltp, 63 kb/s (default)
    Metadata:
      handler_name    : SoundHandler
```

### 一些概念

![](https://pic2.zhimg.com/v2-3e634d74b5ff73fd3bf7be9b8798dc21_r.jpg)

不少同学可能会把「编码(codec)」和「格式(format)」混为一谈，因为编码和格式的命名有不少相同的，但其实他们是两个不同的概念。为了方便理解，可以将牛奶加工和音频视频的编码和格式做个对比。

![](https://pic3.zhimg.com/v2-979052f127337ce1820f13a1b21dac52_r.jpg)


## 功能展现

### 使用方式

```bash
ffmpeg -h

Hyper fast Audio and Video encoder
usage: ffmpeg [options] [[infile options] -i infile]... {[outfile options] outfile}...

```
使用风格为 `ffmpeg {1} {2} -i {3} {4} {5}`，这五个部分的参数依次如下：

- 全局参数
- 输入文件参数
- 输入文件
- 输出文件参数
- 输出文件

也就是 `ffmpeg [全局参数] [输入文件参数] -i [输入文件] [输出文件参数] [输出文件]`

```bash
$ ffmpeg \
-y \ # 全局参数
-c:a libfdk_aac -c:v libx264 \ # 输入文件参数
-i input.mp4 \ # 输入文件
-c:v libvpx-vp9 -c:a libvorbis \ # 输出文件参数
output.webm # 输出文件
```

### 查看格式容器（Muxer and Demuxer）

```bash
ffmpeg -muxers
ffmpeg -demuxers
```

`mp4`、`mkv`、`webm`、`avi`、`ogg`等都是格式容器

### 查看编码器和解码器

```bash
ffmpeg -encoders

ffmpeg -decoders
```
`H.264`、`H.265`、`VP8`、`VP9`、`MP3`、`AAC` 都是编码格式

### 录屏

https://trac.ffmpeg.org/wiki/Capture/Capture/Desktop%E4%B8%AD%E6%96%87%E7%89%88%E6%9C%AC

以Mac系统为例

1. 查看系统设备

```bash
# ffmpeg -f avfoundation -list_devices true -hide_banner  -i ""

[AVFoundation indev @ 0x7fe828e07c40] AVFoundation video devices:
[AVFoundation indev @ 0x7fe828e07c40] [0] FaceTime高清摄像头（内建）
[AVFoundation indev @ 0x7fe828e07c40] [1] Capture screen 0
[AVFoundation indev @ 0x7fe828e07c40] AVFoundation audio devices:
[AVFoundation indev @ 0x7fe828e07c40] [0] Screenflick Loopback
[AVFoundation indev @ 0x7fe828e07c40] [1] 外置麦克风
[AVFoundation indev @ 0x7fe828e07c40] [2] MacBook Pro麦克风
[AVFoundation indev @ 0x7fe828e07c40] [3] ZoomAudioDevice
```

2. 进行视频播放，增加水印

```bash
ffmpeg -y -f avfoundation -framerate 30 -i "1:1" -i smile.png -filter_complex overlay="64:64" output.mkv
```
这里的`1:1`表示第一步中的所有号，也就是`Capture screen 0`和 `外置麦克风`


### 查看文件编码信息

```bash

# ffmpeg -i out.mp4  -hide_banner
Input #0, mov,mp4,m4a,3gp,3g2,mj2, from 'out.mp4':
  Metadata:
    major_brand     : isom
    minor_version   : 512
    compatible_brands: isomiso2avc1mp41
    encoder         : Lavf58.20.100
  Duration: 00:00:05.63, start: 0.000000, bitrate: 1845 kb/s
    Stream #0:0(und): Video: h264 (High) (avc1 / 0x31637661), yuv420p, 3360x2100, 1781 kb/s, 29.81 fps, 30 tbr, 16k tbn, 2000k tbc (default)
    Metadata:
      handler_name    : VideoHandler
    Stream #0:1(und): Audio: vorbis (mp4a / 0x6134706D), 48000 Hz, mono, fltp, 63 kb/s (default)
    Metadata:
      handler_name    : SoundHandler
At least one output file must be specified
```

### 文件转码

```bash
# 转为HEVC/H.265编码
ffmpeg -i out.mkv -c:v libx265 h265.MP4

# 转为H.264编码
ffmpeg -i out.mkv -c:v libx264 out.MP4
```

### 视频截图

```bash

# 在第5s的第一帧截图
ffmpeg -ss 5 -i output.mp4 -vframes 1 -q:v 2 out.png 

# 从第1s开始截图，持续5s，每秒截一帧
ffmpeg -ss 1 -i output.mp4 -t 5 -r 1 output/out-%00d.png 

```

上面例子中，`-vframes 1`指定只截取一帧，`-q:v 2`表示输出的图片质量，一般是1到5之间（1 为质量最高）。



### 生成GIF

```bash
# 从第2s开始，以每秒15帧的速度持续5s进行截图，然后生成gif
ffmpeg -ss 00:00:02.000 -i out.mp4 -pix_fmt rgb24 -t 5  -frames 15 out.gif
```

### 视频裁剪

```bash
# 从第2s开始，连续剪5s然后生成新的视频
ffmpeg -ss 2 -i out.mp4 -t 5 new.mp4

```

### 加速、减速视频

```bash
# 视频加速
ffmpeg -i output.mp4 -vf "setpts=0.5*PTS" new.mp4

# 视频减速
ffmpeg -i output.mp4 -vf "setpts=2*PTS" new.mp4

```

### 音频文件增加图片

```bash
ffmpeg -loop 1 -i smile.jpg -i input.mp3 -c:v libx264 -c:a aac -b:a 192k -shortest output.mp4
```

上面命令中，有两个输入文件，一个是封面图片cover.jpg，另一个是音频文件input.mp3。`-loop 1`参数表示图片无限循环，`-shortest`参数表示音频文件结束，输出视频就结束。

### 提取视频中的音频

```bash
ffmpeg -i https://g.asyncoder.com/sheep.mp4 -vn -y out.mp3
```


### 增加水印

```bash
# 在视频中间增加水印
ffmpeg -i output.mkv -i smile.png -y -filter_complex overlay="(main_w/2)-(overlay_w/2):(main_h/2)-(overlay_h)/2" output.mp4
```

`overlay`用于设置水印位置， `main_w`和`main_h`表示视频宽度和高度, `overlay_w`和`overlay_h`表示水印宽度和高度


### 直播推流

- 安装docker环境

```bash
docker pull alfg/nginx-rtmp
docker run -it -p 1935:1935 -p 8080:80 --rm alfg/nginx-rtmp
```

- 推流

```bash
 ffmpeg -re  -stream_loop -1 -i ~/Movies/霸王龙.mp4  -f flv rtmp://l27.0.0.1:1935/stream/mp4
```

`re`表示输入视频的帧率, `-stream_loop`表示循环播放的次数，`-`表示无限循环

也可以增加水印

```bash
ffmpeg -re  -stream_loop -1 -i ~/Movies/霸王龙.mp4 -i smile.png -filter_complex overlay="(main_w/2)-(overlay_w/2):(main_h/2)-(overlay_h)/2"  -f flv rtmp://l27.0.0.1:1935/stream/mp4
```

推流到腾讯云

```bash
ffmpeg -re  -stream_loop -1 -i ~/Movies/霸王龙.mp4 -f flv 'rtmp://103467.livepush.myqcloud.com/live/mp4?txSecret=1e040cdd5347f8ec0d12df305d3cfad9&txTime=5F38A505'
```
可以通过`https://103467.liveplay.myqcloud.com/live/mp4.flv`进行访问

- 测试

可以通过`ffplay rtmp://l27.0.0.1:1935/stream/mp4`进行测试，也可以访问`ffplay http://127.0.0.1:8080/live/mp4.m3u8`进行测试


![](https://g.asyncoder.com/images/20200814101839-YPRWCa.png)


![](https://g.asyncoder.com/images/20200814101605-1EtOgH.png)


![](https://g.asyncoder.com/images/20200814102307-KLm7ck.png)



## 参考资料

- [ffmpeg 入门笔记](http://einverne.github.io/post/2015/12/ffmpeg-first.html)
- [FFmpeg 视频处理入门教程](https://www.ruanyifeng.com/blog/2020/01/ffmpeg.html)
- [刻意练习FFmpeg系列：音视频基础概念格式和编码
](https://zhuanlan.zhihu.com/p/137686779)
- [FFmpeg概述及编码支持](https://zhuanlan.zhihu.com/p/37516093)
- [FFmpeg中overlay滤镜用法-水印及画中画](https://www.cnblogs.com/leisure_chn/p/10434209.html)
- [ffmpeg录屏](https://trac.ffmpeg.org/wiki/Capture/Capture/Desktop%E4%B8%AD%E6%96%87%E7%89%88%E6%9C%AC)
- [docker-nginx-rtmp](https://github.com/alfg/docker-nginx-rtmp)