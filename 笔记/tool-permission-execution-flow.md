# Bash 权限管理完整流程

> 分析版本: 2026-04-29
> 涉及文件: `src/query.ts`, `src/services/tools/toolExecution.ts`, `src/services/tools/StreamingToolExecutor.ts`, `src/hooks/useCanUseTool.tsx`, `src/tools/BashTool/bashPermissions.ts`, `src/utils/permissions/permissions.ts`

---

## 一、总览

当 Claude API 返回一个 `tool_use` 块（如 bash 命令）时，系统不会立即执行。而是通过多层权限检查决定是否：直接放行(`allow`)、直接拒绝(`deny`)、或向用户弹窗请示(`ask`)。

核心设计目标：**延迟最小化** + **安全可控**。通过 speculative classifier 在后台预跑 allow 分类器，多数安全命令可以跳过弹窗。

---

## 二、完整流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           STAGE 1: 流式触发阶段                              │
└─────────────────────────────────────────────────────────────────────────────┘

    Claude API 流式返回 message
           │
           ▼
    ┌──────────────┐
    │ query.ts:830 │  msgToolUseBlocks = extractToolUseBlocks(message)
    └──────┬───────┘
           │
           ▼
    ┌────────────────────────────┐
    │ streamingToolExecutor.     │  StreamingToolExecutor.ts
    │   addTool(toolBlock, msg)  │  将 tool_use 块加入并发执行队列
    └─────────────┬──────────────┘
                  │
                  ▼
    ┌────────────────────────────┐
    │ StreamingToolExecutor.     │  StreamingToolExecutor.ts
    │   executeTool(toolBlock)   │  队列调度，取出待执行 tool
    └─────────────┬──────────────┘
                  │
                  ▼
    ┌────────────────────────────┐
    │ runToolUse()               │  toolExecution.ts · 解析 input schema
    │                            │  关键：若 tool 是 BASH_TOOL_NAME 且
    │                            │  input 包含 command 字段 →
    │                            │  启动 startSpeculativeClassifierCheck()
    │                            │  （后台预跑 allow 分类器，与后续流程并行）
    └─────────────┬──────────────┘
                  │
                  ▼
    ┌────────────────────────────┐
    │ checkPermissionsAndCallTool│  toolExecution.ts
    │   → canUseTool()           │  useCanUseTool.tsx · 进入权限核心
    └─────────────┬──────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                          STAGE 2: 权限决策阶段                               │
└─────────────────────────────────────────────────────────────────────────────┘

                  │
                  ▼
    ┌────────────────────────────┐
    │ hasPermissionsToUseTool()  │  permissions.ts · 决策入口
    │                            │
    │   ┌─────────────────────┐  │
    │   │ 决策优先级（由高到低）│  │
    │   │ 1. 全局规则匹配      │  │  ← 如 /permissions 中设置的规则
    │   │ 2. Tool 级规则匹配   │  │  ← 工具特定规则
    │   │ 3. Bypass 模式       │  │  ← 如 auto-mode / yolo
    │   │ 4. 静态规则匹配      │  │  ← 代码内置规则
    │   │ 5. 分类器判断        │  │  ← LLM-based prompt rule 匹配
    │   │ 6. 默认 ask          │  │  ← 无匹配 → 弹窗
    │   └─────────────────────┘  │
    └─────────────┬──────────────┘
                  │
                  ▼
    ┌────────────────────────────┐
    │ bashToolHasPermission()    │  Bash 工具专用逻辑
    │   (bashPermissions.ts)     │
    │                            │
    │   ┌──────────────────────┐ │
    │   │ 并行启动三个分类器    │ │
    │   │                      │ │
    │   │ ① deny classifier    │ │  ← 匹配 deny prompt rules
    │   │    classifyBashCommand│ │     "git log and status commands" 等
    │   │                      │ │     若匹配 → 返回 deny
    │   │ ② ask classifier     │ │  ← 匹配 ask prompt rules
    │   │    classifyBashCommand│ │     若匹配 → 返回 ask
    │   │                      │ │     同时生成 pendingClassifierCheck
    │   │ ③ allow classifier   │ │  ← 匹配 allow prompt rules
    │   │    classifyBashCommand│ │     若匹配 → 返回 allow
    │   │                      │ │
    │   │ 优先级: deny > ask > allow │
    │   │ （先检查 deny，再 ask，最后 allow）│
    │   └──────────────────────┘ │
    └─────────────┬──────────────┘
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
     ┌─────┐   ┌─────┐   ┌─────┐
     │allow│   │deny │   │ ask │
     └──┬──┘   └──┬──┘   └──┬──┘
        │         │         │
        ▼         ▼         ▼

┌─────────────────────────────────────────────────────────────────────────────┐
│                       STAGE 3: 分类器内部机制                               │
└─────────────────────────────────────────────────────────────────────────────┘

    [allow / deny / ask 三个分类器结构相同，只是规则源不同]

    classifyBashCommand(command, cwd, descriptions, behavior, signal)
           │
           ▼
    ┌────────────────────────────┐
    │ 1. 获取 prompt rules       │  从配置/权限上下文中读取
    │    getBashPrompt*Desc...   │  如: ["git log and status commands",
    │                            │       "safe read-only operations"]
    └─────────────┬──────────────┘
                  │
                  ▼
    ┌────────────────────────────┐
    │ 2. 构造 LLM 请求           │  使用 sideQuery（轻量 API）
    │                            │  传入：
    │    system prompt:          │    - 当前命令 (command)
    │    "你是命令分类器..."      │    - 工作目录 (cwd)
    │                            │    - 规则描述列表 (descriptions)
    │    user prompt:            │    - 行为类型 (behavior)
    │    command: {cmd}          │    - abort signal
    │    cwd: {cwd}              │
    │    rules: [...]            │
    │    请判断命令是否匹配规则   │
    └─────────────┬──────────────┘
                  │
                  ▼
    ┌────────────────────────────┐
    │ 3. LLM 返回 XML 格式       │
    │    <result>                │
    │      <matches>true/false</matches>
    │      <matched_rule>...</matched_rule>
    │      <confidence>high/medium/low</confidence>
    │      <reason>...</reason>
    │    </result>               │
    └─────────────┬──────────────┘
                  │
                  ▼
    ┌────────────────────────────┐
    │ 4. 解析为 ClassifierResult │
    │    { matches, confidence,  │
    │      matchedDescription,   │
    │      reason }              │
    └────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                       STAGE 4: 交互处理阶段                                  │
└─────────────────────────────────────────────────────────────────────────────┘

    ┌─────────────────────────────────────────────────────────┐
    │ 分支 A: allow（直接放行）                                │
    │                                                         │
    │  hasPermissionsToUseTool 返回 behavior='allow'          │
    │       │                                                 │
    │       ▼                                                 │
    │  useCanUseTool.tsx:113  resolve(buildAllow(...))       │
    │       │                                                 │
    │       ▼                                                 │
    │  工具立即执行，无弹窗                                     │
    └─────────────────────────────────────────────────────────┘

    ┌─────────────────────────────────────────────────────────┐
    │ 分支 B: deny（直接拒绝）                                 │
    │                                                         │
    │  hasPermissionsToUseTool 返回 behavior='deny'           │
    │       │                                                 │
    │       ▼                                                 │
    │  useCanUseTool.tsx:149  resolve(deny result)           │
    │       │                                                 │
    │       ▼                                                 │
    │  若 auto-mode 且被 classifier deny → 记录并通知用户       │
    │  工具不执行                                               │
    └─────────────────────────────────────────────────────────┘

    ┌─────────────────────────────────────────────────────────┐
    │ 分支 C: ask（弹窗请示）← 最复杂路径                      │
    │                                                         │
    │  hasPermissionsToUseTool 返回 behavior='ask'            │
    │  + pendingClassifierCheck（allow 分类器的元数据）        │
    │       │                                                 │
    │       ▼                                                 │
    │  ┌────────────────────────────────────────────┐         │
    │  │ C1: Coordinator Worker 模式               │         │
    │  │    awaitAutomatedChecksBeforeDialog=true   │         │
    │  │    → handleCoordinatorPermission()         │         │
    │  │      (coordinatorHandler.ts)               │         │
    │  │    → 等待自动化检查，不立即弹窗              │         │
    │  └────────────────────────────────────────────┘         │
    │       │ 若未解决，继续...                                │
    │       ▼                                                 │
    │  ┌────────────────────────────────────────────┐         │
    │  │ C2: Swarm Worker 模式                     │         │
    │  │    → handleSwarmWorkerPermission()         │         │
    │  │      (swarmWorkerHandler.ts)               │         │
    │  │    → 转发到 leader 或通过 classifier        │         │
    │  └────────────────────────────────────────────┘         │
    │       │ 若未解决，继续...                                │
    │       ▼                                                 │
    │  ┌────────────────────────────────────────────┐         │
    │  │ C3: Main Agent - 2秒 Grace Period          │         │
    │  │    (useCanUseTool.tsx:242-304)             │         │
    │  │                                            │         │
    │  │    peekSpeculativeClassifierCheck()        │         │
    │  │         │                                  │         │
    │  │         ├─→ 返回结果（<2秒）                │         │
    │  │         │       │                          │         │
    │  │         │       ▼                          │         │
    │  │         │   matches=true && confidence=high│         │
    │  │         │       │                          │         │
    │  │         │       ▼                          │         │
    │  │         │   consumeSpeculativeClassifierCheck()      │
    │  │         │   setClassifierApproval()        │         │
    │  │         │   resolve(buildAllow(...))       │         │
    │  │         │   → 跳过弹窗，直接执行！           │         │
    │  │         │                                  │         │
    │  │         └─→ 超时(2s) 或无匹配                │         │
    │  │                 │                          │         │
    │  │                 ▼                          │         │
    │  │         继续到 C4...                        │         │
    │  └────────────────────────────────────────────┘         │
    │       │                                                 │
    │       ▼                                                 │
    │  ┌────────────────────────────────────────────┐         │
    │  │ C4: 显示 Permission Dialog                 │         │
    │  │    handleInteractivePermission()           │         │
    │  │    (interactiveHandler.ts)                 │         │
    │  │                                            │         │
    │  │    ┌────────────────────────────────────┐  │         │
    │  │    │ 同时后台启动 executeAsyncClassifierCheck()  │  │         │
    │  │    │ （若 pendingClassifierCheck 存在）  │  │         │
    │  │    │ 在弹窗显示期间，LLM 在后台跑 allow   │  │         │
    │  │    │ 分类器，若匹配则自动批准              │  │         │
    │  │    └────────────────────────────────────┘  │         │
    │  │                                            │         │
    │  │    用户看到弹窗: "Allow this command?"     │         │
    │  │         ├─→ 用户点击 Allow → 执行          │         │
    │  │         ├─→ 用户点击 Deny  → 拒绝          │         │
    │  │         └─→ 后台 classifier 匹配 → 自动允许 │         │
    │  └────────────────────────────────────────────┘         │
    └─────────────────────────────────────────────────────────┘
```

---

## 三、关键设计问题：为什么只有 allow 分类器是 speculative 的？

### 3.1 三个分类器的不同角色


| 分类器       | 作用          | 决策权重     | 是否 Speculative |
| --------- | ----------- | -------- | -------------- |
| **deny**  | 识别危险命令，直接拒绝 | 最高（先检查）  | 否              |
| **ask**   | 识别需要人工判断的命令 | 中（其次检查）  | 否              |
| **allow** | 识别安全命令，自动放行 | 最低（最后检查） | **是**          |


### 3.2 为什么 deny/ask 不需要 speculative？

```
决策顺序: deny → ask → allow

- 若 deny 匹配 → 直接拒绝，流程结束
- 若 ask 匹配 → 直接 ask，流程结束（pendingClassifierCheck 会携带 allow 的元数据供后续使用）
- 若 allow 匹配 → 需要前两个都不匹配才会走到这里

所以 allow 是 "rescue" 路径：只有当前面都未命中时才有机会生效。
```

### 3.3 Speculative 的本质：前端加速

```
非 speculative 流程（最坏情况）:
  API 返回 tool_use
       → 解析 input
       → hasPermissionsToUseTool（串行跑 deny→ask→allow）
       → 全部未命中 → behavior='ask'
       → 显示弹窗
       → 用户犹豫/思考
       → 用户点击 Allow
       → 工具执行
       总延迟: 分类器时间 + 弹窗时间 + 用户反应时间

Speculative 流程（优化后）:
  API 返回 tool_use
       → 解析 input
       → 【并行】启动 allow 分类器（后台跑）
       → hasPermissionsToUseTool（deny→ask 都未命中）
       → behavior='ask'
       → 【2秒 grace period】等待 speculative 结果
       → allow 分类器返回 matches=true, confidence=high
       → 直接放行，不显示弹窗！
       总延迟: max(分类器时间, 2秒) 但通常无需弹窗
```

### 3.4 为什么只 speculative allow？

1. **deny 是前置判断** — 如果命令应该被 deny，speculative allow 的结果无用（会被覆盖）
2. **ask 生成 pending check** — ask 分支会在结果中附带 `pendingClassifierCheck`，其中已经包含了 allow 分类器的输入参数。这意味着即使 ask 弹窗显示后，后台仍可以启动 allow 分类器（通过 `executeAsyncClassifierCheck`）
3. **allow 是高频路径** — 大多数日常开发命令（git status, ls, cat 等）都属于 allow 规则，speculative 能最大程度减少弹窗干扰
4. **资源限制** — 同时 speculative 三个 LLM 调用太昂贵；deny 和 ask 的命中率较低，不值得预跑

---

## 四、关键数据结构

### 4.1 PendingClassifierCheck

```typescript
// 附在 ask 决策结果上，供后续后台分类使用
{
  command: string,      // 要分类的命令
  cwd: string,          // 当前工作目录
  descriptions: string[] // allow 规则描述列表
}
```

### 4.2 ClassifierResult

```typescript
{
  matches: boolean,           // 是否匹配规则
  matchedDescription?: string, // 匹配到的规则描述
  confidence: 'high' | 'medium' | 'low',
  reason: string              // 分类理由
}
```

### 4.3 输入 LLM 的信息

```
sideQuery 调用时传入的上下文：
- command: 完整的 bash 命令字符串
- cwd: 执行命令的当前目录
- descriptions: prompt rule 列表（如 ["git log and status commands", "safe read-only operations"]）
- behavior: 'deny' | 'ask' | 'allow'（告诉 LLM 当前检查的方向）
- 系统 prompt: 定义分类器的角色和行为
```

---

## 五、时序图：Main Agent 的典型路径

```
API Stream          StreamingToolExecutor      useCanUseTool           BashPermissions         LLM (sideQuery)
(query.ts)          (StreamingToolExecutor.ts) (useCanUseTool.tsx)     (bashPermissions.ts)    (sideQuery)
   │                        │                        │                      │                      │
   │──tool_use block───────►│                        │                      │                      │
   │                        │──addTool()────────────►│                      │                      │
   │                        │                        │                      │                      │
   │                        │                        │──canUseTool()───────►│                      │
   │                        │                        │                      │                      │
   │                        │  【并行启动 speculative allow classifier】    │                      │
   │                        │◄─startSpeculativeClassifierCheck()────────────┤                      │
   │                        │                        │                      │──classifyBashCommand─┤
   │                        │                        │                      │  (allow, 后台)       │
   │                        │                        │                      │◄─────────────────────│
   │                        │                        │                      │  (异步执行中...)      │
   │                        │                        │                      │                      │
   │                        │                        │◄─hasPermissions...───│                      │
   │                        │                        │  (deny→ask→allow)    │                      │
   │                        │                        │  deny: 未匹配          │                      │
   │                        │                        │  ask:  未匹配          │                      │
   │                        │                        │  allow: 需要检查       │                      │
   │                        │                        │  → behavior='ask'      │                      │
   │                        │                        │                      │                      │
   │                        │                        │──peekSpeculative...─►│                      │
   │                        │                        │  检查后台 allow 结果   │                      │
   │                        │                        │◄─结果未就绪───────────│                      │
   │                        │                        │                      │                      │
   │                        │                        │ 【等待 2 秒 grace period】                   │
   │                        │                        │                      │                      │
   │                        │                        │──peekSpeculative...─►│                      │
   │                        │                        │  再次检查             │                      │
   │                        │                        │◄─matches=true, high──│◄──结果返回───────────│
   │                        │                        │                      │                      │
   │                        │                        │──consumeSpeculative...►                     │
   │                        │                        │──setClassifierApproval()                    │
   │                        │                        │──resolve(buildAllow())                      │
   │                        │◄─Allow Decision────────│                      │                      │
   │                        │                        │                      │                      │
   │                        │──执行工具命令──────────►│                      │                      │
   │                        │                        │                      │                      │
```

---

## 六、相关代码位置速查


| 功能             | 文件                                                        | 关键函数/行                                        |
| -------------- | --------------------------------------------------------- | --------------------------------------------- |
| 流式触发           | `src/query.ts`                                            | L830: `streamingToolExecutor.addTool()`       |
| Speculative 启动 | `src/services/tools/toolExecution.ts`                     | L747-765: `startSpeculativeClassifierCheck()` |
| 并发执行器          | `src/services/tools/StreamingToolExecutor.ts`             | `executeTool()`, `runToolUse()`               |
| 权限主入口          | `src/hooks/useCanUseTool.tsx`                             | `CanUseToolFn`, L61-352                       |
| Grace Period   | `src/hooks/useCanUseTool.tsx`                             | L242-304: 2秒等待逻辑                              |
| Bash 权限决策      | `src/tools/BashTool/bashPermissions.ts`                   | `bashToolHasPermission()`                     |
| 三分类器并行         | `src/tools/BashTool/bashPermissions.ts`                   | L1878-1896: deny/ask/allow 调用                 |
| Speculative 存储 | `src/tools/BashTool/bashPermissions.ts`                   | `speculativeClassifierChecks` Map             |
| 通用权限决策         | `src/utils/permissions/permissions.ts`                    | `hasPermissionsToUseToolInner()`              |
| 交互式处理          | `src/hooks/toolPermission/handlers/interactiveHandler.ts` | `handleInteractivePermission()`               |
| 异步分类检查         | `src/hooks/toolPermission/handlers/interactiveHandler.ts` | `executeAsyncClassifierCheck()`               |
| 分类器 stub       | `src/utils/permissions/bashClassifier.ts`                 | `classifyBashCommand()` 签名                    |
| Auto mode      | `src/utils/permissions/yoloClassifier.ts`                 | 2-stage XML classifier                        |


---

## 七、术语表


| 术语                         | 含义                                          |
| -------------------------- | ------------------------------------------- |
| **Speculative Check**      | 在正式需要结果前，提前在后台启动的分类器调用                      |
| **Grace Period**           | 弹窗显示前的等待窗口（2秒），给 speculative 结果机会           |
| **PendingClassifierCheck** | 附在 ask 决策上的元数据，描述如何后台运行 allow 分类器           |
| **Prompt Rule**            | 自然语言描述的安全规则，如 "git log and status commands" |
| **sideQuery**              | 轻量级 Anthropic API 调用，通常使用 Haiku 模型做分类       |
| **Behavior**               | 权限决策结果类型：`allow` / `deny` / `ask`           |
| **YOLO / Auto Mode**       | 自动批准模式，使用分类器自动判断而非弹窗                        |
| **Coordinator Worker**     | 协调器工作线程，等待自动化检查完成后再决定是否弹窗                   |
| **Swarm Worker**           | 集群工作线程，通过 mailbox 向 leader 转发权限请求           |


## 八、全工具权限管理流程图

> 本图覆盖所有工具的权限决策流程，以 query.ts → StreamingToolExecutor → toolExecution.ts 调用链风格绘制。BashTool 只是其中一条分支。

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  query.ts: 模型流式返回 tool_use 块                                                          │
│  streamingToolExecutor.addTool(toolBlock, assistantMessage)                                  │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                           │
                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  StreamingToolExecutor.ts                                                                    │
│  ├── addTool() → processQueue() → executeTool()                                              │
│  └── runToolUse() → checkPermissionsAndCallTool()                                            │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                           │
                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  toolExecution.ts: checkPermissionsAndCallTool()                                             │
│  ★ Zod 输入验证 (tool.inputSchema.safeParse) — 每个工具有独立 schema                          │
│  ★ 工具级输入验证 (tool.validateInput?) — 可选，工具可自定义验证逻辑                           │
│  ★ 【BashTool 专属】投机启动 allow 分类器 (bashPermissions.ts) — 仅 BashTool 触发            │
│  │      startSpeculativeClassifierCheck() → classifyBashCommand(..., 'allow')               │
│  ├── 运行 PreToolUse Hooks (runPreToolUseHooks)                                             │
│  └── 调用 canUseTool() → hasPermissionsToUseTool()                                           │
│      (useCanUseTool.tsx → permissions.ts)                                                    │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                           │
                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│  permissions.ts: hasPermissionsToUseToolInner()                                              │
│                                                                                              │
│  Phase 1 — Bypass-Immune（不可绕过，即使 bypassPermissions 也生效）                          │
│  ├── 1a. 全局 Deny 规则 (getDenyRuleForTool)                                                │
│  │      alwaysDenyRules 中匹配工具名 → deny                                                  │
│  ├── 1b. 全局 Ask 规则 (getAskRuleForTool)                                                  │
│  │      alwaysAskRules 中匹配工具名 → ask（Bash sandbox 可 auto-allow）                      │
│  ★ 1c. tool.checkPermissions(parsedInput, context) — 22 工具有自定义实现，62 工具默认 allow   │
│  │      ┌───────────────────────────────────────────────────────────────────────────────┐   │
│  │      │ 各工具 checkPermissions 分支（22 个工具有自定义实现）                              │   │
│  │      │                                                                                │   │
│  │      │ Shell 执行工具（最复杂）:                                                        │   │
│  │      │   BashTool        → bashToolHasPermission()        (bashPermissions.ts)        │   │
│  │      │   PowerShellTool  → powershellToolHasPermission()    (powershellPermissions.ts)│   │
│  │      │                                                                                │   │
│  │      │ 文件写入工具:                                                                    │   │
│  │      │   FileEditTool/FileWriteTool/NotebookEditTool                                     │   │
│  │      │   → checkWritePermissionForTool()                   (filesystem.ts)            │   │
│  │      │   [ deny规则 → 可编辑内部路径 → .claude规则 → 安全检查 → ask规则 →              │   │
│  │      │     acceptEdits模式 → allow规则 → 默认ask（带建议）]                             │   │
│  │      │                                                                                │   │
│  │      │ 文件读取工具:                                                                    │   │
│  │      │   FileReadTool/GlobTool/GrepTool/LSPTool                                          │   │
│  │      │   → checkReadPermissionForTool()                    (filesystem.ts)            │   │
│  │      │   [ 可疑路径拦截 → deny规则 → ask规则 → 写入权限覆盖 → 工作目录 →               │   │
│  │      │     内部路径 → allow规则 → 默认ask ]                                             │   │
│  │      │                                                                                │   │
│  │      │ 网络工具:                                                                        │   │
│  │      │   WebFetchTool  → host allowlist 匹配 → allow / 否则 ask                        │   │
│  │      │   WebSearchTool → passthrough → 外层转为 ask                                     │   │
│  │      │   MCPTool       → passthrough → 外层单独处理                                      │   │
│  │      │                                                                                │   │
│  │      │ Agent/控制流工具:                                                                │   │
│  │      │   AgentTool         → passthrough(auto-mode) / allow                             │   │
│  │      │   ExitPlanModeV2Tool→ teammates allow / non-teammates ask                       │   │
│  │      │   SendMessageTool   → bridge ask / others allow                                  │   │
│  │      │                                                                                │   │
│  │      │ 简单/始终允许工具:                                                                │   │
│  │      │   McpAuthTool/TodoWriteTool/SyntheticOutputTool → always allow                   │   │
│  │      │   ConfigTool        → read allow / write ask                                     │   │
│  │      │                                                                                │   │
│  │      │ 始终弹窗/测试工具:                                                                │   │
│  │      │   AskUserQuestionTool/TestingPermissionTool → always ask                         │   │
│  │      │                                                                                │   │
│  │      │ Skill 工具:                                                                      │   │
│  │      │   SkillTool → skill registry lookup → 按 skill 定义决定                          │   │
│  │      │                                                                                │   │
│  │      │ 默认（无 custom checkPermissions）:                                              │   │
│  │      │   62 个工具 → 默认 behavior='allow'（TOOL_DEFAULTS）                             │   │
│  │      └───────────────────────────────────────────────────────────────────────────────┘   │
│  ├── 1d. checkPermissions 返回 deny → 直接 deny                                              │
│  ★ 1e. requiresUserInteraction() 返回 true → ask（工具自身声明是否需要用户交互）              │
│  ├── 1f. 内容级 ask 规则（checkPermissions 返回 ruleBehavior='ask'）→ ask                   │
│  └── 1g. 安全检查 (safetyCheck) → ask（如 .git/、.claude/、shell 配置等）                    │
│                                                                                              │
│  Phase 2 — Bypass & Allow（可被绕过）                                                        │
│  ├── 2a. bypassPermissions 模式 / plan 模式（且可用 bypass）→ allow                          │
│  └── 2b. 全局 Allow 规则 (toolAlwaysAllowedRule) → allow                                   │
│                                                                                              │
│  Phase 3 — Fallback                                                                         │
│  └── 3. passthrough → 转为 ask                                                               │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                           │
                                           ▼
                           ┌───────────────┼───────────────┐
                           │               │               │
                           ▼               ▼               ▼
                      behavior=allow   behavior=deny   behavior=ask
                           │               │               │
                           ▼               ▼               │
                    ┌─────────────┐  ┌─────────────┐      │
                    │ 后处理层     │  │ 直接         │      │
                    │ (wrapper)   │  │ resolve deny │      │
                    │             │  │ → 返回拒绝   │      │
                    │ • 连续拒绝计 │  │   结果       │      │
                    │   数重置     │  │              │      │
                    │   (auto模式) │  │              │      │
                    │             │  │              │      │
                    │ → resolve   │  │              │      │
                    │   allow     │  │              │      │
                    └──────┬──────┘  └──────┬──────┘      │
                           │                │             │
                           ▼                ▼             │
                     resolve allow    resolve deny        │
                                                          │
                                                          │
                                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│ 【第一层】mode-based 后处理  (hasPermissionsToUseTool, permissions.ts:473-955)                      │
│                                                                                             │
│ • dontAsk 模式 → 直接 deny                                                                      │
│   (不可绕过，即使 bypassPermissions 也生效后的 ask 仍会被拦截)                                               │
│                                                                                             │
│ • auto 模式 → 多层判断:                                                                           │
│   1) safetyCheck 非 classifier-approvable + headless → deny                                  │
│   2) requiresUserInteraction() → ask (保持弹窗)                                                 │
│   3) PowerShell + !POWERSHELL_AUTO_MODE → ask/deny                                          │
│   4) acceptEdits fast-path → 再跑 checkPermissions，allow 则放行                                  │
│   5) safe-tool allowlist → allow (跳过 classifier)                                            │
│   6) YOLO classifier (classifyYoloAction) → allow/deny/ask                                  │
│   7) denial limit 超限 → ask (或 headless abort)                                               │
│                                                                                             │
│ • headless/async agents (shouldAvoidPermissionPrompts):                                     │
│   PermissionRequest hooks → 有决策则返回 / 无决策则 auto-deny                                         │
│                                                                                             │
│ 第一层处理完后:                                                                                    │
│   allow → 直接 resolve allow    deny → 直接 resolve deny                                        │
│   仍 ask → 进入第二层 ↓                                                                           │
│                                                                                             │
│ 【第二层】场景分派  (useCanUseTool case 'ask', useCanUseTool.tsx:189-327)                            │
│ （三种场景互斥，顺序判断但只命中一个）                                                                         │
│                                                                                             │
│ 【Coordinator Worker 场景】                                                                     │
│ awaitAutomatedChecksBeforeDialog = true                                                     │
│ → handleCoordinatorPermission()                                                             │
│   顺序等 hooks → ★classifier allow only (Bash) → 批准 resolve / 搞不定 fallthrough                  │
│                                                                                             │
│ 【Swarm Worker 场景】                                                                           │
│ isSwarmWorker() = true                                                                      │
│ → handleSwarmWorkerPermission()                                                             │
│   ★classifier allow only (Bash) → mailbox 请示 leader → leader 批准/拒绝 → resolve                  │
│                                                                                             │
│ 【Main Agent 场景】                                                                             │
│ → ★【2s 宽限期】— 仅 BashTool                                                                     │
│   peekSpeculativeClassifierCheck() → 高置信 allow → resolve allow                              │
│   超时/不匹配 → handleInteractivePermission()                                                    │
│   弹窗队列 + PermissionDialog + 后台 executeAsyncClassifierCheck                                  │
│   用户选择 / 自动批准 → resolve                                                                     │
└───────────────────────────────────────────────────────────────┬─────────────────────────────┘
                                                               │
                              ┌───────────────┴─────────────────────────────┐
                              │                                             │
                              ▼                                             ▼
                       resolve allow                                   resolve deny
┌───────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│  权限 resolve 后，checkPermissionsAndCallTool() 继续:                                                               │
│                                                                                                                   │
│  if (permissionDecision.behavior === 'allow') {                                                                  │
│      ├── 恢复模型原始输入 (processedInput/backfilledClone)                                                     │
│      └── ★ 调用 tool.call() — 每个工具的执行逻辑完全不同                                   │
│          → 真正执行（bash / 文件读写 / 网络请求 / Agent 调用 等）                                                  │
│          → 产出 progress 消息（实时回显）                                                                           │
│          → 返回 ToolResult                                                                                       │
│      ├── endToolExecutionSpan()                                                                             │
│      └── 产出 tool_result 消息                                                                                   │
│  } else {                                                                                                        │
│      → 产出拒绝/错误的 tool_result 消息                                                                            │
│      → 执行 PermissionDenied hooks（如有）                                                                        │
│  }                                                                                                               │
└───────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

> **★ 图例**：流程图中标 ★ 的步骤为**工具可自定义/个性化**的钩子或属性，其余为通用框架逻辑。

### ★ 工具个性化过程清单


| 个性化过程                            | 所在接口/位置                 | 影响范围                              | 默认值                 |
| -------------------------------- | ----------------------- | --------------------------------- | ------------------- |
| **tool.inputSchema**             | `Tool` 接口定义             | 所有工具                              | 每个工具独立 Zod schema   |
| **tool.validateInput**           | `Tool` 接口（可选）           | 有自定义验证的工具                         | `undefined`（跳过）     |
| **投机启动 allow 分类器**               | `toolExecution.ts:747`  | 仅 BashTool（按 `tool.name` 判断）      | 其他工具不触发             |
| **2s 宽限期（grace period）**         | `useCanUseTool.tsx:242` | 仅 BashTool（peek 投机结果，2s 内匹配则跳过弹窗） | 其他工具不触发             |
| **tool.checkPermissions**        | `Tool` 接口               | 22 工具有自定义，62 工具走默认                | `behavior: 'allow'` |
| **tool.requiresUserInteraction** | `Tool` 接口（可选）           | AskUserQuestionTool 等             | `false`             |
| **tool.call**                    | `Tool` 接口               | 所有工具                              | 每个工具完全独立实现          |
| **tool.description**             | `Tool` 接口               | 所有工具                              | 每个工具独立生成弹窗描述        |
| **PermissionDenied hooks**       | `toolExecution.ts` 尾部   | 有注册的工具                            | 无                   |


### 关键节点说明


| 阶段                 | 文件                          | 作用                                                                               |
| ------------------ | --------------------------- | -------------------------------------------------------------------------------- |
| **投机启动**           | `toolExecution.ts:747-765`  | BashTool 专属：在 pre-tool hooks 之前提前后台启动 allow 分类器                                  |
| **通用决策引擎**         | `permissions.ts:1158-1319`  | `hasPermissionsToUseToolInner()` — 跨工具的 Phase 1/2/3 决策层次                         |
| **Bash 权限**        | `bashPermissions.ts`        | `bashToolHasPermission()` — AST 解析、静态规则、prompt 分类器、路径安全、子命令拆分                    |
| **PowerShell 权限**  | `powershellPermissions.ts`  | `powershellToolHasPermission()` — 类似 Bash 但 PowerShell 语义                        |
| **文件写入权限**         | `filesystem.ts:1205`        | `checkWritePermissionForTool()` — deny规则 → 内部路径 → 安全检查 → acceptEdits → allow规则   |
| **文件读取权限**         | `filesystem.ts:1030`        | `checkReadPermissionForTool()` — deny规则 → ask规则 → 写入权限覆盖 → 工作目录 → 内部路径 → allow规则 |
| **WebFetch 权限**    | `WebFetchTool`              | host allowlist 匹配即 allow，否则 ask                                                  |
| **Coordinator 审批** | `coordinatorHandler.ts`     | 协调器工作线程：等待自动化检查完成后再决定是否弹窗                                                        |
| **Swarm 转发**       | `swarmWorkerHandler.ts`     | 集群工作线程：转发权限请求到 leader 或通过 classifier                                             |
| **交互弹窗**           | `interactiveHandler.ts`     | REPL 中显示 PermissionDialog，支持后台分类器自动批准                                            |
| **2s 宽限期**         | `useCanUseTool.tsx:242-304` | 弹窗前等待投机分类器结果，若匹配则跳过弹窗直接 allow                                                    |


### 权限决策优先级（从高到低，全工具通用）

1. **Abort** → 取消（用户手动中断）
2. **全局 Deny 规则** (`getDenyRuleForTool`) → 直接拒绝（不可绕过）
3. **全局 Ask 规则** (`getAskRuleForTool`) → 直接弹窗（Bash sandbox 可 auto-allow）
4. **工具级 Deny** (`tool.checkPermissions` 返回 deny) → 直接拒绝（如 Bash exact-match deny、文件 deny 规则）
5. **Prompt-based Deny 分类器** → 高置信匹配则拒绝（Bash/PowerShell 专属）
6. **Prompt-based Ask 分类器** → 高置信匹配则弹窗（Bash/PowerShell 专属）
7. **安全检查** (safetyCheck) → 弹窗（如 `.git/`、`.claude/`、shell 配置、可疑路径；不可绕过）
8. **Bypass/Plan 模式** → 允许（跳过大部分检查，但 safetyCheck 仍生效）
9. **全局 Allow 规则** (`toolAlwaysAllowedRule`) → 允许
10. **静态规则**（prefix/exact match） → 允许/弹窗（Bash 子命令级别）
11. **Allow 分类器** → 弹窗后后台尝试自动批准（Bash/PowerShell 专属）
12. **默认 passthrough** → 转为 ask（如 WebSearch、MCP、AgentTool auto-mode）

## 九、Auto Mode 自动批准模式

> 本节是**第八节全工具流程图**中 **【第一层】mode-based 后处理 → `auto 模式` 分支**的展开详解。
>
> 在第八节中，第一层 `hasPermissionsToUseToolInner()` 返回 `ask` 后，会进入 mode-based 后处理层。其中 `auto` 模式（以及 `plan` 模式下 `isAutoModeActive()` 为 true 时）不会直接弹窗，而是通过以下机制尝试自动批准或拒绝。

### 在第八节流程图中的位置

```
【第一层】mode-based 后处理 (hasPermissionsToUseTool, permissions.ts:473-955)
  │
  ├── dontAsk 模式 → deny
  │
  ├── auto 模式 → 【本节展开】←══════════════╗
  │   1) safetyCheck 免疫 → deny/ask           ║
  │   2) requiresUserInteraction → ask         ║
  │   3) PowerShell 特殊处理 → ask/deny        ║
  │   4) acceptEdits fast-path → allow         ║
  │   5) safe-tool allowlist → allow           ║
  │   6) YOLO classifier → allow/deny/ask  ←═══╝
  │   7) denial limit 超限 → ask/abort
  │
  └── headless → PermissionRequest hooks → auto-deny
```

### 1. 触发条件

`permissions.ts:518-927` (`hasPermissionsToUseTool`) 中，第一层 `hasPermissionsToUseToolInner()` 返回 `ask` 后：

```
if (mode === 'auto') → 进入 auto 模式处理
if (mode === 'plan' && isAutoModeActive()) → 同样进入 auto 模式处理
```

### 2. Auto Mode 决策链（含 6 个快速通道）

以下按**优先级顺序**执行，任一通道命中即返回，不再后续：

```
第一层返回 ask
      │
      ▼
┌─────────────────────────────────────────────────────────────┐
│ ① safetyCheck 且 classifierApprovable=false                 │
│    → 免疫：headless 则 deny，否则保持 ask                     │
│    （如 .git/、.claude/、shell 配置等敏感路径不可被 auto 覆盖）   │
└─────────────────────────────────────────────────────────────┘
      │ 未命中
      ▼
┌─────────────────────────────────────────────────────────────┐
│ ② tool.requiresUserInteraction?.()                          │
│    → 保持 ask（交互型工具不能自动批准）                        │
└─────────────────────────────────────────────────────────────┘
      │ 未命中
      ▼
┌─────────────────────────────────────────────────────────────┐
│ ③ PowerShell 工具且 POWERSHELL_AUTO_MODE 未开启              │
│    → headless 则 deny，否则跳过 classifier、保持 ask          │
└─────────────────────────────────────────────────────────────┘
      │ 未命中
      ▼
┌─────────────────────────────────────────────────────────────┐
│ ④ acceptEdits 快速通道（非 Agent/REPL 工具）                  │
│    → 临时把 mode 改为 acceptEdits，重新跑 tool.checkPermissions()│
│    → 若返回 allow → 直接批准，跳过 classifier（省一次 API 调用）│
│    例：CWD 内的文件编辑、安全路径的读写等                      │
└─────────────────────────────────────────────────────────────┘
      │ 未命中
      ▼
┌─────────────────────────────────────────────────────────────┐
│ ⑤ Safe-tool allowlist                                       │
│    → 工具在 allowlist 上 → 直接 allow，跳过 classifier        │
└─────────────────────────────────────────────────────────────┘
      │ 未命中
      ▼
┌─────────────────────────────────────────────────────────────┐
│ ⑥ YOLO Classifier (classifyYoloAction)                      │
│    → 基于完整对话历史判断动作安全性                           │
│    → 返回 allow / deny / unavailable                        │
└─────────────────────────────────────────────────────────────┘
```

### 3. YOLO Classifier 上下文构成

`classifyYoloAction()` (`yoloClassifier.ts:1012`) 的输入上下文由四部分构成：

#### 3.1 System Prompt（四层拼装）

```
Layer 1: Base Prompt (auto_mode_system_prompt.txt)
  • 角色：安全分类器
  • 流程：理解动作 → 检查用户意图 → 评估风险
  • 决策类别：
    - BLOCK ALWAYS: 外部代码执行、不可逆本地破坏、未授权持久化、
                    安全削弱、权限提升、网络服务
    - BLOCK unless intent: CWD 外写文件、系统级包管理、git push
    - ALLOW: 读文件、搜索、CWD 内编辑、测试构建、本地 git 操作
  • 关键约束：只考虑用户消息，不考虑 assistant 文本

Layer 2: Permissions Template
  External 构建: permissions_external.txt
  ANT 构建: permissions_anthropic.txt（更详细，含云 CLI/IaC）
  三段式：Allow Rules / Deny Rules / Environment Context

Layer 3: 用户自定义规则 (settings.autoMode)
  • allow[]      → 替换 <user_allow_rules_to_replace>
  • soft_deny[]  → 替换 <user_deny_rules_to_replace>
  • environment[] → 替换 <user_environment_to_replace>
  External: 用户规则替换默认值
  Anthropic: 用户规则追加到默认值

Layer 4: 动态注入（ANT-ONLY）
  • BASH_CLASSIFIER 开启 → 注入 Bash(prompt:) allow/deny 描述
  • POWERSHELL_AUTO_MODE → 注入 PS 专用 deny guidance
```

#### 3.2 Messages / Transcript

从历史 `messages[]` 提取：

| 来源 | 保留内容 | 排除内容 |
|------|---------|---------|
| User 消息 | 文本内容 | 图片/附件 |
| Assistant 消息 | 只保留 `tool_use` 块 | **排除 assistant 文本**（防操控）|
| Queued Command 附件 | prompt 文本作为 user turn | — |

序列化方式（`toCompactBlock`）：

| 内容 | 默认格式 | JSONL 格式（可选）|
|------|---------|-----------------|
| User 文本 | `User: text\n` | `{"user":"text"}\n` |
| Bash | `Bash ls\n` | `{"Bash":"ls"}\n` |
| FileEdit | `FileEdit /path: new_string\n` | `{"FileEdit":"/path: new_string"}\n` |
| Agent | `Agent (Explore): prompt\n` | `{"Agent":"(Explore): prompt"}\n` |
| 无安全相关性 | `''`（省略） | `''`（省略）|

**toAutoClassifierInput** 是各工具自定义的投影：

| 工具 | 投影输出 |
|------|---------|
| BashTool | `input.command` |
| FileEditTool | `${file_path}: ${new_string}` |
| FileWriteTool | `${file_path}: ${content}` |
| AgentTool | `(subagent_type, mode=xxx): ${prompt}` |
| WebFetchTool | `input.url` |
| SkillTool | `skill` 名称 |
| GlobTool/GrepTool | 查询模式 + 路径 |

#### 3.3 Action（当前待评估动作）

```
formatActionForClassifier(toolName, input)
→ { role: 'assistant', content: [{type:'tool_use', name, input}] }
```

追加在 transcript 末尾，与历史 tool_use 用同一序列化方式。

#### 3.4 CLAUDE.md 注入（可选）

```
{
  role: 'user',
  content: [{
    type: 'text',
    text: "The following is the user's CLAUDE.md configuration...\n"
          + "<user_claude_md>\n${claudeMd}\n</user_claude_md>",
    cache_control: getCacheControl({ querySource: 'auto_mode' })
  }]
}
```

### 4. 两种分类器实现

#### Path A: Tool-use Classifier（传统）

```
模型被强制调用 classify_result 工具
→ 输出 {thinking, shouldBlock, reason}
```

工具 schema：`{thinking: string, shouldBlock: boolean, reason: string}`

#### Path B: 2-Stage XML Classifier（新，GrowthBook 控制）

```
Stage 1 (Fast)
  max_tokens: 64, stop_sequences: ["</block>"]
  suffix: "Err on the side of blocking. <block> immediately."
  │
  ├── <block>no</block> → ALLOW，直接返回（省 Stage 2 API 调用）
  └── <block>yes</block> 或解析失败 → 进入 Stage 2
       │
       ▼
Stage 2 (Thinking)
  max_tokens: 4096
  suffix: 要求详细推理，用 <thinking> 再输出 <block>
  │
  ├── <block>yes</block><reason>...</reason> → BLOCK
  ├── <block>no</block> → ALLOW
  └── 解析失败 → 安全默认 BLOCK
```

**Cache 策略**：action block 带 `cache_control`，Stage 2 共享 Stage 1 的 transcript+action prefix，保证 Stage 2 的 prefix 有 cache hit。

### 5. 结果处理

| 分类器结果 | 处理 |
|-----------|------|
| `shouldBlock: false` | allow，重置 consecutive denials |
| `shouldBlock: true` | deny，记录 denial，检查是否超限 |
| 连续 denial 超限 | fallback 到 ask（让用户手动审核）|
| transcript too long | headless → abort；CLI → fallback 到 ask |
| unavailable (API 错误) | iron_gate=closed → deny；iron_gate=open → fallback 到 ask |
| parse failure | 安全默认 → deny |

### 6. Denial Tracking

`denialTracking.ts` 维护两个计数器：

- **consecutiveDenials**：连续被拒次数，allow 时重置
- **totalDenials**：总会话被拒次数

超过阈值后：
- CLI → fallback 到 ask（弹窗让用户手动审核）
- Headless → 抛出 AbortError，终止 agent

### 7. 和 Bash Prompt Classifier 的核心区别

| | Auto Mode (YOLO) | Bash Prompt Classifier |
|--|------------------|------------------------|
| **范围** | 跨所有工具 | 仅 BashTool |
| **输入** | 完整对话历史 (transcript) + 当前 action | 单条命令字符串 + 规则描述 |
| **判断依据** | "这个动作在当前上下文是否安全" | "这条命令是否匹配某条 prompt 规则" |
| **模型** | 主循环同模型（或独立配置）| sideQuery (Haiku) |
| **层级** | 第一层 ask 之后的后处理层 | BashTool 内部第一层阶段 3 |
| **目的** | 全局安全判断，替代用户弹窗 | 规则语义匹配 |

## 十、BashTool 权限决策详解

> `bashToolHasPermission()` 内部完整流程（`bashPermissions.ts:1663-2554`）。
> 这是全工具流程图中 Phase 1c → BashTool 分支的展开。

```
输入: { command, sandbox?, ... }
  │
  ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│ 0. AST 解析 (parseCommandRaw → parseForSecurityFromAst)                                      │
│                                                                                              │
│  tree-sitter 解析命令字符串:                                                                  │
│  ├── too-complex (命令替换/展开/控制流/解析差异)                                              │
│  │   → checkEarlyExitDeny() → deny 或 ask + pendingClassifierCheck                           │
│  ├── simple (干净解析)                                                                       │
│  │   → checkSemantics() → zsh builtins/eval 等语义危险                                       │
│  │       → 语义不安全: checkSemanticsDeny() → deny 或 ask                                    │
│  └── parse-unavailable (tree-sitter 未加载)                                                  │
│      → tryParseShellCommand() → 解析失败 → ask                                               │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                           │
                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│ 1. Sandbox auto-allow                                                                        │
│                                                                                              │
│  sandbox 启用 && auto-allow 开启 && shouldUseSandbox(input)                                   │
│  → checkSandboxAutoAllow() → allow 或 passthrough → 继续                                      │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                           │
                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│ 2. Exact match                                                                               │
│                                                                                              │
│  bashToolCheckExactMatchPermission()                                                         │
│  ├── exact deny → 返回 deny                                                                  │
│  └── exact allow → 暂存结果，继续 (后续可能覆盖)                                              │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                           │
                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│ 3. Prompt-based classifier (deny + ask)                                                      │
│                                                                                              │
│  isClassifierPermissionsEnabled() && !auto mode                                              │
│  ├── deny classifier: classifyBashCommand(..., 'deny')                                   │
│  │   → 高置信匹配 → 返回 deny                                                                │
│  └── ask classifier: classifyBashCommand(..., 'ask')                                    │
│      → 高置信匹配 → 返回 ask + suggestions + pendingClassifierCheck                          │
│                                                                                              │
│  (注意: 此处没有 allow classifier，allow 在第二层处理)                                        │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                           │
                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│ 4. Command operator check                                                                    │
│                                                                                              │
│  checkCommandOperatorPermissions() — 处理管道、重定向等复合结构                                │
│  ├── 每个管道段递归调用 bashToolHasPermission()                                               │
│  ├── 段结果 deny → deny                                                                      │
│  ├── 段结果 ask → ask + pendingClassifierCheck                                               │
│  └── 段结果 allow → 仍需检查原始命令的路径约束 (防止重定向被绕过)                             │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                           │
                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│ 5. Legacy security check (AST 不可用时)                                                      │
│                                                                                              │
│  astSubcommands === null → bashCommandIsSafeAsync()                                          │
│  → 发现 misparsing 模式 (命令注入风险)                                                        │
│    → check exact match allow → allow 或 ask + pendingClassifierCheck                         │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                           │
                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│ 6. Subcommand split & per-subcommand check                                                   │
│                                                                                              │
│  splitCommand() / astSubcommands — 拆分为子命令                                               │
│  ├── 子命令数 > MAX → ask                                                                    │
│  ├── 多个 cd → ask                                                                           │
│  ├── cd + git → ask (防止 bare repo 攻击)                                                    │
│  └── bashToolCheckPermission() 每个子命令:                                                    │
│      ├── static prefix/exact match allow/deny                                                │
│      ├── checkPathConstraints() — 路径边界、输出重定向                                         │
│      └── readOnlyValidation — 只读命令验证                                                   │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
                                           │
                                           ▼
                           ┌───────────────┼───────────────┐
                           │               │               │
                           ▼               ▼               ▼
                      有 deny         有 ask          全部 allow
                           │               │               │
                           ▼               ▼               ▼
                    ┌──────────┐    ┌──────────┐    ┌──────────┐
                    │ 返回 deny │    │ 返回 ask │    │ 返回 allow│
                    │          │    │ + sugg.  │    │          │
                    │          │    │ + pending│    │          │
                    └──────────┘    └──────────┘    └──────────┘
```

### 关键说明

| 阶段 | 函数 | 作用 | 结果 |
|------|------|------|------|
| **0** | `parseCommandRaw` / `parseForSecurityFromAst` | AST 解析 + 语义检查 | too-complex / simple / unavailable |
| **1** | `checkSandboxAutoAllow` | Sandbox 自动放行 | allow / passthrough |
| **2** | `bashToolCheckExactMatchPermission` | 精确匹配**普通规则**（字符串匹配：`Bash(git status)`, `Bash(git:*)` 等） | deny / allow / passthrough |
| **3** | `classifyBashCommand(..., deny/ask)` | **Prompt 规则**语义匹配（`Bash(prompt: ...)`），ANT-ONLY | deny / ask / null |
| **4** | `checkCommandOperatorPermissions` | 管道/重定向处理 | deny / ask / allow |
| **5** | `bashCommandIsSafeAsync` | Legacy 安全检查 | ask / passthrough |
| **6** | `bashToolCheckPermission` | 子命令级权限 | deny / ask / allow |

### Prompt 分类器 `classifyBashCommand`

```
settings.json 中以 "prompt:" 前缀的规则
        │
        │  例: "Bash(prompt: git log and status commands)"
        ▼
┌───────────────────────────┐
│ classifyBashCommand       │
│ (ANT-ONLY, bashClassifier)│
│                           │
│  输入:                    │
│  • command: 命令字符串    │
│  • cwd: 执行目录          │
│  • descriptions[]:        │
│    prompt规则描述列表     │
│  • behavior: deny/ask/allow│
│                           │
│  sideQuery (Haiku) →      │
│  "这条命令是否匹配规则描述?"│
│                           │
│  输出:                    │
│  • matches: boolean       │
│  • matchedDescription     │
│  • confidence: high/med/low│
│  • reason: string         │
└─────────────┬─────────────┘
              │
    ┌─────────┼─────────┐
    │         │         │
    ▼         ▼         ▼
 behavior=deny   behavior=ask   behavior=allow
 + high match    + high match   + high match
    │              │              │
    ▼              ▼              ▼
   deny          ask           allow
                + pending        │
                classifier       │
                               └──→ 第二层自动批准
```

**第一层只跑 deny + ask**。allow 方向的 `classifyBashCommand` 只在**第二层**运行（coordinator / swarm / main agent 的自动批准逻辑）。

### 关于 allow classifier 的位置

第一层 (`bashToolHasPermission`) **没有 allow classifier**。allow classifier 只在**第二层**（coordinator / swarm / main agent）中运行，目的是在弹窗显示时/之前尝试**自动批准**。

第一层返回 `ask` 时，会附带 `pendingClassifierCheck`（包含 allow classifier 所需的参数），供第二层使用。
