# OpenHarness 工具参考手册

## 工具统计

总计 **43+** 个内置工具，分为 12 个类别。

---

## 1. 文件操作工具 (File Operations)

### 1.1 file_read_tool

**功能**: 读取文件内容

**输入参数**:
```python
{
    "path": "文件路径 (str)",
    "offset": "起始行号，0-based (int, default=0)",
    "limit": "最大行数 (int, default=200)"
}
```

**使用示例**:
```python
{
    "path": "src/main.py",
    "offset": 10,
    "limit": 50
}
```

**特点**:
- 自动处理大文件，只读取指定范围
- 支持相对路径和绝对路径
- 文件不存在时返回错误

---

### 1.2 file_write_tool

**功能**: 写入文件内容

**输入参数**:
```python
{
    "path": "文件路径 (str)",
    "content": "文件内容 (str)"
}
```

**使用示例**:
```python
{
    "path": "output/result.txt",
    "content": "Hello, OpenHarness!"
}
```

**特点**:
- 自动创建不存在的目录
- 覆盖已有文件

---

### 1.3 file_edit_tool

**功能**: 编辑文件内容 (基于字符串替换)

**输入参数**:
```python
{
    "path": "文件路径 (str)",
    "old_str": "要替换的字符串 (str)",
    "new_str": "新字符串 (str)",
    "replace_all": "是否替换所有匹配 (bool, default=False)"
}
```

**使用示例**:
```python
{
    "path": "config.yaml",
    "old_str": "version: 1.0",
    "new_str": "version: 2.0"
}
```

**特点**:
- 基于精确字符串匹配
- 支持批量替换
- 自动检查替换结果

---

### 1.4 glob_tool

**功能**: 文件通配符搜索

**输入参数**:
```python
{
    "pattern": "搜索模式 (str, e.g., '**/*.py')",
    "root": "搜索根目录 (str, optional)",
    "limit": "最大结果数 (int, default=200)"
}
```

**使用示例**:
```python
{
    "pattern": "**/*.test.ts",
    "root": "src",
    "limit": 100
}
```

**特点**:
- 支持 `**` 递归匹配
- 可限制搜索范围
- 自动排序和去重

---

## 2. 代码搜索工具 (Code Search)

### 2.1 grep_tool

**功能**: 正则表达式搜索文件内容

**输入参数**:
```python
{
    "pattern": "正则表达式 (str)",
    "path": "搜索路径 (str, optional)",
    "file_glob": "文件过滤模式 (str, default='**/*')",
    "case_sensitive": "是否区分大小写 (bool, default=True)",
    "limit": "最大结果数 (int, default=200)",
    "offset": "结果偏移 (int, default=0)"
}
```

**使用示例**:
```python
{
    "pattern": "class.*Tool.*Base",
    "file_glob": "src/**/*.py"
}
```

**特点**:
- 多线程并行搜索
- 支持结果分页
- 显示匹配行号和上下文

---

### 2.2 lsp_tool

**功能**: LSP (Language Server Protocol) 代码智能

**输入参数**:
```python
{
    "operation": "操作类型 (str, enum)",
    "file_path": "文件路径 (str)",
    "symbol": "符号名称 (str, optional)",
    "line": "行号 (int, optional)",
    "character": "列号 (int, optional)"
}
```

**支持的操作**:
- `document_symbol`: 文档符号列表
- `go_to_definition`: 跳转到定义
- `find_references`: 查找引用
- `hover`: 悬停提示
- `workspace_symbol`: 工作区符号搜索

**使用示例**:
```python
{
    "operation": "go_to_definition",
    "file_path": "src/main.py",
    "symbol": "MyClass"
}
```

---

## 3. 终端工具 (Terminal)

### 3.1 bash_tool

**功能**: 执行 Shell 命令

**输入参数**:
```python
{
    "command": "Shell 命令 (str)",
    "timeout_seconds": "超时时间 (int, default=600, max=3600)",
    "cwd": "工作目录 (str, optional)"
}
```

**使用示例**:
```python
{
    "command": "git status",
    "timeout_seconds": 30
}
```

**返回结果**:
- 成功: `{"output": "...", "is_error": false}`
- 失败: `{"output": "...", "is_error": true}`

**安全特性**:
- 危险命令需要确认 (DEFAULT 模式下)
- 超时自动终止
- 可配置 cwd 限制

---

## 4. Agent 工具 (Agent Tools)

### 4.1 agent_tool

**功能**: 创建子 Agent 任务

**输入参数**:
```python
{
    "description": "任务描述 (str)",
    "prompt": "Agent 提示词 (str)",
    "model": "模型名称 (str, optional)",
    "subagent_type": "Agent 类型 (str, optional)"
}
```

**使用示例**:
```python
{
    "description": "Refactor authentication module",
    "prompt": "Refactor the auth.py file to use JWT tokens instead of session cookies...",
    "model": "claude-3-5-sonnet-20241022"
}
```

**特点**:
- 独立上下文
- 支持自定义模型
- 可指定 Agent 类型 (Explore, worker 等)

---

## 5. 任务管理工具 (Task Management)

### 5.1 task_create_tool

**功能**: 创建后台任务

**输入参数**:
```python
{
    "type": "任务类型 (str: local_bash, local_agent)",
    "description": "任务描述 (str)",
    "command": "Shell 命令 (str, for local_bash)",
    "prompt": "Agent 提示词 (str, for local_agent)",
    "cwd": "工作目录 (str, optional)"
}
```

**使用示例**:
```python
# Shell 任务
{
    "type": "local_bash",
    "description": "Run unit tests",
    "command": "pytest tests/ -v",
    "cwd": "."
}

# Agent 任务
{
    "type": "local_agent",
    "description": "Code review",
    "prompt": "Review the changes in PR #123..."
}
```

---

### 5.2 task_list_tool

**功能**: 列出所有任务

**输入参数**:
```python
{
    "status": "过滤状态 (str: pending, running, completed, failed, killed, optional)"
}
```

**返回**: 任务列表，包含 ID、状态、描述、时间等

---

### 5.3 task_get_tool

**功能**: 获取任务详情

**输入参数**:
```python
{
    "task_id": "任务 ID (str)"
}
```

---

### 5.4 task_stop_tool

**功能**: 停止正在运行的任务

**输入参数**:
```python
{
    "task_id": "任务 ID (str)"
}
```

---

### 5.5 task_output_tool

**功能**: 获取任务输出

**输入参数**:
```python
{
    "task_id": "任务 ID (str)",
    "max_bytes": "最大字节数 (int, default=12000)"
}
```

---

### 5.6 task_update_tool

**功能**: 更新任务状态/描述

**输入参数**:
```python
{
    "task_id": "任务 ID (str)",
    "description": "新描述 (str, optional)",
    "progress": "进度百分比 (int, 0-100, optional)",
    "status_note": "状态备注 (str, optional)"
}
```

---

## 6. 团队管理工具 (Swarm/Team)

### 6.1 team_create_tool

**功能**: 创建 Agent 团队

**输入参数**:
```python
{
    "name": "团队名称 (str)"
}
```

---

### 6.2 team_delete_tool

**功能**: 删除团队

**输入参数**:
```python
{
    "name": "团队名称 (str)"
}
```

---

### 6.3 send_message_tool

**功能**: 向子 Agent 发送消息

**输入参数**:
```python
{
    "task_id": "任务/Agent ID (str)",
    "message": "消息内容 (str)"
}
```

---

## 7. MCP 工具 (Model Context Protocol)

### 7.1 mcp_tool

**功能**: 调用 MCP 服务器工具

**输入参数**:
```python
{
    "server_name": "MCP 服务器名称 (str)",
    "tool_name": "工具名称 (str)",
    "arguments": "工具参数 (dict)"
}
```

**使用示例**:
```python
{
    "server_name": "filesystem",
    "tool_name": "read_file",
    "arguments": {"path": "/tmp/test.txt"}
}
```

---

### 7.2 mcp_auth_tool

**功能**: 配置 MCP 服务器认证

**输入参数**:
```python
{
    "server_name": "MCP 服务器名称 (str)",
    "mode": "认证模式 (str: bearer, header, env)",
    "value": "认证值 (str)",
    "key": "键名 (str, optional)"
}
```

---

### 7.3 list_mcp_resources_tool

**功能**: 列出 MCP 服务器可用资源

**输入参数**:
```python
{
    "server_name": "MCP 服务器名称 (str, optional)"
}
```

---

### 7.4 read_mcp_resource_tool

**功能**: 读取 MCP 资源

**输入参数**:
```python
{
    "server_name": "MCP 服务器名称 (str)",
    "uri": "资源 URI (str)"
}
```

---

## 8. Cron 定时任务工具

### 8.1 cron_create_tool

**功能**: 创建定时任务

**输入参数**:
```python
{
    "name": "任务名称 (str)",
    "schedule": "Cron 表达式 (str, e.g., '0 9 * * 1-5')",
    "command": "要执行的命令 (str)",
    "cwd": "工作目录 (str, optional)",
    "enabled": "是否启用 (bool, default=True)"
}
```

**Cron 表达式示例**:
- `*/5 * * * *`: 每 5 分钟
- `0 9 * * 1-5`: 工作日早上 9 点
- `0 0 * * 0`: 每周日午夜

---

### 8.2 cron_list_tool

**功能**: 列出所有定时任务

**输入参数**: 无

**返回**: Cron 任务列表，包含下次执行时间

---

### 8.3 cron_delete_tool

**功能**: 删除定时任务

**输入参数**:
```python
{
    "name": "任务名称 (str)"
}
```

---

### 8.4 cron_toggle_tool

**功能**: 启用/禁用定时任务

**输入参数**:
```python
{
    "name": "任务名称 (str)",
    "enabled": "是否启用 (bool)"
}
```

---

## 9. Git Worktree 工具

### 9.1 enter_worktree_tool

**功能**: 创建并进入 Git worktree

**输入参数**:
```python
{
    "branch": "分支名称 (str)",
    "create_branch": "是否创建新分支 (bool, default=True)",
    "base_ref": "基于的引用 (str, default='HEAD')",
    "path": "worktree 路径 (str, optional)"
}
```

---

### 9.2 exit_worktree_tool

**功能**: 退出并删除 Git worktree

**输入参数**:
```python
{
    "path": "worktree 路径 (str)"
}
```

---

## 10. Web 工具

### 10.1 web_search_tool

**功能**: 网络搜索

**输入参数**:
```python
{
    "query": "搜索关键词 (str)",
    "max_results": "最大结果数 (int, default=5)"
}
```

**返回**: 搜索结果列表 (标题、URL、摘要)

---

### 10.2 web_fetch_tool

**功能**: 获取网页内容

**输入参数**:
```python
{
    "url": "网页 URL (str)",
    "max_chars": "最大字符数 (int, default=12000)"
}
```

**安全特性**:
- 拒绝 file:// 协议
- 拒绝内网地址
- 验证 URL 安全性

---

## 11. 其他工具

### 11.1 ask_user_question_tool

**功能**: 向用户提问

**输入参数**:
```python
{
    "question": "问题内容 (str)"
}
```

**使用场景**: 需要用户确认或补充信息时

---

### 11.2 brief_tool

**功能**: 缩短文本

**输入参数**:
```python
{
    "text": "要缩短的文本 (str)",
    "max_chars": "最大字符数 (int, default=200)"
}
```

---

### 11.3 config_tool

**功能**: 读取/更新配置

**输入参数**:
```python
{
    "action": "操作 (str: 'show' or 'set')",
    "key": "配置键 (str, optional)",
    "value": "配置值 (str, optional)"
}
```

---

### 11.4 todo_write_tool

**功能**: 管理 TODO 列表

**输入参数**:
```python
{
    "item": "TODO 项 (str)",
    "checked": "是否完成 (bool, default=False)",
    "path": "TODO 文件路径 (str, default='TODO.md')"
}
```

---

### 11.5 skill_tool

**功能**: 加载技能文件

**输入参数**:
```python
{
    "name": "技能名称 (str)"
}
```

**技能文件位置**:
- 内置: `src/openharness/skills/bundled/`
- 用户: `~/.openharness/skills/`
- 项目本地: `.openharness/skills/`

---

### 11.6 sleep_tool

**功能**: 延迟执行

**输入参数**:
```python
{
    "seconds": "延迟秒数 (float, max=30)"
}
```

---

### 11.7 notebook_edit_tool

**功能**: 编辑 Jupyter Notebook

**输入参数**:
```python
{
    "path": "Notebook 路径 (str)",
    "cell_index": "单元格索引 (int)",
    "new_source": "新内容 (str)",
    "cell_type": "单元格类型 (str: 'code' or 'markdown', default='code')",
    "mode": "模式 (str: 'replace' or 'append', default='replace')"
}
```

---

### 11.8 tool_search_tool

**功能**: 搜索可用工具

**输入参数**:
```python
{
    "query": "搜索关键词 (str)"
}
```

---

### 11.9 enter_plan_mode_tool / exit_plan_mode_tool

**功能**: 进入/退出计划模式

**输入参数**: 无

**说明**: 计划模式下，所有修改性工具被禁用，适合规划阶段使用。

---

### 11.10 remote_trigger_tool

**功能**: 触发远程任务

**输入参数**:
```python
{
    "name": "任务名称 (str)",
    "timeout_seconds": "超时时间 (int, default=120)"
}
```

---

## 工具权限分类

### 只读工具 (无需确认)

以下工具在 DEFAULT 模式下会自动允许，无需用户确认：

- `file_read_tool`
- `glob_tool`
- `grep_tool`
- `lsp_tool`
- `web_search_tool`
- `web_fetch_tool`
- `task_list_tool`
- `task_get_tool`
- `task_output_tool`
- `cron_list_tool`
- `list_mcp_resources_tool`
- `read_mcp_resource_tool`
- `brief_tool`
- `config_tool` (show 模式)
- `tool_search_tool`

### 修改性工具 (需要确认)

以下工具在 DEFAULT 模式下需要用户确认：

- `file_write_tool`
- `file_edit_tool`
- `bash_tool` (危险命令)
- `task_create_tool` (消耗资源)
- `team_create_tool`
- `cron_create_tool`
- `mcp_auth_tool` (涉及安全)

### 元控制工具

- `enter_plan_mode_tool`: 进入只读模式
- `exit_plan_mode_tool`: 退出只读模式

---

## 工具开发指南

### 创建自定义工具

```python
from pydantic import BaseModel, Field
from openharness.tools.base import BaseTool, ToolExecutionContext, ToolResult

class MyToolInput(BaseModel):
    arg1: str = Field(..., description="参数1描述")
    arg2: int = Field(default=10, description="参数2描述")

class MyTool(BaseTool):
    name = "my_custom_tool"
    description = "工具描述，给 LLM 看"
    input_model = MyToolInput
    
    async def execute(self, arguments: MyToolInput, context: ToolExecutionContext) -> ToolResult:
        # 实现逻辑
        result = f"Processed {arguments.arg1} with {arguments.arg2}"
        return ToolResult(output=result, is_error=False)
    
    def is_read_only(self, arguments: MyToolInput) -> bool:
        # 返回 True 表示只读工具
        return True
```

### 注册工具

```python
from openharness.tools import create_default_tool_registry

registry = create_default_tool_registry()
registry.register(MyTool())
```
