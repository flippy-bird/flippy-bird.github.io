---
title: smolagents源码学习
date: 2025-11-25 16:43:25
description: 开源项目smolagents源码学习记录
cover: /covers/26.png
# cover: /pan_img/llm_cover.png
tags:
    - Agent
categories:
    - Agent开源项目学习
---


> https://github.com/huggingface/smolagents



#### memory 管理

```python
class AgentMemory:
    def __init__(self, system_prompt:str):
        self.system_prompt: SystemPromptStep = SystemPromptStep(system_prompt=system_prompt)
        self.steps: list[TaskStep | ActionStep | PlanningStep] = []
```

将LLM执行过程的信息划分成了四个部分(主要，其它)：

![image-20250926172929838](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20250926172929838.png)

- TaskStep: 与用户输入相关 (用户提问，上传图片等)
- SystemPromptStep: 系统的prompt
- PlanningStep: 与规划相关的记忆 (**暂时还没遇到**)
- ActionStep：当前送到LLM进行执行的信息

**历史信息的获取**

所以主要在于上面每种类型记忆数据 `to_messages`的实现 需要注意的是前后信息的完整性

```python
# agents.py line:1256
def _step_stream(
        self, memory_step: ActionStep
    ) -> Generator[ChatMessageStreamDelta | ToolCall | ToolOutput | ActionOutput]:
    """
    Perform one step in the ReAct framework: the agent thinks, acts, and observes the result.
    Yields ChatMessageStreamDelta during the run if streaming is enabled.
    At the end, yields either None if the step is not final, or the final answer.
    """
    memory_messages = self.write_memory_to_messages()
        
# agents.py line:758
def write_memory_to_messages(
        self,
        summary_mode: bool = False,
    ) -> list[ChatMessage]:
    """
    Reads past llm_outputs, actions, and observations or errors from the memory into a series of messages 	  that can be used as input to the LLM. Adds a number of keywords (such as PLAN, error, etc) to help 	 the LLM.
    """
    messages = self.memory.system_prompt.to_messages(summary_mode=summary_mode)
    for memory_step in self.memory.steps:
        messages.extend(memory_step.to_messages(summary_mode=summary_mode))
    return messages
```



#### 边界处理

当超过最大尝试次数（默认是20次）时，最后会总结19步的step，然后给出一个最终答案

```python
# agent.py  line:810

def provide_final_answer(self, task: str) -> ChatMessage:
    
    messages : 这里有一个专门针对这种情况的系统prompt
    messages += self.write_memory_to_messages()[1:]
    messages : 需要组装的post_messages
    
    try:
        chat_message: ChatMessage = self.model.generate(messages)
        return chat_message
    except Exception as e:
        return ChatMessage(
            role=MessageRole.ASSISTANT,
            content=[{"type": "text", "text": f"Error in generating final LLM output: {e}"}],
        )
```



#### Agent: CodeAgent

![image-20250929112702011](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20250929112702011.png)

项目的CodeAgent模式算是这个项目里面比较新颖的一种方式了，调用工具的API是通过执行python代码的方式来执行的，但是感觉解析python的AST那部分，就感觉好复杂-_-#，没有FunctionCall的这种方式简洁了。

来源于[Executable Code Actions Elicit Better LLM Agents](https://link.zhihu.com/?target=https%3A//huggingface.co/papers/2402.01030)，这篇论文主要的出发动机是当前LLM Agent通常通过以预定义的格式生成 JSON 或文本来生成Action，这通常受到**约束动作空间（例如，预定义工具的范围）和受限灵活性（例如，无法组合多个工具）的限制**。

> 链接：https://zhuanlan.zhihu.com/p/16341067315
