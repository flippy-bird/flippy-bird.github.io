---
title: RenderDoc使用
date: 2021-11-17 22:54:16
tags: "RenderDoc"
categories: "工具使用"
description: RenderDoc使用过程中的问题
cover: /covers/23.png
---

## RenderDoc使用

<!-- more -->

### 绝对路径

使用RenderDoc调试程序时，程序中涉及到的一些相对路径，可能读取不到，导致无法正常运行程序，因此需要改成绝对路径，如下：

![image](https://raw.githubusercontent.com/nashpan/image-hosting/main/markdown_image/image.4uaqe7a5o420.png)

包括读取图片时，那些路径也要设置为绝对路径

### RenderDoc设置环境变量

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/markdown_image/renderDoc_problem.png" alt="renderDoc_problem" style="zoom:67%;" />

有些程序可能会需要命令行的参数才能执行，并且可能会依赖一些动态库，如图，按照上图进行设置即可

