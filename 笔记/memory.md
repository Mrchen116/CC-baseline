# Memory 相关笔记

## Session Memory

Session Memory 是一份由后台自动维护的 **markdown 笔记**，记录当前会话的核心上下文（任务目标、文件改动、错误修复、工作流等）。它不是给人类看的 transcript，而是给模型在 compaction 时快速恢复上下文的"速查手册"。

其工作流是：

1. 主会话每轮对话后，后台异步 fork 一个子 agent
2. 子 agent 读取现有的 `summary.md`，根据最新对话内容用 `Edit` 工具更新它
3. 当主会话上下文快满触发 compaction 时，直接用这份 `summary.md` 替代"现场总结整段历史"，从而避免再走一次 LLM summary

### 启用条件

**默认关闭。** 需要 GrowthBook flag `tengu_session_memory` 和 `tengu_sm_compact` 同时为 `true`，或设环境变量 `ENABLE_CLAUDE_CODE_SM_COMPACT=1`（`[sessionMemoryCompact.ts:403-420](../src/services/compact/sessionMemoryCompact.ts)`）。只在主 REPL 线程生效，子 agent、bridge 模式、remote mode 都会跳过。

### 生成/更新时机

SessionMemory 的生成有 3 个时机：

1. **自动后台提取（主要方式）**
  - `initSessionMemory()` 在启动时注册 `postSamplingHook`（`[sessionMemory.ts:357-374](../src/services/SessionMemory/sessionMemory.ts)`）
  - 每次 REPL 采样完成后，`extractSessionMemory` hook 被触发（`[sessionMemory.ts:272](../src/services/SessionMemory/sessionMemory.ts)`）
  - 核心判断在 `shouldExtractMemory()`（`[sessionMemory.ts:134-181](../src/services/SessionMemory/sessionMemory.ts)`）：
    - **首次初始化**：上下文 token 数达到 `minimumMessageTokensToInit`（默认 **10000 tokens**，见 `[sessionMemoryUtils.ts:33](../src/services/SessionMemory/sessionMemoryUtils.ts)`）。这是为了避免刚开几句对话就急着记笔记。
    - **后续更新**：需同时满足（`[sessionMemory.ts:168-170](../src/services/SessionMemory/sessionMemory.ts)`）
      - 距离上次提取的上下文增长了 `minimumTokensBetweenUpdate`（默认 **5000 tokens**，`[sessionMemoryUtils.ts:34](../src/services/SessionMemory/sessionMemoryUtils.ts)`）
      - 期间发生了 `toolCallsBetweenUpdates` 次工具调用（默认 **3 次**，`[sessionMemoryUtils.ts:35](../src/services/SessionMemory/sessionMemoryUtils.ts)`）
    - **自然断点提取**：token 增长达标且上一轮助手**没有工具调用**（`hasMetTokenThreshold && !hasToolCallsInLastTurn`）。意思是如果用户和模型纯聊天没有触发工具，也会趁机更新一次，避免有内容遗漏。
  - 提取过程异步 fork 子 agent（`runForkedAgent`，`querySource: 'session_memory'`），用 `Edit` 工具修改 summary.md，不阻塞主对话（`[sessionMemory.ts:318-325](../src/services/SessionMemory/sessionMemory.ts)`）
2. **手动触发**
  - `/summary` 命令调用 `manuallyExtractSessionMemory()`（`[sessionMemory.ts:387](../src/services/SessionMemory/sessionMemory.ts)`），跳过所有阈值检查立即提取一次。
3. **Compaction 时消费**
  - `compact` 命令中，若启用了 `sessionMemoryCompaction`，先调用 `trySessionMemoryCompaction()`（`[compact.ts:58](../src/commands/compact/compact.ts)` → `[sessionMemoryCompact.ts](../src/services/compact/sessionMemoryCompact.ts)`）
  - 会先 `waitForSessionMemoryExtraction()` 等待后台提取完成（最多 15s，`[sessionMemoryUtils.ts:12](../src/services/SessionMemory/sessionMemoryUtils.ts)`）
  - 然后读取 `summary.md`，并用 `truncateSessionMemoryForCompact()` 硬截断超长 section（`[prompts.ts:256-296](../src/services/SessionMemory/prompts.ts)`），再插入 compact 消息中替代传统 summary

### 存储位置

- 路径：`{projectDir}/{sessionId}/session-memory/summary.md`（`[filesystem.ts:269-270](../src/utils/permissions/filesystem.ts)`）
- 目录权限 `0o700`，文件权限 `0o600`（`[sessionMemory.ts:190-206](../src/services/SessionMemory/sessionMemory.ts)`）

### Prompt 与模板

#### 默认模板结构

首次创建文件时，系统会写入 `DEFAULT_SESSION_MEMORY_TEMPLATE`（`[prompts.ts:11](../src/services/SessionMemory/prompts.ts)`）。模板有固定格式：一个 `#`  标题 + 一行 `_斜体说明_`，下面是内容区：

```markdown
# Session Title
_A short and distinctive 5-10 word descriptive title for the session. Super info dense, no filler_

# Current State
_What is actively being worked on right now? Pending tasks not yet completed. Immediate next steps._

# Task specification
_What did the user ask to build? Any design decisions or other explanatory context_

# Files and Functions
_What are the important files? In short, what do they contain and why are they relevant?_

# Workflow
_What bash commands are usually run and in what order? How to interpret their output if not obvious?_

# Errors & Corrections
_Errors encountered and how they were fixed. What did the user correct? What approaches failed and should not be tried again?_

# Codebase and System Documentation
_What are the important system components? How do they work/fit together?_

# Learnings
_What has worked well? What has not? What to avoid? Do not duplicate items from other sections_

# Key results
_If the user asked a specific output such as an answer to a question, a table, or other document, repeat the exact result here_

# Worklog
_Step by step, what was attempted, done? Very terse summary for each step_
```

#### 更新用的 Prompt

子 agent 收到的默认 prompt 在 `getDefaultUpdatePrompt()`（`[prompts.ts:43](../src/services/SessionMemory/prompts.ts)`）中定义。核心指令包括：

- **唯一任务**：用 `Edit` 工具更新 `summary.md`，然后停止。可以并行做多个 Edit。
- **结构铁律**：**绝对不能**修改/删除/新增 section headers（`#`  开头的行），也**绝对不能**修改/删除斜体描述行（`_..._`）。这些模板行必须原样保留，子 agent 只能编辑它们下方的实际内容。
- **内容要求**：要详细、信息密集，包含文件路径、函数名、错误信息、命令、技术细节等。"Key results" 要放完整输出。"Current State" 必须反映最新工作状态。
- **跳过策略**：如果某个 section 没有新内容，直接跳过，不要写"No info yet"之类的 filler。
- **长度限制**：每个 section 约 **2000 tokens**。如果接近上限，要淘汰次要细节、保留最关键信息。
- **不要重复**：已经写在 CLAUDE.md 里的内容不要重复记。
- **隔离意识**：prompt 中明确提醒"这条消息和指令不是真实用户对话的一部分"，所以笔记里不能出现"note-taking"、"session notes extraction"之类的元描述。

此外，prompt 会动态拼接两段长度提醒：

1. **单 section 超限提醒**：统计每个 section 的 token 数，超过 2000 的会列出警告
2. **总预算超限提醒**：如果整文件超过 **12000 tokens**，会加 `CRITICAL:` 提示必须压缩

#### 自定义

用户可以用自己的模板和 prompt 覆盖默认行为：

- 自定义模板：`~/.claude/session-memory/config/template.md`
- 自定义 prompt：`~/.claude/session-memory/config/prompt.md`
- 自定义 prompt 支持 `{{variableName}}` 变量替换，目前可用的变量有 `{{currentNotes}}`（当前笔记内容）和 `{{notesPath}}`（笔记文件路径）

### 内容截断（truncateSessionMemoryForCompact）

在 compaction 时将 session memory 硬截断到合理长度，防止占满 post-compact token 预算（`[prompts.ts:249-253](../src/services/SessionMemory/prompts.ts)`）。

- 按 `#`  section 分割
- 每 section 内容超过约 **8000 字符**（`MAX_SECTION_LENGTH * 4`，即约 2000 tokens）时截断，保留边界附近的整行，并追加 `[... section truncated for length ...]`（`[prompts.ts:260-323](../src/services/SessionMemory/prompts.ts)`）
- 如果整文件超过 **12000 tokens**，会触发上面的 `CRITICAL` 提示让子 agent 主动压缩，但 compaction 时仍会做硬截断兜底

