# oh CLI 启动 TUI 的完整流程分析

## 核心要点

当你输入 `oh` 命令时，OpenHarness 会启动一个 React Ink 交互式终端 UI (TUI)。与 `ohmo` 不同，`oh` 是**核心 Agent 引擎的直接入口**，不依赖工作区。

## 1. 入口点：oh CLI

**文件**: `src/openharness/cli.py`

```python
app = typer.Typer(
    name="openharness",
    help="Oh my Harness! An AI-powered coding assistant.",
    add_completion=False,
    rich_markup_mode="rich",
    invoke_without_command=True,  # ← 关键：允许直接启动
)
```

- 注册子命令：`mcp`, `plugin`, `auth`, `provider`, `cron`, `autopilot`
- **`invoke_without_command=True`** 是关键，这意味着即使没有子命令，也会执行 `main()` 回调

## 2. 主回调函数：main()

**文件**: `src/openharness/cli.py`

这是 `oh` 命令的**核心决策点**：

```python
@app.callback(invoke_without_command=True)
def main(
    ctx: typer.Context,
    # Session control
    continue_session: bool = typer.Option(False, "--continue", "-c", ...),
    resume: str | None = typer.Option(None, "--resume", "-r", ...),
    
    # Model & Effort
    model: str | None = typer.Option(None, "--model", "-m", ...),
    max_turns: int | None = typer.Option(None, "--max-turns", ...),
    
    # Output modes
    print_mode: str | None = typer.Option(None, "--print", "-p", ...),
    
    # System & Context
    system_prompt: str | None = typer.Option(None, "--system-prompt", "-s", ...),
    base_url: str | None = typer.Option(None, "--base-url", ...),
    api_key: str | None = typer.Option(None, "--api-key", "-k", ...),
    api_format: str | None = typer.Option(None, "--api-format", ...),
    
    # Permissions
    permission_mode: str | None = typer.Option(None, "--permission-mode", ...),
    dangerously_skip_permissions: bool = typer.Option(False, "--dangerously-skip-permissions", ...),
    
    # Advanced
    debug: bool = typer.Option(False, "--debug", "-d", ...),
    backend_only: bool = typer.Option(False, "--backend-only", hidden=True),
    task_worker: bool = typer.Option(False, "--task-worker", hidden=True),
    cwd: str = typer.Option(str(Path.cwd()), "--cwd", ...),
) -> None:
    """Start an interactive session or run a single prompt."""
```

### 决策流程

```python
# 1. 检查是否调用了子命令
if ctx.invoked_subcommand is not None:
    return  # 交给子命令处理

# 2. 会话恢复模式
if continue_session or resume is not None:
    session_data = load_session_snapshot(cwd)
    asyncio.run(run_repl(..., restore_messages=session_data.get("messages")))
    return

# 3. 打印模式（非交互）
if print_mode is not None:
    asyncio.run(run_print_mode(prompt=print_mode, ...))
    return

# 4. 任务工作者模式
if task_worker:
    asyncio.run(run_task_worker(...))
    return

# 5. TUI 模式（默认） ⭐
asyncio.run(run_repl(prompt=None, cwd=cwd, model=model, ...))
```

## 3. TUI 启动链路

### 3.1 run_repl() 函数

**文件**: `src/openharness/ui/app.py`

```python
async def run_repl(
    *,
    prompt: str | None = None,
    cwd: str | None = None,
    model: str | None = None,
    max_turns: int | None = None,
    base_url: str | None = None,
    system_prompt: str | None = None,
    api_key: str | None = None,
    api_format: str | None = None,
    backend_only: bool = False,
    restore_messages: list[dict] | None = None,
    restore_tool_metadata: dict[str, object] | None = None,
    permission_mode: str | None = None,
) -> None:
    """Run the default OpenHarness interactive application (React TUI)."""
    
    # Backend-only 模式（供前端调用）
    if backend_only:
        await run_backend_host(...)
        return

    # TUI 模式（启动 React Ink 前端）
    exit_code = await launch_react_tui(
        prompt=prompt,
        cwd=cwd,
        model=model,
        max_turns=max_turns,
        base_url=base_url,
        system_prompt=system_prompt,
        api_key=api_key,
        api_format=api_format,
        permission_mode=permission_mode,
    )
    if exit_code != 0:
        raise SystemExit(exit_code)
```

### 3.2 launch_react_tui() 函数 ⭐ (TUI 的真正启动者)

**文件**: `src/openharness/ui/react_launcher.py`

```python
async def launch_react_tui(
    *,
    prompt: str | None = None,
    cwd: str | None = None,
    model: str | None = None,
    max_turns: int | None = None,
    base_url: str | None = None,
    system_prompt: str | None = None,
    api_key: str | None = None,
    api_format: str | None = None,
    permission_mode: str | None = None,
) -> int:
    """Launch the React terminal frontend as the default UI."""
```

#### 步骤 1: 定位前端

```python
frontend_dir = get_frontend_dir()
# 优先级：
# 1. openharness/_frontend/ (打包版本)
# 2. frontend/terminal/ (开发版本)
```

#### 步骤 2: 自动安装依赖（首次）

```python
npm = _resolve_npm()
if not (frontend_dir / "node_modules").exists():
    install = await asyncio.create_subprocess_exec(npm, "install", ...)
```

#### 步骤 3: 构建后端启动命令

```python
env = os.environ.copy()
env["OPENHARNESS_FRONTEND_CONFIG"] = json.dumps({
    "backend_command": build_backend_command(
        cwd=cwd or str(Path.cwd()),
        model=model,
        max_turns=max_turns,
        base_url=base_url,
        system_prompt=system_prompt,
        api_key=api_key,
        api_format=api_format,
        permission_mode=permission_mode,
    ),
    "initial_prompt": prompt,  # null for TUI mode
    "theme": _resolve_theme(),
})
```

**关键点**：`backend_command` 是 React 前端如何启动后端：
```bash
python -m openharness --backend-only --cwd /path/to/cwd --model claude-3-5-sonnet ...
```

#### 步骤 4: 启动前端进程

```python
tsx_cmd = _resolve_tsx(frontend_dir)
process = await asyncio.create_subprocess_exec(
    *tsx_cmd,
    "src/index.tsx",
    cwd=str(frontend_dir),
    env=env,
)
return await process.wait()
```

- 使用 `tsx` 直接执行 TypeScript（无编译）
- 调用 `frontend/terminal/src/index.tsx`
- 等待进程完成

## 4. React 前端的工作

React Ink 前端做以下工作：

```typescript
// frontend/terminal/src/index.tsx

// 1. 读取后端命令配置
const config = JSON.parse(process.env.OPENHARNESS_FRONTEND_CONFIG)

// 2. 启动后端子进程
const backend = spawn(config.backend_command)

// 3. 建立通信管道（JSON-lines）
backend.stdin // 发送请求
backend.stdout // 接收事件

// 4. 渲染 React Ink UI
// - 用户输入框
// - AI 流式响应
// - 工具执行进度
// - 消息历史
```

## 5. 后端子进程的启动

当 React 前端启动后端时，会执行：

```bash
python -m openharness --backend-only --cwd ... --model ...
```

这触发 `oh` CLI 的 `main()` 函数再次执行，但这次带 `--backend-only` 标志：

```python
if backend_only:
    await run_backend_host(
        cwd=cwd,
        model=model,
        max_turns=max_turns,
        ...
    )
    return
```

### run_backend_host() 的工作

**文件**: `src/openharness/ui/backend_host.py`

```python
class ReactBackendHost:
    async def run(self) -> int:
        # 1. 组装 RuntimeBundle
        self._bundle = await build_runtime(
            model=model,
            max_turns=max_turns,
            system_prompt=system_prompt,
            api_key=api_key,
            api_format=api_format,
            permission_mode=permission_mode,
        )
        
        # 2. 启动运行时
        await start_runtime(self._bundle)
        
        # 3. 循环处理前端请求
        while True:
            line = await stdin_read()
            request = parse_json(line)
            
            if request['type'] == 'user_message':
                # 调用 QueryEngine 处理用户消息
                async for event in self._bundle.engine.submit_message(request['text']):
                    # 将事件流式发送到前端
                    stdout_write(json.dumps(event))
```

## 补充：`python -m openharness`、`oh` 与后端的关系（简要）

前端常用的后端启动命令示例是：

```bash
python -m openharness --backend-only --cwd /path --model <model>
```

这看起来像是在“用 python 启动 openharness”，你可能会问“后端不是 `oh` 吗？”——实际上 `python -m openharness`、`openhar ness`（包入口）和 `oh` 都是同一套程序的不同启动方式：

- `pyproject.toml` 中将 `oh`、`openharness` 等命令指向 `openharness.cli:app`；换言之，`oh` 是一个可执行入口的快捷别名。
- `python -m openharness` 会执行 `src/openharness/__main__.py`，而 `__main__.py` 只是导入并运行 `openharness.cli.app`。因此三者最终都会调用同一个 `main()` 回调。

为什么前端用 `python -m openharness` 而不是直接用 `oh`？主要原因是确保使用当前的 Python 解释器（`sys.executable`），从而在虚拟环境中启动后端，避免路径/环境不一致的问题。

`--backend-only` 的作用：当 `main()` 收到这个隐藏选项时，它不会走默认的 TUI 启动路径，而是直接运行 `run_backend_host()`，使该进程以 JSON-lines 后端主机的角色运行，监听 stdin/stdout 与前端通信。

总结：前端启动后端时用 `python -m openharness --backend-only`，只是以更确定的方式在同一 Python 环境里调用与 `oh` 相同的 CLI 程序；该进程因为带上 `--backend-only` 而成为后端主机。

**RuntimeBundle 包含**：
- `QueryEngine`: Agent 核心逻辑
- `ToolRegistry`: 可用工具（file, shell, web 等）
- `PermissionChecker`: 权限检查
- `HookExecutor`: 钩子系统
- `McpClientManager`: MCP 服务器连接
- 等等...

## 6. 前后端通信协议 (JSON-lines)

### 前端 → 后端（请求）

```json
{"type": "user_message", "text": "用户输入的文本"}
{"type": "permission_response", "request_id": "...", "granted": true}
{"type": "question_response", "request_id": "...", "text": "用户的回答"}
```

### 后端 → 前端（事件）

```json
{"type": "assistant_text_delta", "text": "AI"}
{"type": "assistant_text_delta", "text": "流式"}
{"type": "assistant_text_delta", "text": "响应"}
{"type": "tool_execution_started", "tool_name": "file_read"}
{"type": "tool_execution_completed", "tool_name": "file_read", "result": "..."}
{"type": "assistant_turn_complete"}
{"type": "permission_request", "request_id": "...", "message": "需要执行..."}
{"type": "status_event", "message": "正在处理..."}
```

## 7. 完整启动流程图

```
user input: oh
    ↓
src/openharness/cli.py::main()
    ├─ 解析命令行参数
    ├─ 路由决策（子命令/打印/后端/TUI）
    └─ [TUI 路径] asyncio.run(run_repl(...))
        ↓
    src/openharness/ui/app.py::run_repl()
        ├─ 检查 backend_only 标志
        └─ [TUI 模式] await launch_react_tui(...)
            ↓
        src/openharness/ui/react_launcher.py::launch_react_tui()
            ├─ 定位前端 (frontend/terminal/)
            ├─ npm install (首次)
            ├─ 构建后端启动命令
            │  └─ python -m openharness --backend-only ...
            ├─ 写入环境变量 OPENHARNESS_FRONTEND_CONFIG
            └─ 启动进程: tsx src/index.tsx
                ↓
        React Ink 前端进程 (Node.js)
            ├─ 读取 OPENHARNESS_FRONTEND_CONFIG
            ├─ 启动后端子进程
            │  └─ python -m openharness --backend-only ... [来自 backend_command]
            └─ 建立 JSON-lines 通信
                ↓
        Python 后端进程
            ├─ oh CLI 再次执行，带 --backend-only
            ├─ src/openharness/cli.py::main()
            │  └─ await run_backend_host(...)
            └─ src/openharness/ui/backend_host.py::ReactBackendHost
                ├─ build_runtime()
                │  ├─ 加载配置和认证
                │  ├─ 初始化 ToolRegistry
                │  ├─ 加载 MCP, Hooks, Plugins, Skills
                │  └─ 创建 QueryEngine
                ├─ start_runtime()
                └─ 消息循环
                   ├─ 读取前端请求
                   ├─ 调用 QueryEngine
                   └─ 流式发送事件
                   
与此同时，前端：
    ├─ 监听后端事件流
    ├─ 渲染 UI
    │  ├─ 用户输入框
    │  ├─ AI 响应流
    │  ├─ 工具进度
    │  └─ 消息历史
    └─ 处理用户交互
       └─ 发送请求到后端
```

## 8. oh 与 ohmo 的区别

| 方面 | oh | ohmo |
|------|----|----|
| 入口 | `src/openharness/cli.py` | `ohmo/cli.py` |
| 核心启动 | `launch_react_tui()` | `launch_ohmo_react_tui()` |
| 工作区 | ❌ 无工作区 | ✅ 有工作区 (`~/.ohmo`) |
| 会话持久化 | ✅ 有（存储在 cwd） | ✅ 有（存储在 workspace） |
| 个人化 | ❌ 无 persona | ✅ 有 persona (soul.md) |
| 后端命令 | `python -m openharness ...` | `python -m ohmo ...` |
| 使用场景 | 快速实验 | 长期助手 |

## 9. 常见命令示例

```bash
# 启动 TUI
oh

# 指定模型
oh --model sonnet          # Claude Sonnet
oh -m gpt-4               # 需要配置 OpenAI API

# 限制工作量
oh --max-turns 3          # 最多 3 轮交互

# 自定义 API
oh --base-url https://api.anthropic.com
oh --api-key sk-xxxx
oh --api-format openai

# 权限模式
oh --permission-mode plan              # 需要确认每个工具
oh --dangerously-skip-permissions      # 跳过所有检查

# 系统提示词
oh -s "你是一个编程教练"

# 会话管理
oh --continue              # 继续上一个会话
oh --resume <session-id>  # 恢复指定会话

# 非交互模式
oh -p "写一个 hello world"     # 单次执行，打印结果

# 调试
oh --debug                 # 启用调试日志
```

## 10. 启动时间分析

| 阶段 | 耗时 | 说明 |
|------|------|------|
| CLI 解析 | < 100ms | typer 参数解析 |
| 前端定位 | < 50ms | 查找 frontend/terminal/ |
| npm install | 10-30s | 首次运行（只需一次） |
| 前端启动 | 1-2s | tsx 启动 React 进程 |
| 后端启动 | 2-5s | RuntimeBundle 构建 |
| MCP 连接 | 0-2s | 取决于配置 |
| TUI 首次渲染 | ~500ms | Ink 渲染 |
| **总计** | **~3-7s** | （首次 20-35s）|

## 关键代码位置速查

| 功能 | 文件 | 行号 |
|------|------|------|
| CLI 入口 | `src/openharness/cli.py` | 29-40 |
| main() 回调 | `src/openharness/cli.py` | 1332-1500 |
| run_repl() | `src/openharness/ui/app.py` | 35-73 |
| launch_react_tui() | `src/openharness/ui/react_launcher.py` | 118-175 |
| ReactBackendHost | `src/openharness/ui/backend_host.py` | 49-300 |
| RuntimeBundle | `src/openharness/ui/runtime.py` | 50-100 |
| QueryEngine | `src/openharness/engine/query_engine.py` | 19-205 |


```python
app = typer.Typer(
    name="openharness",
    help=(
        "Oh my Harness! An AI-powered coding assistant.\n\n"
        "Starts an interactive session by default, use -p/--print for non-interactive output."
    ),
    add_completion=False,
    rich_markup_mode="rich",
    invoke_without_command=True,  # ← 重要！有主命令回调
)
```

- 使用 `typer` 框架构建 CLI
- 注册子命令: `mcp`, `plugin`, `auth`, `provider`, `cron`, `autopilot`
- **关键**: `invoke_without_command=True` 表示即使没有子命令，也会执行主回调 `main()`
- **这就是 `oh` 能直接启动 TUI 的原因**

### 1.2 ohmo 命令入口

**文件**: `ohmo/cli.py`

```python
app = typer.Typer(
    name="ohmo",
    help="ohmo: a personal-agent app built on top of OpenHarness.",
    invoke_without_command=True,  # ← 同样重要
    add_completion=False,
)
```

- ohmo 是一个 **更高级的抽象**，基于 OpenHarness 核心构建
- 也有自己的 `main()` 回调函数
- 专注于"个人工作区、persona、会话持久化"
- 与 `oh` 是**平行的两个产品**，而不是层级关系

## 2. oh 命令启动 TUI 的详细流程 ⭐

**这是你最关心的部分！**

### 2.1 入口：main() 回调函数

**文件**: `src/openharness/cli.py`

```python
@app.callback(invoke_without_command=True)
def main(
    ctx: typer.Context,
    # --- Session ---
    continue_session: bool = typer.Option(False, "--continue", "-c", ...),
    resume: str | None = typer.Option(None, "--resume", "-r", ...),
    name: str | None = typer.Option(None, "--name", "-n", ...),
    # --- Model & Effort ---
    model: str | None = typer.Option(None, "--model", "-m", ...),
    effort: str | None = typer.Option(None, "--effort", ...),
    verbose: bool = typer.Option(False, "--verbose", ...),
    max_turns: int | None = typer.Option(None, "--max-turns", ...),
    # --- Output ---
    print_mode: str | None = typer.Option(None, "--print", "-p", ...),
    output_format: str | None = typer.Option(None, "--output-format", ...),
    # --- Permissions ---
    permission_mode: str | None = typer.Option(None, "--permission-mode", ...),
    dangerously_skip_permissions: bool = typer.Option(False, "--dangerously-skip-permissions", ...),
    allowed_tools: Optional[list[str]] = typer.Option(None, "--allowed-tools", ...),
    disallowed_tools: Optional[list[str]] = typer.Option(None, "--disallowed-tools", ...),
    # --- System & Context ---
    system_prompt: str | None = typer.Option(None, "--system-prompt", "-s", ...),
    append_system_prompt: str | None = typer.Option(None, "--append-system-prompt", ...),
    settings_file: str | None = typer.Option(None, "--settings", ...),
    base_url: str | None = typer.Option(None, "--base-url", ...),
    api_key: str | None = typer.Option(None, "--api-key", "-k", ...),
    bare: bool = typer.Option(False, "--bare", ...),
    api_format: str | None = typer.Option(None, "--api-format", ...),
    theme: str | None = typer.Option(None, "--theme", ...),
    # --- Advanced ---
    debug: bool = typer.Option(False, "--debug", "-d", ...),
    mcp_config: Optional[list[str]] = typer.Option(None, "--mcp-config", ...),
    cwd: str = typer.Option(str(Path.cwd()), "--cwd", ...),
    backend_only: bool = typer.Option(False, "--backend-only", hidden=True),
    task_worker: bool = typer.Option(False, "--task-worker", hidden=True),
) -> None:
    """Start an interactive session or run a single prompt."""
```

这个 `main()` 函数是 `oh` 命令的**核心决策点**。

### 2.2 执行路径分支

当用户运行 `oh` 时，根据不同的选项进行路由：

#### 2.2.1 首先检查：是否是子命令

```python
if ctx.invoked_subcommand is not None:
    return  # 交由子命令处理（mcp, plugin, auth 等）
```

用户运行 `oh mcp list` 或 `oh auth setup` 时，会在这里返回。

#### 2.2.2 处理会话恢复

```python
if continue_session or resume is not None:
    # 从存储加载历史会话
    session_data = load_session_snapshot(cwd)  # 或 load_session_by_id()
    asyncio.run(
        run_repl(
            prompt=None,
            cwd=cwd,
            model=session_data.get("model") or model,
            backend_only=backend_only,
            restore_messages=session_data.get("messages"),
            restore_tool_metadata=session_data.get("tool_metadata"),
            permission_mode=permission_mode,
            api_format=api_format,
        )
    )
    return
```

- `--continue`: 恢复当前目录的最后一个会话
- `--resume <id>`: 恢复指定的会话 ID

#### 2.2.3 打印模式（非交互）

```python
if print_mode is not None:
    prompt = print_mode.strip()
    asyncio.run(
        run_print_mode(
            prompt=prompt,
            output_format=output_format or "text",
            cwd=cwd,
            model=model,
            base_url=base_url,
            system_prompt=system_prompt,
            api_key=api_key,
            api_format=api_format,
            permission_mode=permission_mode,
            max_turns=max_turns,
        )
    )
    return
```

- `oh -p "提示词"`: 执行单次提示，打印结果，退出

#### 2.2.4 任务工作者模式

```python
if task_worker:
    asyncio.run(
        run_task_worker(
            cwd=cwd,
            model=model,
            max_turns=max_turns,
            ...
        )
    )
    return
```

- 用于后台任务或子进程代理模式

#### 2.2.5 TUI 交互模式（最常见） ⭐

```python
asyncio.run(
    run_repl(
        prompt=None,
        cwd=cwd,
        model=model,
        max_turns=max_turns,
        backend_only=backend_only,
        base_url=base_url,
        system_prompt=system_prompt,
        api_key=api_key,
        api_format=api_format,
        permission_mode=permission_mode,
    )
)
```

- **用户只需输入 `oh`，就会执行这段代码**
- 所有参数都是可选的，默认值取自配置文件

### 2.3 run_repl() 函数

**文件**: `src/openharness/ui/app.py`

这是 TUI 启动的**直接调用者**：

```python
async def run_repl(
    *,
    prompt: str | None = None,
    cwd: str | None = None,
    model: str | None = None,
    max_turns: int | None = None,
    base_url: str | None = None,
    system_prompt: str | None = None,
    api_key: str | None = None,
    api_format: str | None = None,
    api_client: SupportsStreamingMessages | None = None,
    backend_only: bool = False,
    restore_messages: list[dict] | None = None,
    restore_tool_metadata: dict[str, object] | None = None,
    permission_mode: str | None = None,
) -> None:
    """Run the default OpenHarness interactive application (React TUI)."""
    if backend_only:
        await run_backend_host(...)  # 后端模式
        return

    exit_code = await launch_react_tui(
        prompt=prompt,
        cwd=cwd,
        model=model,
        max_turns=max_turns,
        base_url=base_url,
        system_prompt=system_prompt,
        api_key=api_key,
        api_format=api_format,
        permission_mode=permission_mode,
    )
    if exit_code != 0:
        raise SystemExit(exit_code)
```

**关键点**: `run_repl()` 决策是否以 **后端模式** 还是 **TUI 模式** 运行：
- `backend_only=True`: 启动后端 Host（供前端调用）
- `backend_only=False`: 启动完整的 React TUI（这是默认行为）

### 2.4 launch_react_tui() 函数

**文件**: `src/openharness/ui/react_launcher.py`

这就是 **`oh` 命令启动 TUI 的核心**：

```python
async def launch_react_tui(
    *,
    prompt: str | None = None,
    cwd: str | None = None,
    model: str | None = None,
    max_turns: int | None = None,
    base_url: str | None = None,
    system_prompt: str | None = None,
    api_key: str | None = None,
    api_format: str | None = None,
    permission_mode: str | None = None,
) -> int:
    """Launch the React terminal frontend as the default UI."""
```

#### 步骤 1: 定位前端目录

```python
frontend_dir = get_frontend_dir()
package_json = frontend_dir / "package.json"
if not package_json.exists():
    raise RuntimeError(f"React terminal frontend is missing: {package_json}")
```

前端位置优先级：
1. 打包版本: `openharness/_frontend/`（pip 安装）
2. 开发版本: `frontend/terminal/`（源代码结构）

#### 步骤 2: 自动安装前端依赖

```python
npm = _resolve_npm()
if not (frontend_dir / "node_modules").exists():
    install = await asyncio.create_subprocess_exec(
        npm,
        "install",
        "--no-fund",
        "--no-audit",
        cwd=str(frontend_dir),
    )
    if await install.wait() != 0:
        raise RuntimeError("Failed to install React terminal frontend dependencies")
```

**首次运行时**会自动执行 `npm install`（这会比较慢，20-30s）。

#### 步骤 3: 构建后端启动命令

```python
env = os.environ.copy()
env["OPENHARNESS_FRONTEND_CONFIG"] = json.dumps(
    {
        "backend_command": build_backend_command(
            cwd=cwd or str(Path.cwd()),
            model=model,
            max_turns=max_turns,
            base_url=base_url,
            system_prompt=system_prompt,
            api_key=api_key,
            api_format=api_format,
            permission_mode=permission_mode,
        ),
        "initial_prompt": prompt,
        "theme": _resolve_theme(),
    }
)
```

**关键**: 通过环境变量 `OPENHARNESS_FRONTEND_CONFIG` 传递：
- **backend_command**: React 前端如何启动 Python 后端
  ```
  python -m openharness --backend-only --cwd ... --model ... [其他参数]
  ```
- **initial_prompt**: 初始提示词（TUI 模式下为 null）
- **theme**: UI 主题

#### 步骤 4: 启动 React 前端进程

```python
tsx_cmd = _resolve_tsx(frontend_dir)
process = await asyncio.create_subprocess_exec(
    *tsx_cmd,
    "src/index.tsx",
    cwd=str(frontend_dir),
    env=env,
    stdin=None,
    stdout=None,
    stderr=None,
)
return await process.wait()
```

- 使用 `tsx` 直接执行 TypeScript（无编译）
- 调用 `frontend/terminal/src/index.tsx` 的主函数
- 等待进程完成，返回退出码

---

## 2. ohmo 命令启动 TUI 的流程

### 比较：oh vs ohmo

| 方面 | oh | ohmo |
|------|----|----|
| 入口 | `src/openharness/cli.py` | `ohmo/cli.py` |
| 主回调 | `main()` | `main()` |
| TUI 启动函数 | `launch_react_tui()` | `launch_ohmo_react_tui()` |
| 工作区 | 无工作区（每次启动独立） | 有工作区（`~/.ohmo`，持久化） |
| 功能 | 核心 Agent 引擎 | 个人助手（包括 persona、session 恢复等） |
| 后端初始化 | 基础配置 | 增加 ohmo 工作区语义 |

### ohmo 的启动不同之处

**文件**: `ohmo/runtime.py`

```python
async def launch_ohmo_react_tui(...) -> int:
    """Launch the shared React terminal UI with an ohmo backend."""
    # ... 前面的步骤和 oh 相同 ...
    
    env["OPENHARNESS_FRONTEND_CONFIG"] = json.dumps(
        {
            "backend_command": build_ohmo_backend_command(  # ← 关键区别！
                cwd=cwd_path,
                workspace=workspace_root,  # ← 工作区路径
                model=model,
                max_turns=max_turns,
                provider_profile=profile,
            ),
            "initial_prompt": None,
            "theme": "default",
        }
    )
```

**ohmo 特有的**：
- `build_ohmo_backend_command()` 返回 `python -m ohmo --backend-only [args]`
- 包含 `--workspace` 参数，指向 `~/.ohmo`

---

## 3. 两条 TUI 启动路径的详细对比

当用户运行 `ohmo` 或 `oh` 时，会进入这个决策点：

```python
@app.callback(invoke_without_command=True)
def main(
    ctx: typer.Context,
    print_mode: str | None = typer.Option(None, "--print", "-p", ...),
    model: str | None = typer.Option(None, "--model", ...),
    profile: str | None = typer.Option(None, "--profile", ...),
    workspace: str | None = typer.Option(None, "--workspace", ...),
    max_turns: int | None = typer.Option(None, "--max-turns", ...),
    cwd: str = typer.Option(str(Path.cwd()), "--cwd", ...),
    backend_only: bool = typer.Option(False, "--backend-only", hidden=True),
    resume: str | None = typer.Option(None, "--resume", ...),
    continue_session: bool = typer.Option(False, "--continue", ...),
) -> None:
```

### 2.1 执行路径选择

三条执行路径（按优先级）：

#### 路径 A: 子命令模式
```python
if ctx.invoked_subcommand is not None:
    return  # 交由子命令处理
```
- 用户运行 `ohmo init`, `ohmo config`, `ohmo gateway start` 等
- 不启动 TUI

#### 路径 B: 后端模式（给 React 前端用）
```python
if backend_only:
    raise SystemExit(
        asyncio.run(
            run_ohmo_backend(...)
        )
    )
```
- 用户运行 `ohmo --backend-only` （通常不直接用）
- React 前端会子进程启动这个模式
- 不显示 TUI

#### 路径 C: 打印模式（非交互）
```python
if print_mode is not None:
    raise SystemExit(
        asyncio.run(
            run_ohmo_print_mode(
                prompt=print_mode,  # 用户输入的文本
                cwd=cwd_path,
                workspace=workspace_root,
                model=model,
                max_turns=max_turns,
                provider_profile=profile,
            )
        )
    )
```
- 用户运行 `ohmo -p "你好"` 或 `ohmo --print "write hello world"`
- 在标准输出输出结果，然后退出

#### 路径 D: TUI 模式 ⭐ （最常见）
```python
raise SystemExit(
    asyncio.run(
        launch_ohmo_react_tui(
            cwd=cwd_path,
            workspace=workspace_root,
            model=model,
            max_turns=max_turns,
            provider_profile=profile,
        )
    )
)
```
- 用户直接运行 `ohmo` 或 `oh`，无额外参数
- **这是本分析的焦点**

## 3. TUI 启动的关键函数

### 3.1 launch_ohmo_react_tui()

**文件**: `ohmo/runtime.py`

这是启动 React TUI 的主函数：

```python
async def launch_ohmo_react_tui(
    *,
    cwd: str | None = None,
    workspace: str | Path | None = None,
    model: str | None = None,
    max_turns: int | None = None,
    provider_profile: str | None = None,
) -> int:
    """Launch the shared React terminal UI with an ohmo backend."""
```

#### 步骤 1: 检查前端文件

```python
frontend_dir = get_frontend_dir()
package_json = frontend_dir / "package.json"
if not package_json.exists():
    raise RuntimeError(f"React terminal frontend is missing: {package_json}")
```

- 前端在 `frontend/terminal/` 目录
- 如果是打包版本，在 `openharness/_frontend/`

#### 步骤 2: 安装前端依赖

```python
npm = _resolve_npm()
if not (frontend_dir / "node_modules").exists():
    install = await asyncio.create_subprocess_exec(
        npm,
        "install",
        "--no-fund",
        "--no-audit",
        cwd=str(frontend_dir),
    )
    if await install.wait() != 0:
        raise RuntimeError("Failed to install React terminal frontend dependencies")
```

- 如果第一次运行，自动执行 `npm install`
- 这确保了 React 和 Ink 依赖都已安装

#### 步骤 3: 构建前端启动命令

```python
cwd_path = str(Path(cwd or Path.cwd()).resolve())
workspace_root = initialize_workspace(workspace)
env = os.environ.copy()
env["OPENHARNESS_FRONTEND_CONFIG"] = json.dumps(
    {
        "backend_command": build_ohmo_backend_command(
            cwd=cwd_path,
            workspace=workspace_root,
            model=model,
            max_turns=max_turns,
            provider_profile=provider_profile,
        ),
        "initial_prompt": None,
        "theme": "default",
    }
)
```

- **关键**: 通过环境变量 `OPENHARNESS_FRONTEND_CONFIG` 告诉 React 前端如何启动后端
- 后端命令是 `python -m ohmo --backend-only [args]`

#### 步骤 4: 启动 React 前端进程

```python
tsx_cmd = _resolve_tsx(frontend_dir)
process = await asyncio.create_subprocess_exec(
    *tsx_cmd,
    "src/index.tsx",
    cwd=str(frontend_dir),
    env=env,
    stdin=None,
    stdout=None,
    stderr=None,
)
return await process.wait()
```

- 使用 `tsx` 直接执行 TypeScript 文件（不编译）
- 调用 `frontend/terminal/src/index.tsx` 作为主入口
- 等待进程退出，返回状态码

## 4. React 前端启动过程

### 4.1 前端入口

**文件**: `frontend/terminal/src/index.tsx`

前端是一个 React Ink 应用：
- 读取环境变量 `OPENHARNESS_FRONTEND_CONFIG`
- 从中提取后端启动命令和配置
- 启动后端子进程
- 建立 JSON-lines 通信管道

### 4.2 关键配置项

从 `OPENHARNESS_FRONTEND_CONFIG` 环境变量：

```json
{
  "backend_command": ["python", "-m", "ohmo", "--backend-only", "--cwd", "...", "--workspace", "..."],
  "initial_prompt": null,
  "theme": "default"
}
```

- **backend_command**: 前端如何启动后端（子进程）
- **initial_prompt**: 是否有初始提示词（TUI 模式下为 null）
- **theme**: UI 主题

## 5. 后端启动过程

### 5.1 run_ohmo_backend()

**文件**: `ohmo/runtime.py`

当前端启动后端子进程时：

```python
async def run_ohmo_backend(
    *,
    cwd: str | None = None,
    workspace: str | Path | None = None,
    model: str | None = None,
    max_turns: int | None = None,
    provider_profile: str | None = None,
    api_client: SupportsStreamingMessages | None = None,
    restore_messages: list[dict] | None = None,
    restore_tool_metadata: dict[str, object] | None = None,
    backend_only: bool = True,
) -> int:
    """Run the shared React backend host with ohmo workspace semantics."""
```

#### 步骤 1: 初始化工作区

```python
cwd_path = str(Path(cwd or Path.cwd()).resolve())
workspace_root = initialize_workspace(workspace)
extra_skill_dirs, extra_plugin_roots = _ohmo_extra_roots(workspace_root)
```

#### 步骤 2: 启动后端 Host

```python
return await run_backend_host(
    cwd=cwd_path,
    model=model,
    max_turns=max_turns,
    system_prompt=build_ohmo_system_prompt(cwd_path, workspace=workspace_root),
    active_profile=provider_profile,
    api_client=api_client,
    restore_messages=restore_messages,
    restore_tool_metadata=restore_tool_metadata,
    enforce_max_turns=max_turns is not None,
    session_backend=OhmoSessionBackend(workspace_root),
    extra_skill_dirs=extra_skill_dirs,
    extra_plugin_roots=extra_plugin_roots,
)
```

- 调用 `src/openharness/ui/backend_host.py` 中的 `run_backend_host()`

### 5.2 ReactBackendHost

**文件**: `src/openharness/ui/backend_host.py`

```python
class ReactBackendHost:
    """Drive the OpenHarness runtime over a structured stdin/stdout protocol."""

    async def run(self) -> int:
        self._bundle = await build_runtime(...)
        await start_runtime(self._bundle)
        # 监听前端请求，处理消息循环
```

#### 主要职责：

1. **构建 RuntimeBundle**: 组装所有必要的运行时组件
2. **启动运行时**: 初始化引擎、权限检查器、MCP 管理器等
3. **消息循环**: 
   - 从前端 stdin 读取 JSON-lines 请求
   - 调用 OpenHarness 核心处理
   - 发送结果回到前端 stdout

## 6. 完整启动流程图

```
┌─────────────────────────────────────────────────────────────────┐
│ 用户在终端输入: oh 或 ohmo                                        │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│ Python Entry: src/openharness/cli.py 或 ohmo/cli.py             │
│ (typer app dispatches to main callback)                         │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│ ohmo/cli.py::main() - 模式决策                                    │
│                                                                  │
│ ├─ 是否指定了子命令? ──Yes──▶ 执行子命令（不启动TUI）           │
│ ├─ --backend-only? ──Yes──▶ 启动后端 Host（前端调用）           │
│ ├─ --print? ──Yes──▶ 打印模式（非交互，单次）                   │
│ └─ None of above ──▶ 启动 TUI (launch_ohmo_react_tui) ⭐         │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼ (TUI 路径)
┌─────────────────────────────────────────────────────────────────┐
│ ohmo/runtime.py::launch_ohmo_react_tui()                         │
│                                                                  │
│ 1. 定位前端目录 (frontend/terminal/)                             │
│ 2. npm install (if needed)                                      │
│ 3. 构建后端命令                                                  │
│    └─▶ python -m ohmo --backend-only [args]                     │
│ 4. 通过环境变量传递配置                                          │
│    └─▶ OPENHARNESS_FRONTEND_CONFIG=<json>                       │
│ 5. 启动 tsx src/index.tsx                                       │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│ React Ink App (frontend/terminal/src/index.tsx)                  │
│                                                                  │
│ 1. 读取 OPENHARNESS_FRONTEND_CONFIG                              │
│ 2. 启动后端子进程 (--backend-only)                                │
│ 3. 建立 JSON-lines 通信管道 (stdin/stdout)                       │
│ 4. 渲染交互式 TUI                                                │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼ (后端进程)
┌─────────────────────────────────────────────────────────────────┐
│ ohmo/runtime.py::run_ohmo_backend()                              │
│ →  src/openharness/ui/backend_host.py::run_backend_host()       │
│                                                                  │
│ ReactBackendHost:                                                │
│ 1. 构建 RuntimeBundle                                            │
│    ├─ 加载 MCP servers                                          │
│    ├─ 初始化 ToolRegistry                                       │
│    ├─ 加载 Hooks/Plugins/Skills                                 │
│    ├─ 创建 QueryEngine                                          │
│    └─ 设置权限检查器                                             │
│ 2. 启动运行时                                                    │
│ 3. 循环处理前端请求                                              │
│    ├─ 读取用户消息                                               │
│    ├─ 调用 QueryEngine 处理                                      │
│    ├─ 流式发送结果事件                                           │
│    └─ 处理工具执行、权限提示等                                   │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│ 前端接收和渲染事件，显示交互式 TUI                                │
│                                                                  │
│ ├─ 显示用户输入框                                                │
│ ├─ 流式展示 AI 响应 (AssistantTextDelta)                        │
│ ├─ 显示工具执行进度                                              │
│ ├─ 处理用户交互 (input, shortcuts, etc)                         │
│ └─ 更新状态显示                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 7. 数据流：前端 ↔ 后端

### 7.1 通信协议

使用 JSON-lines 格式在 stdin/stdout 上：

**前端 → 后端** (请求):
```json
{"type": "user_message", "text": "用户输入的文本"}
{"type": "permission_response", "granted": true}
```

**后端 → 前端** (事件):
```json
{"type": "assistant_text_delta", "text": "流式响应..."}
{"type": "tool_execution_started", "tool_name": "..."}
{"type": "tool_execution_completed", "tool_name": "...", "result": "..."}
{"type": "assistant_turn_complete"}
```

### 7.2 关键类

**前端请求**: `src/openharness/ui/protocol.py`
```python
class FrontendRequest(BaseModel):
    type: str  # "user_message", "permission_response", "question_response"
    ...
```

**后端事件**: `src/openharness/engine/stream_events.py`
```python
class StreamEvent(BaseModel):
    type: str
    ...
```

## 8. 关键环境变量和配置

| 变量 | 来源 | 用途 |
|------|------|------|
| `OPENHARNESS_FRONTEND_CONFIG` | Python 设置 | 前端读取后端启动命令和配置 |
| `HOME` 或 `USERPROFILE` | 系统 | 确定 ~/.ohmo 工作区位置 |
| OpenHarness 全局配置 | `~/.openharness/` | provider 认证、MCP 配置等 |
| ohmo 工作区配置 | `~/.ohmo/` | persona、session、gateway 配置 |

## 9. 启动时间关键路径

| 步骤 | 函数 | 文件 | 耗时因素 |
|------|------|------|---------|
| CLI 解析 | `typer.Typer.callback()` | `ohmo/cli.py` | 快速 |
| 工作区初始化 | `initialize_workspace()` | `ohmo/workspace.py` | 快速 |
| npm install | `npm install` | - | **首次较慢** (~10-30s) |
| 前端启动 | `tsx src/index.tsx` | `frontend/terminal/` | 快速 (~1-2s) |
| RuntimeBundle 构建 | `build_runtime()` | `src/openharness/ui/runtime.py` | 中等 (~2-5s) |
| 权限检查、MCP 连接 | `startup hooks` | - | 取决于配置 |
| 显示 TUI | Ink 首次渲染 | React Ink | 快速 (~0.5s) |

## 10. 常见启动选项组合

| 命令 | 行为 | 场景 |
|------|------|------|
| `ohmo` | 启动 TUI | 交互式聊天 |
| `ohmo -p "你好"` | 单次打印模式 | 脚本集成 |
| `ohmo --continue` | 恢复上一个 session | 继续对话 |
| `ohmo --resume <id>` | 恢复指定 session | 历史 session 恢复 |
| `ohmo --model gpt-4` | 切换模型 | 模型尝试 |
| `ohmo --profile openai-compatible` | 使用不同的 provider | 多厂商切换 |
| `ohmo init` | 初始化工作区 | 首次设置 |
| `ohmo config` | 重新配置 | 修改设置 |

## 总结

OpenHarness 的 TUI 启动是一个**分层启动**过程：

1. **Python CLI 层** (typer): 接收命令行参数，决策执行模式
2. **运行时组装层** (build_runtime): 组装 Agent 核心组件
3. **前后端桥接层**: 
   - 前端：React Ink (TypeScript/Node.js)
   - 后端：Python asyncio (JSON-lines 通信)
4. **Agent 核心** (QueryEngine): 处理消息、调用工具、执行循环

这个架构的好处：
- **模式灵活**: 同一套 Agent 核心支持 CLI、TUI、API、IM gateway
- **前后端分离**: React 前端可独立升级，后端是语言无关的 JSON 协议
- **会话持久化**: 可以暂停、恢复、迁移会话

