# API Client 架构 - 官方 SDK vs 原始 HTTP

本文档详细说明 OpenHarness 的 API Client 架构，解释为什么使用官方 SDK 而非原始 HTTP client，以及如何实现多提供商支持。

---

## 架构概览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           QueryEngine (查询引擎)                             │
│                              (engine/query.py)                               │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ async for event in context.api_client.stream_message(request):       │   │
│  │                                                                      │   │
│  │ 不关心底层是 Anthropic 还是 OpenAI，只依赖协议接口                      │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                    ↑                                         │
│              SupportsStreamingMessages (Protocol)                            │
│                                    │                                         │
├────────────────────────────────────┼─────────────────────────────────────────┤
│                          API Client Layer                                    │
│                                                                              │
│  ┌─────────────────────────┐  ┌─────────────────────────┐  ┌─────────────┐  │
│  │  AnthropicApiClient     │  │ OpenAICompatibleClient  │  │ CopilotClient│ │
│  │  (api/client.py)        │  │ (api/openai_client.py)  │  │(api/copilot_ │ │
│  │                         │  │                         │  │ client.py)  │  │
│  ├─────────────────────────┤  ├─────────────────────────┤  ├─────────────┤  │
│  │ anthropic.AsyncAnthropic│  │    openai.AsyncOpenAI   │  │  (delegates  │  │
│  │         ↓               │  │         ↓               │  │   to OpenAI  │  │
│  │  官方 SDK 流式 API       │  │  官方 SDK 流式 API       │  │  client +    │  │
│  │  messages.stream()      │  │  chat.completions.create│  │  headers)    │  │
│  │                         │  │         ↓               │  │              │  │
│  │  ┌─────────────────┐    │  │  stream=True            │  │              │  │
│  │  │ SSE (Server-    │    │  │                         │  │              │  │
│  │  │  Sent Events)   │    │  │  ┌─────────────────┐    │  │              │  │
│  │  │  自动处理        │    │  │  │ SSE 流式响应     │    │  │              │  │
│  │  └─────────────────┘    │  │  │ 自动处理         │    │  │              │  │
│  │                         │  │  └─────────────────┘    │  │              │  │
│  └─────────────────────────┘  └─────────────────────────┘  └─────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 核心设计原则

### 为什么要使用官方 SDK 而非原始 HTTP Client？

| 优势 | 官方 SDK | 原始 HTTP Client |
|------|---------|------------------|
| **类型安全** | ✅ 完整的类型定义 | ❌ 需要自行定义模型 |
| **流式处理** | ✅ 内置 SSE 解析 | ❌ 需手动实现 SSE |
| **自动重连** | ✅ 内置网络层优化 | ❌ 需自行实现 |
| **认证处理** | ✅ 支持多种认证方式 | ❌ 需手动处理 Header |
| **错误处理** | ✅ 结构化异常类型 | ❌ 需解析 HTTP 状态码 |
| **功能更新** | ✅ SDK 跟随 API 更新 | ❌ 需手动适配新功能 |

> **关键决策**: OpenHarness 封装官方 SDK，在 SDK 之上做统一抽象，而非重复造轮子。

---

## 三种 Client 实现详解

### 1. AnthropicApiClient - Claude 官方 API

**文件**: `src/openharness/api/client.py:117-257`

```python
from anthropic import AsyncAnthropic

class AnthropicApiClient:
    """Thin wrapper around the Anthropic async SDK with retry logic."""

    def _create_client(self) -> AsyncAnthropic:
        kwargs: dict[str, Any] = {}
        if self._api_key:
            kwargs["api_key"] = self._api_key
        if self._auth_token:
            kwargs["auth_token"] = self._auth_token
            kwargs["default_headers"] = ...
        if self._base_url:
            kwargs["base_url"] = self._base_url
        return AsyncAnthropic(**kwargs)

    async def _stream_once(self, request: ApiMessageRequest) -> AsyncIterator[ApiStreamEvent]:
        """Single attempt at streaming a message."""
        params = {
            "model": request.model,
            "messages": [message.to_api_param() for message in request.messages],
            "max_tokens": request.max_tokens,
            "system": request.system_prompt,
            "tools": request.tools,
        }
        
        # 使用 SDK 的流式 API
        stream_api = self._client.beta.messages if self._claude_oauth else self._client.messages
        async with stream_api.stream(**params) as stream:
            async for event in stream:
                # SDK 自动处理 SSE，提供结构化事件
                if event.type == "content_block_delta":
                    delta = event.delta
                    if delta.type == "text_delta":
                        yield ApiTextDeltaEvent(text=delta.text)
            
            # 获取完整消息和用量
            final_message = await stream.get_final_message()
            yield ApiMessageCompleteEvent(
                message=assistant_message_from_api(final_message),
                usage=UsageSnapshot(...),
            )
```

**底层**: `anthropic>=0.40.0` (官方 Python SDK)

**调用方式**:
```
sdk.messages.stream(
    model="claude-sonnet-4-6",
    messages=[...],
    max_tokens=4096
) → SSE 流 → 结构化事件
```

---

### 2. OpenAICompatibleClient - OpenAI 兼容 API

**文件**: `src/openharness/api/openai_client.py`

```python
from openai import AsyncOpenAI

class OpenAICompatibleClient:
    """Client for OpenAI-compatible APIs (OpenAI, DashScope, GitHub Models, etc.)"""

    def __init__(self, api_key: str, base_url: str | None = None):
        self._client = AsyncOpenAI(api_key=api_key, base_url=base_url)

    async def stream_message(self, request: ApiMessageRequest) -> AsyncIterator[ApiStreamEvent]:
        # 将 Anthropic 格式消息转换为 OpenAI 格式
        openai_messages = _convert_messages_to_openai(
            request.messages, 
            request.system_prompt
        )
        openai_tools = _convert_tools_to_openai(request.tools)
        
        # 使用 SDK 的流式 API
        async for chunk in self._client.chat.completions.create(
            model=request.model,
            messages=openai_messages,
            tools=openai_tools or None,
            stream=True,  # ← 启用流式输出
            **_token_limit_param_for_model(request.model, request.max_tokens),
        ):
            delta = chunk.choices[0].delta
            
            # 处理文本增量
            if delta.content:
                yield ApiTextDeltaEvent(text=delta.content)
            
            # 处理工具调用增量
            if delta.tool_calls:
                for tc in delta.tool_calls:
                    # 累计工具调用片段
                    ...
        
        # 构造完整消息
        yield ApiMessageCompleteEvent(message=final_message, usage=...)
```

**底层**: `openai>=1.0.0` (官方 Python SDK)

**支持的服务商**:
| 服务商 | base_url 示例 | 说明 |
|--------|--------------|------|
| OpenAI | `https://api.openai.com/v1` | 官方 API |
| Alibaba DashScope | `https://dashscope.aliyuncs.com/compatible-mode/v1` | 阿里云 |
| GitHub Models | `https://models.inference.ai.azure.com` | GitHub Models |
| 本地 Ollama | `http://localhost:11434/v1` | 本地模型 |

---

### 3. CopilotClient - GitHub Copilot

**文件**: `src/openharness/api/copilot_client.py`

```python
class CopilotClient:
    """Copilot-aware API client implementing SupportsStreamingMessages.
    
    Copilot API 是 OpenAI-compatible 的，所以复用 OpenAICompatibleClient
    只需添加 Copilot-specific headers 和认证方式。
    """

    def __init__(self, github_token: str | None = None):
        auth_info = load_copilot_auth()
        token = github_token or auth_info.github_token
        
        # Copilot API endpoint
        base_url = copilot_api_base(auth_info.enterprise_url)
        
        # 构建带 Copilot headers 的 OpenAI client
        raw_openai = AsyncOpenAI(
            api_key=token,
            base_url=base_url,
            default_headers={
                "User-Agent": "openharness/0.1.0",
                "Openai-Intent": "conversation-edits",
            }
        )
        
        # 复用 OpenAICompatibleClient，但替换底层 SDK client
        self._inner = OpenAICompatibleClient(api_key=token, base_url=base_url)
        self._inner._client = raw_openai  # 注入自定义 headers

    async def stream_message(self, request: ApiMessageRequest) -> AsyncIterator[ApiStreamEvent]:
        # 委托给内部的 OpenAICompatibleClient
        async for event in self._inner.stream_message(request):
            yield event
```

**特点**: 
- 复用 `OpenAICompatibleClient` 的核心逻辑
- 仅替换 HTTP headers 和认证
- Copilot API 与 OpenAI API 结构一致

---

## 统一协议抽象

### Protocol 定义

**文件**: `src/openharness/api/client.py:79-84`

```python
class SupportsStreamingMessages(Protocol):
    """Protocol used by the query engine in tests and production."""

    async def stream_message(self, request: ApiMessageRequest) -> AsyncIterator[ApiStreamEvent]:
        """Yield streamed events for the request."""
```

**核心价值**: 
- QueryEngine 只依赖协议，不关心具体实现
- 可以在测试中用 mock 对象替换
- 未来添加新 LLM 提供商只需实现协议

### 为什么看不到 `OpenAICompatibleClient(SupportsStreamingMessages)` 的继承代码？

这是 Python **`Protocol`（结构化子类型）** 的特性，与传统继承不同：

```python
# ❌ 传统继承（需要显式声明）
class OpenAICompatibleClient(SupportsStreamingMessages):
    ...

# ✅ Protocol 的做法（隐式满足，只需方法签名匹配）
class OpenAICompatibleClient:          # ← 没有继承任何东西
    async def stream_message(          # ← 方法名、参数类型完全匹配
        self, request: ApiMessageRequest
    ) -> AsyncIterator[ApiStreamEvent]:
        ...
```

**Python 规则**：只要一个类实现了 `Protocol` 要求的所有方法，且签名完全一致，它就自动满足该协议，无需任何声明。这也叫 **duck typing（鸭子类型）**：

> "如果它走路像鸭子，叫声像鸭子，那它就是鸭子。"

**三个 Client 的满足情况**：

| Client | 显式继承 | 满足 Protocol | 原因 |
|--------|---------|--------------|------|
| `AnthropicApiClient` | ❌ 无 | ✅ 是 | 实现了 `stream_message(request: ApiMessageRequest)` |
| `OpenAICompatibleClient` | ❌ 无 | ✅ 是 | 实现了 `stream_message(request: ApiMessageRequest)` |
| Mock/测试对象 | ❌ 无 | ✅ 是 | 只要实现同名方法即满足 |

**`QueryContext` 中的声明**（[src/openharness/engine/query.py](../src/openharness/engine/query.py)）：

```python
@dataclass
class QueryContext:
    api_client: SupportsStreamingMessages   # ← 类型是 Protocol
    ...
```

`run_query()` 在调用 `context.api_client.stream_message(...)` 时，Python 不检查 `api_client` 的具体类型，只要它有 `stream_message` 方法即可。运行时完全由传入的具体对象决定用哪个实现。

### 统一事件类型

**文件**: `src/openharness/api/client.py:50-76`

```python
@dataclass(frozen=True)
class ApiTextDeltaEvent:
    """Incremental text produced by the model."""
    text: str

@dataclass(frozen=True)
class ApiMessageCompleteEvent:
    """Terminal event containing the full assistant message."""
    message: ConversationMessage
    usage: UsageSnapshot
    stop_reason: str | None = None

@dataclass(frozen=True)
class ApiRetryEvent:
    """A recoverable upstream failure that will be retried automatically."""
    message: str
    attempt: int
    max_attempts: int
    delay_seconds: float

ApiStreamEvent = ApiTextDeltaEvent | ApiMessageCompleteEvent | ApiRetryEvent
```

| SDK 原始事件 | 统一事件 | Anthropic SDK | OpenAI SDK |
|-------------|---------|---------------|------------|
| 文本增量 | `ApiTextDeltaEvent` | `content_block_delta` | `chunk.choices[0].delta.content` |
| 消息完成 | `ApiMessageCompleteEvent` | `stream.get_final_message()` | `[DONE]` 后的完整响应 |
| 工具调用 | 含在 `ConversationMessage` | `content_block` (type=tool_use) | `choices[0].message.tool_calls` |
| 错误重试 | `ApiRetryEvent` | 由重试逻辑生成 | 由重试逻辑生成 |

---

## `build_runtime()` 如何选择注入哪个 Client

**文件**: `src/openharness/ui/runtime.py` — `_resolve_api_client_from_settings()`

```
build_runtime(active_profile="openai-compatible", ...)
      │
      ├─ 1. settings = load_settings().merge_cli_overrides(...)
      │       ↑ 合并: 配置文件 → profile 字段 → CLI 参数
      │
      └─ 2. if api_client 参数已传入:
              └─ 直接使用（测试/SDK 场景）
         else:
              └─ _resolve_api_client_from_settings(settings)
```

`_resolve_api_client_from_settings()` 内部的决策树（按优先级）：

```
settings.api_format == "copilot"?
    └─ YES → CopilotClient(model=...)          ← GitHub Copilot OAuth

settings.provider == "openai_codex"?
    └─ YES → CodexApiClient(auth_token, base_url)  ← Codex 订阅

settings.provider == "anthropic_claude"?
    └─ YES → AnthropicApiClient(auth_token, claude_oauth=True, ...)  ← Claude 订阅

settings.api_format in ("openai", "openai_compat")?
    └─ YES → OpenAICompatibleClient(api_key, base_url, timeout)
                ↑ 你用的是这个！DashScope/OpenAI/DeepSeek 等

默认 (fallback)
    └─ AnthropicApiClient(api_key, base_url)   ← 标准 Anthropic API key
```

**`api_format` 从哪里来？**

```
oh provider use openai-compatible
    │
    ├─ 写入 settings: active_profile = "openai-compatible"
    │
    └─ ProviderProfile.api_format = "openai"   ← settings.py default_provider_profiles()
```

内置 profile 的 `api_format` 映射（`src/openharness/config/settings.py`）：

| Profile | `api_format` | 注入的 Client |
|---------|-------------|--------------|
| `claude-api` | `"anthropic"` | `AnthropicApiClient` |
| `claude-subscription` | `"anthropic"` + `provider=anthropic_claude` | `AnthropicApiClient(claude_oauth=True)` |
| `openai-compatible` | `"openai"` | `OpenAICompatibleClient` |
| `codex` | `"openai"` + `provider=openai_codex` | `CodexApiClient` |
| `github-copilot` | `"copilot"` | `CopilotClient` |

**完整调用链（以 DashScope 为例）**：

```
oh run "帮我写代码" --profile openai-compatible
      │
      ├─ build_runtime(active_profile="openai-compatible")
      │       └─ settings.api_format == "openai"
      │             └─ OpenAICompatibleClient(api_key=..., base_url="https://dashscope...")
      │
      └─ QueryEngine.submit_message("帮我写代码")
              └─ run_query(context, messages)
                      └─ context.api_client.stream_message(request)
                              ↑ 运行时调用 OpenAICompatibleClient.stream_message()
                              └─ _convert_messages_to_openai(messages)  ← Anthropic→OpenAI格式转换
                              └─ self._client.chat.completions.create(stream=True)
```

---

## 格式转换机制 Anthropic ↔ OpenAI

**文件**: `src/openharness/api/openai_client.py`

### 核心差异对照表

| 方面 | Anthropic 格式 | OpenAI 格式 |
|------|--------------|------------|
| system prompt | 独立参数 `"system": [...]` | 第一条消息 `{"role":"system", "content":"..."}` |
| tool 定义 | `{"name":"...", "input_schema":{...}}` | `{"type":"function", "function":{"parameters":{...}}}` |
| tool 调用（assistant） | content block: `{"type":"tool_use", "id":"...", "input":{...}}` | `message.tool_calls: [{"id":"...", "function":{"arguments":"..."}}]` |
| tool 结果（user） | content block: `{"type":"tool_result", "tool_use_id":"..."}` | 独立消息 `{"role":"tool", "tool_call_id":"..."}` |
| 图片 | `{"type":"image", "source":{"type":"base64",...}}` | `{"type":"image_url", "image_url":{"url":"data:..."}}` |

### 1. `_convert_messages_to_openai()` — 消息列表转换

```
messages (Anthropic 格式)                    openai_messages (OpenAI 格式)
─────────────────────────────────            ─────────────────────────────────────────
[system_prompt]              →               {"role":"system", "content": system_prompt}

{"role":"user",              →               {"role":"user", "content": "用户文字"}
 "content":[TextBlock]}

{"role":"assistant",         →               {"role":"assistant",
 "content":[                                  "content": "回复文字",       ← text parts 合并
   TextBlock("回复文字"),                     "tool_calls": [              ← tool_use → tool_calls
   ToolUseBlock(id, name, input)              {"id":..., "type":"function",
 ]}                                             "function":{"name":..., "arguments": json}}
                                             ]}

{"role":"user",              →               {"role":"tool",               ← 每个 ToolResult 单独一条消息
 "content":[                                  "tool_call_id": tr.tool_use_id,
   ToolResultBlock(id, content)               "content": tr.content}
 ]}                                          {"role":"user", "content":"..."}  ← text 块（若有）
```

**关键注意**：Anthropic 里 tool result 和用户文字都放在同一条 `role="user"` 消息里，但 OpenAI 要求它们**分开**：tool result 是 `role="tool"`，文字是 `role="user"`。

### 2. `_convert_assistant_message()` — Assistant 消息的特殊处理

```python
openai_msg = {
    "role": "assistant",
    "content": "".join(text_parts) or None,     # text blocks 合并，空则 None
    "tool_calls": [                              # ToolUseBlock → tool_calls
        {
            "id": tu.id,
            "type": "function",
            "function": {
                "name": tu.name,
                "arguments": json.dumps(tu.input),  # input dict → JSON 字符串
            },
        }
        for tu in tool_uses
    ],
    # 思考模型特殊字段（Kimi k2.5 等）：
    "reasoning_content": msg._reasoning or "",   # 若有 tool_calls 则必须提供此字段
}
```

**思考模型（Kimi/DeepSeek-R1）的 reasoning_content**：
- 流式解析时，`delta.reasoning_content` 被累积到 `collected_reasoning`
- 存储在 `final_message._reasoning`（动态属性）
- 下一轮发回给 API 时，`_convert_assistant_message()` 从 `_reasoning` 读出并填入 `reasoning_content`
- 若有 `tool_calls` 但没有 reasoning，也必须填 `""` 否则 Kimi 报错

### 3. `_convert_tools_to_openai()` — Tool Schema 转换

```python
# Anthropic 格式（来自 tool.to_api_schema()）
{"name": "read_file", "description": "...", "input_schema": {"type":"object", "properties":{...}}}

# OpenAI 格式（转换后）
{"type": "function", "function": {
    "name": "read_file",
    "description": "...",
    "parameters": {"type":"object", "properties":{...}}  # input_schema → parameters
}}
```

### 4. 流式解析的特殊处理

`_stream_once()` 里收到 OpenAI stream chunks 时做了两件额外的事：

**① `<think>...</think>` 标签过滤**（用于 DeepSeek-R1 等内联推理模型）

```python
_think_buf += delta.content
visible, _think_buf = _strip_think_blocks(_think_buf)
if visible:
    collected_content += visible
    yield ApiTextDeltaEvent(text=visible)   # 只 yield 可见内容
```

- `_strip_think_blocks()` 用正则移除完整的 `<think>...</think>` 块
- 未闭合的 `<think>` 保留在缓冲区，等下一个 chunk 补全再决定是否过滤

**② tool_calls 的增量拼接**

OpenAI stream 中 tool_calls 是按 index 分块发来的（`tc_delta.index`），需要自行拼接：

```python
collected_tool_calls: dict[int, dict] = {}
for tc_delta in delta.tool_calls:
    idx = tc_delta.index          # 0, 1, 2, ... 对应多个并发 tool
    entry = collected_tool_calls.setdefault(idx, {"id":"", "name":"", "arguments":""})
    entry["arguments"] += tc_delta.function.arguments   # arguments 是流式追加的
```

最终按 `sorted(collected_tool_calls.keys())` 保证顺序，跳过 `name == ""` 的"幻影"工具调用（部分 provider 会发送空白 tool call）。

---

## 重试机制封装

**文件**: `src/openharness/api/client.py:160-196`

```python
async def stream_message(self, request: ApiMessageRequest) -> AsyncIterator[ApiStreamEvent]:
    """Yield text deltas and the final assistant message with retry on transient errors."""
    last_error: Exception | None = None

    for attempt in range(MAX_RETRIES + 1):
        try:
            # 实际调用
            async for event in self._stream_once(request):
                yield event
            return  # 成功，退出重试循环
            
        except OpenHarnessApiError:
            raise  # 认证错误不重试
            
        except Exception as exc:
            last_error = exc
            if attempt >= MAX_RETRIES or not _is_retryable(exc):
                raise  # 不可重试，抛出异常
            
            # 计算退避延迟
            delay = _get_retry_delay(attempt, exc)
            
            # 通知上层重试
            yield ApiRetryEvent(
                message=str(exc),
                attempt=attempt + 1,
                max_attempts=MAX_RETRIES + 1,
                delay_seconds=delay,
            )
            await asyncio.sleep(delay)
```

**重退避策略**: 指数退避 + 抖动 (jitter)
```python
delay = min(BASE_DELAY * (2 ** attempt), MAX_DELAY)  # 指数退避
jitter = random.uniform(0, delay * 0.25)            # 随机抖动
return delay + jitter
```

**可重试错误**:
- HTTP 429 (Rate Limit)
- HTTP 500, 502, 503, 529 (服务器错误)
- 网络连接错误 (ConnectionError, TimeoutError)

---

## 调用链路总结

### 完整请求流程

```
用户输入
    ↓
QueryEngine.submit_message(prompt)
    ↓
run_query(context, messages)
    ↓
context.api_client.stream_message(ApiMessageRequest(...))  ← 协议接口
    ↓
┌──────────────────────────────────────┐
│ AnthropicApiClient /                 │
│ OpenAICompatibleClient /             │
│ CopilotClient                        │
│                                      │
│ 1. _refresh_client_auth()            │
│ 2. anthropic.AsyncAnthropic          │
│    └─→ messages.stream()             │
│       └─→ HTTP POST /v1/messages     │
│          └─→ Server-Sent Events      │
│             └─→ SDK 解析 SSE         │
│                └─→ yield 结构化事件  │
└──────────────────────────────────────┘
    ↓
ApiTextDeltaEvent / ApiMessageCompleteEvent
    ↓
StreamEvent 转换 (query.py 中)
    ↓
AssistantTextDelta / AssistantTurnComplete
    ↓
前端 UI 渲染
```

---

## 关键代码文件

| 功能 | 文件 | 行号范围 |
|-----|------|---------|
| 协议定义 | `api/client.py` | L79-84 |
| Anthropic 实现 | `api/client.py` | L117-257 |
| OpenAI 兼容实现 | `api/openai_client.py` | L1-200+ |
| Copilot 实现 | `api/copilot_client.py` | L48-130 |
| 事件类型 | `api/client.py` | L50-76 |
| 重试逻辑 | `api/client.py` | L160-196 |
| 格式转换 | `api/openai_client.py` | L59-185 |
| 查询引擎使用 | `engine/query.py` | L528-555 |

---

## 设计要点总结

1. **SDK First**: 复用官方 SDK 的成熟实现，专注于业务逻辑而非 HTTP 细节

2. **协议抽象**: 通过 `SupportsStreamingMessages` 协议解耦 QueryEngine 和具体实现

3. **统一事件**: 无论底层是 Anthropic 还是 OpenAI，对外暴露统一的事件类型

4. **自动转换**: 内部自动处理 Anthropic ↔ OpenAI 的格式差异

5. ** resilience**: 内置指数退避重试，提高服务稳定性
