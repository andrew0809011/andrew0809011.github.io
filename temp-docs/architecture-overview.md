# OpenHarness 架构总览

## 项目简介

**OpenHarness** 是一个开源的 AI Agent 基础设施框架，提供 Agent 运行时所需的核心能力：工具使用(Tool-use)、技能(Skills)、记忆(Memory)和多 Agent 协调(Multi-agent Coordination)。它是 Claude Code 的开源 Python 实现。

**ohmo** 是基于 OpenHarness 构建的个人 AI Agent 应用，支持通过 Feishu/Slack/Telegram/Discord 等平台进行交互，能够独立执行代码编写、测试运行、PR 创建等任务。

---

## 整体架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              用户交互层 (UI Layer)                            │
├─────────────────────────────────────────────────────────────────────────────┤
│  React TUI (Ink)    │    Textual TUI    │    CLI Print Mode    │   API     │
│  (frontend/terminal)│    (备选)          │    (非交互式)         │  (Backend)│
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              运行时层 (Runtime Layer)                         │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ QueryEngine │  │  Session    │  │   Bridge    │  │  Coordinator Mode   │  │
│  │   查询引擎   │  │   会话管理   │  │   桥接模式   │  │     协调器模式       │  │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              核心服务层 (Core Services)                       │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐          │
│  │   API    │ │   Auth   │ │  Memory  │ │   MCP    │ │   LSP    │          │
│  │  Client  │ │  Manager │ │  System  │ │  Client  │ │ Service  │          │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐                        │
│  │   Cron   │ │  Config  │ │   Task   │ │   Swarm  │                        │
│  │Scheduler │ │ Settings │ │  Manager │ │   Team   │                        │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              工具层 (Tools Layer)                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  File Ops │ Bash │ Search │ Web │ Agent │ Task │ Team │ Skill │ MCP │ ...  │
│  (文件操作)│(终端)│(搜索)  │(网络)│(代理) │(任务)│(团队)│(技能) │(MCP)│     │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              安全与治理层 (Governance)                        │
├─────────────────────────────────────────────────────────────────────────────┤
│  Permission Modes │ Path Rules │ Command Rules │ Pre/Post Hooks │ Sandboxing│
│    (权限模式)      │  (路径规则) │   (命令规则)   │   (钩子系统)    │ (沙箱)   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 关键组件详解

### 1. 查询引擎 (QueryEngine)

**路径**: `src/openharness/engine/query_engine.py`

QueryEngine 是 OpenHarness 的核心组件，负责管理与 LLM 的对话循环：

- **对话管理**: 维护消息历史 (`_messages`)
- **工具调用循环**: 处理 Assistant 的 Tool Use → 执行工具 → 返回结果给 Assistant
- **上下文压缩**: 自动压缩超过阈值的历史消息
- **Token 追踪**: 通过 `CostTracker` 追踪 API 使用量
- **Coordinator 集成**: 支持协调器模式下的上下文传递

**核心方法**:
- `submit_message()`: 提交用户消息并执行查询循环
- `continue_pending()`: 继续被中断的工具循环
- `has_pending_continuation()`: 检查是否有待处理的工具结果

### 2. 工具系统 (Tools)

**路径**: `src/openharness/tools/`

所有工具继承自 `BaseTool` 抽象基类：

```python
class BaseTool(ABC):
    name: str                    # 工具名称
    description: str             # 工具描述
    input_model: type[BaseModel] # Pydantic 输入模型
    
    @abstractmethod
    async def execute(self, arguments: BaseModel, context: ToolExecutionContext) -> ToolResult:
        """执行工具"""
```

**43+ 内置工具分类**:

| 类别 | 工具 | 说明 |
|------|------|------|
| **文件操作** | `file_read_tool`, `file_write_tool`, `file_edit_tool`, `glob_tool` | 文件读写和搜索 |
| **代码搜索** | `grep_tool`, `lsp_tool` | 文本搜索和代码智能 |
| **终端** | `bash_tool` | 执行 Shell 命令 |
| **Agent** | `agent_tool` | 创建子 Agent 任务 |
| **任务** | `task_create_tool`, `task_list_tool`, `task_get_tool`, `task_stop_tool` | 后台任务管理 |
| **团队** | `team_create_tool`, `team_delete_tool`, `send_message_tool` | 多 Agent 协调 |
| **技能** | `skill_tool` | 动态加载技能文件 |
| **MCP** | `mcp_tool`, `mcp_auth_tool`, `list_mcp_resources_tool` | MCP 协议支持 |
| **Cron** | `cron_create_tool`, `cron_list_tool`, `cron_delete_tool`, `cron_toggle_tool` | 定时任务 |
| **工作区** | `enter_worktree_tool`, `exit_worktree_tool` | Git worktree 管理 |
| **Web** | `web_search_tool`, `web_fetch_tool` | 网络搜索和获取 |
| **开发** | `notebook_edit_tool`, `todo_write_tool` | Jupyter/TODO 管理 |
| **配置** | `config_tool`, `enter_plan_mode_tool`, `exit_plan_mode_tool` | 配置和权限模式 |

### 3. API 客户端层

**路径**: `src/openharness/api/`

支持多种 LLM Provider：

| Provider | 客户端 | 说明 |
|----------|--------|------|
| Anthropic | `AnthropicApiClient` | Claude API，支持 OAuth |
| OpenAI | `OpenAIApiClient` | OpenAI 兼容 API |
| GitHub Copilot | `CopilotClient` | Copilot 代理模式 |
| Codex | `CodexClient` | OpenAI Codex |

**特性**:
- 指数退避重试机制
- 流式响应处理
- 内置 Token 使用追踪

### 4. 权限与治理系统

**路径**: `src/openharness/permissions/`

**权限模式** (`PermissionMode`):
- `FULL_AUTO`: 全自动模式，无需确认
- `DEFAULT`: 默认模式，敏感操作需确认
- `PLAN`: 计划模式，阻止所有修改操作

**安全检查层**:
1. **敏感路径保护**: 内置 SSH keys、AWS credentials 等路径黑名单
2. **路径规则**: Glob 模式匹配允许/拒绝路径
3. **命令规则**: 禁止危险命令 (如 `rm -rf /`)
4. **工具白名单/黑名单**: 显式允许/拒绝特定工具

### 5. 记忆系统 (Memory)

**路径**: `src/openharness/memory/`

- **CLAUDE.md 发现**: 自动检测并注入项目上下文
- **MEMORY.md 持久化**: 跨会话记忆存储
- **自动扫描**: `MemoryScan` 定期扫描记忆文件
- **搜索**: 支持记忆内容检索

### 6. Swarm 多 Agent 系统

**路径**: `src/openharness/swarm/`

支持多后端类型的 Agent 协调：

| 后端类型 | 说明 |
|----------|------|
| `subprocess` | 子进程后台 Agent |
| `in_process` | 进程内 Agent |
| `tmux` | Tmux 窗格可视化 |
| `iterm2` | iTerm2 窗格可视化 |

**核心组件**:
- `Mailbox`: Agent 间消息传递
- `TeamLifecycle`: 团队生命周期管理
- `WorktreeManager`: Git worktree 隔离

### 7. 任务系统 (Tasks)

**路径**: `src/openharness/tasks/`

支持的任务类型:
- `local_bash`: 本地 Shell 任务
- `local_agent`: 本地 Agent 任务
- `remote_agent`: 远程 Agent 任务
- `in_process_teammate`: 进程内队友

**状态管理**: pending → running → completed/failed/killed

### 8. MCP (Model Context Protocol)

**路径**: `src/openharness/mcp/`

- 支持 HTTP 和 stdio 传输
- 自动重连机制
- 工具和资源发现
- 认证管理

---

## ohmo 应用架构

**路径**: `ohmo/`

ohmo 是基于 OpenHarness 的个人 Agent 应用：

```
ohmo/
├── cli.py              # CLI 入口
├── gateway/            # 网关服务 (多平台消息接入)
│   ├── service.py      # 网关服务实现
│   ├── bridge.py       # 与 OpenHarness 桥接
│   └── models.py       # 数据模型
├── runtime.py          # ohmo 运行时
├── session_storage.py  # 会话存储
├── workspace.py        # 工作区管理
├── memory.py           # 记忆管理
└── prompts.py          # 提示词模板
```

**支持的接入渠道**:
- Telegram
- Slack
- Discord
- Feishu (飞书)
- DingTalk (钉钉)
- QQ
- Matrix
- WhatsApp
- Email

---

## 前端架构 (React TUI)

**路径**: `frontend/terminal/`

使用 **Ink** (React for Terminal) 构建的 TUI：

```
frontend/terminal/src/
├── app.tsx                  # 主应用组件
├── components/              # UI 组件
│   ├── CommandPicker.tsx    # 命令选择器
│   ├── ConversationView.tsx # 对话视图
│   ├── ModalHost.tsx        # 模态框容器
│   ├── PromptInput.tsx      # 输入框
│   ├── SelectModal.tsx      # 选择对话框
│   ├── StatusBar.tsx        # 状态栏
│   ├── SwarmPanel.tsx       # Swarm 面板
│   └── TodoPanel.tsx        # TODO 面板
├── hooks/                   # React Hooks
│   └── useBackendSession.ts # 后端会话管理
└── theme/                   # 主题系统
    └── ThemeContext.tsx
```

**通信协议**: JSON Lines over stdio

---

## 数据流向

```
用户输入
    │
    ▼
┌────────────────┐
│  UI (React)    │◄────── 渲染 Assistant 回复
└────────────────┘
    │
    │ JSON Protocol
    ▼
┌────────────────┐
│  Backend Host  │
└────────────────┘
    │
    ▼
┌────────────────┐
│  QueryEngine   │◄────── 管理对话历史
└────────────────┘
    │
    ▼ 工具调用
┌────────────────┐
│  ToolRegistry  │
└────────────────┘
    │
    ▼
┌────────────────┐
│  具体工具实现   │◄────── 权限检查
└────────────────┘
    │
    ▼
  执行结果
```

---

## 配置系统

**配置文件位置**: `~/.openharness/`

```
~/.openharness/
├── config.json              # 主配置
├── credentials.json         # 凭据存储
├── copilot_auth.json        # Copilot 认证
├── mcp_servers.json         # MCP 服务器配置
├── tasks/                   # 任务记录
└── memory/                  # 持久化记忆
    └── <project_id>/
        └── MEMORY.md
```

**ohmo 配置** (`~/.ohmo/`):
```
~/.ohmo/
├── soul.md              # Agent 人设
├── user.md              # 用户信息
├── state.json           # 运行状态
├── gateway.json         # 网关配置
└── channels/            # 渠道配置
```

---

## 扩展机制

### 1. 技能系统 (Skills)

**路径**: `src/openharness/skills/`

动态加载 `.md` 技能文件，支持：
- 内置技能 (bundled)
- 用户技能 (user-defined)
- 插件技能 (plugin-provided)

### 2. 插件系统 (Plugins)

**路径**: `src/openharness/plugins/`

支持 hooks、tools、commands 的插件扩展。

### 3. 钩子系统 (Hooks)

**路径**: `src/openharness/hooks/`

PreToolUse / PostToolUse 钩子，用于：
- 工具调用拦截
- 日志记录
- 自定义逻辑注入

---

## 测试策略

```
tests/
├── unit/              # 单元测试
├── integration/       # 集成测试
└── e2e/               # 端到端测试 (6 个测试套件)
```

**覆盖率**: 114+ 测试通过

---

## 技术栈

| 组件 | 技术 |
|------|------|
| 后端 | Python 3.10+, Pydantic, Typer |
| API 客户端 | Anthropic SDK, OpenAI SDK, httpx |
| TUI | Ink (React), TypeScript |
| 配置 | YAML, JSON |
| 测试 | pytest, pytest-asyncio |
| 任务调度 | croniter |
| 通信 | WebSockets |
