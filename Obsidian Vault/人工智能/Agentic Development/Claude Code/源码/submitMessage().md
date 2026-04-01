## 完整调用链路

  QueryEngine.submitMessage(prompt)                                              
 ├─ 1. 初始化配置
  │     ├─ setCwd()
  │     ├─ fetchSystemPromptParts()      → 构建 system prompt
  │     └─ 构建 processUserInputContext  → 组装工具列表、权限模式、状态等
  │
  ├─ 2. [[processUserInput.ts]] (prompt)         ← 第一阶段：预处理用户输入
  │     └─ 返回 { messages, shouldQuery, model, resultText }
  │
  ├─ 3. this.mutableMessages.push(...)   ← 把新消息追加到会话历史
  │
  ├─ 4. recordTranscript()               ← 持久化到磁盘（用于 /resume）
  │
  ├─ 5. yield buildSystemInitMessage()   ← 向外 yield 系统初始化消息（SDK层可见）
  │
  ├─ 6. if (!shouldQuery) → yield result, return   ← slash 命令短路出口
  │
  │   ─────── 以下只有 shouldQuery=true 才执行 ──────
  │
  ├─ 7. getAttachmentMessages()          ← 加载 memory 附件、skill 附件等
  │
  └─ 8. for await (msg of query({...}))  ← 第二阶段：query 循环（含流式）
        │
	    ├─ queryLoop() 内的每次迭代:
        │   ├─ 上下文压缩（snip/micro/auto compact）
        │   ├─ callModel() → SSE 流式 API
        │   ├─ 收到 tool_use → runTools()
        │   └─ 追加 tool_result → 继续下一次迭代
        │
        └─ yield 每条消息给调用方
             ├─ assistant 消息（流式文字）
             ├─ tool_result 消息
             └─ 最终 result 消息（含 cost / usage）