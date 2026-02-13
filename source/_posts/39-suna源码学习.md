---
title: suna源码学习
date: 2026-02-13 13:46:55
description: suna一个比较完整的agent项目，包含前后端，学习一下
cover: /covers/2.webp
tags:
    - agent
categories:
    - Agent开源项目学习
---



### 源码阅读

这个项目的后端确实比较全面，模块也是真的多，代码量也是真的大

> 废话部分：后端部分使用python实现，但是看出了一股js的味道，每个模块有个聚合的api.py，这个不就是js项目里面的index.ts嘛=_=#

#### agent部分

OK,先找到源码Agent部分，位于`backend/core/agents/`目录下，既然这个目录下有`api.py`文件，这个应该是入口，从这个文件出发，找了一个`unified_agent_start`函数，里面实际执行的函数是`start_agent_run`这个函数，然后这个函数里面实际会执行操作的函数是`_background_setup_and_execute`

