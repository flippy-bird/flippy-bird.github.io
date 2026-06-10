---
title: claude code 源码学习
date: 2026-06-02 20:23:39
description: 记录claude code的源码学习到的东西
cover: /covers/11.webp
tags:
    - Agent
categories:
    - Agent开源项目学习
---



### System Prompt 构造

![image-20260602202855951](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260602202855951.png)

claude code的提示词是几个部分动态组装的，主要包括下面几个部分：

- 角色定义与安全红线
- 行为准则
- 操作安全
- 工具使用指南
- Git 安全协议
- 输出风格约束
- 环境信息注入
- 其它信息

一段组装后的 System Prompt 大概如下面所示（简化版）：

![image-20260602203116023](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260602203116023.png)

可能启发就是写prompt的时候可以分层(分模块)去写，方便管理和更新

### 记忆系统

#### 记忆的分类

对应的Claude Code的指令是 (/memory 需要主动开启，见官网：[存储指令和记忆](https://code.claude.com/docs/zh-CN/memory#%E5%AE%83%E5%A6%82%E4%BD%95%E5%B7%A5%E4%BD%9C) )

Claude code 将记忆分成了4个类型：

```typescript
export const MEMORY_TYPES = [
  'user',      // 用户画像：角色、偏好、知识水平
  'feedback',  // 行为反馈：该做什么、不该做什么
  'project',   // 项目动态：在做什么、截止日期、协作信息 (日期需要转成绝对日期)
  'reference', // 外部指针：哪里能找到什么信息
] as const
```

对于行为反馈，不仅记住了规则本身，还要求记录为什么和如何应用

```python
'''
规则本身：集成测试必须使用真实数据库，不能用 mock
Why：上季度 mock 测试全部通过但生产环境迁移失败了
How to apply：在这个模块写测试时，始终连接真实数据库
'''
```

![image-20260602204427466](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260602204427466.png)

同样的，记住什么比较重要，不记什么，也比较重要：

![image-20260602204614601](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260602204614601.png)

这个排除清单背后的核心原则是：**可以从当前代码推导出来的信息，一律不存**。

#### 记忆的存储

是按照类似skills的方式，按需加载

**MEMORY.md 索引始终被加载到 System Prompt 里**，但独立记忆文件**按需加载**。

![image-20260602204828034](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260602204828034.png)

#### 记忆的召回

在做记忆的召回时，专门使用了便宜的sonnet模型做记忆检索。

并且是用户提交信息之后，立刻就开始了，和主模型的API调用并行执行

```typescript
// query.ts 中的调用——在进入主循环之前就启动记忆预取
using pendingMemoryPrefetch = startRelevantMemoryPrefetch(
  state.messages,
  state.toolUseContext,
)
```

时序大概是这样的：

![image-20260602205214262](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260602205214262.png)

我有一个问题诶：如果第一轮的时候，就需要这些记忆内容呢，或者第一轮的时候就需要用户的偏好呢，那不就有问题了嘛？



### 上下文窗口管理

Claude Code 的核心理念是：**压缩一定有信息损失，所以能不压就不压，必须压的时候从最轻的手段开始**。采用五步走的策略

![image-20260603141010586](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260603141010586.png)

#### 第一步: 大结果存磁盘

**Claude Code 怎么做？** 它在工具结果进入消息列表**之前**，就先做一道「体检」：

```typescript
async function maybePersistLargeToolResult(
  toolResultBlock: ToolResultBlockParam,
  toolName: string,
): Promise<ToolResultBlockParam> {
const size = contentSize(content)
// 单个工具结果超过阈值（默认约 50KB）？
if (size <= threshold) {
    return toolResultBlock  // 没超，原样通过
  }
// 超了！把完整内容存到磁盘文件
const result = await persistToolResult(content, toolUseId)
// 用一个 2KB 的预览替换原内容
const preview = buildLargeToolResultMessage(result)
return { ...toolResultBlock, content: preview }
}
```

除了单个工具的限制，还有一个**消息级的总量控制**，同一条消息里所有工具结果的总大小不能超过 200KB。如果超了，系统会挑出最大的那几个结果存磁盘，直到总量降到限制以内。

这一层的精妙之处在于：**完整内容并没有丢**，它还在磁盘上。如果模型后面真的需要那个大文件的某个片段，它可以再次调用 Read 工具去读取特定的行范围。

#### 第二步: 砍掉远古消息

**问题是什么？** 一次长对话可能有上百轮。对话开头那几轮的内容，比如用户最初的探索性提问、模型早期的试探性回答，到了后面几乎完全没用了。但它们仍然占着宝贵的上下文空间。

```typescript
if (feature('HISTORY_SNIP')) {
  const snipResult = snipModule.snipCompactIfNeeded(messagesForQuery)
  messagesForQuery = snipResult.messages
  snipTokensFreed = snipResult.tokensFreed
  if (snipResult.boundaryMessage) {
    yield snipResult.boundaryMessage  // 插入边界标记
  }
}
```

它不做任何摘要，不总结「前面聊了什么」，直接砍掉。听起来很暴力，但对于那些确实已经完全过时的消息来说，这是代价最低的做法，因为它**不需要额外调用大模型来生成摘要**，零 API 开销。

![image-20260603141805246](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260603141805246.png)

#### 第三步: 裁剪老的工具输出

 Micro-Compact 的核心思想是**时间衰减**：越老的工具结果越不重要，可以被裁剪。但是，不是所有工具的结果都能裁剪：

```typescript
const COMPACTABLE_TOOLS = new Set([
  FILE_READ_TOOL_NAME,    // 读文件 → 可以重新读
  ...SHELL_TOOL_NAMES,    // 执行命令 → 可以重新执行
  GREP_TOOL_NAME,         // 搜索 → 可以重新搜
  GLOB_TOOL_NAME,         // 查找文件 → 可以重新查
  WEB_SEARCH_TOOL_NAME,   // 搜索网页 → 可以重新搜
  FILE_EDIT_TOOL_NAME,    // 编辑文件 → 结果可裁剪
  FILE_WRITE_TOOL_NAME,   // 写文件 → 结果可裁剪
])
```

看到规律了吗？**可以被裁剪的，都是「可重新获取」的工具**，Read 的结果可以再读一次，Bash 的输出可以再执行一次，搜索结果可以再搜一次。

但 AgentTool（子 Agent 的输出）、TaskTool（任务状态）这类工具的结果**永远不会被裁剪**，因为子 Agent 的推理过程是不可重复的，砍掉就真的丢了。

具体裁剪逻辑是「保留最近 N 个，清理其余的」：

```typescript
// 收集所有可裁剪工具的结果 ID
const compactableIds = collectCompactableToolIds(messages)
// 保留最近 5 个，其余全部清理
const keepRecent = Math.max(1, config.keepRecent)  // 至少保留 1 个
const keepSet = new Set(compactableIds.slice(-keepRecent))
const clearSet = compactableIds.filter(id => !keepSet.has(id))
```

被裁剪的工具结果会被替换成一个标记：

```typescript
export const TIME_BASED_MC_CLEARED_MESSAGE =
  '[Old tool result content cleared]'
```







### 可参考资料

1. [万字长文图解 Claude Code 剖析源码：架构设计、Agent工作模式、System Prompt、记忆系统、上下文窗口管理等](https://zhuanlan.zhihu.com/p/2025176118068621451)
