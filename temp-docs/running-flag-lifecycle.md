# `_running` 标志和消息循环生命周期

## 概述

在 `ReactBackendHost` 中，`self._running` 标志控制消息循环的生命周期。虽然名字叫 `_running`，但它的作用与实际行为略有不同。

---

## 初始化阶段

**文件**: [backend_host.py#L72-L78](../src/openharness/ui/backend_host.py#L72-L78)

```python
class ReactBackendHost:
    def __init__(self, config: BackendHostConfig) -> None:
        self._config = config
        self._bundle = None
        self._write_lock = asyncio.Lock()
        self._request_queue: asyncio.Queue[FrontendRequest] = asyncio.Queue()
        self._permission_requests: dict[str, asyncio.Future[bool]] = {}
        self._question_requests: dict[str, asyncio.Future[str]] = {}
        self._permission_lock = asyncio.Lock()
        self._busy = False
        self._running = True  # ← 初始设为 True
```

**关键点**: 
- `_running` 在构造函数中设置为 `True`
- **一旦初始化后，在 ReactBackendHost 中从不被设置为 False**

---

## 主消息循环

**文件**: [backend_host.py#L114-L160](../src/openharness/ui/backend_host.py#L114-L160)

```python
async def run(self) -> int:
    # ... 初始化代码 ...
    
    reader = asyncio.create_task(self._read_requests())
    try:
        # ┌───────────────────────────────────────────────┐
        # │ 主消息循环                                     │
        # └───────────────────────────────────────────────┘
        while self._running:  # ← 条件检查
            request = await self._request_queue.get()
            
            # ┌───────────────────────────────────────────────┐
            # │ 循环退出条件 1: shutdown 请求                 │
            # └───────────────────────────────────────────────┘
            if request.type == "shutdown":
                await self._emit(BackendEvent(type="shutdown"))
                break  # ← 退出循环 (不改变 _running)
            
            # ... 其他请求类型处理 ...
            
            # ┌───────────────────────────────────────────────┐
            # │ 循环退出条件 2: 权限/问题响应不继续            │
            # └───────────────────────────────────────────────┘
            if request.type in ("permission_response", "question_response"):
                continue  # ← 直接继续，不处理
            
            # ┌───────────────────────────────────────────────┐
            # │ 循环退出条件 3: 命令处理返回 False            │
            # └───────────────────────────────────────────────┘
            if request.type == "apply_select_command":
                if self._busy:
                    await self._emit(BackendEvent(type="error", message="Session is busy"))
                    continue
                
                self._busy = True
                try:
                    should_continue = await self._apply_select_command(
                        request.command or "",
                        request.value or "",
                    )
                finally:
                    self._busy = False
                
                if not should_continue:  # ← 检查返回值
                    await self._emit(BackendEvent(type="shutdown"))
                    break  # ← 退出循环
                continue
            
            # ┌───────────────────────────────────────────────┐
            # │ 循环退出条件 4: 处理行返回 False              │
            # └───────────────────────────────────────────────┘
            if request.type == "submit_line":
                if self._busy:
                    await self._emit(BackendEvent(type="error", message="Session is busy"))
                    continue
                
                line = (request.line or "").strip()
                if not line:
                    continue
                
                self._busy = True
                try:
                    should_continue = await self._process_line(line)  # ← 核心处理
                finally:
                    self._busy = False
                
                if not should_continue:  # ← 检查返回值
                    await self._emit(BackendEvent(type="shutdown"))
                    break  # ← 退出循环
    
    finally:
        # ┌───────────────────────────────────────────────┐
        # │ 清理阶段                                       │
        # └───────────────────────────────────────────────┘
        reader.cancel()
        with contextlib.suppress(asyncio.CancelledError):
            await reader
        if self._bundle is not None:
            await close_runtime(self._bundle)
    
    return 0
```

---

## 循环退出机制

在 `ReactBackendHost` 中，循环退出有 **4 个可能的触发条件**，但 **都通过 `break` 而非改变 `_running`**：

| 触发条件 | 来源 | 代码位置 | 说明 |
|--------|------|--------|------|
| **1. shutdown 请求** | 前端或 EOF | L123-L125 | 用户关闭应用或 stdin 关闭 |
| **2. 命令返回 False** | `_apply_select_command()` | L133-L142 | 某些特殊命令要求退出 |
| **3. 处理行返回 False** | `_process_line()` | L154-L159 | 用户输入或命令导致要求退出 |
| **4. 异常或清理** | try-finally | L161-L166 | 无论如何都会执行清理 |

---

## `should_continue` 的来源

### 来源 1: handle_line()

**文件**: [runtime.py#L487-L580](../src/openharness/ui/runtime.py#L487-L580)

```python
async def handle_line(
    bundle: RuntimeBundle,
    line: str,
    *,
    print_system: SystemPrinter,
    render_event: StreamRenderer,
    clear_output: ClearHandler,
) -> bool:
    """返回 True 表示继续运行，False 表示停止"""
    
    parsed = bundle.commands.lookup(line)
    
    if parsed is not None:
        command, args = parsed
        result = await command.handler(args, CommandContext(...))
        
        # 命令执行完成
        await _render_command_result(result, ...)
        
        # 如果命令返回了要提交的提示，处理它
        if result.submit_prompt is not None:
            # 提交给引擎
            try:
                async for event in bundle.engine.submit_message(result.submit_prompt):
                    await render_event(event)
            except MaxTurnsExceeded as exc:
                await print_system(f"Stopped after {exc.max_turns} turns")
        
        return True  # ← 命令处理完成，继续运行
    
    # 普通消息，提交给引擎
    try:
        async for event in bundle.engine.submit_message(line):
            await render_event(event)
    except MaxTurnsExceeded as exc:
        await print_system(f"Stopped after {exc.max_turns} turns")
    
    return True  # ← 消息处理完成，继续运行
```

**关键观察**: 在正常情况下，`handle_line()` **总是返回 True**。

---

### 来源 2: _process_line()

**文件**: [backend_host.py#L194-L290](../src/openharness/ui/backend_host.py#L194-L290)

```python
async def _process_line(self, line: str, *, transcript_line: str | None = None) -> bool:
    # ... 处理逻辑 ...
    
    should_continue = await handle_line(
        self._bundle,
        line,
        print_system=_print_system,
        render_event=_render_event,
        clear_output=_clear_output,
    )
    
    # 发送完成事件
    await self._emit(self._status_snapshot())
    await self._emit(BackendEvent.tasks_snapshot(get_task_manager().list_tasks()))
    await self._emit(BackendEvent(type="line_complete"))
    
    return should_continue  # ← 直接返回 handle_line 的返回值
```

---

### 来源 3: _apply_select_command()

**文件**: [backend_host.py#L323-L330](../src/openharness/ui/backend_host.py#L323-L330)

```python
async def _apply_select_command(self, command_name: str, value: str) -> bool:
    command = command_name.strip().lstrip("/").lower()
    selected = value.strip()
    
    line = self._build_select_command_line(command, selected)
    if line is None:
        await self._emit(BackendEvent(type="error", message=f"Unknown select command: {command_name}"))
        await self._emit(BackendEvent(type="line_complete"))
        return True  # ← 返回 True（继续）
    
    # 转换为命令行并处理
    return await self._process_line(line, transcript_line=f"/{command}")
    # ← 返回 _process_line 的结果
```

---

## 何时返回 False？

在标准的 OpenHarness 实现中，**`should_continue` 几乎总是 `True`**。

返回 `False` 的情况非常罕见，可能包括：

### 理论上的返回 False 情况：

1. **特殊命令处理** - 某些命令可能被设计为返回 `False` 来要求退出
2. **引擎配置** - 某些特殊的运行时配置可能导致返回 `False`
3. **扩展功能** - 如果 OpenHarness 的命令系统进行了定制

---

## 实际循环退出流程

```
用户操作
    ↓
    ├─ 关闭应用
    │  └─ 前端连接关闭 / stdin 关闭
    │     └─ _read_requests() 接收 EOF
    │        └─ 发送 FrontendRequest(type="shutdown")
    │           └─ 主循环接收 shutdown 请求
    │              └─ 发送 BackendEvent(type="shutdown")
    │                 └─ break ← 退出循环
    │
    ├─ 正常交互
    │  └─ 提交命令或消息
    │     └─ handle_line() 返回 True
    │        └─ 继续循环
    │
    └─ 异常或资源不足
       └─ 某个命令返回 False（极少见）
          └─ 发送 BackendEvent(type="shutdown")
             └─ break ← 退出循环
```

---

## `_running` 的实际作用

在当前实现中，`self._running` 的作用有限：

```python
while self._running:  # ← 这个条件几乎总是 True
    # ... 通过 break 而非改变 _running 来退出
    break  # ← 实际的退出机制
```

**更准确的流程图**:
```python
while True:  # ← 实际上是无限循环
    request = await self._request_queue.get()
    
    if should_exit:
        break  # ← 真正的退出机制
    
    # 继续处理
```

---

## 为什么设计成这样？

1. **灵活性** - `_running` 可以在其他线程/方法中设置为 False 来强制退出
2. **优雅关闭** - `break` 允许执行 finally 清理逻辑
3. **扩展性** - 如果需要在外部强制停止循环，只需 `host._running = False`

---

## 清理阶段

**文件**: [backend_host.py#L161-L166](../src/openharness/ui/backend_host.py#L161-L166)

```python
finally:
    # ┌─────────────────────────────────────────┐
    # │ 无论如何都会执行的清理                  │
    # └─────────────────────────────────────────┘
    
    # 1. 取消请求读取任务
    reader.cancel()
    with contextlib.suppress(asyncio.CancelledError):
        await reader
    
    # 2. 关闭运行时
    if self._bundle is not None:
        await close_runtime(self._bundle)

return 0  # ← 退出应用
```

---

## 时间线示例

```
T0: 创建 ReactBackendHost
    ↓
T1: __init__ → _running = True
    ↓
T2: run() 启动
    ├─ 初始化运行时
    ├─ 发送 ready 事件
    ├─ 启动 _read_requests() 任务
    ↓
T3: 进入 while self._running 循环
    ├─ 等待请求...
    ↓
T4-T100: 处理多个 submit_line 请求
    ├─ 每次 should_continue = True
    ├─ 继续循环...
    ↓
T101: 用户关闭应用
    └─ 前端发送 shutdown 请求
       ├─ 收到请求
       ├─ 发送 BackendEvent(type="shutdown")
       ├─ break ← 退出 while 循环
       ├─ 进入 finally 块
       ├─ 取消 _read_requests() 任务
       ├─ 关闭运行时
       └─ return 0
```

---

## 核心要点

| 要点 | 说明 |
|-----|------|
| **_running 初值** | `True` (在 `__init__` 中设置) |
| **_running 修改** | 在 ReactBackendHost 中**从不改变** |
| **循环退出机制** | 通过 `break` 而非 `_running = False` |
| **应该继续的标志** | `should_continue` (来自 handle_line) |
| **退出触发** | shutdown 请求、命令返回 False、异常 |
| **cleanup** | 无论如何都在 finally 块中执行 |

---

## 相关文件

- [backend_host.py](../src/openharness/ui/backend_host.py) - 主循环实现
- [runtime.py](../src/openharness/ui/runtime.py) - handle_line 实现
- [protocol.py](../src/openharness/ui/protocol.py) - 事件和请求类型
