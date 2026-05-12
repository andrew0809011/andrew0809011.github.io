# 消息循环：权限、工具执行和前端集成

这个文档深入分析消息循环中的三个关键机制。

## 一、多并发权限请求处理

### 架构：Future-based 请求池

**位置**: [backend_host.py#L69-79](src/openharness/ui/backend_host.py#L69-L79)

```python
class ReactBackendHost:
    def __init__(self, config: BackendHostConfig) -> None:
        # 并发权限请求池
        self._permission_requests: dict[str, asyncio.Future[bool]] = {}
        # 并发问题请求池
        self._question_requests: dict[str, asyncio.Future[str]] = {}
        # 序列化写锁（保护 stdout，但不限制并发权限）
        self._permission_lock = asyncio.Lock()
```

### 权限请求流程

#### 1. 后端发起请求 (`_ask_permission()`)
**位置**: [backend_host.py#L756-L775](src/openharness/ui/backend_host.py#L756-L775)

```python
async def _ask_permission(self, tool_name: str, reason: str) -> bool:
    """发起权限请求（可被多个工具并发调用）"""
    async with self._permission_lock:
        # 生成唯一请求 ID
        request_id = uuid4().hex
        
        # 创建 Future，该 Future 将在收到前端响应时解决
        future: asyncio.Future[bool] = asyncio.get_running_loop().create_future()
        
        # 存储 Future 在请求池中
        self._permission_requests[request_id] = future
        
    # 发送权限请求模态框到前端
    await self._emit(
        BackendEvent(
            type="modal_request",
            modal={
                "kind": "permission",
                "request_id": request_id,
                "tool_name": tool_name,
                "reason": reason,
            },
        )
    )
    
    # ← 在这里阻塞，等待用户响应
    try:
        return await asyncio.wait_for(future, timeout=300)
    except asyncio.TimeoutError:
        return False  # 超时默认拒绝
    finally:
        # 清理已完成的请求
        self._permission_requests.pop(request_id, None)
```

#### 2. 前端响应 → 后端处理
**位置**: [backend_host.py#L152-L175](src/openharness/ui/backend_host.py#L152-L175)

```python
async def _read_requests(self) -> None:
    """后台任务：读取前端请求"""
    while True:
        raw = await asyncio.to_thread(sys.stdin.buffer.readline)
        payload = raw.decode("utf-8").strip()
        request = FrontendRequest.model_validate_json(payload)
        
        # ← 前端发送权限响应
        if request.type == "permission_response" and request.request_id in self._permission_requests:
            future = self._permission_requests[request.request_id]
            if not future.done():
                # ← 唤醒对应的等待协程
                future.set_result(bool(request.allowed))
            continue
        
        # 其他请求类型继续放入队列
        await self._request_queue.put(request)
```

### 并发场景示例

**多工具并发执行 + 权限请求**:

```
1. 用户输入
   "写一个配置文件，然后创建目录，最后执行初始化脚本"
   ↓
2. LLM 返回 3 个 ToolUseBlock：
   - write_file (config.json)  ← 需要权限
   - mkdir (src/)             ← 需要权限  
   - bash (init.sh)           ← 需要权限
   ↓
3. 并发执行 asyncio.gather([
   _execute_tool_call(...write_file...),  # 阻塞：等待权限
   _execute_tool_call(...mkdir...),       # 阻塞：等待权限
   _execute_tool_call(...bash...),        # 阻塞：等待权限
])
   ↓
4. 后端发送 3 个权限请求到前端
   OHJSON:{"type":"modal_request","modal":{"kind":"permission","request_id":"abc123","tool_name":"write_file",...}}\n
   OHJSON:{"type":"modal_request","modal":{"kind":"permission","request_id":"def456","tool_name":"mkdir",...}}\n
   OHJSON:{"type":"modal_request","modal":{"kind":"permission","request_id":"ghi789","tool_name":"bash",...}}\n
   ↓
5. 用户在前端逐一响应（可以任意顺序）
   用户允许 mkdir  →  前端发送权限响应（request_id: def456, allowed: true）
                     _read_requests 接收 → future.set_result(True) → 唤醒 mkdir 执行
   用户拒绝 bash   →  前端发送权限响应（request_id: ghi789, allowed: false）
                     _read_requests 接收 → future.set_result(False) → 唤醒 bash 返回错误
   用户允许 write_file → 前端发送权限响应（request_id: abc123, allowed: true）
                        _read_requests 接收 → future.set_result(True) → 唤醒 write_file 执行
   ↓
6. 所有协程完成，返回结果给 LLM
   - write_file: "Successfully wrote config.json"
   - mkdir: "Created directory src/"
   - bash: "Permission denied for bash"
```

**关键观察**:
- `_permission_lock` 只保护 Future 的创建和存储，**不阻止并发请求**
- 每个权限请求有自己的 Future，可以独立等待
- 用户可以以任意顺序响应多个权限请求

---

## 二、工具执行的完整流程

### 工具执行流程图

```
engine.submit_message(prompt)
  ↓
run_query(context, messages)
  ├─ while turn_count < max_turns:
  │  ├─ auto_compact (条件)
  │  ├─ api_client.stream_message() ← LLM API 调用
  │  │  ├─ AssistantTextDelta 事件
  │  │  ├─ ApiMessageCompleteEvent ← LLM 完成，包含 ToolUseBlock
  │  │  └─ ApiRetryEvent / ApiTextDeltaEvent / ...
  │  │
  │  ├─ 如果 LLM 返回了工具调用：
  │  │  ├─ yield ToolExecutionStarted ← 工具开始
  │  │  │
  │  │  ├─ 并发执行工具（asyncio.gather）
  │  │  │  ├─ _execute_tool_call(tool_1)
  │  │  │  ├─ _execute_tool_call(tool_2)
  │  │  │  └─ _execute_tool_call(tool_3)
  │  │  │
  │  │  ├─ yield ToolExecutionCompleted ← 每个工具完成
  │  │  │
  │  │  └─ 构建 tool_results → 发送给 LLM
  │  │
  │  └─ 如果 LLM 返回了文本：
  │     └─ yield AssistantTurnComplete ← 轮次完成，return
  │
  └─ 返回 StreamEvent 生成器
```

### `_execute_tool_call()` 详解
**位置**: [query.py#L625-L700](src/openharness/engine/query.py#L625-L700)

```python
async def _execute_tool_call(
    context: QueryContext,
    tool_name: str,
    tool_use_id: str,
    tool_input: dict[str, object],
) -> ToolResultBlock:
    """执行单个工具调用，处理权限和异常"""
    
    # 1. 查找工具定义
    tool = context.tool_registry.get(tool_name)
    if tool is None:
        return ToolResultBlock(
            tool_use_id=tool_use_id,
            content=f"Unknown tool: {tool_name}",
            is_error=True,
        )
    
    # 2. 验证输入
    try:
        parsed_input = tool.input_model.model_validate(tool_input)
    except Exception as exc:
        return ToolResultBlock(
            tool_use_id=tool_use_id,
            content=f"Invalid input for {tool_name}: {exc}",
            is_error=True,
        )
    
    # 3. 权限检查
    _file_path = _resolve_permission_file_path(context.cwd, tool_input, parsed_input)
    _command = _extract_permission_command(tool_input, parsed_input)
    
    decision = context.permission_checker.evaluate(
        tool_name,
        is_read_only=tool.is_read_only(parsed_input),
        file_path=_file_path,
        command=_command,
    )
    
    # 4. 权限处理
    if not decision.allowed:
        # 权限被拒绝，但需要确认？
        if decision.requires_confirmation and context.permission_prompt is not None:
            # ← 发送权限请求，阻塞等待用户响应
            confirmed = await context.permission_prompt(tool_name, decision.reason)
            if not confirmed:
                return ToolResultBlock(
                    tool_use_id=tool_use_id,
                    content=decision.reason or f"Permission denied for {tool_name}",
                    is_error=True,
                )
        else:
            # 权限直接拒绝
            return ToolResultBlock(
                tool_use_id=tool_use_id,
                content=decision.reason or f"Permission denied for {tool_name}",
                is_error=True,
            )
    
    # 5. 执行工具
    log.debug("executing %s ...", tool_name)
    t0 = time.monotonic()
    try:
        result = await tool.execute(
            parsed_input,
            ToolExecutionContext(
                cwd=context.cwd,
                metadata={...},
            ),
        )
    except Exception as exc:
        log.exception("tool execution error: %s", tool_name)
        return ToolResultBlock(
            tool_use_id=tool_use_id,
            content=f"Error executing {tool_name}: {exc}",
            is_error=True,
        )
    
    # 6. 返回结果
    elapsed = time.monotonic() - t0
    log.debug("executed %s in %.2fs err=%s output_len=%d",
              tool_name, elapsed, result.is_error, len(result.output or ""))
    
    return ToolResultBlock(
        tool_use_id=tool_use_id,
        content=result.output,
        is_error=result.is_error,
    )
```

### 并发工具执行
**位置**: [query.py#L550-570](src/openharness/engine/query.py#L550-L570)

```python
async def run_query(...):
    # 一个轮次中，LLM 可能返回多个工具调用
    tool_calls = final_message.tool_uses  # list[ToolUseBlock]
    
    if len(tool_calls) == 1:
        # 单工具：顺序执行
        tc = tool_calls[0]
        yield ToolExecutionStarted(tool_name=tc.name, tool_input=tc.input), None
        result = await _execute_tool_call(context, tc.name, tc.id, tc.input)
        yield ToolExecutionCompleted(...), None
        tool_results = [result]
    else:
        # 多工具：并发执行 ← 这是关键！
        # 1. 先发送所有工具开始事件
        for tc in tool_calls:
            yield ToolExecutionStarted(tool_name=tc.name, tool_input=tc.input), None
        
        # 2. 并发执行所有工具
        async def _run(tc):
            return await _execute_tool_call(context, tc.name, tc.id, tc.input)
        
        # 关键：return_exceptions=True 确保一个失败不影响其他
        raw_results = await asyncio.gather(
            *[_run(tc) for tc in tool_calls],
            return_exceptions=True,  # ← 允许单个失败
        )
        
        # 3. 收集结果并发送完成事件
        tool_results = []
        for tc, result in zip(tool_calls, raw_results):
            if isinstance(result, BaseException):
                # 工具抛异常
                tool_results.append(ToolResultBlock(
                    tool_use_id=tc.id,
                    content=f"Tool execution error: {result}",
                    is_error=True,
                ))
            else:
                tool_results.append(result)
            
            # 每个工具完成后发送事件
            yield ToolExecutionCompleted(
                tool_name=tc.name,
                output=result.content if hasattr(result, 'content') else str(result),
                is_error=result.is_error if hasattr(result, 'is_error') else False,
            ), None
    
    # 4. 构建 ToolResultBlock 消息并返回给 LLM
    messages.append(ConversationMessage(
        role="user",
        content=tool_results,
    ))
    
    # 5. 继续循环 → LLM 再次调用（或停止）
```

### 权限检查决策树
**位置**: [permissions/checker.py#L84-L156](src/openharness/permissions/checker.py#L84-L156)

```
PermissionChecker.evaluate(tool_name, is_read_only, file_path, command)
├─ 1. 内置敏感路径检查（总是应用）
│  └─ if file_path matches SENSITIVE_PATH_PATTERNS:
│     return PermissionDecision(allowed=False, reason="...")
│
├─ 2. 工具显式拒绝列表
│  └─ if tool_name in denied_tools:
│     return PermissionDecision(allowed=False)
│
├─ 3. 工具显式允许列表
│  └─ if tool_name in allowed_tools:
│     return PermissionDecision(allowed=True)
│
├─ 4. 路径级规则
│  └─ for path_rule in path_rules:
│     if file_path matches rule.pattern:
│        return PermissionDecision(allowed=rule.allow)
│
├─ 5. 命令拒绝模式
│  └─ for pattern in denied_commands:
│     if command matches pattern:
│        return PermissionDecision(allowed=False)
│
├─ 6. 权限模式决策
│  ├─ PLAN 模式: 阻止所有变异操作
│  │  └─ if not is_read_only:
│  │     return PermissionDecision(allowed=False)
│  │
│  ├─ FULL_AUTO 模式: 允许所有
│  │  └─ return PermissionDecision(allowed=True)
│  │
│  └─ DEFAULT 模式: 只读允许，变异需确认
│     └─ if is_read_only:
│        return PermissionDecision(allowed=True)
│     else:
│        return PermissionDecision(
│            allowed=False,
│            requires_confirmation=True,  ← 用户会被提示
│            reason="Mutating tools require user confirmation..."
│        )
```

---

## 三、前端事件处理

### 前端接收 BackendEvent 流

前端通过 JSON-lines 协议接收来自后端的事件流。每行是一个完整的 JSON 对象，前缀为 `OHJSON:`。

**协议格式**:
```
OHJSON:{...json event...}\n
OHJSON:{...json event...}\n
```

### React 前端集成点

**文件**: [autopilot-dashboard/src/App.tsx](autopilot-dashboard/src/App.tsx)

前端需要：
1. 连接 WebSocket 或打开子进程 stdin/stdout
2. 按行解析接收的事件
3. 根据事件类型更新 UI 状态
4. 响应用户交互（权限确认、问题回答）

### 典型事件序列和 UI 更新

```typescript
// 后端事件流 → 前端 UI 更新

1. ready: { type: "ready", state: {...}, commands: [...], ... }
   → 初始化仪表板，显示可用命令

2. transcript_item: { type: "transcript_item", item: { role: "user", text: "..." } }
   → 在对话区域显示用户输入

3. assistant_delta: { type: "assistant_delta", message: "..." }
   → 实时流式显示 LLM 回复（逐个单词/字符）

4. tool_started: { 
     type: "tool_started", 
     tool_name: "read_file", 
     tool_input: { "path": "/src/main.py" } 
   }
   → 在对话区域显示"正在执行 read_file ..."

5. compact_progress: { 
     type: "compact_progress", 
     phase: "auto", 
     checkpoint: "save_messages", 
     message: "Auto-compacting..." 
   }
   → 显示推理进度/记忆压缩状态

6. tool_completed: { 
     type: "tool_completed", 
     tool_name: "read_file", 
     output: "文件内容...", 
     is_error: false 
   }
   → 显示工具执行结果

7. assistant_complete: { 
     type: "assistant_complete", 
     message: "完整回复...", 
     item: {...} 
   }
   → 对话轮次完成

8. modal_request: { 
     type: "modal_request", 
     modal: { 
       kind: "permission", 
       request_id: "abc123", 
       tool_name: "bash", 
       reason: "需要权限执行命令" 
     } 
   }
   → 显示权限确认对话框
     用户选择 Allow/Deny → 前端发送：
     { type: "permission_response", request_id: "abc123", allowed: true }

9. modal_request: { 
     type: "modal_request", 
     modal: { 
       kind: "question", 
       request_id: "def456", 
       question: "你要继续吗？" 
     } 
   }
   → 显示问题对话框
     用户输入答案 → 前端发送：
     { type: "question_response", request_id: "def456", answer: "是的" }

10. status_snapshot: { 
      type: "status_snapshot", 
      state: {...}, 
      mcp_servers: [...], 
      bridge_sessions: [...] 
    }
    → 更新右侧状态面板（MCP 服务器状态、会话列表等）

11. tasks_snapshot: { 
      type: "tasks_snapshot", 
      tasks: [{status: "pending", title: "..."}, ...] 
    }
    → 更新 TODO 列表/任务面板

12. line_complete: { type: "line_complete" }
    → 轮次完成，前端可以接受新输入

13. shutdown: { type: "shutdown" }
    → 后端关闭，清理连接
```

### 前端权限对话流程

```
后端                          前端                      用户
  │
  ├─ emit modal_request
  │  (permission, request_id: "abc", tool: "bash")
  │                   ↓
  │          显示权限对话框
  │          "Allow tool bash?"
  │          [Allow] [Deny]
  │                   ↓
  │              用户点击 Allow
  │                   ↓
  │          发送 permission_response
  │          (request_id: "abc", allowed: true)
  ├─ permission_response
  │  received ← 读线程处理
  │  set future result
  │  (协程唤醒) ─→ 继续执行 bash
  │
  └─ 工具完成，回到主循环
```

### 前端实现关键点

```typescript
// React 组件伪代码
function App() {
  const [transcript, setTranscript] = useState([])
  const [pendingModals, setPendingModals] = useState({})
  const [isStreaming, setIsStreaming] = useState(false)
  
  useEffect(() => {
    // 打开 WebSocket 或连接后端
    const ws = new WebSocket("ws://localhost:...")
    
    ws.onmessage = (event) => {
      const line = event.data
      // 移除协议前缀
      if (!line.startsWith("OHJSON:")) return
      
      const eventJson = line.slice(7)
      const event = JSON.parse(eventJson)
      
      // 分发事件
      switch (event.type) {
        case "transcript_item":
          setTranscript(prev => [...prev, event.item])
          break
        
        case "assistant_delta":
          // 实时流式更新
          setTranscript(prev => {
            const last = prev[prev.length - 1]
            if (last?.role === "assistant") {
              return [...prev.slice(0, -1), { ...last, text: (last.text || "") + event.message }]
            }
            return prev
          })
          break
        
        case "tool_started":
          setTranscript(prev => [...prev, { role: "tool", tool_name: event.tool_name, tool_input: event.tool_input }])
          break
        
        case "tool_completed":
          // 更新最后一个工具项
          break
        
        case "modal_request":
          if (event.modal.kind === "permission") {
            // 显示权限对话框
            setPendingModals(prev => ({
              ...prev,
              [event.modal.request_id]: {
                type: "permission",
                tool_name: event.modal.tool_name,
                reason: event.modal.reason,
              }
            }))
          }
          break
        
        case "status_snapshot":
          updateStatusPanel(event.state)
          break
        
        case "line_complete":
          setIsStreaming(false)
          // 解锁输入框
          break
      }
    }
  }, [])
  
  // 用户发送权限响应
  function handlePermissionResponse(requestId: string, allowed: boolean) {
    ws.send(JSON.stringify({
      type: "permission_response",
      request_id: requestId,
      allowed: allowed,
    }))
    setPendingModals(prev => {
      const next = { ...prev }
      delete next[requestId]
      return next
    })
  }
  
  // 用户提交输入
  function handleSubmit(line: string) {
    setIsStreaming(true)
    ws.send(JSON.stringify({
      type: "submit_line",
      line: line,
    }))
  }
}
```

---

## 总结：完整交互流程

```
用户在前端输入命令
  ↓
前端发送 submit_line 请求
  ↓
后端 _read_requests 接收并放入队列
  ↓
主循环 _process_line() 调用 handle_line()
  ↓
handle_line() 调用 engine.submit_message()
  ↓
run_query() 开始工具循环
  ├─ api_client.stream_message() ← LLM API 调用
  │  └─ yield AssistantTextDelta events → 前端显示流式文本
  ├─ 工具执行流程：
  │  ├─ yield ToolExecutionStarted
  │  ├─ asyncio.gather() 并发执行多个工具
  │  │  ├─ 权限检查
  │  │  ├─ 需要权限？→ context.permission_prompt() ← 阻塞，等待 Future
  │  │  │             ↓
  │  │  │         后端发送 modal_request 事件 → 前端显示权限对话
  │  │  │             ↓
  │  │  │         用户允许/拒绝 → 前端发送 permission_response
  │  │  │             ↓
  │  │  │         _read_requests 接收 → future.set_result() ← 唤醒执行
  │  │  │
  │  │  └─ 执行工具 → 返回结果
  │  │
  │  └─ yield ToolExecutionCompleted events
  │
  ├─ 如果有工具结果，继续循环 → LLM 再处理
  └─ 最终 yield AssistantTurnComplete
    ↓
BackendEvent 发送序列
  ├─ transcript_item (用户/系统/工具)
  ├─ assistant_delta (流式文本)
  ├─ tool_started / tool_completed
  ├─ modal_request (权限/问题) ← 可以有多个并发
  ├─ status_snapshot
  ├─ tasks_snapshot
  └─ line_complete ← 可以接受下一个输入
    ↓
前端更新 UI 显示完整对话
```

---

## 关键设计洞察

### 1. **并发 vs 序列化**
- **权限请求**：完全并发（多个 Future 独立等待）
- **工具执行**：可并发（使用 asyncio.gather）
- **stdout 写入**：序列化（使用 _write_lock）

### 2. **阻塞点**
- 权限请求在 `context.permission_prompt()` 阻塞
- 工具执行在 `tool.execute()` 阻塞
- stdin 读取在 `sys.stdin.buffer.readline()` 阻塞（在线程中）

### 3. **事件流式处理**
- LLM API 流式事件即时发送给前端（不缓存）
- 工具执行结果等待后再发送
- 进度事件频繁发送，保持 UI 响应性

### 4. **生命周期管理**
- `self._running` 控制主循环
- `self._busy` 防止新请求在处理期间提交
- `asyncio.Queue` 解耦 I/O 和处理

