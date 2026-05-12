# OpenHarness Coding Agent — 容器化架构文档

> 文档路径：`docker-sandbox/`
> 验证日期：2026-05-06

---

## 一、目录结构

```
docker-sandbox/
├── Dockerfile          # 镜像构建脚本
├── ubi.repo            # Cargosmart Artifactory yum 源配置
├── agent_api.py        # FastAPI SSE 流式 API 服务（核心）
├── docker-compose.yml  # 容器编排 + 目录挂载
└── projs/              # 项目工作区（宿主机 ↔ 容器共享）
    ├── evdp-api/       # 待 agent 操作的项目（示例）
    └── evdp-ui/
```

---

## 二、整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                      宿主机 / 调用方                          │
│                                                             │
│   curl / Python client                                      │
│        │  POST /run  (JSON)                                 │
│        ↓                                                    │
└────────┼────────────────────────────────────────────────────┘
         │ HTTP SSE stream
┌────────▼────────────────────────────────────────────────────┐
│                   Docker 容器                                │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  agent_api.py (FastAPI + uvicorn :8080)              │   │
│  │                                                      │   │
│  │  POST /run                                           │   │
│  │    ├─ 校验 proj_dir 路径安全性                        │   │
│  │    ├─ 构建 OpenAI API 客户端                          │   │
│  │    └─ 启动 OpenHarness RuntimeBundle                 │   │
│  │         │                                            │   │
│  │         ├─ build_runtime()  初始化工具/权限/插件       │   │
│  │         ├─ start_runtime()  启动 MCP/hook 等服务      │   │
│  │         ├─ handle_line()    驱动 LLM 对话循环         │   │
│  │         └─ close_runtime()  清理资源                  │   │
│  │                                                      │   │
│  │  SSE 事件 → asyncio.Queue → StreamingResponse        │   │
│  └──────────────────────────────────────────────────────┘   │
│                          │                                   │
│              /workspace/projs/ (挂载点)                      │
│                          │                                   │
└──────────────────────────┼──────────────────────────────────┘
                           │ volume mount
┌──────────────────────────▼──────────────────────────────────┐
│              宿主机 docker-sandbox/projs/                    │
│                 evdp-api/  evdp-ui/  ...                    │
└─────────────────────────────────────────────────────────────┘
                           │ HTTP
┌──────────────────────────▼──────────────────────────────────┐
│              LLM 服务（外部）                                 │
│         http://10.222.46.16:5000/v1                         │
│         model: kimi-k2.5-deploy-gs                          │
└─────────────────────────────────────────────────────────────┘
```

---

## 三、Dockerfile 说明

```dockerfile
FROM artifact-docker-base-image-local.cargosmart.com/common/python-oocl:3.12.11-20250731
```

| 层级 | 操作 | 说明 |
|------|------|------|
| 1 | `COPY ubi.repo` + `yum install git` | 按 OOCL 规范通过 Artifactory 安装系统包，**不使用 dnf** |
| 2 | `COPY pyproject.toml / src / ohmo / frontend` | 复制项目源码，`frontend/` 是 hatchling 构建必需的 |
| 3 | `pip install fastapi uvicorn[standard]` | 安装 API 服务依赖 |
| 4 | `pip install .` | 安装 openharness 本体（非 editable，适合容器） |
| 5 | `COPY agent_api.py` | 复制 API 服务入口 |
| 6 | `mkdir /workspace/projs` + `chown node` | 创建挂载点，赋予 node 用户权限 |
| 7 | `USER node` | **切换非 root 用户**（安全扫描要求），uid=1000 与宿主机 andrew 匹配 |

**关键设计决策：**
- 使用基础镜像内置的 `node` 用户（uid=1000），避免创建新用户与已有 uid 冲突
- uid=1000 与宿主机 `andrew` 用户一致，挂载的 `projs/` 目录无需额外授权即可读写
- 不挂载 `~/.openharness` 目录，API 配置通过环境变量注入，运行时目录在容器内自动创建

---

## 四、agent_api.py 详解

### 4.1 整体职责

`agent_api.py` 是一个 **FastAPI HTTP 服务**，将 OpenHarness SDK 的能力封装为 RESTful API，以 **SSE（Server-Sent Events）流式协议** 实时推送 agent 的运行过程。

相比直接运行 TUI 交互界面，它解决了以下问题：
- 无 TTY 环境（容器/CI/远程调用）下运行 agent
- 将 agent 运行输出结构化、网络化，供上游系统消费
- 固定工作目录为指定的项目路径，防止 agent 越权访问其他目录

### 4.2 API 端点

#### `GET /health`
健康检查，返回服务状态和挂载根目录。

```json
{"status": "ok", "projs_root": "/workspace/projs"}
```

#### `POST /run`
启动一次 agent 任务，**以 SSE 流式返回**运行过程。

**请求参数（JSON）：**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `prompt` | string | ✅ | 发给 agent 的任务描述 |
| `proj_dir` | string | ✅ | 项目目录名（`evdp-api`）或绝对路径，**必须在 PROJS_ROOT 内** |
| `model` | string | — | 覆盖模型名，默认读 `OPENHARNESS_MODEL` 环境变量 |
| `max_turns` | int | — | 最大对话轮数，默认 50，上限 200 |
| `system_prompt` | string | — | 追加到默认系统提示词后的自定义提示 |
| `api_key` | string | — | 覆盖 API Key |
| `base_url` | string | — | 覆盖 LLM base URL |
| `api_format` | string | — | `openai` 或 `anthropic` |

**SSE 事件类型：**

| event | data 字段 | 说明 |
|-------|-----------|------|
| `text` | `{"chunk": "..."}` | 助手文字片段，实时流式输出 |
| `tool_start` | `{"tool_name": "...", "tool_input": {...}}` | 工具开始执行 |
| `tool_done` | `{"tool_name": "...", "is_error": bool, "output_snippet": "..."}` | 工具执行完毕 |
| `error` | `{"message": "..."}` | 运行期错误（不中断流，会继续到 done） |
| `done` | `{"status": "ok"/"error", "cwd": "..."}` | 任务结束 |

### 4.3 核心流程

```
POST /run
    │
    ├─ _resolve_cwd(proj_dir)          # 路径安全校验（防穿越）
    │
    ├─ _build_api_client(req)          # 构建 LLM API 客户端
    │       优先级：请求参数 > 环境变量
    │
    └─ StreamingResponse(_run_agent_stream)
            │
            ├─ asyncio.Queue           # 事件缓冲队列
            │
            ├─ _agent_task()           # 后台 Task
            │       ├─ build_runtime() # 初始化：工具注册表、权限检查器、MCP客户端、插件
            │       ├─ start_runtime() # 启动：MCP连接、Hook重载器
            │       ├─ handle_line()   # 驱动 LLM 对话循环（多轮工具调用）
            │       └─ close_runtime() # 清理资源（finally，确保执行）
            │
            └─ yield SSE events        # 从 Queue 取事件，序列化为 SSE 格式推送
```

### 4.4 安全设计

1. **路径安全**：`proj_dir` 通过 `Path.resolve()` + `relative_to(PROJS_ROOT)` 双重验证，拒绝路径穿越（如 `../../etc/passwd`）
2. **权限模式**：agent 运行在 `full_auto` 模式（自动批准所有工具调用），适合无人值守场景
3. **非 root 运行**：容器以 `node`（uid=1000）用户运行
4. **配置分离**：敏感配置（API Key）通过环境变量注入，不硬编码在代码中

---

## 五、docker-compose.yml 说明

```yaml
volumes:
  - ./projs:/workspace/projs        # 项目目录双向挂载

environment:
  OPENHARNESS_API_FORMAT: "openai"
  OPENHARNESS_BASE_URL: "http://10.222.46.16:5000/v1"
  OPENHARNESS_API_KEY: "sk-local-demo"
  OPENHARNESS_MODEL: "kimi-k2.5-deploy-gs"
  PROJS_ROOT: "/workspace/projs"
```

**关键配置说明：**
- `./projs` 挂载到 `/workspace/projs`，agent 写入的文件实时出现在宿主机
- 所有 LLM 配置通过环境变量注入，无需修改代码
- 不挂载 `~/.openharness`，运行时状态（插件、会话）存在容器内（重启后重置）

---

## 六、快速启动

```bash
# 1. 在项目根目录构建镜像
cd /home/andrew/work/poc/OpenHarness
docker build -f docker-sandbox/Dockerfile -t openharness-coding-agent:latest .

# 2. 启动服务
docker compose -f docker-sandbox/docker-compose.yml up -d

# 3. 健康检查
curl http://localhost:8080/health

# 4. 调用 agent（流式）
curl -N -X POST http://localhost:8080/run \
  -H 'Content-Type: application/json' \
  -d '{
    "prompt": "分析这个项目的代码结构并生成 README.md",
    "proj_dir": "evdp-api",
    "max_turns": 30
  }'

# 5. 停止服务
docker compose -f docker-sandbox/docker-compose.yml down
```

---

## 七、环境变量参考

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `PROJS_ROOT` | `/workspace/projs` | 项目挂载根目录，agent 只能访问此目录下的路径 |
| `OPENHARNESS_API_FORMAT` | `openai` | LLM API 格式：`openai` / `anthropic` |
| `OPENHARNESS_BASE_URL` | — | LLM 服务地址，如 `http://10.222.46.16:5000/v1` |
| `OPENHARNESS_API_KEY` | `sk-local` | LLM API Key |
| `OPENHARNESS_MODEL` | — | 默认模型名，如 `kimi-k2.5-deploy-gs` |

---

## 八、已知限制

- **无并发任务隔离**：同一容器内多个 `/run` 请求并发运行，各自在不同 `cwd` 操作，不会冲突，但共享同一套工具注册表
- **会话无持久化**：容器重启后 agent 的会话历史清空（`~/.openharness/sessions/` 在容器内）
- **无鉴权**：当前 API 无 Token 验证，建议在生产环境前添加 Bearer Token 中间件
