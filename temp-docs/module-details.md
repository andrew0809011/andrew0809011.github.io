# OpenHarness 模块详解

## 目录结构

```
OpenHarness/
├── src/openharness/           # 核心框架源代码
│   ├── api/                   # API 客户端层
│   ├── auth/                  # 认证管理
│   ├── bridge/                # 桥接模式
│   ├── channels/              # 消息渠道
│   ├── commands/              # CLI 命令注册
│   ├── config/                # 配置系统
│   ├── coordinator/           # 协调器模式
│   ├── engine/                # 查询引擎核心
│   ├── hooks/                 # 钩子系统
│   ├── keybindings/           # 键盘快捷键
│   ├── mcp/                   # MCP 协议实现
│   ├── memory/                # 记忆系统
│   ├── output_styles/         # 输出样式
│   ├── permissions/           # 权限控制
│   ├── personalization/       # 个性化
│   ├── plugins/               # 插件系统
│   ├── prompts/               # 提示词管理
│   ├── sandbox/               # 沙箱环境
│   ├── services/              # 后台服务
│   ├── skills/                # 技能系统
│   ├── state/                 # 应用状态
│   ├── swarm/                 # 多 Agent 协调
│   ├── tasks/                 # 任务管理
│   ├── themes/                # 主题系统
│   ├── tools/                 # 工具实现
│   ├── ui/                    # UI 层
│   └── utils/                 # 工具函数
├── ohmo/                      # ohmo 个人 Agent 应用
├── frontend/terminal/         # React TUI 前端
├── tests/                     # 测试套件
├── docs/                      # 文档
├── scripts/                   # 安装脚本
└── .claude/skills/            # Claude 技能定义
```

---

## 核心模块详述

### 1. API 客户端层 (`src/openharness/api/`)

#### 1.1 client.py - Anthropic 客户端

```python
class AnthropicApiClient:
    """Anthropic API 客户端包装器"""
    
    # 功能:
    - 指数退避重试 (MAX_RETRIES=3)
    - OAuth 认证支持
    - 流式消息处理
    - Token 使用量追踪
```

**关键类**:
- `ApiMessageRequest`: API 请求参数
- `ApiTextDeltaEvent`: 文本增量事件
- `ApiMessageCompleteEvent`: 消息完成事件
- `SupportsStreamingMessages`: 流式消息协议

#### 1.2 openai_client.py - OpenAI 兼容客户端

支持 OpenAI 兼容 API (包括 OpenRouter、本地模型等)。

#### 1.3 copilot_client.py - GitHub Copilot

支持 GitHub Copilot 代理模式。

#### 1.4 provider.py & registry.py

Provider 注册和管理，支持运行时切换 Provider。

---

### 2. 查询引擎 (`src/openharness/engine/`)

#### 2.1 query_engine.py

**QueryEngine 类属性**:

```python
class QueryEngine:
    _api_client: SupportsStreamingMessages  # API 客户端
    _tool_registry: ToolRegistry            # 工具注册表
    _permission_checker: PermissionChecker  # 权限检查器
    _messages: list[ConversationMessage]    # 对话历史
    _cost_tracker: CostTracker              # Token 追踪
    _max_turns: int | None                  # 最大轮数限制
```

**核心流程**:

```
submit_message(user_prompt)
    │
    ├── 构建 QueryContext
    │
    └── run_query(context, messages)
            │
            ├── 检查 Token 阈值 → 自动压缩历史
            │
            └── 调用 API
                    │
                    ├── Assistant 返回文本 → 直接输出
                    │
                    └── Assistant 返回 tool_use
                            │
                            ├── 权限检查
                            ├── 执行工具
                            ├── 构造 tool_result
                            └── 递归调用 (下一turn)
```

#### 2.2 query.py

核心查询执行逻辑，包括：
- 上下文窗口管理
- 自动压缩 (auto-compact)
- 工具结果收集

#### 2.3 messages.py

对话消息类型定义：
- `ConversationMessage`: 通用消息
- `TextBlock`: 文本块
- `ToolUseBlock`: 工具调用块
- `ToolResultBlock`: 工具结果块

#### 2.4 stream_events.py

流事件类型：
- `AssistantTextDelta`: 文本增量
- `AssistantTurnComplete`: Turn 完成
- `ToolExecutionStarted/Completed`: 工具执行事件
- `ErrorEvent`: 错误事件
- `StatusEvent`: 状态事件
- `CompactProgressEvent`: 压缩进度

#### 2.5 cost_tracker.py

Token 使用追踪和成本估算。

---

### 3. 工具系统 (`src/openharness/tools/`)

#### 3.1 base.py - 工具基类

```python
@dataclass
class ToolExecutionContext:
    cwd: Path                      # 当前工作目录
    metadata: dict[str, Any]       # 元数据传递

@dataclass(frozen=True)
class ToolResult:
    output: str                    # 输出内容
    is_error: bool                 # 是否错误
    metadata: dict[str, Any]       # 结果元数据

class BaseTool(ABC):
    name: str                      # 工具名 (snake_case)
    description: str               # 描述 (给 LLM 看)
    input_model: type[BaseModel]   # Pydantic 输入模型
    
    @abstractmethod
    async def execute(self, arguments: BaseModel, context: ToolExecutionContext) -> ToolResult
    
    def is_read_only(self, arguments: BaseModel) -> bool
        # 返回 True 表示只读工具，不需要权限确认
    
    def to_api_schema(self) -> dict[str, Any]
        # 转换为 Anthropic API 工具格式

class ToolRegistry:
    # 工具名 → 工具实例的映射
```

#### 3.2 文件操作工具

**file_read_tool.py**:
- 支持 `offset` 和 `limit` 分页
- 自动检测大文件并截断

**file_write_tool.py**:
- 写入文件内容
- 自动创建目录

**file_edit_tool.py**:
- 基于字符串替换的编辑
- 支持 `replace_all` 批量替换

**glob_tool.py**:
- 文件搜索 (通配符)
- 支持 `root` 指定搜索根目录

#### 3.3 代码搜索工具

**grep_tool.py**:
- 正则表达式搜索
- 支持 `file_glob` 文件过滤
- 多线程并行搜索

**lsp_tool.py**:
- LSP 语言服务器集成
- 支持 `document_symbol`, `go_to_definition`, `find_references`, `hover`

#### 3.4 终端工具

**bash_tool.py**:
- 执行 Shell 命令
- 可配置超时 (默认 600s)
- 工作目录控制

#### 3.5 Agent 工具

**agent_tool.py**:
- 创建子 Agent 任务
- 支持 `subagent_type` 指定 Agent 类型

#### 3.6 任务工具

| 工具 | 功能 |
|------|------|
| `task_create_tool` | 创建后台任务 |
| `task_list_tool` | 列出所有任务 |
| `task_get_tool` | 获取任务详情 |
| `task_stop_tool` | 停止任务 |
| `task_output_tool` | 获取任务输出 |
| `task_update_tool` | 更新任务状态 |

#### 3.7 团队工具

| 工具 | 功能 |
|------|------|
| `team_create_tool` | 创建 Agent 团队 |
| `team_delete_tool` | 删除团队 |
| `send_message_tool` | 向 Agent 发送消息 |

#### 3.8 MCP 工具

| 工具 | 功能 |
|------|------|
| `mcp_tool` | 调用 MCP 服务器工具 |
| `mcp_auth_tool` | 配置 MCP 认证 |
| `list_mcp_resources_tool` | 列出 MCP 资源 |
| `read_mcp_resource_tool` | 读取 MCP 资源 |

#### 3.9 Cron 工具

| 工具 | 功能 |
|------|------|
| `cron_create_tool` | 创建定时任务 |
| `cron_list_tool` | 列出 Cron 任务 |
| `cron_delete_tool` | 删除 Cron 任务 |
| `cron_toggle_tool` | 启用/禁用 Cron 任务 |

---

### 4. 权限系统 (`src/openharness/permissions/`)

#### 4.1 modes.py

```python
class PermissionMode(str, Enum):
    FULL_AUTO = "full_auto"    # 全自动，无需确认
    DEFAULT = "default"         # 默认，敏感操作需确认
    PLAN = "plan"               # 计划模式，阻止修改
```

#### 4.2 checker.py

**PermissionChecker**:

评估流程:
1. 敏感路径检查 (内置规则，不可覆盖)
2. 工具黑名单检查
3. 工具白名单检查
4. 路径规则检查
5. 命令模式检查
6. 权限模式判断

**内置敏感路径**:
```python
SENSITIVE_PATH_PATTERNS = (
    "*/.ssh/*",              # SSH keys
    "*/.aws/credentials",     # AWS 凭据
    "*/.config/gcloud/*",     # GCP 凭据
    "*/.azure/*",             # Azure 凭据
    "*/.gnupg/*",             # GPG keys
    "*/.docker/config.json",  # Docker 凭据
    "*/.kube/config",         # Kubernetes 凭据
    "*/.openharness/credentials.json",
)
```

---

### 5. 记忆系统 (`src/openharness/memory/`)

#### 5.1 types.py

```python
@dataclass(frozen=True)
class MemoryHeader:
    path: Path           # 文件路径
    title: str           # 标题
    description: str     # 描述
    modified_at: float   # 修改时间
    memory_type: str     # 类型
    body_preview: str    # 内容预览
```

#### 5.2 manager.py

**MemoryManager** 功能:
- 记忆文件的 CRUD
- 记忆内容搜索

#### 5.3 scan.py

**MemoryScan**:
- 递归扫描项目目录
- 发现 `CLAUDE.md` 文件
- 发现 `MEMORY.md` 文件

#### 5.4 memdir.py

持久化记忆目录管理，每个项目在 `~/.openharness/memory/<project_id>/` 下有独立的 MEMORY.md。

---

### 6. Swarm 多 Agent 系统 (`src/openharness/swarm/`)

#### 6.1 types.py

后端类型:
```python
BackendType = Literal["subprocess", "in_process", "tmux", "iterm2"]
```

#### 6.2 subprocess_backend.py

子进程 Agent 实现:
- 通过 stdin/stdout 通信
- 支持 poll 查询状态
- 权限同步

#### 6.3 in_process.py

进程内 Agent (轻量级)。

#### 6.4 mailbox.py

Agent 间消息传递机制。

#### 6.5 team_lifecycle.py

团队生命周期管理:
- 创建团队
- 添加成员
- 解散团队

#### 6.6 worktree.py

Git worktree 管理，为每个 Agent 提供隔离的工作目录。

---

### 7. 任务系统 (`src/openharness/tasks/`)

#### 7.1 types.py

```python
TaskType = Literal["local_bash", "local_agent", "remote_agent", "in_process_teammate"]
TaskStatus = Literal["pending", "running", "completed", "failed", "killed"]

@dataclass
class TaskRecord:
    id: str
    type: TaskType
    status: TaskStatus
    description: str
    cwd: str
    output_file: Path
    # ... 其他字段
```

#### 7.2 manager.py

**BackgroundTaskManager**:
- 任务创建和调度
- 任务状态监控
- 任务输出收集

#### 7.3 local_shell_task.py & local_agent_task.py

具体任务执行实现。

---

### 8. MCP 协议实现 (`src/openharness/mcp/`)

#### 8.1 client.py

**MCPClient**:
- 支持 stdio 和 HTTP 传输
- 自动重连机制
- 工具发现
- 资源访问

#### 8.2 config.py

MCP 服务器配置管理。

---

### 9. 配置系统 (`src/openharness/config/`)

#### 9.1 settings.py

**Settings** 类 (Pydantic BaseModel):
- 全局配置
- Provider 设置
- MCP 服务器列表
- 权限设置

#### 9.2 paths.py

配置路径管理:
- `~/.openharness/config.json`
- `~/.openharness/credentials.json`

#### 9.3 schema.py

渠道配置模型 (供 ohmo 网关使用)。

---

### 10. UI 层 (`src/openharness/ui/`)

#### 10.1 app.py

**运行模式**:
- `run_repl()`: 交互式 REPL (React TUI)
- `run_task_worker()`: 后台任务工作器
- `run_print_mode()`: 非交互式打印模式

#### 10.2 runtime.py

运行时构建和管理:
- `build_runtime()`: 构建运行时
- `start_runtime()`: 启动运行时
- `handle_line()`: 处理用户输入行
- `close_runtime()`: 清理资源

#### 10.3 react_launcher.py

React TUI 启动器:
- 编译 TypeScript
- 启动 Node.js 进程
- 进程间通信

#### 10.4 backend_host.py

后端主机模式 (纯 Python，无 TUI)。

---

### 11. CLI (`src/openharness/cli.py`)

**命令分组**:

```python
app = typer.Typer()

# MCP 管理
mcp_app = typer.Typer(name="mcp")
- mcp list      # 列出 MCP 服务器
- mcp add       # 添加 MCP 服务器
- mcp remove    # 移除 MCP 服务器

# 插件管理
plugin_app = typer.Typer(name="plugin")
- plugin list       # 列出插件
- plugin install    # 安装插件
- plugin uninstall  # 卸载插件

# Cron 管理
cron_app = typer.Typer(name="cron")
- cron start    # 启动调度器
- cron stop     # 停止调度器
- cron status   # 查看状态
- cron list     # 列出任务

# 认证管理
auth_app = typer.Typer(name="auth")

# Provider 管理
provider_app = typer.Typer(name="provider")
```

---

### 12. ohmo 模块 (`ohmo/`)

#### 12.1 cli.py

ohmo CLI 入口，基于 typer。

#### 12.2 gateway/

**网关服务** - 多平台消息接入：
- `service.py`: 网关主服务
- `bridge.py`: 与 OpenHarness 桥接
- `router.py`: 消息路由
- `models.py`: 数据模型

#### 12.3 workspace.py

ohmo 工作区管理 (`~/.ohmo/`):
```python
get_workspace_root()    # ~/.ohmo
get_soul_path()         # ~/.ohmo/soul.md (Agent 人设)
get_user_path()         # ~/.ohmo/user.md (用户信息)
get_state_path()        # ~/.ohmo/state.json
get_gateway_config_path()  # ~/.ohmo/gateway.json
```

#### 12.4 session_storage.py

ohmo 会话存储后端。

#### 12.5 runtime.py

ohmo 运行时封装。

---

### 13. 前端模块 (`frontend/terminal/`)

#### 13.1 技术栈
- **Ink**: React for Terminal
- **TypeScript**: 类型安全
- **esbuild**: 快速编译

#### 13.2 核心组件

**app.tsx**:
- 主应用逻辑
- 键盘事件处理
- 命令解析
- 状态管理

**hooks/useBackendSession.ts**:
- 与 Python 后端通信
- JSON Lines 协议处理
- 消息收发

**components/**:
| 组件 | 功能 |
|------|------|
| `ConversationView` | 对话历史显示 |
| `PromptInput` | 用户输入框 |
| `CommandPicker` | `/` 命令补全 |
| `SelectModal` | 选择对话框 |
| `SwarmPanel` | Agent 团队显示 |
| `TodoPanel` | TODO 列表 |
| `StatusBar` | 状态栏 |

---

## 模块依赖图

```
cli.py
  │
  ├──► api/  ───────────►  anthropic, openai SDK
  │
  ├──► engine/  ────────►  api/, tools/, permissions/
  │
  ├──► ui/  ────────────►  engine/, tools/
  │
  ├──► swarm/  ─────────►  tasks/, tools/
  │
  ├──► tasks/  ─────────►  ui/ (runtime)
  │
  ├──► tools/  ─────────►  permissions/, memory/, mcp/
  │
  ├──► permissions/  ───►  config/
  │
  ├──► memory/  ────────►  config/
  │
  ├──► mcp/  ───────────►  config/
  │
  └──► config/  ────────►  (基础层)
```

---

## 配置项清单

### 全局配置 (`~/.openharness/config.json`)

| 配置项 | 类型 | 说明 |
|--------|------|------|
| `provider_profile` | str | 默认 Provider |
| `model` | str | 默认模型 |
| `max_tokens` | int | 最大 token 数 |
| `system_prompt` | str | 系统提示词 |
| `theme` | str | 主题名称 |
| `output_style` | str | 输出样式 |
| `mcp_servers` | dict | MCP 服务器配置 |
| `plugins` | list | 已安装插件 |
| `permission.mode` | str | 权限模式 |
| `permission.allowed_tools` | list | 允许的工具列表 |
| `permission.denied_tools` | list | 禁止的工具列表 |
| `permission.path_rules` | list | 路径规则 |

### 环境变量

| 变量 | 说明 |
|------|------|
| `ANTHROPIC_API_KEY` | Anthropic API 密钥 |
| `OPENAI_API_KEY` | OpenAI API 密钥 |
| `OPENAI_BASE_URL` | OpenAI 兼容端点 |
| `OPENHARNESS_DEBUG` | 调试模式 |
| `OPENHARNESS_HOME` | 配置目录覆盖 |
