# submit_line 请求流追踪

## 概述

`submit_line` 是前端 React 应用发送给后端的主要请求类型，用于提交用户输入的命令或消息。

---

## 请求来源

### 1. 👤 用户输入主消息（最常见）
**文件**: [frontend/terminal/src/App.tsx#L367](../frontend/terminal/src/App.tsx#L367)

**触发时机**: 用户在终端输入任何命令或消息并按 Enter

```typescript
const handleSubmit = (value: string) => {
    // 检查是否是特殊命令（/plan, /resume 等）
    if (handleCommand(value)) {
        setHistory((items) => [...items, value]);
        setHistoryIndex(-1);
        setInput('');
        return;
    }
    
    // 普通消息提交
    session.sendRequest({type: 'submit_line', line: value});  // ← 这里
    setHistory((items) => [...items, value]);
    setHistoryIndex(-1);
    setInput('');
    session.setBusy(true);
};
```

---

### 2. 🎛️ /plan 命令切换
**文件**: [frontend/terminal/src/App.tsx#L161-L163](../frontend/terminal/src/App.tsx#L161-L163)

**触发时机**: 用户执行 `/plan` 命令来切换 plan 模式

```typescript
if (trimmed === '/plan') {
    const currentMode = String(session.status.permission_mode ?? 'default');
    if (currentMode === 'plan') {
        session.sendRequest({type: 'submit_line', line: '/plan off'});  // ← 关闭 plan
    } else {
        session.sendRequest({type: 'submit_line', line: '/plan on'});   // ← 开启 plan
    }
    session.setBusy(true);
    return true;
}
```

---

### 3. 🚀 会话初始化时
**文件**: [frontend/terminal/src/hooks/useBackendSession.ts#L207](../frontend/terminal/src/hooks/useBackendSession.ts#L207)

**触发时机**: 会话建立后，自动发送初始提示词

```typescript
if (config.initial_prompt && !sentInitialPrompt.current) {
    sentInitialPrompt.current = true;  // 防止重复发送
    sendRequest({type: 'submit_line', line: config.initial_prompt});  // ← 这里
    setBusy(true);
}
```

---

## 后端处理流程

### 请求协议定义
**文件**: [src/openharness/ui/protocol.py#L19-L32](../src/openharness/ui/protocol.py#L19-L32)

```python
class FrontendRequest(BaseModel):
    type: Literal[
        "submit_line",
        "permission_response",
        "question_response",
        "list_sessions",
        "select_command",
        "apply_select_command",
        "shutdown",
    ]
    line: str | None = None  # ← submit_line 使用这个字段
```

### 消息循环处理
**文件**: [src/openharness/ui/backend_host.py#L143-L157](../src/openharness/ui/backend_host.py#L143-L157)

```python
if request.type != "submit_line":
    await self._emit(BackendEvent(type="error", message=f"Unknown request type: {request.type}"))
    continue

if self._busy:  # ← 忙碌检查，防止并发处理
    await self._emit(BackendEvent(type="error", message="Session is busy"))
    continue

line = (request.line or "").strip()
if not line:  # ← 空行检查
    await self._emit(BackendEvent(type="error", message="Empty input"))
    continue

should_continue = await self._process_line(line)  # ← 核心处理逻辑
```

---

## 完整请求流

```
┌─────────────────────────────────┐
│ 用户输入 / 命令 / 初始化          │
└────────────┬────────────────────┘
             │
             ↓
┌─────────────────────────────────┐
│ [App.tsx / useBackendSession]    │
│ sendRequest({                   │
│   type: 'submit_line',          │
│   line: 'user_input'            │
│ })                              │
└────────────┬────────────────────┘
             │ (JSON-lines on stdin)
             ↓
┌─────────────────────────────────┐
│ [backend_host.py]               │
│ _read_requests()                │
│ → request_queue.put(request)    │
└────────────┬────────────────────┘
             │
             ↓
┌─────────────────────────────────┐
│ [backend_host.py]               │
│ 消息循环                         │
│ → _process_line(line)           │
│   • 解析命令                      │
│   • 执行 Agent                   │
│   • 流式发送事件                  │
└────────────┬────────────────────┘
             │ (JSON-lines on stdout)
             ↓
    后端事件流返回前端
```

---

## 关键点

| 概念 | 说明 |
|------|------|
| **请求队列** | 使用 `asyncio.Queue` 解耦前端输入和后端处理 |
| **忙碌标志** | `self._busy` 防止用户在等待时提交新请求 |
| **异步处理** | `_read_requests()` 独立任务负责读取，不阻塞主循环 |
| **线程安全** | 使用 `asyncio.to_thread()` 处理阻塞的 stdin 读取 |
| **初始提示** | 会话初始化时自动发送配置的初始提示词 |

---

## 相关文件

- [backend_host.py](../src/openharness/ui/backend_host.py) - 消息循环核心实现
- [protocol.py](../src/openharness/ui/protocol.py) - 请求/事件协议定义
- [App.tsx](../frontend/terminal/src/App.tsx) - 前端 React 应用
- [useBackendSession.ts](../frontend/terminal/src/hooks/useBackendSession.ts) - React Hook，管理后端连接
