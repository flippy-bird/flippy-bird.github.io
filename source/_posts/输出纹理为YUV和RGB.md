---
title: 输出纹理为YUV和RGB
date: 2021-09-04 13:53:23
tags: 
    - bug
categories: 
    - OpenGL
description: 记录一个使用OpenGL ES在ios上渲染不正确的问题
---

## 输出纹理为YUV 和RGB

<!-- more -->


写这篇博客，是因为我在使用OpenGL ES渲染从视频获取的纹理时(使用YUV的方式)，出现了颜色偏差的问题，但是我仔细看了我的代码，采样坐标，参数设置等都没有问题，然后我也学习了别人的相关代码，发现使用的方式都是一样的，因此被这个问题卡主了，于是只好换一种方式了，因此借鉴了下面这篇文章的设置方法，将视频输出数据直接设置为RGB，解决了颜色偏差的问题

> [AVPlayer初体验之视频解纹理](https://cloud.tencent.com/developer/article/1141269)

### 1.YUV纹理

由于视频的编码格式基本都是YUV420，然后苹果的[demo代码](https://developer.apple.com/library/archive/samplecode/AVBasicVideoOutput/Listings/AVBasicVideoOutput_APLEAGLView_m.html#//apple_ref/doc/uid/DTS40013109-AVBasicVideoOutput_APLEAGLView_m-DontLinkElementID_6)中通过AVPlayerItemVideoOutput获取Y-Pannel和 UV-Pannel 两张纹理，最后在Shader中对两种纹理组合处理。

设置`AVPlayerItemVideoOutput`的部分代码

```objectivec
NSDictionary *pixBuffAttributes = @{(id)kCVPixelBufferPixelFormatTypeKey: @(kCVPixelFormatType_420YpCbCr8BiPlanarVideoRange)};
self.videoOutput = [[AVPlayerItemVideoOutput alloc] initWithPixelBufferAttributes:pixBuffAttributes];
```

输出纹理的部分代码

```objectivec
//Y-Plane
glActiveTexture(GL_TEXTURE0);
err = CVOpenGLESTextureCacheCreateTextureFromImage(kCFAllocatorDefault, _videoTextureCache, pixelBuffer, NULL, GL_TEXTURE_2D, GL_RED_EXT, frameWidth, frameHeight, GL_RED_EXT, GL_UNSIGNED_BYTE, 0, &_lumaTexture);
//UV-plane
glActiveTexture(GL_TEXTURE1);
err = CVOpenGLESTextureCacheCreateTextureFromImage(kCFAllocatorDefault, _videoTextureCache, pixelBuffer, NULL, GL_TEXTURE_2D, GL_RG_EXT, frameWidth / 2, frameHeight / 2, GL_RG_EXT, GL_UNSIGNED_BYTE, 1, &_chromaTexture);
```

其中的`kCVPixelFormatType_420YpCbCr8BiPlanarVideoRange`是CoreVideo中指定的[Pixel Format Identifiers](https://developer.apple.com/documentation/corevideo/1563591-pixel_format_identifiers?language=objc) 类型，在`OpenGLES2`环境下其对应的参数是`GL_RED_EXT`和`GL_RG_EXT`。

### 2.RGB纹理

首先要明白一点，上图中明确说明，`BGRA`的输出格式是`420v`的两倍多带宽(*More than 2x bandwidth*)，并且在该图来源,WWDC的[这个视频](https://developer.apple.com/videos/play/wwdc2011/419/)的`27:00`位置明确说明420v的输出格式效率会明显高于BGRA的输出格式(*It does come across if you can avoid using BGRA and doing your work in YUV, it's more efficient from bandwidth standpoint*),但是反过来，对于OpenGL来说，两张纹理的性能又会低于一张纹理。而且直接使用使用`BGRA`毕竟会方便很多，因为输出的直接就是一张纹理。但是现在毕竟都iOS11时代了，所以影响可以忽略不计。

设置`AVPlayerItemVideoOutput`的代码

```objectivec
NSDictionary *pixBuffAttributes = @{(id)kCVPixelBufferPixelFormatTypeKey: @(kCVPixelFormatType_32BGRA)};
self.videoOutput = [[AVPlayerItemVideoOutput alloc] initWithPixelBufferAttributes:pixBuffAttributes];
```

输出纹理的代码

```objectivec
CVReturn textureRet = CVOpenGLESTextureCacheCreateTextureFromImage(kCFAllocatorDefault, self.videoTextureCache, pixelBuffer, nil, GL_TEXTURE_2D, GL_RGBA, width, height, GL_BGRA, GL_UNSIGNED_BYTE, 0, &_textureOutput);
```

`BGRA`对应的输出格式是`kCVPixelFormatType_32BGRA`,其对应的从Buffer读纹理的参数是`GL_RGBA`和`GL_BGRA`。

> 2021.9.3
>
> 后面想了一下，我是输出到一个CALayer上的，CALayer上也有默认的颜色显示方式，应该是这一项不匹配导致的，因为当时的bug是颜色偏蓝，偏红(我试着改变了输出的RGB值时出现的现象)，可能是这个原因，因为当时没有考虑到这一点，后面如果遇到类似问题，试着查一下显示Layer的显示方式。



