# OpenHarness 支持的特殊命令（Commands）

## 概述

OpenHarness 前端和后端支持多种斜杠命令 `/command`，这些命令用于控制会话行为、配置设置、切换模式等。

---

## 所有支持的命令

### 1. 🎛️ `/plan` - 切换 Plan 模式
**功能**: 切换权限模式（plan 模式 ↔ default 模式）

**触发方式**: 
- 前端识别 `/plan` 命令
- 自动发送 `/plan on` 或 `/plan off` 作为 submit_line 请求

**代码位置**: `frontend/terminal/src/App.tsx#L158-L168`

```typescript
if (trimmed === '/plan') {
    const currentMode = String(session.status.permission_mode ?? 'default');
    if (currentMode === 'plan') {
        session.sendRequest({type: 'submit_line', line: '/plan off'});
    } else {
        session.sendRequest({type: 'submit_line', line: '/plan on'});
    }
}
```

---

### 2. 📋 `/resume` - 恢复会话
**功能**: 列出并选择一个之前保存的会话进行恢复

**触发方式**:
```typescript
session.sendRequest({type: 'select_command', command: 'resume'});
```

**流程**: 
- 后端返回 `select_request` 事件
- 前端显示下拉菜单
- 用户选择后发送 `/resume <session_id>`

**后端处理**: backend_host.py#L335 (`src/openharness/ui/backend_host.py#L335`)
```python
if command == "resume":
    return f"/resume {value}" if value else "/resume"
```

---

### 3. 🔌 `/provider` - 切换 AI 提供商
**功能**: 选择不同的 AI 模型提供商（OpenAI, Anthropic, 等）

**下拉菜单选项**:
- openai
- anthropic
- local
- 等等

**后端处理**:
```python
if command == "provider":
    return f"/provider {value}"
```

---

### 4. 🎨 `/theme` - 切换主题
**功能**: 切换 UI 主题（亮色 ↔ 暗色）

**支持的选项**:
- light
- dark
- auto

**后端处理**:
```python
if command == "theme":
    return f"/theme {value}"
```

---

### 5. 🎬 `/output-style` - 切换输出样式
**功能**: 改变 AI 输出的格式

**支持的选项**:
- default - 标准输出
- codex - 代码优化的紧凑输出

**后端处理**:
```python
if command == "output-style":
    return f"/output-style {value}"
```

---

### 6. 💪 `/effort` - 设置努力级别
**功能**: 控制 Agent 的处理深度和资源消耗

**支持的选项**:
- low
- medium
- high

**后端处理**:
```python
if command == "effort":
    return f"/effort {value}"
```

---

### 7. 🔄 `/passes` - 设置处理遍数
**功能**: 设置 Agent 可以进行的处理遍数

**后端处理**:
```python
if command == "passes":
    return f"/passes {value}"
```

---

### 8. 🔢 `/turns` - 设置最大轮次
**功能**: 设置 Agent 可以进行的最大对话轮次

**后端处理**:
```python
if command == "turns":
    return f"/turns {value}"
```

---

### 9. ⚡ `/fast` - 快速模式
**功能**: 启用/禁用快速模式（降低质量但提高速度）

**支持的选项**:
- on
- off

**后端处理**:
```python
if command == "fast":
    return f"/fast {value}"
```

---

### 10. ⌨️ `/vim` - Vim 快捷键模式
**功能**: 启用/禁用 Vim 快捷键

**支持的选项**:
- on
- off

**后端处理**:
```python
if command == "vim":
    return f"/vim {value}"
```

---

### 11. 🎤 `/voice` - 语音输入模式
**功能**: 启用/禁用语音输入功能

**支持的选项**:
- on
- off

**后端处理**:
```python
if command == "voice":
    return f"/voice {value}"
```

---

### 12. 🤖 `/model` - 切换模型
**功能**: 选择具体的 AI 模型

**示例**:
- gpt-4
- gpt-3.5-turbo
- claude-3-opus
- 等等

**后端处理**:
```python
if command == "model":
    return f"/model {value}"
```

---

### 13. 🔐 `/permissions` - 权限设置
**功能**: 管理 Agent 的执行权限

**后端处理**:
```python
if command == "permissions":
    return f"/permissions {value}"
```

---

## 命令分类

### 配置类命令
这些命令用于改变会话配置：
- `/provider` - AI 提供商
- `/theme` - 主题
- `/output-style` - 输出格式
- `/model` - AI 模型

### 行为控制类命令
这些命令改变 Agent 的行为方式：
- `/effort` - 努力级别
- `/passes` - 处理遍数
- `/turns` - 最大轮次
- `/fast` - 快速模式
- `/permissions` - 执行权限

### 模式切换类命令
这些命令改变输入/输出模式：
- `/plan` - Plan 模式
- `/vim` - Vim 快捷键
- `/voice` - 语音输入

### 会话管理类命令
这些命令用于管理会话：
- `/resume` - 恢复会话

---

## 命令处理流程

```
前端识别命令
    │
    ├─ 斜杠命令且需要下拉菜单?
    │  ├─ YES: 发送 select_command 请求
    │  │       后端返回 select_request 事件
    │  │       前端显示下拉菜单
    │  │       用户选择 → apply_select_command 请求
    │  │       最终转换为 /command value 并发送 submit_line
    │  │
    │  └─ NO: 直接发送 submit_line 请求
    │
    └─ 后端处理
       ├─ 查找命令处理器
       ├─ 执行命令逻辑
       ├─ 返回 CommandResult
       └─ 可能产生 submit_prompt 给 AI 引擎
```

---

## 前端命令检测

位置: `frontend/terminal/src/App.tsx#L135-L180`

```typescript
const handleCommand = (value: string): boolean => {
    const trimmed = value.trim().toLowerCase();

    // /plan → toggle plan mode
    if (trimmed === '/plan') {
        const currentMode = String(session.status.permission_mode ?? 'default');
        if (currentMode === 'plan') {
            session.sendRequest({type: 'submit_line', line: '/plan off'});
        } else {
            session.sendRequest({type: 'submit_line', line: '/plan on'});
        }
        session.setBusy(true);
        return true;
    }

    // /resume → request session list from backend
    if (trimmed === '/resume') {
        session.sendRequest({type: 'select_command', command: 'resume'});
        return true;
    }

    return false;  // 不是特殊命令，继续发送为普通消息
};
```

---

## 相关文件

- backend_host.py#L335-L360 (`src/openharness/ui/backend_host.py#L335-L360`) - 命令行转换
- App.tsx#L135-L180 (`frontend/terminal/src/App.tsx#L135-L180`) - 前端命令处理
- runtime.py#L487 (`src/openharness/ui/runtime.py#L487`) - 后端命令分发
