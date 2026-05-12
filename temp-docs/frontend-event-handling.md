# 前端事件处理详解 - 后端事件到 UI 更新

## 概述

前端通过 `useBackendSession` Hook 接收后端发送的 JSON-lines 事件流，并根据事件类型更新 UI 状态。

---

## 事件处理入口

**文件**: [frontend/terminal/src/hooks/useBackendSession.ts#L179-L437](../frontend/terminal/src/hooks/useBackendSession.ts#L179-L437)

**关键函数**: `handleEvent(event: BackendEvent)`

这个函数是一个大的条件分支，根据 `event.type` 分发处理逻辑。

---

## 所有事件类型及处理逻辑

### 1. 🟢 `ready` 事件 - 会话准备就绪

**触发**: 后端初始化完成

**处理**:
```typescript
if (event.type === 'ready') {
    setReady(true);  // ← 会话就绪
    
    // 更新状态快照
    const statusSnapshot = stableStringify(event.state ?? {});
    lastStatusSnapshotRef.current = statusSnapshot;
    const nextStatus = event.state ?? {};
    statusRef.current = nextStatus;
    setStatus(nextStatus);  // ← UI 显示状态
    
    // 更新任务列表
    const tasksSnapshot = stableStringify(event.tasks ?? []);
    lastTasksSnapshotRef.current = tasksSnapshot;
    setTasks(event.tasks ?? []);  // ← UI 显示任务
    
    // 更新命令列表
    setCommands(event.commands ?? []);  // ← 可用命令
    
    // 更新 MCP 服务器列表
    const mcpSnapshot = stableStringify(event.mcp_servers ?? []);
    lastMcpSnapshotRef.current = mcpSnapshot;
    setMcpServers(event.mcp_servers ?? []);
    
    // 更新 Bridge 会话列表
    const bridgeSnapshot = stableStringify(event.bridge_sessions ?? []);
    lastBridgeSnapshotRef.current = bridgeSnapshot;
    setBridgeSessions(event.bridge_sessions ?? []);
    
    // ⭐ 如果配置了初始提示词，自动提交
    if (config.initial_prompt && !sentInitialPrompt.current) {
        sentInitialPrompt.current = true;
        sendRequest({type: 'submit_line', line: config.initial_prompt});
        setBusy(true);
    }
}
```

**UI 更新**:
- ✅ 页面标题 / 窗口标题为 Ready
- ✅ 显示初始状态（权限模式、模型等）
- ✅ 列出可用任务
- ✅ 自动发送初始提示词

---

### 2. 📸 `state_snapshot` 事件 - 状态更新

**触发**: 会话状态改变（设置更改、权限改变等）

**处理**:
```typescript
if (event.type === 'state_snapshot') {
    // 增量更新 - 只有新数据与缓存不同时才更新
    const statusSnapshot = stableStringify(event.state ?? {});
    if (statusSnapshot !== lastStatusSnapshotRef.current) {
        lastStatusSnapshotRef.current = statusSnapshot;
        setStatus(event.state ?? {});  // ← UI 重新渲染
    }
    
    // 类似地处理 MCP 服务器和 Bridge 会话
    // 这样做是为了避免不必要的 React 重新渲染
}
```

**优化**: 使用稳定的 JSON 字符串化对比，防止无限重新渲染

---

### 3. 📋 `tasks_snapshot` 事件 - 任务列表更新

**触发**: 任务被添加/完成/删除

**处理**:
```typescript
if (event.type === 'tasks_snapshot') {
    const tasksSnapshot = stableStringify(event.tasks ?? []);
    if (tasksSnapshot !== lastTasksSnapshotRef.current) {
        lastTasksSnapshotRef.current = tasksSnapshot;
        startTransition(() => {
            setTasks(event.tasks ?? []);
        });
    }
}
```

**UI 更新**: 右侧面板显示当前任务列表

---

### 4. 💬 `transcript_item` 事件 - 对话项添加

**触发**: 用户输入、AI 响应、系统消息、工具调用

**处理**:
```typescript
if (event.type === 'transcript_item' && event.item) {
    queueTranscriptItem(event.item as TranscriptItem);
    // ← 添加到待处理队列，稍后批量渲染
}
```

**转录项类型**:
- `role: "user"` - 用户消息
- `role: "assistant"` - AI 助手消息
- `role: "system"` - 系统消息
- `role: "tool"` - 工具执行
- `role: "tool_result"` - 工具结果
- `role: "status"` - 状态消息

---

### 5. 📡 `status` 事件 - 实时状态消息

**触发**: 实时进度消息（正在处理中...）

**处理**:
```typescript
if (event.type === 'status') {
    const message = event.message?.trim();
    if (!message) return;
    
    queueTranscriptItem({role: 'status', text: message});
    if (busy) {
        setBusyLabel(message);  // ← 更新忙碌标签
    }
}
```

**UI 更新**: 显示 "正在...处理中" 的状态标签

---

### 6. 📊 `compact_progress` 事件 - 上下文压缩进度

**触发**: 当对话很长需要压缩时

**处理**:
```typescript
if (event.type === 'compact_progress') {
    const phase = String(event.compact_phase ?? '');
    const trigger = String(event.compact_trigger ?? '');
    
    if (phase === 'hooks_start') {
        setBusyLabel('Preparing retry compaction…');
    } else if (phase === 'context_collapse_start') {
        setBusyLabel('Collapsing oversized context…');
    } else if (phase === 'compact_start') {
        setBusyLabel(trigger === 'reactive' 
            ? 'Context is too large. Compacting and retrying…'
            : 'Compacting conversation memory…'
        );
    } else if (phase === 'compact_retry') {
        setBusyLabel(`Retrying compaction (${attempt})…`);
    } else if (phase === 'compact_failed') {
        setBusyLabel('Compaction failed. Continuing without it…');
    }
    
    if (event.message) {
        queueTranscriptItem({role: 'status', text: event.message});
    }
}
```

**UI 更新**: 显示压缩进度标签

---

### 7. 🔤 `assistant_delta` 事件 - AI 增量文本

**触发**: AI 生成文本（流式输出）

**处理**:
```typescript
if (event.type === 'assistant_delta') {
    const delta = event.message ?? '';
    if (!delta) return;
    
    const isCodexStyle = String(statusRef.current.output_style ?? 'default') === 'codex';
    
    if (isCodexStyle) {
        // Codex 模式：缓存文本，避免频繁重新渲染
        assistantBufferRef.current += delta;
        return;
    }
    
    // 默认模式：累积并在阈值时刷新
    pendingAssistantDeltaRef.current += delta;
    
    if (pendingAssistantDeltaRef.current.length >= ASSISTANT_DELTA_FLUSH_CHARS) {
        flushAssistantDelta();  // ← 立即渲染
        return;
    }
    
    // 设置防抖计时器
    if (!assistantFlushTimerRef.current) {
        assistantFlushTimerRef.current = setTimeout(() => {
            assistantFlushTimerRef.current = null;
            flushAssistantDelta();
        }, ASSISTANT_DELTA_FLUSH_MS);  // ← 100ms 防抖
    }
}
```

**优化**:
- Codex 模式下缓存，避免过多渲染
- 默认模式下使用防抖（100ms）
- 累积到一定字符数（如 50 字符）立即刷新

---

### 8. ✅ `assistant_complete` 事件 - AI 回答完成

**触发**: AI 完成一个完整的回答

**处理**:
```typescript
if (event.type === 'assistant_complete') {
    if (assistantFlushTimerRef.current) {
        clearTimeout(assistantFlushTimerRef.current);
        assistantFlushTimerRef.current = null;
    }
    
    flushTranscriptItems();  // ← 刷新所有待处理项
    
    const isCodexStyle = String(statusRef.current.output_style ?? 'default') === 'codex';
    
    if (isCodexStyle) {
        // Codex 模式：合并缓存的增量
        if (pendingAssistantDeltaRef.current) {
            assistantBufferRef.current += pendingAssistantDeltaRef.current;
            pendingAssistantDeltaRef.current = '';
        }
    } else {
        flushAssistantDelta();
    }
    
    const text = event.message ?? assistantBufferRef.current;
    setTranscript((items) => [...items, {role: 'assistant', text}]);  // ← 添加到转录
    
    clearAssistantDelta();
    setBusyLabel(undefined);  // ⚠️ 不在这里清除 busy，等待 line_complete
}
```

**关键点**: 
- ⚠️ 不立即清除 `busy`（工具调用可能接下来进行）
- 等待 `line_complete` 事件才真正结束

---

### 9. 🏁 `line_complete` 事件 - 行处理完成

**触发**: 用户提交的一行输入处理完毕（最终结束信号）

**处理**:
```typescript
if (event.type === 'line_complete') {
    clearAssistantDelta();
    setBusy(false);          // ← ✅ 真正的结束信号
    setBusyLabel(undefined);
}
```

**UI 更新**: 
- ✅ 清除加载旋转图标
- ✅ 允许用户输入新的消息
- ✅ 显示输入框

---

### 10. 🔧 `tool_started` 和 `tool_completed` 事件 - 工具执行

**触发**: 当 Agent 调用工具时

**处理**:
```typescript
if ((event.type === 'tool_started' || event.type === 'tool_completed') && event.item) {
    if (event.type === 'tool_started') {
        setBusy(true);
        setBusyLabel(`Running ${event.tool_name ?? 'tool'}...`);
    } else {
        setBusyLabel('Processing...');
    }
    
    const enrichedItem: TranscriptItem = {
        ...event.item,
        tool_name: event.item.tool_name ?? event.tool_name ?? undefined,
        tool_input: event.item.tool_input ?? undefined,
        is_error: event.item.is_error ?? event.is_error ?? undefined,
    };
    
    queueTranscriptItem(enrichedItem);  // ← 显示工具调用
}
```

**UI 更新**: 显示 "正在运行 {tool_name}..." 的状态

---

### 11. 🧹 `clear_transcript` 事件 - 清空转录

**触发**: 命令要求清空屏幕

**处理**:
```typescript
if (event.type === 'clear_transcript') {
    flushTranscriptItems();
    clearPendingTranscriptItems();
    setTranscript([]);              // ← 清空整个对话历史
    clearAssistantDelta();
    setBusyLabel(undefined);
}
```

---

### 12. 📌 `select_request` 事件 - 下拉菜单请求

**触发**: 需要用户从下拉菜单选择

**处理**:
```typescript
if (event.type === 'select_request') {
    const m = event.modal ?? {};
    setSelectRequest({
        title: String(m.title ?? 'Select'),
        command: String(m.command ?? ''),
        options: event.select_options ?? [],
    });
    // ← UI 显示模态选择框
}
```

**用户选择后**: 前端发送 `apply_select_command` 请求

---

### 13. 🔲 `modal_request` 事件 - 模态框请求

**触发**: 需要用户输入确认

**处理**:
```typescript
if (event.type === 'modal_request') {
    setModal(event.modal ?? null);  // ← 显示模态框
}
```

---

### 14. ❌ `error` 事件 - 错误消息

**触发**: 后端发生错误

**处理**:
```typescript
if (event.type === 'error') {
    flushTranscriptItems();
    queueTranscriptItem({role: 'system', text: `error: ${event.message ?? 'unknown error'}`});
    clearAssistantDelta();
    setBusy(false);
    setBusyLabel(undefined);
}
```

**UI 更新**: 显示红色错误消息

---

### 15. ✓ `todo_update` 事件 - TODO 项更新

**触发**: TodoWrite 工具完成时

**处理**:
```typescript
if (event.type === 'todo_update') {
    if (event.todo_markdown != null) {
        setTodoMarkdown(event.todo_markdown);  // ← 更新右侧面板
    }
}
```

---

### 16. 👥 `swarm_status` 事件 - Swarm 多智能体状态

**触发**: 多智能体系统状态更新

**处理**:
```typescript
if (event.type === 'swarm_status') {
    if (event.swarm_teammates != null) {
        setSwarmTeammates(event.swarm_teammates);  // ← 更新队友列表
    }
    if (event.swarm_notifications != null) {
        setSwarmNotifications((prev) => 
            [...prev, ...event.swarm_notifications!].slice(-20)  // 保留最新 20 条
        );
    }
}
```

---

### 17. 🎭 `plan_mode_change` 事件 - Plan 模式切换

**触发**: 权限模式改变时

**处理**:
```typescript
if (event.type === 'plan_mode_change') {
    if (event.plan_mode != null) {
        setStatus((s) => {
            const next = {...s, permission_mode: event.plan_mode};
            statusRef.current = next;
            return next;
        });
    }
}
```

**UI 更新**: 更新状态栏中的权限模式显示

---

### 18. 🛑 `shutdown` 事件 - 会话关闭

**触发**: 后端关闭连接

**处理**:
```typescript
if (event.type === 'shutdown') {
    onExit(0);  // ← 退出应用
}
```

---

## 事件处理流程图

```
后端发送 JSON-lines 事件
    ↓
前端接收并解析
    ↓
handleEvent(event)
    ├─ ready → 初始化 UI 状态
    ├─ state_snapshot → 增量更新状态
    ├─ tasks_snapshot → 更新任务列表
    ├─ transcript_item → 排队添加对话项
    ├─ status → 显示状态标签
    ├─ compact_progress → 显示压缩进度
    ├─ assistant_delta → 流式显示文本（防抖）
    ├─ assistant_complete → 完成 AI 回答
    ├─ line_complete → ✅ 清除忙碌状态
    ├─ tool_started/completed → 显示工具执行
    ├─ clear_transcript → 清空对话
    ├─ select_request → 显示下拉菜单
    ├─ modal_request → 显示模态框
    ├─ error → 显示错误消息
    ├─ todo_update → 更新 TODO 面板
    ├─ swarm_status → 更新多智能体状态
    ├─ plan_mode_change → 更新权限模式
    └─ shutdown → 退出应用
```

---

## 关键优化点

| 优化 | 说明 |
|------|------|
| **增量更新** | state_snapshot 使用稳定字符串化对比，防止无必要重新渲染 |
| **防抖** | assistant_delta 使用 100ms 防抖，避免频繁渲染 |
| **批处理** | queueTranscriptItem 批量渲染对话项 |
| **缓存** | assistantBufferRef 缓存 AI 文本，减少状态更新 |
| **阈值** | 文本累积到 50 字符时立即刷新，提高响应性 |

---

## 状态管理

```typescript
// 主要状态
const [transcript, setTranscript]              // 对话历史
const [status, setStatus]                      // 会话状态
const [tasks, setTasks]                        // 任务列表
const [busy, setBusy]                          // 忙碌标志
const [busyLabel, setBusyLabel]                // 忙碌标签
const [ready, setReady]                        // 就绪标志

// 缓冲和引用
const assistantBufferRef                       // AI 文本缓冲
const pendingAssistantDeltaRef                 // 待处理增量
const assistantFlushTimerRef                   // 防抖计时器
const lastStatusSnapshotRef                    // 上次状态快照（对比用）
```

---

## 相关文件

- [useBackendSession.ts](../frontend/terminal/src/hooks/useBackendSession.ts) - 事件处理 Hook
- [protocol.py](../src/openharness/ui/protocol.py) - 事件数据模型
