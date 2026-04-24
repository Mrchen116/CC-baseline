# Memory 相关笔记

- **Prompt 与评测（eval）**：系统提示里的文案并非随手写的；很多段落会注明经评测验证（例如 `Eval-validated (memory-prompt-iteration.eval.ts, …)`）。对比会细到**章节标题**：同一段正文，只改标题（如「Before recommending from memory」vs 更抽象的「Trusting what you recall」），效果会差很多——注释里写明一类标题在 append 场景下 3/3，另一类 0/3。详见 [memoryTypes.ts 中 `TRUSTING_RECALL_SECTION` 上方注释](../src/memdir/memoryTypes.ts)（约 224–239、241–244 行）。

## Session Memory

### 生成/更新时机

SessionMemory 的生成有 3 个时机：

1. **自动后台提取（主要方式）**
   - `initSessionMemory()` 在启动时注册 `postSamplingHook`（[`sessionMemory.ts:357-374`](../src/services/SessionMemory/sessionMemory.ts)）
   - 每次 REPL 采样完成后，`extractSessionMemory` hook 被触发（[`sessionMemory.ts:272`](../src/services/SessionMemory/sessionMemory.ts)）
   - 核心判断在 `shouldExtractMemory()`（[`sessionMemory.ts:134-181`](../src/services/SessionMemory/sessionMemory.ts)）：
     - **首次初始化**：上下文 token 数达到 `minimumMessageTokensToInit`（默认 **10000 tokens**，见 [`sessionMemoryUtils.ts:33`](../src/services/SessionMemory/sessionMemoryUtils.ts)）
     - **后续更新**：需同时满足（[`sessionMemory.ts:168-170`](../src/services/SessionMemory/sessionMemory.ts)）
       - 距离上次提取的上下文增长了 `minimumTokensBetweenUpdate`（默认 **5000 tokens**，[`sessionMemoryUtils.ts:34`](../src/services/SessionMemory/sessionMemoryUtils.ts)）
       - 期间发生了 `toolCallsBetweenUpdates` 次工具调用（默认 **3 次**，[`sessionMemoryUtils.ts:35`](../src/services/SessionMemory/sessionMemoryUtils.ts)）
     - **自然断点提取**：token 增长达标且上一轮助手**没有工具调用**（`hasMetTokenThreshold && !hasToolCallsInLastTurn`）
   - 提取过程异步 fork 子 agent（`runForkedAgent`，`querySource: 'session_memory'`），用 `Edit` 工具修改 summary.md，不阻塞主对话（[`sessionMemory.ts:318-325`](../src/services/SessionMemory/sessionMemory.ts)）

2. **手动触发**
   - `/summary` 命令调用 `manuallyExtractSessionMemory()`（[`sessionMemory.ts:387`](../src/services/SessionMemory/sessionMemory.ts)），跳过所有阈值检查立即提取一次。

3. **Compaction 时消费**
   - `compact` 命令中，若启用了 `sessionMemoryCompaction`，先调用 `trySessionMemoryCompaction()`（[`compact.ts:58`](../src/commands/compact/compact.ts) → [`sessionMemoryCompact.ts`](../src/services/compact/sessionMemoryCompact.ts)）
   - 会先 `waitForSessionMemoryExtraction()` 等待后台提取完成（最多 15s，[`sessionMemoryUtils.ts:12`](../src/services/SessionMemory/sessionMemoryUtils.ts)）
   - 然后读取 `summary.md`，并用 `truncateSessionMemoryForCompact()` 硬截断超长 section（[`prompts.ts:256-296`](../src/services/SessionMemory/prompts.ts)），再插入 compact 消息中替代传统 summary

### 存储位置

- 路径：`{projectDir}/{sessionId}/session-memory/summary.md`（[`filesystem.ts:269-270`](../src/utils/permissions/filesystem.ts)）
- 目录权限 `0o700`，文件权限 `0o600`（[`sessionMemory.ts:190-206`](../src/services/SessionMemory/sessionMemory.ts)）

### 内容截断（truncateSessionMemoryForCompact）

- 在 compaction 时将 session memory 硬截断到合理长度，防止占满 post-compact token 预算（[`prompts.ts:249-253`](../src/services/SessionMemory/prompts.ts)）
- 按 `# ` section 分割，每 section 内容超过约 **8000 字符**（`MAX_SECTION_LENGTH * 4`，即约 2000 tokens）时截断，并追加 `[... section truncated for length ...]`（[`prompts.ts:260-323`](../src/services/SessionMemory/prompts.ts)）

