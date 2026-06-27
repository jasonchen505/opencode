# LLM & Agent 技术面试五类问题应对指南

> 基于 OpenCode 项目深度分析，针对五类核心面试能力准备

---

## 目录

1. [底层原理理解](#1-底层原理理解)
2. [实验和方案验证能力](#2-实验和方案验证能力)
3. [问题定位能力](#3-问题定位能力)
4. [工程落地能力](#4-工程落地能力)
5. [业务与实际场景理解](#5-业务与实际场景理解)

---

## 1. 底层原理理解

> **核心要求**：讲清楚方法解决什么问题、存在哪些局限性、有哪些改进方法

### 1.1 LLM Provider 适配的四轴分解设计

**解决的问题**：
不同 LLM provider（OpenAI、Anthropic、Gemini 等）的 API 接口差异巨大，如果为每个 provider 单独实现，会导致：
1. 大量重复代码（每个 provider 300-400 行）
2. Bug 修复需要在多处同步
3. 新增 provider 成本高

**设计方案**：
```typescript
Route = {
  protocol: Protocol,    // 语义 API 契约
  endpoint: Endpoint,    // URL 构建
  auth: Auth,           // 认证
  framing: Framing      // 字节流 -> 帧转换
}
```

**为什么这么设计**：
1. **Protocol 层**处理请求体构建和流解析，这是 provider 间差异最大的部分
2. **Endpoint 层**处理 URL 构建，很多 "OpenAI-compatible" provider 共享相同协议但 endpoint 不同
3. **Auth 层**处理认证，不同 provider 使用 bearer token、API key、OAuth 等不同方式
4. **Framing 层**处理字节流解析，大部分用 SSE，但 Bedrock 用 AWS event-stream

**局限性**：
1. 新增协议类型需要实现完整的 Protocol（如 WebSocket）
2. 某些 provider 特有功能需要通过 `providerOptions` escape hatch
3. 流式 tool call 的参数累积逻辑需要在每个 Protocol 中重复实现

**改进方法**：
1. 抽象出通用的 tool call 流式累积器
2. 使用代码生成自动适配新 provider
3. 引入 provider capability 检测机制

**面试回答模板**：
```
"我们在设计 LLM 层时，发现不同 provider 的 API 差异主要体现在四个方面：
协议格式、URL 结构、认证方式、字节流解析。

所以我们采用了四轴分解的设计，将这四个关注点分离。
比如 DeepSeek、TogetherAI 等都复用 OpenAIChat.protocol，
只需 5-15 行配置就能接入新 provider。

这样做的好处是 Bug 修复能自动传播，但局限性是
新增协议类型（如 WebSocket）需要实现完整的 Protocol。

我们的改进方向是引入更智能的 provider capability 检测，
让系统能自动选择最佳的协议适配策略。"
```

### 1.2 Prompt Caching 策略

**解决的问题**：
LLM 调用成本高，特别是在 tool-use 循环中，一个 user turn 会扩展成多个 assistant/tool round-trip，每次都重新计算整个 prompt 会浪费大量 token。

**设计方案**：
```typescript
cache: "auto"  // 默认策略
// 在 3 个位置放置缓存断点：
// 1. 最后一个 tool 定义
// 2. 最后一个 system part
// 3. 最新的 user message
```

**为什么在最后一个 user message 处缓存**：
在 tool-use 循环中，一个 user turn 会扩展成多个 assistant/tool round-trip，这些 round-trip 共享相同的前缀。在该边界缓存可以让每个 intra-turn API 调用都命中缓存。

**局限性**：
1. 不同 provider 的缓存行为不一致
2. 缓存有最小 token 阈值要求
3. 缓存 TTL 需要根据使用模式调整

**改进方法**：
1. 动态调整缓存断点位置
2. 基于历史使用模式预测最佳缓存策略
3. 跨 provider 的缓存命中率监控

**面试回答模板**：
```
"Prompt caching 是我们优化成本的关键策略。

核心思路是在 prompt 的关键位置放置缓存断点，
特别是最后一个 user message 处，因为在 tool-use 循环中，
一个 user turn 会扩展成多个 assistant/tool round-trip，
它们共享相同的前缀。

数学上，Anthropic 的缓存读取成本只有 0.1x，
一次复用就能回本。

但局限性是不同 provider 的缓存行为不一致，
OpenAI 是隐式缓存，Anthropic 需要显式标记。

我们的改进方向是引入动态缓存断点，
基于历史使用模式预测最佳缓存位置。"
```

### 1.3 Context Epoch 与上下文管理

**解决的问题**：
长对话中，上下文会不断增长，需要管理：
1. 何时压缩上下文
2. 如何保持上下文一致性
3. 如何处理上下文变更

**设计方案**：
```typescript
ContextEpoch = {
  baseline: string        // 不可变的基线系统上下文
  snapshot: JSON          // 用于比较的快照
  messages: MidConversationSystemMessage[]  // 变更记录
}
```

**为什么需要 Context Epoch**：
1. **不可变基线**：保证 provider 缓存的一致性
2. **变更追踪**：记录上下文变更，支持审计
3. **原子更新**：上下文变更和事件提交原子化

**局限性**：
1. 基线重建需要重新计算整个上下文
2. 变更检测需要序列化比较
3. 跨 session 的上下文共享困难

**改进方法**：
1. 增量式基线更新
2. 更高效的变更检测算法
3. 引入上下文模板机制

**面试回答模板**：
```
"上下文管理是 Code Agent 的核心挑战之一。

我们设计了 Context Epoch 机制，
每个 epoch 有一个不可变的基线系统上下文，
用于 provider 缓存的一致性。

当上下文变更时，我们会生成 Mid-Conversation System Message，
记录变更内容，而不是直接修改基线。

这样做的好处是支持审计和回滚，
但局限性是基线重建需要重新计算。

我们的改进方向是引入增量式基线更新，
只更新变更的部分，而不是重建整个上下文。"
```

### 1.4 工具输出截断策略

**解决的问题**：
工具执行结果可能非常大（如读取大文件、执行复杂命令），如果全部放入上下文，会导致：
1. 上下文窗口溢出
2. Token 消耗过高
3. 响应速度变慢

**设计方案**：
```typescript
// 截断策略
const TRUNCATE_CONFIG = {
  maxLines: 2000,           // 最大行数
  maxBytes: 50 * 1024,      // 最大字节数
  preserveStart: true,      // 保留开头
  preserveEnd: true,        // 保留结尾
}

// 截断后生成临时文件保存完整内容
if (truncated) {
  const outputPath = await saveToTempFile(fullContent)
  return { content: truncated, metadata: { outputPath } }
}
```

**为什么保留开头和结尾**：
1. 开头通常包含文件头部信息、import 语句等
2. 结尾通常包含返回值、总结等
3. 中间部分可以通过 offset/limit 按需读取

**局限性**：
1. 截断可能丢失关键信息
2. 临时文件需要定期清理
3. 不同工具可能需要不同的截断策略

**改进方法**：
1. 基于语义的智能截断（保留函数签名、类定义等）
2. 工具自定义截断策略
3. 引入摘要机制代替简单截断

---

## 2. 实验和方案验证能力

> **核心要求**：怎么证明方案有效，追问实验细节

### 2.1 如何验证 Provider 适配的正确性

**验证方法**：
```typescript
// 录制测试（Recorded Tests）
const recorded = recordedTests({ prefix: "openai-chat", requires: ["OPENAI_API_KEY"] })

recorded.effect("streams text", () =>
  Effect.gen(function* () {
    // 测试流式文本生成
  }),
)

recorded.effect("handles tool calls", () =>
  Effect.gen(function* () {
    // 测试工具调用
  }),
)
```

**实验细节**：
1. **录制模式**：`RECORD=true` 录制真实 API 响应
2. **回放模式**：默认使用录制的响应，无需真实 API
3. **匹配策略**：验证 method、URL、headers、body 的一致性
4. **多步骤支持**：一个 cassette 文件支持多个请求/响应对

**如何证明有效性**：
1. 覆盖所有 provider 的核心功能
2. 边界情况测试（错误处理、超时、重试）
3. 性能基准测试（延迟、吞吐量）

**面试回答模板**：
```
"我们使用录制测试来验证 Provider 适配的正确性。

核心思路是录制真实的 API 响应，然后在测试中回放。
这样既能保证测试的真实性，又不需要每次都调用真实 API。

具体做法是：
1. 设置 RECORD=true 录制响应
2. 验证请求的 method、URL、headers、body
3. 支持多步骤测试（如 tool-use 循环）

有效性证明：
- 覆盖所有 provider 的核心功能
- 包含边界情况（错误、超时、重试）
- 性能基准测试"
```

### 2.2 如何验证工具执行的正确性

**验证方法**：
```typescript
// 工具执行测试
describe("EditTool", () => {
  it("should edit file correctly", async () => {
    // 准备测试文件
    await writeFile("test.ts", "const a = 1")
    
    // 执行编辑
    const result = await editTool.execute({
      filePath: "test.ts",
      oldString: "const a = 1",
      newString: "const a = 2"
    })
    
    // 验证结果
    expect(result.output).toContain("edited successfully")
    expect(await readFile("test.ts")).toBe("const a = 2")
  })
  
  it("should handle line endings correctly", async () => {
    // 测试 \r\n vs \n 处理
  })
  
  it("should handle BOM correctly", async () => {
    // 测试 UTF-8 BOM 处理
  })
})
```

**实验细节**：
1. **文件系统隔离**：每个测试使用独立的临时目录
2. **边界情况**：空文件、大文件、特殊字符
3. **并发测试**：多个编辑操作同时进行
4. **回滚验证**：编辑失败后文件状态

**面试回答模板**：
```
"工具执行的验证我们采用多层次测试策略：

1. 单元测试：验证工具的核心逻辑
   - 文件编辑的精确匹配
   - 行尾处理（\r\n vs \n）
   - BOM 处理

2. 集成测试：验证工具与系统的交互
   - 文件系统操作
   - 权限检查
   - 并发控制

3. 端到端测试：验证完整的工具调用流程
   - LLM 生成工具调用
   - 工具执行
   - 结果返回

有效性证明：
- 覆盖所有边界情况
- 并发安全性验证
- 错误处理验证"
```

### 2.3 如何验证上下文压缩的效果

**验证方法**：
```typescript
// 压缩效果测试
describe("Compaction", () => {
  it("should compress long conversation", async () => {
    // 准备长对话
    const messages = generateLongConversation(100)
    
    // 执行压缩
    const compacted = await compaction.compact(messages)
    
    // 验证压缩效果
    expect(compacted.summary).toBeDefined()
    expect(compacted.recentMessages.length).toBeLessThan(messages.length)
    expect(countTokens(compacted)).toBeLessThan(countTokens(messages) * 0.5)
  })
  
  it("should preserve important information", async () => {
    // 验证关键信息保留
  })
  
  it("should handle multiple compressions", async () => {
    // 验证多次压缩的累积效果
  })
})
```

**实验细节**：
1. **Token 计数**：使用 tiktoken 精确计算
2. **信息保留率**：验证关键信息是否保留
3. **压缩比**：不同对话长度的压缩效果
4. **性能**：压缩操作的耗时

**面试回答模板**：
```
"上下文压缩的验证我们关注三个维度：

1. 压缩比：目标是压缩到原来的 30-50%
2. 信息保留率：关键信息（如代码、决策）必须保留
3. 性能：压缩操作耗时控制在 1 秒内

具体做法：
- 使用 tiktoken 精确计算 token 数
- 设计信息保留率指标（如关键词命中率）
- A/B 测试对比压缩前后的 LLM 响应质量

有效性证明：
- 压缩后 LLM 响应质量不下降
- Token 消耗降低 50% 以上
- 用户满意度不降低"
```

---

## 3. 问题定位能力

> **核心要求**：排查问题的方法论，优化思路与解决方案

### 3.1 LLM 调用失败的排查

**问题现象**：
LLM 调用返回错误或超时

**排查流程**：
```
1. 检查错误类型
   ├─ 4xx 错误：请求格式问题
   ├─ 5xx 错误：Provider 服务问题
   └─ 超时：网络或 Provider 问题

2. 检查请求内容
   ├─ Token 数是否超限
   ├─ 工具定义是否正确
   └─ 消息格式是否正确

3. 检查 Provider 状态
   ├─ API key 是否有效
   ├─ 配额是否用尽
   └─ 服务是否可用
```

**关键代码**：
```typescript
// 重试机制
export function retryable(error: Err, provider: string) {
  // context overflow 错误不重试
  if (SessionV1.ContextOverflowError.isInstance(error)) return undefined
  
  if (SessionV1.APIError.isInstance(error)) {
    const status = error.data.statusCode
    // 5xx 错误是瞬时失败，应该重试
    if (!error.data.isRetryable && !(status >= 500)) return undefined
    
    // 解析重试延迟
    const retryAfterMs = error.data.responseHeaders?.["retry-after-ms"]
    if (retryAfterMs) return cap(Number.parseFloat(retryAfterMs))
  }
}
```

**优化思路**：
1. **指数退避**：避免重试风暴
2. **重试预算**：限制最大重试次数
3. **熔断机制**：连续失败后停止重试
4. **降级策略**：切换到备用 provider

**面试回答模板**：
```
"LLM 调用失败的排查我们有标准化流程：

1. 首先看错误类型：
   - 4xx 是请求问题，检查 token 数、格式
   - 5xx 是 Provider 问题，检查服务状态
   - 超时是网络问题，检查连通性

2. 然后检查请求内容：
   - Token 数是否超限
   - 工具定义是否正确
   - 消息格式是否正确

3. 最后检查 Provider 状态：
   - API key 是否有效
   - 配额是否用尽
   - 服务是否可用

优化思路：
- 指数退避避免重试风暴
- 重试预算限制最大重试次数
- 熔断机制停止连续失败的重试
- 降级策略切换备用 provider"
```

### 3.2 工具执行失败的排查

**问题现象**：
工具执行返回错误或超时

**排查流程**：
```
1. 检查工具输入
   ├─ 参数是否符合 Schema
   ├─ 文件路径是否存在
   └─ 权限是否足够

2. 检查执行过程
   ├─ 是否有异常抛出
   ├─ 是否超时
   └─ 是否被中断

3. 检查输出
   ├─ 输出是否被截断
   ├─ 输出格式是否正确
   └─ 是否有副作用
```

**关键代码**：
```typescript
// 工具执行错误处理
const failToolCall = Effect.fn("SessionProcessor.failToolCall")(function* (toolCallID: string, error: unknown) {
  const match = yield* readToolCall(toolCallID)
  if (!match || match.part.state.status !== "running") return false
  
  yield* session.updatePart({
    ...match.part,
    state: {
      status: "error",
      input: match.part.state.input,
      error: errorMessage(error),
      time: { start: match.part.state.time.start, end: Date.now() },
    },
  })
  
  // 权限拒绝导致阻塞
  if (error instanceof PermissionV1.RejectedError || error instanceof Question.RejectedError) {
    ctx.blocked = ctx.shouldBreak
  }
  
  yield* settleToolCall(toolCallID)
  return true
})
```

**优化思路**：
1. **输入验证**：提前发现参数错误
2. **超时控制**：避免长时间阻塞
3. **权限检查**：提前拒绝无权限操作
4. **错误提示**：帮助 LLM 自我纠正

**面试回答模板**：
```
"工具执行失败的排查我们采用分层策略：

1. 输入层：验证参数是否符合 Schema
   - 类型检查
   - 范围检查
   - 格式检查

2. 执行层：监控执行过程
   - 超时检测（默认 30 秒）
   - 异常捕获
   - 中断处理

3. 输出层：验证输出结果
   - 输出截断检测
   - 格式验证
   - 副作用检查

优化思路：
- 提前验证输入，减少无效执行
- 设置合理超时，避免阻塞
- 详细的错误提示，帮助 LLM 自我纠正"
```

### 3.3 上下文溢出的排查

**问题现象**：
Provider 返回 context overflow 错误

**排查流程**：
```
1. 检查 Token 数
   ├─ 系统提示词 token 数
   ├─ 历史消息 token 数
   ├─ 工具定义 token 数
   └─ 当前输入 token 数

2. 检查压缩状态
   ├─ 是否已触发压缩
   ├─ 压缩是否成功
   └─ 压缩效果如何

3. 检查 Provider 限制
   ├─ 上下文窗口大小
   ├─ 输出 token 限制
   └─ 是否有特殊限制
```

**关键代码**：
```typescript
// 溢出检测
export function isOverflow(input: {
  tokens: SessionV1.Assistant["tokens"]
  model: Provider.Model
}) {
  const contextWindow = model.contextWindow
  const outputTokens = model.maxTokens ?? 4096
  const totalTokens = input.tokens.input + input.tokens.output
  
  // 预留 10% 的缓冲区
  return totalTokens > contextWindow * 0.9 - outputTokens
}

// 自动压缩
if (isOverflow(tokens, model)) {
  yield* compaction.create({
    sessionID,
    agent: agent.name,
    model,
    auto: true,
    overflow: true,
  })
}
```

**优化思路**：
1. **预防性压缩**：在溢出前主动压缩
2. **智能截断**：保留重要信息，截断冗余信息
3. **分批处理**：将大任务拆分成小任务
4. **上下文预算**：为不同部分分配 token 预算

**面试回答模板**：
```
"上下文溢出的排查我们有完整的诊断流程：

1. 首先检查 Token 数分布：
   - 系统提示词占比
   - 历史消息占比
   - 工具定义占比
   - 当前输入占比

2. 然后检查压缩状态：
   - 是否已触发压缩
   - 压缩是否成功
   - 压缩效果如何

3. 最后检查 Provider 限制：
   - 上下文窗口大小
   - 输出 token 限制

优化思路：
- 预防性压缩：在溢出前主动压缩
- 智能截断：保留重要信息
- 分批处理：大任务拆分成小任务
- 上下文预算：为不同部分分配 token 预算"
```

---

## 4. 工程落地能力

> **核心要求**：理论结合实际，部署、稳定性、监控

### 4.1 如何部署 LLM 应用

**部署架构**：
```
┌─────────────────────────────────────────────────────────────┐
│                      负载均衡层                              │
│  (Nginx/Cloudflare)                                         │
├─────────────────────────────────────────────────────────────┤
│                      API Server 层                           │
│  (多个实例，无状态)                                         │
├─────────────────────────────────────────────────────────────┤
│                      数据层                                  │
│  (SQLite/PostgreSQL)                                        │
├─────────────────────────────────────────────────────────────┤
│                      存储层                                  │
│  (S3/本地存储)                                              │
└─────────────────────────────────────────────────────────────┘
```

**关键决策**：
1. **无状态设计**：API Server 不保存状态，便于水平扩展
2. **本地数据库**：SQLite 用于开发，PostgreSQL 用于生产
3. **会话持久化**：会话状态保存在数据库，支持断线重连

**部署细节**：
```typescript
// 服务器配置
const server = {
  port: 4096,
  host: "0.0.0.0",
  cors: {
    origin: ["https://opencode.ai", "http://localhost:5173"],
    credentials: true,
  },
  rateLimit: {
    windowMs: 15 * 60 * 1000, // 15 分钟
    max: 100, // 每个 IP 最多 100 个请求
  },
}
```

**面试回答模板**：
```
"LLM 应用的部署我们采用分层架构：

1. 负载均衡层：使用 Nginx 或 Cloudflare
   - 请求路由
   - SSL 终止
   - 限流

2. API Server 层：多个无状态实例
   - 水平扩展
   - 健康检查
   - 自动重启

3. 数据层：SQLite/PostgreSQL
   - 会话持久化
   - 事件存储
   - 配置管理

4. 存储层：S3/本地存储
   - 文件存储
   - 临时文件
   - 备份

关键决策：
- 无状态设计便于水平扩展
- 会话持久化支持断线重连
- 分层架构便于独立扩展"
```

### 4.2 如何保证系统稳定性

**稳定性策略**：
```typescript
// 1. 限流
const rateLimit = {
  windowMs: 15 * 60 * 1000,
  max: 100,
  message: "Too many requests, please try again later",
}

// 2. 熔断
const circuitBreaker = {
  timeout: 3000,
  errorThresholdPercentage: 50,
  resetTimeout: 30000,
}

// 3. 重试
const retry = {
  maxRetries: 3,
  initialDelay: 1000,
  maxDelay: 10000,
  backoffFactor: 2,
}

// 4. 超时
const timeout = {
  request: 30000,
  tool: 60000,
  session: 300000,
}
```

**监控指标**：
```typescript
// 监控指标
const metrics = {
  // 性能指标
  requestLatency: histogram("request_latency_ms"),
  toolExecutionTime: histogram("tool_execution_ms"),
  llmLatency: histogram("llm_latency_ms"),
  
  // 错误指标
  errorRate: counter("error_rate"),
  retryCount: counter("retry_count"),
  timeoutCount: counter("timeout_count"),
  
  // 业务指标
  activeSessions: gauge("active_sessions"),
  tokenUsage: counter("token_usage"),
  costEstimate: counter("cost_estimate"),
}
```

**面试回答模板**：
```
"系统稳定性我们采用四层防护：

1. 限流：防止恶意请求
   - 基于 IP 的限流
   - 基于用户的限流
   - 基于 API key 的限流

2. 熔断：防止级联故障
   - 错误率超过 50% 触发熔断
   - 30 秒后尝试恢复
   - 半开状态逐步放量

3. 重试：处理瞬时故障
   - 指数退避
   - 最大重试次数限制
   - 重试预算

4. 超时：防止长时间阻塞
   - 请求超时 30 秒
   - 工具执行超时 60 秒
   - 会话超时 5 分钟

监控指标：
- 性能指标：延迟、吞吐量
- 错误指标：错误率、重试次数
- 业务指标：活跃会话、token 使用量"
```

### 4.3 如何保证数据回滚与监控

**数据回滚策略**：
```typescript
// 快照机制
const snapshot = {
  // 编辑前创建快照
  beforeEdit: async (filePath: string) => {
    const content = await readFile(filePath)
    await saveSnapshot(filePath, content)
  },
  
  // 回滚到快照
  rollback: async (filePath: string) => {
    const snapshot = await getSnapshot(filePath)
    if (snapshot) {
      await writeFile(filePath, snapshot.content)
    }
  },
}

// 会话回滚
const sessionRollback = {
  // 回滚到指定消息
  rollbackToMessage: async (sessionID: string, messageID: string) => {
    const session = await getSession(sessionID)
    const messageIndex = session.messages.findIndex(m => m.id === messageID)
    if (messageIndex >= 0) {
      session.messages = session.messages.slice(0, messageIndex + 1)
      await updateSession(session)
    }
  },
}
```

**监控体系**：
```typescript
// 监控告警
const alerts = {
  // 错误率告警
  errorRate: {
    threshold: 0.1, // 10%
    window: "5m",
    action: "notify",
  },
  
  // 延迟告警
  latency: {
    threshold: 5000, // 5 秒
    window: "1m",
    action: "notify",
  },
  
  // 成本告警
  cost: {
    threshold: 100, // $100
    window: "1d",
    action: "notify",
  },
}
```

**面试回答模板**：
```
"数据回滚与监控我们采用完整的方案：

1. 数据回滚：
   - 文件编辑前创建快照
   - 会话支持回滚到指定消息
   - 数据库支持事务回滚

2. 监控体系：
   - 性能监控：延迟、吞吐量
   - 错误监控：错误率、异常类型
   - 业务监控：活跃会话、token 使用量
   - 成本监控：API 调用成本

3. 告警机制：
   - 错误率超过 10% 告警
   - 延迟超过 5 秒告警
   - 成本超过预算告警

4. 自动恢复：
   - 自动重试瞬时故障
   - 自动降级高负载服务
   - 自动切换备用 provider"
```

---

## 5. 业务与实际场景理解

> **核心要求**：场景价值、用户关心什么、上线成本、优先级

### 5.1 Code Agent 的目标用户与场景

**目标用户**：
1. **个人开发者**：需要快速原型开发、代码重构
2. **小型团队**：需要代码审查、知识共享
3. **大型团队**：需要代码规范、自动化测试

**核心场景**：
```
1. 代码生成
   - 新功能开发
   - 代码重构
   - 测试用例生成

2. 代码理解
   - 代码库探索
   - 文档生成
   - 代码审查

3. 问题排查
   - Bug 定位
   - 性能优化
   - 错误分析

4. 自动化
   - CI/CD 集成
   - 代码规范检查
   - 依赖更新
```

**用户关心什么**：
1. **准确性**：生成的代码是否正确
2. **速度**：响应时间是否可接受
3. **成本**：API 调用成本是否可控
4. **安全性**：是否会有安全风险

**面试回答模板**：
```
"Code Agent 的目标用户是开发者，核心场景包括：

1. 代码生成：新功能、重构、测试
2. 代码理解：探索、文档、审查
3. 问题排查：Bug 定位、性能优化
4. 自动化：CI/CD、规范检查

用户最关心的是：
1. 准确性：代码是否正确
2. 速度：响应是否快速
3. 成本：是否可控
4. 安全性：是否有风险

我们的设计优先级是：
1. 准确性 > 速度 > 成本 > 安全性"
```

### 5.2 上线成本分析

**成本构成**：
```typescript
const costBreakdown = {
  // API 调用成本
  apiCalls: {
    inputTokens: 0.01, // $0.01 / 1K tokens
    outputTokens: 0.03, // $0.03 / 1K tokens
    averageTokensPerSession: 10000,
    averageCostPerSession: 0.2, // $0.20
  },
  
  // 基础设施成本
  infrastructure: {
    server: 50, // $50 / 月
    database: 20, // $20 / 月
    storage: 10, // $10 / 月
  },
  
  // 人力成本
  personnel: {
    development: 10000, // $10,000 / 月
    maintenance: 2000, // $2,000 / 月
  },
}
```

**成本优化策略**：
```typescript
const costOptimization = {
  // 1. Prompt caching
  promptCaching: {
    savings: "50-70%",
    implementation: "自动缓存断点",
  },
  
  // 2. 智能截断
  smartTruncation: {
    savings: "30-50%",
    implementation: "基于语义的截断",
  },
  
  // 3. 批处理
  batchProcessing: {
    savings: "20-30%",
    implementation: "合并多个请求",
  },
  
  // 4. 缓存
  caching: {
    savings: "40-60%",
    implementation: "响应缓存",
  },
}
```

**面试回答模板**：
```
"上线成本主要包括三部分：

1. API 调用成本：
   - 平均每个会话 $0.20
   - 主要消耗在 input tokens
   - 通过 prompt caching 可降低 50-70%

2. 基础设施成本：
   - 服务器 $50/月
   - 数据库 $20/月
   - 存储 $10/月

3. 人力成本：
   - 开发 $10,000/月
   - 维护 $2,000/月

成本优化策略：
1. Prompt caching：降低 50-70%
2. 智能截断：降低 30-50%
3. 批处理：降低 20-30%
4. 响应缓存：降低 40-60%

优先级：
1. Prompt caching（收益最高）
2. 智能截断（实现简单）
3. 响应缓存（效果明显）"
```

### 5.3 如果资源有限，优先优化什么

**优先级排序**：
```
1. P0：准确性
   - 代码生成正确率
   - 工具执行成功率
   - 错误处理完善度

2. P1：成本优化
   - Prompt caching
   - 智能截断
   - 批处理

3. P2：用户体验
   - 响应速度
   - 错误提示
   - 文档完善

4. P3：功能扩展
   - 新 provider 支持
   - 新工具集成
   - 高级功能
```

**具体优化方案**：
```typescript
const optimizationPlan = {
  // 第一阶段：准确性（1-2 周）
  phase1: {
    focus: "准确性",
    tasks: [
      "完善输入验证",
      "改进错误处理",
      "增加边界情况测试",
    ],
    expectedImprovement: "成功率从 80% 提升到 95%",
  },
  
  // 第二阶段：成本优化（2-4 周）
  phase2: {
    focus: "成本优化",
    tasks: [
      "实现 prompt caching",
      "优化截断策略",
      "实现批处理",
    ],
    expectedImprovement: "成本降低 50%",
  },
  
  // 第三阶段：用户体验（4-8 周）
  phase3: {
    focus: "用户体验",
    tasks: [
      "优化响应速度",
      "改进错误提示",
      "完善文档",
    ],
    expectedImprovement: "用户满意度提升 30%",
  },
}
```

**面试回答模板**：
```
"如果资源有限，我的优先级是：

1. P0：准确性（1-2 周）
   - 完善输入验证
   - 改进错误处理
   - 增加边界情况测试
   - 目标：成功率从 80% 提升到 95%

2. P1：成本优化（2-4 周）
   - 实现 prompt caching
   - 优化截断策略
   - 实现批处理
   - 目标：成本降低 50%

3. P2：用户体验（4-8 周）
   - 优化响应速度
   - 改进错误提示
   - 完善文档
   - 目标：用户满意度提升 30%

理由：
1. 准确性是基础，用户不会容忍错误的代码
2. 成本优化直接影响商业可行性
3. 用户体验影响用户留存

资源分配：
- 70% 资源用于准确性
- 20% 资源用于成本优化
- 10% 资源用于用户体验"
```

---

## 总结：面试应对策略

### 1. 底层原理理解

**回答框架**：
```
1. 解决什么问题
2. 为什么这么设计
3. 局限性是什么
4. 有哪些改进方法
```

**关键点**：
- 理解设计决策的权衡
- 能够解释 trade-off
- 提出有建设性的改进方案

### 2. 实验和方案验证能力

**回答框架**：
```
1. 验证方法是什么
2. 实验细节是什么
3. 如何证明有效性
4. 有哪些边界情况
```

**关键点**：
- 有完整的测试策略
- 能够设计合理的实验
- 关注边界情况和异常

### 3. 问题定位能力

**回答框架**：
```
1. 问题现象是什么
2. 排查流程是什么
3. 根本原因是什么
4. 解决方案是什么
```

**关键点**：
- 有系统化的排查方法
- 能够定位根本原因
- 提出预防性措施

### 4. 工程落地能力

**回答框架**：
```
1. 部署架构是什么
2. 如何保证稳定性
3. 如何监控告警
4. 如何回滚恢复
```

**关键点**：
- 有完整的部署方案
- 关注稳定性和可用性
- 有监控和告警机制

### 5. 业务与实际场景理解

**回答框架**：
```
1. 目标用户是谁
2. 核心场景是什么
3. 用户关心什么
4. 优先级是什么
```

**关键点**：
- 理解业务价值
- 能够权衡优先级
- 关注成本和收益

---

*本文档基于 OpenCode 项目深度分析生成，最后更新: 2026-06-27*
