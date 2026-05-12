# OpenHarness SDK 一次性调用模式

> 权限全默认通过 · 完成后自动退出 · 结构化回调  
> 适用于：CI/CD 脚本、后端服务、跨语言调用

---

## 核心结论

OpenHarness 已内置三层 SDK 调用模式，无需任何改动即可集成。

| 层次 | 函数/类 | 适用场景 |
|------|---------|---------|
| **高层** | `run_print_mode()` | 简单输出，stdout 打印 |
| **中层** | `build_runtime` + `handle_line` | 需要结构化事件回调 ✅ 推荐 |
| **底层** | `QueryContext` | 自定义工具注册、Swarm |

---

## 权限模式说明

```python
permission_mode="full_auto"  # 所有工具自动通过
```

| 模式 | 行为 |
|------|------|
| `default` | 写操作弹窗确认（交互式默认） |
| `full_auto` | 全部自动通过（**SDK 推荐**） |
| `plan` | 阻止所有写操作（只读审查） |

> ⚠️ 即使是 `full_auto`，`~/.ssh/id_*`、`~/.aws/credentials`、`~/.kube/config` 等敏感路径始终受保护（硬编码在 `permissions/checker.py`）。

---

## 最简用法：`run_print_mode()`

位置：`src/openharness/ui/app.py`

```python
import asyncio
from openharness.ui.app import run_print_mode

asyncio.run(run_print_mode(
    prompt="帮我分析这个项目的结构",
    permission_mode="full_auto",   # 所有权限自动通过
    output_format="text",          # 或 "stream-json"
    api_key="sk-...",              # 或从环境变量读取
    model="claude-sonnet-4-6",
    max_turns=50,
    cwd="/path/to/project",
))
# ↑ 函数返回即完成，自动退出
```

`output_format="stream-json"` 时，stdout 输出结构化事件：
```json
{"type": "tool_started", "tool_name": "Bash", "tool_input": {"command": "ls"}}
{"type": "tool_completed", "tool_name": "Bash", "output": "...", "is_error": false}
{"type": "assistant_delta", "text": "..."}
{"type": "assistant_complete", "text": "..."}
```

---

## 推荐用法：带回调的中层 SDK

参见示例文件：[examples/sdk_one_shot.py](../examples/sdk_one_shot.py)

```python
import asyncio
from examples.sdk_one_shot import run_agent_once, ToolCall

async def main():
    result = await run_agent_once(
        prompt="列出当前目录的 Python 文件",
        cwd="/path/to/project",
        permission_mode="full_auto",  # 内部已设置，此参数仅文档说明
        max_turns=20,

        # 流式文字回调
        on_text=lambda chunk: print(chunk, end="", flush=True),

        # 工具开始回调
        on_tool_start=lambda name, inp: print(f"\n[工具] {name} 开始"),

        # 工具完成回调（含结果）
        on_tool_done=lambda tc: print(f"[工具] {tc.tool_name} ✓ ({len(tc.output)}字符)"),
    )

    print(f"成功: {result.success}")
    print(f"工具调用: {len(result.tool_calls)} 次")
    print(f"退出码: {result.exit_code}")

asyncio.run(main())
```

### `AgentRunResult` 结构

```python
@dataclass
class AgentRunResult:
    text: str                 # 助手最终回复文本
    tool_calls: list[ToolCall] # 所有工具调用记录
    success: bool             # False = 运行期发生错误
    error_message: str        # 错误详情
    exit_code: int            # 0=正常, 1=错误
```

### `ToolCall` 结构

```python
@dataclass
class ToolCall:
    tool_name: str            # 工具名，如 "Bash", "Write"
    tool_input: dict          # 工具参数
    output: str               # 工具输出
    is_error: bool            # True = 工具执行失败
```

---

## 底层用法：直接构建 Runtime

适合需要多轮会话或自定义工具的场景：

```python
import asyncio
from openharness.ui.runtime import build_runtime, start_runtime, handle_line, close_runtime

async def run_task(prompt: str) -> str:
    collected = []

    async def render(event):
        from openharness.engine.stream_events import AssistantTextDelta
        if isinstance(event, AssistantTextDelta):
            collected.append(event.text)

    bundle = await build_runtime(
        permission_mode="full_auto",
        permission_prompt=lambda *_: True,  # 冗余保险
        ask_user_prompt=lambda *_: "",      # 禁止交互提问
        enforce_max_turns=True,
        max_turns=50,
    )
    await start_runtime(bundle)
    try:
        await handle_line(
            bundle, prompt,
            print_system=lambda _: None,
            render_event=render,
            clear_output=lambda: None,
        )
    finally:
        await close_runtime(bundle)  # ← 自动清理

    return "".join(collected)

result = asyncio.run(run_task("用一句话解释递归"))
```

---

## CLI 方式（跨语言集成）

```bash
# 权限全通过，完成后退出，stream-json 输出
python -m openharness --dangerously-skip-permissions \
    --print "帮我分析这个项目" \
    --output-format stream-json \
    --max-turns 50 \
    --cwd /path/to/project
```

```bash
# 等价的环境变量方式
OPENHARNESS_PERMISSION_MODE=full_auto \
python -m openharness --print "..." --output-format stream-json
```

其他语言（Node.js、Go 等）通过子进程调用并解析 stream-json 即可。

---

## 关键源码位置

| 文件 | 说明 |
|------|------|
| [src/openharness/ui/app.py](../src/openharness/ui/app.py) | `run_print_mode()` 高层入口 |
| [src/openharness/ui/runtime.py](../src/openharness/ui/runtime.py) | `build_runtime()` 中层入口 |
| [src/openharness/permissions/modes.py](../src/openharness/permissions/modes.py) | `PermissionMode` 枚举 |
| [src/openharness/permissions/checker.py](../src/openharness/permissions/checker.py) | 权限评估逻辑，敏感路径保护 |
| [src/openharness/engine/stream_events.py](../src/openharness/engine/stream_events.py) | 所有事件类型定义 |
| [examples/sdk_one_shot.py](../examples/sdk_one_shot.py) | 完整可运行示例 |
