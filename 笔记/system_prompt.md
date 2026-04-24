# System prompt 相关笔记

- **Output style（输出风格）** 通过配置项 `**outputStyle`** 控制（字符串，写在 Claude Code 的 `**~/.claude/settings.json**` 或仓库 `**.claude/settings.json**` 里）。**默认不写或写 `"default"`** 时等价于「无额外风格」——不注入 Explanatory/Learning 那套追加 prompt（见 `[getOutputStyleConfig](../src/constants/outputStyles.ts)`）。**内置两种可选名**（需与代码里键名一致）：`**Explanatory`**、`**Learning**`；也可使用自定义 Markdown 样式（`~/.claude/output-styles/` 或项目 `.claude/output-styles/`），此时 `outputStyle` 填该样式的 `**name**`。示例：`"outputStyle": "Explanatory"`。

## 模型真正看到的结构（`callModel` 一刻）

`[QueryEngine](../src/QueryEngine.ts)` 把三份送进 `[query()](../src/query.ts)`，真正请求前在 `[query.ts](../src/query.ts)`（约 449–451、660 行）用 `[appendSystemContext` / `prependUserContext](../src/utils/api.ts)` 改成下面形状。

```text
systemPrompt:  [ getSystemPrompt 各段… ] [ 可选 memory ] [ 可选 append ] [ getSystemContext 打成的一段 ]
messages:       [ getUserContext + getCoordinatorUserContext → 一条 reminder ] + 真实轮次…
```

- `**getSystemPrompt` 各段…**：下文 `**## getSystemPrompt` 默认非 ant / ant 对照`** 大表里那些节，每项对应数组里的一段。  
- `**可选 memory【正常无】`**：`QueryEngine` 在**已设自定义 system** 且环境 `**CLAUDE_COWORK_MEMORY_PATH_OVERRIDE`** 时追加的 `**loadMemoryPrompt()**`，教模型怎么用外部记忆的读写规则；无独立 `##`。  
- `**可选 append**`：配置 `**appendSystemPrompt**`（CLI/`main.tsx` 等还可能把 Chrome、proactive 等**拼进同一段字符串**），无独立 `##`。  
- `**getSystemContext` 打成的一段`**：下文 **`## getSystemContext`**；多键合成**一条**` 键: 值` 文本，挂在 **system 数组末尾**。  
- **一条 reminder**：`[prependUserContext](../src/utils/api.ts)` 插在 `**messages` 最前**的一条 **user、meta** 消息，外层是 `<system-reminder>…</system-reminder>`，里面是 `**# claudeMd`**、`**# currentDate**` 等；正文来自下文 `**## getUserContext**`，协调器时在同一包里多 `**# workerToolsContext**`（`**## getCoordinatorUserContext**` 已 merge 进 `userContext`）。

**特例**：若 `**customSystemPrompt**` 存在，`[fetchSystemPromptParts](../src/utils/queryContext.ts)` 会 `**defaultSystemPrompt = []**`、`**systemContext = {}**`，则上面示意里 `**getSystemPrompt` 默认多段**与 **getSystemContext 段**常空掉，但 **memory / append** 仍可按条件出现。

## Coordinator 模式（与下面 `getSystemPrompt` 大表的关系）

**生效条件（简述）**：`feature('COORDINATOR_MODE')` 且环境 `CLAUDE_CODE_COORDINATOR_MODE` 等为真（`isCoordinatorMode()`），且主线程**没有** `mainThreadAgentDefinition`（见 `[buildEffectiveSystemPrompt](../src/utils/systemPrompt.ts)`）。

**两条路径不一致（当前实现）**：

| 路径 | system 侧 | user reminder 侧 |
|------|-----------|------------------|
| **REPL** | 用 `[getCoordinatorSystemPrompt](../src/coordinator/coordinatorMode.ts)` **整段替换**下文大表里由 `getSystemPrompt` 产出的默认多段（Intro、# System、Doing tasks、Using tools、动态段等一律不再出现）；**仍保留** `appendSystemPrompt` 追加在 coordinator 正文之后。 | 与常规模块相同，并合并 `getCoordinatorUserContext` → `# workerToolsContext`（见下文 **## getCoordinatorUserContext**）。 |
| **QueryEngine（SDK ask）** | **不**走 `buildEffectiveSystemPrompt`，仍按 `fetchSystemPromptParts` + `asSystemPrompt` 组装（即仍是 `getSystemPrompt` 默认表那一套 + 可选 custom/memory/append），**没有** coordinator 主编排正文。 | 同样合并 `getCoordinatorUserContext`，故 reminder 里仍有 `# workerToolsContext`。 |

**读表注意**：一旦 REPL 进入上述 coordinator 分支，下面「`getSystemPrompt` 默认非 ant / ant 对照」大表描述的是**被替换掉、不再进 system 数组**的内容；协调器专属正文以 `coordinatorMode.ts` 为准。

## `getSystemPrompt` 默认非 ant / ant 对照

说明：

- 这里对应 [src/utils/queryContext.ts](../src/utils/queryContext.ts) 调到的 `[getSystemPrompt](../src/constants/prompts.ts)`。
- 只看默认主路径：不考虑 `CLAUDE_CODE_SIMPLE`、proactive/kairos 分支。
- “默认”表示很多动态 section 实际可能是 `null`，比如 language/output style/MCP/memory/scratchpad/FRC/token budget；表里写的是“这个模块会返回什么”，以及 ant / 非 ant 的差异。


| 模块                               | 非 ant 原字符串                                                                                                                                                                                                                                                                                                                                                                                | ant 原字符串                                                                                                                                                                                                                                       | 中文注释 / 区别                                                                                                                                           |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| Intro                            | `You are an interactive agent that helps users with software engineering tasks. Use the instructions below and the tools available to you to assist the user.` 加上 `CYBER_RISK_INSTRUCTION` 和 `IMPORTANT: You must NEVER generate or guess URLs...`，见 [prompts.ts:179](../src/constants/prompts.ts)                                                                                        | 同左                                                                                                                                                                                                                                             | Intro 本身不分 ant / 非 ant。差异只来自 `outputStyleConfig !== null` 时首句会变成 `according to your "Output Style" below...`。                                       |
| `# System`                       | `All text you output outside of tool use is displayed to the user...` / `Tools are executed in a user-selected permission mode...` / `Tool results and user messages may include <system-reminder>...` / `Tool results may include data from external sources...` / hooks 说明 / `The system will automatically compress prior messages...`，见 [prompts.ts:186](../src/constants/prompts.ts) | 同左                                                                                                                                                                                                                                             | 完全相同，都是系统级行为约束。                                                                                                                                     |
| `# Doing tasks` 基础部分             | 包含：做软件工程任务、默认按工程语境理解模糊请求、先读代码再改、尽量不新建文件、不做时间预估、失败先诊断、注意安全、不要保留兼容性 hack、`/help`/反馈指引，见 [prompts.ts:221](../src/constants/prompts.ts)                                                                                                                                                                                                                                                       | 同左，再额外插入多条 ant 专属规则，见下几行                                                                                                                                                                                                                       | 两边共享同一个大模块，但 ant 会塞入更多“判断力、验证、忠实汇报”要求。                                                                                                              |
| `# Doing tasks` 注释策略             | 无 ant 专属注释规则                                                                                                                                                                                                                                                                                                                                                                              | `Default to writing no comments...` / `Don't explain WHAT the code does...` / `Don't remove existing comments unless...` / `Before reporting a task complete, verify it actually works...`，见 [prompts.ts:205](../src/constants/prompts.ts)     | ant 更强约束“少注释、注释只写 WHY、完成前必须验证”。非 ant 没这些硬限制。                                                                                                        |
| `# Doing tasks` 纠偏 / 主动指出误解      | 无                                                                                                                                                                                                                                                                                                                                                                                         | `If you notice the user's request is based on a misconception, or spot a bug adjacent to what they asked about, say so...`，见 [prompts.ts:225](../src/constants/prompts.ts)                                                                     | ant 更强调协作者姿态，不只是执行，还要指出用户误解或邻近问题。                                                                                                                   |
| `# Doing tasks` 忠实汇报             | 无                                                                                                                                                                                                                                                                                                                                                                                         | `Report outcomes faithfully: if tests fail, say so... Never claim "all tests pass" when output shows failures...`，见 [prompts.ts:237](../src/constants/prompts.ts)                                                                              | ant 明确要求不能把失败说成成功，也不能把未验证说成已验证。                                                                                                                     |
| `# Doing tasks` Claude Code 问题反馈 | 无                                                                                                                                                                                                                                                                                                                                                                                         | `If the user reports a bug, slowness, or unexpected behavior with Claude Code itself... recommend /issue or /share...`，见 [prompts.ts:243](../src/constants/prompts.ts)                                                                         | 只在 ant 下引导产品反馈闭环。                                                                                                                                   |
| `# Executing actions with care`  | `Carefully consider the reversibility and blast radius of actions...` 后面跟高风险操作举例和确认原则，见 [prompts.ts:255](../src/constants/prompts.ts)                                                                                                                                                                                                                                                     | 同左                                                                                                                                                                                                                                             | 完全相同，都是风险操作先确认。                                                                                                                                     |
| `# Using your tools`             | `Do NOT use the Bash tool when a relevant dedicated tool is provided...`、读/改/写/搜文件优先专用工具、可并行调用工具、任务管理工具提示，见 [prompts.ts:269](../src/constants/prompts.ts)                                                                                                                                                                                                                                 | 基本同左                                                                                                                                                                                                                                           | 主体逻辑相同。差异更多取决于运行态，例如 ant-native 可能 `hasEmbeddedSearchTools()` 为真，于是少了 `Glob/Grep` 那两句，改成依赖 bash 内的 `find/grep`。这不是 `USER_TYPE===ant` 直接分支，而是工具集合差异。 |
| `# Tone and style` 共同项           | `Only use emojis if the user explicitly requests it...` / `When referencing specific functions... file_path:line_number` / `When referencing GitHub issues... owner/repo#123` / `Do not use a colon before tool calls...`，见 [prompts.ts:430](../src/constants/prompts.ts)                                                                                                                 | 同左，但少一条“短而简洁”                                                                                                                                                                                                                                  | 共有大部分格式规则。                                                                                                                                          |
| `# Tone and style` 简洁度要求         | 额外有：`Your responses should be short and concise.`，见 [prompts.ts:433](../src/constants/prompts.ts)                                                                                                                                                                                                                                                                                         | 无这一句                                                                                                                                                                                                                                           | 非 ant 明确再压一层“短和简洁”；ant 不放这句，因为 ant 的沟通要求是“清晰优先，不是越短越好”。                                                                                             |
| 输出效率 / 用户沟通 section              | `# Output efficiency`：`IMPORTANT: Go straight to the point... Be extra concise.` / `Keep your text output brief and direct...` / `If you can say it in one sentence, don't use three.`，见 [prompts.ts:416](../src/constants/prompts.ts)                                                                                                                                                    | `# Communicating with the user`：`When sending user-facing text, you're writing for a person, not logging to a console...` / 要求首次 tool call 前说明、关键节点更新、完整句、少 jargon、按用户水平调节解释深度、强调“可理解性优先于极致简短”，见 [prompts.ts:404](../src/constants/prompts.ts) | 这是两边最大差异。非 ant 的目标是“快、短、直给”；ant 的目标是“写给人看得懂”，允许略长，但要求更可读、更像真实协作更新。                                                                                  |
| `# Session-specific guidance`    | 取决于工具集和会话状态，可能包含：`If you do not understand why the user has denied a tool call...`、`! <command>`、AgentTool / Explore agent / SkillTool / DiscoverSkills 指引，见 [prompts.ts:352](../src/constants/prompts.ts)                                                                                                                                                                                | 基本同左，但在满足 `feature('VERIFICATION_AGENT') && tengu_hive_evidence` 时，可能额外出现一整段 verification contract，见 [prompts.ts:390](../src/constants/prompts.ts)                                                                                             | 这个模块不是简单按 ant / 非 ant 分叉，而是看工具和特性。最大差异是 verification agent contract 基本是 ant-only A/B。                                                               |
| `memory`                         | `loadMemoryPrompt()` 的返回值，见 [prompts.ts:495](../src/constants/prompts.ts)                                                                                                                                                                                                                                                                                                                 | 同左                                                                                                                                                                                                                                             | 是否有内容取决于记忆系统，不取决于 ant。                                                                                                                              |
| `ant_model_override`             | `null`，见 [prompts.ts:136](../src/constants/prompts.ts)                                                                                                                                                                                                                                                                                                                                    | `getAntModelOverrideConfig()?.defaultSystemPromptSuffix                                                                                                                                                                                        |                                                                                                                                                     |
| `# Environment`                  | 由 `computeSimpleEnvInfo(...)` 生成：`# Environment`、cwd、git/worktree、additional dirs、platform、shell、OS、model 描述、knowledge cutoff、最新 Claude family / Claude Code 形态 / fast mode 说明，见 [prompts.ts:705](../src/constants/prompts.ts)                                                                                                                                                            | 基本同左                                                                                                                                                                                                                                           | 默认大体相同。只有 `isUndercover()` 时 ant 会去掉 model 相关说明；你问的是默认状态，所以可理解为两边基本一致。                                                                              |
| `# Language`                     | `Always respond in ${languagePreference}...`，若设置了语言偏好，见 [prompts.ts:142](../src/constants/prompts.ts)                                                                                                                                                                                                                                                                                     | 同左                                                                                                                                                                                                                                             | 与 ant 无关。                                                                                                                                           |
| `# Output Style`                 | `# Output Style: ${outputStyleConfig.name}` 加上 style prompt，若配置了 output style，见 [prompts.ts:151](../src/constants/prompts.ts)                                                                                                                                                                                                                                                             | 同左                                                                                                                                                                                                                                             | 与 ant 无关。                                                                                                                                           |
| `# MCP Server Instructions`      | `# MCP Server Instructions` 加上每个 MCP server 的 `instructions`，若有已连接 MCP 且提供说明，见 [prompts.ts:160](../src/constants/prompts.ts)                                                                                                                                                                                                                                                              | 同左                                                                                                                                                                                                                                             | 与 ant 无关。                                                                                                                                           |
| `# Scratchpad Directory`         | 若 `isScratchpadEnabled()`，返回 `IMPORTANT: Always use this scratchpad directory...`，见 [prompts.ts:797](../src/constants/prompts.ts)                                                                                                                                                                                                                                                         | 同左                                                                                                                                                                                                                                             | 与 ant 无关。                                                                                                                                           |
| `# Function Result Clearing`     | 默认通常无；若 `feature('CACHED_MICROCOMPACT')` 打开且配置允许，返回 `Old tool results will be automatically cleared from context...`，见 [prompts.ts:822](../src/constants/prompts.ts)                                                                                                                                                                                                                      | 同左                                                                                                                                                                                                                                             | 与 ant 无直接分支关系；但实际功能更偏 ant-only 环境可用。                                                                                                                |
| `summarize_tool_results`         | `When working with tool results, write down any important information you might need later in your response, as the original tool result may be cleared later.`，见 [prompts.ts:833](../src/constants/prompts.ts)                                                                                                                                                                           | 同左                                                                                                                                                                                                                                             | 完全相同。                                                                                                                                               |
| `numeric_length_anchors`         | 无                                                                                                                                                                                                                                                                                                                                                                                         | `Length limits: keep text between tool calls to ≤25 words. Keep final responses to ≤100 words unless the task requires more detail.`，见 [prompts.ts:529](../src/constants/prompts.ts)                                                           | ant-only。很有意思的是，ant 一边强调“写给人看”，另一边又加了数值长度锚点，说明它想要的是“清晰但不要啰嗦”。                                                                                       |
| `token_budget`                   | 若 `feature('TOKEN_BUDGET')` 打开，返回：`When the user specifies a token target... Keep working until you approach the target...`，见 [prompts.ts:538](../src/constants/prompts.ts)                                                                                                                                                                                                               | 同左                                                                                                                                                                                                                                             | 与 ant 无关，属于预算驱动续跑。                                                                                                                                  |
| `brief`                          | `...(feature('KAIROS') \|\| feature('KAIROS_BRIEF') ? [systemPromptSection('brief', () => getBriefSection())] : [])`，见 [prompts.ts:552](../src/constants/prompts.ts)                                                                                                                                                                                                                     | 同左                                                                                                                                                                                                                                             | 受 KAIROS/KAIROS_BRIEF 门控；实际字符串内容来自 `getBriefSection()`，非 ant / ant 在这里无直接分支。                                                                                                 |


## `getSystemContext`

见 `[context.ts](../src/context.ts)` 中 `getSystemContext`（约 113–151 行）。**会话级 memoize**，同一对话只算一次。返回 `Promise<Record<string, string>>`；另若开了 `**BREAK_CACHE_COMMAND`** 且设置了 `setSystemPromptInjection`，可能多一个 `**cacheBreaker`: `[CACHE_BREAKER: …]**`（打断 prompt 缓存，ant 调试向），略过即可。

**主体是 `gitStatus` 字符串**（来自同文件里的 `getGitStatus`，约 36–111 行）：满足 **非** `CLAUDE_CODE_REMOTE`、`shouldIncludeGitInstructions()` 为真、且当前目录是 git 仓库且未在测试中出错时，才会放进返回对象。含义是**对话开始瞬间**的仓库快照，**会话内不会刷新**。

`**gitStatus` 正文结构**（各段之间 `\n\n` 拼接，整体为英文）：

1. 固定说明一句：`This is the git status at the start of the conversation. Note that this status is a snapshot in time, and will not update during the conversation.`
2. `Current branch: <getBranch()>`
3. `Main branch (you will usually use this for PRs): <getDefaultBranch()>`
4. 若配置了 `git config user.name`：`Git user: <name>`（无则整段省略）
5. `Status:` 下一行起为 `**git status --short`** 的 stdout（空则写 `(clean)`）；超过 **2000 字符**会截断并追加一句提示用 Bash 跑完整 `git status`
6. `Recent commits:` 下一行起为 `**git log --oneline -n 5`** 的 stdout

底层并行命令见源码 `Promise.all`：`getBranch`、`getDefaultBranch`、`status --short`、`log -n 5`、`config user.name`。

## `getUserContext`

见 [context.ts](../src/context.ts) 中 `getUserContext`；`claudeMd` 正文由 `[getClaudeMds(filterInjectedMemoryFiles(await getMemoryFiles()))](../src/utils/claudemd.ts)` 生成。会话内 memoize 缓存。

**两部分：**

1. `**claudeMd`（可选）**
  - 前缀固定为 `MEMORY_INSTRUCTION_PROMPT`：  
   `Codebase and user instructions are shown below. Be sure to adhere to these instructions. IMPORTANT: These instructions OVERRIDE any default behavior and you MUST follow them exactly as written.`  
  - 其后按 `[getMemoryFiles](../src/utils/claudemd.ts)` 得到的 `**MemoryFileInfo[]` 顺序**，每条展开为 `Contents of <path> (说明):\n\n正文`（TeamMem 会包一层 XML），段与段之间 `\n\n` 拼接。  
  - **顺序含义**：数组为**低优先级 → 高优先级**；**越靠后的文件在全文里越靠后**（通常更「当道」）。概要顺序：**Managed（含托管侧 `.claude/rules/*.md`）→ User（`~/.claude/CLAUDE.md` 与用户 `~/.claude/rules/*.md`）→ 从盘符根沿路径向下直到 CWD**：每层依次 `**CLAUDE.md`**、`**.claude/CLAUDE.md**`、`**.claude/rules/**/*.md**`、`**CLAUDE.local.md**` → 若开启 env，`**--add-dir` 附加目录**上同样四类 → **AutoMem 入口（如 memory.md）** → **TeamMem**。`@include` 会把被包含文件**插到包含文件之前**。`filterInjectedMemoryFiles` 在 GrowthBook `tengu_moth_copse` 为真时可去掉 AutoMem/TeamMem。`context.ts` 里若禁用 CLAUDE 发现（如 `CLAUDE_CODE_DISABLE_CLAUDE_MDS`，或 `--bare` 且无附加目录）则整段 `claudeMd` 不出现。细节以 `claudemd.ts` 文件头注释与 `getMemoryFiles` 实现为准。
2. `**currentDate`（必有）**
  `Today's date is ${getLocalISODate()}.`

## `getCoordinatorUserContext`

定义见 `[coordinatorMode.ts` 中 `getCoordinatorUserContext](../src/coordinator/coordinatorMode.ts)`（约 80–109 行）。**仅在协调器模式**（`feature('COORDINATOR_MODE')` 且 `CLAUDE_CODE_COORDINATOR_MODE` 等使 `isCoordinatorMode()` 为真）下非空；否则返回 `**{}`**。  
`[QueryEngine.ts](../src/QueryEngine.ts)` 会把返回值 **spread 进 `userContext`**，与 `getUserContext` 的 `baseUserContext` 合并。

**返回结构：** `{ workerToolsContext: string }`（单键）。

---

### 第 1 段（必有，协调器模式下）

模板（`${AGENT_TOOL_NAME}` 在源码中为 `**Agent`**）：

```text
Workers spawned via the Agent tool have access to these tools: <workerTools>
```

其中 `**<workerTools>**` 为**排序后逗号+空格连接**的工具名字符串：

- `**CLAUDE_CODE_SIMPLE` 为真时**（字母序）：  
`Bash, Edit, Read`
- **否则**（默认）：对 `[ASYNC_AGENT_ALLOWED_TOOLS](../src/constants/tools.ts)` 中每一项，若不在 `[INTERNAL_WORKER_TOOLS](../src/coordinator/coordinatorMode.ts)` 中则保留，再 `.sort().join(', ')`。当前集合与内部 worker 集合的交集实际只有 `**StructuredOutput`**（`SYNTHETIC_OUTPUT_TOOL_NAME`），故等价于从 async 列表去掉该项。按当前源码，结果为（字母序，一行）：  
  `Bash, Edit, EnterWorktree, ExitWorktree, Glob, Grep, NotebookEdit, PowerShell, Read, Skill, TodoWrite, ToolSearch, WebFetch, WebSearch, Write`

---

### 第 2 段（可选：`mcpClients.length > 0`）

```text


Workers also have access to MCP tools from connected MCP servers: <serverNames>
```

`<serverNames>` 为当前已连接 MCP 的 `**c.name**` 用 `**,**`  拼接。

---

### 第 3 段（可选：scratchpad）

当**同时**满足：`scratchpadDir` 传入非空（`QueryEngine` 侧为 `isScratchpadEnabled() ? getScratchpadDir() : undefined`），且 GrowthBook 门控 `**tengu_scratch`** 为真（`isScratchpadGateEnabled()`）时，再追加：

```text


Scratchpad directory: <scratchpadDir>
Workers can read and write here without permission prompts. Use this for durable cross-worker knowledge — structure files however fits the work.
```

---

### 完整示例（非 SIMPLE、无 MCP、无 scratchpad 段）

`workerToolsContext` 整段为**一行**（仅第 1 段）：

```text
Workers spawned via the Agent tool have access to these tools: Bash, Edit, EnterWorktree, ExitWorktree, Glob, Grep, NotebookEdit, PowerShell, Read, Skill, TodoWrite, ToolSearch, WebFetch, WebSearch, Write
```

### 完整示例（带 MCP 名 `foo` 与 scratchpad 路径）

```text
Workers spawned via the Agent tool have access to these tools: Bash, Edit, EnterWorktree, ExitWorktree, Glob, Grep, NotebookEdit, PowerShell, Read, Skill, TodoWrite, ToolSearch, WebFetch, WebSearch, Write

Workers also have access to MCP tools from connected MCP servers: foo

Scratchpad directory: /path/to/scratchpad
Workers can read and write here without permission prompts. Use this for durable cross-worker knowledge — structure files however fits the work.
```

