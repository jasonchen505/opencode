# LLM & Agent 面试准备指南

> 基于 OpenCode 项目的深度技术分析，为 LLM 算法实习面试准备

---

## 目录

1. [项目概述与架构](#1-项目概述与架构)
2. [LLM 核心概念](#2-llm-核心概念)
3. [Agent 系统设计](#3-agent-系统设计)
4. [Code Agent 实现细节](#4-code-agent-实现细节)
5. [工具系统设计](#5-工具系统设计)
6. [会话管理与上下文](#6-会话管理与上下文)
7. [面试重点与深挖点](#7-面试重点与深挖点)
8. [可以主动介绍的项目亮点](#8-可以主动介绍的项目亮点)

---

## 1. 项目概述与架构

### 1.1 OpenCode 是什么

OpenCode 是一个**开源的 AI 编程助手**，类似于 Cursor、GitHub Copilot 等产品。它是一个完整的 Code Agent 系统，能够：

- 理解代码库上下文
- 执行文件操作（读写、编辑、搜索）
- 运行 shell 命令
- 与多个 LLM provider 交互
- 管理会话和上下文

### 1.2 核心架构

```
┌─────────────────────────────────────────────────────────────┐
│                      用户界面层                              │
│  (TUI/Web/Desktop/CLI)                                      │
├─────────────────────────────────────────────────────────────┤
│                      API Server 层                           │
│  (HTTP API, WebSocket, SSE)                                 │
├─────────────────────────────────────────────────────────────┤
│                      Session 层                              │
│  (会话管理, 上下文组装, 提示词工程)                          │
├─────────────────────────────────────────────────────────────┤
│                      Agent 层                                │
│  (Agent 选择, 权限管理, 工具调度)                            │
├─────────────────────────────────────────────────────────────┤
│                      LLM 层                                  │
│  (Provider 抽象, 协议适配, 流式处理)                        │
├─────────────────────────────────────────────────────────────┤
│                      Tool 层                                 │
│  (工具注册, 执行, 权限控制, 输出截断)                       │
├─────────────────────────────────────────────────────────────┤
│                      Core 层                                 │
│  (数据库, 文件系统, Git, 事件系统)                          │
└─────────────────────────────────────────────────────────────┘
```

### 1.3 技术栈

- **语言**: TypeScript (Bun runtime)
- **框架**: Effect (函数式 Effect 系统)
- **数据库**: SQLite (Drizzle ORM)
- **UI**: SolidJS + OpenTUI
- **包管理**: Bun workspaces (Monorepo)

---

## 2. LLM 核心概念

### 2.1 LLM 抽象层设计

OpenCode 的 LLM 层是**Schema-first**的设计，核心思想是：

```typescript
// 一个类型化的请求，统一所有 provider
const request = LLM.request({
  model: OpenAI.configure({ apiKey }).responses("gpt-4o-mini"),
  system: "You are concise.",
  prompt: "Say hello in one short sentence.",
  generation: { maxTokens: 40 },
})

// 流式响应，provider 无关的事件流
const response = yield* LLMClient.stream(request)
```

**面试考察点**:
- 为什么要 Schema-first？（类型安全、provider 无关、易于测试）
- 如何处理不同 provider 的差异？（Protocol 层适配）

### 2.2 Provider 适配模式

OpenCode 使用**四轴分解**来处理不同 provider：

```typescript
Route = {
  protocol: Protocol,    // 语义 API 契约（body 构建、流解析）
  endpoint: Endpoint,    // URL 构建
  auth: Auth,           // 认证
  framing: Framing      // 字节流 -> 帧的转换
}
```

**关键设计决策**:
- DeepSeek、TogetherAI 等都复用 `OpenAIChat.protocol`，只需 5-15 行配置
- Bug 修复在一个 protocol 中，自动传播到所有使用该 protocol 的 provider

**面试深挖点**:
```
Q: 为什么要把 Protocol 和 Provider 分离？
A: 因为很多 "OpenAI-compatible" 的 provider 共享相同的 API 协议，
   只是 endpoint、auth 不同。分离后可以复用协议实现。

Q: 如何添加一个新的 provider？
A: 通常是 5-15 行 Route.make() 调用，指定 protocol、endpoint、auth、framing
```

### 2.3 流式处理与事件系统

```typescript
// LLM 事件类型
type LLMEvent = 
  | { type: "text-delta", text: string }
  | { type: "tool-call", id: string, name: string, input: unknown }
  | { type: "tool-result", id: string, result: unknown }
  | { type: "finish", reason: string, usage: Usage }
  | { type: "reasoning", text: string }  // 推理过程
```

**面试考察点**:
- 流式处理的实现原理（SSE、WebSocket）
- 如何处理 tool call 的流式参数累积
- 错误处理和重试机制

### 2.4 Prompt Caching

OpenCode 实现了**智能缓存策略**：

```typescript
cache: "auto"  // 默认策略
// 在 3 个位置放置缓存断点：
// 1. 最后一个 tool 定义
// 2. 最后一个 system part
// 3. 最新的 user message
```

**面试深挖点**:
```
Q: 为什么在最后一个 user message 处缓存？
A: 在 tool-use 循环中，一个 user turn 会扩展成多个 assistant/tool round-trip，
   都共享该前缀。在该边界缓存可以让每个 intra-turn API 调用都命中缓存。

Q: 不同 provider 的缓存行为？
A: Anthropic: 显式 cache_control 标记
   Bedrock: 显式 cachePoint 标记
   OpenAI/Gemini: 隐式缓存，不需要标记
```

---

## 3. Agent 系统设计

### 3.1 Agent 类型与职责

OpenCode 有多个内置 Agent：

```typescript
agents = {
  build: "默认 agent，完全访问权限",
  plan: "只读 agent，禁止文件编辑",
  general: "通用子 agent，用于复杂搜索和多步任务",
  explore: "快速代码探索 agent，只读权限",
  compaction: "上下文压缩 agent",
  title: "生成会话标题",
  summary: "生成会话摘要"
}
```

**面试考察点**:
- 为什么需要多个 agent？（不同场景需要不同的权限和能力）
- Agent 的权限模型如何工作？（基于规则的权限系统）

### 3.2 Agent 权限系统

```typescript
permission = Permission.merge(
  defaults,           // 默认权限
  Permission.fromConfig({
    question: "allow",      // 允许提问
    plan_enter: "allow",    // 允许进入 plan 模式
    edit: { "*": "deny" },  // 禁止所有编辑
  }),
  user,               // 用户配置覆盖
)
```

**面试深挖点**:
```
Q: 权限系统的优先级如何？
A: 用户配置 > Agent 配置 > 默认配置

Q: 如何防止 Agent 执行危险操作？
A: 1. 权限规则匹配（通配符模式）
   2. 用户确认机制（ask 权限）
   3. 路径白名单/黑名单
```

### 3.3 Subagent 机制

```typescript
// TaskTool 实现子 agent 调度
const TaskTool = Tool.define("task", Effect.gen(function* () {
  return {
    execute: (params, ctx) => Effect.gen(function* () {
      // 创建子会话
      const session = yield* sessions.create(...)
      // 在子会话中执行任务
      yield* sessions.prompt({ sessionID, prompt: params.prompt })
      // 返回结果
      return { output: result }
    })
  }
}))
```

**面试考察点**:
- 子 agent 如何与父 agent 通信？
- 如何处理子 agent 的错误？
- 后台任务的实现原理

---

## 4. Code Agent 实现细节

### 4.1 代码编辑工具

OpenCode 的 Edit 工具是核心，实现来自 Cline 和 Gemini CLI：

```typescript
const Parameters = Schema.Struct({
  filePath: Schema.String,      // 文件路径
  oldString: Schema.String,     // 要替换的文本
  newString: Schema.String,     // 替换后的文本
  replaceAll: Schema.optional(Schema.Boolean),  // 替换所有
})
```

**关键实现细节**:
1. **行尾处理**: 自动检测和转换 `\r\n` vs `\n`
2. **BOM 处理**: 正确处理 UTF-8 BOM
3. **并发控制**: 使用 Semaphore 防止同一文件的并发编辑
4. **Diff 生成**: 使用 `diff` 库生成 unified diff

**面试深挖点**:
```
Q: 为什么不用正则表达式替换？
A: 正则无法处理多行匹配，且容易出错。精确字符串匹配更可靠。

Q: 如何处理 LLM 生成的不精确匹配？
A: 1. 行尾标准化
   2. 前后空白处理
   3. 模糊匹配（如果精确匹配失败）

Q: 如何保证编辑的原子性？
A: 使用 Semaphore 串行化同一文件的编辑操作
```

### 4.2 Shell 工具

```typescript
// Shell 命令解析使用 tree-sitter
const ShellTool = Tool.define("bash", Effect.gen(function* () {
  const parser = yield* TreeSitter
  return {
    execute: (params, ctx) => Effect.gen(function* () {
      // 解析命令，提取文件路径
      const scan = parseCommand(params.command)
      // 检查权限
      yield* checkPermissions(scan)
      // 执行命令
      const result = yield* execCommand(params.command)
      return { output: result }
    })
  }
}))
```

**面试考察点**:
- 如何安全地执行用户提供的 shell 命令？
- 如何提取命令中的文件路径用于权限检查？

### 4.3 文件搜索工具

```typescript
// Grep 工具使用 ripgrep
const GrepTool = Tool.define("grep", Effect.gen(function* () {
  const ripgrep = yield* Ripgrep.Service
  return {
    execute: (params) => ripgrep.search(params.pattern, params.path)
  }
}))
```

**面试深挖点**:
```
Q: 为什么选择 ripgrep 而不是 grep？
A: 1. 更快（Rust 实现）
   2. 自动忽略 .gitignore 中的文件
   3. 支持 Unicode
   4. 跨平台一致性
```

---

## 5. 工具系统设计

### 5.1 工具定义与执行

```typescript
// 工具定义
interface ToolDef<Parameters, Result> {
  id: string
  description: string
  parameters: Schema<Parameters>  // Effect Schema
  execute(args: Parameters, ctx: ToolContext): Effect<Result>
}

// 工具执行流程
1. Schema 验证输入
2. 权限检查
3. 执行工具逻辑
4. 截断输出（如果太长）
5. 返回结果
```

**面试考察点**:
- 为什么用 Schema 而不是 JSON Schema？（类型安全、运行时验证）
- 工具输出截断策略是什么？

### 5.2 工具输出截断

```typescript
// 截断策略
const TRUNCATE_CONFIG = {
  maxLines: 1000,           // 最大行数
  maxBytes: 50_000,         // 最大字节数
  preserveStart: true,      // 保留开头
  preserveEnd: true,        // 保留结尾
}

// 截断后生成临时文件保存完整内容
if (truncated) {
  const outputPath = await saveToTempFile(fullContent)
  return { content: truncated, metadata: { outputPath } }
}
```

**面试深挖点**:
```
Q: 为什么要截断工具输出？
A: 1. 防止上下文窗口溢出
   2. 减少 token 消耗
   3. 提高响应速度

Q: 截断后如何保留完整内容？
A: 保存到临时文件，返回文件路径供后续引用
```

### 5.3 权限系统

```typescript
// 权限规则
type PermissionRule = {
  permission: string        // 工具名模式，如 "edit", "shell"
  pattern: string          // 资源模式，如 "*.ts", "/tmp/*"
  action: "allow" | "ask" | "deny"
}

// 权限检查流程
function checkPermission(tool: string, resource: string): Action {
  // 1. 从 agent 配置获取权限规则
  // 2. 按优先级匹配规则
  // 3. 返回 allow/ask/deny
}
```

**面试考察点**:
- 如何设计一个灵活的权限系统？
- 如何处理权限冲突？

---

## 6. 会话管理与上下文

### 6.1 会话生命周期

```typescript
// 会话状态机
SessionState = 
  | "created"      // 创建
  | "running"      // 运行中
  | "waiting"      // 等待用户输入
  | "compacting"   // 上下文压缩中
  | "completed"    // 完成
  | "error"        // 错误

// 会话持久化
Session = {
  id: SessionID
  messages: Message[]
  tokens: TokenUsage
  cost: number
  metadata: Record<string, unknown>
}
```

### 6.2 上下文组装

```typescript
// System Context 组装
SystemContext = {
  environment: string      // 环境信息（目录、平台、日期）
  instructions: string     // AGENTS.md 中的指令
  skills: string          // 可用技能说明
  mcp: string             // MCP 服务器指令
}

// 组装流程
function assembleContext(session, agent, model) {
  return [
    ...providerSpecificPrompt(model),  // provider 特定提示
    ...systemContext.environment,       // 环境信息
    ...agent.prompt,                   // agent 特定提示
    ...session.history,                // 会话历史
    ...currentPrompt,                  // 当前用户输入
  ]
}
```

**面试深挖点**:
```
Q: 如何管理上下文窗口？
A: 1. Token 计数
   2. 自动压缩（当超过阈值时）
   3. 智能截断（保留最近的对话）

Q: 压缩策略是什么？
A: 1. 保留最近 N 轮对话
   2. 生成历史摘要
   3. 用摘要替换旧对话
```

### 6.3 Context Epoch

这是一个重要的概念：

```typescript
// Context Epoch 管理
ContextEpoch = {
  baseline: string        // 不可变的基线系统上下文
  snapshot: JSON          // 用于比较的快照
  messages: MidConversationSystemMessage[]  // 变更记录
}

// 生命周期
1. 初始化：创建完整基线
2. 变更检测：比较当前值与快照
3. 更新：生成 Mid-Conversation System Message
4. 重置：压缩后创建新 epoch
```

**面试考察点**:
- 为什么需要 Context Epoch？
- 如何处理上下文变更？

---

## 7. 面试重点与深挖点

### 7.1 LLM 应用层面

**可以问的问题**:
1. 如何设计一个 LLM 应用的架构？
2. 如何处理多个 LLM provider 的差异？
3. 如何优化 LLM 调用的成本和延迟？
4. 如何实现流式响应？
5. 如何处理 LLM 的错误和重试？

**OpenCode 的解决方案**:
```
1. 架构：分层设计，LLM 层与业务逻辑分离
2. Provider 差异：Protocol 层适配，四轴分解
3. 成本优化：Prompt caching、智能截断
4. 流式响应：SSE/WebSocket，事件驱动
5. 错误处理：重试策略、fallback 机制
```

### 7.2 Agent 设计层面

**可以问的问题**:
1. 如何设计一个 Agent 的权限系统？
2. 如何实现 Agent 的工具调用？
3. 如何处理 Agent 的上下文管理？
4. 如何实现多 Agent 协作？
5. 如何保证 Agent 的安全性？

**OpenCode 的解决方案**:
```
1. 权限系统：基于规则的权限匹配，支持通配符
2. 工具调用：Schema 验证 + 权限检查 + 执行 + 截断
3. 上下文管理：Context Epoch + 自动压缩
4. 多 Agent：TaskTool 实现子 agent 调度
5. 安全性：权限确认、路径白名单、输出截断
```

### 7.3 Code Agent 特定

**可以问的问题**:
1. 如何实现代码编辑工具？
2. 如何安全地执行 shell 命令？
3. 如何处理代码库的上下文？
4. 如何实现代码搜索？
5. 如何保证编辑的正确性？

**OpenCode 的解决方案**:
```
1. 代码编辑：精确字符串匹配 + diff 生成
2. Shell 执行：tree-sitter 解析 + 权限检查
3. 上下文：AGENTS.md + 项目结构分析
4. 代码搜索：ripgrep 集成
5. 正确性：Semaphore 并发控制 + 回滚机制
```

### 7.4 后训练相关

**可以问的问题**:
1. 如何收集训练数据？
2. 如何评估 Agent 的性能？
3. 如何优化 Agent 的提示词？
4. 如何处理 Agent 的错误？

**OpenCode 的设计**:
```
1. 数据收集：会话持久化、事件记录
2. 性能评估：工具调用成功率、用户满意度
3. 提示词优化：provider 特定的提示词模板
4. 错误处理：重试机制、用户确认
```

---

## 8. 可以主动介绍的项目亮点

### 8.1 架构设计亮点

1. **Effect 系统的使用**
   - 类型安全的错误处理
   - 依赖注入
   - 并发控制
   - 资源管理

2. **Schema-first 设计**
   - 运行时类型验证
   - 自动 JSON Schema 生成
   - 跨 provider 的类型一致性

3. **四轴分解的 Provider 适配**
   - 高度可复用
   - 易于扩展
   - Bug 修复自动传播

### 8.2 工程实践亮点

1. **测试策略**
   - 录制测试（Recorded Tests）
   - Fixture-first 测试
   - 避免 mock

2. **代码风格**
   - 函数式编程
   - 不可变数据
   - 早期返回

3. **性能优化**
   - Prompt caching
   - 智能截断
   - 懒加载

### 8.3 可以深挖的技术点

1. **流式处理的实现**
   ```
   - SSE 的工作原理
   - 如何处理 tool call 的流式参数
   - 错误恢复机制
   ```

2. **上下文管理**
   ```
   - Token 计数算法
   - 压缩策略
   - Context Epoch 的设计
   ```

3. **工具系统**
   ```
   - Schema 验证
   - 权限控制
   - 输出截断
   ```

4. **多 Agent 协作**
   ```
   - 子 agent 调度
   - 后台任务
   - 结果聚合
   ```

---

## 9. 面试中可能遇到的具体问题

### 9.1 设计类问题

**Q: 如何设计一个 Code Agent？**
```
A: 参考 OpenCode 的设计：
1. 分层架构：LLM 层、Agent 层、Tool 层、Core 层
2. 工具系统：Schema 验证 + 权限检查 + 执行
3. 上下文管理：自动压缩 + Context Epoch
4. 安全性：权限确认 + 路径白名单
```

**Q: 如何处理 LLM 的不确定性？**
```
A: 1. 重试机制（带退避）
   2. 用户确认（高风险操作）
   3. 输出验证（Schema 检查）
   4. 回滚机制（编辑操作）
```

**Q: 如何优化 LLM 调用成本？**
```
A: 1. Prompt caching（减少重复计算）
   2. 智能截断（减少 token 消耗）
   3. 批处理（合并多个请求）
   4. 缓存（避免重复调用）
```

### 9.2 实现类问题

**Q: 如何实现流式 tool call？**
```typescript
// 伪代码
function handleStreamToolCall(delta) {
  if (delta.type === "tool-call-start") {
    currentToolCall = { id: delta.id, name: delta.name, args: "" }
  } else if (delta.type === "tool-call-delta") {
    currentToolCall.args += delta.arguments
  } else if (delta.type === "tool-call-end") {
    // 解析完整的参数
    const args = JSON.parse(currentToolCall.args)
    // 执行工具
    executeTool(currentToolCall.name, args)
  }
}
```

**Q: 如何实现权限系统？**
```typescript
// 伪代码
function checkPermission(tool, resource, agent) {
  const rules = agent.permissionRules
  for (const rule of rules) {
    if (matchPattern(tool, rule.permission) &&
        matchPattern(resource, rule.pattern)) {
      return rule.action
    }
  }
  return "deny"  // 默认拒绝
}
```

**Q: 如何实现上下文压缩？**
```typescript
// 伪代码
function compactContext(messages, budget) {
  // 1. 计算当前 token 数
  const currentTokens = countTokens(messages)
  if (currentTokens <= budget) return messages

  // 2. 找到需要压缩的部分
  const oldMessages = messages.slice(0, -RECENT_TURNS)
  const recentMessages = messages.slice(-RECENT_TURNS)

  // 3. 生成摘要
  const summary = generateSummary(oldMessages)

  // 4. 组装新上下文
  return [
    { role: "system", content: summary },
    ...recentMessages
  ]
}
```

### 9.3 优化类问题

**Q: 如何减少 LLM 调用延迟？**
```
A: 1. 流式响应（减少首字节时间）
   2. Prompt caching（减少计算）
   3. 并行工具执行（减少等待）
   4. 预加载（预测下一步）
```

**Q: 如何提高工具调用成功率？**
```
A: 1. Schema 验证（提前发现错误）
   2. 重试机制（处理临时错误）
   3. 用户确认（高风险操作）
   4. 错误提示（帮助 LLM 自我纠正）
```

---

## 10. 总结

### 10.1 核心概念

1. **LLM 抽象层**: Schema-first，provider 无关
2. **Agent 系统**: 多 agent，权限控制
3. **工具系统**: 类型安全，权限检查
4. **上下文管理**: 自动压缩，Context Epoch
5. **流式处理**: 事件驱动，增量更新

### 10.2 面试准备建议

1. **理解架构**: 能画出系统架构图
2. **掌握细节**: 能解释关键实现
3. **准备案例**: 能举具体例子
4. **思考优化**: 能提出改进方案

### 10.3 可以展示的代码

1. LLM 层: `packages/llm/src/`
2. Agent 层: `packages/opencode/src/agent/`
3. 工具层: `packages/opencode/src/tool/`
4. 会话层: `packages/opencode/src/session/`

---

## 附录：关键文件索引

| 模块 | 文件路径 | 说明 |
|------|----------|------|
| LLM 核心 | `packages/llm/src/llm.ts` | LLM 请求构建 |
| Provider | `packages/llm/src/providers/` | Provider 实现 |
| Protocol | `packages/llm/src/protocols/` | 协议适配 |
| Agent | `packages/opencode/src/agent/agent.ts` | Agent 定义 |
| 工具注册 | `packages/opencode/src/tool/registry.ts` | 工具管理 |
| 会话 | `packages/opencode/src/session/session.ts` | 会话管理 |
| 编辑工具 | `packages/opencode/src/tool/edit.ts` | 代码编辑 |
| Shell | `packages/opencode/src/tool/shell.ts` | 命令执行 |

---

*本文档基于 OpenCode 项目源码分析生成，最后更新: 2026-06-27*
