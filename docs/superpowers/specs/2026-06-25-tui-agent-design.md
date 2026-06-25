# TUI Coding Agent — 设计规范文档

**日期：** 2026-06-25  
**状态：** 已审阅 ✅  
**技术栈：** TypeScript · Ink · Vitest  

---

## 一、项目概述

实现一个最小可用的 **TUI 终端编码 Agent**，对标 Claude Code / Cursor Agent / Codex CLI 等工具的核心形态。

程序在终端内以图形化交互界面（TUI）呈现，用户通过自然语言下达开发任务，Agent 结合代码仓库上下文理解意图、自主规划步骤、调用工具、基于结果继续推理，直到任务完成、失败终止或给出明确回复。

**核心考察点：** Agent Loop · 工具系统 · 权限控制 · LLM Provider 接入 · 会话上下文 · 配置管理 · TUI 交互体验

---

## 二、技术选型

| 维度 | 选型 | 理由 |
|------|------|------|
| 语言 | TypeScript | 类型安全，适合 AI 工具的复杂数据结构 |
| TUI 框架 | [Ink](https://github.com/vadimdemedes/ink) | React 组件范式写终端 UI，组件化强，JSX 开发体验好 |
| 测试框架 | Vitest | 快速、现代，内置 Mock，与 ESM 项目配合好 |
| LLM Provider | 多 Provider 抽象层 | 支持 OpenAI / Anthropic / WPS CodingPlan |

---

## 三、整体架构

采用**单进程分层架构**，通过 async generator + EventEmitter 解耦 TUI 渲染与 Agent 执行。

```
┌─────────────────────────────────────────────┐
│                  Ink TUI 层                  │
│  ChatHistory · InputBox · ToolCallCard       │
│  PermissionPrompt · StatusBar · StreamText   │
└──────────────────────┬──────────────────────┘
                       │ useAgent hook（单向数据流）
┌──────────────────────▼──────────────────────┐
│              Agent Loop 核心                 │
│        状态机驱动，维护 Session 状态          │
└────────┬──────────────┬──────────────────────┘
         │              │
┌────────▼────┐  ┌──────▼──────────────────────┐
│ Tool System │  │     Provider Adapter 层       │
│ Registry    │  │ OpenAI · Anthropic · WPS      │
│ Executor    │  └──────────────────────────────┘
└─────────────┘
         │
┌────────▼────────────────────────────────────┐
│              Config Manager                  │
│         用户级配置 + 项目级配置合并           │
└─────────────────────────────────────────────┘
```

**设计原则：**
- `agent/loop.ts` 是纯业务逻辑，不依赖 Ink，可独立测试
- 工具分两类：`readonly`（自动执行）、`dangerous`（需要批量用户确认）
- UI 通过 `useAgent` hook 订阅 Agent 状态，严格单向数据流

---

## 四、目录结构

```
tui-agent/
├── src/
│   ├── main.tsx                    # 程序入口，挂载 Ink App
│   ├── app.tsx                     # 顶层 React 组件，持有全局状态
│   │
│   ├── agent/
│   │   ├── loop.ts                 # Agent Loop 核心（状态机）
│   │   ├── session.ts              # 会话上下文管理（历史消息）
│   │   └── types.ts                # Agent 相关类型定义
│   │
│   ├── tools/
│   │   ├── registry.ts             # 工具注册与分发
│   │   ├── executor.ts             # 工具执行器（带权限检查）
│   │   ├── list-dir.ts             # 只读：目录列举
│   │   ├── read-file.ts            # 只读：文件读取
│   │   ├── search-files.ts         # 只读：文件匹配/内容搜索
│   │   ├── write-file.ts           # 危险：文件写入
│   │   ├── edit-file.ts            # 危险：文件编辑
│   │   ├── shell.ts                # 危险：Shell 命令执行
│   │   └── types.ts                # 工具类型定义
│   │
│   ├── providers/
│   │   ├── base.ts                 # Provider 抽象接口
│   │   ├── openai.ts               # OpenAI 兼容实现
│   │   ├── anthropic.ts            # Anthropic 实现
│   │   ├── wps.ts                  # WPS CodingPlan 协议实现
│   │   └── factory.ts              # Provider 工厂
│   │
│   ├── config/
│   │   ├── loader.ts               # 配置加载与合并（项目级优先）
│   │   └── types.ts                # 配置类型定义
│   │
│   ├── ui/
│   │   ├── components/
│   │   │   ├── ChatHistory.tsx     # 对话历史展示
│   │   │   ├── InputBox.tsx        # 用户输入框（支持 /命令）
│   │   │   ├── ToolCallCard.tsx    # 工具调用展示卡片
│   │   │   ├── PermissionPrompt.tsx # 批量权限确认弹框
│   │   │   ├── StatusBar.tsx       # 底部状态栏（模型/状态/轮次）
│   │   │   └── StreamText.tsx      # 流式文本渲染
│   │   └── hooks/
│   │       ├── useAgent.ts         # Agent Loop 与 UI 的桥接 hook
│   │       └── useInput.ts         # 输入处理与命令解析 hook
│   │
│   └── history/
│       └── logger.ts               # .ai_history/logs/ 日志写入模块
│
├── tests/
│   ├── agent/loop.test.ts          # Agent Loop 单元测试
│   ├── tools/executor.test.ts      # 工具执行与权限测试
│   ├── providers/mock.test.ts      # Mock LLM Provider 测试
│   └── config/loader.test.ts       # 配置优先级测试
│
├── .ai_history/logs/               # AI 对话内容沉淀（运行时写入）
├── deliverables/                   # 验证产物截图
├── docs/
│   └── superpowers/specs/          # 本文档所在位置
├── package.json
├── tsconfig.json
└── vitest.config.ts
```

---

## 五、Agent Loop 状态机

### 状态定义

```typescript
type AgentState =
  | 'IDLE'                // 等待用户输入
  | 'THINKING'            // LLM 推理中（含流式输出）
  | 'COLLECTING_TOOLS'    // 收集本轮所有 tool_calls
  | 'AWAITING_PERMISSION' // 等待用户对危险工具批量确认
  | 'EXECUTING_TOOLS'     // 执行工具（只读自动 + 已授权危险工具）
  | 'ERROR'               // 错误/超时/达到最大轮次
```

### 状态转移

```
IDLE
  → [用户输入] → THINKING

THINKING
  → [LLM 返回文本流，结束后无 tool_calls] → IDLE
  → [LLM 返回 tool_calls] → COLLECTING_TOOLS
  → [超时 / 达到 maxLoopIterations / 网络错误] → ERROR

COLLECTING_TOOLS
  → [本轮含危险工具] → AWAITING_PERMISSION
  → [本轮全为只读工具] → EXECUTING_TOOLS

AWAITING_PERMISSION
  → [用户确认完毕（含全部拒绝/部分允许/全部允许）]
      → 被拒绝工具的结果立即写入会话历史
      → 若有允许的工具 → EXECUTING_TOOLS
      → 若全部拒绝 → THINKING（直接继续推理，历史已含拒绝信息）
  → [用户操作超时（30s）] → 视为全部拒绝 → THINKING

EXECUTING_TOOLS
  → [所有工具结果返回] → 结果写入会话历史 → THINKING（继续推理）

ERROR
  → [用户输入新消息] → IDLE
```

### 批量权限确认机制

当一轮 LLM 响应含多个 tool_calls 时：
1. 将所有 `dangerous` 类型工具收集成确认列表
2. TUI 展示每个工具的名称、参数摘要
3. 用户可对每个工具选 `y`（允许）/ `n`（拒绝），或 `A`（全部允许）/ `N`（全部拒绝）
4. 被拒绝的工具调用以 `{"role": "tool", "content": "用户拒绝执行: <工具名> <参数摘要>"}` 写入会话历史
5. 被允许的工具正常执行，结果写入会话历史
6. 所有工具处理完毕后，Agent 继续推理

---

## 六、工具系统

### 工具接口

```typescript
interface Tool {
  name: string                    // LLM 调用时使用的函数名
  description: string             // 传给 LLM 的功能描述
  parameters: JSONSchema          // 参数 schema（传给 LLM）
  type: 'readonly' | 'dangerous'  // 权限分类
  execute(params: unknown): Promise<ToolResult>
}

interface ToolResult {
  success: boolean
  output?: string   // 成功时的输出内容
  error?: string    // 失败时的错误信息
}
```

### 工具清单

| 工具名 | 类型 | 入参 | 返回说明 |
|--------|------|------|----------|
| `list_dir` | 只读 | `path: string` | 目录树 JSON，含文件类型和大小 |
| `read_file` | 只读 | `path: string`, `start_line?: number`, `end_line?: number` | 文件内容字符串（含行号） |
| `search_files` | 只读 | `pattern: string`, `dir?: string`, `glob?: string` | 匹配文件路径列表 + 内容片段 |
| `write_file` | **危险** | `path: string`, `content: string` | 成功确认或错误信息 |
| `edit_file` | **危险** | `path: string`, `old_str: string`, `new_str: string` | 成功确认 + 差异摘要，或错误（找不到/不唯一） |
| `run_shell` | **危险** | `command: string`, `timeout?: number` | `{ stdout, stderr, exit_code }` |

### 工具执行器行为

- `readonly` 工具：直接执行，无需询问
- `dangerous` 工具：在 `AWAITING_PERMISSION` 状态下等待用户授权后执行
- 工具执行超时（默认继承 `config.timeout`，`run_shell` 可自定义）使用 `AbortController`
- 工具执行异常时，错误信息写入 `ToolResult.error`，不中断 Agent Loop

---

## 七、Provider 抽象层

### 抽象接口

```typescript
interface LLMProvider {
  name: string
  chat(
    messages: Message[],
    tools: ToolDefinition[],
    options: ChatOptions
  ): AsyncGenerator<ChatChunk>
}

type ChatChunk =
  | { type: 'text_delta'; delta: string }
  | { type: 'tool_call'; id: string; name: string; arguments: string }
  | { type: 'done'; usage?: TokenUsage }
  | { type: 'error'; error: Error }

interface ChatOptions {
  model: string
  maxTokens: number
  signal?: AbortSignal    // 超时控制
}
```

### 各 Provider 实现要点

**OpenAIProvider（兼容所有 OpenAI 兼容接口）：**
- 使用 `stream: true`，解析 SSE 事件流
- 工具调用格式：`function_calling` / `tool_calls`

**AnthropicProvider：**
- 使用 Anthropic Messages API 的 streaming 模式
- 工具调用格式：`tool_use` content block

**WPSProvider（WPS CodingPlan 协议）：**
- 工具调用解析：按协议格式映射到内部 `ChatChunk` 类型
- 流式输出：SSE 或 chunked transfer 解析
- 超时控制：`AbortController`，超时时间取 `config.timeout`
- 基础重试：指数退避（1s → 2s → 4s），最多重试 3 次，仅对网络错误（5xx、ECONNRESET）重试，不对 4xx 重试

**Provider 工厂：** 根据 `config.provider` 字段实例化对应实现，API Key 从环境变量注入。

---

## 八、配置管理

### 配置层级（项目级优先于用户级）

| 配置层级 | 文件路径 | 说明 |
|----------|---------|------|
| 用户级 | `~/.tui-agent/config.json` | 全局默认值 |
| 项目级 | `<cwd>/.tui-agent.json` | 覆盖用户级配置 |

合并规则：深度合并，项目级的任意字段覆盖用户级同名字段。

### 配置内容

```jsonc
{
  "provider": "openai",           // openai | anthropic | wps
  "model": "gpt-4o",
  "baseUrl": "https://api.openai.com/v1",
  "timeout": 30000,               // 单位 ms，默认 30s
  "maxLoopIterations": 20,        // Agent Loop 最大轮次
  "maxTokens": 4096               // LLM 单次最大 token
}
```

### 敏感信息规则

- API Key **不得**写入任何配置文件
- 从环境变量读取：`OPENAI_API_KEY` / `ANTHROPIC_API_KEY` / `WPS_API_KEY`
- 配置加载时，若检测到含 `key`/`secret`/`token` 字样的字段，警告并忽略

---

## 九、会话上下文管理

- 会话历史以 `Message[]` 形式维护（`role: user | assistant | tool`）
- 写入历史的内容：用户输入、模型回复（含文本和 tool_calls）、工具执行结果、权限拒绝结果、错误信息
- 会话历史在内存中保存，`/clear` 命令清空内存历史（不影响 `.ai_history/logs/` 文件）

### 对话内容沉淀

每次会话结束（`/exit` 或程序退出）时，将本次会话关键内容写入：

```
.ai_history/logs/YYYY-MM-DD-HH-mm-ss.jsonl
```

每行一个 JSON 对象，格式：
```jsonc
{"ts": 1719302400000, "role": "user", "content": "..."}
{"ts": 1719302401000, "role": "assistant", "content": "...", "tool_calls": [...]}
{"ts": 1719302402000, "role": "tool", "name": "read_file", "result": "..."}
```

---

## 十、TUI 界面设计

### 布局

```
┌─────────────────────────────────────────────┐
│  会话历史区（ChatHistory）                    │
│  - 用户消息（蓝色标签 [You]）                 │
│  - 模型回复（绿色标签 [Agent]，流式渲染）      │
│  - 工具调用卡片（ToolCallCard，含状态图标）    │
│  - 权限确认弹框（PermissionPrompt，覆盖层）    │
│  ...（滚动）                                  │
├─────────────────────────────────────────────┤
│  输入框（InputBox）                           │
│  > _                                          │
├─────────────────────────────────────────────┤
│  状态栏（StatusBar）                          │
│  [gpt-4o] [THINKING] [轮次: 3/20]             │
└─────────────────────────────────────────────┘
```

### 内置命令

| 命令 | 功能 |
|------|------|
| `/help` | 显示内置命令帮助 |
| `/clear` | 清空当前会话历史（内存） |
| `/model` | 查看当前模型名称 |
| `/model <name>` | 切换到指定模型 |
| `/status` | 查看 Agent 运行状态和配置摘要 |
| `/exit` | 退出程序（写入历史日志后退出） |
| `Ctrl+C` | 同 `/exit` |

---

## 十一、测试范围

按需求要求，测试至少覆盖以下场景：

| 测试场景 | 测试文件 | 关键测试点 |
|----------|----------|-----------|
| Agent 主循环 | `tests/agent/loop.test.ts` | 状态转移、最大轮次终止、错误处理 |
| 工具调用与结果回传 | `tests/tools/executor.test.ts` | 只读工具自动执行、危险工具等待授权 |
| 权限确认与拒绝 | `tests/tools/executor.test.ts` | 拒绝结果写入历史、拒绝后继续推理 |
| 配置优先级 | `tests/config/loader.test.ts` | 项目级覆盖用户级、缺失字段使用默认值 |
| Mock LLM Provider | `tests/providers/mock.test.ts` | 流式输出、tool_calls 解析、重试逻辑 |

---

## 十二、交付物

- **可运行程序：** `npm start` 或编译后的二进制
- **验证产物：** 用本 Agent 自身完成一个小任务（如贪吃蛇游戏），截图放入 `deliverables/`
- **测试覆盖：** `npm test` 全部通过

---

## 十三、扩展方向（可选加分）

- **上下文压缩：** 当会话历史超过 token 阈值时，自动压缩旧消息（摘要替代原文），保留工具调用结果
- **历史搜索：** `/history <keyword>` 命令搜索历史对话
- **多会话管理：** 支持命名会话，`/session list` / `/session switch <name>`
