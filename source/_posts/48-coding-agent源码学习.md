---
title: Coding Agent源码学习
date: 2026-06-02 20:23:39
description: 记录从Coding Agent源码学习到的东西
cover: /covers/11.webp
tags:
    - Agent
categories:
    - Agent开源项目学习
---



## Mimo Code (6月10日)

> [MiMo Code：将编程 Agent 扩展到长程任务](https://mimo.xiaomi.com/zh/blog/mimo-code-long-horizon)

### 并行计划采样

Max Mode 在每一轮并行生成 N 个候选方案（默认 N=5），每个候选独立完成推理和工具调用规划，但不实际执行。**然后由同一个模型作为 judge**，对比所有候选的推理过程和行动计划，选出最优的一个执行。

在 SWE-Bench Pro 上，Max Mode 相比单次采样提升 10-20%，代价是约 4～5 倍的 token 消耗。



### 工具调用准确性

模型通过什么格式发出工具调用，直接影响准确率和 token 效率。部分模型（特别是 GPT 5.5 系列）在输出结构化 JSON 时格式错误率较高，XML 相比 JSON 效果略好。(**Token 开销大**：花括号、引号、键名重复出现，占用很多 token; **格式错误率高**：特别是某些模型，经常漏掉引号、括号不匹配，导致解析失败)

小米采用的方法(6月10日的版本无这部分，到时候放出来了可以关注一下具体的做法和差异)  **我们最终发现采用受限的命令行语法**，同样的调用意图用这种格式表达所需的 token 更少，格式错误也更低，因为大部分模型在 shell 环境下的训练数据密度高。

让模型输出类似：

```shell
search --query "今天北京天气" --limit 3
```

一个完整的例子如下：

假设我们让模型“帮我查一下杭州明天的天气，如果下雨就提醒我带伞”。

传统 JSON 工具调用，模型需要分步输出两段 JSON：

```json
{
  "tool": "get_weather",
  "params": {
    "city": "杭州",
    "date": "2026-06-16"
  }
}
```

收到结果后，再判断天气，然后：

```json
{
  "tool": "send_reminder",
  "params": {
    "message": "明天有雨，记得带伞",
    "time": "2026-06-15 20:00"
  }
}
```

#### 改用受限的命令行语法（简洁，稳健）

模型可以直接输出：

```shell
get_weather --city "杭州" --date "2026-06-16"
```

收到返回的天气数据后，再输出：

```shell
send_reminder --message "明天有雨，记得带伞" --time "2026-06-15 20:00"
```



### Dynamic Workflow （大规模并行编排）

当任务规模足够大（例如将整个项目从一种语言迁移到另一种语言），需要同时协调几十甚至上百个并行工作单元时，逐轮的工具调用不再够用。

传统的做法是把流程写进 SKILL.md，用自然语言告诉模型"先做 A，再做 B，如果 C 就做 D"。这在简单场景下能用，但在复杂流程中会系统性失效：上下文压缩可能吞掉步骤、模型可能跳过某些环节、分支和重试逻辑靠模型判断而非代码保证、同一流程两次执行路径不同。**本质问题是：编排逻辑以自然语言存在，而自然语言是模糊的、可遗忘的、不可验证的。**

Dynamic Workflow 把编排逻辑从 prompt 变成代码。主 Agent 生成一段 JavaScript 脚本，在隔离沙箱中确定性执行。



### 记忆

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260615110024027.png" alt="image-20260615110024027" style="zoom: 50%;" />

把会话想象成一串从左到右排开的 turn。窗口有上限，turn 在累积，窗口终会被填满。如果不干预，会话到达上限时要么结束，要么悄悄退化。

运行时在到达上限之前的几个固定位置介入。我们称这些位置为 checkpoint。每个 checkpoint 处，运行时派出一个独立的 writer subagent：读取迄今的对话，将一份结构化状态写入磁盘。主 Agent 继续工作，writer 并发执行，互不干扰。

主要有一个点是：  **提前压缩** ---> checkpoint 是在远低于上限处触发(20%, 45%, 70%), 主要考虑的原因是：模型在高上下文利用率下能力会衰减。这在文献中被称为 "lost in the middle"：随着输入变长，对中段材料的注意力下降，结构化提取的可靠性显著降低。因此小米采用了提前进行压缩的策略

#### 四层记忆

- **Session 记忆**（checkpoint.md）：只在当前逻辑会话内存活，记录这次会话的完整工作状态。
- **Project 记忆**（MEMORY.md）：项目级持久知识——架构决定、用户规则、反复验证过的技术事实。当某条观察在多次 session checkpoint 中稳定下来，writer 将其从 session 层提升到这里。
- **Global 记忆**：用户级偏好，跨项目生效。
- **History**：每个会话的完整 SQLite 轨迹——每条消息、每次工具调用原文存储，不做索引。当结构化记忆中找不到某个细节时，Agent 通过 history 工具回溯到原始记录。



## Claude Code

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





## 可参考资料

1. [万字长文图解 Claude Code 剖析源码：架构设计、Agent工作模式、System Prompt、记忆系统、上下文窗口管理等](https://zhuanlan.zhihu.com/p/2025176118068621451)
