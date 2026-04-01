## 整体结构

- `queryLoop` 是一个 `async function*`（异步生成器），在一个 `while(true)` 无限循环中不断迭代，每次迭代代表一个"轮次"（turn）——即向模型发送请求、获取响应、执行工具、再决定是否继续。

---

## 核心职责

1. **状态管理** — 将参数分为不可变参数（`systemPrompt`、`canUseTool` 等）和可变状态（`messages`、`toolUseContext`、`turnCount` 等），每次迭代开头解构状态，在 `continue` 时整体重新赋值。
2. **上下文压缩（Compaction）** — 这是代码中最复杂的部分。当对话变得太长时，它有多层压缩策略：snip（裁剪）→ microcompact（微压缩）→ context collapse（上下文折叠）→ autocompact（自动压缩）→ reactive compact（响应式压缩，在遇到 prompt-too-long 413 错误后触发）。这些策略按顺序逐级应用，确保对话不会超出模型的上下文窗口。
3. **工具执行** — 模型返回 `tool_use` 块时，循环会执行对应的工具，收集 `tool_result`，并将结果附加到消息中，然后进入下一次迭代让模型继续。
4. **错误恢复** — 处理多种错误场景：prompt-too-long 时尝试压缩后重试；max-output-tokens 时先尝试提高到 64k 再重试；媒体文件过大时通过 reactive compact 去除后重试；还支持 fallback model（模型降级），在主模型失败时切换到备用模型。
5. **预取优化（Prefetch）** — 并行预取记忆（memory prefetch）和技能发现（skill discovery prefetch），在模型流式输出的 5-30 秒内同时运行，避免串行等待。
6. **Token 预算追踪** — 跨压缩边界追踪 `task_budget.remaining`，确保压缩后服务端仍能正确计算剩余预算。
7. **中断处理** — 支持用户中断（Ctrl+C），中断时会消费流式工具执行器的剩余结果，生成合成的 tool_result，并清理 MCP 资源。
8. **轮次限制与停止钩子（Stop Hooks）** — 检查 `maxTurns` 限制，执行 post-sampling hooks，并在 API 错误时跳过 stop hooks 以避免死循环。

这是一个复杂的**AI 代理运行时核心循环**，负责"思考→行动→观察→再思考"的完整闭环，同时处理大量的边界情况和性能优化。

---
## AI 代理运行时核心循环
### 1. 思考（Think）— 调用模型生成响应

对应代码中的 `deps.callModel(...)` 及其 `for await` 流式消费循环：

```typescript
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt: fullSystemPrompt,
  thinkingConfig: toolUseContext.options.thinkingConfig,
  tools: toolUseContext.options.tools,
  signal: toolUseContext.abortController.signal,
  options: { model: currentModel, ... },
})) {
  // 流式接收模型的 assistant message
  assistantMessages.push(message)

  // 关键判断：模型是否输出了 tool_use 块
  const msgToolUseBlocks = message.message.content
    .filter(content => content.type === 'tool_use')
  if (msgToolUseBlocks.length > 0) {
    toolUseBlocks.push(...msgToolUseBlocks)
    needsFollowUp = true  // ← 这个标志决定了循环是否继续
  }
}
```

这一段做的事情是：把完整的对话历史（`messagesForQuery`）、系统提示、工具定义等发给模型 API，然后流式接收模型的回复。模型在"思考"后，要么输出纯文本（表示任务完成），要么输出 `tool_use` 块（表示它想调用某个工具）。

`needsFollowUp = true` 就是闭环的驱动信号——只要模型请求了工具调用，循环就不会在这一轮结束。

---

### 2. 行动（Act）— 执行工具调用

对应代码中工具执行部分：

```typescript
// 获取工具执行结果（流式或批量）
const toolUpdates = streamingToolExecutor
  ? streamingToolExecutor.getRemainingResults()
  : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)

for await (const update of toolUpdates) {
  if (update.message) {
    yield update.message          // 向调用方输出工具执行过程
    toolResults.push(...)         // 收集 tool_result
  }
  if (update.newContext) {
    updatedToolUseContext = { ...update.newContext, queryTracking }
  }
}
```

这里有两种执行模式：`StreamingToolExecutor`（流式，模型一边输出 tool_use 块就一边开始执行）和 `runTools`（批量，等模型输出完再统一执行）。工具可以是文件读写、bash 命令、MCP 服务器调用等。

注意在流式模式下，工具执行实际上跟"思考"阶段是**交叠**的——模型还在流式输出时，已完成的 tool_use 块就已经开始执行了（`streamingToolExecutor.addTool(toolBlock, message)` 在 streaming 循环里被调用）。

---

### 3. 观察（Observe）— 收集结果并附加上下文

工具执行完成后，代码还会收集额外的上下文信息：

```typescript
// 收集附件消息（文件变更通知、队列中的命令等）
for await (const attachment of getAttachmentMessages(...)) {
  yield attachment
  toolResults.push(attachment)
}

// 消费记忆预取结果（相关的 memory 文件）
if (pendingMemoryPrefetch && pendingMemoryPrefetch.settledAt !== null) {
  const memoryAttachments = filterDuplicateMemoryAttachments(
    await pendingMemoryPrefetch.promise,
    toolUseContext.readFileState,
  )
  for (const memAttachment of memoryAttachments) {
    toolResults.push(createAttachmentMessage(memAttachment))
  }
}

// 消费技能发现预取结果
if (skillPrefetch && pendingSkillPrefetch) {
  const skillAttachments = await skillPrefetch.collectSkillDiscoveryPrefetch(...)
  for (const att of skillAttachments) {
    toolResults.push(createAttachmentMessage(att))
  }
}
```

这一步把工具的返回值（`tool_result`）、文件变更、记忆系统的相关上下文、技能发现结果等全部收集起来，作为下一轮"思考"的输入。

---

### 4. 再思考（Re-think）— 组装新状态并 `continue`

最后，代码将所有信息拼装成新的 `state`，然后 `continue` 回到 `while(true)` 循环顶部：

```typescript
const next: State = {
  messages: [
    ...messagesForQuery,      // 原始消息历史
    ...assistantMessages,      // 模型刚才的回复（含 tool_use）
    ...toolResults,            // 工具执行结果 + 附件
  ],
  toolUseContext: toolUseContextWithQueryTracking,
  autoCompactTracking: tracking,
  turnCount: nextTurnCount,
  maxOutputTokensRecoveryCount: 0,
  hasAttemptedReactiveCompact: false,
  pendingToolUseSummary: nextPendingToolUseSummary,
  transition: { reason: 'next_turn' },
  ...
}
state = next
// → continue 回到 while(true) 顶部，开始下一轮"思考"
```

这就是闭环的关键衔接点：把 `原始历史 + 模型回复 + 工具结果` 拼成新的 `messages` 数组，赋给 `state`，然后循环自然回到顶部，在下一次迭代中再次调用 `deps.callModel()`，模型就能看到上一轮工具的执行结果，继续决策。

---

### 5. 退出条件 — 循环何时终止

闭环在以下条件下终止（`needsFollowUp === false` 时进入退出路径）：

````typescript
if (!needsFollowUp) {
  // 模型没有请求工具调用 → 执行 stop hooks → return
  const stopHookResult = yield* handleStopHooks(...)
  if (stopHookResult.blockingErrors.length > 0) {
    // stop hook 返回了阻塞错误 → 注入错误消息，continue 再试
    state = { messages: [..., ...blockingErrors], ... }
    continue
  }
  return { reason: 'completed' }
}
```

也就是说，只有当模型的回复中**不包含任何 `tool_use` 块**时，才认为任务完成。但即使如此，还会经过 stop hooks 检查——如果 hook 认为模型不应该停下来（比如违反了某些规则），会注入阻塞错误消息，强制再循环一轮。

---

**总结一下整个循环的数据流：**
```
while(true) {
  ① 压缩/准备 messagesForQuery （snip → microcompact → collapse → autocompact）
  ② 调用模型 deps.callModel(messagesForQuery) → 流式接收 assistantMessages
  ③ 如果有 tool_use → 执行工具 → 收集 toolResults
  ④ 收集附件、记忆、技能发现
  ⑤ state.messages = [...messagesForQuery, ...assistantMessages, ...toolResults]
  ⑥ continue → 回到 ①
  
  如果没有 tool_use → stop hooks → return
}
````
