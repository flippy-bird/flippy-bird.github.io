---
title: Metal中加载MTLLibrary
date: 2021-09-04 13:54:16
tags: "Metal"
categories: "iOS新手村"
description: 使用Metal加载指定shader文件
cover: /covers/20.webp
---

## Metal 中加载MTLLibrary

<!-- more -->


一般情形下，我们加载MTLLibrary会使用如下代码：

```objectivec
id<MTLLibrary> defaultLibrary = [self.mtkView.device newDefaultLibrary]
```

确实这样是正确的，没有问题的，但是如果我们是开发一个静态库，.Metal 文件放置在你的静态库工程文件中，在测试时使用上面的函数就会找不到这个Metal文件，当然，你把Metal文件复制到你的开发工程中，可以正常运行，但是这样的话，Metal文件就暴露出来了，而且我们给其他人提供静态库时，不可能顺带还把Metal文件给他，因此需要换一种方法加载Metal文件

在苹果的官方论坛上，提供了一种方法[一种方法](https://developer.apple.com/forums/thread/52287)

```objectivec
NSError *error = nil;
NSString *libPath = [[NSBundle mainBundle] pathForResource:@"MySecretMetalLib" ofType:@"metallib"];
id<MTLLibrary> library = [device newLibraryWithFile:libPath error:&error];
```

当然，如果我们查看device创建library的方法文档的话，也会找到下面的方法，即将metal文件转化成字符串，然后传入，类似于OpenGL中传入shader文件。方法如下：

```objectivec
id<MTLLibrary> defaultLibrary = [m_metalRendererDevice newLibraryWithSource:shaderSource options: nil error:nil];
```

对应的shaderSource文件为：

![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/markdown_image/image.16nns81u3bts.png) 

当然我们也可以手动添加双引号将Metal文件转化为字符串，但是个人觉得太麻烦了，然后就学习了别人的这种字符串化写法，当然如果是C++的话，可以直接使用R();当然，开始的时候还遇到了其它问题，注意到上面图中的红色部分，一开始，我没有添加，后面添加了就可以正常读取Metal文件中后面的函数了。

另外，字符串化里面好像不能添加如下的引用（不太确定，但是我的metal文件确实没有编译成功，可能和metal文件的编译方式有关系，先记录在这儿吧，后面解决坑了再回来填坑。

```objectivec
#import "ShaderTypes.h"   //直接将结构体写进字符串里面就行，
```

