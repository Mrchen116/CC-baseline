# Session Transcript (JSONL) 设计与机制

## 1. 设计哲学：把"持久真相"与"工作内存"分离

Anthropic 在 [Managed Agents](https://www.anthropic.com/engineering/managed-agents) 一文中阐述了这套系统的核心架构哲学，也是 Claude Code 代码里 JSONL 设计的"所以然"：

### 1.1 Session 是"外化的上下文对象"

> "The session is an append-only log of everything that happened and functions as a context object that lives outside Claude's context window."

JSONL 文件不是服务于当前 API 调用的"工作内存"，而是一个**独立于模型上下文窗口之外的历史档案馆**。运行时内存里的 `mutableMessages` 负责即时的 API 交互；JSONL 负责忠实地记录一切，哪怕某些内容已经从当前 API 上下文中被裁剪掉。

### 1.2 Harness 是牲畜，不是宠物（Cattle, not pets）

> "Nothing in the harness needs to survive a crash."

因为 session log 在 harness（`query.ts` / `QueryEngine.ts` / REPL）外面，所以运行进程完全可以随意崩溃、重启、替换。新的 harness 只需要 `wake(sessionId)`，调用 `loadTranscriptFile` 把 JSONL 读回来，就能完整恢复状态。代码里每一轮用户输入、assistant 输出、工具结果都实时追加到 JSONL，正是这种"进程可随时死去"的底气来源。

### 1.3 持久化存储与上下文工程解耦

> "We can't predict what specific context engineering will be required in future models."

因此， Anthropic 把边界划得很清楚：

- **Session（JSONL）**只保证"所有事件都可恢复"。它不做任何 smart 的压缩或筛选，只 append。
- **Harness** 承担所有上下文管理策略：组织、缓存、裁剪、compaction、microcompact、reactive compact。

这意味着，如果未来有更好的上下文工程方法，可以直接重放 JSONL 重新组织，而无需改变持久化格式。代码里 compaction 不修改 JSONL、只修改"内存中发给 API 的视图"，正是这一哲学的直接体现。

### 1.4 API System Prompt 不持久化

一个常见的误解是：JSONL 既然存了 `type: "system"` 的消息，那系统提示词（system prompt）一定也在里面。实际上 **API 系统提示词从不写入 JSONL**。

`src/context.ts` 每轮请求前动态组装 system prompt（git status、CLAUDE.md 内容等），在 `src/services/api/claude.ts` 中直接传给 Anthropic SDK，用完即弃。JSONL 里的 `type: "system"` 消息是**终端 UI 元数据**（如 `api_metrics`、`compact_boundary`、`api_error` 等），它们只负责在 resume 后恢复终端状态，不会发给 LLM。

这再次体现了 Anthropic 的设计边界：**JSONL 只记录"发生了什么事件"，不记录"每轮请求时如何组装 prompt"**——prompt 工程属于 harness 的工作内存，不在持久化范围内。

### 1.5 通过 `getEvents()` 任意选择历史切片

因为 JSONL 是完整、不可变的，harness 可以在恢复时灵活地"读取子集"——跳过 `compact_boundary` 之前的旧历史，或从某个 `leaf` 节点往回追，或加载 sidechain 的 agent 子对话。文件是单一真相源，读取策略可以千变万化。

---

## 2. 核心定位与文件布局

每个 session 有一个**仅追加（append-only）**的主 JSONL 日志文件：

```
~/.claude/projects/<project-hash>/sessions/<sessionId>.jsonl
```

它是会话的**唯一持久化真相源**，用于进程崩溃或退出后通过 `--resume` / `--continue` 恢复完整状态。

### 2.1 子 agent（Sidechain）的独立文件

子 agent 的内部对话**不写主文件**，而是写到旁链（sidechain）文件中：

```
~/.claude/projects/<project-hash>/sessions/<sessionId>/subagents/agent-<agentId>.jsonl
```

- 旁链消息带有 `isSidechain: true` 和 `agentId` 字段。
- 主 session resume 时**不加载**旁链文件，子 agent 的聊天记录对主链透明。
- 子 agent 恢复（如用户显式点击继续）时，通过 `getAgentTranscriptPath(agentId)` 拼出路径，单独读取。

### 2.2 Fork Session 的新文件

`--fork-session` 不接管原 JSONL，而是保留启动时生成的新 `sessionId`，由 `useLogMessages` 在 REPL mount 时把旧消息逐条 `recordTranscript` 写入**新文件**。每条消息都会重新 stamp（`sessionId`、`timestamp`、`cwd` 等覆盖为 fork 后的值），确保 `content-replacement` 等元数据的 `sessionId` 键一致。

---

## 3. 写入模型：仅追加，不修改历史

### 3.1 写入器
`src/utils/sessionStorage.ts` 中的 `TranscriptWriter` 负责写入：

- 使用 `fs.appendFile` 批量追加新行。
- 带内存缓冲队列和定时 flush（默认异步批量写），不阻塞主线程。
- 日常对话运行时**只写不读**；内存中的 `mutableMessages` 是运行时真相源。

### 3.2 Compaction（上下文压缩）与日志的关系

无论发生哪种压缩，**已写入的 JSONL 行不会被修改或删除**：

- **MicroCompact**：仅把旧工具输出替换为占位符发送给 API，原始内容仍留在 JSONL 中。
- **传统 API 摘要压缩 / Session Memory Compact**：在内存里生成摘要，然后插入一条 `compact_boundary` 边界标记。resume 时通过边界标记跳过前面的旧历史，但文件本身不会重写。

> 唯一例外是 `removeMessageByUuid`，用于清理失败的流式消息孤立记录。它会做尾部 `truncate + write`，触发场景极少。

### 3.3 Fork Session 的写入

`--fork-session` 的写入与普通 resume 不同：

| 维度 | 普通 resume (`--resume`) | Fork resume (`--fork-session`) |
|------|-------------------------|-------------------------------|
| **Session ID** | 复用旧的 | 保留启动时生成的新 ID |
| **JSONL 文件** | 接管原来的（`adoptResumedSessionFile`） | **新文件**（`getTranscriptPath()` 按新 ID 推导） |
| **消息复制方式** | 已经在文件里，无需复制 | `useLogMessages` 在 REPL mount 时逐条 `recordTranscript` 写入 |
| **stamp 处理** | 原样保留 | 每条消息重新 stamp（`sessionId`、`timestamp`、`cwd` 覆盖为新值） |

重新 stamp 的原因：如果不覆盖 `sessionId`，新 JSONL 中的消息会带着旧 `sessionId`，而后续写入的 `content-replacement` 元数据是新 `sessionId`，导致 `loadFullLog` 按 `sessionId` 查找 replacement 时 miss，工具结果被误分类为 `FROZEN`，缓存失效。

---

## 4. 消费场景：何时读取这个文件

| 场景 | 读取方式 | 说明 |
|------|---------|------|
| **启动恢复** (`--continue`, `--resume`) | `loadConversationForResume` → `loadTranscriptFile` | 全量或按需解析 JSONL，沿 `parentUuid` 重建对话链 |
| **列表展示** (`/resume` picker) | `loadSameRepoMessageLogs` / `loadAllProjectsMessageLogs` | 先纯 `stat` 扫描，再对候选会话读 `head/tail`（约 16KB）提取标题和摘要 |
| **子 agent 恢复** | `getAgentTranscript` | 读取 sidechain 文件 `agents/<agentId>.jsonl` |

### 4.1 `loadTranscriptFile` 的加载策略

- 小文件（<5MB）：直接全量读取。
- 大文件（>5MB）：利用 `compact_boundary` 做前向跳读，跳过 pre-boundary 的过期内容，只保留 post-boundary 的消息和元数据。
- 然后通过 `buildConversationChain` 从最新 `leaf` 消息沿 `parentUuid` 往回追，重建完整对话链。

### 4.2 为什么 JSONL 不是单链表而是 DAG

正常对话时，每条消息的 `parentUuid` 指向上一条，文件里确实是个链表。但**rewind（回退）**会让它变成**有向无环图（DAG）**。

例如用户 rewind 到 B 后重新发了问题 E，append-only 不删旧数据：

```
A → B → C → D
      ↘
        E
```

- `C.parentUuid = B`，`E.parentUuid = B`
- `D.parentUuid = C`

`buildConversationChain` 恢复时，先从所有 terminal 消息（没有 child 的节点）中找到最新的 leaf（E），然后沿 `parentUuid` 往回追溯到 root。死分支（C → D）被丢弃。**物理存储是 DAG，逻辑恢复是单链。**

其他产生分支的情况：
- **Fork session**：复制现有链到新 session 继续
- **Snip（删除消息）**：`applySnipRemovals` 重连 parentUuid，被删的链段物理保留
- **并行 tool_use 拆分**：streaming 时一个 assistant 响应拆成 N 条消息，兄弟块通过 `message.id` 关联

---

## 5. JSONL 行类型

每一行是一个独立的 JSON 对象，分两大类：**消息类条目** 和 **元数据类条目**。

### 5.1 消息类条目（TranscriptMessage）

对应 `user / assistant / system / attachment / progress` 等消息，是文件中的主体内容。

#### 基础字段（来自 `Message`）

| 字段 | 说明 |
|------|------|
| `type` | `"user"` / `"assistant"` / `"system"` / `"attachment"` / `"progress"` / `"grouped_tool_use"` / `"collapsed_read_search"`。注意 `"system"` 是 UI 元数据（如 `api_metrics`、`compact_boundary`），**不是** API system prompt。 |
| `uuid` | 消息唯一标识 |
| `isMeta` | `user` 消息是否为合成提示（如 `/compact` 注入的指令） |
| `isCompactSummary` | 是否为压缩后的摘要消息 |
| `toolUseResult` | 工具执行结果（内部标记） |
| `isVisibleInTranscriptOnly` | 是否仅在 transcript 中可见 |
| `attachment` | `attachment` 类型时的附件对象 |
| `message` | `assistant` 类型时的 API 原始结构（`role`, `id`, `content`, `usage` 等） |

#### 序列化附加字段（SerializedMessage）

**每条消息都会重复带上这些 session 级字段**，因为 JSONL 不支持共享头：

| 字段 | 说明 |
|------|------|
| `cwd` | 当前工作目录 |
| `sessionId` | 所属会话 UUID |
| `timestamp` | ISO 格式写入时间 |
| `version` | CLI 版本号（如 `2.1.888`） |
| `userType` | 用户类型标识 |
| `entrypoint` | 入口点（`cli` / `sdk-ts` / `sdk-py` 等） |
| `gitBranch` | 当前 git 分支 |
| `slug` | 会话 slug（用于 plan 文件关联） |

#### 链式结构字段

| 字段 | 说明 |
|------|------|
| `parentUuid` | 父消息 UUID（`null` 表示链头） |
| `logicalParentUuid` | 逻辑父 UUID（compact boundary 时保留逻辑关系） |
| `isSidechain` | 是否为子 agent 的旁链消息 |
| `agentId` | 子 agent ID（旁链消息有值） |
| `teamName` | swarm team 名称 |
| `agentName` | agent 自定义名称 |
| `agentColor` | agent 颜色 |
| `promptId` | `user` 消息关联的 OTel `prompt.id` |

### 5.2 元数据类条目

所有元数据条目都有固定的 `type` discriminator，字段也是固定的。

| `type` | 记录的内容 | 写入时机 |
|--------|-----------|----------|
| `summary` | 某条对话链（leaf）的**摘要文本**。 | AgentTool 子 agent 结束时写入 sidechain，方便 resume 快速加载。 |
| `custom-title` | 用户给会话起的**自定义标题**。 | 执行 `/rename` 时写入；compact 前后和会话退出时会通过 `reAppendSessionMetadata` 重新刷到尾部，防止被截断丢失。 |
| `ai-title` | AI 自动生成的**会话标题**。 | AI 生成标题后写入一次；不会被重复刷新，后续可被新的 AI 标题覆盖，但不能覆盖用户自定义标题。 |
| `last-prompt` | 用户**最近一轮输入的文本**。 | 每轮用户发消息后缓存写入；compact/退出时也会刷新，用于 `/resume` 列表展示"最近做了什么"。 |
| `task-summary` | 后台任务（fork）的**阶段性摘要**。 | 长任务运行时每隔几分钟 fork 子进程生成并写入，用于 `claude ps` 展示实时状态。 |
| `tag` | 用户给会话打的**标签**。 | 执行 `/tag` 时写入；compact/退出时也会刷新。 |
| `agent-name` | Agent **自定义名称**。 | `/rename` 修改 agent 外观时写入；compact/退出时刷新。 |
| `agent-color` | Agent **颜色**。 | 同上。 |
| `agent-setting` | Agent **配置类型**。 | `--agent` 切换 agent 定义时写入；compact/退出时刷新。 |
| `pr-link` | 关联的 **GitHub PR** 信息。 | `/branch pr` 等命令关联 PR 时写入；compact/退出时刷新。 |
| `mode` | 会话运行**模式**（`coordinator` / `normal`）。 | 用户或系统切换模式时写入（如从 normal 切到 coordinator）。 |
| `worktree-state` | 当前是否处于 **worktree** 中。 | 进入或退出 worktree 时写入；值为对象表示"在 worktree 里"，值为 `null` 表示"已退出"。 |
| `content-replacement` | 某些工具结果被**替换为更小 stub** 的记录。 | 内容替换机制触发时写入；resume 时按记录把 stub 还原回原文，保证 prompt cache 稳定。 |
| `file-history-snapshot` | **文件编辑历史**的快照。 | 文件历史状态变更时写入，resume 后可继续追踪文件变更。 |
| `attribution-snapshot` | Claude 对代码的**字符级贡献量**快照。 | 文件大规模变更后写入，用于 `/commit` 时的 co-author 归因。 |
| `speculation-accept` | **推测执行**被用户接受的记录。 | prompt suggestion 的推测编辑被确认应用时写入，用于统计节省时间。 |
| `marble-origami-commit` | **Context Collapse** 提交（归档了哪些消息）。 | 每次 context-collapse 折叠一段消息时写入；resume 后 `/context` 能还原折叠视图。 |
| `marble-origami-snapshot` | **Context Collapse** 队列快照。 | staged collapse 状态变化后写入（last-wins）；resume 后恢复未完成的折叠调度。 |
| `queue-operation` | **消息队列**的操作记录。 | 异步消息队列处理时写入，resume 后重建队列状态。 |

---

## 6. 设计权衡

| 选择 | 原因 |
|------|------|
| **JSONL 而非 JSON** | 支持 append-only：每条消息追加一行，无需读取、解析、修改、重写整个文件。 |
| **每条消息自带 session 级字段** | 由于 append-only 无法共享文件头，只能用磁盘空间换写入性能。1000 轮会话会把 `sessionId` / `cwd` / `version` 重复写 1000 次。 |
| **compaction 不删旧数据** | 文件只增不减，通过 `compact_boundary` 在**读取时**跳过旧历史。保证崩溃安全：即使压缩过程中进程死掉，已写入的数据仍完整可用。 |
| **元数据重复 re-append** | 在 compact 前后和退出时，把 `custom-title`、`tag`、`last-prompt` 等重新刷到文件尾部，确保它们落在 `readLiteMetadata` 扫描的 tail 窗口内，否则 resume 列表会看不到。 |
