# 上下文管理和压缩

- `CACHED_MICROCOMPACT`：通过给请求插入 `cache_edits` 和给旧 `tool_result` 标记 `cache_reference`，对服务端已缓存前缀做增量删除，尽量在清理旧工具结果的同时保住 prompt cache 命中；主逻辑见 [microCompact.ts](../src/services/compact/microCompact.ts)、[claude.ts](../src/services/api/claude.ts)、[query.ts](../src/query.ts)、[prompts.ts](../src/constants/prompts.ts)。

### applyToolResultBudget

- 目标：在真正进入 compact 之前，先给 `tool_result` 做一层“消息级瘦身”，避免单轮工具返回把上下文直接撑爆；调用点见 [query.ts#L380](../src/query.ts)、实现见 [toolResultStorage.ts](../src/utils/toolResultStorage.ts)。
- 预算限制：默认 `MAX_TOOL_RESULTS_PER_MESSAGE_CHARS = 200_000`，可通过 GrowthBook flag `tengu_hawthorn_window` 覆盖，见 [toolLimits.ts](../src/constants/toolLimits.ts)。

#### 分组与评估逻辑

1. **按 API-level user message 分组**
  `collectCandidatesByMessage` 先把 messages 按 API-level 的 user message 分组。`normalizeMessagesForAPI` 会合并连续 user message，所以本地状态里 N 条分散的 `user` 消息在模型眼里可能是一条；预算必须按模型实际看到的 wire 体积来算，否则 N 条各自“看似不超预算”的结果合并后会直接突破上限，见 [toolResultStorage.ts#L600](../src/utils/toolResultStorage.ts)。
2. **组内 partition**
  对每条 API user message，将其中的 `tool_result` 分成三类：
  - `mustReapply`：之前已经被替换过的 → 直接复用缓存的 replacement 字符串（零 I/O，字节级一致）
  - `frozen`：之前已看过但未替换的 → 不能再动，否则会破坏 prompt cache 前缀
  - `fresh`：本轮新出现的 → 唯一可能被替换的候选，见 [toolResultStorage.ts#L649](../src/utils/toolResultStorage.ts)。
3. **逐个从大到小替换**
  对 `fresh` 候选，`selectFreshToReplace` 按 `size` **从大到小排序**，然后逐个 push 进替换列表，每选一个就 `remaining -= c.size`，直到 `frozenSize + remainingFreshSize <= limit`（默认 200000 字符）。**不是整组一起压缩**，而是挑最大的几个单独替换，见 [toolResultStorage.ts#L675](../src/utils/toolResultStorage.ts)。
4. **persist + replace**
  选中的 `tool_result` 被 `persistToolResult` 写入磁盘（文件名按 `tool_use_id` 命名，已有则跳过），内容替换成 `<persisted-output>` 包裹的 preview 引用（文件路径 + 前 2000 字节预览），见 [toolResultStorage.ts#L728](../src/utils/toolResultStorage.ts)。
   replacement 形态：

```txt
<persisted-output>
Output too large (...). Full output saved to: ...
Preview (first ...):
...
</persisted-output>
```

#### 例子

一条 API user message 里有 5 个并行 `tool_result`，大小分别是 **80K、80K、80K、10K、10K**，总 260K：

- `frozenSize = 0`，`freshSize = 260K`，`limit = 200K` → 超预算。
- `fresh` 排序后：80K、80K、80K、10K、10K。
- 先替换第一个 80K：`remaining = 180K` ≤ 200K，停止。
- 实际只替换**一个** 80K 的结果；其余 4 个保留原样。

如果 5 个结果都是 60K，总 300K：

- 替换第一个 60K：`remaining = 240K` > 200K，继续。
- 替换第二个 60K：`remaining = 180K` ≤ 200K，停止。
- 最终替换**两个** 60K 的结果；其余 3 个保留。

#### 状态策略

这层不是每轮都重新决定，而是按 `tool_use_id` 记住“之前见过什么、替换过什么”。

- 已经替换过的结果后续继续复用同一 replacement（`mustReapply` 路径）。
- 已经发给模型但未替换的结果则冻结为原文（`frozen`），避免 prompt 前缀抖动。
- 被跳过的工具（如 Read，`maxResultSizeChars = Infinity`）会标记为 `seen` 并冻结，永不替换，见 [toolResultStorage.ts#L816](../src/utils/toolResultStorage.ts)。

#### 与后续 compact 的关系

`applyToolResultBudget` 解决的是“大工具结果先卸载”，后面的 `snip / microcompact / autocompact` 解决的是“整段历史进一步压缩”；前者是工具结果专项预算，后者是整体上下文治理，主流程见 [query.ts](../src/query.ts)。

### snipCompactIfNeeded

- 定位：`snip` 看起来是识别无用的旧历史，把一段旧历史直接从活跃上下文里剪掉”的轻量压缩；调用点见 [query.ts#L404](../src/query.ts)。
- 机制线索：snip boundary 会记录 `removedUuids`，恢复会话时会按这些 UUID 重放删除，并修复消息链路；说明它的核心动作是“删除哪些旧消息”，不是生成总结，见 [sessionStorage.ts](../src/utils/sessionStorage.ts)。
- 视图策略：REPL 为了保留 scrollback，会保留完整历史，但模型侧读取消息时可通过 `projectSnippedView` 投影掉被 snip 的消息；无头/SDK 路径则倾向于直接截断内存中的历史，见 [messages.ts](../src/utils/messages.ts)、[QueryEngine.ts](../src/QueryEngine.ts)。
- 与其它压缩的关系：`snip` 更像“先删掉一部分旧包袱”，`microcompact / autocompact` 再继续处理剩余上下文；前者偏删除式，后者偏压缩式，主流程见 [query.ts](../src/query.ts)。
- 当前仓库状态：核心实现目前是 stub，实际代码默认不会真的 snip；但从接口、注释和恢复逻辑看，设计意图已经比较明确，见 [snipCompact.ts](../src/services/compact/snipCompact.ts)、[snipProjection.ts](../src/services/compact/snipProjection.ts)。

#### 最可能的算法

1. **按上下文增长节奏提示，但不是纯 token 硬阈值自动执行**
  现有线索更像“上下文变重时，系统按一段固定节奏 nudges 模型考虑 snip”，而不是像 autocompact 那样“超过 X token 立即压缩”。`context_efficiency` attachment 的注释明确写了 pacing 由 `shouldNudgeForSnips` 控制，且存在 `10k interval`，这个 interval 会在 prior nudges、snip markers、snip boundaries、compact boundaries 时重置，见 [attachments.ts#L3960](../src/utils/attachments.ts)。
2. **先登记 marker，再在 query 前统一执行**
  代码里区分了 `snip marker` 和 `snip boundary`：marker 是 internal registration marker，不直接给用户看；boundary 才表示“真正执行过一次 snip”，见 [Message.tsx#L203](../src/components/Message.tsx)。因此更像两阶段：先由 `SnipTool` 或 `/force-snip` 之类入口登记“哪段历史可裁剪”，再由 [query.ts#L404](../src/query.ts) 的 `snipCompactIfNeeded()` 在每轮 query 前统一落地。
3. **真正执行时是“删中段消息”，不是摘要**
  恢复逻辑的注释已经把设计说透了：`compact_boundary` 是截掉前缀，`snip` 是删除历史中间 ranges；执行时会把被删除消息记成 `removedUuids`，恢复时重放删除，并修复 surviving message 的 `parentUuid` 链，避免 resume 时把整段未 snip 历史重新串回来，见 [sessionStorage.ts#L1963](../src/utils/sessionStorage.ts)。
4. **模型视图删掉，REPL scrollback 保留**
  执行后并不一定把所有原始消息都从 UI 层彻底抹掉。更像是 REPL 保留全量 transcript 方便滚动查看，而 API 视图通过 `projectSnippedView()` 投影掉被 snip 的消息；resume / replay 时又可以根据 boundary 重放这次删除，见 [messages.ts#L4685](../src/utils/messages.ts)、[QueryEngine.ts#L1266](../src/QueryEngine.ts)。
5. **返回粗粒度 token 回收量，供 autocompact 阈值修正**
  `snipCompactIfNeeded()` 的返回值里有 `tokensFreed`，并且 query loop 会把它继续传给 autocompact。说明设计上 snip 不是 UI 隐藏，而是真要从模型输入里删掉一段历史，并估算释放了多少 token，避免 autocompact 用 stale usage 误判，见 [query.ts#L397](../src/query.ts)、[autoCompact.ts#L165](../src/services/compact/autoCompact.ts)。

### microcompactMessages

- 定位：`microcompact` 是 `autocompact` 前的一层轻量清理，专门处理旧 `tool_result`；调用点见 [query.ts#L415](../src/query.ts)，实现见 [microCompact.ts](../src/services/compact/microCompact.ts)。
- 只有在COMPACTABLE_TOOLS内的工具会被压，其他都会跳过！
- 冷路径：如果距离上一条 assistant 消息超过 60 分钟，就认为服务端 cache 基本过期；这时直接把较老的 `tool_result` 内容清空成 `[Old tool result content cleared]`，只保留最近 5 条，见 [timeBasedMCConfig.ts](../src/services/compact/timeBasedMCConfig.ts)、[microCompact.ts](../src/services/compact/microCompact.ts)。
- 热路径：设计上是如果 cache 还热且模型支持 cache editing，就不改本地 `messages`，而是生成 `cache_edits` 交给 API 层去删服务端缓存前缀里的旧 `tool_result`；但当前仓库里 `cachedMicrocompact.ts` 是 stub，这条路径现在实际不生效，见 [microCompact.ts](../src/services/compact/microCompact.ts)、[cachedMicrocompact.ts](../src/services/compact/cachedMicrocompact.ts)、[claude.ts](../src/services/api/claude.ts)。
- 关系：`microcompact` 解决的是“旧工具结果太占上下文”，`autocompact` 解决的是“整体上下文仍然过大”；前者更轻、更偏工具结果专项治理，后者更重。

### contextCollapse

- 定位：`context collapse` 想做的是比 `autocompact` 更细粒度的分段归档；如果它先把上下文压到阈值下，后面的 `autocompact` 就可以不触发，调用点见 [query.ts#L440](../src/query.ts)。
- 形态：设计上不是把整段历史压成一个总摘要，而是按多个 `span` 归档；状态里有 `collapsedSpans / collapsedMessages / stagedSpans`，界面文案也会显示“多少个 span 被 summarized”，见 [index.ts](../src/services/contextCollapse/index.ts)、[context-noninteractive.ts](../src/commands/context/context-noninteractive.ts)。
- 存储方式：它不是往 REPL 消息数组里 `yield` 新摘要，而是把摘要放进 collapse store，读取时通过 `projectView()` 投影出 collapsed 视图，所以效果可以跨 turn 持续，见 [query.ts](../src/query.ts)、[operations.ts](../src/services/contextCollapse/operations.ts)、[persist.ts](../src/services/contextCollapse/persist.ts)。
- 当前仓库状态：核心实现目前是 stub，`applyCollapsesIfNeeded()` 和 `projectView()` 都是 no-op；因此只能确认它要做“分段归档”，看不到真实的 span 划分规则，也不能确认是按 turn、token 还是主题切分，见 [index.ts](../src/services/contextCollapse/index.ts)、[operations.ts](../src/services/contextCollapse/operations.ts)。

#### 最可能的算法

1. **接近上下文上限时，先用 collapse 接管 headroom 管理**
  它的触发条件看起来比 autocompact 更前置。`autoCompact.ts` 里的注释明确提到，collapse 模式下由 `90% commit-start / 95% blocking-spawn` 这套流程来管理 headroom，因此一旦 `isContextCollapseEnabled()` 为真，就会 suppress proactive autocompact，避免 autocompact 抢先把上下文粗暴压成单摘要，见 [autoCompact.ts#L197](../src/services/compact/autoCompact.ts)。
2. **先把旧历史切成 span 放入 staged queue**
  类型定义里不只有 “collapsed”，还有 `stagedSpans`，持久化快照还记录了 `staged: [{ startUuid, endUuid, summary, risk, stagedAt }]`、`armed`、`lastSpawnTokens`。这说明算法更像“先识别一段可折叠历史，排进 staged queue，等待后台处理”，而不是当前轮同步把整段历史马上改写，见 [logs.ts#L247](../src/types/logs.ts)。
3. **后台 ctx-agent 为每个 staged span 产出摘要，再提交为 committed collapse**
  `ContextCollapseCommitEntry` 记录的是 `summaryUuid / summaryContent / summary / firstArchivedUuid / lastArchivedUuid`，并且注释明确说 archived messages 本身不需要持久化，因为它们已经在 transcript 里；commit 只持久化“摘要占位符 + splice instruction”。因此最可能的执行器是后台 spawn 一个 ctx-agent，总结 span，成功后把这个 span 从 staged 变成 committed，见 [logs.ts#L247](../src/types/logs.ts)。
4. **读取时不是“插一条总结消息”，而是投影成 collapsed placeholder**
  `projectView()` 的职责更像是：扫描当前消息数组，找到 `firstArchivedUuid..lastArchivedUuid` 对应的原始 span，把它替换成一个 `<collapsed id="...">summary</collapsed>` 之类的占位符视图。这个 placeholder 不直接显示在对话里，只在 context 统计里体现成“多少个 span summarized”，见 [ContextVisualization.tsx#L18](../src/components/ContextVisualization.tsx)、[context.tsx#L16](../src/commands/context/context.tsx)。
5. **413 / prompt-too-long 时走 drain：先排空 staged spans，再 retry**
  在错误恢复链里，`prompt-too-long` 会先被 withheld，不立刻暴露；第一优先不是 reactive compact，而是 `recoverFromOverflow()`。注释明确说它会“先把所有 staged 的 context-collapse 彻底排空”，也就是把还没正式提交的 spans 尽快 collapse 掉，再用新的 collapsed view 重试；只有这条路失败，才退化到 reactive compact，见 [query.ts#L1058](../src/query.ts)。
6. **设计目标是保留细粒度上下文，而不是退化成整段摘要**
  整个算法存在的意义就是：在接近上下文上限时，优先把“已经完成、可归档”的旧 span 逐段折叠掉，让最近上下文继续保留原始消息粒度；只有 collapse 已无法继续释放空间，才需要 autocompact / reactive compact 这种更重、更粗的摘要路径，见 [query.ts#L429](../src/query.ts)、[autoCompact.ts#L197](../src/services/compact/autoCompact.ts)。

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
