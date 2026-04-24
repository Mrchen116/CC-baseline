# 上下文管理和压缩

- `CACHED_MICROCOMPACT`：通过给请求插入 `cache_edits` 和给旧 `tool_result` 标记 `cache_reference`，对服务端已缓存前缀做增量删除，尽量在清理旧工具结果的同时保住 prompt cache 命中；主逻辑见 [microCompact.ts](../src/services/compact/microCompact.ts)、[claude.ts](../src/services/api/claude.ts)、[query.ts](../src/query.ts)、[prompts.ts](../src/constants/prompts.ts)。

### applyToolResultBudget

- 目标：在真正进入 compact 之前，先给 `tool_result` 做一层“消息级瘦身”，避免单轮工具返回把上下文直接撑爆；调用点见 [query.ts#L380](../src/query.ts)、实现见 [toolResultStorage.ts](../src/utils/toolResultStorage.ts)。
- 处理对象：预算是按一个 API-level `user message` 整组计算的；同一组里的多个 `tool_result` 会把大小加在一起算，不是本地数组里一条 `user` 消息算一次。这样多工具并行返回时，能按模型实际看到的 wire 体积处理，见 [toolResultStorage.ts](../src/utils/toolResultStorage.ts)。
- 处理方式：超预算的结果不会直接截断，而是把完整内容落盘，再把消息里的 `tool_result.content` 替换成“文件路径 + 预览”的短文本；这样既减小上下文，又保留回读入口，见 [toolResultStorage.ts](../src/utils/toolResultStorage.ts)。
- replacement 形态示意：

```txt
<persisted-output>
Output too large (...). Full output saved to: ...
Preview (first ...):
...
</persisted-output>
```

- 状态策略：这层不是每轮都重新决定，而是按 `tool_use_id` 记住“之前见过什么、替换过什么”；已经替换过的结果后续继续复用同一 replacement，已经发给模型但未替换的结果则冻结为原文，避免 prompt 前缀抖动，见 [toolResultStorage.ts](../src/utils/toolResultStorage.ts)。
- 与后续 compact 的关系：`applyToolResultBudget` 解决的是“大工具结果先卸载”，后面的 `snip / microcompact / autocompact` 解决的是“整段历史进一步压缩”；前者是工具结果专项预算，后者是整体上下文治理，主流程见 [query.ts](../src/query.ts)。

### snipCompactIfNeeded

- 定位：`snip` 看起来是识别无用的旧历史，把一段旧历史直接从活跃上下文里剪掉”的轻量压缩；调用点见 [query.ts#L404](../src/query.ts)。
- 机制线索：snip boundary 会记录 `removedUuids`，恢复会话时会按这些 UUID 重放删除，并修复消息链路；说明它的核心动作是“删除哪些旧消息”，不是生成总结，见 [sessionStorage.ts](../src/utils/sessionStorage.ts)。
- 视图策略：REPL 为了保留 scrollback，会保留完整历史，但模型侧读取消息时可通过 `projectSnippedView` 投影掉被 snip 的消息；无头/SDK 路径则倾向于直接截断内存中的历史，见 [messages.ts](../src/utils/messages.ts)、[QueryEngine.ts](../src/QueryEngine.ts)。
- 与其它压缩的关系：`snip` 更像“先删掉一部分旧包袱”，`microcompact / autocompact` 再继续处理剩余上下文；前者偏删除式，后者偏压缩式，主流程见 [query.ts](../src/query.ts)。
- 当前仓库状态：核心实现目前是 stub，实际代码默认不会真的 snip；但从接口、注释和恢复逻辑看，设计意图已经比较明确，见 [snipCompact.ts](../src/services/compact/snipCompact.ts)、[snipProjection.ts](../src/services/compact/snipProjection.ts)。

### microcompactMessages

- 定位：`microcompact` 是 `autocompact` 前的一层轻量清理，专门处理旧 `tool_result`；调用点见 [query.ts#L415](../src/query.ts)，实现见 [microCompact.ts](../src/services/compact/microCompact.ts)。
- 冷路径：如果距离上一条 assistant 消息超过 60 分钟，就认为服务端 cache 基本过期；这时直接把较老的 `tool_result` 内容清空成 `[Old tool result content cleared]`，只保留最近 5 条，见 [timeBasedMCConfig.ts](../src/services/compact/timeBasedMCConfig.ts)、[microCompact.ts](../src/services/compact/microCompact.ts)。
- 热路径：设计上是如果 cache 还热且模型支持 cache editing，就不改本地 `messages`，而是生成 `cache_edits` 交给 API 层去删服务端缓存前缀里的旧 `tool_result`；但当前仓库里 `cachedMicrocompact.ts` 是 stub，这条路径现在实际不生效，见 [microCompact.ts](../src/services/compact/microCompact.ts)、[cachedMicrocompact.ts](../src/services/compact/cachedMicrocompact.ts)、[claude.ts](../src/services/api/claude.ts)。
- 关系：`microcompact` 解决的是“旧工具结果太占上下文”，`autocompact` 解决的是“整体上下文仍然过大”；前者更轻、更偏工具结果专项治理，后者更重。

### contextCollapse

- 定位：`context collapse` 想做的是比 `autocompact` 更细粒度的分段归档；如果它先把上下文压到阈值下，后面的 `autocompact` 就可以不触发，调用点见 [query.ts#L440](../src/query.ts)。
- 形态：设计上不是把整段历史压成一个总摘要，而是按多个 `span` 归档；状态里有 `collapsedSpans / collapsedMessages / stagedSpans`，界面文案也会显示“多少个 span 被 summarized”，见 [index.ts](../src/services/contextCollapse/index.ts)、[context-noninteractive.ts](../src/commands/context/context-noninteractive.ts)。
- 存储方式：它不是往 REPL 消息数组里 `yield` 新摘要，而是把摘要放进 collapse store，读取时通过 `projectView()` 投影出 collapsed 视图，所以效果可以跨 turn 持续，见 [query.ts](../src/query.ts)、[operations.ts](../src/services/contextCollapse/operations.ts)、[persist.ts](../src/services/contextCollapse/persist.ts)。
- 当前仓库状态：核心实现目前是 stub，`applyCollapsesIfNeeded()` 和 `projectView()` 都是 no-op；因此只能确认它要做“分段归档”，看不到真实的 span 划分规则，也不能确认是按 turn、token 还是主题切分，见 [index.ts](../src/services/contextCollapse/index.ts)、[operations.ts](../src/services/contextCollapse/operations.ts)。

### autocompact

- 定位：`autocompact` 是前面几层轻量治理之后的重型兜底；只有上下文仍然超过阈值时才触发，调用点见 [query.ts#L453](../src/query.ts#L453)、实现见 [autoCompact.ts](../src/services/compact/autoCompact.ts)。
- 触发方式：它会用当前 `messagesForQuery` 估算 token，并减去 `snipTokensFreed`；如果仍高于 autocompact 阈值，就进入 compact 分支，见 [autoCompact.ts](../src/services/compact/autoCompact.ts)。

#### session-memory compact [不调用llm]

- 入口：`autocompact` 触发后，会先尝试 `trySessionMemoryCompaction()`；只有这条路失败，才会回退到完整 compact，见 [autoCompact.ts](../src/services/compact/autoCompact.ts)、[sessionMemoryCompact.ts](../src/services/compact/sessionMemoryCompact.ts)。
- summary 来源：这里不现场重跑“总结整段历史”的 prompt，而是直接读取 session memory 文件 `getSessionMemoryContent()`。这份文件是主线程会话过程中由 post-sampling hook 后台维护的一份 markdown 记忆，见 [sessionMemoryUtils.ts](../src/services/SessionMemory/sessionMemoryUtils.ts)、[sessionMemory.ts](../src/services/SessionMemory/sessionMemory.ts)。
- 边界：`lastSummarizedMessageId` 表示 session memory 大致已经覆盖到哪条消息；compact 时会以它为参考，只保留边界之后的一段最近原始消息，见 [sessionMemoryUtils.ts](../src/services/SessionMemory/sessionMemoryUtils.ts)、[sessionMemoryCompact.ts](../src/services/compact/sessionMemoryCompact.ts)。
- 最近尾巴：保留的不是固定几条消息，而是`lastSummarizedMessageId`后，一段“足够继续工作”的尾部上下文；默认要求至少保留 `10k` tokens、至少 `5` 条带文本消息，若剩余消息不能满足，则继续往前取`lastSummarizedMessageId`前的消息，直到满足条件为止，并尽量控制在 `40k` tokens 内，同时不能拆开 `tool_use/tool_result` 配对，见 [sessionMemoryCompact.ts](../src/services/compact/sessionMemoryCompact.ts)。
- 先区分两层：`session-memory compact` 先产出一份本地 `CompactionResult`，其中包含 `boundaryMarker + summaryMessages + messagesToKeep + attachments + hookResults`；但真正发给 LLM 前还会经过 `normalizeMessagesForAPI()`，普通 `system` 消息会被过滤、`attachment` 会被并入 `user` 消息，见 [sessionMemoryCompact.ts](../src/services/compact/sessionMemoryCompact.ts)、[messages.ts](../src/utils/messages.ts)、[query.ts](../src/query.ts)。
- 真正发给 LLM 的形态大致如下；其中 `messagesToKeep` 仍然是一条条原始消息，不会被再拼成字符串：

```txt
[systemPrompt parameter]               // 单独参数，不在 messages 列表里
systemPrompt + systemContext

[messages parameter]

0. user message (isMeta = true)       // prependUserContext(userContext)
   content: <system-reminder>...</system-reminder>

1. user message (isCompactSummary = true)
   content:
   "This session is being continued from a previous conversation that ran out of context.
   The summary below covers the earlier portion of the conversation.

   # Current State
   - 用户当前在看 query.ts 的上下文压缩链路
   - 已确认 applyToolResultBudget / microcompact / autocompact 的职责

   # Important Findings
   - tool_result budget 是按 API-level user message 分组算
   - microcompact 冷路径会把旧 tool_result 清成 [Old tool result content cleared]
   - contextCollapse 当前仓库还是 stub

   # Next Step
   - 继续理解完整 compact 的 prompt 和产物

   If you need specific details from before compaction, read the full transcript at:
   /path/to/session.jsonl

   Recent messages are preserved verbatim."

2. assistant/user/...                 // messagesToKeep 中的一条原始消息
   content: [...]

3. assistant/user/...                 // messagesToKeep 中的另一条原始消息
   content: [...]

4. assistant/user/...                 // 继续原样保留
   content: [...]

5. user message(s)                    // 由 attachments / hookResults 归一化而来
   content: [plan attachment / hook_additional_context / 其他 hook 内容]
```

- 本地 `boundaryMarker` 仍会进入 transcript / REPL 状态，用来标记发生过 compact；但它不会进入真正发给 LLM 的 `messages` 列表。
- 换句话说：旧历史不会再以原始消息链继续存在，而是被替换成“一条装着 session memory 文件内容的 compact summary message”；再拼上一段最近原始消息尾巴，以及少量需要补回来的上下文。`systemPrompt` 本身不走 compact 消息链，而是每轮 query 继续单独传。
- summary.md读取时，每个# 内限制超过8000字符，则硬截断。
- 附件和 hook：`attachments` 在当前实现里主要是可选的 `planAttachment`；`hookResults` 在这条路径里最终会作为 attachment 内容并入 `user` 消息，而不是单独的“hook role”。

#### 完整 compact [调用llm]

- 回退分支：如果 session-memory compact 不可用、为空，或压完后仍超过阈值，就走 `compactConversation()`，见 [autoCompact.ts](../src/services/compact/autoCompact.ts)、[compact.ts](../src/services/compact/compact.ts)。
- 先区分两层：第一层是“为了生成 compact summary，当下发给总结模型的上下文”；第二层是“compact 成功后，下一轮继续主对话时真正发给模型的上下文”。两层不要混在一起，见 [compact.ts#L442](../src/services/compact/compact.ts#L442)、[compact.ts#L615](../src/services/compact/compact.ts#L615)、[messages.ts](../src/utils/messages.ts)。
- 压缩时给到总结模型的上下文大致如下：

```txt
[systemPrompt parameter]
systemPrompt + systemContext

[messages parameter]
0. user/assistant/...                 // 原来的整段历史消息
1. user message                       // summaryRequest
   content: getCompactPrompt(customInstructions)
            = NO_TOOLS_PREAMBLE + BASE_COMPACT_PROMPT + optional customInstructions + NO_TOOLS_TRAILER
            也就是说“禁止调用工具、只能返回纯文本总结”是这条 user message 文本的一部分，不是单独消息
```

- 这一阶段模型返回的是一条 assistant message；代码从里面提取 `summary` 文本，再用 `getCompactUserSummaryMessage()` 包成后续继续使用的 compact summary message，见 [prompt.ts](../src/services/compact/prompt.ts)、[compact.ts](../src/services/compact/compact.ts)。
- 这份 `summary` 的正文格式来自 `BASE_COMPACT_PROMPT`，核心是一个详细结构化总结。最终写回 compact summary message 时，`<analysis>` 会被剥掉，只保留格式化后的 `Summary:` 正文，里面会按下面这类 section 组织内容，见 [prompt.ts](../src/services/compact/prompt.ts)：

```txt
Summary:
1. Primary Request and Intent:
   ...

2. Key Technical Concepts:
   - ...

3. Files and Code Sections:
   - ...

4. Errors and fixes:
   - ...

5. Problem Solving:
   ...

6. All user messages:
   - ...

7. Pending Tasks:
   - ...

8. Current Work:
   ...

9. Optional Next Step:
   ...
```

- compact 成功后，本地会先得到 `CompactionResult = boundaryMarker + summaryMessages + attachments + hookResults`。其中 `boundaryMarker` 只用于 transcript / REPL 标记 compact 边界，不会进入真正发给 LLM 的 `messages` 列表。
- 压缩后继续主对话时，真正发给模型的上下文大致如下：

```txt
[systemPrompt parameter]
systemPrompt + systemContext

[messages parameter]
0. user message (isMeta = true)       // prependUserContext(userContext)
   content: <system-reminder>...</system-reminder>

1. user message (isCompactSummary = true)
   content:
   "This session is being continued from a previous conversation that ran out of context.
   The summary below covers the earlier portion of the conversation.

   Summary:
   1. Primary Request and Intent:
      用户在排查 query.ts 的上下文压缩链路，希望厘清 session-memory compact、完整 compact、microcompact 的真实行为。

   2. Key Technical Concepts:
      - session memory
      - autocompact
      - compact boundary

   3. Files and Code Sections:
      - src/query.ts
      - src/services/compact/autoCompact.ts
      - src/services/compact/compact.ts

   4. Errors and fixes:
      - 之前对 attachment / system message 的理解有误，已重新按 normalizeMessagesForAPI 校正。

   5. Problem Solving:
      逐步厘清 compact 前后本地消息结构与真正发给 LLM 的 messages 结构。

   6. All user messages:
      - ...

   7. Pending Tasks:
      - 继续理解剩余压缩链路

   8. Current Work:
      - 正在整理 autocompact 两条分支的真实上下文形态

   9. Optional Next Step:
      - 继续往后看 compact 后的继续执行逻辑

   If you need specific details from before compaction, read the full transcript at:
   /path/to/session.jsonl"

2. user message(s) (isMeta = true)    // 由 attachments 归一化而来，条数不固定，连续时也可能被 merge
   content: [最近读过的文件内容（最多 5 个） / plan file / invoked skills / plan mode instructions / MCP instructions / agent status ...]

3. user message(s) (isMeta = true)    // 由 hookResults 归一化而来
   content: [SessionStart hook additional context / 其他 hook 内容]
```

- 其中“最近读过的文件内容”是指 `readFileState` 里最近访问的文件，经 `createPostCompactFileAttachments()` 重新读取后补回；默认最多恢复 `5` 个文件，还受单文件和总 token budget 限制。
- 其中 `invoked skills` 是“这个 session / 当前 agent 已经调用过的 skills 的内容”，只在完整 compact 路径里补；当前 `session-memory compact` 路径不会补这一项。
- 和 session-memory compact 的区别：完整 compact 不保留一段原始尾巴 `messagesToKeep`；旧历史整体被“compact summary message + 恢复运行态上下文的 meta user messages”替代。

