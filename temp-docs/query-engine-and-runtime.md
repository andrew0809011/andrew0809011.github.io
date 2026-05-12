# QueryEngine 与 Runtime 的配合关系

## 一句话总结

- **`RuntimeBundle`**：Session 的"配件盒"，持有所有资源（API client、工具注册表、权限检查器等）
- **`QueryEngine`**：消息循环的"驾驶员"，持有对话历史，驱动 model → tool 的多轮循环
- **`handle_line()`**：连接两者的桥梁，把用户输入路由到 `QueryEngine`

---

## 层次结构

```
用户输入 (stdin / TUI)
      │
      ▼
handle_line(bundle, line)          ← runtime.py 的核心函数
      │
      ├─ 命令？ (e.g. /clear /fast)
      │     └─ command.handler(...)
      │
      └─ 普通消息
            │
            ▼
      bundle.engine.submit_message(line)   ← QueryEngine
            │
            ▼
         run_query()                        ← engine/query.py
            │
            ├─ 自动压缩检查 (auto-compact)
            │
            ├─ api_client.stream_message()  ← 调用 LLM
            │
            ├─ 流式 yield AssistantTextDelta / AssistantTurnComplete
            │
            └─ 有 tool_use？
                  ├─ 是 → 执行工具 → 追加 tool_result → 再次循环
                  └─ 否 → 结束，return
```

---

## RuntimeBundle 是什么

`build_runtime()` 在 session 开始时组装一次，返回 `RuntimeBundle`，它只是一个**数据容器**：

| 字段 | 作用 |
|------|------|
| `api_client` | 与 LLM 通信（Anthropic / OpenAI / Copilot） |
| `tool_registry` | 所有可用工具的注册表（内置 + MCP + Plugin） |
| `engine` | QueryEngine 实例（持有对话历史） |
| `app_state` | UI 状态（当前 model、permission mode 等），只用于前端显示 |
| `hook_executor` | 生命周期钩子（SESSION_START、STOP 等） |
| `mcp_manager` | MCP 服务连接管理器 |
| `commands` | slash command 注册表 |
| `session_backend` | 会话快照持久化 |
| `settings_overrides` | CLI 传入的覆盖参数，保证 `/fast` 等命令不会"快照回退" |

**它本身不驱动任何循环**，只是把东西打包在一起。

---

## QueryEngine 做了什么

`QueryEngine` 是**有状态的对话管理器**，核心职责：

### 1. 持有对话历史

```python
self._messages: list[ConversationMessage]
```

每一轮 user → assistant → tool_result 的消息都追加在这里。`submit_message()` 把新的用户消息 append 进去，`run_query()` 结束后再把 model 回复 append 进去。

### 2. 驱动 model → tool 多轮循环

`submit_message()` 内部调用 `run_query()`，这是真正的循环：

```
while turn_count < max_turns:
    1. 检查是否需要自动压缩 (auto-compact)
    2. 调用 api_client.stream_message()   → 流式输出 AssistantTextDelta
    3. 收到 ApiMessageCompleteEvent       → yield AssistantTurnComplete
    4. final_message 有 tool_use?
       - 有 → 执行工具 → 追加 tool_result → continue (下一轮)
       - 无 → return (结束)
```

**max_turns** 是每次用户输入能触发的最大 model 轮次，默认 8，超过抛出 `MaxTurnsExceeded`。

### 3. 追踪 tool_metadata（跨轮次状态）

每次工具执行完毕，`_record_tool_carryover()` 会记录到 `tool_metadata`：

- `read_file_state`：最近读过哪些文件
- `recent_work_log`：最近的操作日志
- `task_focus_state`：当前目标、已验证工作
- `invoked_skills`：用过哪些 skill
- `async_agent_tasks`：派生的异步 agent 任务

这些信息会注入 system prompt，让 model 在每轮 "知道自己做过什么"。

### 4. 权限检查

调用工具前，`PermissionChecker` 会检查是否允许，不允许则弹出 `permission_prompt` 询问用户。

---

## 两者如何配合：完整流程

```
build_runtime()
    │
    ├─ 初始化 api_client (由 settings 决定用哪个 provider)
    ├─ 初始化 tool_registry (内置工具 + MCP + Plugin 工具)
    ├─ 初始化 PermissionChecker
    └─ 创建 QueryEngine(api_client, tool_registry, PermissionChecker, ...)
    
    返回 RuntimeBundle
          │
          ▼
    handle_line(bundle, "帮我读一下 README")
          │
          ├─ 更新 hook_executor（重载最新 hook）
          ├─ 更新 system_prompt（基于最新 settings）
          ├─ bundle.engine.set_system_prompt(system_prompt)
          │
          └─ async for event in bundle.engine.submit_message("帮我读一下 README"):
                │
                ├─ [内部] run_query() turn 1:
                │     LLM → 决定调用 read_file tool
                │     yield AssistantTextDelta / AssistantTurnComplete
                │     执行 read_file → 得到结果
                │     追加 tool_result 消息
                │
                ├─ [内部] run_query() turn 2:
                │     LLM → 根据文件内容回复
                │     yield AssistantTextDelta / AssistantTurnComplete
                │     无 tool_use → return
                │
                └─ 循环结束
          │
          ▼
    bundle.session_backend.save_snapshot(...)  ← 持久化快照
    sync_app_state(bundle)                     ← 刷新 UI 状态
```

---

## 关键设计决策

### 为什么 system_prompt 每次都重建？

```python
# handle_line() 里
system_prompt = build_runtime_system_prompt(
    settings,
    cwd=bundle.cwd,
    latest_user_prompt=line,   # ← 把用户当前输入也注入进去
    ...
)
bundle.engine.set_system_prompt(system_prompt)
```

因为 system prompt 包含动态内容（当前时间、最近读过的文件、已验证的工作），每次用户输入都需要重新生成，让 model 拿到最新的上下文。

### 为什么 settings_overrides 存在 RuntimeBundle？

```python
bundle.settings_overrides = {"model": "claude-opus-4", ...}
```

CLI 传入的参数（`--model`、`--api-format`）不应该被 `/fast` 等命令刷新 settings 后覆盖。`current_settings()` 每次从磁盘 load 再 merge CLI 覆盖，保证 CLI 参数优先。

### `continue_pending()` vs `submit_message()`

| | `submit_message()` | `continue_pending()` |
|---|---|---|
| 使用场景 | 普通用户输入 | 上次因 max_turns 中断，tool_result 还没被 model 回复 |
| 是否追加新 user message | ✅ | ❌ |
| 对应 slash command | 无 | `/continue` |

---

## system_prompt 是如何动态构建的

每次用户提交输入，`handle_line()` 都会调用 `build_runtime_system_prompt()` 重新组装 system prompt。它由多个 section 拼接而成：

```
build_runtime_system_prompt(settings, cwd, latest_user_prompt)
      │
      ├─ 1. 基础人设 + 环境信息 (build_system_prompt)
      │       ├─ _BASE_SYSTEM_PROMPT: "You are OpenHarness..."
      │       └─ # Environment: OS / Shell / cwd / Date / Git branch
      │
      ├─ 2. Session 模式（可选）
      │       └─ fast_mode → "Fast mode: prefer concise replies..."
      │
      ├─ 3. Reasoning 配置
      │       └─ effort / passes 设置
      │
      ├─ 4. 可用 Skills 列表（可选）
      │       └─ "# Available Skills\n- skill_name: description..."
      │
      ├─ 5. 子 Agent 委托说明
      │       └─ "# Delegation And Subagents..."
      │
      ├─ 6. 项目约定（可选）
      │       └─ CLAUDE.md / .oh/instructions.md 等文件内容
      │
      ├─ 7. 本地环境规则（可选）
      │       └─ load_local_rules() 从 personalization 加载
      │
      ├─ 8. Issue / PR 上下文（可选）
      │       └─ .oh/issue.md / .oh/pr_comments.md
      │
      └─ 9. Memory（可选）
              ├─ load_memory_prompt(): 全局记忆文件概要
              └─ find_relevant_memories(latest_user_prompt): 与当前输入相关的记忆文件
```

### 为什么每次用户输入都重建？

- **`latest_user_prompt`** 作为参数传入，用于 `find_relevant_memories()` 做语义搜索，找到与本次提问最相关的记忆文件并注入
- **`Date`** 每次重建都是当前时间，保持时效性
- **`cwd`**、**skills** 可能在 session 中发生变化（`/cd` 命令）

### 关键文件

| 功能 | 文件 |
|------|------|
| system_prompt 总组装 | [src/openharness/prompts/context.py](../src/openharness/prompts/context.py) |
| 基础人设 + 环境 | [src/openharness/prompts/system_prompt.py](../src/openharness/prompts/system_prompt.py) |
| 环境信息采集 | [src/openharness/prompts/environment.py](../src/openharness/prompts/environment.py) |
| Memory 注入 | [src/openharness/memory/](../src/openharness/memory/) |
| Skills 枚举 | [src/openharness/skills/loader.py](../src/openharness/skills/loader.py) |

---

## `messages` 列表的完整生命周期

> 核心问题：每一轮 LLM 思考或工具执行结果，**何时**、**以什么身份**被追加到 `messages`？

### 一图总览

```
submit_message(prompt)                    ← query_engine.py
│
├─ [1] self._messages.append(user_message)
│       role="user", text=用户输入
│
├─ query_messages = list(self._messages)  ← 浅拷贝，后续 run_query 在此对象上直接 append
│
└─ run_query(context, query_messages)     ← query.py，以下均在此函数内
       │
       ├─ ┌─────────── while turn_count < max_turns ───────────┐
       │  │                                                      │
       │  │  [A] auto_compact_if_needed(messages)               │
       │  │       若压缩：messages 被替换为压缩后的新列表        │
       │  │                                                      │
       │  │  api_client.stream_message(messages)  ← 发送给 LLM  │
       │  │       │                                              │
       │  │       ├─ ApiTextDeltaEvent  → yield AssistantTextDelta (不修改 messages)
       │  │       ├─ ApiRetryEvent      → yield StatusEvent     (不修改 messages)
       │  │       └─ ApiMessageCompleteEvent → final_message 暂存
       │  │                                                      │
       │  │  [2] messages.append(final_message)                  │
       │  │       role="assistant"                               │
       │  │       content = TextBlock(s) + ToolUseBlock(s)       │
       │  │       ↑ 此时 yield AssistantTurnComplete             │
       │  │       ↑ submit_message 监听到后立即同步:             │
       │  │       ↑   self._messages = list(query_messages)      │
       │  │                                                      │
       │  │  final_message.tool_uses 为空？                      │
       │  │       是 → return（循环结束）                        │
       │  │       否 → 执行工具                                  │
       │  │                                                      │
       │  │  执行工具（单个顺序 / 多个并发）                     │
       │  │       yield ToolExecutionStarted  (不修改 messages)  │
       │  │       yield ToolExecutionCompleted (不修改 messages) │
       │  │                                                      │
       │  │  [3] messages.append(                                │
       │  │         ConversationMessage(                         │
       │  │           role="user",                               │
       │  │           content=[ToolResultBlock, ...]             │
       │  │         )                                            │
       │  │       )                                              │
       │  │       ↑ 注意：role 是 "user"，不是 "tool"            │
       │  │       ↑ 这是 Anthropic API 的约定                    │
       │  │                                                      │
       │  │  turn_count += 1 → 进入下一轮                       │
       │  └──────────────────────────────────────────────────────┘
```

### 三次关键 append

| 步骤 | 发生位置 | role | content |
|------|---------|------|---------|
| **[1]** | `submit_message()` | `"user"` | 用户输入文本 |
| **[2]** | `run_query()` 每轮末 | `"assistant"` | LLM 的完整回复（文字 + tool_use 声明） |
| **[3]** | `run_query()` 工具执行后 | `"user"` | 所有工具的结果（`ToolResultBlock` 列表） |

**重点**：LLM 流式输出的每个 `ApiTextDeltaEvent`（即"思考/打字"过程）**不会**追加到 `messages`，只有 `ApiMessageCompleteEvent` 触发后，完整的 `final_message` 才被 append。

### `self._messages` 的同步时机

`submit_message()` 传给 `run_query()` 的是 `query_messages`（`self._messages` 的浅拷贝），两者是**不同的列表对象**。`self._messages` 只在以下时刻同步：

```python
# query_engine.py submit_message()
async for event, usage in run_query(context, query_messages):
    if isinstance(event, AssistantTurnComplete):
        self._messages = list(query_messages)  # ← 每次 LLM 完整回复后同步
```

即：**每次 LLM 完整回复后**（步骤 [2]），`self._messages` 才被更新为 `query_messages` 的当前快照。若 `run_query()` 中途报错，`self._messages` 保留到上一次成功的 assistant turn 为止。

### Coordinator 模式的特殊处理

当 system prompt 以 `"You are a **coordinator**."` 开头时，每轮 LLM 调用前会检查并**临时弹出** coordinator context message：

```python
# run_query() 内
if context.system_prompt.startswith("You are a **coordinator**."):
    if messages[-1].role == "user" and messages[-1].text.startswith("# Coordinator User Context"):
        coordinator_context_message = messages.pop()   # 暂时移除

# ... LLM 调用 ...

messages.append(final_message)                         # [2] append assistant 回复

if coordinator_context_message is not None:
    messages.append(coordinator_context_message)       # 重新追加到末尾
```

目的：保证 coordinator context 始终是 messages 列表的**最后一条 user 消息**，让每轮 LLM 拿到最新的 worker 上下文。

### 压缩（auto-compact）对 messages 的影响

每轮循环开头调用 `auto_compact_if_needed()`，若触发压缩：

```python
messages, was_compacted = last_compaction_result
# messages 可能是全新的列表（旧消息被摘要替换）
```

此时 `messages` 变量被重新赋值，指向压缩后的新列表。`query_messages`（`submit_message()` 里的引用）并不知道这件事，所以**压缩后 `self._messages` 的下一次同步**会拿到压缩后的版本。

---

## `_execute_tool_call` 详解

`_execute_tool_call` 是单次工具执行的完整生命周期，被 `run_query()` 调用（单工具顺序调用 / 多工具并发调用）。

### 执行流程（9 个阶段）

```
_execute_tool_call(context, tool_name, tool_use_id, tool_input)
│
├─ [1] PRE_TOOL_USE hook（可选）
│       hook_executor.execute(HookEvent.PRE_TOOL_USE, ...)
│       若 pre_hooks.blocked → 立即返回 ToolResultBlock(is_error=True)
│
├─ [2] 工具查找
│       tool = context.tool_registry.get(tool_name)
│       若 None → 返回 "Unknown tool" 错误
│
├─ [3] 输入校验
│       parsed_input = tool.input_model.model_validate(tool_input)
│       若校验失败 → 返回 "Invalid input" 错误
│
├─ [4] 路径 & 命令标准化
│       _file_path = _resolve_permission_file_path(cwd, raw_input, parsed_input)
│           优先从 raw_input dict 取 file_path/path/root
│           次从 parsed_input 属性取，相对路径转绝对路径
│       _command = _extract_permission_command(raw_input, parsed_input)
│           取 raw_input["command"] 或 parsed_input.command
│
├─ [5] 权限检查
│       decision = permission_checker.evaluate(
│           tool_name, is_read_only=..., file_path=_file_path, command=_command
│       )
│       若不允许：
│           ├─ requires_confirmation AND permission_prompt 存在
│           │     → 发 NOTIFICATION hook（弹出提示）
│           │     → 调用 permission_prompt(tool_name, reason)  ← 等待用户确认
│           │     → 用户拒绝 → 返回 "Permission denied" 错误
│           └─ 否则直接返回 "Permission denied" 错误
│
├─ [6] 实际执行
│       result = await tool.execute(
│           parsed_input,
│           ToolExecutionContext(
│               cwd=context.cwd,
│               metadata={
│                   "tool_registry": ...,       ← 工具可以调用其他工具
│                   "ask_user_prompt": ...,     ← 工具可以向用户提问
│                   **context.tool_metadata,    ← 携带跨轮次状态
│               },
│               hook_executor=...,
│           )
│       )
│
├─ [7] 封装结果
│       tool_result = ToolResultBlock(
│           tool_use_id=tool_use_id,   ← 与 LLM 请求中的 tool_use_id 对应
│           content=result.output,
│           is_error=result.is_error,
│       )
│
├─ [8] 记录 carryover（副作用，见下节）
│       _record_tool_carryover(context, tool_name, tool_input, tool_output, ...)
│
├─ [9] POST_TOOL_USE hook（可选）
│       hook_executor.execute(HookEvent.POST_TOOL_USE, {
│           tool_name, tool_input, tool_output, tool_is_error
│       })
│
└─ return ToolResultBlock
```

### 权限检查决策矩阵

| `decision.allowed` | `requires_confirmation` | `permission_prompt` 存在 | 结果 |
|---|---|---|---|
| ✅ | - | - | 继续执行 |
| ❌ | ✅ | ✅ | 弹出确认，用户决定 |
| ❌ | ✅ | ❌ | 直接拒绝（无 UI 时的 fallback） |
| ❌ | ❌ | - | 直接拒绝 |

### `ToolExecutionContext.metadata` 里有什么

工具执行时拿到的 `metadata` 包含三类内容：

| key | 来源 | 用途 |
|-----|------|------|
| `"tool_registry"` | `context.tool_registry` | 工具可以查询或调用其他工具（`agent` 工具靠此派发子任务） |
| `"ask_user_prompt"` | `context.ask_user_prompt` | 工具可以向用户提问（`ask_followup_question` 工具靠此实现） |
| `"task_focus_state"`, `"read_file_state"`, ... | `context.tool_metadata` | 跨轮次状态（已读文件、工作日志等） |

### `_record_tool_carryover` 都记录了什么

工具执行成功（`is_error=False`）后，自动更新 `tool_metadata`：

| 触发条件 | 记录内容 |
|---------|---------|
| 任意工具 + 有 `resolved_file_path` | `active_artifacts` 追加路径 |
| `read_file` | `read_file_state`、`recent_verified_work`、`recent_work_log` |
| `skill` | `invoked_skills`、`active_artifacts`、`recent_work_log` |
| `agent` / `send_message` | `async_agent_state`、`async_agent_tasks`、`recent_verified_work` |
| `enter_plan_mode` / `exit_plan_mode` | `permission_mode` 字段 |
| `web_fetch` | `active_artifacts`、`recent_verified_work` |
| `web_search` / `glob` / `grep` / `bash` | `recent_verified_work` 或 `recent_work_log` |

> 这些记录最终在下一次 `handle_line()` 时被注入 system prompt，让 LLM "记得"上一轮做过什么。

### 错误处理原则

- 所有错误都返回 `ToolResultBlock(is_error=True)`，**不抛异常**
- 这样 `run_query()` 里 `asyncio.gather(return_exceptions=True)` 收到的是正常 `ToolResultBlock`，不会导致其他并发工具被取消
- 唯一的例外是 `BaseException`（如 `asyncio.CancelledError`），这在 `run_query()` 的 `gather` 结果处理里统一包装成错误消息

---

## 工具注册表（ToolRegistry）与内置工具

### ToolRegistry 结构

定义在 [src/openharness/tools/base.py](../src/openharness/tools/base.py)，是一个简单的 `dict` 包装器：

```python
class ToolRegistry:
    _tools: dict[str, BaseTool]

    def register(tool: BaseTool)       # 按 tool.name 注册
    def get(name: str) -> BaseTool | None
    def list_tools() -> list[BaseTool]
    def to_api_schema() -> list[dict]  # 转换成 Anthropic API 格式发给 LLM
```

### BaseTool 接口

```python
class BaseTool(ABC):
    name: str                        # 工具唯一标识
    description: str                 # 注入 LLM 的描述
    input_model: type[BaseModel]     # Pydantic 输入校验模型

    async def execute(arguments, context: ToolExecutionContext) -> ToolResult
    def is_read_only(arguments) -> bool   # 默认 False，只读工具重写为 True
    def to_api_schema() -> dict           # {name, description, input_schema}
```

### 注册流程

```
build_runtime()                          ← ui/runtime.py
    │
    ├─ create_default_tool_registry(mcp_manager)   ← tools/__init__.py
    │       │
    │       ├─ 逐个 register() 38 个内置工具
    │       └─ 动态 register() MCP 工具（McpToolAdapter）
    │
    └─ 再 register() 插件工具（plugin.tools）
```

### 内置工具清单

| 工具名 | 功能 | 只读 |
|--------|------|------|
| `bash` | 执行 shell 命令 | ❌ |
| `read_file` | 读取文本文件 | ✅ |
| `write_file` | 创建或覆盖文件 | ❌ |
| `edit_file` | 替换文件中的文本片段 | ❌ |
| `glob` | 通配符文件搜索 | ✅ |
| `grep` | 正则表达式内容搜索 | ✅ |
| `notebook_edit` | 编辑 Jupyter notebook 单元格 | ❌ |
| `lsp` | 代码智能查询（定义、引用、悬停） | ✅ |
| `skill` | 读取已加载的 skill 内容 | ✅ |
| `tool_search` | 按名称/描述搜索可用工具 | ✅ |
| `web_search` | DuckDuckGo 网页搜索 | ✅ |
| `web_fetch` | 获取并处理网页内容 | ✅ |
| `brief` | 文本缩短/摘要 | ✅ |
| `sleep` | 暂停执行指定秒数 | ✅ |
| `config` | 读取/更新 OpenHarness 设置 | ❌ |
| `ask_user_question` | 向用户提问 | ✅ |
| `task_create` / `task_list` / `task_get` / `task_stop` / `task_update` / `task_output` | 后台任务管理 | 部分 |
| `enter_plan_mode` / `exit_plan_mode` | 切换权限模式 | ❌ |
| `enter_worktree` / `exit_worktree` | git worktree 管理 | ❌ |
| `cron_create` / `cron_list` / `cron_delete` / `cron_toggle` | 本地 cron 任务 | 部分 |
| `agent` | 生成本地后台 agent 任务 | ❌ |
| `send_message` | 向运行中的 agent 发送消息 | ❌ |
| `team_create` / `team_delete` | 轻量级内存团队管理 | ❌ |
| `todo_write` | markdown TODO 清单管理 | ❌ |
| `remote_trigger` | 立即触发 cron 任务 | ❌ |
| `mcp_auth` | 配置 MCP 服务器认证 | ❌ |
| `list_mcp_resources` / `read_mcp_resource` | MCP 资源访问 | ✅ |
| `mcp_tool_*` | MCP 服务器提供（动态注册） | 动态 |
| `plugin_*` | 插件提供（动态注册） | 动态 |

---

## 文件位置速查

| 功能 | 文件 |
|------|------|
| RuntimeBundle 组装 | [src/openharness/ui/runtime.py](../src/openharness/ui/runtime.py) |
| QueryEngine 定义 | [src/openharness/engine/query_engine.py](../src/openharness/engine/query_engine.py) |
| run_query 循环 | [src/openharness/engine/query.py](../src/openharness/engine/query.py) |
| handle_line 桥梁 | [src/openharness/ui/runtime.py](../src/openharness/ui/runtime.py)（函数 `handle_line`） |
| ToolRegistry / BaseTool | [src/openharness/tools/base.py](../src/openharness/tools/base.py) |
| 内置工具注册 | [src/openharness/tools/__init__.py](../src/openharness/tools/__init__.py) |
| 各内置工具实现 | [src/openharness/tools/](../src/openharness/tools/)（各子文件） |
