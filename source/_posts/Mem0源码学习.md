---
title: Mem0源码学习
date: 2025-12-01 13:54:24
description: 开源项目Mem0源码学习记录
tags:
    - Agent
    - memory
categories:
    - Agent开源项目学习
---

> https://github.com/mem0ai/mem0

### 1. 项目的核心代码

从memory/base.py 里面的基类可以看到，记忆功能主要实现了下面的功能

```python
class MemoryBase(ABC)
	@abstractmethod
	def get(self, memory_id):  # 根据id获取对应id的memory
     
    @abstractmethod
    def get_all(self):         # 列出所有的memory
        
    @abstractmethod
    def update(self, memory_id, data):    # 更新id的memory
        
    @abstractmethod
    def delete(self, memory_id):   # 删除id的memory
        
    @abstractmethod
    def history(self, memory_id):  # Get the history of changes for a memory by ID.
```

mem0的核心实现在memory/main.py 文件中

```python
class Memory(MemoryBase):
	# 这里除了实现上面base定义的基本功能外，有一个search的核心功能实现
    def search(self, query, ...):  # 这里会根据用户的query去寻找最相关的memory
	
class AsyncMemory(MemoryBase):  # 异步的接口不用再使用另外一个基类，直接使用了MemoryBase，但是在实现时使用了异步
 	async def get(self, memory_id):
```

#### 1.1 search

看search里面的实现，就是标准的RAG流程，embedding query, 然后从向量数据库或者图数据库召回，做一个ReRank重排序

```python
# 是通过下面的函数来实现向量召回的
def _search_vector_store(self, query, filters, limit, threshold: Optional[float] = None):
    embeddings = self.embedding_model.embed(query, "search")
    # 这一步就是向量检索了
    memories = self.vector_store.search(query=query, vectors=embeddings, limit=limit, filters=filters)
	...
    
    
 def search(self, query, ...):
    ...
	with concurrent.futures.ThreadPoolExecutor() as executor:
        future_memories = executor.submit(self._search_vector_store, query, effective_filters, limit, threshold)
        ## 这里的self.graph就是图关系库
        future_graph_entities = (executor.submit(self.graph.search, query, effective_filters, limit) if self.enable_graph else None)
    
```

关于Memory类中使用到的LLM，embedding, graph,rerank模型等等都是使用**工厂模式**来初始化的

```python
class Memory(MemoryBase):
    self.embedding_model = EmbedderFactory.create(
            self.config.embedder.provider,
            self.config.embedder.config,
            self.config.vector_store.config,
        )
    
    self.vector_store = VectorStoreFactory.create(
            self.config.vector_store.provider, self.config.vector_store.config
        )
    self.llm = LlmFactory.create(self.config.llm.provider, self.config.llm.config)
    if config.reranker:
        self.reranker = RerankerFactory.create(
            config.reranker.provider, 
            config.reranker.config
        )
    if self.config.graph_store.config:
        provider = self.config.graph_store.provider
        self.graph = GraphStoreFactory.create(provider, self.config)
        self.enable_graph = True
    else:
        self.graph = None
```

工厂函数里面对应的就是各个可以支持的实现，这个实现是在 mem0/utils/factory.py 文件中，下面的是图关系数据库的支持代码

```python
class GraphStoreFactory:
    """
    Factory for creating MemoryGraph instances for different graph store providers.
    Usage: GraphStoreFactory.create(provider_name, config)
    """

    provider_to_class = {
        "memgraph": "mem0.memory.memgraph_memory.MemoryGraph",
        "neptune": "mem0.graphs.neptune.neptunegraph.MemoryGraph",
        "neptunedb": "mem0.graphs.neptune.neptunedb.MemoryGraph",
        "kuzu": "mem0.memory.kuzu_memory.MemoryGraph",
        "default": "mem0.memory.graph_memory.MemoryGraph",
    }
     
    @classmethod
    def create(cls, provider_name, config):
        class_type = cls.provider_to_class.get(provider_name, cls.provider_to_class["default"])
        try:
            GraphClass = load_class(class_type)
        except (ImportError, AttributeError) as e:
            raise ImportError(f"Could not import MemoryGraph for provider '{provider_name}': {e}")
        return GraphClass(config)
```

可以看到mem0支持4中图关系数据库memgraph、neptune、neptunedb、kuzu(为什么会放在两个不同文件夹下面呢？这个暂时还没看明白)，其它的LLM，embedding等等类似；



#### 1.2 add

这个函数是mem0的核心,主要通过两个子函数来实现

```python
def add(self,
        messages,
        *,
        user_id: Optional[str] = None,
        agent_id: Optional[str] = None,
        run_id: Optional[str] = None,
        metadata: Optional[Dict[str, Any]] = None,
        infer: bool = True,
        memory_type: Optional[str] = None,
        prompt: Optional[str] = None,):
    with concurrent.futures.ThreadPoolExecutor() as executor:
        # 处理embedding相关的
        future1 = executor.submit(self._add_to_vector_store, messages, processed_metadata, effective_filters, infer)
        # 图关系数据库  
        future2 = executor.submit(self._add_to_graph, messages, effective_filters)

        concurrent.futures.wait([future1, future2])

        vector_store_result = future1.result()
        graph_result = future2.result()
```

而核心中的核心是_add_to_vector_store 这个函数

```python
# 当 infer 为false时，是常规的记忆存储，embedding,入库 (直接入库的模式)
def _add_to_vector_store(self, messages, metadata, filters, infer):
    
    # 当infer 为true时，将会使用LLM分析对话内容，提取有意义的事实，并智能地决定如何处理这些记忆（添加、更新、删除或保持不变）
    ### 这里是提取实体关系的逻辑
    parsed_messages = parse_messages(messages)

    if self.config.custom_fact_extraction_prompt:
        system_prompt = self.config.custom_fact_extraction_prompt
        user_prompt = f"Input:\n{parsed_messages}"
    else:
        # Determine if this should use agent memory extraction based on agent_id presence
        # and role types in messages
        is_agent_memory = self._should_use_agent_memory_extraction(messages, metadata)
        system_prompt, user_prompt = get_fact_retrieval_messages(parsed_messages, is_agent_memory)

    response = self.llm.generate_response(
        messages=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_prompt},
        ],
        response_format={"type": "json_object"},
    )

    # 抽取实体后，开始进行判断下一步对记忆进行 增删改查的逻辑了，也是借助了大模型来实现
    # 这部分太长省略了
```

可以观察一下mem0中抽取用户或者Agent 特性的提示词 (太长，只展示一部分，具体文件在：mem0/configs/prompts.py 里面的USER_MEMORY_EXTRACTION_PROMPT和AGENT_MEMORY_EXTRACTION_PROMPT)

首先，LLM主要任务是：

```python
AGENT_MEMORY_EXTRACTION_PROMPT = f'''
You are a Personal Information Organizer, specialized in accurately storing facts, user memories, and preferences. 
Your primary role is to extract relevant pieces of information from conversations and organize them into distinct, manageable facts. 
This allows for easy retrieval and personalization in future interactions. Below are the types of information you need to focus on and the detailed instructions on how to handle the input data.

# [IMPORTANT]: GENERATE FACTS SOLELY BASED ON THE USER'S MESSAGES. DO NOT INCLUDE INFORMATION FROM ASSISTANT OR SYSTEM MESSAGES.
# [IMPORTANT]: YOU WILL BE PENALIZED IF YOU INCLUDE INFORMATION FROM ASSISTANT OR SYSTEM MESSAGES.

Types of Information to Remember:

1. Store Personal Preferences: Keep track of likes, dislikes, and specific preferences in various categories such as food, products, activities, and entertainment.
2. Maintain Important Personal Details: Remember significant personal information like names, relationships, and important dates.
3. Track Plans and Intentions: Note upcoming events, trips, goals, and any plans the user has shared.
4. Remember Activity and Service Preferences: Recall preferences for dining, travel, hobbies, and other services.
5. Monitor Health and Wellness Preferences: Keep a record of dietary restrictions, fitness routines, and other wellness-related information.
6. Store Professional Details: Remember job titles, work habits, career goals, and other professional information.
7. Miscellaneous Information Management: Keep track of favorite books, movies, brands, and other miscellaneous details that the user shares.
'''
```

具体的few shot例子如下：给了一些正例和反例(不用提取的例子)

```python
AGENT_MEMORY_EXTRACTION_PROMPT = f'''
User: Hi.
Assistant: Hello! I enjoy assisting you. How can I help today?
Output: {{"facts" : []}}

User: There are branches in trees.
Assistant: That's an interesting observation. I love discussing nature.
Output: {{"facts" : []}}

User: Hi, I am looking for a restaurant in San Francisco.
Assistant: Sure, I can help with that. Any particular cuisine you're interested in?
Output: {{"facts" : ["Looking for a restaurant in San Francisco"]}}

User: Yesterday, I had a meeting with John at 3pm. We discussed the new project.
Assistant: Sounds like a productive meeting. I'm always eager to hear about new projects.
Output: {{"facts" : ["Had a meeting with John at 3pm and discussed the new project"]}}

User: Hi, my name is John. I am a software engineer.
Assistant: Nice to meet you, John! My name is Alex and I admire software engineering. How can I help?
Output: {{"facts" : ["Name is John", "Is a Software engineer"]}}

'''
```

不知道这种写法会不会限制LLM输出用户不想要的，prompt里面存在类似下面的句子：

```python
# [IMPORTANT]: GENERATE FACTS SOLELY BASED ON THE USER'S MESSAGES. DO NOT INCLUDE INFORMATION FROM ASSISTANT OR SYSTEM MESSAGES.
# [IMPORTANT]: YOU WILL BE PENALIZED IF YOU INCLUDE INFORMATION FROM ASSISTANT OR SYSTEM MESSAGES.
```

获取到哪些用户的习惯或者其它一些与用户相关信息之后，然后使用embedding匹配从memory中召回一些与这些信息相关的信息；具体的代码如下：

```python
# new_retrieved_facts 即是从这条消息中获取到的用户偏好
for new_mem in new_retrieved_facts:
    messages_embeddings = self.embedding_model.embed(new_mem, "add")
    new_message_embeddings[new_mem] = messages_embeddings
    existing_memories = self.vector_store.search(
        query=new_mem,
        vectors=messages_embeddings,
        limit=5,
        filters=search_filters,
    )
    for mem in existing_memories:
        retrieved_old_memory.append({"id": mem.id, "text": mem.payload.get("data", "")})

```

回溯到相关的历史memory之后，然后使用LLM确定是否添加新的记忆，还是删除更新旧的记忆，具体的职责相关prompt如下

```python
DEFAULT_UPDATE_MEMORY_PROMPT = """You are a smart memory manager which controls the memory of a system.
You can perform four operations: (1) add into the memory, (2) update the memory, (3) delete from the memory, and (4) no change.

Based on the above four operations, the memory will change.

Compare newly retrieved facts with the existing memory. For each new fact, decide whether to:
- ADD: Add it to the memory as a new element
- UPDATE: Update an existing memory element
- DELETE: Delete an existing memory element
- NONE: Make no change (if the fact is already present or irrelevant)


"""
```

few shot 的例子如下：展示的是删除这个操作的例子，完整的可以看mem0/configs/prompts.py 文件中的DEFAULT_UPDATE_MEMORY_PROMPT

```python
DEFAULT_UPDATE_MEMORY_PROMPT = """


There are specific guidelines to select which operation to perform:

3. **Delete**: If the retrieved facts contain information that contradicts the information present in the memory, then you have to delete it. Or if the direction is to delete the memory, then you have to delete it.
Please note to return the IDs in the output from the input IDs only and do not generate any new ID.
- **Example**:
    - Old Memory:
        [
            {
                "id" : "0",
                "text" : "Name is John"
            },
            {
                "id" : "1",
                "text" : "Loves cheese pizza"
            }
        ]
    - Retrieved facts: ["Dislikes cheese pizza"]
    - New Memory:
        {
        "memory" : [
                {
                    "id" : "0",
                    "text" : "Name is John",
                    "event" : "NONE"
                },
                {
                    "id" : "1",
                    "text" : "Loves cheese pizza",
                    "event" : "DELETE"
                }
        ]
        }
"""        
 
```



#### 1.3 其它

memory/telemetry.py 文件，这个文件的作用类似埋点，用来记录用户使用mem0的数据情况（在gemini cli， qwen code中看到同样的文件）
