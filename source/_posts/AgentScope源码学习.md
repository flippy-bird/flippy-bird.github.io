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

具体实现的代码大概如下：(代码路径：agentscope/src/agentscope/agent/_agent_base.py),  首先这里的 self. _reply_task获取到当前的任务，用于后面的中断处理，从代码也可以看到，self.reply里面发生了中断会触发 raise asyncio.CancellerError，然后这个函数去捕获，然后在这个函数里面去触发执行handle_interrupt的任务 （这个可以在react_agent里面的reasoning和acting步骤可以找到例子）

```python
async def __call__(self, *args: Any, **kwargs: Any) -> Msg:
    """Call the reply function with the given arguments."""
    self._reply_id = shortuuid.uuid()

    reply_msg: Msg | None = None
    try:
        self._reply_task = asyncio.current_task()
        reply_msg = await self.reply(*args, **kwargs)

    # The interruption is triggered by calling the interrupt method
    except asyncio.CancelledError:
        reply_msg = await self.handle_interrupt(*args, **kwargs)

    finally:
        # Broadcast the reply message to all subscribers
        if reply_msg:
            await self._broadcast_to_subscribers(reply_msg)
        self._reply_task = None

    return reply_msg
```

如果存在发生了中断，使用task.cancel()函数取消任务

```python
async def interrupt(self, msg: Msg | list[Msg] | None = None) -> None:
    """Interrupt the current reply process."""
    if self._reply_task and not self._reply_task.done():
        self._reply_task.cancel(msg)
```





#### Hooks

使用了python元编程来实现，在元类中添加一些wrap类或者函数，使得在**函数调用前后会执行一段注入的代码**，其实现主要是在/src/agentscope/agent/_agent_meta.py这个文件的 _wrap_with_hooks这个函数, 从下面的代码可以看到，依次执行pre-hooks, 原先的函数，post-hooks

```python
def _wrap_with_hooks(
    original_func: Callable,
) -> Callable:
    """A decorator to wrap the original async function with pre- and post-hooks

    Args:
        original_func (`Callable`):
            The original async function to be wrapped with hooks.
    """
    func_name = original_func.__name__.replace("_", "")

    @wraps(original_func)
    async def async_wrapper(
        self: AgentBase,
        *args: Any,
        **kwargs: Any,
    ) -> Any:
        """The wrapped function, which call the pre- and post-hooks before and
        after the original function."""

        # Unify all positional and keyword arguments into a keyword arguments
        normalized_kwargs = _normalize_to_kwargs(
            original_func,
            self,
            *args,
            **kwargs,
        )

        current_normalized_kwargs = normalized_kwargs

        # pre-hooks
        pre_hooks = list(
            getattr(self, f"_instance_pre_{func_name}_hooks").values(),
        ) + list(
            getattr(self, f"_class_pre_{func_name}_hooks").values(),
        )
        for pre_hook in pre_hooks:
            modified_keywords = await _execute_async_or_sync_func(
                pre_hook,
                self,
                deepcopy(current_normalized_kwargs),
            )
            if modified_keywords is not None:
                assert isinstance(modified_keywords, dict), (
                    f"Pre-hook must return a dict of keyword arguments, rather"
                    f" than {type(modified_keywords)} from hook "
                    f"{pre_hook.__name__}"
                )
                current_normalized_kwargs = modified_keywords

        # original function
        # handle positional and keyword arguments specifically
        args = current_normalized_kwargs.get("args", [])
        kwargs = current_normalized_kwargs.get("kwargs", {})
        others = {
            k: v
            for k, v in current_normalized_kwargs.items()
            if k not in ["args", "kwargs"]
        }
        current_output = await original_func(
            self,
            *args,
            **others,
            **kwargs,
        )

        # post_hooks
        post_hooks = list(
            getattr(self, f"_instance_post_{func_name}_hooks").values(),
        ) + list(
            getattr(self, f"_class_post_{func_name}_hooks").values(),
        )
        for post_hook in post_hooks:
            modified_output = await _execute_async_or_sync_func(
                post_hook,
                self,
                deepcopy(current_normalized_kwargs),
                deepcopy(current_output),
            )
            if modified_output is not None:
                current_output = modified_output
        return current_output

    return async_wrapper
```



#### Agent Skill

skill的思想是动态加载propmt，信息，我先跑了一个agentscope官方的例子，可以看到，当需要读取skill更多的内容时，使用框架内置的工具view_text_file读取SKILL.md中的内容，然后使用内置的bash工具execute_shell_command来执行相应的脚本

![image-20260122103559281](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260122103559281.png)

分析了SKILL.md的全部内容之后，按照SKILL.md里面的说明来进行工作了

![image-20260122103937813](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260122103937813.png)

对于代码的实现方面来看，agentscope在原先system prompt基础上拼接了skill相关的prompt

src/agentscope/agent/_react_agent.py

![image-20260122111910857](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260122111910857.png)

![image-20260122112043982](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260122112043982.png)

把这个prompt加载到系统的上下文之后，就不需要做其他事情了，因为上面的prompt也说明了，如果需要使用skill，那么需要应该需要调用 查看内容相关的工具去读取SKILL.md的内容，然后再进行下一步的操作了。这个实现还是蛮直观的



#### Plan的实现

实现了一些plan的功能函数，然后让Agent去调用和更改当前的plan

![image-20251119115208155](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20251119115208155.png)

### 其它

字节也出了一个类似框架：[veadk-python](https://github.com/volcengine/veadk-python)

