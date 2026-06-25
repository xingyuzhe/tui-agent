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
│   │   ├── types.ts                # Agent 相关类型定义
│   │   └── skills/
│   │       ├── registry.ts         # Skill 注册与查找
│   │       ├── loader.ts           # 读取并解析 skill markdown
│   │       ├── types.ts            # Skill 类型定义
│   │       └── builtin/            # 内置 Skill 文件（.md）
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
| `/context` | 强制刷新仓库上下文（重新读取 README / AGENT.md / git log） |
| `/exit` | 退出程序（写入历史日志后退出） |
| `Ctrl+C` 单次 | 中断当前 Agent 任务，回到 IDLE |
| `Ctrl+C` 两次 | 退出程序 |

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
| Skills 系统 | `tests/agent/skills.test.ts` | 内置/项目级加载、同名覆盖、load_skill 工具执行 |

---

## 十二、交付物

- **可运行程序：** `npm start` 或编译后的二进制
- **验证产物：** 用本 Agent 自身完成一个小任务（如贪吃蛇游戏），截图放入 `deliverables/`
- **测试覆盖：** `npm test` 全部通过

---

## 十三、数据模型

所有跨模块传递的核心数据结构定义如下，作为整个项目的类型契约。

### 消息类型（Message）

```typescript
// 用户消息
interface UserMessage {
  role: 'user'
  content: string
}

// 助手消息（可含流式文本和/或工具调用）
interface AssistantMessage {
  role: 'assistant'
  content: string          // 文本部分（流式累积后的完整内容）
  tool_calls?: ToolCall[]  // 若有工具调用，列表非空
}

// 工具结果消息（每个 tool_call 对应一条）
interface ToolResultMessage {
  role: 'tool'
  tool_call_id: string     // 关联 AssistantMessage 中的 ToolCall.id
  name: string             // 工具名称
  content: string          // 执行结果（成功输出或错误信息）
}

type Message = UserMessage | AssistantMessage | ToolResultMessage
```

### 工具调用（ToolCall）

```typescript
// LLM 请求执行的工具调用（存在于 AssistantMessage）
interface ToolCall {
  id: string               // LLM 生成的唯一 ID，用于关联 ToolResultMessage
  name: string             // 工具函数名
  arguments: string        // JSON 字符串，解析后得到工具参数
}

// 传给 LLM 的工具描述（区别于内部 Tool 接口）
interface ToolDefinition {
  type: 'function'
  function: {
    name: string
    description: string
    parameters: JSONSchema
  }
}
```

### 权限审批请求（PermissionRequest）

```typescript
// Agent Loop 传给 TUI 的待确认工具列表
interface PermissionRequest {
  items: PermissionItem[]
}

interface PermissionItem {
  toolCall: ToolCall
  tool: Tool               // 对应的工具实例，用于展示类型和参数摘要
  decision?: 'allow' | 'deny'  // 用户决策（填写后返回给 Loop）
}
```

### Agent 状态快照（AgentSnapshot）

```typescript
// useAgent hook 向 UI 暴露的只读状态
interface AgentSnapshot {
  state: AgentState
  messages: Message[]          // 完整会话历史（渲染用）
  streamingText: string        // 当前正在流式输出的文本（THINKING 状态时非空）
  pendingPermission: PermissionRequest | null  // 待确认的权限请求
  loopIteration: number        // 当前轮次
  error: string | null         // ERROR 状态时的错误信息
}
```

---

## 十四、系统 Prompt 构造

系统 Prompt 在每次对话请求前动态构造，由 `session.ts` 的 `buildSystemPrompt()` 负责生成。

### 构成结构

```
[1] 角色定义
[2] 当前环境信息（工作目录、时间、操作系统）
[3] 仓库上下文（git 状态 + README 摘要，可选）
[4] 可用工具说明（权限分类）
[5] 可用 Skills 列表
[6] 行为约束
```

### 各部分内容

**[1] 角色定义（固定）**
```
你是一个终端编码助手（TUI Coding Agent）。你在用户的终端中运行，
帮助用户完成代码库相关的开发任务。你可以读取文件、搜索代码、
编辑文件和执行命令。你需要主动使用工具探索代码库来理解任务上下文，
而不是依赖用户描述。
```

**[2] 环境信息（运行时注入）**
```
当前工作目录：/path/to/project
当前时间：2026-06-25 18:00 CST
操作系统：Windows 10 / macOS / Linux
```

**[3] 仓库上下文（启动时采集，可通过 /context 刷新）**
- 读取项目根目录的 `README.md` 前 100 行
- 读取 `AGENT.md` / `.tui-agent/context.md`（若存在，作为项目专属上下文）
- 执行 `git log --oneline -5`（若为 git 仓库）
- 若文件不存在，跳过，不报错

**[4] 可用工具说明（从 ToolRegistry 动态生成）**
```
## 可用工具
- 只读工具（自动执行，无需确认）：list_dir, read_file, search_files, load_skill
- 危险工具（需要用户确认后执行）：write_file, edit_file, run_shell
```

**[5] 可用 Skills 列表（从 SkillRegistry 动态生成）**
```
## 可用 Skills（通过 load_skill 工具加载详细步骤）
- code-review: 对当前代码变更进行系统性代码审查
- write-tests: 为指定模块生成单元测试
- ...
```

**[6] 行为约束（固定）**
```
## 行为约束
- 执行危险操作前，通过工具调用触发用户确认，不要在文本中询问
- 用户拒绝执行某操作时，将拒绝信息纳入推理上下文，调整方案继续
- 单次任务循环不超过 {maxLoopIterations} 轮，超出时主动终止并说明原因
- 不要编造文件内容，必须通过 read_file 工具读取后再引用
```

### 构造时机

- **首次对话**：完整构造，含仓库上下文
- **后续对话**：复用缓存的系统 Prompt（仓库上下文部分不重复抓取）
- **`/context` 命令**：强制刷新仓库上下文部分
- **`/model` 切换**：重新构造系统 Prompt（新模型可能有不同约束）

---

## 十五、中断 / 取消机制

用户在任意非 IDLE 状态下可以中断当前 Agent 行为，中断是非破坏性的。

### 触发方式

| 触发 | 适用状态 | 行为 |
|------|---------|------|
| `Esc` 键 | THINKING / EXECUTING_TOOLS | 软中断：当前轮完成后停止（不中止已发出的请求） |
| `Ctrl+C` 单次 | THINKING / EXECUTING_TOOLS | 硬中断：立即取消当前 LLM 请求 / 工具执行 |
| `Ctrl+C` 两次（1s 内） | 任意状态 | 退出程序，写入日志后终止 |

### 中断实现

```typescript
// AbortController 贯穿 Agent Loop
class AgentLoop {
  private abortController: AbortController | null = null

  interrupt(hard: boolean) {
    if (hard) {
      this.abortController?.abort()  // 取消 LLM 请求 + 工具执行
    } else {
      this.softInterruptFlag = true  // 当前轮结束后检查此标志
    }
  }
}
```

**AbortSignal 传递路径：**
- `abortController.signal` → `ChatOptions.signal` → Provider 的 `fetch` 调用
- `abortController.signal` → `run_shell` 的子进程 `AbortSignal`

### 中断后的状态处理

```
硬中断发生时：
  THINKING  → 丢弃未完成的流式文本 → 写入"[任务已取消]"消息 → IDLE
  EXECUTING_TOOLS → 已完成的工具结果保留 → 未完成的工具标记为"已取消" → IDLE

软中断发生时：
  当前轮工具执行完毕 → 不进入下一轮 THINKING → 写入"[已暂停，等待指令]" → IDLE
```

### 与 AWAITING_PERMISSION 状态的关系

- `AWAITING_PERMISSION` 状态下，`Esc` / `Ctrl+C` 等价于"全部拒绝"，将所有待确认工具标记为拒绝并写入历史，然后回到 `THINKING` 继续推理（而非退出）

---

## 十六、Skills 系统

Skills 是 Agent 按需自主调用的行为规范文件。Agent 在推理过程中判断当前任务是否匹配某个 Skill，若匹配则读取该 Skill 的指导内容并注入上下文，按其步骤执行。

### 工作流程

```
用户输入: "帮我做一次代码审查"
         ↓
  THINKING 状态 — LLM 推理
         ↓ LLM 调用 load_skill("code-review")
  SkillLoader 读取 .tui-agent/skills/code-review.md
         ↓ 将 skill 内容以 system 消息注入会话上下文
  THINKING 状态（继续推理，携带 Skill 指导）
         ↓ 按照 Skill 定义的步骤调用工具、生成输出
```

### Skill 文件格式

```markdown
---
name: code-review
description: "对当前代码变更进行系统性代码审查，输出问题列表和建议"
---

# Code Review Skill

## 执行步骤
1. 用 run_shell("git diff HEAD") 获取当前变更
2. 用 read_file 读取受影响的关键文件
3. 检查以下维度：正确性、性能、安全、可读性
4. 输出结构化报告：[严重程度] 文件:行号 — 问题描述
```

### Skill 加载机制

- **启动时：** `SkillRegistry` 扫描两个位置的 Skill 文件：
  1. `src/agent/skills/builtin/`（内置 Skills，随程序分发）
  2. `<cwd>/.tui-agent/skills/`（项目级自定义 Skills）
- **系统 Prompt：** 将所有已注册 Skill 的 `name + description` 列表注入系统 Prompt，让 LLM 知道哪些 Skill 可用
- **按需加载：** LLM 通过调用内置工具 `load_skill(name)` 触发加载，Skill 内容以 `role: system` 消息注入当前会话上下文（不写入持久化历史）
- **优先级：** 同名时项目级 Skill 覆盖内置 Skill

### load_skill 工具定义

```typescript
{
  name: 'load_skill',
  description: '加载一个 Skill 的详细执行步骤到当前上下文，当任务复杂度需要结构化指导时使用',
  parameters: {
    name: { type: 'string', description: 'Skill 名称，来自已注册的 Skill 列表' }
  },
  type: 'readonly'   // 只读，无需用户确认
}
```

### 内置 Skills 清单

| Skill 名 | 触发场景描述 |
|----------|------------|
| `code-review` | 对代码变更进行系统性审查 |
| `write-tests` | 为指定模块生成单元测试 |
| `explain-code` | 解释代码逻辑并生成注释文档 |
| `fix-bug` | 系统性排查并修复 bug |
| `refactor` | 按最佳实践重构指定模块 |

### 目录结构补充

```
src/agent/
├── skills/
│   ├── registry.ts          # Skill 注册与查找
│   ├── loader.ts            # 读取并解析 skill markdown（含 frontmatter）
│   ├── types.ts             # Skill 类型定义
│   └── builtin/
│       ├── code-review.md
│       ├── write-tests.md
│       ├── explain-code.md
│       ├── fix-bug.md
│       └── refactor.md

.tui-agent/
└── skills/                  # 项目级自定义 Skills（用户创建）
```

### 测试补充

| 测试场景 | 测试文件 | 关键测试点 |
|----------|----------|-----------|
| Skill 注册与加载 | `tests/agent/skills.test.ts` | 内置/项目级加载、同名覆盖、frontmatter 解析 |
| load_skill 工具执行 | `tests/agent/skills.test.ts` | Skill 不存在时的错误处理、注入上下文正确 |

---

## 十七、扩展方向（可选加分）

- **上下文压缩：** 当会话历史超过 token 阈值时，自动压缩旧消息（摘要替代原文），保留工具调用结果
- **历史搜索：** `/history <keyword>` 命令搜索历史对话
- **多会话管理：** 支持命名会话，`/session list` / `/session switch <name>`
