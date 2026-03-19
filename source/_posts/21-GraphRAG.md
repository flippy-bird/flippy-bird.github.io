---
title: GraphRAG
date: 2025-3-13 10:14:50
description: 学习了解GraphRAG
cover: /covers/11.webp
tags:
    - Rag
categories:
    - RAG框架学习
---



## 1. why we need GraphRAG?

> https://www.microsoft.com/en-us/research/blog/graphrag-unlocking-llm-discovery-on-narrative-private-data/

- 基本的RAG 难以建立信息关联。当回答某个问题需要遍历不同信息片段并通过其共享属性来提供新的综合见解时，就会出现这种情况。
- 基本RAG 在被要求整体理解大型数据集合甚至单个大型文档中的汇总语义概念时表现不佳。

> 我想到的一个例子是：假设我有一个介绍 LLM的文章，然后分段式介绍了LLM的一些特点(假设每一段不显式的包含LLM信息，用它，他或者其他代词指代)，然后这些信息被分别分块，然后embedding，在query (问LLM的优点， 需要根据LLM的特点来总结)，search的时候，应该就不会查到这些信息。



## 2. GraphRAG的原理

### 2.1 构建阶段

**在建立索引**(index)这一步，主要按照下面的步骤进行：
- 将输入语料库分割为一系列的文本单元（TextUnits），这些单元作为处理以下步骤的可分析单元，并在我们的输出中提供细粒度的引用。
- 使用 LLM 从文本单元中提取所有实体、关系和关键声明。并且对每个实体，关系，文本生成embedding向量
- 使用 Leiden 技术 对知识图谱进行层次聚类。
- GraphRAG通过计算实体之间的关系，填充关系表，并生成关于实体的社区报告来总结不同实体之间的关系与上下文，自下而上地生成每个社区层级及其组成部分的摘要。这有助于对数据集的整体理解。

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20251128104304468.png" alt="image-20251128104304468"  />

> 具体的例子可以参考这篇知乎文章的例子: [GraphRAG快速入门与原理详解](https://zhuanlan.zhihu.com/p/13801755777)

### 2.1 查询阶段

**在查询这一步**，又分为两种方式：

- 全局搜索，用于通过利用社区层级摘要来推理有关语料库的整体问题。
- 局部搜索，用于通过扩展到其邻居和相关概念来推理特定实体的情况。

#### 局部搜索

![image-20251128140852857](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20251128140852857.png)

有点类似BM25关键词搜索，然后找到一些相关的实体关系之后，然后再利用图谱的节点关系再在领近查找更多信息，然后再汇总

#### 全局搜索

![image-20251128141135147](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20251128141135147.png)

根据用户的问题，全局搜索会搜索相关的社区报告 ，并且给每一个社区报告打分(**使用LLM来进行**)，根据打分的高低，然后将最相关的社区报告给到大模型的上下文中；



## 3. 项目源码学习

- [ ] TODO



## 可参考资料

1. https://github.com/gusye1234/nano-graphrag
