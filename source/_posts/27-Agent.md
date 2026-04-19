---
title: Agent
date: 2025-12-11 21:01:18
description: Agent相关paper，知识持续跟进
cover: /covers/27.webp
tags:
    - LLM
    - Agent
---



### 范式--plan and solve

采用的是"先谋后动"的策略，先进行plan，然后根据具体的计划进行solve；具体的原理图如下：

![image-20260127104708406](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260127104708406.png)

下面举一个具体的代码例子

#### plan

对于plan部分的实现，主要还是prompt工程 （定义了角色，任务，输出格式的要求等）

````python
PLANNING_PROMPT_TEMPLATE = '''
你是一个顶级的AI规划专家。你的任务是将用户提出的复杂问题分解成一个由多个简单步骤组成的行动计划。
请确保计划中的每个步骤都是一个独立的、可执行的子任务，并且严格按照逻辑顺序排列。
你的输出必须是一个Python列表，其中每个元素都是一个描述子任务的字符串。

问题: {question}

请严格按照以下格式输出你的计划,```python与```作为前后缀是必要的:
```python
["步骤1", "步骤2", "步骤3", ...]
```
'''
````

plan部分的实现代码：

```python
class Planner:
    def __init__(self, llm_client):
        self.llm_client = llm_client

    def plan(self, question: str) -> list[str]:
        """
        根据用户问题生成一个行动计划。
        """
        prompt = PLANNER_PROMPT_TEMPLATE.format(question=question)
        
        # 为了生成计划，我们构建一个简单的消息列表
        messages = [{"role": "user", "content": prompt}]
        
        print("--- 正在生成计划 ---")
        # 使用流式输出来获取完整的计 划
        response_text = self.llm_client.think(messages=messages) or ""
        
        print(f"✅ 计划已生成:\n{response_text}")
        
        # 解析LLM输出的列表字符串
        try:
            # 找到```python和```之间的内容
            plan_str = response_text.split("```python")[1].split("```")[0].strip()
            # 使用ast.literal_eval来安全地执行字符串，将其转换为Python列表
            plan = ast.literal_eval(plan_str)
            return plan if isinstance(plan, list) else []
        except (ValueError, SyntaxError, IndexError) as e:
            print(f"❌ 解析计划时出错: {e}")
            print(f"原始响应: {response_text}")
            return []
        except Exception as e:
            print(f"❌ 解析计划时发生未知错误: {e}")
            return []
```

#### solve

计划好了之后,需要按照既定的计划去执行了，而对于执行部分，有一个需要注意的地方是：**状态管理** 后一步的执行需要依赖前面步骤的结果等等，因此需要做一个状态的管理。

对于执行器，是**在给定上下文的基础上，专注完成当前的这一个步骤**，因此prompt部分需要包含下面的关键信息：

- 原始问题：确保模型始终了解最终目标；
- 完整计划：让模型了解当前步骤在整个计划中的位置；
- 历史步骤与结果：之前的上下文
- 当前步骤：模型需要关注的当前任务：

因此执行器的提示词如下：

```python
EXECUTOR_PROMPT_TEMPLATE = """
你是一位顶级的AI执行专家。你的任务是严格按照给定的计划，一步步地解决问题。
你将收到原始问题、完整的计划、以及到目前为止已经完成的步骤和结果。
请你专注于解决“当前步骤”，并仅输出该步骤的最终答案，不要输出任何额外的解释或对话。

# 原始问题:
{question}

# 完整计划:
{plan}

# 历史步骤与结果:
{history}

# 当前步骤:
{current_step}

请仅输出针对“当前步骤”的回答:
"""

```

executor部分的代码如下：

```python
class Executor:
    def __init__(self, llm_client):
        self.llm_client = llm_client

    def execute(self, question: str, plan: list[str]) -> str:
        """
        根据计划，逐步执行并解决问题。
        """
        history = "" # 用于存储历史步骤和结果的字符串
        
        print("\n--- 正在执行计划 ---")
        
        for i, step in enumerate(plan):
            print(f"\n-> 正在执行步骤 {i+1}/{len(plan)}: {step}")
            
            prompt = EXECUTOR_PROMPT_TEMPLATE.format(
                question=question,
                plan=plan,
                history=history if history else "无", # 如果是第一步，则历史为空
                current_step=step
            )
            
            messages = [{"role": "user", "content": prompt}]
            
            response_text = self.llm_client.think(messages=messages) or ""
            
            # 更新历史记录，为下一步做准备
            history += f"步骤 {i+1}: {step}\n结果: {response_text}\n\n"
            
            print(f"✅ 步骤 {i+1} 已完成，结果: {response_text}")

        # 循环结束后，最后一步的响应就是最终答案
        final_answer = response_text
        return final_answer

```

#### plan and solve

前面有了plan和solve的模块后，现在将他们整合到一起就可以了,就可以合成一个完整的agent了

```python
class PlanAndSolveAgent:
    def __init__(self, llm_client):
        """
        初始化智能体，同时创建规划器和执行器实例。
        """
        self.llm_client = llm_client
        self.planner = Planner(self.llm_client)
        self.executor = Executor(self.llm_client)

    def run(self, question: str):
        """
        运行智能体的完整流程:先规划，后执行。
        """
        print(f"\n--- 开始处理问题 ---\n问题: {question}")
        
        # 1. 调用规划器生成计划
        plan = self.planner.plan(question)
        
        # 检查计划是否成功生成
        if not plan:
            print("\n--- 任务终止 --- \n无法生成有效的行动计划。")
            return

        # 2. 调用执行器执行计划
        final_answer = self.executor.execute(question, plan)
        
        print(f"\n--- 任务完成 ---\n最终答案: {final_answer}")
```



### 范式-- Reflection

> 后知后觉：我在翻译项目上的三段式翻译，其实就是使用到了这种范式Reflection

就像人类对于一个问题的思考一样，会先给出一个初步的答案，然后在这个答案上进行反思，然后进行改进；Reflection的思想也基于此，Reflection的核心流程可以概括为一个简洁的三步循环：执行 ---> 反思 ---> 优化

1. **执行 (Execution)**：首先，智能体使用我们熟悉的方法（如 ReAct 或 Plan-and-Solve）尝试完成任务，生成一个初步的解决方案或行动轨迹。这可以看作是“初稿”。
2. **反思 (Reflection)**：接着，智能体进入反思阶段。它会调用一个独立的、或者带有特殊提示词的大语言模型实例，来扮演一个“评审员”的角色。这个“评审员”会审视第一步生成的“初稿”，并从多个维度进行评估，例如：
   - **事实性错误**：是否存在与常识或已知事实相悖的内容？
   - **逻辑漏洞**：推理过程是否存在不连贯或矛盾之处？
   - **效率问题**：是否有更直接、更简洁的路径来完成任务？
   - **遗漏信息**：是否忽略了问题的某些关键约束或方面？ 根据评估，它会生成一段结构化的**反馈 (Feedback)**，指出具体的问题所在和改进建议。
3. **优化 (Refinement)**：最后，智能体将“初稿”和“反馈”作为新的上下文，再次调用大语言模型，要求它根据反馈内容对初稿进行修正，生成一个更完善的“修订稿”。

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260127112311466.png" alt="image-20260127112311466" style="zoom:67%;" />

#### 记忆模块

Reflection 的核心在于迭代，而迭代的前提是能够记住之前的尝试和获得的反馈。因此，一个“短期记忆”模块是实现该范式的必需品。这个记忆模块将负责存储每一次“执行-反思”循环的完整轨迹。

```python
from typing import List, Dict, Any, Optional

class Memory:
    """
    一个简单的短期记忆模块，用于存储智能体的行动与反思轨迹。
    """

    def __init__(self):
        """
        初始化一个空列表来存储所有记录。
        """
        self.records: List[Dict[str, Any]] = []

    def add_record(self, record_type: str, content: str):
        """
        向记忆中添加一条新记录。

        参数:
        - record_type (str): 记录的类型 ('execution' 或 'reflection')。
        - content (str): 记录的具体内容 (例如，生成的代码或反思的反馈)。
        """
        record = {"type": record_type, "content": content}
        self.records.append(record)
        print(f"📝 记忆已更新，新增一条 '{record_type}' 记录。")

    def get_trajectory(self) -> str:
        """
        将所有记忆记录格式化为一个连贯的字符串文本，用于构建提示词。
        """
        trajectory_parts = []
        for record in self.records:
            if record['type'] == 'execution':
                trajectory_parts.append(f"--- 上一轮尝试 (代码) ---\n{record['content']}")
            elif record['type'] == 'reflection':
                trajectory_parts.append(f"--- 评审员反馈 ---\n{record['content']}")
        
        return "\n\n".join(trajectory_parts)

    def get_last_execution(self) -> Optional[str]:
        """
        获取最近一次的执行结果 (例如，最新生成的代码)。
        如果不存在，则返回 None。
        """
        for record in reversed(self.records):
            if record['type'] == 'execution':
                return record['content']
        return None

```

#### 初始、反思、优化的prompt

假设我们的场景是写代码的场景，我们需要一个初始的执行的提示词

```python
INITIAL_PROMPT_TEMPLATE = """
你是一位资深的Python程序员。请根据以下要求，编写一个Python函数。
你的代码必须包含完整的函数签名、文档字符串，并遵循PEP 8编码规范。

要求: {task}

请直接输出代码，不要包含任何额外的解释。
"""
```

- **反思提示词 (Reflection Prompt)** ：这个提示词是 Reflection 机制的灵魂。它指示模型扮演“代码评审员”的角色，对上一轮生成的代码进行批判性分析，并提供具体的、可操作的反馈。

````python
REFLECT_PROMPT_TEMPLATE = """
你是一位极其严格的代码评审专家和资深算法工程师，对代码的性能有极致的要求。
你的任务是审查以下Python代码，并专注于找出其在<strong>算法效率</strong>上的主要瓶颈。

# 原始任务:
{task}

# 待审查的代码:
```python
{code}
```

请分析该代码的时间复杂度，并思考是否存在一种<strong>算法上更优</strong>的解决方案来显著提升性能。
如果存在，请清晰地指出当前算法的不足，并提出具体的、可行的改进算法建议（例如，使用筛法替代试除法）。
如果代码在算法层面已经达到最优，才能回答“无需改进”。

请直接输出你的反馈，不要包含任何额外的解释。
"""

````

- **优化提示词 (Refinement Prompt)** ：当收到反馈后，这个提示词将引导模型根据反馈内容，对原有代码进行修正和优化。

```python

REFINE_PROMPT_TEMPLATE = """
你是一位资深的Python程序员。你正在根据一位代码评审专家的反馈来优化你的代码。

# 原始任务:
{task}

# 你上一轮尝试的代码:
{last_code_attempt}
评审员的反馈：
{feedback}

请根据评审员的反馈，生成一个优化后的新版本代码。
你的代码必须包含完整的函数签名、文档字符串，并遵循PEP 8编码规范。
请直接输出优化后的代码，不要包含任何额外的解释。
"""

```

#### 整合

```python
class ReflectionAgent:
    def __init__(self, llm_client, max_iterations=3):
        self.llm_client = llm_client
        self.memory = Memory()
        self.max_iterations = max_iterations

    def run(self, task: str):
        print(f"\n--- 开始处理任务 ---\n任务: {task}")

        # --- 1. 初始执行 ---
        print("\n--- 正在进行初始尝试 ---")
        initial_prompt = INITIAL_PROMPT_TEMPLATE.format(task=task)
        initial_code = self._get_llm_response(initial_prompt)
        self.memory.add_record("execution", initial_code)

        # --- 2. 迭代循环:反思与优化 ---
        for i in range(self.max_iterations):
            print(f"\n--- 第 {i+1}/{self.max_iterations} 轮迭代 ---")

            # a. 反思
            print("\n-> 正在进行反思...")
            last_code = self.memory.get_last_execution()
            reflect_prompt = REFLECT_PROMPT_TEMPLATE.format(task=task, code=last_code)
            feedback = self._get_llm_response(reflect_prompt)
            self.memory.add_record("reflection", feedback)

            # b. 检查是否需要停止
            if "无需改进" in feedback:
                print("\n✅ 反思认为代码已无需改进，任务完成。")
                break

            # c. 优化
            print("\n-> 正在进行优化...")
            refine_prompt = REFINE_PROMPT_TEMPLATE.format(
                task=task,
                last_code_attempt=last_code,
                feedback=feedback
            )
            refined_code = self._get_llm_response(refine_prompt)
            self.memory.add_record("execution", refined_code)
        
        final_code = self.memory.get_last_execution()
        print(f"\n--- 任务完成 ---\n最终生成的代码:\n```python\n{final_code}\n```")
        return final_code

    def _get_llm_response(self, prompt: str) -> str:
        """一个辅助方法，用于调用LLM并获取完整的流式响应。"""
        messages = [{"role": "user", "content": prompt}]
        response_text = self.llm_client.think(messages=messages) or ""
        return response_text

   
```



### 范式-Pi Agent

> 知乎说明：https://zhuanlan.zhihu.com/p/2004665077618458930
>
> 项目地址：https://github.com/badlogic/pi-mono





### learn-claude-code

> https://github.com/shareAI-lab/learn-claude-code

#### todo的实现方法(s03小节)

主要还是通过工具的方式进行实现，实现了一个todo的工具，然后llm去调用

![image-20260227111312219](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260227111312219.png)

需要注意的是，有一个小点，如上图，添加了一个强制更新的代码，让llm定期更新todo，具体代码如下：是在用户消息前面添加了一个调用todo工具的提醒(这种方式，我感觉是不是对于用户的体验不是太好呀，必须用户下一次的输出的时候才更新todo，不具备实时性，cline的体验不是这样的，可以具体的去看看cline在这方面的实现，不过思想上可以学习一下)

```python
    while True:
        # Nag reminder: if 3+ rounds without a todo update, inject reminder
        if rounds_since_todo >= 3 and messages:
            last = messages[-1]
            if last["role"] == "user" and isinstance(last.get("content"), list):
                last["content"].insert(0, {"type": "text", "text": "<reminder>Update your todos.</reminder>"})
```



#### subagent (s04小节)

![image-20260227112546053](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260227112546053.png)

其实也是类似工具调用的方式来实现的，demo代码给了一个简单示例，感觉应该需要指明subagent的能力范围，不然和主agent重合，感觉会一直不会调用subagent，具体相关实现部分如下:

```python
# -- Parent tools: base tools + task dispatcher --
PARENT_TOOLS = CHILD_TOOLS + [
    {"name": "task", "description": "Spawn a subagent with fresh context. It shares the filesystem but not conversation history.",
     "input_schema": {"type": "object", "properties": {"prompt": {"type": "string"}, "description": {"type": "string", "description": "Short description of the task"}}, "required": ["prompt"]}},
]


def agent_loop(messages: list):
    while True:
        response = client.messages.create(
            model=MODEL, system=SYSTEM, messages=messages,
            tools=PARENT_TOOLS, max_tokens=8000,
        )
        messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":
            return
        results = []
        for block in response.content:
            if block.type == "tool_use":
                # 当工具调用输出为task时，即subagent的工具时，会调用写好的subagent的代码
                if block.name == "task":
                    desc = block.input.get("description", "subtask")
                    print(f"> task ({desc}): {block.input['prompt'][:80]}")
                    output = run_subagent(block.input["prompt"])
                else:
                    handler = TOOL_HANDLERS.get(block.name)
                    output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
                print(f"  {str(output)[:200]}")
                results.append({"type": "tool_result", "tool_use_id": block.id, "content": str(output)})
        messages.append({"role": "user", "content": results})
```

subagent代码和主agent代码一样,也是一个agentloop

```python
# -- Subagent: fresh context, filtered tools, summary-only return --
def run_subagent(prompt: str) -> str:
    sub_messages = [{"role": "user", "content": prompt}]  # fresh context
    for _ in range(30):  # safety limit
        response = client.messages.create(
            model=MODEL, system=SUBAGENT_SYSTEM, messages=sub_messages,
            tools=CHILD_TOOLS, max_tokens=8000,
        )
        sub_messages.append({"role": "assistant", "content": response.content})
        if response.stop_reason != "tool_use":
            break
        results = []
        for block in response.content:
            if block.type == "tool_use":
                handler = TOOL_HANDLERS.get(block.name)
                output = handler(**block.input) if handler else f"Unknown tool: {block.name}"
                results.append({"type": "tool_result", "tool_use_id": block.id, "content": str(output)[:50000]})
        sub_messages.append({"role": "user", "content": results})
    # Only the final text returns to the parent -- child context is discarded
    return "".join(b.text for b in response.content if hasattr(b, "text")) or "(no summary)"
```



#### 压缩上下文(s06小节)

![image-20260227114313837](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260227114313837.png)

其实，也一样，造一个compact的tool给到llm调用即可

```python
def auto_compact(messages: list) -> list:
    # Save full transcript to disk
    TRANSCRIPT_DIR.mkdir(exist_ok=True)
    transcript_path = TRANSCRIPT_DIR / f"transcript_{int(time.time())}.jsonl"
    with open(transcript_path, "w") as f:
        for msg in messages:
            f.write(json.dumps(msg, default=str) + "\n")
    print(f"[transcript saved: {transcript_path}]")
    # Ask LLM to summarize
    conversation_text = json.dumps(messages, default=str)[:80000]
    response = client.messages.create(
        model=MODEL,
        messages=[{"role": "user", "content":
            "Summarize this conversation for continuity. Include: "
            "1) What was accomplished, 2) Current state, 3) Key decisions made. "
            "Be concise but preserve critical details.\n\n" + conversation_text}],
        max_tokens=2000,
    )
    summary = response.content[0].text
    # Replace all messages with compressed summary
    return [
        {"role": "user", "content": f"[Conversation compressed. Transcript: {transcript_path}]\n\n{summary}"},
        {"role": "assistant", "content": "Understood. I have the context from the summary. Continuing."},
    ]
```

需要注意上面代码的最后一部分,手动创造了一个对话消息=_=#, 这个做法当时和我做llm给webp动图打标，想要给参考示例时的做法一致，至于cline，或者claude里面是否是这样做的，还需要再看一下

```python
    return [
        {"role": "user", "content": f"[Conversation compressed. Transcript: {transcript_path}]\n\n{summary}"},
        {"role": "assistant", "content": "Understood. I have the context from the summary. Continuing."},
    ]
```

#### agent teams(s09、s10小节 )

主要可以学习的部分在于一个MessageBUS（信箱的思想，每个agent都有各自的信箱，demo中的信箱是文件），主要思想就是通过一个lead派发任务给不同的子成员，并且将对应的任务写到子成员的文件中(通过send_message函数来实现)

![image-20260302120116534](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260302120116534.png)





### **Yunjue Agent**

> https://github.com/YunjueTech/Yunjue-Agent/blob/main/README_zh.md
>
> [从零开始构建自进化智能体的心路历程](https://mp.weixin.qq.com/s/-Kt5SMUcXRrT5ECI45bb3w)



### Harness

> https://github.com/HKUDS/OpenHarness

### 自进化Agent

> [一文搞懂Hermes：新顶流Agent如何从经验中自我进化](https://mp.weixin.qq.com/s/yHva-zLaRTxe8b4HSUr86Q?poc_token=HIjy3mmjYAPhz51RpL_Xx7T4pqjqe2ggbF0FqdSU)

![image-20260415101358284](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260415101358284.png)

用一句话概括这个系统的本质：Skills 系统让 AI Agent 像人类专家一样积累经验——把成功的做法写成 SOP，在使用中持续修订，并且可以分享给其他人。

#### 为什么不能每次都扫描文件系统？？

一个用户可能有几十甚至上百个 Skill。每次对话启动时都去递归扫描 ~/.hermes/skills/ 目录、解析每个 SKILL.md 的 YAML frontmatter，这个开销不可忽视——尤其是在消息平台（Telegram、Discord）上，Gateway 进程需要同时服务多个用户的多个对话。

Hermes 的解决方案是两层缓存：

![image-20260415101942128](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260415101942128.png)





### 可参考资料

#### 基础理论

1. https://github.com/datawhalechina/hello-agents
2. https://github.com/huggingface/agents-course
3. [如何从零开始实现一个 AI Agent 框架（理论+实践）](https://mp.weixin.qq.com/s/z1aDaPFhjYb2cv-9bFbdaQ)
4. [一文讲透如何构建Harness——六大组件全解析](https://mp.weixin.qq.com/s/HwqEaXSGkcYgUNrzB2okuA)
5. [逆向深扒Claude Code源码，我发现了什么！？](https://mp.weixin.qq.com/s/hskSjAkezaV2epVzUq6ziw)

#### 具体实践

1. https://github.com/NirDiamant/GenAI_Agents
1. **[learn-claude-code](https://github.com/shareAI-lab/learn-claude-code)**

#### Multi-Agent

1. [Choosing the Right Multi-Agent Architecture](https://www.blog.langchain.com/choosing-the-right-multi-agent-architecture/)

#### 开源项目

1. [clawbot](https://github.com/clawdbot/clawdbot/tree/main)
1. [解构Clawdbot：本地架构、记忆管理、Agent 编排与上下文组装原理](https://mp.weixin.qq.com/s/xfqcMeEEZ1kXth-cyREoow?poc_token=HKxdgWmjcv7M45KEAIc3IR4kArfcLlg0B-gMHmfn)
1. 实际项目中使用到plan and solve范式的： https://github.com/univa-agent/univa
1. [ChatDev](https://github.com/OpenBMB/ChatDev/tree/main)
1. 框架是plan execute reflection [[agentUniverse](https://github.com/agentuniverse-ai/agentUniverse)] 
1. 一个完整的前后端项目： [CoPaw](https://github.com/agentscope-ai/CoPaw)， 阿里基于agentscope agent和openclawed思想搭建的个人助手项目：是一个完整的前后端项目，后端是fastapi，前端是vite + React
1. claude code 源码：[盘一下Claude Code源码里的8个隐藏新功能。](https://mp.weixin.qq.com/s/5lAB7CPkX2FL6sH2W6H9wQ)   https://github.com/instructkr/claude-code

#### 其它

1. [关于Agent的思考（2025）](https://zhuanlan.zhihu.com/p/1905773415048143364?share_code=NQtZKL8Ohhxi&utm_psn=2004545549689435933)
2. [LLM as judge, 添加了一个反思流程](https://www.zhihu.com/pin/2004326834599388153?native=1&scene=share&share_code=1bKncEgBn7ASk&utm_psn=2004490813309208116)
3. [AI编程的下半场来了？学会用Agent Skill解决编程的痛点问题]()



