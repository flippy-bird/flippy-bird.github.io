---
title: LLM的plan调研
date: 2025-12-08 14:58:33
description: 对于agent的规划部分比较感兴趣，因此看了一下目前各个开源方案
tags:
    - Agent
    - plan
---



### 1. langchain/langgraph

> 参考资料：
>
> 1. https://docs.langchain.com/oss/python/langchain/middleware/built-in#to-do-list

langchain的思想是通过middleware的形式来添加plan的能力，可以参考源码部分：langchain/libs/langchain_v1/langchain/agents/middleware/todo.py （12月8日，后续不知道会不会变化）**其本质仍然是一个工具函数的调用，内部的一个工具是write_todos**, 相关的middleware是 TodoListMiddleware

这个工具的具体定义是：关于write_todos这个工具的描述，这个描述写的很长，具体可以去源码查看；

```python
@tool(description=WRITE_TODOS_TOOL_DESCRIPTION)
def write_todos(todos: list[Todo], tool_call_id: Annotated[str, InjectedToolCallId]) -> Command:
    """Create and manage a structured task list for your current work session."""
    return Command(
        update={
            "todos": todos,
            "messages": [ToolMessage(f"Updated todo list to {todos}", tool_call_id=tool_call_id)],
        }
    )
```

system prompt是：

```python
WRITE_TODOS_SYSTEM_PROMPT = """## `write_todos`

You have access to the `write_todos` tool to help you manage and plan complex objectives.
Use this tool for complex objectives to ensure that you are tracking each necessary step and giving the user visibility into your progress.
This tool is very helpful for planning complex objectives, and for breaking down these larger complex objectives into smaller steps.

It is critical that you mark todos as completed as soon as you are done with a step. Do not batch up multiple steps before marking them as completed.
For simple objectives that only require a few steps, it is better to just complete the objective directly and NOT use this tool.
Writing todos takes time and tokens, use it when it is helpful for managing complex many-step problems! But not for simple few-step requests.

## Important To-Do List Usage Notes to Remember
- The `write_todos` tool should never be called multiple times in parallel.
- Don't be afraid to revise the To-Do list as you go. New information may reveal new tasks that need to be done, or old tasks that are irrelevant."""
```



### 2. Qwen code

> https://github.com/QwenLM/qwen-code

采用的是内置工具的方式来实现agent的plan能力，源代码部分可参考：qwen-code/packages/core/src/tools/todoWrite.ts

具体的提供了下面的todo工具类：

```typescript
class TodoWriteToolInvocation extends BaseToolInvocation<
  TodoWriteParams,
  ToolResult
> {}

// 暴露给外面的todo工具类
export class TodoWriteTool extends BaseDeclarativeTool<
  TodoWriteParams,
  ToolResult
> {}
```

关于工具的描述，截取一部分prompt如下：

```typescript
const todoWriteToolDescription = `
Use this tool to create and manage a structured task list for your current coding session. This helps you track progress, organize complex tasks, and demonstrate thoroughness to the user.
It also helps the user understand the progress of the task and overall progress of their requests.

## When to Use This Tool
Use this tool proactively in these scenarios:

1. Complex multi-step tasks - When a task requires 3 or more distinct steps or actions
2. Non-trivial and complex tasks - Tasks that require careful planning or multiple operations
3. User explicitly requests todo list - When the user directly asks you to use the todo list
4. User provides multiple tasks - When users provide a list of things to be done (numbered or comma-separated)
5. After receiving new instructions - Immediately capture user requirements as todos
6. When you start working on a task - Mark it as in_progress BEFORE beginning work. Ideally you should only have one todo as in_progress at a time
7. After completing a task - Mark it as completed and add any new follow-up tasks discovered during implementation

## When NOT to Use This Tool

Skip using this tool when:
1. There is only a single, straightforward task
2. The task is trivial and tracking it provides no organizational benefit
3. The task can be completed in less than 3 trivial steps
4. The task is purely conversational or informational

NOTE that you should not use this tool if there is only one trivial task to do. In this case you are better off just doing the task directly.

## Examples of When to Use the Todo List 
   ...
`;

```



### 3. gemini cli

和qwen code 类似，源码部分在：gemini-cli/packages/core/src/tools/write-todos.ts

```typescript
export class WriteTodosTool extends BaseDeclarativeTool<
  WriteTodosToolParams,
  ToolResult
> {}
```

工具说明的prompt截取如下：

```typescript
export const WRITE_TODOS_DESCRIPTION = `This tool can help you list out the current subtasks that are required to be completed for a given user request. The list of subtasks helps you keep track of the current task, organize complex queries and help ensure that you don't miss any steps. With this list, the user can also see the current progress you are making in executing a given task.

Depending on the task complexity, you should first divide a given task into subtasks and then use this tool to list out the subtasks that are required to be completed for a given user request.
Each of the subtasks should be clear and distinct. 

Use this tool for complex queries that require multiple steps. If you find that the request is actually complex after you have started executing the user task, create a todo list and use it. If execution of the user task requires multiple steps, planning and generally is higher complexity than a simple Q&A, use this tool.

DO NOT use this tool for simple tasks that can be completed in less than 2 steps. If the user query is simple and straightforward, do not use the tool. If you can respond with an answer in a single turn then this tool is not required.

## Task state definitions

- pending: Work has not begun on a given subtask.
- in_progress: Marked just prior to beginning work on a given subtask. You should only have one subtask as in_progress at a time.
- completed: Subtask was successfully completed with no errors or issues. If the subtask required more steps to complete, update the todo list with the subtasks. All steps should be identified as completed only when they are completed.
- cancelled: As you update the todo list, some tasks are not required anymore due to the dynamic nature of the task. In this case, mark the subtasks as cancelled.


## Methodology for using this tool
1. Use this todo list as soon as you receive a user request based on the complexity of the task.
2. Keep track of every subtask that you update the list with.
3. Mark a subtask as in_progress before you begin working on it. You should only have one subtask as in_progress at a time.
4. Update the subtask list as you proceed in executing the task. The subtask list is not static and should reflect your progress and current plans, which may evolve as you acquire new information.
5. Mark a subtask as completed when you have completed it.
6. Mark a subtask as cancelled if the subtask is no longer needed.
7. You must update the todo list as soon as you start, stop or cancel a subtask. Don't batch or wait to update the todo list.


## Examples of When to Use the Todo List
 ...
`;
```



### 4. AgentScope

> https://github.com/agentscope-ai/agentscope

也是通过工具调用的方式来实现的，可以参考我之前的文档 [AgentScope源码学习](https://flippy-bird.github.io/2025/11/25/AgentScope%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0/)



### 5.cline

> https://github.com/cline/cline

应该也是通过调用的工具的方式让Agent具备了plan的能力，相关的提示词源码是：[cline/src/core/prompts/system-prompt/components/task_progress.ts](https://github.com/cline/cline/blob/main/src/core/prompts/system-prompt/components/task_progress.ts)

对应的prompt如下：可以看到，

```typescript
const UPDATING_TASK_PROGRESS = `UPDATING TASK PROGRESS

You can track and communicate your progress on the overall task using the task_progress parameter supported by every tool call. Using task_progress ensures you remain on task, and stay focused on completing the user's objective. This parameter can be used in any mode, and with any tool call.

- When switching from PLAN MODE to ACT MODE, you must create a comprehensive todo list for the task using the task_progress parameter
- Todo list updates should be done silently using the task_progress parameter - do not announce these updates to the user
- Use standard Markdown checklist format: "- [ ]" for incomplete items and "- [x]" for completed items
- Keep items focused on meaningful progress milestones rather than minor technical details. The checklist should not be so granular that minor implementation details clutter the progress tracking.
- For simple tasks, short checklists with even a single item are acceptable. For complex tasks, avoid making the checklist too long or verbose.
- If you are creating this checklist for the first time, and the tool use completes the first step in the checklist, make sure to mark it as completed in your task_progress parameter.
- Provide the whole checklist of steps you intend to complete in the task, and keep the checkboxes updated as you make progress. It's okay to rewrite this checklist as needed if it becomes invalid due to scope changes or new information.
- If a checklist is being used, be sure to update it any time a step has been completed.
- The system will automatically include todo list context in your prompts when appropriate - these reminders are important.

Example:
<execute_command>
<command>npm install react</command>
<requires_approval>false</requires_approval>
<task_progress>
- [x] Set up project structure
- [x] Install dependencies
- [ ] Create components
- [ ] Test application
</task_progress>
</execute_command>`
```

对于Vscode中ide部分，貌似是通过一个focus chain模块来实现的，具体部分可参考源代码文件：

- cline/src/core/task/focus-chain/
- 
