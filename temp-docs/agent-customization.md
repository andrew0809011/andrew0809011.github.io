# OpenHarness Agent 自定义能力详解

> 本文档详细介绍 OpenHarness 提供的 Agent 自定义能力，包括系统提示词、技能系统、插件系统、自定义 Agent 定义、项目级配置等。

---

## 目录

1. [概述](#概述)
2. [系统提示词 (System Prompt) 自定义](#系统提示词-system-prompt-自定义)
3. [项目级配置 (CLAUDE.md)](#项目级配置-claudemd)
4. [技能系统 (Skills)](#技能系统-skills)
5. [插件系统 (Plugins)](#插件系统-plugins)
6. [自定义 Agent 定义](#自定义-agent-定义)
7. [运行时参数自定义](#运行时参数自定义)
8. [最佳实践](#最佳实践)

---

## 概述

OpenHarness 提供了多层次的 Agent 自定义能力：

| 层级 | 自定义方式 | 作用范围 | 配置文件/位置 |
|------|-----------|---------|--------------|
| **全局** | 系统提示词、基础设置 | 所有会话 | `~/.openharness/settings.json` |
| **项目级** | 项目特定指令 | 当前项目 | `CLAUDE.md`, `.claude/CLAUDE.md` |
| **技能** | 可复用的任务模板 | 按需加载 | `~/.openharness/skills/`, `.claude/skills/` |
| **插件** | 扩展工具/命令/Agent | 全局/项目 | `~/.openharness/plugins/`, `.openharness/plugins/` |
| **Agent 定义** | 自定义子 Agent | 全局/项目 | `~/.openharness/agents/`, 插件内 `agents/` |
| **运行时** | 临时参数覆盖 | 单次会话 | API 参数、CLI 参数 |

---

## 系统提示词 (System Prompt) 自定义

### 1. 全局系统提示词

通过 `settings.json` 设置全局自定义系统提示词：

```json
{
  "system_prompt": "自定义的系统提示词内容..."
}
```

**加载优先级**（从高到低）：
1. API/CLI 传入的 `system_prompt` 参数
2. `settings.json` 中的 `system_prompt`
3. 内置默认系统提示词

**代码示例**（agent_api.py）：
```python
class RunRequest(BaseModel):
    system_prompt: str | None = Field(None, description="自定义系统提示词（追加到默认提示词后）")
```

### 2. 系统提示词构建流程

系统提示词由多个部分组装而成（`src/openharness/prompts/context.py`）：

```
1. 基础系统提示词 (base)
2. 环境信息 (Environment)
3. 会话设置 (Session Mode, Reasoning Settings)
4. 可用技能列表 (Available Skills)
5. 委派与 Subagent 说明 (Delegation)
6. 项目指令 (CLAUDE.md)
7. 本地环境规则 (Local Rules)
8. Issue/PR 上下文
9. 记忆系统 (Memory)
```

---

## 项目级配置 (CLAUDE.md)

### 1. 自动发现机制

OpenHarness 会自动从工作目录向上查找 `CLAUDE.md` 文件：

```python
# src/openharness/prompts/claudemd.py
def discover_claude_md_files(cwd: str | Path) -> list[Path]:
    current = Path(cwd).resolve()
    results: list[Path] = []
    
    for directory in [current, *current.parents]:
        for candidate in (
            directory / "CLAUDE.md",
            directory / ".claude" / "CLAUDE.md",
        ):
            if candidate.exists():
                results.append(candidate)
        
        # 还会查找 .claude/rules/ 目录下的 .md 文件
        rules_dir = directory / ".claude" / "rules"
        if rules_dir.is_dir():
            for rule in sorted(rules_dir.glob("*.md")):
                results.append(rule)
```

### 2. 支持的文件位置

```
项目目录/
├── CLAUDE.md                 # 项目根目录配置
├── .claude/
│   ├── CLAUDE.md            # 隐藏目录配置
│   └── rules/
│       ├── naming.md        # 命名规范
│       └── workflow.md      # 工作流程
└── src/
    └── ...
```

### 3. 使用场景

**场景1：项目编码规范**
```markdown
# CLAUDE.md

## 编码规范

- 使用 TypeScript 严格模式
- 所有函数必须添加 JSDoc 注释
- 优先使用函数式编程风格
- 禁止使用 any 类型

## 构建命令

```bash
npm run build    # 编译
npm run test     # 运行测试
npm run lint     # 代码检查
```

## 项目结构

- `src/components/` - React 组件
- `src/hooks/` - 自定义 Hooks
- `src/utils/` - 工具函数
```

**场景2：特定领域指令**
```markdown
# CLAUDE.md

## 微服务开发指南

- 每个服务必须有独立的 Dockerfile
- 使用 gRPC 进行服务间通信
- 数据库变更必须通过 migration 文件
- API 端点必须包含版本前缀 /v1/

## 安全要求

- 所有 API 必须验证 JWT Token
- 禁止在日志中输出敏感信息
- 使用参数化查询防止 SQL 注入
```

### 4. 本地环境规则 (`~/.openharness/local_rules/rules.md`)

OpenHarness 会自动加载 `~/.openharness/local_rules/rules.md` 文件，将其内容注入到系统提示词中。

**代码实现**（`src/openharness/personalization/rules.py`）：

```python
_RULES_DIR = Path("~/.openharness/local_rules").expanduser()
_RULES_FILE = _RULES_DIR / "rules.md"

def load_local_rules() -> str:
    """Load the local rules markdown, or empty string if none exist."""
    if _RULES_FILE.exists():
        return _RULES_FILE.read_text(encoding="utf-8").strip()
    return ""
```

**使用场景示例**：

#### Git 工作流规则

创建一个规则文件 `~/.openharness/local_rules/rules.md`：

```markdown
# Git 工作流规则

## 适用范围
在 Git 项目中进行代码修改时必须遵循的工作流程。

## 工作流程

### 第一步：同步最新代码
```bash
git pull
```

### 第二步：创建独立工作空间（Git Worktree）
必须使用 `git worktree` 创建独立的工作目录来开发新功能：

```bash
# 获取当前分支名（主分支）
MAIN_BRANCH=$(git branch --show-current)

# 创建新的 worktree
git worktree add ../<worktree-name> -b <branch-name>
cd ../<worktree-name>
```

### 第三步：编写代码
在新的 worktree 中完成代码修改。

### 第四步：提交并推送
```bash
git add .
git commit -m "feat: add new feature"
git push -u origin <branch-name>
```

### 第五步：创建 Merge Request
创建一个 Merge Request 到主分支（创建 worktree 前所在的分支）。

**重要说明**：
- 请勿直接推送到主分支
- MR 必须经过 code review
```

**Docker 镜像中的应用**：

在容器化环境中，可以在 Dockerfile 中预置规则文件：

```dockerfile
# 复制本地规则到镜像
COPY docker-sandbox/.openharness/rules.md /tmp/rules.md
RUN mkdir -p /home/node/.openharness/local_rules && \
    cp /tmp/rules.md /home/node/.openharness/local_rules/rules.md && \
    chown -R node:node /home/node/.openharness
```

这样所有使用这个镜像启动的 Agent 都会遵循相同的 Git 工作流规范。

#### 其他常见规则类型

**代码审查规则**：
```markdown
# 代码审查规则

每次代码修改后必须：
1. 运行所有测试：pytest -q
2. 检查代码风格：black --check .
3. 检查类型：mypy src/
4. 确认测试覆盖率不低于 80%
```

**安全编码规则**：
```markdown
# 安全编码规则

- 禁止在代码中硬编码 API Key、密码等敏感信息
- 所有用户输入必须进行验证和转义
- 使用参数化查询防止 SQL 注入
- 日志中禁止输出个人身份信息（PII）
```

---

## 技能系统 (Skills)

### 1. 技能概述

技能是可复用的任务指令模板，可以通过 `skill` 工具按需加载。

**核心特性**：
- 按需加载（不会自动注入到系统提示词）
- 多层来源（内置、用户、插件）
- YAML frontmatter 元数据

### 2. 技能来源与优先级

```python
# src/openharness/skills/loader.py
def load_skill_registry(
    cwd: str | Path | None = None,
    *,
    extra_skill_dirs: Iterable[str | Path] | None = None,
    extra_plugin_roots: Iterable[str | Path] | None = None,
) -> SkillRegistry:
    """Load bundled and user-defined skills."""
    # 1. 内置技能 (src/openharness/skills/bundled/)
    # 2. 用户技能 (~/.openharness/skills/)
    # 3. 额外技能目录 (extra_skill_dirs)
    # 4. 插件提供的技能
```

### 3. 技能文件格式

**目录结构**：
```
skills/
├── my-skill/           # 技能目录名作为默认技能名
│   └── SKILL.md       # 必须包含 SKILL.md
├── another-skill/
│   └── SKILL.md
```

**SKILL.md 格式**：
```markdown
---
name: custom-skill-name          # 自定义技能名（可选）
description: 技能的简短描述     # 显示在技能列表中
version: 1.0.0
---

# 技能标题

## 使用时机
何时应该使用这个技能...

## 工作流程
1. 步骤一...
2. 步骤二...
3. 步骤三...

## 规则与约束
- 规则1...
- 规则2...
```

### 4. 内置技能列表

位于 `src/openharness/skills/bundled/content/`：

| 技能名 | 描述 |
|-------|------|
| `plan` | 编码前制定实现计划 |
| `diagnose` | 诊断代码问题 |
| `debug` | 调试程序 |
| `test` | 编写和运行测试 |
| `commit` | 生成规范的 Git commit message |
| `review` | 代码审查 |
| `simplify` | 简化复杂代码 |

### 5. 使用技能

**通过 Agent API**：
```python
# 技能会自动在系统提示词中列出
# 用户可以通过以下方式调用：
# skill(name="plan")
```

**技能调用流程**：
```
1. QueryEngine 收到用户消息
2. 如果用户提到 "plan" 或请求制定计划
3. Agent 调用 `skill(name="plan")` 加载详细指令
4. Agent 按照技能中的流程执行
```

---

## 插件系统 (Plugins)

### 1. 插件目录结构

```
plugin-name/
├── plugin.json           # 插件清单（必需）
├── commands/            # 自定义命令 (.md 文件)
├── skills/              # 技能定义 (SKILL.md)
├── agents/              # Agent 定义 (.md 文件)
├── tools/               # 自定义工具 (.py 文件)
├── hooks/               # 钩子配置
└── mcp.json             # MCP 服务器配置
```

### 2. plugin.json 格式

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "插件描述",
  "enabled_by_default": true,
  "commands_dir": "commands",
  "skills_dir": "skills",
  "tools_dir": "tools",
  "agents_dir": "agents",
  "hooks_file": "hooks.json",
  "mcp_file": "mcp.json"
}
```

### 3. 插件安装位置

```python
# src/openharness/plugins/loader.py
def get_user_plugins_dir() -> Path:
    """~/.openharness/plugins/"""

def get_project_plugins_dir(cwd: str | Path) -> Path:
    """./.openharness/plugins/"""
```

### 4. 插件启停控制

```bash
# 查看已安装插件
oh plugin list

# 启用/禁用插件
oh plugin enable my-plugin
oh plugin disable my-plugin
```

**settings.json 配置**：
```json
{
  "enabled_plugins": {
    "my-plugin": true,
    "another-plugin": false
  }
}
```

### 5. 自定义工具开发

在插件的 `tools/` 目录下创建 Python 文件：

```python
# my-plugin/tools/my_tool.py
from openharness.tools.base import BaseTool
from pydantic import BaseModel

class MyToolArgs(BaseModel):
    arg1: str
    arg2: int = 10

class MyTool(BaseTool):
    name = "my_tool"
    description = "我的自定义工具"
    args_schema = MyToolArgs

    async def run(self, args: MyToolArgs, context) -> str:
        # 实现工具逻辑
        return f"处理结果: {args.arg1}"
```

---

## 自定义 Agent 定义

### 1. Agent 定义文件

**内置 Agent**（`src/openharness/coordinator/agent_definitions.py`）：
- `general-purpose` - 通用 Agent
- `Explore` - 代码库探索
- `Plan` - 计划制定
- `worker` - 代码实现
- `verification` - 结果验证
- `claude-code-guide` - Claude Code 指南

**自定义 Agent 位置**：
- `~/.openharness/agents/*.md` - 用户级 Agent
- 插件内 `agents/*.md` - 插件提供的 Agent

### 2. Agent 定义格式

```markdown
---
name: my-custom-agent
description: |
  何时应该使用这个 Agent...
  可以包含多行描述

# 工具控制
tools: file_read, file_write, bash              # 允许的工具列表
disallowed_tools: agent, file_edit              # 禁止的工具

# 模型配置
model: inherit                                   # inherit 或具体模型名
effort: medium                                   # low/medium/high 或数字

# 权限与行为
permission_mode: default                         # default/acceptEdits/bypassPermissions/plan/dontAsk
max_turns: 100

# 技能与扩展
skills: plan, debug
mcp_servers: []

# 视觉识别
color: blue                                      # red/green/blue/yellow/purple/orange/cyan/magenta/white/gray

# 生命周期
background: false
memory: user                                     # user/project/local
initial_prompt: "记住检查测试覆盖率"

# 其他设置
omit_claude_md: false                            # 是否跳过 CLAUDE.md 注入
critical_system_reminder: "务必运行测试"        # 每次用户输入时重复提醒
---

（可选的详细系统提示词正文）

你是一个非常谨慎的代码审查员...
```

### 3. Agent 加载优先级

```python
# src/openharness/coordinator/agent_definitions.py
def get_all_agent_definitions() -> list[AgentDefinition]:
    """Return all agent definitions: built-in + user + plugin."""
    # Merge order (last writer wins for same name):
    # 1. Built-in agents (lowest priority)
    # 2. User agents (~/.openharness/agents/)
    # 3. Plugin agents (highest priority)
```

### 4. 调用自定义 Agent

**通过 Agent API**：
```python
# 使用 system_prompt 和 skill 参数
RunRequest(
    prompt="分析这个代码库",
    proj_dir="my-project",
    system_prompt="你是一个专注性能优化的专家...",
    max_turns=100
)
```

**通过 agent 工具**：
```python
agent(
    description="性能分析",
    prompt="分析这个函数的性能瓶颈",
    subagent_type="my-custom-agent"  # 使用自定义 Agent
)
```

---

## 运行时参数自定义

### 1. Agent API 层

来自 `docker-sandbox/agent_api.py`：

```python
class RunRequest(BaseModel):
    prompt: str                    # 任务描述
    proj_dir: str                  # 项目目录
    model: str | None              # 覆盖模型名
    max_turns: int = 50            # 最大对话轮数
    system_prompt: str | None      # 自定义系统提示词
    api_key: str | None            # 覆盖 API Key
    base_url: str | None           # 覆盖 API Base URL
    api_format: str | None         # API 格式: openai/anthropic
```

### 2. 运行时配置（Runtime Bundle）

```python
# src/openharness/ui/runtime.py
class AgentRuntimeBundle:
    """运行时配置包"""
    extra_skill_dirs: tuple[str, ...]      # 额外技能目录
    extra_plugin_roots: tuple[str, ...]    # 额外插件目录
    settings: Settings                     # 完整设置对象
    engine: QueryEngine                    # 查询引擎实例
```

**运行时参数应用顺序**：
```
1. 基础设置 (settings.json)
2. 项目级设置 (CLAUDE.md 等)
3. 额外技能/插件目录
4. 请求的 system_prompt（追加/覆盖）
→ 构建最终运行时系统提示词
```

### 3. 环境变量覆盖

```bash
# API 配置
export OPENHARNESS_API_KEY=sk-xxx
export OPENHARNESS_BASE_URL=https://api.example.com
export OPENHARNESS_API_FORMAT=openai
export OPENHARNESS_MODEL=gpt-4

# 运行时行为
export OPENHARNESS_MAX_TURNS=100
```

---

## 最佳实践

### 1. 项目级定制工作流程

```bash
# 1. 在项目根目录创建 CLAUDE.md
echo "# My Project Guidelines" > CLAUDE.md

# 2. 添加项目特定的技能
mkdir -p .claude/skills/onboarding
cat > .claude/skills/onboarding/SKILL.md << 'EOF'
---
name: onboarding
description: 新成员项目入门指南
---

## 第一步
阅读 README.md 和 CONTRIBUTING.md

## 第二步
运行 setup 脚本：
```bash
./scripts/setup.sh
```
EOF

# 3. 可选：创建项目特定的 Agent
mkdir -p .claude/agents
cat > .claude/agents/security-review.md << 'EOF'
---
name: security-reviewer
description: 安全审计 Agent
tools: file_read, grep, bash
disallowed_tools: file_write, file_edit
model: sonnet
---

专注发现安全漏洞：
- SQL 注入
- XSS
- 敏感信息泄露
- 权限绕过
EOF
```

### 2. 技能设计原则

1. **聚焦单一职责**：每个技能只做一件事
2. **清晰的触发条件**：让 Agent 知道何时使用
3. **可执行的步骤**：提供具体的操作流程
4. **示例丰富**：包含使用示例

### 3. 插件开发最佳实践

1. **渐进式增强**：默认启用基础功能
2. **文档完整**：提供清晰的 README
3. **依赖清晰**：明确列出外部依赖
4. **测试覆盖**：为自定义工具编写测试

### 4. 系统提示词设计

1. **保持简洁**：过长的提示词会稀释重点
2. **结构化**：使用标题和列表
3. **示例驱动**：用例子说明期望的行为
4. **逐步优化**：从基础版本开始迭代

### 5. 多 Agent 协作模式

```python
# 主 Agent 委派任务
agent(description="规划阶段", prompt="设计架构", subagent_type="Plan")
agent(description="实现阶段", prompt="编写代码", subagent_type="worker")
agent(description="测试阶段", prompt="验证实现", subagent_type="verification")
```

---

## 总结

OpenHarness 提供了丰富的 Agent 自定义能力：

| 能力 | 实现方式 | 适用场景 |
|------|---------|---------|
| **全局系统提示词** | `settings.json` | 个人偏好、全局行为 |
| **CLAUDE.md** | 项目根目录 | 项目规范、构建流程 |
| **Skills** | `SKILL.md` 文件 | 可复用任务模板 |
| **Plugins** | `plugin.json` + 扩展 | 工具/命令/Agent 扩展 |
| **自定义 Agent** | `.md` 定义文件 | 特定角色/工作流 |
| **运行时参数** | API/CLI 参数 | 单次任务定制 |

这些机制可以组合使用，创建高度定制化的 AI 助手体验。
