---
title: agentscope-runtime学习
date: 2026-03-23 11:22:57
description: runtime也是agent框架里面比较重要的一环，通过这个项目学习一下runtime的职责和实现原理
cover: /covers/5.webp
tags:
    - agent
categories:
    - Agent开源项目学习
---

### Agentscope-runtime

> https://github.com/agentscope-ai/agentscope-runtime

agentscope runtime是什么，可以类似于我们在写一个python应用时，使用FastAPI进行网页化处理的工具，它会管理用户的请求，并发访问等部署时需要考虑的一些问题，在agentscope runtime的实现里，其核心实现继承FastAPI，因此我们可以将它类比成FastAPI来使用

核心的流程是在`src/agentscope_runtime/engine/app/agent_app.py`文件里，然后真正执行的调度是在`src/agentscope_runtime/engine/runner.py` 这个文件里

```python

    def __init__(
        self,
        *,
        **kwargs: Any,
    ):
        self._user_lifespan = lifespan

        fastapi_kwargs = {
            "title": app_name,
            "description": app_description,
            "version": __version__,
            "lifespan": self._lifespan_manager,
            **kwargs,
        }

        FastAPI.__init__(self, **fastapi_kwargs)

        self.init_routing_manager(broker_url, backend_url)
        
        # 这里后面会被初始化成用户的agent流程，然后作为一个属性放到 self._runner 里
        self._query_handler: Optional[Callable] = None 
        self._init_handler: Optional[Callable] = None
        self._shutdown_handler: Optional[Callable] = None
        self._framework_type: Optional[str] = None

        if runner:
            self._runner = runner
            self._add_endpoint_router()
        else:
            self._runner = Runner()           # 运行的核心

        self.deployment_mode = mode

        # 在这里通过redis 的广播方式实现了业务层的task取消
        self._setup_interrupt_service(
            interrupt_backend,
            interrupt_redis_url,
        )

        self._setup_builtin_routes()
        
        if custom_endpoints:
            self.restore_custom_endpoints(custom_endpoints)

        self._add_middleware()
```

而agent的执行逻辑是通过类似注册的方式，注册到系统中,可以看到，是通过`agent_app.query`这个`wrapper`来实现的

下面是官方给的一个demo

```python
@agent_app.query(framework="agentscope")
async def query_func(
    self,
    msgs,
    request: AgentRequest = None,
    **kwargs,
):
    assert kwargs is not None, "kwargs is Required for query_func"
    session_id = request.session_id
    user_id = request.user_id

    toolkit = Toolkit()
    toolkit.register_tool_function(execute_python_code)

    agent = ReActAgent(
        name="Friday",
        model=DashScopeChatModel(
            "qwen-turbo",
            api_key=os.getenv("DASHSCOPE_API_KEY"),
            enable_thinking=True,
            stream=True,
        ),
        sys_prompt="You're a helpful assistant named Friday.",
        toolkit=toolkit,
        memory=InMemoryMemory(),
        formatter=DashScopeChatFormatter(),
    )

    await self.session.load_session_state(
        session_id=session_id,
        user_id=user_id,
        agent=agent,
    )

    async for msg, last in stream_printing_messages(
        agents=[agent],
        coroutine_task=agent(msgs),
    ):
        yield msg, last

    await self.session.save_session_state(
        session_id=session_id,
        user_id=user_id,
        agent=agent,
    )
```

而`agent_app.query`这个`wrapper`在`agent_app.py`里对应的代码如下：

```python
    def query(self, framework: Optional[str] = "agentscope"):
        """
        Register run hook and optionally specify agent framework.
        Allowed framework values: 'agentscope', 'autogen', 'agno', 'langgraph'.
        """
        allowed_frameworks = {"agentscope", "autogen", "agno", "langgraph"}
        if framework not in allowed_frameworks:
            raise ValueError(f"framework must be one of {allowed_frameworks}")

        def decorator(func: Callable):
            self._query_handler = func    # 这里将agent的逻辑转交给了self._query_handler
            self._framework_type = framework

            self._build_runner()         # 在这里又将self._query_handler转交到了runner
            self._add_endpoint_router()

            return func

        return decorator
```

如上，我们这里看一下这个`self._build_runner()`代码

```python
    def _build_runner(self):
        """Bind decorated handlers to the internal Runner instance."""
        if self._runner is None:
            self._runner = Runner()

        if self._framework_type:
            self._runner.framework_type = self._framework_type

        handlers = [
            ("query_handler", self._query_handler),                
            ("init_handler", self._init_handler),
            ("shutdown_handler", self._shutdown_handler),
        ]
        for attr, handler in handlers:
            if handler:
                setattr(
                    self._runner,
                    attr,
                    types.MethodType(handler, self._runner),        # 可以看到，self._query_handler 
                )                                                   # 变成了self._runner的一个属性
```



#### 任务取消的实现

在上面的代码中，在初始化的时候，有一行代码是`self._setup_interrupt_service`来启动中断服务，其核心思想是利用redis的广播机制来实现的，当用户点击停止按钮时，会向对应的 channel发送一个STOP的消息，任务查询到这个消息后，触发`task.cancel()`，然后raise取消的错误，让任务停止，具体实现如下：

```python
┌─────────────────────────────────────────────────────────────────┐
│                     进程 A (用户请求中断)                         │
│                                                                 │
│  stop_chat("user1", "session1")                                 │
│         │                                                       │
│         ▼                                                       │
│  publish_event("chan:user1:session1", "STOP")                   │
│         │                                                       │
└─────────┼───────────────────────────────────────────────────────┘
          │
          │ Redis Pub/Sub 广播
          │ 频道："chan:user1:session1"
          │ 消息："STOP"
          │
          ▼
┌─────────┼───────────────────────────────────────────────────────┐
│         │                进程 B (正在运行的任务)                  │
│         │                                                       │
│  ┌──────▼────────────────────────────────────────────┐          │
│  │ _interrupt_signal_listener() 订阅频道              │          │
│  │         │                                         │          │
│  │         ▼                                         │          │
│  │  收到 "STOP" 消息                                  │          │
│  │         │                                         │          │
│  │         ▼                                         │          │
│  │  task_to_cancel.cancel()  ⭐ 取消任务              │          │
│  └─────────┼─────────────────────────────────────────┘          │
│            │                                                    │
│            ▼                                                    │
│  worker_task 抛出 CancelledError                                │
│            │                                                    │
│            ▼                                                    │
│  设置 is_interrupted = True                                     │
│  状态更新为 STOPPED                                             │
└─────────────────────────────────────────────────────────────────┘

```

#### sandbox

提供sandbox服务，好吧，其实这个可以单独拿出来，毕竟sandbox是在定义agent逻辑，定义agent工具时，是在工具函数里面启用了sandbox，所以这部分就不用专门在这个仓库里面看了，去看其它的agent的sanbox项目即可，毕竟项目叫runtime，这个集成一个sanbox作为一个功能的补充吧
