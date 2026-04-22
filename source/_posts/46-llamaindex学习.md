---
title: llamaindex学习
date: 2024-12-22 09:58:47
description: llamaindex框架学习
tags:
    - RAG
---

### LLM

#### Tool Calling

```Python
from llama_index.core.tools import FunctionTool
from llama_index.llms.openai import OpenAI

def generate_song(name: str, artist: str) -> Song:
    """Generates a song with provided name and artist."""
    return {"name": name, "artist": artist}
    
tool = FunctionTool.from_defaults(fn=generate_song)
llm = OpenAI(model="gpt-4o")
response = llm.predict_and_call(
    [tool],
    "Pick a random song for me",
)
print(str(response))
```

#### 多模态

```Python
from llama_index.core.llms import ChatMessage, TextBlock, ImageBlock
from llama_index.llms.openai import OpenAI

llm = OpenAI(model="gpt-4o")

messages = [
    ChatMessage(
        role="user",
        blocks=[
           ImageBlock(path="image.png"),
           TextBlock(text="Desctibe the image in a few sentences"),],
    )
]

resp = llm.chat(messages)
print(resp.message.content)
```

### Agent

> [Llamahub ](https://llamahub.ai/?tab=agent)  上有很多现成可用的tools

![agent_flow](https://raw.githubusercontent.com/nashpan/image-hosting/main/agent_flow.png)

### State

这个概念类似于context上下文，有了上下文，相当于给我们的Agent添加上了Memory的功能，可以记忆或者加载之前的数据。

### API

#### Tavily 

> （LLamaindex 和langchain使用的 搜索引擎工具）

```Python
api-key = "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"
```

### RAG

#### 分层的思想

> [Multi-Document Agents (V1)](https://docs.llamaindex.ai/en/latest/examples/agent/multi_document_agents-v1/)

主要的思路就是：先找上一级的资料，然后找第二级相关的资料

![image-20260422100258041](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260422100258041.png)

### Bug记录

#### 搜索

使用tavily_tool工具时，如果使用原始的工具的名字，如下面的代码所示，最开始没有修改名字，会报下面的错误

![image_3](https://raw.githubusercontent.com/nashpan/image-hosting/main/image_3.png)

修改名字后可以正常运行(下面注释的部分)

```Python
llm = OpenAILike(
    model="qwen-max",
    api_base="https://dashscope.aliyuncs.com/compatible-mode/v1",
    api_key="sk-78827d8b494440c2b143642aae9ec42f",
    is_function_calling_model=True,
    is_chat_model=True
)

tavily_tool = TavilyToolSpec( api_key="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")
search_web = tavily_tool.to_tool_list()[0]
# search_web.metadata.name = "web_lookup"  # 改为更中性化的名字

workflow = AgentWorkflow.from_tools_or_functions(
    [search_web],
    llm=llm,
    system_prompt="You're a helpful assistant that can search the web for information."
)
```

#### 加载Html数据

![image_1](https://raw.githubusercontent.com/nashpan/image-hosting/main/image_1.png)

这里使用的是unstructed 这个库去读的html的内容，下面的代码加了split_documents之后就报了上面的错，不加就正常，暂时先不加，后面有时间看下源码解决一下。

```python
for idx, f in enumerate(useful_files):
    if idx >= doc_limit:
        break
    print(f"Idx {idx} / {len(useful_files)}")
    doc_loaded = reader.load_data(file=f, split_documents=True)   
    loaded_doc = Document(
        text="\n\n".join(d.get_content() for d in doc_loaded),
        metadata={"path": str(f)},
    )

    docs.append(loaded_doc)
    print(loaded_doc.metadata["path"])
```

#### 部署

https://docs.llamaindex.ai/en/stable/module_guides/llama_deploy/

第一个点： 首先开启一个 api-server ，然后部署一个服务，然后就可以连接了，是总共有三步，在更新server的时候，不要忘了，一共有三步

第二个点：

![image_4](https://raw.githubusercontent.com/nashpan/image-hosting/main/image_4.png)

这里需要添加教程中的echo_workfow, 而不是自己的文件名称，一开始按照教程，我以为填写的是自己写的类名，后来在run的时候一直报错！
