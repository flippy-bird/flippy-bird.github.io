---
title: LLM memory
date: 2025-12-01 13:47:10
description: LLM memory 综合文档记录
cover: /covers/15.webp
tags:
    - Agent
    - memory
---

### OpenViking

> https://github.com/volcengine/OpenViking



### MemU

从他的tests部分的文件入手，基本用法是初始化一个 `MemoryService` (路径位于：`memu/app/service.py`文件) , 然后两个核心功能`memorize`和`retrive`，也是记忆库的话，一般都是两个核心功能点，入库和出库(即查询)；因此从这两个接口出发，研究他的源码；

#### memorize

先看一个总体的图，然后对照这个图去看源码：

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260318152605250.png" alt="image-20260318152605250" style="zoom:80%;" />

具体的源码入口:

```python
async def memorize():
    ctx = self._get_context()
    store = self._get_database()
    user_scope = self.user_model(**user).model_dump() if user is not None else None
    await self._ensure_categories_ready(ctx, store, user_scope)

    memory_types = self._resolve_memory_types()

    state: WorkflowState = ...
    # 核心的部分在这个地方了
    result = await self._run_workflow("memorize", state)
    response = cast(dict[str, Any] | None, result.get("response"))

    return response
```

而上面核心执行部分继续跟踪后，流程如下：

```python
'''
_run_workflow()            : src/memu/app/service.py:350         
     |
PipelineManager.build()    : src/memu/workflow/pipeline.py:47       获取工作流执行的步骤(7个步骤)
     |
WorkflowRunner.run()       : src/memu/workflow/runner.py: 31
     |
run_steps()                : src/memu/workflow/step.py               核心执行循环
'''
```

上面的7个步骤，其实是在系统初始化的时候，就已经注册好了, 同样的，对于`retrieve`操作也是同理的

```python
class MemoryService(MemorizeMixin, RetrieveMixin, CRUDMixin):
    def __init__():
        # 这一行代码
        self._register_pipelines()
        
        
    def _register_pipelines(self) -> None:
        memo_workflow = self._build_memorize_workflow()
        memo_initial_keys = self._list_memorize_initial_keys()
        # 这里就是我们的主角了
        self._pipelines.register("memorize", memo_workflow, initial_state_keys=memo_initial_keys)
        rag_workflow = self._build_rag_retrieve_workflow()
        retrieve_initial_keys = self._list_retrieve_initial_keys()
        self._pipelines.register("retrieve_rag", rag_workflow, initial_state_keys=retrieve_initial_keys)
        llm_workflow = self._build_llm_retrieve_workflow()
        self._pipelines.register("retrieve_llm", llm_workflow, initial_state_keys=retrieve_initial_keys)
        ...
```

而在`self._build_memorize_workflow()`里面，我们可以找到具体的7个执行步骤了

```python
    def _build_memorize_workflow(self) -> list[WorkflowStep]:
        ## 备注：WorkflowStep里面的参数我只保留了前三个个参数，为了直观 
        steps = [
            WorkflowStep(
                step_id="ingest_resource",
                role="ingest",
                handler=self._memorize_ingest_resource,
            ),
            WorkflowStep(
                step_id="preprocess_multimodal",
                role="preprocess",
                handler=self._memorize_preprocess_multimodal,
            ),
            WorkflowStep(
                step_id="extract_items",
                role="extract",
                handler=self._memorize_extract_items,
            ),
            WorkflowStep(
                step_id="dedupe_merge",
                role="dedupe_merge",
                handler=self._memorize_dedupe_merge,
            ),
            WorkflowStep(
                step_id="categorize_items",
                role="categorize",
                handler=self._memorize_categorize_items,
            ),
            WorkflowStep(
                step_id="persist_index",
                role="persist",
                handler=self._memorize_persist_and_index,
            ),
            WorkflowStep(
                step_id="build_response",
                role="emit",
                handler=self._memorize_build_response,
            ),
        ]
        return steps
```

第三个参数的hander，对应的就是每一步的处理函数了，核心也就在每个步骤对应的函数了

而run_steps， 就是按照上面的步骤一步一步执行即可

```python
async def run_steps(
    name: str,
    steps: list[WorkflowStep],
    initial_state: WorkflowState,
    context: WorkflowContext = None,
    interceptor_registry: WorkflowInterceptorRegistry | None = None,
) -> WorkflowState:
    from memu.workflow.interceptor import (
        WorkflowStepContext,
        run_after_interceptors,
        run_before_interceptors,
        run_on_error_interceptors,
    )

    snapshot = interceptor_registry.snapshot() if interceptor_registry else None
    strict = interceptor_registry.strict if interceptor_registry else False

    state = dict(initial_state)
    for step in steps:
        missing = step.requires - state.keys()
        if missing:
            msg = f"Workflow '{name}' missing required keys for step '{step.step_id}': {', '.join(sorted(missing))}"
            raise KeyError(msg)
        step_context: dict[str, Any] = dict(context) if context else {}
        step_context["step_id"] = step.step_id
        if step.config:
            step_context["step_config"] = step.config

        # Build interceptor context
        interceptor_ctx = WorkflowStepContext(
            workflow_name=name,
            step_id=step.step_id,
            step_role=step.role,
            step_context=step_context,
        )

        # Run before interceptors
        if snapshot and snapshot.before:
            await run_before_interceptors(snapshot.before, interceptor_ctx, state, strict=strict)

        try:
            state = await step.run(state, step_context)   # 这里就是每一步的执行函数，前后的代码都是hooks
        except Exception as e:
            if snapshot and snapshot.on_error:
                await run_on_error_interceptors(snapshot.on_error, interceptor_ctx, state, e, strict=strict)
            raise

        # Run after interceptors
        if snapshot and snapshot.after:
            await run_after_interceptors(snapshot.after, interceptor_ctx, state, strict=strict)

    return state
```



#### retrieve

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260318152833494.png" alt="image-20260318152833494" style="zoom:80%;" />

流程和上面是一样的，召回有两种方式，rag和llm，对于rag的工作流如下(简化版本)

```python
    def _build_rag_retrieve_workflow(self) -> list[WorkflowStep]:
        steps = [
            WorkflowStep(
                step_id="route_intention",
                role="route_intention",
                handler=self._rag_route_intention,
                requires={"route_intention", "original_query", "context_queries", "skip_rewrite"},
            ),
            WorkflowStep(
                step_id="route_category",
                role="route_category",
                handler=self._rag_route_category,
            ),
            WorkflowStep(
                step_id="sufficiency_after_category",
                role="sufficiency_check",
                handler=self._rag_category_sufficiency,
            ),
            WorkflowStep(
                step_id="recall_items",
                role="recall_items",
                handler=self._rag_recall_items,
            ),
            WorkflowStep(
                step_id="sufficiency_after_items",
                role="sufficiency_check",
                handler=self._rag_item_sufficiency,
            ),
            WorkflowStep(
                step_id="recall_resources",
                role="recall_resources",
                handler=self._rag_recall_resources,
            ),
            WorkflowStep(
                step_id="build_context",
                role="build_context",
                handler=self._rag_build_context,
            ),
        ]
        return steps
```

**意图理解部分**：做了一个二分类的判断(感觉函数名路由这个词是不是不太对)，在这一步的时候让llm去判断需不需要retrieve，如果需要，那么根据context，做一个query的改写，然后进行后面的步骤；MemU写提示词似乎也喜欢按照工作流的方式，例如这个分类的系统prompt如下：

```python
SYSTEM_PROMPT = """
# Task Objective
Determine whether the current query requires retrieving information from memory or can be answered directly without retrieval.
If retrieval is required, rewrite the query to include relevant contextual information.

# Workflow
1. Review the **Query Context** to understand prior conversation and available background.
2. Analyze the **Current Query**.
3. Consider the **Retrieved Content**, if any.
4. Decide whether memory retrieval is required based on the criteria.
5. If retrieval is needed, rewrite the query to incorporate relevant context from the query context.
6. If retrieval is not needed, keep the original query unchanged.

# Rules
- **NO_RETRIEVE** for:
  - Greetings, casual chat, or acknowledgments
  - Questions about only the current conversation/context
  - General knowledge questions
  - Requests for clarification
  - Meta-questions about the system itself
- **RETRIEVE** for:
  - Questions about past events, conversations, or interactions
  - Queries about user preferences, habits, or characteristics
  - Requests to recall specific information
  - Questions referencing historical data
- Do not add external knowledge beyond the provided context.
- If retrieval is not required, return the original query exactly.

# Output Format
Use the following structure:

<decision>
RETRIEVE or NO_RETRIEVE
</decision>

<rewritten_query>
If RETRIEVE: provide a rewritten query incorporating relevant context.
If NO_RETRIEVE: return `{query}` unchanged.
</rewritten_query>
"""

USER_PROMPT = """
# Input
Query Context:
{conversation_history}

Current Query:
{query}

Retrieved Content:
{retrieved_content}
"""
```

下面的操作就是rag的基本操作了，先根据query查询相关目录，然后查询相关文件等等，最后总结一下；

而对于llm的召回方式，也类似，虽然，但是，仓库里有两个一样功能的函数,这个有点没明白,只是函数名不一样，为什么要写两遍呢 =_=#  ，下面的召回，重排序，是使用llm来去做判断，而没有使用embedding了

![image-20260318144519215](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260318144519215.png)



#### 其它

##### MemU关于视频的处理

在memorize里面，对于视频的处理，是截取视频最中间的一帧，然后做的图片分析的，因此如果后续真的用到了这个库，视频部分一定要记得这个点；

```python
   ##  src/memu/utils/video.py
    
    async def _preprocess_video(
        self, local_path: str, template: str, llm_client: Any | None = None
    ) -> list[dict[str, str | None]]:
        """
        Preprocess video data - extract description and caption using Vision API.

        Extracts the middle frame from the video and analyzes it using Vision API.

        Args:
            local_path: Path to the video file
            template: Prompt template for video analysis

        Returns:
            List with single resource containing text (description) and caption
        """
        try:
            # Check if ffmpeg is available
            if not VideoFrameExtractor.is_ffmpeg_available():
                logger.warning("ffmpeg not available, cannot process video. Returning None.")
                return [{"text": None, "caption": None}]

            # Extract middle frame from video
            logger.info(f"Extracting frame from video: {local_path}")
            frame_path = VideoFrameExtractor.extract_middle_frame(local_path)

            try:
                # Call Vision API with extracted frame
                logger.info(f"Analyzing video frame with Vision API: {frame_path}")
                client = llm_client or self._get_llm_client()
                processed = await client.vision(prompt=template, image_path=frame_path, system_prompt=None)
                description, caption = self._parse_multimodal_response(processed, "detailed_description", "caption")
                return [{"text": description, "caption": caption}]
            finally:
                # Clean up temporary frame file
                import pathlib

                try:
                    pathlib.Path(frame_path).unlink(missing_ok=True)
                    logger.debug(f"Cleaned up temporary frame: {frame_path}")
                except Exception as e:
                    logger.warning(f"Failed to clean up frame {frame_path}: {e}")

        except Exception as e:
            logger.error(f"Video preprocessing failed: {e}", exc_info=True)
            return [{"text": None, "caption": None}]
```





### 参考资料
1. [LLM memory evaluation](https://mp.weixin.qq.com/s/lhb8fI8JbhRGKIrPIc5hJw?poc_token=HF8pLWmjxuFHk8JTOg3JmyE4SBkpcixgjmie8JkP)
2. [EverMemos](https://github.com/EverMind-AI/EverMemOS/tree/main/evaluation)
3. [MemOS](https://github.com/MemTensor/MemOS)
4. [zep](https://github.com/getzep/zep)
5. [memu](https://github.com/NevaMind-AI/memU)
6. [letta](https://github.com/letta-ai/letta)
7. [北大出品，最火、最全的Agent记忆综述！！](https://github.com/Shichun-Liu/Agent-Memory-Paper-List) 相关[微信公众号地址](https://mp.weixin.qq.com/s/4q4bgDEOzTPUubnaYzG4wA)
8. [最近很多的clewdbot的记忆实现](https://zhuanlan.zhihu.com/p/1999506236597637279)