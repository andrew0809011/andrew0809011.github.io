# ReactBackendHost 消息循环实现

## 概述

`ReactBackendHost` 实现了一个基于 JSON-lines 协议的异步消息循环，连接前端 React 应用和后端 OpenHarness 运行时。消息循环通过 stdin/stdout 通信，处理用户输入、执行查询、流式发送事件。

**核心文件**: [src/openharness/ui/backend_host.py](src/openharness/ui/backend_host.py)

---

## 消息循环架构

```
前端 (React)
    ↓ (JSON-lines on stdin)
    ↓
┌─────────────────────────────────────┐
│  ReactBackendHost.run()             │ ← 主入口
│  ├─ build_runtime()                 │ ← 初始化运行时
│  ├─ start_runtime()                 │ ← 启动引擎
│  └─ 消息循环 (while self._running)   │ ← 处理请求
└─────────────────────────────────────┘
    ↑
    ↑ (JSON-lines on stdout)
    ↑
后端事件发送
```

---

## 核心组件

### 1. **主消息循环** (`run()` 方法)
**位置**: [backend_host.py#L84-L150](src/openharness/ui/backend_host.py#L84-L150)

```python
async def run(self) -> int:
    # 初始化阶段
    self._bundle = await build_runtime(...)  # 加载配置、认证、工具等
    await start_runtime(self._bundle)         # 启动运行时
    await self._emit(BackendEvent.ready(...)) # 发送 ready 事件
    
    # 启动请求读取任务
    reader = asyncio.create_task(self._read_requests())
    
    try:
        # 消息循环
        while self._running:
            request = await self._request_queue.get()  # ← 阻塞等待请求
            
            # 根据请求类型处理
            if request.type == "shutdown":
                break
            elif request.type == "submit_line":  # ← 主要请求类型
                self._busy = True
                try:
                    should_continue = await self._process_line(line)
                finally:
                    self._busy = False
            # ... 其他请求类型
```

**关键特性**：
- **异步驱动**: 使用 `asyncio.Queue` 处理请求
- **忙碌标志**: `self._busy` 防止并发请求
- **写锁**: `self._write_lock` 保护 stdout 输出
- **运行标志**: `self._running` 控制循环生命周期

---

## reader Task 与 asyncio.Queue 数据流详解

### 为什么要分成两个协程？

`run()` 主循环和 `_read_requests()` 作为独立 Task 并发运行，核心原因是：
> `sys.stdin.buffer.readline()` 是**阻塞调用**，如果放在主循环里，处理请求时就无法及时响应 interrupt 信号。

### `asyncio.to_thread()` 的作用

```python
raw = await asyncio.to_thread(sys.stdin.buffer.readline)
```

- `readline()` 是同步阻塞的——它会一直等到用户（前端进程）写入一行
- `asyncio.to_thread()` 把这个调用丢到**线程池**，不占用 event loop 线程
- `await` 让 event loop 可以在等待期间调度其他协程（如 `_emit` 发送事件、permission Future 等待）
- 返回 `bytes`，空 bytes 表示 stdin 关闭（EOF），触发 shutdown

### 完整数据流图

```
stdin (前端进程写入 JSON-lines)
        │
        ▼
┌───────────────────────────────────────────────────────────────┐
│                   asyncio Event Loop (单线程)                  │
│                                                               │
│  ┌─────────────────────────────┐   ┌───────────────────────┐ │
│  │ Task: _read_requests()      │   │ 主协程: run() 循环     │ │
│  │                             │   │                       │ │
│  │  while True:                │   │  while self._running: │ │
│  │    ┌─────────────────────┐  │   │    request =          │ │
│  │    │ 子线程池             │  │   │    await queue.get()  │ │
│  │    │ readline() ← 阻塞   │  │   │         ▲             │ │
│  │    │ 返回 bytes           │  │   │         │             │ │
│  │    └──────────┬──────────┘  │   │         │ queue.put() │ │
│  │               │ decode+parse│   │         │             │ │
│  │               ▼             │   │         │             │ │
│  │  ┌────────────────────────┐ │   │         │             │ │
│  │  │ 请求类型分流            │ │   │         │             │ │
│  │  │                        │─┼───┼─────────┘ 普通请求    │ │
│  │  │ permission_response ───┼─┼──→ future.set_result()    │ │
│  │  │ question_response   ───┼─┼──→ future.set_result()    │ │
│  │  │ interrupt           ───┼─┼──→ task.cancel()          │ │
│  │  │ 其他               ────┼─┼──→ queue.put(request)     │ │
│  │  └────────────────────────┘ │   │                       │ │
│  └─────────────────────────────┘   └───────────────────────┘ │
└───────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
                                   process_line / handle_line
                                              │
                                              ▼
                               stdout (JSON-lines BackendEvent)
```

### 两个 Task 的职责对比

| | `_read_requests` (reader Task) | `run()` 主循环 |
|--|--|--|
| 角色 | **生产者** | **消费者** |
| 启动方式 | `asyncio.create_task(...)` | 主协程直接运行 |
| 阻塞点 | `to_thread(readline)` 在子线程 | `queue.get()` 等待队列 |
| 短路处理 | permission / question / interrupt 直接处理，**不入队** | 只处理队列里的普通请求 |
| 生命周期 | 直到 EOF，`finally` 中被 `reader.cancel()` 清理 | 直到 `self._running = False` 或 shutdown |

### interrupt 能实时响应的原因

即使主循环正在 `await handle_line(...)` 处理耗时请求，reader Task 依然运行，一旦收到 `interrupt` 类型请求，直接调用：
```python
await self._interrupt_active_request()  # → self._active_request_task.cancel()
```
这会向正在运行的 `_process_line` Task 抛出 `CancelledError`，中断执行。

---

## `asyncio.create_task` 与 `asyncio.to_thread` 的关系

> **它们不是配套的，是两个完全独立的工具，解决不同维度的问题。**

### 职责对比

| | `asyncio.create_task()` | `asyncio.to_thread()` |
|--|--|--|
| 解决问题 | **并发**：让多个协程同时运行 | **兼容性**：让同步阻塞代码不卡 event loop |
| 层面 | 协程调度 | 线程/异步边界 |
| 返回值 | `Task` 对象（可 cancel/await） | `Coroutine`（需要 await 拿结果） |
| 必须配套 | 否 | 否 |

### 可以单独使用

```python
# create_task 单独使用（不涉及 to_thread）
async def count_down():
    for i in range(3):
        await asyncio.sleep(1)  # 纯异步
        print(i)

task = asyncio.create_task(count_down())  # 并发运行

# to_thread 单独使用（不涉及 create_task）
async def read_file():
    data = await asyncio.to_thread(open("/tmp/big.txt").read)  # 阻塞 IO
    return data
```

### 为什么在 `_read_requests` 里两者同时出现

```python
# run() 方法中：
reader = asyncio.create_task(self._read_requests())  # ← create_task

# _read_requests() 内部：
raw = await asyncio.to_thread(sys.stdin.buffer.readline)  # ← to_thread
```

- **需要 `create_task`**：让 reader 和主循环**并发**运行，互不阻塞
- **需要 `to_thread`**：`readline()` 是同步阻塞调用，不用 `to_thread` 会冻结整个 event loop

去掉任一个都有严重问题：

```
去掉 create_task（直接 await _read_requests()）
  → 主循环永远不会开始，程序挂死

去掉 to_thread（直接调用 readline()）
  → event loop 被卡死，所有协程冻结（包括 _emit、permission 等待等）
```

它们恰好同时出现，是因为这个场景同时需要"并发"和"阻塞 IO 兼容"两个特性。

### 2. **请求读取任务** (`_read_requests()` 方法)
**位置**: [backend_host.py#L152-L175](src/openharness/ui/backend_host.py#L152-L175)

```python
async def _read_requests(self) -> None:
    while True:
        # 阻塞读取 stdin（在线程中执行）
        raw = await asyncio.to_thread(sys.stdin.buffer.readline)
        
        if not raw:  # EOF
            await self._request_queue.put(FrontendRequest(type="shutdown"))
            return
        
        # 解析 JSON 请求
        payload = raw.decode("utf-8").strip()
        request = FrontendRequest.model_validate_json(payload)
        
        # 处理特殊响应（权限/问题）
        if request.type == "permission_response":
            future = self._permission_requests[request.request_id]
            future.set_result(bool(request.allowed))  # 唤醒等待的协程
            continue
        
        if request.type == "question_response":
            future = self._question_requests[request.request_id]
            future.set_result(request.answer)
            continue
        
        # 将其他请求放入队列
        await self._request_queue.put(request)
```

**关键特性**：
- **后台任务**: 独立运行，不阻塞主循环
- **线程处理**: `asyncio.to_thread()` 处理阻塞的 stdin 读取
- **请求队列**: 将前端请求解耦到主循环
- **Future 机制**: 权限/问题响应通过 Future 与阻塞的请求匹配

### 3. **请求处理** (主循环中的分支逻辑)
**位置**: [backend_host.py#L102-L144](src/openharness/ui/backend_host.py#L102-L144)

```python
while self._running:
    request = await self._request_queue.get()
    
    # 请求类型分发
    if request.type == "shutdown":
        await self._emit(BackendEvent(type="shutdown"))
        break
    
    elif request.type in ("permission_response", "question_response"):
        # 已在 _read_requests 中处理，跳过
        continue
    
    elif request.type == "list_sessions":
        await self._handle_list_sessions()
    
    elif request.type == "select_command":
        await self._handle_select_command(request.command or "")
    
    elif request.type == "apply_select_command":
        # 处理下拉菜单选择（provider, theme 等）
        should_continue = await self._apply_select_command(...)
    
    elif request.type == "submit_line":  # ← 主要路径
        # 检查是否忙碌
        if self._busy:
            await self._emit(BackendEvent(type="error", message="Session is busy"))
            continue
        
        self._busy = True
        try:
            should_continue = await self._process_line(line)
        finally:
            self._busy = False
        
        if not should_continue:
            break
```

---

## 核心流程：提交行处理

### `_process_line()` 方法
**位置**: [backend_host.py#L177-L210](src/openharness/ui/backend_host.py#L177-L210)

这是消息循环的**核心执行路径**，处理用户输入：

```python
async def _process_line(self, line: str, *, transcript_line: str | None = None) -> bool:
    # 1. 发送用户输入到前端（记录到 transcript）
    await self._emit(
        BackendEvent(
            type="transcript_item",
            item=TranscriptItem(role="user", text=transcript_line or line)
        )
    )
    
    # 2. 定义事件处理回调
    async def _print_system(message: str) -> None:
        """处理系统消息"""
        await self._emit(
            BackendEvent(type="transcript_item", item=TranscriptItem(role="system", text=message))
        )
    
    async def _render_event(event: StreamEvent) -> None:
        """处理流事件（文本、工具执行、进度等）"""
        if isinstance(event, AssistantTextDelta):
            await self._emit(BackendEvent(type="assistant_delta", message=event.text))
        elif isinstance(event, CompactProgressEvent):
            await self._emit(BackendEvent(type="compact_progress", ...))
        elif isinstance(event, AssistantTurnComplete):
            await self._emit(BackendEvent(type="assistant_complete", ...))
        elif isinstance(event, ToolExecutionStarted):
            await self._emit(BackendEvent(type="tool_started", ...))
        elif isinstance(event, ToolExecutionCompleted):
            await self._emit(BackendEvent(type="tool_completed", ...))
        elif isinstance(event, ErrorEvent):
            await self._emit(BackendEvent(type="error", ...))
        # ... 更多事件类型
    
    async def _clear_output() -> None:
        """清除输出"""
        await self._emit(BackendEvent(type="clear_transcript"))
    
    # 3. 调用运行时处理（从 runtime.py）
    should_continue = await handle_line(
        self._bundle,
        line,
        print_system=_print_system,       # 系统消息回调
        render_event=_render_event,        # 流事件回调
        clear_output=_clear_output,        # 清除输出回调
    )
    
    # 4. 发送最终状态
    await self._emit(self._status_snapshot())
    await self._emit(BackendEvent.tasks_snapshot(get_task_manager().list_tasks()))
    await self._emit(BackendEvent(type="line_complete"))
    
    return should_continue  # 是否继续运行
```

### `handle_line()` 函数 (来自 runtime.py)
**位置**: [src/openharness/ui/runtime.py#L481-550](src/openharness/ui/runtime.py#L481-550)

```python
async def handle_line(
    bundle: RuntimeBundle,
    line: str,
    *,
    print_system: SystemPrinter,        # 系统消息回调
    render_event: StreamRenderer,       # 流事件回调
    clear_output: ClearHandler,         # 清除输出回调
) -> bool:
    """处理一行提交的输入"""
    
    # 1. 查找命令（/help, /provider 等）
    parsed = bundle.commands.lookup(line)
    if parsed is not None:
        command, args = parsed
        result = await command.handler(args, CommandContext(...))
        await _render_command_result(result, print_system, clear_output, render_event)
        
        # 如果命令要求提交提示词
        if result.submit_prompt is not None:
            system_prompt = build_runtime_system_prompt(...)
            try:
                # 2. 提交到 LLM 引擎，流式处理事件
                async for event in bundle.engine.submit_message(submit_prompt):
                    await render_event(event)  # ← 调用前端回调发送事件
            except MaxTurnsExceeded:
                await print_system(f"Stopped after {max_turns} turns")
            finally:
                # 保存会话快照
                bundle.session_backend.save_snapshot(...)
        
        return result.should_continue
    
    # 3. 如果不是命令，直接作为用户消息提交
    system_prompt = build_runtime_system_prompt(...)
    try:
        async for event in bundle.engine.submit_message(line):
            await render_event(event)  # ← 流式事件发送
    except MaxTurnsExceeded:
        await print_system(...)
    finally:
        bundle.session_backend.save_snapshot(...)
    
    return True  # 继续运行
```

---

## 事件流详解

### 流事件处理 (`_render_event()`)
**位置**: [backend_host.py#L199-L264](src/openharness/ui/backend_host.py#L199-L264)

事件类型及其处理：

| 事件类型 | 来源 | 处理 | 目标事件 |
|---------|------|------|--------|
| `AssistantTextDelta` | LLM 流式输出 | 流式文本 | `assistant_delta` |
| `CompactProgressEvent` | 推理进度 | 推理状态 | `compact_progress` |
| `AssistantTurnComplete` | LLM 完成 | 最终回复 + 任务快照 | `assistant_complete` + `tasks_snapshot` |
| `ToolExecutionStarted` | 工具执行 | 记录输入 | `tool_started` |
| `ToolExecutionCompleted` | 工具完成 | 结果 + 状态 | `tool_completed` + `status_snapshot` |
| `ErrorEvent` | 错误发生 | 错误消息 | `error` |
| `StatusEvent` | 状态变化 | 状态信息 | `transcript_item` |

**示例：工具执行事件流**
```python
# 1. 工具开始
await self._emit(BackendEvent(
    type="tool_started",
    tool_name="file_write",
    tool_input={"path": "test.txt", "content": "..."}
))

# 2. 工具完成
await self._emit(BackendEvent(
    type="tool_completed",
    tool_name="file_write",
    output="Successfully wrote test.txt",
    is_error=False
))

# 3. 更新状态
await self._emit(self._status_snapshot())

# 4. 更新任务
await self._emit(BackendEvent.tasks_snapshot(...))
```

---

## 事件发送机制

### `_emit()` 方法
**位置**: [backend_host.py#L787-L798](src/openharness/ui/backend_host.py#L787-L798)

```python
async def _emit(self, event: BackendEvent) -> None:
    """发送一个事件到前端"""
    async with self._write_lock:  # ← 线程安全
        # 构建 JSON-lines 协议消息
        payload = _PROTOCOL_PREFIX + event.model_dump_json() + "\n"
        #         "OHJSON:" + {...} + "\n"
        
        # 写入 stdout
        buffer = getattr(sys.stdout, "buffer", None)
        if buffer is not None:
            buffer.write(payload.encode("utf-8"))
            buffer.flush()
        else:
            sys.stdout.write(payload)
            sys.stdout.flush()
```

**协议格式**:
```
OHJSON:{"type":"transcript_item","item":{"role":"user","text":"hello"}}\n
OHJSON:{"type":"assistant_delta","message":"Hi"}\n
OHJSON:{"type":"assistant_delta","message":" there"}\n
...
```

---

## 权限和问题处理

### 异步权限请求
**位置**: [backend_host.py#L756-L775](src/openharness/ui/backend_host.py#L756-L775)

```python
async def _ask_permission(self, tool_name: str, reason: str) -> bool:
    """向前端请求权限（阻塞）"""
    async with self._permission_lock:
        request_id = uuid4().hex
        # 创建 Future，在收到响应时解决
        future = asyncio.get_running_loop().create_future()
        self._permission_requests[request_id] = future
        
        # 1. 发送权限请求到前端
        await self._emit(BackendEvent(
            type="modal_request",
            modal={
                "kind": "permission",
                "request_id": request_id,
                "tool_name": tool_name,
                "reason": reason,
            }
        ))
        
        # 2. 等待前端响应（超时 300s）
        try:
            return await asyncio.wait_for(future, timeout=300)
        except asyncio.TimeoutError:
            return False  # 默认拒绝
        finally:
            self._permission_requests.pop(request_id, None)
```

**交互流**:
```
后端                              前端
  ├─ emit modal_request
  │   request_id: "abc123"
  │   tool_name: "file_write"
  │
  └─ await future ←────────────────┐
                                   │
                        用户选择 Allow/Deny
                                   │
                    发送 permission_response
                    request_id: "abc123"
                    allowed: true
                                   │
  ┌──────────────────────────────→┘
  │
  future.set_result(True)
  continue
```

---

## 数据结构

### 前端请求 (`FrontendRequest`)
**位置**: [src/openharness/ui/protocol.py](src/openharness/ui/protocol.py)

```python
@dataclass
class FrontendRequest:
    type: str  # "submit_line", "shutdown", "select_command", etc.
    line: str | None = None  # 用户输入
    command: str | None = None  # 命令名
    value: str | None = None  # 选择值
    request_id: str | None = None  # 权限/问题请求 ID
    allowed: bool | None = None  # 权限响应
    answer: str | None = None  # 问题回答
```

### 后端事件 (`BackendEvent`)
```python
@dataclass
class BackendEvent:
    type: str  # "transcript_item", "assistant_delta", "tool_started", etc.
    message: str | None = None
    item: TranscriptItem | None = None  # 对话项
    tool_name: str | None = None
    tool_input: dict | None = None
    modal: dict | None = None  # 权限/问题模态框
    select_options: list[dict] | None = None
    # ... 更多字段
```

---

## 完整消息流示例

用户输入 "告诉我 /Users/test.txt 的内容"：

```
1. 前端发送
   OHJSON:{"type":"submit_line","line":"告诉我 /Users/test.txt 的内容"}\n

2. 后端读取
   _read_requests() → request_queue.put(FrontendRequest(type="submit_line", line="..."))

3. 主循环处理
   request = await request_queue.get()
   await _process_line("告诉我 /Users/test.txt 的内容")

4. 发送用户 transcript
   OHJSON:{"type":"transcript_item","item":{"role":"user","text":"..."}}\n

5. 调用 handle_line → engine.submit_message()

6. 工具执行开始
   OHJSON:{"type":"tool_started","tool_name":"read_file","tool_input":{"path":"/Users/test.txt"}}\n

7. 工具执行完成，LLM 开始生成回复
   OHJSON:{"type":"assistant_delta","message":"文件内容是："}\n
   OHJSON:{"type":"assistant_delta","message":"..."}\n

8. 工具完成
   OHJSON:{"type":"tool_completed","tool_name":"read_file","output":"文件内容","is_error":false}\n

9. LLM 完成
   OHJSON:{"type":"assistant_complete","message":"...完整回复...","item":{...}}\n
   OHJSON:{"type":"tasks_snapshot","tasks":[...]}\n

10. 行处理完成
    OHJSON:{"type":"status_snapshot",...}\n
    OHJSON:{"type":"line_complete"}\n
```

---

## 关键设计特性

### 1. **异步并发**
- 主循环和请求读取分离，不相互阻塞
- 使用 `asyncio.Queue` 解耦 I/O 和处理

### 2. **流式事件**
- 引擎通过 async generator 流式发送事件
- 每个事件立即通过回调发送到前端
- 支持实时文本流、进度更新等

### 3. **线程安全**
- `_write_lock` 保护 stdout 写入
- `_permission_lock` 保护权限请求
- 所有 I/O 通过 `asyncio.to_thread()` 处理

### 4. **忙碌状态管理**
- `self._busy` 防止并发处理
- 前端应等待 `line_complete` 后才发送下一个请求

### 5. **Future-based 请求/响应**
- 权限和问题请求等待 Future
- 响应返回时唤醒等待的协程
- 避免使用回调金字塔

---

## 总结：消息循环体现的代码位置

| 层级 | 位置 | 功能 |
|-----|------|------|
| **主循环** | [backend_host.py#L84-L150](src/openharness/ui/backend_host.py#L84-L150) | `run()` - 初始化和消息分发 |
| **请求读取** | [backend_host.py#L152-L175](src/openharness/ui/backend_host.py#L152-L175) | `_read_requests()` - stdin 解析 |
| **请求分发** | [backend_host.py#L102-L144](src/openharness/ui/backend_host.py#L102-L144) | 主循环中的类型检查和分发 |
| **行处理** | [backend_host.py#L177-L210](src/openharness/ui/backend_host.py#L177-L210) | `_process_line()` - 执行核心逻辑 |
| **运行时处理** | [runtime.py#L481-550](src/openharness/ui/runtime.py#L481-550) | `handle_line()` - 命令/消息分发 |
| **事件处理** | [backend_host.py#L199-L264](src/openharness/ui/backend_host.py#L199-L264) | `_render_event()` - 事件转换 |
| **事件发送** | [backend_host.py#L787-L798](src/openharness/ui/backend_host.py#L787-L798) | `_emit()` - stdout 输出 |
| **权限处理** | [backend_host.py#L756-L775](src/openharness/ui/backend_host.py#L756-L775) | `_ask_permission()` - 模态框 |
