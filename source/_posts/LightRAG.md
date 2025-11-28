---
title: LightRAG
date: 2025-11-28 15:06:12
description: 学习了解LightRAG
tags:
    - Rag
categories:
    - RAG框架学习
---

### 1. 背景

待详细看完完整论文后补充，大概就是GraphRag虽然效果比较好，但是速度慢，建立图的过程中消耗的token也很多，因此有了LightRag；

### 2. LightRAG的原理

> [深度解析比微软的GraphRAG简洁很多的LightRAG，一看就懂](https://zhuanlan.zhihu.com/p/4821793882)

先看了知乎上的一些解读，感觉就是简化版本的GraphRAG，在索引阶段的处理方式是差不多的，只是LightRAG将GraphRAG中比较慢的部分去掉了(社区报告部分，因此也去掉了global search)，数据处理流程是：先分块，然后提取实体和关系，然后入库，index部分的工作就完成了；

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20251128151441858.png" alt="image-20251128151441858" style="zoom: 67%;" />

在查询阶段，提供了4中方式：

- 最基本的向量相似度匹配
- local search: 根据用户的query生成一些low-level 的关键词，然后根据生成的关键词去建立好的图谱查询；
- global search: 根据用户的query生成一些high-level 的关键词，然后根据生成的关键词去建立好的图谱查询；
- 混合搜索：local search + global search 

![image-20251128151926968](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20251128151926968.png)

### 3. 源码解析

- [ ] TODO
