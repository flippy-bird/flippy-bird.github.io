---
title: LightRAG
date: 2025-11-28 15:06:12
description: 学习了解LightRAG
cover: /covers/14.webp
tags:
    - Rag
categories:
    - RAG框架学习
---

### 1. 背景

待详细看完完整论文后补充，大概就是GraphRag虽然效果比较好，但是速度慢，建立图的过程中消耗的token也很多，因此有了LightRag；

### 2. LightRAG的原理

> [深度解析比微软的GraphRAG简洁很多的LightRAG，一看就懂](https://zhuanlan.zhihu.com/p/4821793882)

先看了知乎上的一些解读，感觉就是简化版本的GraphRAG，在索引阶段的处理方式是差不多的，只是LightRAG将GraphRAG中比较慢的部分去掉了(社区报告部分，因此也去掉了global search)，数据处理流程是：先分块，然后提取实体和关系，然后入库，index部分的工作就完成了；

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20251128151441858.png" alt="image-20251128151441858" style="zoom: 67%;" />

在查询阶段，提供了4中方式：

- 最基本的向量相似度匹配
- local search: 根据用户的query生成一些low-level 的关键词，然后根据生成的关键词去建立好的图谱查询；
- global search: 根据用户的query生成一些high-level 的关键词，然后根据生成的关键词去建立好的图谱查询；
- 混合搜索：local search + global search 

![image-20251128151926968](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20251128151926968.png)

### 3. 源码解析

- [ ] TODO

看了源码之后，讲道理，lightRAG里面对于一些边界case处理真的不错，感觉是个工程级的项目，值得推敲学习一下，当然也导致了代码有点长了=_=#

#### 分段(chunks)

关于怎样切分，提取实体关系等等，都是放在了`lightrag/operate.py`文件里面

其实LightRag里面的切分逻辑还是比较简单，实现了按照特定字符切分和按照固定的token_size切分方式，具体的源码在`chunking_by_token_size()`函数里面,核心逻辑如下：

```python
def chunking_by_token_size(tokenizer, content, split_by_character, 
                          chunk_overlap_token_size=100, chunk_token_size=1200):
    tokens = tokenizer.encode(content)
    results = []
    
    # 方式1：按特定字符分割（如换行符）
    if split_by_character:
        raw_chunks = content.split(split_by_character)
        for chunk in raw_chunks:
            tokens = tokenizer.encode(chunk)
            # 如果超过限制，再按 token 大小切分
            if len(tokens) > chunk_token_size:
                for start in range(0, len(tokens), 
                                  chunk_token_size - chunk_overlap_token_size):
                    chunk_content = tokenizer.decode(
                        tokens[start:start + chunk_token_size]
                    )
                    results.append({"tokens": ..., "content": ..., "chunk_order_index": ...})
    else:
        # 方式2：直接按 token 大小切分
        for index, start in enumerate(
            range(0, len(tokens), chunk_token_size - chunk_overlap_token_size)
        ):
            chunk_content = tokenizer.decode(tokens[start:start + chunk_token_size])
            results.append(...)
    
    return results

```



#### 实体/关系提取

实体/关系的抽取是使用LLM来进行的，按照上面的流程图，在获取chunks之后，使用LLM获取chunk中的实体/关系，项目使用prompt部分都在`lightrag/prompt.py`文件里面，功能函数在`extract_entities()`里面实现，核心流程如下

```python
async def extract_entities(chunks, global_config, ...):
    # 1. 构建提示词
    entity_extraction_system_prompt = PROMPTS["entity_extraction_system_prompt"].format(...)
    entity_extraction_user_prompt = PROMPTS["entity_extraction_user_prompt"].format(...)
    
    # 2. 并发处理 chunks（受 llm_model_max_async 限制）
    semaphore = asyncio.Semaphore(chunk_max_async)
    
    async def _process_single_content(chunk):
        # 2.1 调用 LLM 提取
        final_result, timestamp = await use_llm_func_with_cache(
            entity_extraction_user_prompt, use_llm_func, ...
        )
        
        # 2.2 解析结果
        maybe_nodes, maybe_edges = await _process_extraction_result(final_result, ...)
        
        # 2.3 【Gleaning】如果开启，再提取一次以捕获遗漏
        if entity_extract_max_gleaning > 0:_summarize_descriptions
            glean_result = await use_llm_func_with_cache(
                entity_continue_extraction_user_prompt, ..., 
                history_messages=[user_prompt, final_result]  # 带历史
            )
            glean_nodes, glean_edges = await _process_extraction_result(glean_result, ...)
            # 合并结果（选描述更长的）
            maybe_nodes = merge_nodes(maybe_nodes, glean_nodes)
        
        return maybe_nodes, maybe_edges
    
    # 3. 并发执行所有 chunks
    tasks = [_process_with_semaphore(c) for c in chunks]
    chunk_results = await asyncio.gather(*tasks)
    
    return chunk_results  # [(nodes1, edges1), (nodes2, edges2), ...]
```

可以看到，代码做了防御性变成，`_process_extraction_result`对llm可能得各种情形都进行了处理，**同时考虑到chunks之前的处理互补干扰，因此这里还使用了`asyncio.Semaphore`来实现并发执行**

得到所有的实体和关系后，合并入库`merge_nodes_and_edges`, 这里函数里面有比较多的逻辑，可以在源码部分详细查看，这里只分析主要代码；

```python
async def merge_nodes_and_edges(chunk_results, knowledge_graph_inst, 
                               entity_vdb, relationships_vdb, ...):
    # 收集所有实体和关系
    all_nodes = defaultdict(list)
    all_edges = defaultdict(list)
    
    for maybe_nodes, maybe_edges in chunk_results:
        for entity_name, entities in maybe_nodes.items():
            all_nodes[entity_name].extend(entities)
        for edge_key, edges in maybe_edges.items():
            sorted_edge_key = tuple(sorted(edge_key))  # 统一排序，无向图
            all_edges[sorted_edge_key].extend(edges)
    
    # Phase 1: 并发处理所有实体
    entity_tasks = []
    for entity_name, entities in all_nodes.items():
        task = asyncio.create_task(_locked_process_entity_name(entity_name, entities))
        entity_tasks.append(task)
    
    processed_entities = await asyncio.gather(*entity_tasks, return_exceptions=True)
    
    # Phase 2: 并发处理所有关系
    edge_tasks = []
    for edge_key, edges in all_edges.items():
        task = asyncio.create_task(_locked_process_edges(edge_key, edges))
        edge_tasks.append(task)
    
    processed_edges = await asyncio.gather(*edge_tasks, return_exceptions=True)
    
    # Phase 3: 更新文档级索引
    await full_entities_storage.upsert({
        doc_id: {"entity_names": [...], "count": ...}
    })
    await full_relations_storage.upsert({
        doc_id: {"relation_pairs": [...], "count": ...}
    })
```



#### 查询

查询的主要功能实现也在 `lightrag/operate.py`文件里面，主函数是`kg_query`,其主要流程如下：

```python
async def kg_query(query, knowledge_graph_inst, entities_vdb, 
                   relationships_vdb, text_chunks_db, query_param, ...):
    # 1. 提取关键词
    hl_keywords, ll_keywords = await get_keywords_from_query(
        query, query_param, global_config
    )
    
    # 2. 构建查询上下文（4 阶段流水线）
    context_result = await _build_query_context(
        query, ll_keywords, hl_keywords, 
        knowledge_graph_inst, entities_vdb, relationships_vdb, 
        text_chunks_db, query_param
    )
    
    if query_param.only_need_context:
        return QueryResult(content=context_result.context, raw_data=context_result.raw_data)
    
    # 3. 调用 LLM
    sys_prompt = PROMPTS["rag_response"].format(
        response_type=query_param.response_type,
        context_data=context_result.context
    )
    
    response = await use_model_func(
        query, system_prompt=sys_prompt, 
        stream=query_param.stream
    )
    
    return QueryResult(content=response, raw_data=context_result.raw_data)
```

提取关键词那一步就是上面第二张图里面的low-level keywords和high-level keywords,这一部分的prompt如下：从prompt里面可以看到区别如下：

- **high_level_keywords**：用于表示总体概念或主题，捕捉用户的核心意图、所属领域或问题类型。
- **low_level_keywords**：用于表示具体实体或细节，识别特定实体、专有名词、技术术语、产品名称或具体项目。

```python
PROMPTS["keywords_extraction"] = """---Role---
You are an expert keyword extractor, specializing in analyzing user queries for a Retrieval-Augmented Generation (RAG) system. Your purpose is to identify both high-level and low-level keywords in the user's query that will be used for effective document retrieval.

---Goal---
Given a user query, your task is to extract two distinct types of keywords:
1. **high_level_keywords**: for overarching concepts or themes, capturing user's core intent, the subject area, or the type of question being asked.
2. **low_level_keywords**: for specific entities or details, identifying the specific entities, proper nouns, technical jargon, product names, or concrete items.

---Instructions & Constraints---
1. **Output Format**: Your output MUST be a valid JSON object and nothing else. Do not include any explanatory text, markdown code fences (like ```json), or any other text before or after the JSON. It will be parsed directly by a JSON parser.
2. **Source of Truth**: All keywords must be explicitly derived from the user query, with both high-level and low-level keyword categories are required to contain content.
3. **Concise & Meaningful**: Keywords should be concise words or meaningful phrases. Prioritize multi-word phrases when they represent a single concept. For example, from "latest financial report of Apple Inc.", you should extract "latest financial report" and "Apple Inc." rather than "latest", "financial", "report", and "Apple".
4. **Handle Edge Cases**: For queries that are too simple, vague, or nonsensical (e.g., "hello", "ok", "asdfghjkl"), you must return a JSON object with empty lists for both keyword types.
5. **Language**: All extracted keywords MUST be in {language}. Proper nouns (e.g., personal names, place names, organization names) should be kept in their original language.

---Examples---
{examples}

---Real Data---
User Query: {query}

---Output---
Output:"""
```

而`kg_query`中的第二步，则是查询中的核心，主要通过下面四个函数来实现 `_perform_kg_search`, `_apply_token_truncation`, `_merge_all_chunks`, `_build_context_str`

```python
async def _build_query_context(query, ll_keywords, hl_keywords, ...):
    # Stage 1: 纯检索
    search_result = await _perform_kg_search(
        query, ll_keywords, hl_keywords,
        knowledge_graph_inst, entities_vdb, relationships_vdb, 
        text_chunks_db, query_param
    )
    
    # Stage 2: Token 截断
    truncation_result = await _apply_token_truncation(
        search_result, query_param, global_config
    )
    
    # Stage 3: 合并 Chunks
    merged_chunks = await _merge_all_chunks(
        truncation_result["filtered_entities"],
        truncation_result["filtered_relations"],
        search_result["vector_chunks"],
        query, knowledge_graph_inst, text_chunks_db, query_param
    )
    
    # Stage 4: 构建最终上下文
    context, raw_data = await _build_context_str(
        truncation_result["entities_context"],
        truncation_result["relations_context"],
        merged_chunks, query, query_param, global_config
    )
    
    return QueryContextResult(context=context, raw_data=raw_data)
```

#### 总结

其实在实体和关系提取之后，对实体/关系进行合并的时候，会有一个总结(summary)的操作，这一步主要是将信息进行整合，以减少token，举个例子如下:  (主要的函数是`merge_nodes_and_edges()`)

```python
# 同一个实体从不同 chunks 提取到的描述列表
description_list = [
    "OpenAI是一家人工智能研究公司",
    "OpenAI开发了GPT系列模型", 
    "OpenAI成立于2015年",
    "OpenAI的总部在旧金山",
    "OpenAI由Sam Altman领导",
    # ... 可能有几十个描述
]


第一次 Map：将10个短描述 → 3个中描述
第二次 Map：将3个中描述 → 1个长描述
最终结果："OpenAI是2015年成立于旧金山的人工智能研究公司，由Sam Altman领导，开发了GPT系列模型..."
```

合并的策略采取的是 map-reduce, 分而治之的方法, 具体的函数可以看`_summarize_descriptions`

__为什么用 Map-Reduce？__

- 单次 LLM 调用有 token 限制（`summary_context_size`）
- 大量描述无法一次处理完
- 分而治之：先局部摘要，再全局摘要

```python
# 迭代直到满足条件
while True:
    total_tokens = sum(len(tokenizer.encode(desc)) for desc in current_list)
    
    # 可以一次处理完？
    if total_tokens <= summary_context_size:
        return await _summarize_descriptions(current_list)
    
    # 需要分批（Map 阶段）
    chunks = split_into_chunks(current_list, summary_context_size)
    
    # 每批生成摘要（Reduce 阶段）
    new_summaries = []
    for chunk in chunks:
        if len(chunk) == 1:
            new_summaries.append(chunk[0])  # 优化：单条不调用 LLM
        else:
            summary = await _summarize_descriptions(chunk)
            new_summaries.append(summary)
    
    current_list = new_summaries  # 递归处理

```

