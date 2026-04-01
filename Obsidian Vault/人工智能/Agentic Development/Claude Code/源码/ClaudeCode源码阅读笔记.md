
## 整体技术栈

  - Runtime: Bun（而非 Node.js），利用 bun:bundle 特性实现编译时 Dead Code Elimination
  - UI: React + Ink（终端 TUI）
  - CLI 解析: Commander.js
  - API 客户端: @anthropic-ai/sdk（官方 Anthropic SDK）
  ---
## 启动流程（main.tsx）

启动时有意设计了并行预取来减少冷启动延迟：

  main.tsx 加载顺序（有意的副作用顺序）:
    1. profileCheckpoint('main_tsx_entry')     ← 性能采样点
    2. startMdmRawRead()                        ← 并行读取 MDM 策略（macOS plutil/Windows reg query）
    3. startKeychainPrefetch()                  ← 并行预读 keychain（OAuth token + legacy API key）
    4. 重模块懒加载（OpenTelemetry, gRPC, GrowthBook...）
    5. Commander.js 解析 CLI 参数
    6. 初始化 React/Ink 渲染器

  ---
## LLM 交互核心（最重要部分）

### 调用链路：
用户输入
    └→ QueryEngine.[[submitMessage()]]          [QueryEngine.ts]
         └→ fetchSystemPromptParts()         构建 system prompt
         └→ query()                          [query.ts]
              └→ [[queryLoop()]]                ← 核心循环（while(true)）
                   ├─ 上下文压缩处理:
                   │   ├─ snipCompactIfNeeded()    历史裁剪
                   │   ├─ microcompact()            微压缩
                   │   ├─ contextCollapse()         上下文折叠
                   │   └─ autocompact()             自动压缩（触发阈值时 LLM 摘要）
                   │
                   ├─ deps.callModel(...)    ← 实际 API 调用（services/api/claude.ts）
                   │   └→ Anthropic Beta Messages API（流式）
                   │
                   ├─ 流式处理 assistant 消息（streaming for await）
                   │
                   ├─ 发现 tool_use blocks → needsFollowUp = true
                   │
                   ├─ runTools()             [services/tools/toolOrchestration.ts]
                   │   ├─ 只读工具: 并发执行（最多10个，CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY）
                   │   └─ 写操作工具: 串行执行
                   │
                   └─ 将 tool_result 追加到 messages → 继续循环

### 完整架构图（从用户输入开始）：
```            
用户在终端输入文字 / 按回车                                                               
  ┌─────────────────────────────────────────────────────────┐
  │                  两条入口路径                             │
  └────────────────┬───────────────────┬────────────────────┘
                   │                   │
       交互式 REPL  │                   │ 无头/SDK 模式（-p 参数）
                   ▼                   ▼
           REPL.tsx                main.tsx
           onSubmit()              └→ cli/print.ts
             │                         └→ ask()  [QueryEngine.ts]
             ▼
      handlePromptSubmit.ts
      executeUserInput()
             │
             ├─ 解析 slash 命令、图片、粘贴内容
             ├─ 构建 QueuedCommand[]
             │
             ▼
      processUserInput()            ← 同样调用这里
      [processUserInput.ts]
             │
             ├─ slash 命令 → 本地执行，shouldQuery=false → 直接结束
             ├─ 图片 → resize/base64
             ├─ 执行 UserPromptSubmit hooks
             └─ 返回 { messages, shouldQuery }
                   │
       ┌───────────┴────────────────┐
       │ shouldQuery=false          │ shouldQuery=true
       ▼                            ▼
    直接返回结果             REPL: onQuery() → onQueryImpl()
    (slash 命令输出)         SDK:  QueryEngine.submitMessage()
                                    │
                     ┌──────────────┴───────────────────┐
                     │  两者都调用 query() [query.ts]    │
                     └──────────────┬───────────────────┘
                                    │
                                    ▼
                             queryLoop()  ← while(true) 循环
                                    │
                    ┌───────────────┼───────────────────┐
                    │               │                   │
                    ▼               ▼                   ▼
             上下文压缩        callModel()          runTools()
             (snip/micro/     [claude.ts]          [toolOrchestration.ts]
              autocompact)    SSE 流式 API           并发/串行执行工具
                                    │
                            for await (part of stream)
                            content_block_delta...
                                    │
                            发现 tool_use block?
                            ├─ 是 → runTools() → 追加 tool_result → 继续循环
                            └─ 否 → 循环结束，返回结果

```

  callModel（services/api/claude.ts）关键参数，实际调用 Anthropic Beta Messages API（流式）
```
  anthropicClient.beta.messages.stream({
    model: currentModel,           // claude-sonnet-4-6 / claude-opus-4-6 等
    messages: normalizedMessages,  // 历史消息 + 上下文注入
    system: systemPrompt,          // 多段 system prompt
    tools: toolSchemas,            // 所有可用 tool 的 JSON schema
    thinking: thinkingConfig,      // extended thinking / adaptive / disabled
    betas: [                       // 实验性功能 beta header
      'prompt-caching-2024-07-31',
      'interleaved-thinking-2025-05-14',
      'task-budgets-2026-03-13,
    ],
    max_tokens: "",
  })
```

  ---
### 上下文管理策略

**最复杂的部分，按触发顺序**:

| **策略**              | **触发时机**       | **说明**                  |
| ------------------- | -------------- | ----------------------- |
| **snipCompact**     | 每次循环           | 裁剪历史中的低价值片段             |
| **microcompact**    | 每次循环           | 将部分 `tool_result` 缓存/压缩 |
| **contextCollapse** | 接近限制前          | 折叠旧 tool 调用为摘要          |
| **autoCompact**     | 超过阈值           | 用 LLM 生成整个历史的摘要消息，替换原历史 |
| **reactiveCompact** | 收到 413/PTL 错误后 | 响应式触发压缩后重试              |

  ---
### 工具系统

  每个工具遵循统一接口：

  ```typescript
  
  type Tool = {
    name: string
    inputSchema: ZodSchema         // 输入校验
    call(input, context): AsyncGenerator  // 异步生成器（支持流式输出）
    isConcurrencySafe: boolean     // 决定是否可与其他工具并发执行
    maxResultSizeChars?: number    // 控制 tool_result 是否受 budget 截断
  }

  ```
  `toolOrchestration.ts` 将工具调用分组：
  - 只读/无副作用（isConcurrencySafe=true）→ 并发执行
  - 写操作/有副作用 → 串行执行

  ---
### 权限系统

  每次工具调用前都经过 canUseTool() 检查，支持多种模式：
  - default — 逐个提示用户
  - plan — 只读模式，禁止写操作
  - bypassPermissions — 全自动（SDK 无头模式）
  - auto — 基于 TRANSCRIPT_CLASSIFIER 自动判断

  ---
### 多智能体架构

  主 REPL（main thread）
    └→ AgentTool.call()           spawn 子 agent
         └→ 独立的 QueryEngine 实例
              ├─ 独立的消息历史
              ├─ 独立的 abort controller
              └─ 结果通过 yield 返回给父 agent

  coordinator/（COORDINATOR_MODE feature flag）
    └→ 管理多 agent 并行任务分配

  ---
## 总结

  1. 整个 query 循环是 AsyncGenerator，LLM 流式输出、tool 执行结果、状态变更都通过 yield 向 UI 层推送，解耦了 LLM 逻辑与渲染逻辑
  2. feature flag（bun:bundle）做编译时 tree-shaking，PROACTIVE、KAIROS、BRIDGE_MODE 等功能在发布版本中被完全剥离，不是运行时开关
  3. Prompt Caching 深度集成：microcompact 通过 cache_control 标记保留 tool_result 的 prompt cache，并在 cache_deleted_input_tokens 回来后才生成 boundary 消息
  4. token budget 追踪 贯穿全程：cost-tracker.ts + API 返回的 usage 字段，支持 task_budget beta 参数（task-budgets-2026-03-13）
