---
title: Metal渲染到纹理
date: 2021-09-04 13:54:42
tags: "Metal"
categories: "iOS新手村"
description: 使用Metal渲染到纹理
cover: /covers/19.png
---

## 渲染到纹理

<!-- more -->

滤镜链

> [Metal - 5 滤镜链（渲染到纹理）](http://81.71.66.22/Metal-5-filter-chain/)

Metal的标准坐标系NDC、帧缓存坐标系FrameBuffer Coordinate (也叫Viewport Coorninate)以及纹理坐标系（Texture Coordinate)的原点不一致，还会分别对他们进行介绍和解析。

在Metal中要渲染到纹理有如下步骤：

### 1.创建空纹理

```objectivec
+ (nullable id<MTLTexture>)createEmptyTexture: (id<MTLDevice>)device
                                    WithWidth: (size_t)width
                                   withHeight: (size_t)height
                                        usage: (MTLTextureUsage)usage {
    
    MTLTextureDescriptor *texDescriptor = [MTLTextureDescriptor texture2DDescriptorWithPixelFormat: MTLPixelFormatRGBA8Unorm width: width height: height mipmapped: false];
    texDescriptor.usage = usage;
    
    id<MTLTexture> texture = [device newTextureWithDescriptor: texDescriptor];
    
    return texture;
}
```

- usage 纹理的用处，是 `MTLTextureUsage` 枚举，有如下枚举值：
  - `MTLTextureUsageUnknown` 未知用处（运行时设置），会损耗性能
  - `MTLTextureUsageShaderRead` 该纹理将被读取
  - `MTLTextureUsageShaderWrite` 该纹理将被写入
  - `MTLTextureUsageRenderTarget` 该纹理将作为渲染目标
  - `MTLTextureUsagePixelFormatView` 在 PixelFormatView 被使用（暂不介绍）

回到 demo 中，这个中间 buffer 是 **当次渲染的渲染目标** ，也将 **作为下一次渲染的输入被读取（采样）**，因此 usage 是 `MTLTextureUsageShaderRead | MTLTextureUsageRenderTarget`。

完整调用：

```objectivec
_grayResultTexutre = [MetalUtils createEmptyTexture: _device
                                              WithWidth: CGImageGetWidth(cgImage)
                                             withHeight: CGImageGetHeight(cgImage)
                                                  usage: MTLTextureUsageShaderRead | MTLTextureUsageRenderTarget];
```

### 2.空纹理关联到渲染目标

以往为了要将渲染结果展示，我们的渲染目标是与 Layer 关联起来的：

```objectivec
RenderTargetDesc.colorAttachments[0].texture = currentDrawable.texture;
```

但是这次我们需要渲染到纹理上，因此要把 **渲染目标和要渲染到的纹理** 关联起来：

```objectivec
_grayRenderTargetDesc.colorAttachments[0].texture = _grayResultTexutre;
```

### 3.提交指令，执行渲染

指令是被 ”装“ 到 Encoder 上的，因此渲染过程也大同小异，但是由于 **不需要展示渲染结果（即不需要上屏 swap buffer）**，所以不需要 `presentDrawable`。

```objectivec
- (void)render_gray {
    
    // 将本次 Command Encoder 和 渲染的目标（MTLTexture）关联起来
    _grayRenderTargetDesc.colorAttachments[0].texture = _grayResultTexutre;
    
    id<MTLCommandBuffer> commandBuffer = [_queue commandBuffer];
    commandBuffer.label = @"Gray Command Buffer";
    
    id<MTLRenderCommandEncoder> encoder = [commandBuffer renderCommandEncoderWithDescriptor: _grayRenderTargetDesc];
    encoder.label = @"Gray Command Encoder";
    
    [encoder setViewport: (MTLViewport) {
        .originX = 0,
        .originY = 0,
        .width = _sourceTexutre.width,
        .height = _sourceTexutre.height,
        .znear = 0,
        .zfar = 1
    }];
    
    [encoder setRenderPipelineState: _grayRenderPipelineState];
    
    [encoder setVertexBytes: grayVertices
                     length: sizeof(grayVertices)
                    atIndex: 0];
    
    [encoder setVertexBytes: texCoor
                     length: sizeof(texCoor)
                    atIndex: 1];
    
    [encoder setFragmentTexture: _sourceTexutre
                        atIndex: 0];
    
    [encoder setFragmentBytes: &_grayIntensity
                       length: sizeof(float)
                      atIndex: 0];
    
    [encoder drawIndexedPrimitives: MTLPrimitiveTypeTriangleStrip
                        indexCount: 6
                         indexType: MTLIndexTypeUInt32
                       indexBuffer: _indexBuffer
                 indexBufferOffset: 0];
    
    [encoder endEncoding];
    
    [commandBuffer commit];
}
```

注意，Command Buffer 并不需要 `presentDrawable:`。
