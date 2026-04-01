## 核心定位：输入预处理器

`processUserInput` 的本质是：**“把原始用户输入，变成一组可执行、可审计、可分流的上下文消息，并决定是否继续发起 query。”**

- **入口函数：** `src/utils/processUserInput/processUserInput.ts:117`
    
- **分流核心：** `src/utils/processUserInput/processUserInput.ts:275`

---

## 整体执行流程

从用户按下回车到决定是否请求模型，经历了以下五个阶段：

1. **即时反馈：** 在 `prompt` 模式下，系统会立刻将原始输入渲染到 UI 界面，避免用户感知延迟。
    
2. **核心处理 (`processUserInputBase`)：** 进行格式转换、图片处理和命令解析。
    
3. **出口判断：** 如果 `shouldQuery === false`（例如纯命令执行），流程直接在此短路返回。
    
4. **Hook 拦截：** 运行 `UserPromptSubmit` 钩子，进行最后的合规检查或上下文注入。
    
5. **结果输出：** 返回包含 `messages`、`model` 配置和 `allowedTools` 的结构化对象。

---

## 图片处理逻辑（关键性能点）

图片不会被直接投喂给模型，而是经过高强度的预处理以平衡性能和成本：

|**场景**|**处理流程**|**技术细节**|
|---|---|---|
|**内联图片块**|走 `maybeResizeAndDownsampleImageBlock`|进行压缩、缩放和降采样，确保符合 API 限制。|
|**粘贴图片**|`storeImages` -> 并行 Resize|存储后并行处理，生成包含元信息的隐藏消息。|
|**性能埋点**|`query_image_processing_start/end`|记录图片处理耗时，这通常是预处理中最沉重的环节。|

---

## 分流分支：命令与文本

系统根据输入内容的前缀和类型，将请求分发到不同的子处理器：

- **Slash Commands (`/` 开头):** * 检查是否为远程安全命令 (`isBridgeSafeCommand`)。
    
    - 如果是敏感或非法远程命令，直接返回提示，不将原始命令发给模型。
        
- **特殊关键字:** * 例如 `ultraplan` 会被自动重写并路由至 `/ultraplan` 逻辑。
    
- **基础输入:** * **Bash:** 走 `processBashCommand`。
    
    - **文本:** 走 `processTextPrompt`。
        
    - **公共逻辑:** 所有分支最后都会调用 `addImageMetadataMessage` 补全图片元数据。

---

## Hooks 后处理：最后的守门人

在返回结果前，`executeUserPromptSubmitHooks()` 拥有最高级别的控制权：

- **阻断 (Blocking):** 如果检测到安全风险或错误，将输入替换为系统警告，并强制停止 Query。
    
- **注入 (Augmentation):** 根据 Hook 逻辑，向当前上下文追加额外的 Message 或上下文信息。
    
- **链式触发:** 通过 `nextInput` 引导系统进入下一个自动化步骤。

---

## 返回值清单 (ProcessUserInputBaseResult)

最终输出给 `QueryEngine` 的是一个精密的控制对象：

- **`messages`**: 最终构建的消息数组（含用户意图、附件、图片元数据）。
    
- **`shouldQuery`**: 布尔值，控制是否进入 LLM 推理循环。
    
- **`allowedTools`**: 明确本次请求模型权限范围内的工具集。
    
- **`model / effort`**: 针对本次请求指定的模型版本或推理强度（如 Thinking Process 配置）。
    
- **`resultText`**: 非交互模式（CLI 直接输出）下的文本内容。