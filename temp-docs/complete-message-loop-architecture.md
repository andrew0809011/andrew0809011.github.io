# 完整消息循环架构 - 从请求到响应

## 高层概览

```
┌─────────────────────────────────────────────────────────────────┐
│                      前端 (React/TypeScript)                     │
│                                                                 │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ 1. useBackendSession Hook                              │    │
│  │    ├─ 读取 stdout 流（JSON-lines）                      │    │
│  │    └─ handleEvent(event) 处理每个事件                   │    │
│  └────────────────────────────────────────────────────────┘    │
│         ↑ stdout                    ↓ stdin                     │
│         │ 事件流                     │ 请求                      │
│         │ JSON-lines                 │ JSON-lines               │
└─────────┼──────────────────────────────────────────────────────┘
         │                            │
    [通过 Electron IPC / 进程通信]    │
         │                            │
         ↓                            ↓
┌─────────────────────────────────────────────────────────────────┐
│              后端 (Python/OpenHarness)                           │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ 2. ReactBackendHost.run()                                │  │
│  │    ├─ 初始化运行时 (build_runtime)                       │  │
│  │    ├─ 启动运行时 (start_runtime)                         │  │
│  │    ├─ 发送 ready 事件                                    │  │
│  │    └─ 主消息循环 (while self._running)                   │  │
│  │       ├─ 从 request_queue 获取请求                      │  │
│  │       └─ 根据 request.type 分发处理                      │  │
│  └──────────────────────────────────────────────────────────┘  │
│         ↑ request_queue                                        │
│         │                                                      │
│  ┌──────┴──────────────────────────────────────────────────┐  │
│  │ 3. _read_requests()                                      │  │
│  │    ├─ 独立异步任务                                      │  │
│  │    ├─ 在线程中读取 stdin                                │  │
│  │    ├─ 解析 JSON 请求                                   │  │
│  │    └─ 放入 request_queue                                │  │
│  └──────────────────────────────────────────────────────────┘  │
│         ↑                                                      │
│         │ 前端请求 (JSON)                                     │
│         │                                                      │
└─────────┴──────────────────────────────────────────────────────┘
```

---

## 详细流程 - 一个完整的 submit_line 周期

### 第一步：前端发送请求

**文件**: [frontend/terminal/src/App.tsx#L367](../frontend/terminal/src/App.tsx#L367)

```typescript
// 用户按 Enter
const handleSubmit = (value: string) => {
    // 检查是否是特殊命令
    if (handleCommand(value)) {
        setInput('');
        return;
    }
    
    // 普通消息 → 发送 submit_line 请求
    session.sendRequest({
        type: 'submit_line',
        line: value  // e.g., "告诉我 /Users/test.txt 的内容"
    });
    
    // 设置忙碌状态
    session.setBusy(true);
    setInput('');  // 清空输入框
};
```

**请求数据** (JSON):
```json
{"type":"submit_line","line":"告诉我 /Users/test.txt 的内容"}
```

**传输**: 通过 stdin (或 IPC) 发送到后端

---

### 第二步：后端读取请求

**文件**: [backend_host.py#L152-L175](../src/openharness/ui/backend_host.py#L152-L175)

```python
async def _read_requests(self) -> None:
    """独立异步任务，不阻塞主循环"""
    while True:
        # 在线程中阻塞读取 stdin
        raw = await asyncio.to_thread(sys.stdin.buffer.readline)
        
        if not raw:  # EOF 或连接关闭
            await self._request_queue.put(FrontendRequest(type="shutdown"))
            return
        
        # 解析 JSON 请求
        payload = raw.decode("utf-8").strip()
        request = FrontendRequest.model_validate_json(payload)
        # ↓ FrontendRequest(type='submit_line', line='...', ...)
        
        # 特殊响应类型单独处理
        if request.type == "permission_response":
            future = self._permission_requests[request.request_id]
            future.set_result(bool(request.allowed))
            continue
        
        # 其他请求放入队列
        await self._request_queue.put(request)
```

**关键点**:
- 使用 `asyncio.to_thread()` 处理阻塞的 stdin 读取
- 不在主循环中，独立运行
- 使用 Future 处理权限/问题响应

---

### 第三步：主消息循环处理

**文件**: [backend_host.py#L102-L157](../src/openharness/ui/backend_host.py#L102-L157)

```python
async def run(self) -> int:
    # 初始化
    self._bundle = await build_runtime(...)
    await start_runtime(self._bundle)
    await self._emit(BackendEvent.ready(...))
    
    # 启动请求读取任务
    reader = asyncio.create_task(self._read_requests())
    
    try:
        # 主消息循环
        while self._running:
            # ┌─────────────────────────────────────┐
            # │ 阻塞等待请求                         │
            # └─────────────────────────────────────┘
            request = await self._request_queue.get()
            
            # ┌─────────────────────────────────────┐
            # │ 请求分发                             │
            # └─────────────────────────────────────┘
            if request.type == "shutdown":
                break
            elif request.type == "submit_line":  # ← 我们的情况
                if self._busy:
                    await self._emit(BackendEvent(type="error", message="Session is busy"))
                    continue
                
                self._busy = True  # ← 设置忙碌标志
                try:
                    should_continue = await self._process_line(request.line or "")
                finally:
                    self._busy = False
                
                if not should_continue:
                    break
```

---

### 第四步：处理行 (_process_line)

**文件**: [backend_host.py#L194-L290](../src/openharness/ui/backend_host.py#L194-L290)

```python
async def _process_line(self, line: str) -> bool:
    # ┌─────────────────────────────────────┐
    # │ 1. 发送用户输入转录项                 │
    # └─────────────────────────────────────┘
    await self._emit(
        BackendEvent(type="transcript_item", 
                    item=TranscriptItem(role="user", text=line))
    )
    # ↓ 前端接收，显示用户消息
    
    # ┌─────────────────────────────────────┐
    # │ 2. 调用 handle_line 处理             │
    # └─────────────────────────────────────┘
    should_continue = await handle_line(
        self._bundle,
        line,
        print_system=_print_system,       # 系统消息回调
        render_event=_render_event,        # 事件流回调
        clear_output=_clear_output,        # 清空回调
    )
    
    # ┌─────────────────────────────────────┐
    # │ 3. 处理过程中：流式事件              │
    # └─────────────────────────────────────┘
    # → AssistantTextDelta → assistant_delta 事件
    # → ToolExecutionStarted → tool_started 事件
    # → ToolExecutionCompleted → tool_completed 事件
    # → 等等...
    
    # ┌─────────────────────────────────────┐
    # │ 4. 发送完成事件                     │
    # └─────────────────────────────────────┘
    await self._emit(self._status_snapshot())
    await self._emit(BackendEvent.tasks_snapshot(get_task_manager().list_tasks()))
    await self._emit(BackendEvent(type="line_complete"))  # ← 关键！
    
    return should_continue
```

---

### 第五步：handle_line - 命令分发

**文件**: [runtime.py#L487-L550](../src/openharness/ui/runtime.py#L487-L550)

```python
async def handle_line(bundle, line, ...) -> bool:
    # ┌─────────────────────────────────────┐
    # │ 步骤 A: 查找命令                    │
    # └─────────────────────────────────────┘
    parsed = bundle.commands.lookup(line)
    
    if parsed is not None:
        # 这是一个命令 (如 /plan, /resume 等)
        command, args = parsed
        result = await command.handler(args, CommandContext(...))
        
        # 处理命令结果
        await _render_command_result(result, ...)
        
        # 如果命令返回了 submit_prompt，继续提交给 AI
        if result.submit_prompt is not None:
            async for event in bundle.engine.submit_message(result.submit_prompt):
                await render_event(event)
        
        return True
    
    # ┌─────────────────────────────────────┐
    # │ 步骤 B: 普通消息，提交给 AI 引擎     │
    # └─────────────────────────────────────┘
    try:
        async for event in bundle.engine.submit_message(line):
            # ↓ 每个事件都立即通过 render_event 回调处理
            await render_event(event)
    except MaxTurnsExceeded as exc:
        await print_system(f"Stopped after {exc.max_turns} turns")
    
    return True
```

---

### 第六步：引擎处理 - 流式事件

**流程**:
```
engine.submit_message(line)
    ↓
提交给 LLM / Agent
    ↓
流式接收事件
    ├─ AssistantTextDelta ("你可以...")
    ├─ ToolExecutionStarted (tool_name: "read_file")
    ├─ ToolExecutionCompleted (output: "...")
    ├─ AssistantTextDelta ("文件内容是...")
    ├─ AssistantTextDelta ("...")
    └─ AssistantTurnComplete
```

**处理**:
```python
async def _render_event(event: StreamEvent) -> None:
    if isinstance(event, AssistantTextDelta):
        await self._emit(BackendEvent(type="assistant_delta", message=event.text))
        # ↓ 前端立即开始流式显示文字
    
    if isinstance(event, ToolExecutionStarted):
        await self._emit(BackendEvent(type="tool_started", 
                                     tool_name=event.tool_name,
                                     tool_input=event.tool_input))
        # ↓ 前端显示 "正在运行 read_file..."
    
    if isinstance(event, ToolExecutionCompleted):
        await self._emit(BackendEvent(type="tool_completed",
                                     tool_name=event.tool_name,
                                     output=event.output,
                                     is_error=event.is_error))
        # ↓ 前端显示工具结果
    
    if isinstance(event, AssistantTurnComplete):
        await self._emit(BackendEvent(type="assistant_complete",
                                     message=event.message.text.strip()))
        # ↓ 前端完成此轮 AI 输出
```

---

### 第七步：后端发送事件

**文件**: [backend_host.py#L167-L185](../src/openharness/ui/backend_host.py#L167-L185)

```python
async def _emit(self, event: BackendEvent) -> None:
    """发送事件到前端"""
    async with self._write_lock:  # ← 保护并发写入
        # 序列化为 JSON
        json_str = event.model_dump_json(exclude_none=True)
        
        # 写入 stdout
        sys.stdout.buffer.write(json_str.encode("utf-8"))
        sys.stdout.buffer.write(b"\n")  # 换行分隔
        sys.stdout.buffer.flush()       # 立即发送

# 示例输出 (JSON-lines)
# {"type":"transcript_item","item":{"role":"user","text":"告诉我..."}}
# {"type":"assistant_delta","message":"你可以"}
# {"type":"assistant_delta","message":"..."}
# {"type":"tool_started","tool_name":"read_file",...}
# {"type":"tool_completed","tool_name":"read_file",...}
# {"type":"assistant_delta","message":"文件内容是"}
# {"type":"assistant_complete","message":"文件内容是..."}
# {"type":"line_complete"}
```

---

### 第八步：前端接收事件

**文件**: [useBackendSession.ts#L145-L175](../frontend/terminal/src/hooks/useBackendSession.ts#L145-L175)

```typescript
// 在 useEffect 中设置事件监听
useEffect(() => {
    const readLoop = async () => {
        for await (const line of readLines(process.stdout)) {
            // 每行是一个 JSON 事件
            const event = JSON.parse(line) as BackendEvent;
            
            // ↓ 分发处理
            handleEvent(event);
        }
    };
    
    const reader = asyncio.create_task(readLoop());
}, []);
```

---

### 第九步：前端事件处理

**文件**: [useBackendSession.ts#L179-L437](../frontend/terminal/src/hooks/useBackendSession.ts#L179-L437)

```typescript
const handleEvent = (event: BackendEvent): void => {
    if (event.type === 'transcript_item') {
        queueTranscriptItem(event.item);  // ← 显示用户消息
    }
    
    if (event.type === 'assistant_delta') {
        // 流式显示文本（防抖，100ms）
        pendingAssistantDeltaRef.current += event.message;
        // 累积到 50 字符或超时后刷新
    }
    
    if (event.type === 'assistant_complete') {
        // 完成此轮 AI 输出
        setTranscript(items => [...items, {role: 'assistant', text: ...}]);
    }
    
    if (event.type === 'line_complete') {
        // ← 最终结束信号
        setBusy(false);  // 清除加载旋转
        setBusyLabel(undefined);
        // 允许用户输入新消息
    }
};
```

---

### 第十步：UI 更新

**变化**:
```
之前 (空白):
> 

发送 submit_line 请求...

等待中 (加载旋转):
> 告诉我 /Users/test.txt 的内容
[Loading spinner] Processing...

接收 transcript_item (user):
> 告诉我 /Users/test.txt 的内容

接收 assistant_delta (流式):
> 告诉我 /Users/test.txt 的内容

AI: 我来帮你读
我来帮你读取那个...

接收 tool_started:
> 告诉我 /Users/test.txt 的内容

AI: 我来帮你读取那个文件。

[Running read_file...]

接收 tool_completed:
> 告诉我 /Users/test.txt 的内容

AI: 我来帮你读取那个文件。

read_file:
/Users/test.txt: Hello World

接收 assistant_delta (继续):
> 告诉我 /Users/test.txt 的内容

AI: 我来帮你读取那个文件。

read_file:
/Users/test.txt: Hello World

完成了。文件内容是 "Hello World"。

接收 line_complete:
> 告诉我 /Users/test.txt 的内容

AI: 我来帮你读取那个文件。

read_file:
/Users/test.txt: Hello World

完成了。文件内容是 "Hello World"。

> _  (输入框可用)
```

---

## 关键架构特性

| 特性 | 实现 | 目的 |
|------|------|------|
| **异步处理** | asyncio + 独立任务 | 不阻塞主循环 |
| **流式输出** | 立即 emit 每个事件 | 最小延迟，实时反馈 |
| **并发保护** | `self._busy` 标志 + `write_lock` | 防止竞态条件 |
| **忙碌状态** | 从 `busy` 到 `line_complete` | 用户不会在处理中重复提交 |
| **防抖优化** | 100ms 防抖 + 50 字符阈值 | 平衡响应性和渲染性能 |
| **JSON-lines** | 每行一个 JSON 事件 | 易于流式处理 |
| **错误隔离** | try-finally 保证清理 | 即使出错也能恢复 |

---

## 时间线示例

```
时间    | 后端                          | 前端
--------|-------------------------------|------------------
T0      | 接收 submit_line 请求         | 显示输入，设置忙碌
T1      | emit transcript_item (user)   | 显示用户消息
T2      | 提交给 engine                 | 显示加载旋转
T3      | emit assistant_delta "我"     | 缓冲 + 防抖
T4      | emit assistant_delta "来帮"   | 缓冲 + 防抖
T5      | 刷新 = "我来帮"               | 显示 "我来帮"
T6      | emit tool_started (read)      | 显示 "Running read_file..."
T7      | 执行工具                      | 等待...
T8      | emit tool_completed           | 显示工具结果
T9      | emit assistant_delta "完成"   | 缓冲
T10     | emit assistant_complete       | 完成此轮
T11     | emit line_complete            | ✅ 清除忙碌，允许输入
T12     | 等待下一个请求                | 用户可以输入
```

---

## 文件依赖关系

```
前端入口
├─ App.tsx
│  ├─ handleSubmit()
│  └─ sendRequest({type: 'submit_line', ...})
│
└─ useBackendSession.ts
   ├─ sendRequest() 发送到 stdout
   ├─ readLoop() 读取 stdout
   └─ handleEvent() 处理每个事件
      └─ setTranscript / setBusy / etc.

后端入口
├─ backend_host.py
│  ├─ run() 主循环
│  ├─ _read_requests() 读取请求
│  └─ _process_line(line) 处理行
│     └─ _emit(event) 发送事件
│
├─ runtime.py
│  └─ handle_line(line, ...) 命令分发
│     └─ engine.submit_message(line) AI 处理
│
├─ protocol.py
│  ├─ FrontendRequest
│  └─ BackendEvent
│
└─ types.py
   ├─ TranscriptItem
   ├─ StreamEvent
   └─ ...
```

---

## 总结流程（一句话）

**用户输入 → 前端发送 submit_line → 后端读取并处理 → 流式发送事件 → 前端防抖显示 → 用户看到实时反馈**

