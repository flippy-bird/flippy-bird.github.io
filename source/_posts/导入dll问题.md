---
title: 导入dll问题
date: 2021-11-17 22:46:59
tags: 
    - bug
description: C++ 导入dll动态库时的顺序问题
cover: /covers/1.png
categories: 
    - "C++"
---

## 导入dll问题

<!-- more -->

### 问题

我在AE中导入dll时，导入失败，但是我确实加载了所有的dll库

### Solution

问题出在我加载的dll也有依赖关系，所以需要按照顺序来加载dll库如下

![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/markdown_image/image.3aavo2gruf20.png)

导入ffmpeg的dll时，在AE中一定要按照上面的顺序进行导入，因为后面的库会依赖前面的库，我之前在这里卡了很久，我当时想着逐个导入，然而我选择的第一个是avformat,这个库会依赖avutil，avcodec，因此代码运行到这儿的时候就会崩掉。

### 其它

1.在解决dll相关问题时，可以使用depends软件查看库依赖，很好用

2.可以使用vs的一些命令行工具查看dll的架构，64 or 32

 