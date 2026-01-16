---
title: AgentScope源码学习
date: 2025-11-25 19:47:53
description: 开源项目AgentScope源码学习记录
cover: /covers/8.webp
# cover: /pan_img/AS_cover.webp
tags:
    - Agent
categories:
    - Agent开源项目学习
---

### AgentScope (阿里的) 

> https://github.com/agentscope-ai/agentscope

#### 记忆

长期记忆部分使用了mem0这个工具，当然，代码里面也提到了，可以使用阿里自家的ReMe这个记忆框架；

在26年1月份的更新中，添加了数据库的支持(使用Redis和Sql)，使用Redis应该是保存临时的对话历史信息，Sql是为了保存长期记忆；

#### Agent

这一块儿使用的是基本的React模式，输出最后的回答，也成了一个工具；多了一个

```python
### AgentBase
# 在AgentBase里面有一个虚函数 reply
def reply()

### 在ReactAgentBase里面有基本的React框架
def _reasoning()   # 虚函数

def _acting()      # 虚函数

### 在ReactAgent里面
def reply():           这里就类似OpenManus里面的step()函数的作用了，从源码来看，逻辑完全一样
    self._reasoning()
    ...
    self._acting()
```

在Agent里面有一个observe，感觉是为了观察到外界信息准备的接口(用于多Agent之间的信息互动)

```python
async def observe(self, msg: Msg | list[Msg] | None) -> None:
    """Receive the given message(s) without generating a reply.

    Args:
        msg (`Msg | list[Msg] | None`):
            The message(s) to be observed.
    """
    raise NotImplementedError(
        f"The observe function is not implemented in"
        f" {self.__class__.__name__} class.",
    )
```

#### 多Agent互动

在这个框架里面使用的是swarm模式，似乎比较简单，每个agent observe其它agent的输出，添加到自己的记忆里面去就好了

在src/pipeline/_msghub.py文件夹里面 （或者见类名时的说明）

```python
async def broadcast(self, msg: list[Msg] | Msg) -> None:
    """Broadcast the message to all participants.

    Args:
        msg (`list[Msg] | Msg`):
            Message(s) to be broadcast among all participants.
    """
    for agent in self.participants:
        await agent.observe(msg)
```

#### Interrupt(中断介入)

这个好像还不错，可以看一下，文档在https://doc.agentscope.io/tutorial/task_tool.html#interrupting-tool-execution， 对于工具执行的取消，是利用了asyncio的取消机制来实现的



#### Hooks

使用了python元编程来实现，在元类中添加一些



#### Plan的实现

实现了一些plan的功能函数，然后让Agent去调用和更改当前的plan

![image-20251119115208155](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20251119115208155.png)

### 其它

字节也出了一个类似框架：[veadk-python](https://github.com/volcengine/veadk-python)

