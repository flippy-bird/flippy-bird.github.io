---
title: 模型推理基本知识-tensorRT
date: 2026-01-26 10:14:09
description: 学习TensorRT相关的知识
cover: /covers/34.webp
tags:
    - inference
categories:
    - 推理部署
---



## TensorRT

tensorRT所做的优化是基于GPU进行优化，当然也是更喜欢那种一大块一大块的矩阵运算，尽量直通到底。因此对于通道数比较多的卷积层和反卷积层，优化力度是比较大的；如果是比较繁多复杂的各种细小op操作(例如reshape、gather、split等)，那么TensorRT的优化力度就没有那么夸张了。

为了更充分利用GPU的优势，**我们在设计模型的时候，可以更加偏向于模型的并行性**，因为同样的计算量，“大而整”的GPU运算效率远超“小而碎”的运算。

### tensorRT做了哪些？

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20250903151421704.png" alt="image-20250903151421704" style="zoom: 67%;" />

- 量化：量化即IN8量化或者FP16以及TF32等不同于常规FP32精度的使用，这些精度可以显著提升模型执行速度并且不会保持原先模型的精度
- 算子融合(层与张量融合)：简单来说就是通过融合一些计算op或者去掉一些多余op来减少数据流通次数以及显存的频繁使用来提速
- 内核自动调整：根据不同的显卡构架、SM数量、内核频率等(例如1080TI和2080TI)，选择不同的优化策略以及计算方式，寻找最合适当前构架的计算方式
- 动态张量显存：通过显存池复用，仅在运行阶段为张量分配内存来提高内存复用率，减少内存消耗并避免分配开销以实现更高效的执行效率。
- 多流执行：使用CUDA中的stream技术，最大化实现并行操作

   简单来说，针对计算密集型任务，例如矩阵乘法和卷积等操作，TensorRT 可以通过优化算子提升计算效率，对于缓存密集型的任务，TensorRT可以通过算子融合的方式减少缓存和拷贝的数据量以提高显存访问的效率，同时，也可通过使用低精度的数据类型减少计算时间和内存的使用量以加快运行速度。

tensorRT支持FP32、FP16、TF32、INT8等常见的数据格式，下面是各种精度的区别：

![image-20250903152654430](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20250903152654430.png)

TF32：第三代Tensor Core支持的一种数据类型，是一种截短的 Float32 数据格式，将FP32中23个尾数位截短为10bits，而指数位仍为8bits，总长度为19(=1+8 +10)。保持了与FP16同样的精度(尾数位都是 10 位），同时还保持了FP32的动态范围指数位都是8位)；



### 模型转换

一般路径是：pytorch模型 ---> onnx模型 ---> TensorRT模型

- pytorch 模型转onnx模型

```python
import torch.onnx

# 转换后ONNX模型的存储路径
onnx_model = "model.onnx"
# 需要转换的Pytorch模型
torch_model = "model.pth" 
# 加载Pytorch模型权重
model = model.load_state_dict(torch.load(model))
# 导出模型前，调用model.eval()
model.eval()
batch_size = 1 # 随机的取值，当设置dynamic_axes后影响不大
# dummy_input理解为一个输入的实例，仅提供输入shape、type等信息，无需关心数值
dummy_input = torch.randn(batch_size, 3, 224, 224, dtype=torch.float16) 
# 导出模型
torch.onnx.export(model,
                  dummy_input,
                  onnx_model,
                  export_params=True, # 指定为True或默认, 模型参数也会被导出
                  # https://github.com/onnx/onnx/blob/main/docs/Operators.md
                  # 算子无法支持时，会提示ONNX export failed: Couldn't export operator xxx
                  opset_version=23, #ONNX算子集的版本，与torch版本相关
                  do_constant_folding=True, #是否执行常量折叠优化
                  #输入参数名称，建议与forward函数中定义参数保持一致
                  input_names = ['input'], 
                  #输出张量的名称，建议与torch模型中定义参数保持一致
                  output_names = ['output'], 
                  # dynamic_axes将batch_size的维度指定为动态
                  # 0代表第0维为动态输入，batch_size为对这一维度的命名，可按需修改
                  # input与output为input_names与output_names中定义的名称
                  dynamic_axes={'input' : {0 : 'batch_size'},    
                                'output' : {0 : 'batch_size'}})
```



- 使用onnxruntime验证一下onnx模型

这个模型需要的输入是三个 

```python
import onnx
import numpy as np
import onnxruntime as rt
import cv2

model_path = '/home/oldpan/code/models/Resnet34_3inputs_448x448_20200609.onnx'

# 验证模型合法性
onnx_model = onnx.load(model_path)
onnx.checker.check_model(onnx_model)

# 读入图像并调整为输入维度
image = cv2.imread("data/images/person.png")
image = cv2.resize(image, (448,448))
image = image.transpose(2,0,1)
image = np.array(image)[np.newaxis, :, :, :].astype(np.float32)

# 设置模型session以及输入信息
sess = rt.InferenceSession(model_path)
input_name1 = sess.get_inputs()[0].name
input_name2 = sess.get_inputs()[1].name
input_name3 = sess.get_inputs()[2].name

output = sess.run(None, {input_name1: image, input_name2: image, input_name3: image})
print(output)
```

校验ONNX和Pytorch输出结果是否一致

```python
y = model(x)
y_onnx = model_onnx(x)

# check the output against PyTorch
print(torch.max(torch.abs(y - y_trt)))
```



- onnx模型转换成TensorRT模型

直接使用TensorRT包中的 .../bin/trtexec 这个工具，他的源码是：[工具源码](https://github.com/onnx/onnx-tensorrt)



### TensorRT实际工作流程

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20250904102646692.png" alt="image-20250904102646692" style="zoom:80%;" />



### 实际demo

*TODO*



### 验证

- 使用Polygraphy查看onnx与TRT模型的输出差异：[工具polygraphy](https://github.com/NVIDIA/TensorRT/tree/master/tools/Polygraphy)



### 参考资料

1. [TensorRT详细入门指北，如果你还不了解TensorRT，过来看看吧！](https://zhuanlan.zhihu.com/p/371239130)
2. [[利用 TensorRT 实现深度学习模型的构建与加速](https://zhuanlan.zhihu.com/p/22577993507)]
