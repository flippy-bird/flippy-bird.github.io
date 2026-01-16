---
title: langgraph官方文档学习
date: 2025-04-08 14:58:33
description: 长期不使用langgraph，导致又忘记了，写个blog记录一下，方面快速回顾
cover: /covers/13.webp
tags:
    - Agent
    - langgraph
---

顺序可能不按照官方文档的顺序，按照自己感兴趣的模块，然后通过demo 代码的方式来理解这个模块的用法

### subgraph

在langgraph里面，可以通过两种方式将subgraph添加到 流程中

- 通过工具调用的方式引入 (invoke a graph from a node)

例如下面的官方例子中，subgraph通过主流程中call_subgraph这个tools函数来触发subgraph的调用

```python
from typing_extensions import TypedDict
from langgraph.graph.state import StateGraph, START

class SubgraphState(TypedDict):
    bar: str

# Subgraph

def subgraph_node_1(state: SubgraphState):
    return {"bar": "hi! " + state["bar"]}

subgraph_builder = StateGraph(SubgraphState)
subgraph_builder.add_node(subgraph_node_1)
subgraph_builder.add_edge(START, "subgraph_node_1")
subgraph = subgraph_builder.compile()

# Parent graph

class State(TypedDict):
    foo: str

def call_subgraph(state: State):   
    # Transform the state to the subgraph state
    subgraph_output = subgraph.invoke({"bar": state["foo"]})  
    # Transform response back to the parent state
    return {"foo": subgraph_output["bar"]}

builder = StateGraph(State)
builder.add_node("node_1", call_subgraph)
builder.add_edge(START, "node_1")
graph = builder.compile()
```

- 直接以子节点的方式添加到流程中

```python
from typing_extensions import TypedDict
from langgraph.graph.state import StateGraph, START

class State(TypedDict):
    foo: str

# Subgraph

def subgraph_node_1(state: State):
    return {"foo": "hi! " + state["foo"]}

subgraph_builder = StateGraph(State)
subgraph_builder.add_node(subgraph_node_1)
subgraph_builder.add_edge(START, "subgraph_node_1")
subgraph = subgraph_builder.compile()

# Parent graph

builder = StateGraph(State)
builder.add_node("node_1", subgraph)  
builder.add_edge(START, "node_1")
graph = builder.compile()
```

**通过传参也可以看到，子节点的方式，subgraph共享主graph的State**



### human in loop



### 可学习资料
1.[Multi-Agent全面爆发！一文详解多智能体核心架构及LangGraph框架](https://mp.weixin.qq.com/s/XhFbLTLcSjDj0r3KGT9EOg)
2.[破除AI Agent自主操控风险：万字解读LangGraph“人工干预”机制 ，附零基础实战](https://mp.weixin.qq.com/s/m37YhDsKsQn2HWuV7LKwgg)
