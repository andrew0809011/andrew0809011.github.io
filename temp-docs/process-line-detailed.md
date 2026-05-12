# _process_line() 处理流程详解

## 概述

`_process_line()` 是后端处理用户输入的核心方法。它接收一个用户输入字符串，然后经过以下步骤处理：

1. **记录用户输入** - 添加到转录记录（transcript）
2. **查找命令** - 检查是否是斜杠命令（/command）
3. **执行命令或提交消息** - 根据类型分别处理
4. **流式处理事件** - 将后端事件流式发送到前端

---

## 处理流程详解

### 位置
[src/openharness/ui/backend_host.py#L194-L290](../src/openharness/ui/backend_host.py#L194-L290)

### 完整代码流程

```python
async def _process_line(self, line: str, *, transcript_line: str | None = None) -> bool:
    # ┌─────────────────────────────────────────────────────────────┐
    # │ 步骤 1: 发出用户输入转录项 (TranscriptItem)                   │
    # └─────────────────────────────────────────────────────────────┘
    await self._emit(
        BackendEvent(
            type="transcript_item", 
            item=TranscriptItem(role="user", text=transcript_line or line)
        )
    )

    # 定义了三个关键的事件处理回调函数
    
    # ┌─────────────────────────────────────────────────────────────┐
    # │ 回调 1: _print_system() - 系统消息输出                        │
    # └─────────────────────────────────────────────────────────────┘
    async def _print_system(message: str) -> None:
        await self._emit(
            BackendEvent(
                type="transcript_item", 
                item=TranscriptItem(role="system", text=message)
            )
        )

    # ┌─────────────────────────────────────────────────────────────┐
    # │ 回调 2: _render_event() - 处理后端流式事件                     │
    # └─────────────────────────────────────────────────────────────┘
    async def _render_event(event: StreamEvent) -> None:
        # 2.1 处理 AI 助手的文本增量（流式输出）
        if isinstance(event, AssistantTextDelta):
            await self._emit(BackendEvent(type="assistant_delta", message=event.text))
            return
        
        # 2.2 处理紧凑进度事件（上下文压缩、重试等）
        if isinstance(event, CompactProgressEvent):
            await self._emit(
                BackendEvent(
                    type="compact_progress",
                    compact_phase=event.phase,
                    compact_trigger=event.trigger,
                    attempt=event.attempt,
                    compact_checkpoint=event.checkpoint,
                    compact_metadata=event.metadata,
                    message=event.message,
                )
            )
            return
        
        # 2.3 处理助手回合完成事件
        if isinstance(event, AssistantTurnComplete):
            await self._emit(
                BackendEvent(
                    type="assistant_complete",
                    message=event.message.text.strip(),
                    item=TranscriptItem(role="assistant", text=event.message.text.strip()),
                )
            )
            await self._emit(BackendEvent.tasks_snapshot(get_task_manager().list_tasks()))
            return
        
        # 2.4 处理工具执行开始事件
        if isinstance(event, ToolExecutionStarted):
            self._last_tool_inputs[event.tool_name] = event.tool_input or {}
            await self._emit(
                BackendEvent(
                    type="tool_started",
                    tool_name=event.tool_name,
                    tool_input=event.tool_input,
                    item=TranscriptItem(
                        role="tool",
                        text=f"{event.tool_name} {json.dumps(event.tool_input, ensure_ascii=True)}",
                        tool_name=event.tool_name,
                        tool_input=event.tool_input,
                    ),
                )
            )
            return
        
        # 2.5 处理工具执行完成事件
        if isinstance(event, ToolExecutionCompleted):
            await self._emit(
                BackendEvent(
                    type="tool_completed",
                    tool_name=event.tool_name,
                    output=event.output,
                    is_error=event.is_error,
                    item=TranscriptItem(
                        role="tool_result",
                        text=event.output,
                        tool_name=event.tool_name,
                        is_error=event.is_error,
                    ),
                )
            )
            # 刷新任务快照
            await self._emit(BackendEvent.tasks_snapshot(get_task_manager().list_tasks()))
            await self._emit(self._status_snapshot())
            
            # 特殊处理：TodoWrite 工具
            if event.tool_name in ("TodoWrite", "todo_write"):
                tool_input = self._last_tool_inputs.get(event.tool_name, {})
                todos = tool_input.get("todos") or tool_input.get("content") or []
                if isinstance(todos, list) and todos:
                    lines = []
                    for item in todos:
                        if isinstance(item, dict):
                            checked = item.get("status", "") in ("done", "completed", "x", True)
                            text = item.get("content") or item.get("text") or str(item)
                            lines.append(f"- [{'x' if checked else ' '}] {text}")
                    if lines:
                        await self._emit(
                            BackendEvent(type="todo_update", todo_markdown="\n".join(lines))
                        )
                else:
                    await self._emit_todo_update_from_output(event.output)
            
            # 特殊处理：权限/plan 模式切换
            if event.tool_name in ("set_permission_mode", "plan_mode"):
                assert self._bundle is not None
                new_mode = self._bundle.app_state.get().permission_mode
                await self._emit(
                    BackendEvent(type="plan_mode_change", plan_mode=new_mode)
                )
            return
        
        # 2.6 处理错误事件
        if isinstance(event, ErrorEvent):
            await self._emit(BackendEvent(type="error", message=event.message))
            await self._emit(
                BackendEvent(
                    type="transcript_item", 
                    item=TranscriptItem(role="system", text=event.message)
                )
            )
            return
        
        # 2.7 处理状态事件
        if isinstance(event, StatusEvent):
            await self._emit(
                BackendEvent(
                    type="transcript_item", 
                    item=TranscriptItem(role="system", text=event.message)
                )
            )
            return

    # ┌─────────────────────────────────────────────────────────────┐
    # │ 回调 3: _clear_output() - 清空转录                            │
    # └─────────────────────────────────────────────────────────────┘
    async def _clear_output() -> None:
        await self._emit(BackendEvent(type="clear_transcript"))

    # ┌─────────────────────────────────────────────────────────────┐
    # │ 步骤 2: 处理用户输入行 (关键!)                               │
    # └─────────────────────────────────────────────────────────────┘
    should_continue = await handle_line(
        self._bundle,
        line,
        print_system=_print_system,        # 系统消息回调
        render_event=_render_event,         # 事件流回调
        clear_output=_clear_output,         # 清空回调
    )

    # ┌─────────────────────────────────────────────────────────────┐
    # │ 步骤 3: 发送完成事件                                         │
    # └─────────────────────────────────────────────────────────────┘
    await self._emit(self._status_snapshot())
    await self._emit(BackendEvent.tasks_snapshot(get_task_manager().list_tasks()))
    await self._emit(BackendEvent(type="line_complete"))
    
    return should_continue  # 返回是否继续运行
```

---

## handle_line() - 命令分发

`handle_line()` 是 `_process_line()` 调用的核心函数，位于 [runtime.py#L487](../src/openharness/ui/runtime.py#L487)

```python
async def handle_line(
    bundle: RuntimeBundle,
    line: str,
    *,
    print_system: SystemPrinter,
    render_event: StreamRenderer,
    clear_output: ClearHandler,
) -> bool:
    """处理提交的一行输入"""
    
    # ┌─────────────────────────────────────────────────────────────┐
    # │ 第一步: 查找命令                                             │
    # └─────────────────────────────────────────────────────────────┘
    parsed = bundle.commands.lookup(line)  # ← 查找 /command
    
    if parsed is not None:
        # ✅ 这是一个命令 (如 /plan, /resume, /provider 等)
        command, args = parsed
        result = await command.handler(
            args,
            CommandContext(
                engine=bundle.engine,
                hooks_summary=bundle.hook_summary(),
                # ... 更多上下文信息
            ),
        )
        
        # 处理命令结果
        await _render_command_result(result, print_system, clear_output, render_event)
        
        # 如果命令返回了要提交的提示词（submit_prompt）
        if result.submit_prompt is not None:
            original_model = bundle.engine.model
            if result.submit_model:
                bundle.engine.set_model(result.submit_model)
            
            # 构建系统提示词
            settings = bundle.current_settings()
            submit_prompt = result.submit_prompt
            system_prompt = build_runtime_system_prompt(...)
            bundle.engine.set_system_prompt(system_prompt)
            
            # 提交给 AI 引擎进行处理
            try:
                async for event in bundle.engine.submit_message(submit_prompt):
                    await render_event(event)  # 流式处理事件
            except MaxTurnsExceeded as exc:
                await print_system(f"Stopped after {exc.max_turns} turns")
            finally:
                if result.submit_model:
                    bundle.engine.set_model(original_model)
        
        return True  # 继续运行
    
    # ┌─────────────────────────────────────────────────────────────┐
    # │ 第二步: 如果不是命令，提交给 AI 引擎                         │
    # └─────────────────────────────────────────────────────────────┘
    # 这是普通消息，直接提交给 Agent
    try:
        async for event in bundle.engine.submit_message(line):
            await render_event(event)  # 流式处理事件
    except MaxTurnsExceeded as exc:
        await print_system(f"Stopped after {exc.max_turns} turns")
    
    return True  # 继续运行
```

---

## 事件处理流程图

```
用户输入字符串
    ↓
_process_line(line)
    │
    ├─ 发送 TranscriptItem(role="user", text=line) ← 前端显示用户输入
    │
    ├─ handle_line(line)
    │   ├─ 查找命令 (commands.lookup)
    │   │  │
    │   │  ├─ YES: 执行命令处理器 → 可能产生 submit_prompt
    │   │  │
    │   │  └─ NO: 直接提交给 AI 引擎
    │   │
    │   └─ engine.submit_message(line)
    │       ├─ AssistantTextDelta → assistant_delta 事件
    │       ├─ CompactProgressEvent → compact_progress 事件
    │       ├─ ToolExecutionStarted → tool_started 事件
    │       ├─ ToolExecutionCompleted → tool_completed 事件
    │       ├─ AssistantTurnComplete → assistant_complete 事件
    │       ├─ ErrorEvent → error 事件
    │       └─ StatusEvent → status 事件
    │
    ├─ 发送 state_snapshot
    ├─ 发送 tasks_snapshot
    ├─ 发送 line_complete ← 表示处理完成
    │
    └─ 返回 should_continue
```

---

## 关键点总结

| 阶段 | 说明 | 关键函数/类 |
|------|------|----------|
| **输入阶段** | 接收用户输入字符串 | `_process_line()` |
| **转录阶段** | 将用户输入添加到转录记录 | `TranscriptItem(role="user")` |
| **分发阶段** | 区分命令 vs 普通消息 | `handle_line()` + `commands.lookup()` |
| **命令阶段** | 执行斜杠命令（/plan 等） | `command.handler()` |
| **AI 处理阶段** | 提交给 Agent 引擎处理 | `bundle.engine.submit_message()` |
| **流式事件阶段** | 处理 AI 生成的事件流 | `_render_event()` |
| **完成阶段** | 发送完成信号 | `line_complete` 事件 |

---

## 特殊事件处理

### 1. TodoWrite 工具完成时
当 `TodoWrite` 或 `todo_write` 工具执行完毕，自动发送 `todo_update` 事件：
```python
if event.tool_name in ("TodoWrite", "todo_write"):
    # 提取 todos 列表
    # 格式化为 Markdown checkbox 列表
    # 发送 BackendEvent(type="todo_update", todo_markdown=...)
```

### 2. 权限模式切换时
当 `set_permission_mode` 或 `plan_mode` 工具执行完毕，自动发送 `plan_mode_change` 事件：
```python
if event.tool_name in ("set_permission_mode", "plan_mode"):
    new_mode = self._bundle.app_state.get().permission_mode
    # 发送 BackendEvent(type="plan_mode_change", plan_mode=new_mode)
```

### 3. 上下文压缩（Compaction）
当处理非常长的对话时，系统会自动触发各种 `CompactProgressEvent`：
- `hooks_start` - 开始钩子处理
- `context_collapse_start` - 开始上下文折叠
- `session_memory_start` - 开始会话内存压缩
- `compact_start` - 开始完整压缩
- `compact_retry` - 重试压缩

---

## 相关文件

- [backend_host.py](../src/openharness/ui/backend_host.py) - 消息循环和 _process_line 实现
- [runtime.py](../src/openharness/ui/runtime.py) - handle_line 实现和命令分发
- [protocol.py](../src/openharness/ui/protocol.py) - 事件和请求数据模型
