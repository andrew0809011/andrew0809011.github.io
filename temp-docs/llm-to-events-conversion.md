# LLM 原始内容到项目事件的转换流程

本文档详细说明 `QueryEngine.submit_message()` 方法如何将 LLM 的原始流式响应转换为 OpenHarness 内部事件系统。

---

## 架构概览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         QueryEngine.submit_message()                         │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ 1. 接收用户输入                                                        │   │
│  │    prompt (str) → ConversationMessage                                │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                    ↓                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ 2. 调用 run_query()                                                    │   │
│  │    - 构建 QueryContext                                                 │   │
│  │    - 启动对话循环                                                      │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                    ↓                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ 3. 与 LLM API 交互                                                     │   │
│  │    api_client.stream_message() → 原始 API 事件流                        │   │
│  │    - ApiTextDeltaEvent                                                 │   │
│  │    - ApiMessageCompleteEvent                                          │   │
│  │    - ApiRetryEvent                                                    │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                    ↓                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ 4. 事件转换层                                                          │   │
│  │    原始 API 事件 → StreamEvent (项目事件)                               │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                    ↓                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ 5. 工具执行循环                                                        │   │
│  │    生成 ToolExecutionStarted/ToolExecutionCompleted 事件              │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                    ↓                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ 6. 返回 AsyncIterator[StreamEvent] 给调用者                            │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 详细流程

### 第一步：入口 - submit_message

**文件**: [src/openharness/engine/query_engine.py#L147-L190](../src/openharness/engine/query_engine.py#L147-L190)

```python
async def submit_message(self, prompt: str | ConversationMessage) -> AsyncIterator[StreamEvent]:
    """Append a user message and execute the query loop."""
    # 1. 将字符串输入转换为 ConversationMessage
    user_message = (
        prompt
        if isinstance(prompt, ConversationMessage)
        else ConversationMessage.from_user_text(prompt)
    )
    
    # 2. 记录用户目标到元数据
    if user_message.text.strip():
        remember_user_goal(self._tool_metadata, user_message.text)
    
    # 3. 添加到消息历史
    self._messages.append(user_message)
    
    # 4. 触发钩子（如果配置了 hook_executor）
    if self._hook_executor is not None:
        await self._hook_executor.execute(...)
    
    # 5. 构建 QueryContext
    context = QueryContext(
        api_client=self._api_client,
        tool_registry=self._tool_registry,
        # ... 其他配置
    )
    
    # 6. 调用 run_query 并流式返回事件
    async for event, usage in run_query(context, query_messages):
        if isinstance(event, AssistantTurnComplete):
            self._messages = list(query_messages)  # 更新消息历史
        if usage is not None:
            self._cost_tracker.add(usage)
        yield event
```

**关键点**:
- `submit_message` 是上层 API，负责准备上下文
- 实际的事件转换逻辑在 `run_query()` 中
- 返回 `AsyncIterator[StreamEvent]` 支持流式输出

---

### 第二步：核心循环 - run_query

**文件**: [src/openharness/engine/query.py#L455-L660](../src/openharness/engine/query.py#L455-L660)

```python
async def run_query(
    context: QueryContext,
    messages: list[ConversationMessage],
) -> AsyncIterator[tuple[StreamEvent, UsageSnapshot | None]]:
    """Run the conversation loop until the model stops requesting tools."""
    turn_count = 0
    while context.max_turns is None or turn_count < context.max_turns:
        turn_count += 1
        
        # 1. 检查并执行自动压缩（如果需要）
        async for event, usage in _stream_compaction(trigger="auto"):
            yield event, usage
        
        # 2. 流式调用 LLM API
        async for event in context.api_client.stream_message(
            ApiMessageRequest(...)
        ):
            # ┌─────────────────────────────────────────────────────────────────┐
            # │                    核心事件转换逻辑                              │
            # └─────────────────────────────────────────────────────────────────┘
            if isinstance(event, ApiTextDeltaEvent):
                # API 文本增量 → AssistantTextDelta
                yield AssistantTextDelta(text=event.text), None
                continue
                
            if isinstance(event, ApiRetryEvent):
                # API 重试事件 → StatusEvent
                yield StatusEvent(message=f"Request failed; retrying..."), None
                continue
                
            if isinstance(event, ApiMessageCompleteEvent):
                # API 消息完成 → 记录 final_message
                final_message = event.message
                usage = event.usage
```

---

### 第三步：事件转换详细映射

#### 1. 文本增量事件

**转换**: `ApiTextDeltaEvent` → `AssistantTextDelta`

```python
# 来源: src/openharness/engine/query.py#L538-L539
if isinstance(event, ApiTextDeltaEvent):
    yield AssistantTextDelta(text=event.text), None
```

| LLM 原始内容 | 项目事件 | 说明 |
|-------------|---------|------|
| `{"type": "content_block_delta", "delta": {"text": "Hello"}}` | `AssistantTextDelta(text="Hello")` | 流式文本片段 |

---

#### 2. 重试事件

**转换**: `ApiRetryEvent` → `StatusEvent`

```python
# 来源: src/openharness/engine/query.py#L541-L548
if isinstance(event, ApiRetryEvent):
    yield StatusEvent(
        message=(
            f"Request failed; retrying in {event.delay_seconds:.1f}s "
            f"(attempt {event.attempt + 1} of {event.max_attempts}): {event.message}"
        )
    ), None
```

| LLM 原始内容 | 项目事件 | 说明 |
|-------------|---------|------|
| 请求超时/失败 | `StatusEvent(message="Request failed; retrying...")` | 向用户显示重试状态 |

---

#### 3. 消息完成事件

**转换**: `ApiMessageCompleteEvent` → `AssistantTurnComplete`

```python
# 来源: src/openharness/engine/query.py#L550-L552
if isinstance(event, ApiMessageCompleteEvent):
    final_message = event.message
    usage = event.usage

# ... 后续处理 ...

# 来源: src/openharness/engine/query.py#L588
messages.append(final_message)
yield AssistantTurnComplete(message=final_message, usage=usage), usage
```

| LLM 原始内容 | 项目事件 | 说明 |
|-------------|---------|------|
| 完整响应消息 | `AssistantTurnComplete(message, usage)` | 一轮对话完成，包含完整消息和用量 |

**AssistantTurnComplete 结构**:
```python
@dataclass(frozen=True)
class AssistantTurnComplete:
    message: ConversationMessage  # 包含 TextBlock / ToolUseBlock
    usage: UsageSnapshot          # token 使用量
```

---

### 第四步：工具调用事件转换

当 LLM 返回包含工具调用的消息时，系统会生成额外的事件：

```python
# 来源: src/openharness/engine/query.py#L604-L616
if len(tool_calls) == 1:
    # 单个工具：顺序执行
    tc = tool_calls[0]
    yield ToolExecutionStarted(tool_name=tc.name, tool_input=tc.input), None
    result = await _execute_tool_call(context, tc.name, tc.id, tc.input)
    yield ToolExecutionCompleted(
        tool_name=tc.name,
        output=result.content,
        is_error=result.is_error,
    ), None
```

#### 工具事件映射表

| 阶段 | 项目事件 | 字段 | 说明 |
|-----|---------|------|------|
| 开始 | `ToolExecutionStarted` | `tool_name`, `tool_input` | 工具开始执行 |
| 完成 | `ToolExecutionCompleted` | `tool_name`, `output`, `is_error` | 工具执行完成 |

**示例流程**:
```
LLM 返回: {"tool_use": {"name": "read_file", "input": {"path": "test.py"}}}
    ↓
生成事件: ToolExecutionStarted(tool_name="read_file", tool_input={"path": "test.py"})
    ↓
执行工具 read_file
    ↓
生成事件: ToolExecutionCompleted(tool_name="read_file", output="content...", is_error=False)
```

---

### 第五步：错误处理事件

**转换**: 异常 → `ErrorEvent`

```python
# 来源: src/openharness/engine/query.py#L553-L567
except Exception as exc:
    error_msg = str(exc)
    if "connect" in error_msg.lower() or "timeout" in error_msg.lower():
        yield ErrorEvent(message=f"Network error: {error_msg}..."), None
    else:
        yield ErrorEvent(message=f"API error: {error_msg}"), None
    return
```

| 异常类型 | 项目事件 | 说明 |
|---------|---------|------|
| 网络错误 | `ErrorEvent(message="Network error: ...", recoverable=True)` | 可恢复错误 |
| API 错误 | `ErrorEvent(message="API error: ...")` | 其他错误 |

---

## 完整事件类型定义

**文件**: [src/openharness/engine/stream_events.py](../src/openharness/engine/stream_events.py)

```python
StreamEvent = (
    AssistantTextDelta      # 文本增量
    | AssistantTurnComplete  # 一轮完成
    | ToolExecutionStarted   # 工具开始
    | ToolExecutionCompleted # 工具完成
    | ErrorEvent            # 错误
    | StatusEvent           # 状态消息
    | CompactProgressEvent  # 压缩进度
)
```

| 事件类 | 来源 | 触发条件 |
|-------|------|---------|
| `AssistantTextDelta` | `ApiTextDeltaEvent` | LLM 流式返回文本 |
| `AssistantTurnComplete` | `ApiMessageCompleteEvent` | 一轮对话完成 |
| `ToolExecutionStarted` | 工具调用检测 | LLM 请求使用工具 |
| `ToolExecutionCompleted` | 工具执行完成 | 工具返回结果 |
| `ErrorEvent` | 异常捕获 | API 调用失败 |
| `StatusEvent` | `ApiRetryEvent` / 压缩 | 重试或系统状态 |
| `CompactProgressEvent` | 上下文压缩 | 自动压缩进度 |

---

## 事件流示例

### 场景：用户询问代码，LLM 先回答再读取文件

```
用户输入: "解释 src/main.py 的内容"
    ↓
[事件流开始]
    ↓
AssistantTextDelta(text="我来帮您查看")
AssistantTextDelta(text=" src/main.py")
AssistantTextDelta(text=" 的内容。")
AssistantTurnComplete(message=..., usage=...)  # LLM 决定调用工具
    ↓
ToolExecutionStarted(tool_name="read_file", tool_input={"path": "src/main.py"})
ToolExecutionCompleted(tool_name="read_file", output="...", is_error=False)
    ↓
AssistantTextDelta(text="这个文件包含...")
AssistantTextDelta(text="主要功能是...")
AssistantTurnComplete(message=..., usage=...)  # 完成，无更多工具调用
[事件流结束]
```

---

## 关键代码路径

| 功能 | 文件 | 行号 |
|-----|------|------|
| 入口方法 | `query_engine.py` | L147-190 |
| 核心循环 | `query.py` | L455-660 |
| 事件转换 | `query.py` | L528-588 |
| 工具执行 | `query.py` | L604-655 |
| 事件定义 | `stream_events.py` | L12-89 |
| 消息模型 | `messages.py` | L14-116 |

---

## 设计要点

1. **流式处理**: 使用 `AsyncIterator` 实现真正的流式输出，用户可以实时看到结果

2. **分层架构**: 
   - 底层: API 客户端处理原始 LLM 响应
   - 中层: `run_query` 转换为项目事件
   - 上层: UI 层消费事件并渲染

3. **状态管理**: `AssistantTurnComplete` 更新消息历史，确保多轮对话连续性

4. **错误恢复**: 网络错误可恢复，提示重试；压缩机制处理超长上下文

5. **工具整合**: 工具调用和结果被转换为事件，保持事件流的统一性
