---
title: Vibe Coding 相关
date: 2026-01-14 16:10:43
description: 总结一些Vibe Coding的一些使用经验
tags:
    - vibe coding
categories:
    - 工具使用
---

### Cursor使用

> 参考cursor官方文档：https://cursor.com/cn/docs

#### 规则

规则的工作原理：规则在提示级别提供持久、可重用的上下文。应用后，规则内容会被加入到模型上下文的开头。这为 AI 在生成代码、理解编辑或协助处理工作流时提供一致的指导。

cursor提供了四种规则：项目规则、用户规则、团队规则、AGENTS.MD

##### 项目规则

项目规则以 markdown 文件形式存放在 `.cursor/rules` 中，并纳入版本控制。规则可以通过路径模式限定作用范围，主要有以下的作用：

- 沉淀与你代码库相关的领域知识
- 自动化项目特定的工作流或模板
- 统一风格或架构决策

```python
.cursor/rules/
  react-patterns.mdc       # 带前置元数据的规则(描述、globs)
  api-guidelines.md        # 简单 markdown 规则
  frontend/                # 在文件夹中组织规则
    components.md
```

而对于每个markdown的内容，如下

```markdown
---
description: "This rule provides standards for frontend components and API validation"
alwaysApply: false
---

...rest of the rule content
```

##### 用户规则

应用于全局的，将在所有项目上应用，这个在settings --> Rules 里面，如下面的例子：

![image-20260116180357582](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260116180357582.png)

可以看到，因为我设置了这个规则，我和cursor交互时(chat时候)，返回给我的都是中文回复；

##### AGENTS.MD

`AGENTS.md` 是一个用于定义 agent 指令的简单 markdown 文件。将它放在项目根目录中，作为 `.cursor/rules` 的替代选项，适用于简单直接的用例。**`AGENTS.md` 是一个没有元数据或复杂配置的纯 markdown 文件。对于只需要简单、易读指令**，而不想引入结构化规则额外负担的项目来说，它是理想选择。

官方的一个简单例子如下：

```markdown
# Project Instructions

## Code Style

- Use TypeScript for all new files
- Prefer functional components in React
- Use snake_case for database columns

## Architecture

- Follow the repository pattern
- Keep business logic in service layers
```

当然也是支持嵌套的

```python
project/
  AGENTS.md              # 全局指令
  frontend/
    AGENTS.md            # 前端专用指令
    components/
      AGENTS.md          # 组件专用指令
  backend/
    AGENTS.md            # 后端专用指令
```



#### skills 支持

技能何时工作？Cursor 启动时会自动从技能目录中发现技能，并将它们提供给 Agent。Agent 会查看可用的技能，并根据上下文决定何时使用。

##### 技能加载目录

![image-20260116181303076](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260116181303076.png)

每个技能都应是一个包含 `SKILL.md` 文件的文件夹：

```python
.cursor/
└── skills/
    └── my-skill/
        └── SKILL.md
```

关于SKILL.md的格式，下面是一个简单的例子,当然在my-skill文件夹可以添加一些资源，脚本等等文件夹，这个可以查看claude的标准来创建自己的skills

```markdown
---
name: my-skill
description: Short description of what this skill does and when to use it.
---

# My Skill

Detailed instructions for the agent.

## When to Use

- Use this skill when...
- This skill is helpful for...

## Instructions

- Step-by-step guidance for the agent
- Domain-specific conventions
- Best practices and patterns
```







### 参考资料

1. [认知重建：Speckit 用了三个月，我放弃了——走出工具很强但用不好的困境](https://zhuanlan.zhihu.com/p/1993009461451831150)
