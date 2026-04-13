---
title: MCP
date: 2025-11-25 14:02:24
description: Agent相关协议概念汇总(如MCP,Cli, A2A等学习，记录)
cover: /covers/17.webp
tags: 
    - Agent
    - MCP
---

### MCP

话不多说，直接上图即可

![image-20250821171413971](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20250821171413971.png)

有MCP和没有MCP的区别，提升了效率

![image-20250821171443789](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20250821171443789.png)

MCP的架构图：主要是由Host、Client和Server三部分组成

![image-20250821171800487](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20250821171800487.png)

 


#### 个人demo
> [mcp demo](https://github.com/flippy-bird/agent/tree/main/examples/mcp_demo)



### MCP-> CLI

 CLI: Agent训练时包含大量的代码，CLI天生就是 LLM的母语，GUI是方便人类使用，CLI是方便Agent使用；

- MCP需要一次性列出所有工具的说明和介绍，但是有时候并不需要这么多的工具，因此比较耗费token (当然现在也有一些公司将MCP按照渐进式披露的思想进行分批加载，但是目前主流的MCP做法还是一次性暴露所有的工具)
- CLI是渐进式的披露的思想，当需要调用某个命令的时候，可以 --help来加载命令的参数，因此可以有效减少token的消耗；



### 参考资料

#### MCP

1. [MCP (Model Context Protocol)，一篇就够了。](https://zhuanlan.zhihu.com/p/29001189476)
2. [python SDK的官方文档](https://github.com/modelcontextprotocol/python-sdk)
3. 测试MCP工具接入使用的地址：[阿里MCP](https://bailian.console.aliyun.com/?utm_content=se_1021227952&gclid=EAIaIQobChMI28G-yLObjwMV3JC5BR0zMijXEAAYASAAEgKc4_D_BwE&tab=mcp#/mcp-market)

#### CLI

1. [CLI-Anything: Making ALL Software Agent-Native](https://github.com/HKUDS/CLI-Anything)  
2. [opencli](https://github.com/jackwener/opencli)

