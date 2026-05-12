# OpenHarness 实现细节

## 1. 查询引擎核心实现

### 1.1 对话循环 (Conversation Loop)

**位置**: `src/openharness/engine/query.py`

```python
async def run_query(
    context: QueryContext,
    messages: list[ConversationMessage],
) -> AsyncIterator[tuple[StreamEvent, UsageSnapshot | None]]:
    """
    核心对话循环：
    1. 发送消息到 LLM
    2. 接收 Assistant 响应
    3. 如果有 tool_use，执行工具
    4. 将 tool_result 添加到消息
    5. 再次发送给 LLM (递归)
    """
```

**轮次控制 (Turn Control)**:
- `max_turns`: 限制单次用户输入的最大 Agent 轮数
- 防止无限循环
- 默认值为 8

**上下文压缩 (Auto-Compact)**:

```python
if context.auto_compact_threshold_tokens:
    estimated = estimate_tokens(query_messages)
    if estimated > context.auto_compact_threshold_tokens:
        # 触发自动压缩
        compacted_messages = await compact_history(...)
```

压缩策略:
1. 保留系统提示词
2. 保留最近的 N 条消息
3. 压缩旧消息为摘要
4. 保留任务状态和频道日志

### 1.2 Token 估算

**位置**: `src/openharness/services/token_estimation.py`

```python
def estimate_tokens(messages: list[ConversationMessage]) -> int:
    """
    基于字符数的简化估算：
    - 英文: 1 token ≈ 4 characters
    - 中文: 1 token ≈ 1-2 characters
    """
```

### 1.3 成本追踪

**位置**: `src/openharness/engine/cost_tracker.py`

```python
class CostTracker:
    """追踪 API 调用成本"""
    
    def add(self, usage: UsageSnapshot):
        self._total.input_tokens += usage.input_tokens
        self._total.output_tokens += usage.output_tokens
```

---

## 2. 工具系统实现

### 2.1 工具注册机制

**位置**: `src/openharness/tools/__init__.py`

```python
def create_default_tool_registry() -> ToolRegistry:
    """创建默认工具注册表，注册所有内置工具"""
    registry = ToolRegistry()
    
    # 文件操作
    registry.register(FileReadTool())
    registry.register(FileWriteTool())
    registry.register(FileEditTool())
    registry.register(GlobTool())
    
    # 搜索
    registry.register(GrepTool())
    registry.register(LspTool())
    
    # 终端
    registry.register(BashTool())
    
    # 43+ 其他工具...
    
    return registry
```

### 2.2 工具执行流程

```
ToolUseBlock (from LLM)
        │
        ▼
  ToolRegistry.get(tool_name)
        │
        ▼
  permission_checker.evaluate()
        │
        ├─► 拒绝 → 返回错误结果
        │
        └─► 允许 → tool.execute(arguments, context)
                        │
                        ▼
                  ToolResult
```

### 2.3 工具参数验证

使用 **Pydantic** 模型进行参数验证:

```python
class BashToolInput(BaseModel):
    command: str = Field(..., description="Shell command to execute")
    timeout_seconds: int = Field(default=600, ge=1, le=3600)
    cwd: str | None = Field(default=None, description="Working directory")
```

---

## 3. UI 通信协议

### 3.1 协议格式

**位置**: `src/openharness/ui/protocol.py`

React TUI ↔ Python Backend 通过 **JSON Lines over stdio** 通信。

**Frontend → Backend 消息类型**:

```typescript
// 提交用户输入
type SubmitLine = {
    type: 'submit_line';
    line: string;
};

// 执行命令 (如 /provider, /theme)
type SelectCommand = {
    type: 'select_command';
    command: string;
};

// 响应选择对话框
type ApplySelectCommand = {
    type: 'apply_select_command';
    command: string;
    value: string;
};

// 应用权限决策
type ApplyPermission = {
    type: 'apply_permission';
    decision: 'allow' | 'deny';
};

// 关闭
type Shutdown = {
    type: 'shutdown';
};
```

**Backend → Frontend 消息类型**:

```typescript
// 文本增量
type AssistantTextDelta = {
    type: 'assistant_text_delta';
    text: string;
};

// Tool 执行开始
type ToolExecutionStarted = {
    type: 'tool_execution_started';
    tool_name: string;
};

// Tool 执行完成
type ToolExecutionCompleted = {
    type: 'tool_execution_completed';
    tool_name: string;
    result: string;
    is_error: boolean;
};

// 权限请求弹窗
type PermissionModal = {
    type: 'permission_modal';
    tool_name: string;
    reason: string;
};

// 选择对话框
type SelectRequest = {
    type: 'select_request';
    title: string;
    options: SelectOption[];
};

// 状态更新
type StatusUpdate = {
    type: 'status';
    status: SessionStatus;
};
```

### 3.2 会话状态

```typescript
type SessionStatus = {
    theme?: string;
    output_style?: string;
    permission_mode?: string;
    model?: string;
    provider?: string;
    // ...
};
```

---

## 4. 权限检查实现

### 4.1 检查流程

**位置**: `src/openharness/permissions/checker.py`

```python
def evaluate(self, tool_name: str, *, is_read_only: bool, 
             file_path: str | None = None, 
             command: str | None = None) -> PermissionDecision:
    
    # 1. 内置敏感路径检查 (不可绕过)
    if file_path:
        for pattern in SENSITIVE_PATH_PATTERNS:
            if fnmatch.fnmatch(file_path, pattern):
                return PermissionDecision(allowed=False, reason="...")
    
    # 2. 工具黑名单
    if tool_name in self._settings.denied_tools:
        return PermissionDecision(allowed=False, reason="...")
    
    # 3. 工具白名单
    if tool_name in self._settings.allowed_tools:
        return PermissionDecision(allowed=True, reason="...")
    
    # 4. 路径规则
    if file_path and self._path_rules:
        ...
    
    # 5. 命令模式检查
    if command:
        for pattern in self._settings.denied_commands:
            if fnmatch.fnmatch(command, pattern):
                return PermissionDecision(allowed=False, reason="...")
    
    # 6. 权限模式判断
    if self._settings.mode == PermissionMode.FULL_AUTO:
        return PermissionDecision(allowed=True)
    
    if is_read_only:
        return PermissionDecision(allowed=True)
    
    if self._settings.mode == PermissionMode.PLAN:
        return PermissionDecision(allowed=False, reason="Plan mode...")
    
    # DEFAULT 模式需要确认
    return PermissionDecision(allowed=False, requires_confirmation=True)
```

### 4.2 交互式确认

当 `requires_confirmation=True` 时:
1. Backend 发送 `permission_modal` 消息给 Frontend
2. Frontend 显示确认对话框
3. 用户选择 allow/deny
4. Frontend 发送 `apply_permission` 消息
5. Backend 继续或中止工具执行

---

## 5. Swarm 多 Agent 实现

### 5.1 子进程后端

**位置**: `src/openharness/swarm/subprocess_backend.py`

```python
class SubprocessBackend:
    """子进程 Agent 后端"""
    
    async def spawn(self, command: list[str]) -> str:
        """启动子进程 Agent"""
        process = await asyncio.create_subprocess_exec(
            *command,
            stdin=asyncio.subprocess.PIPE,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE,
        )
        return task_id
    
    async def send_message(self, task_id: str, message: str):
        """发送消息到子进程 (通过 stdin)"""
        process = self._processes[task_id]
        process.stdin.write(message.encode() + b'\n')
        await process.stdin.drain()
```

### 5.2 消息传递机制

**Mailbox 模式**:

```
┌─────────┐    ┌─────────┐    ┌─────────┐
│ Agent A │◄──►│ Mailbox │◄──►│ Agent B │
└─────────┘    └─────────┘    └─────────┘
                     │
                     ▼
               消息队列
               (SQLite/内存)
```

### 5.3 团队生命周期

**位置**: `src/openharness/swarm/team_lifecycle.py`

```python
class TeamLifecycle:
    """管理团队创建、加入、离开、解散"""
    
    async def create_team(self, name: str) -> Team:
        team = Team(name=name, id=generate_id())
        await self._registry.register(team)
        return team
    
    async def add_teammate(self, team_id: str, agent_id: str, 
                          backend_type: BackendType):
        # 根据 backend_type 创建 Agent
        if backend_type == "subprocess":
            await self._subprocess_backend.spawn(...)
        elif backend_type == "in_process":
            await self._in_process_backend.spawn(...)
```

---

## 6. 任务系统实现

### 6.1 任务状态机

```
    ┌─────────┐
    │ pending │◄──── 任务创建
    └────┬────┘
         │ start
         ▼
    ┌─────────┐
    │ running │◄──── 执行中
    └────┬────┘
         │
    ┌────┼────┐
    ▼    ▼    ▼
┌──────┐┌────┐┌─────┐
│completed││failed││killed│
└──────┘└────┘└─────┘
```

### 6.2 任务存储

**位置**: `src/openharness/tasks/manager.py`

```python
class BackgroundTaskManager:
    _tasks_dir: Path  # ~/.openharness/tasks/
    
    def _task_file(self, task_id: str) -> Path:
        return self._tasks_dir / f"{task_id}.json"
    
    async def save_task(self, task: TaskRecord):
        """持久化任务到 JSON"""
        async with aiofiles.open(self._task_file(task.id), 'w') as f:
            await f.write(json.dumps(asdict(task)))
```

### 6.3 输出捕获

```python
async def _capture_output(process, output_file: Path):
    """实时捕获子进程输出到文件"""
    async with aiofiles.open(output_file, 'w') as f:
        while True:
            line = await process.stdout.readline()
            if not line:
                break
            await f.write(line.decode())
            await f.flush()
```

---

## 7. MCP 协议实现

### 7.1 连接管理

**位置**: `src/openharness/mcp/client.py`

```python
class MCPClient:
    """MCP 客户端实现"""
    
    async def connect(self, config: MCPServerConfig):
        if config.transport == "stdio":
            self._session = await self._connect_stdio(config)
        elif config.transport == "http":
            self._session = await self._connect_http(config)
    
    async def _connect_stdio(self, config):
        process = await asyncio.create_subprocess_exec(
            *config.command,
            stdin=PIPE,
            stdout=PIPE,
        )
        return MCPSession(process)
```

### 7.2 工具调用

```python
async def call_tool(self, server_name: str, tool_name: str, 
                   arguments: dict) -> ToolResult:
    client = self._clients[server_name]
    
    # 调用 MCP 工具
    result = await client.session.call_tool(tool_name, arguments)
    
    return ToolResult(
        output=result.content,
        is_error=result.isError,
    )
```

### 7.3 自动重连

```python
async def _with_retry(self, operation):
    """自动重连机制"""
    for attempt in range(MAX_RETRIES):
        try:
            return await operation()
        except MCPConnectionError:
            if attempt < MAX_RETRIES - 1:
                await self._reconnect()
                await asyncio.sleep(BACKOFF_DELAY)
            else:
                raise
```

---

## 8. 记忆系统实现

### 8.1 CLAUDE.md 发现

**位置**: `src/openharness/prompts/claudemd.py`

```python
async def find_claude_md(start_path: Path) -> list[Path]:
    """从 start_path 向上查找 CLAUDE.md"""
    found = []
    current = start_path.resolve()
    
    while current != current.parent:
        claude_md = current / "CLAUDE.md"
        if claude_md.exists():
            found.append(claude_md)
        current = current.parent
    
    return found[::-1]  # 从根到叶子
```

### 8.2 记忆注入

```python
async def build_system_prompt(cwd: Path, base_prompt: str) -> str:
    """构建系统提示词，注入记忆内容"""
    claude_mds = await find_claude_md(cwd)
    
    context_parts = [base_prompt]
    
    for md_file in claude_mds:
        content = await read_file(md_file)
        context_parts.append(f"<context from=\"{md_file}\">\n{content}\n</context>")
    
    # 注入 MEMORY.md
    memory = await load_memory_for_project(cwd)
    if memory:
        context_parts.append(f"<memory>\n{memory}\n</memory>")
    
    return "\n\n".join(context_parts)
```

### 8.3 记忆搜索

**位置**: `src/openharness/memory/search.py`

```python
async def search_memory(query: str, project_path: Path) -> list[MemoryHeader]:
    """基于关键词的记忆搜索"""
    memory_dir = get_memory_dir(project_path)
    results = []
    
    for memory_file in memory_dir.glob("**/*.md"):
        content = await read_file(memory_file)
        if query.lower() in content.lower():
            header = parse_memory_header(memory_file, content)
            results.append(header)
    
    return results
```

---

## 9. 前端实现细节

### 9.1 React Hook - useBackendSession

**位置**: `frontend/terminal/src/hooks/useBackendSession.ts`

```typescript
export function useBackendSession(config: FrontendConfig, onExit: () => void) {
    const [transcript, setTranscript] = useState<TranscriptItem[]>([]);
    const [assistantBuffer, setAssistantBuffer] = useState('');
    const [busy, setBusy] = useState(false);
    const [status, setStatus] = useState<SessionStatus>({});
    
    useEffect(() => {
        const process = spawn('node', [compiledApp], { stdio: ['pipe', 'pipe', 'pipe'] });
        
        // 读取 JSON Lines
        const rl = createInterface(process.stdout!);
        rl.on('line', (line) => {
            const event = JSON.parse(line);
            handleBackendEvent(event);
        });
        
        return () => process.kill();
    }, []);
    
    const sendRequest = (request: FrontendRequest) => {
        process.stdin!.write(JSON.stringify(request) + '\n');
    };
    
    const handleBackendEvent = (event: BackendEvent) => {
        switch (event.type) {
            case 'assistant_text_delta':
                setAssistantBuffer(prev => prev + event.text);
                break;
            case 'tool_execution_started':
                // 添加 spinner
                break;
            case 'tool_execution_completed':
                // 显示工具结果
                break;
            // ...
        }
    };
    
    return { transcript, assistantBuffer, busy, status, sendRequest };
}
```

### 9.2 键盘输入处理

**位置**: `frontend/terminal/src/app.tsx`

```typescript
useInput((chunk, key) => {
    // Ctrl+C → 退出
    if (key.ctrl && chunk === 'c') {
        session.sendRequest({ type: 'shutdown' });
        exit();
        return;
    }
    
    // Tab → 命令补全
    if (key.tab) {
        if (commandHints.length > 0) {
            setInput(commandHints[0] + ' ');
        }
        return;
    }
    
    // Enter → 提交
    if (key.return && !session.busy && !session.modal) {
        handleSubmit(input);
        return;
    }
    
    // ↑↓ → 历史记录
    if (key.upArrow) {
        navigateHistory(-1);
    }
    if (key.downArrow) {
        navigateHistory(1);
    }
});
```

### 9.3 主题系统

**位置**: `frontend/terminal/src/theme/ThemeContext.tsx`

```typescript
type Theme = {
    name: string;
    colors: {
        primary: string;
        secondary: string;
        success: string;
        error: string;
        warning: string;
        muted: string;
        background: string;
        foreground: string;
    };
};

const themes: Record<string, Theme> = {
    default: { /* ... */ },
    dark: { /* ... */ },
    light: { /* ... */ },
    // ...
};
```

---

## 10. 配置持久化

### 10.1 设置加载流程

```python
# 1. 查找配置目录
OPENHARNESS_HOME = os.environ.get('OPENHARNESS_HOME', '~/.openharness')

# 2. 加载配置文件
config_path = Path(OPENHARNESS_HOME) / 'config.json'
if config_path.exists():
    settings = Settings.parse_file(config_path)
else:
    settings = Settings()  # 默认值

# 3. 项目本地配置 (.openharness/config.json)
local_config = Path.cwd() / '.openharness' / 'config.json'
if local_config.exists():
    local_settings = Settings.parse_file(local_config)
    settings = merge_settings(settings, local_settings)
```

### 10.2 配置合并策略

```python
def merge_settings(global_settings: Settings, local_settings: Settings) -> Settings:
    """
    合并策略:
    1. 本地配置覆盖全局配置
    2. 数组合并 (如 allowed_tools)
    3. 字典递归合并 (如 mcp_servers)
    """
    merged = global_settings.copy()
    
    for key, value in local_settings:
        if isinstance(value, list):
            setattr(merged, key, getattr(merged, key) + value)
        elif isinstance(value, dict):
            getattr(merged, key).update(value)
        else:
            setattr(merged, key, value)
    
    return merged
```

---

## 11. 错误处理

### 11.1 API 错误封装

**位置**: `src/openharness/api/errors.py`

```python
class OpenHarnessApiError(Exception):
    """Base API error"""
    pass

class AuthenticationFailure(OpenHarnessApiError):
    """认证失败 (401)"""
    pass

class RateLimitFailure(OpenHarnessApiError):
    """ Rate limit (429)"""
    retry_after: float
    pass

class RequestFailure(OpenHarnessApiError):
    """请求失败 (5xx)"""
    pass
```

### 11.2 重试策略

```python
RETRYABLE_STATUS_CODES = {429, 500, 502, 503, 529}
MAX_RETRIES = 3
BASE_DELAY = 1.0  # seconds

def _get_retry_delay(attempt: int, exc: Exception | None = None) -> float:
    # 使用 Retry-After header (如果有)
    if isinstance(exc, APIStatusError):
        retry_after = exc.headers.get('retry-after')
        if retry_after:
            return min(float(retry_after), MAX_DELAY)
    
    # 指数退避 + 抖动
    delay = min(BASE_DELAY * (2 ** attempt), MAX_DELAY)
    jitter = random.uniform(0, delay * 0.25)
    return delay + jitter
```

### 11.3 工具错误处理

```python
async def execute_tool(tool: BaseTool, arguments: BaseModel, context: ToolExecutionContext) -> ToolResult:
    try:
        result = await tool.execute(arguments, context)
        return result
    except PermissionError as e:
        return ToolResult(output=str(e), is_error=True, metadata={"error_type": "permission"})
    except TimeoutError as e:
        return ToolResult(output=f"Timeout: {e}", is_error=True, metadata={"error_type": "timeout"})
    except Exception as e:
        logger.exception("Tool execution failed")
        return ToolResult(output=f"Error: {e}", is_error=True, metadata={"error_type": "exception"})
```

---

## 12. 性能优化

### 12.1 并发执行

**Grep 工具并行搜索**:

```python
async def grep_parallel(pattern: str, files: list[Path], max_workers: int = 4):
    semaphore = asyncio.Semaphore(max_workers)
    
    async def search_one(file: Path):
        async with semaphore:
            return await grep_file(pattern, file)
    
    results = await asyncio.gather(*[search_one(f) for f in files])
    return flatten(results)
```

### 12.2 大文件处理

```python
async def read_large_file(path: Path, limit: int = 200, offset: int = 0) -> str:
    """高效读取大文件的指定行范围"""
    lines = []
    async with aiofiles.open(path, 'r') as f:
        # 跳过 offset 行
        for _ in range(offset):
            await f.readline()
        
        # 读取 limit 行
        for _ in range(limit):
            line = await f.readline()
            if not line:
                break
            lines.append(line)
    
    return ''.join(lines)
```

### 12.3 缓存策略

```python
@functools.lru_cache(maxsize=128)
def get_tool_schema(tool_name: str) -> dict:
    """缓存工具 schema"""
    tool = registry.get(tool_name)
    return tool.to_api_schema()
```

---

## 13. 安全考虑

### 13.1 敏感数据保护

```python
# 1. 敏感路径黑名单 (不可覆盖)
SENSITIVE_PATH_PATTERNS = (
    "*/.ssh/*",
    "*/.aws/credentials",
    "*/.config/gcloud/*",
    # ...
)

# 2. 命令注入防护
def sanitize_command(command: str) -> str:
    # 拒绝危险模式
    DANGEROUS_PATTERNS = [
        r";\s*rm\s+-rf\s+/",
        r"`.*`",  # 反引号
        r"\$\(.*\)",  # 命令替换
    ]
    for pattern in DANGEROUS_PATTERNS:
        if re.search(pattern, command):
            raise ValueError(f"Dangerous command pattern detected: {pattern}")
    return command

# 3. URL 验证
def validate_url(url: str) -> bool:
    parsed = urlparse(url)
    # 拒绝 file:// 协议
    if parsed.scheme == 'file':
        return False
    # 拒绝内网地址
    if is_private_ip(parsed.hostname):
        return False
    return True
```

### 13.2 子进程隔离

```python
async def safe_subprocess(command: list[str], cwd: Path, timeout: int):
    # 1. 使用绝对路径
    cmd = [str(Path(cwd) / command[0])] + command[1:]
    
    # 2. 限制环境变量
    env = os.environ.copy()
    env.pop('AWS_SECRET_ACCESS_KEY', None)
    env.pop('GITHUB_TOKEN', None)
    
    # 3. 超时控制
    process = await asyncio.create_subprocess_exec(
        *cmd,
        cwd=cwd,
        env=env,
    )
    
    try:
        stdout, stderr = await asyncio.wait_for(
            process.communicate(),
            timeout=timeout
        )
    except asyncio.TimeoutError:
        process.kill()
        raise
```

---

## 14. 调试与监控

### 14.1 日志系统

```python
import logging

logger = logging.getLogger('openharness')

# 配置
logging.basicConfig(
    level=logging.DEBUG if os.environ.get('OPENHARNESS_DEBUG') else logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('~/.openharness/debug.log'),
        logging.StreamHandler(),
    ]
)
```

### 14.2 性能监控

```python
class PerformanceMonitor:
    def __init__(self):
        self._metrics: dict[str, list[float]] = {}
    
    @contextmanager
    def measure(self, operation: str):
        start = time.time()
        try:
            yield
        finally:
            duration = time.time() - start
            self._metrics.setdefault(operation, []).append(duration)
            logger.debug(f"{operation} took {duration:.2f}s")
```
