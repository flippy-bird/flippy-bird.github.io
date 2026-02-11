---
title: nanobot源码学习
date: 2026-02-06 11:29:52
description: 什么？没有时间学习openclawd，那就来学习nanobot吧！ 
cover: /covers/0.webp
tags:
    - agent
categories:
    - Agent开源项目学习
---

> https://github.com/HKUDS/nanobot
>
> 基于openclawd原理使用python 4000行代码实现的简易openclawd



### 总架构图

![image-20260210135650860](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260210135650860.png)

从图和原理上来看，是比较容易理解的，大致的想法是，建立一个Agent Loop，可以接受各种聊天应用的消息，然后利用常规agent范式去处理这些消息，然后将结果一一发送回原先的聊天应用中；感觉像是 Agent服务 + websocket串联聊天应用这么一个方式，消息的接受处理是一个生产者，消费者模型；

### 源码阅读

#### 主流程

从Readme中部署时的命令出发，最后一步是`nanobot gateway`, 我们在创建命令行指令的文件 `nanobot/cli/commands.py`文件中找到`gateway`函数

```python
@app.command()
def gateway():
    config = load_config()
    # 在这里创建了一个消息总线
    bus = MessageBus()
    provider = _make_provider(config)
    session_manager = SessionManager(config.workspace_path)
    
    # 定时的功能，可以先不用管
    cron_store_path = get_data_dir() / "cron" / "jobs.json"
    cron = CronService(cron_store_path)
    
    # Agent相关的服务
    agent = AgentLoop(
        bus=bus,
        provider=provider,
        workspace=config.workspace_path,
        model=config.agents.defaults.model,
        max_iterations=config.agents.defaults.max_tool_iterations,
        brave_api_key=config.tools.web.search.api_key or None,
        exec_config=config.tools.exec,
        cron_service=cron,
        restrict_to_workspace=config.tools.restrict_to_workspace,
        session_manager=session_manager,
    )
    
    # 心跳服务
    async def on_heartbeat(prompt: str) -> str:
        """Execute heartbeat through the agent."""
        return await agent.process_direct(prompt, session_key="heartbeat")
    
    heartbeat = HeartbeatService(
        workspace=config.workspace_path,
        on_heartbeat=on_heartbeat,
        interval_s=30 * 60,  # 30 minutes
        enabled=True
    )
    
    # 飞书类聊天服务
    channels = ChannelManager(config, bus, session_manager=session_manager)

    async def run():
        try:
            await cron.start()
            await heartbeat.start()
            await asyncio.gather(
                agent.run(),
                channels.start_all(),
            )
        except KeyboardInterrupt:
			...
    
    asyncio.run(run())
```

从这个启动服务的函数可以看到，主要是创建一个消息总线`bus`,然后创建一个agent服务，创建一些chat类的服务，两者通过这个消息总线bus来串联，首先，我们来看agent部分

#### Agent部分

这部分代码在`nanobot/agent/loop.py`这个文件里面，主要功能函数是`run`

```python
async def run(self) -> None:
    """Run the agent loop, processing messages from the bus."""
    self._running = True
    logger.info("Agent loop started")

    while self._running:
        try:
            # 这里从消息队列里面取消息，如果没有消息，则阻塞，但是这里设置了timeout，因此触发下面的超时错误后继续
            msg = await asyncio.wait_for(
                self.bus.consume_inbound(),
                timeout=1.0
            )

            # Process it
            try:
                response = await self._process_message(msg)
                if response:
                    await self.bus.publish_outbound(response)
            except Exception as e:
                logger.error(f"Error processing message: {e}")
                # Send error response
                await self.bus.publish_outbound(OutboundMessage(
                    channel=msg.channel,
                    chat_id=msg.chat_id,
                    content=f"Sorry, I encountered an error: {str(e)}"
                ))
        except asyncio.TimeoutError:
            continue

def stop(self) -> None:
    """Stop the agent loop."""
    self._running = False
    logger.info("Agent loop stopping")
```

可以看到，内部是一个无限循环，只有当调用了stop函数修改了状态时，才结束这个循环

至于上面流程中的`self._process_message()`函数(下面是简化版本)，看到` while iteration < self.max_iterations:`就知道是我们熟悉的老演员了，是基本的ReAct模式

```python
    async def _process_message(self, msg: InboundMessage) -> OutboundMessage | None:
        # Handle system messages (subagent announces)
        # The chat_id contains the original "channel:chat_id" to route back to
        if msg.channel == "system":
            return await self._process_system_message(msg)
        
        preview = msg.content[:80] + "..." if len(msg.content) > 80 else msg.content
        logger.info(f"Processing message from {msg.channel}:{msg.sender_id}: {preview}")
        
        # Get or create session
        session = self.sessions.get_or_create(msg.session_key)
        
		...
        
        # Agent loop
        iteration = 0
        final_content = None
        
        while iteration < self.max_iterations:
            iteration += 1
            
            # Call LLM
            response = await self.provider.chat(
                messages=messages,
                tools=self.tools.get_definitions(),
                model=self.model
            )
            
            # Handle tool calls
            if response.has_tool_calls:
                # Add assistant message with tool calls
                tool_call_dicts = [
                    {
                        "id": tc.id,
                        "type": "function",
                        "function": {
                            "name": tc.name,
                            "arguments": json.dumps(tc.arguments)  # Must be JSON string
                        }
                    }
                    for tc in response.tool_calls
                ]
                messages = self.context.add_assistant_message(
                    messages, response.content, tool_call_dicts,
                    reasoning_content=response.reasoning_content,
                )
                
                # Execute tools
                for tool_call in response.tool_calls:
                    args_str = json.dumps(tool_call.arguments, ensure_ascii=False)
                    logger.info(f"Tool call: {tool_call.name}({args_str[:200]})")
                    result = await self.tools.execute(tool_call.name, tool_call.arguments)
                    messages = self.context.add_tool_result(
                        messages, tool_call.id, tool_call.name, result
                    )

        return OutboundMessage(
            channel=msg.channel,
            chat_id=msg.chat_id,
            content=final_content,
            metadata=msg.metadata or {},  # Pass through for channel-specific needs (e.g. Slack thread_ts)
        )
```

处理完请求后就往装载处理消息的队列里面塞即可

#### channel部分

这部分是发送消息，生产者的部分，根据上面的主流程，需要关注`channels.start_all()` 这个函数

在 `nanobot/channels/manager.py`这个文件

```python
    async def start_all(self) -> None:
        """Start all channels and the outbound dispatcher."""
        if not self.channels:
            logger.warning("No channels enabled")
            return
        
        # 开启消息推送分发的服务
        self._dispatch_task = asyncio.create_task(self._dispatch_outbound())
        
        # 打开消息生产的服务，channel可以开始发送消息
        tasks = []
        for name, channel in self.channels.items():
            logger.info(f"Starting {name} channel...")
            tasks.append(asyncio.create_task(self._start_channel(name, channel)))
        
        # Wait for all to complete (they should run forever)
        await asyncio.gather(*tasks, return_exceptions=True)
```

先看`self._dispatch_outbound()` 这个推送服务

```python
    async def _dispatch_outbound(self) -> None:
        """Dispatch outbound messages to the appropriate channel."""
        logger.info("Outbound dispatcher started")
        
        while True:
            try:
                msg = await asyncio.wait_for(
                    self.bus.consume_outbound(),
                    timeout=1.0
                )
                
                channel = self.channels.get(msg.channel)
                if channel:
                    try:
                        await channel.send(msg)
                    except Exception as e:
                        logger.error(f"Error sending to {msg.channel}: {e}")
                else:
                    logger.warning(f"Unknown channel: {msg.channel}")
                    
            except asyncio.TimeoutError:
                continue
            except asyncio.CancelledError:
                break
```

和之前的逻辑一样，如果有消息总线里面有结果，那么获取消息，然后发送给对应的channel，否则一直触发超时，然后继续；

再看一下`self._start_channel`,里面有一个功能函数 `channel.start()`,这个对应每个聊天应用的实现，这里我们查看飞书channel的实现

```python
    async def start(self) -> None:
        """Start the Feishu bot with WebSocket long connection."""
        if not FEISHU_AVAILABLE:
            logger.error("Feishu SDK not installed. Run: pip install lark-oapi")
            return
        
        if not self.config.app_id or not self.config.app_secret:
            logger.error("Feishu app_id and app_secret not configured")
            return
        
        self._running = True
        self._loop = asyncio.get_running_loop()
        
        # Create Lark client for sending messages
        self._client = lark.Client.builder() \
            .app_id(self.config.app_id) \
            .app_secret(self.config.app_secret) \
            .log_level(lark.LogLevel.INFO) \
            .build()
        
        # Create event handler (only register message receive, ignore other events)
        event_handler = lark.EventDispatcherHandler.builder(
            self.config.encrypt_key or "",
            self.config.verification_token or "",
        ).register_p2_im_message_receive_v1(
            self._on_message_sync
        ).build()
        
        # Create WebSocket client for long connection
        self._ws_client = lark.ws.Client(
            self.config.app_id,
            self.config.app_secret,
            event_handler=event_handler,
            log_level=lark.LogLevel.INFO
        )
        
        # Start WebSocket client in a separate thread
        def run_ws():
            try:
                self._ws_client.start()
            except Exception as e:
                logger.error(f"Feishu WebSocket error: {e}")
        
        self._ws_thread = threading.Thread(target=run_ws, daemon=True)
        self._ws_thread.start()
        
        logger.info("Feishu bot started with WebSocket long connection")
        logger.info("No public IP required - using WebSocket to receive events")
        
        # Keep running until stopped  注意这个里面也加了一个循环
        while self._running:
            await asyncio.sleep(1)
```

注意：最后也特意添加了一个循环，用来保持协程的活跃， 防止start()立即返回



### 其它可参考资料

1. [Nanobot 项目深度解读报告](https://zhuanlan.zhihu.com/p/2003494508445336693?share_code=FBP8qWqPKJN6&utm_psn=2004357085224276031)
