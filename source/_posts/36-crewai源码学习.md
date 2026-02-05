---
title: crewai源码学习
date: 2026-02-03 11:07:51
description: 记录阅读crewai源码的笔记
tags:
    - agent
categories:
    - Agent开源项目学习
---







### 小技巧

#### 任务超时处理

如果是同步代码 (`对应crewAI/lib/crewai/src/crewai/agent/core.py 中的_execute_with_timeout函数`), 利用了`concurrent.futures`中的 future的特性，可以设置一个超时时间

```python
    import concurrent.futures

    with concurrent.futures.ThreadPoolExecutor() as executor:
        future = executor.submit(
            self._execute_without_timeout, task_prompt=task_prompt, task=task
        )

        try:
            return future.result(timeout=timeout)    
        except concurrent.futures.TimeoutError as e:
            future.cancel()
            raise TimeoutError(
                f"Task '{task.description}' execution timed out after {timeout} seconds. Consider increasing max_execution_time or optimizing the task."
            ) from e
        except Exception as e:
            future.cancel()
            raise RuntimeError(f"Task execution failed: {e!s}") from e
```

如果是异步代码，则更为简单,使用 `asyncio.wait_for`即可

```python
        try:
            return await asyncio.wait_for(
                self._aexecute_without_timeout(task_prompt, task),
                timeout=timeout,
            )
        except asyncio.TimeoutError as e:
            raise TimeoutError(
                f"Task '{task.description}' execution timed out after {timeout} seconds. "
                "Consider increasing max_execution_time or optimizing the task."
            ) from e
```



