# OpenHarness 项目文档

> 本目录包含 OpenHarness 项目的完整技术分析文档

## 文档结构

```
my-docs/
├── README.md                    # 本文件 - 文档索引
├── architecture-overview.md     # 整体架构总览
├── module-details.md            # 各模块详细说明
├── implementation-details.md    # 实现细节与技术要点
├── tools-reference.md           # 43+ 工具参考手册
└── agent-customization.md       # Agent 自定义能力详解
```

---

## 快速导航

### 1. 整体架构
📄 **`architecture-overview.md`**

- 五层架构图
- 核心组件详解 (QueryEngine, Tool System, Permission, Memory, Swarm)
- ohmo 应用架构
- 前端架构 (React TUI)
- 数据流向
- 配置系统
- 技术栈清单

---

### 2. Agent 自定义能力
📄 **`agent-customization.md`**

OpenHarness 提供了多层次的 Agent 自定义能力：

- **系统提示词自定义** - 全局/项目级/运行时覆盖
- **CLAUDE.md 项目指令** - 自动发现机制和使用场景
- **技能系统 (Skills)** - 可复用任务模板，支持内置/用户/插件
- **插件系统 (Plugins)** - 扩展工具、命令、Agent 定义
- **自定义 Agent** - YAML frontmatter 定义子 Agent
- **运行时参数** - API/CLI 层动态配置
- **最佳实践** - 项目定制工作流程

---

### 3. 模块详解
📄 **`module-details.md`**

- 完整目录结构
- 逐个模块源码分析
- API 客户端层 (Anthropic, OpenAI, Copilot, Codex)
- 查询引擎层 (对话循环、Token 估算、成本追踪)
- 工具系统 (43+ 工具分类详解)
- 权限系统 (模式、规则、检查流程)
- 记忆系统 (CLAUDE.md, MEMORY.md)
- Swarm 多 Agent 系统 (4 种后端)
- 任务系统 (状态机、存储)
- MCP 协议实现
- UI 层 (3 种运行模式)
- ohmo 模块
- 前端模块
- 模块依赖图
- 配置项清单

---

### 3. 实现细节
📄 **`implementation-details.md`**

- **查询引擎**: 对话循环、轮次控制、上下文压缩
- **工具系统**: 注册机制、执行流程、参数验证
- **UI 通信协议**: JSON Lines over stdio 详细规范
- **权限检查**: 6 层安全检查流程图
- **Swarm 实现**: 子进程后端、Mailbox 消息传递
- **任务系统**: 状态机、存储机制、输出捕获
- **MCP 协议**: 连接管理、工具调用、自动重连
- **记忆系统**: CLAUDE.md 发现算法、记忆注入、搜索
- **前端实现**: useBackendSession Hook、键盘处理、主题系统
- **配置持久化**: 加载流程、合并策略
- **错误处理**: API 错误封装、重试策略
- **性能优化**: 并发执行、大文件处理、缓存策略
- **安全考虑**: 敏感数据保护、子进程隔离
- **调试监控**: 日志系统、性能监控

---

### 4. 工具参考
📄 **`tools-reference.md`**

完整的 43+ 工具使用手册：

| 类别 | 工具数量 | 内容 |
|------|---------|------|
| 文件操作 | 4 | file_read, file_write, file_edit, glob |
| 代码搜索 | 2 | grep, lsp |
| 终端 | 1 | bash |
| Agent | 1 | agent |
| 任务管理 | 6 | task_create, list, get, stop, output, update |
| 团队管理 | 3 | team_create, delete, send_message |
| MCP | 4 | mcp, mcp_auth, list_mcp_resources, read_mcp_resource |
| Cron | 4 | cron_create, list, delete, toggle |
| Git Worktree | 2 | enter_worktree, exit_worktree |
| Web | 2 | web_search, web_fetch |
| 其他 | 11 | ask_user, brief, config, todo, skill, sleep, notebook, tool_search 等 |

包含每个工具的：
- 功能说明
- 输入参数详解
- 使用示例
- 特点说明
- 返回结果格式

---

## 项目统计

| 指标 | 数值 |
|------|------|
| Python 源码文件 | 150+ |
| 内置工具 | 43+ |
| 测试用例 | 114+ |
| MCP 支持 | HTTP / stdio |
| 接入渠道 | 10+ (Telegram, Slack, Discord, Feishu, etc.) |
| 权限模式 | 3 (Full Auto, Default, Plan) |
| Swarm 后端 | 4 (Subprocess, In-process, Tmux, iTerm2) |

---

## 核心设计原则

1. **模块化**: 清晰的模块边界，高内聚低耦合
2. **可扩展**: 插件系统、技能系统、钩子系统
3. **安全**: 多层权限检查、敏感路径保护
4. **性能**: 并发执行、流式响应、自动压缩
5. **易用**: 多模态 UI、丰富的 CLI 命令

---

## 阅读建议

**快速了解**: 阅读 `architecture-overview.md`

**深入开发**: 结合阅读 `module-details.md` 和 `implementation-details.md`

**使用工具**: 查阅 `tools-reference.md`

**自定义 Agent**: 参考 `agent-customization.md`

---

## 相关资源

- 项目主页: https://github.com/HKUDS/OpenHarness
- README (英文): `README.md`
- README (中文): `README.zh-CN.md`
- 更新日志: `CHANGELOG.md`
- 贡献指南: `CONTRIBUTING.md`
