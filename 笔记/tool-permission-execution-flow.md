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
│                        STAGE 3: 分类器内部机制                               │
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

| 分类器 | 作用 | 决策权重 | 是否 Speculative |
|--------|------|----------|-----------------|
| **deny** | 识别危险命令，直接拒绝 | 最高（先检查） | 否 |
| **ask** | 识别需要人工判断的命令 | 中（其次检查） | 否 |
| **allow** | 识别安全命令，自动放行 | 最低（最后检查） | **是** |

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
   │                        │                        │                      │◄─────────────────────┤
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

| 功能 | 文件 | 关键函数/行 |
|------|------|------------|
| 流式触发 | `src/query.ts` | L830: `streamingToolExecutor.addTool()` |
| Speculative 启动 | `src/services/tools/toolExecution.ts` | L747-765: `startSpeculativeClassifierCheck()` |
| 并发执行器 | `src/services/tools/StreamingToolExecutor.ts` | `executeTool()`, `runToolUse()` |
| 权限主入口 | `src/hooks/useCanUseTool.tsx` | `CanUseToolFn`, L61-352 |
| Grace Period | `src/hooks/useCanUseTool.tsx` | L242-304: 2秒等待逻辑 |
| Bash 权限决策 | `src/tools/BashTool/bashPermissions.ts` | `bashToolHasPermission()` |
| 三分类器并行 | `src/tools/BashTool/bashPermissions.ts` | L1878-1896: deny/ask/allow 调用 |
| Speculative 存储 | `src/tools/BashTool/bashPermissions.ts` | `speculativeClassifierChecks` Map |
| 通用权限决策 | `src/utils/permissions/permissions.ts` | `hasPermissionsToUseToolInner()` |
| 交互式处理 | `src/hooks/toolPermission/handlers/interactiveHandler.ts` | `handleInteractivePermission()` |
| 异步分类检查 | `src/hooks/toolPermission/handlers/interactiveHandler.ts` | `executeAsyncClassifierCheck()` |
| 分类器 stub | `src/utils/permissions/bashClassifier.ts` | `classifyBashCommand()` 签名 |
| Auto mode | `src/utils/permissions/yoloClassifier.ts` | 2-stage XML classifier |

---

## 七、术语表

| 术语 | 含义 |
|------|------|
| **Speculative Check** | 在正式需要结果前，提前在后台启动的分类器调用 |
| **Grace Period** | 弹窗显示前的等待窗口（2秒），给 speculative 结果机会 |
| **PendingClassifierCheck** | 附在 ask 决策上的元数据，描述如何后台运行 allow 分类器 |
| **Prompt Rule** | 自然语言描述的安全规则，如 "git log and status commands" |
| **sideQuery** | 轻量级 Anthropic API 调用，通常使用 Haiku 模型做分类 |
| **Behavior** | 权限决策结果类型：`allow` / `deny` / `ask` |
| **YOLO / Auto Mode** | 自动批准模式，使用分类器自动判断而非弹窗 |
| **Coordinator Worker** | 协调器工作线程，等待自动化检查完成后再决定是否弹窗 |
| **Swarm Worker** | 集群工作线程，通过 mailbox 向 leader 转发权限请求 |


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
                           ▼               ▼               ▼
┌────────────────────────────┐    ┌──────────────┐   ┌──────────────────────────────────────┐
│ 进入后处理层 (wrapper)      │    │ 直接 resolve  │   │ 进入后处理层 (wrapper)               │
│                            │    │ deny         │   │                                      │
│ • 连续拒绝计数重置          │    │ → 返回拒绝   │   │ • dontAsk 模式: ask → deny           │
│   (auto mode 中)            │    │   结果       │   │ • auto 模式:                         │
│                            │    │              │   │   - acceptEdits fast-path             │
│                            │    │              │   │   - Safe-tool allowlist 跳过分类器     │
│                            │    │              │   │   - YOLO classifier 判断              │
│                            │    │              │   │   - Denial tracking（连续拒绝上限）    │
│                            │    │              │   │ • headless/async agents:              │
│                            │    │              │   │   PermissionRequest hooks →           │
│                            │    │              │   │   无决策则 auto-deny                  │
│                            │    │              │   │                                      │
│                            │    │              │   │ 后处理仍 ask → useCanUseTool 交互处理: │
│                            │    │              │   │                                      │
│                            │    │              │   │ ① handleCoordinatorPermission()      │
│                            │    │              │   │    (coordinatorHandler.ts)            │
│                            │    │              │   │    → 批准则 resolve allow             │
│                            │    │              │   │                                      │
│                            │    │              │   │ ② handleSwarmWorkerPermission()      │
│                            │    │              │   │    (swarmWorkerHandler.ts)            │
│                            │    │              │   │    → 批准则 resolve allow             │
│                            │    │              │   │                                      │
│                            │    │              │   │ ③ 【2s 宽限期】                       │
│                            │    │              │   │    peekSpeculativeClassifierCheck()   │
│                            │    │              │   │    (bashPermissions.ts)               │
│                            │    │              │   │    → 高置信 allow → resolve allow     │
│                            │    │              │   │    → 超时/不匹配 → 继续弹窗            │
│                            │    │              │   │                                      │
│                            │    │              │   │ ④ handleInteractivePermission()      │
│                            │    │              │   │    (interactiveHandler.ts)            │
│                            │    │              │   │    → 弹窗队列 + PermissionDialog       │
│                            │    │              │   │    → 后台 executeAsyncClassifierCheck │
│                            │    │              │   │    → 用户选择 / 自动批准 → resolve      │
└────────────────────────────┘    └──────────────┘   └──────────────────────────────────────┘
                                                                                         │
                                                                           ┌─────────────┴─────────────┐
                                                                           │                           │
                                                                           ▼                           ▼
                                                                    resolve allow                 resolve deny
                                                                           │                           │
                                                                           ▼                           ▼
┌──────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│  权限 resolve 后，checkPermissionsAndCallTool() 继续:                                                               │
│                                                                                                                   │
│  if (permissionDecision.behavior === 'allow') {                                                                  │
│      ├── 恢复模型原始输入 (processedInput/backfilledClone)                                                          │
│      └── ★ 调用 tool.call() — 每个工具的执行逻辑完全不同                                        │
│          → 真正执行（bash / 文件读写 / 网络请求 / Agent 调用 等）                                                  │
│          → 产出 progress 消息（实时回显）                                                                           │
│          → 返回 ToolResult                                                                                       │
│      ├── endToolExecutionSpan()                                                                                  │
│      └── 产出 tool_result 消息                                                                                   │
│  } else {                                                                                                        │
│      → 产出拒绝/错误的 tool_result 消息                                                                            │
│      → 执行 PermissionDenied hooks（如有）                                                                        │
│  }                                                                                                               │
└──────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

> **★ 图例**：流程图中标 ★ 的步骤为**工具可自定义/个性化**的钩子或属性，其余为通用框架逻辑。

### ★ 工具个性化过程清单

| 个性化过程 | 所在接口/位置 | 影响范围 | 默认值 |
|---|---|---|---|
| **tool.inputSchema** | `Tool` 接口定义 | 所有工具 | 每个工具独立 Zod schema |
| **tool.validateInput** | `Tool` 接口（可选） | 有自定义验证的工具 | `undefined`（跳过） |
| **投机启动 allow 分类器** | `toolExecution.ts:747` | 仅 BashTool（按 `tool.name` 判断） | 其他工具不触发 |
| **tool.checkPermissions** | `Tool` 接口 | 22 工具有自定义，62 工具走默认 | `behavior: 'allow'` |
| **tool.requiresUserInteraction** | `Tool` 接口（可选） | AskUserQuestionTool 等 | `false` |
| **tool.call** | `Tool` 接口 | 所有工具 | 每个工具完全独立实现 |
| **tool.description** | `Tool` 接口 | 所有工具 | 每个工具独立生成弹窗描述 |
| **PermissionDenied hooks** | `toolExecution.ts` 尾部 | 有注册的工具 | 无 |

### 关键节点说明

| 阶段 | 文件 | 作用 |
|---|---|---|
| **投机启动** | `toolExecution.ts:747-765` | BashTool 专属：在 pre-tool hooks 之前提前后台启动 allow 分类器 |
| **通用决策引擎** | `permissions.ts:1158-1319` | `hasPermissionsToUseToolInner()` — 跨工具的 Phase 1/2/3 决策层次 |
| **Bash 权限** | `bashPermissions.ts` | `bashToolHasPermission()` — AST 解析、静态规则、prompt 分类器、路径安全、子命令拆分 |
| **PowerShell 权限** | `powershellPermissions.ts` | `powershellToolHasPermission()` — 类似 Bash 但 PowerShell 语义 |
| **文件写入权限** | `filesystem.ts:1205` | `checkWritePermissionForTool()` — deny规则 → 内部路径 → 安全检查 → acceptEdits → allow规则 |
| **文件读取权限** | `filesystem.ts:1030` | `checkReadPermissionForTool()` — deny规则 → ask规则 → 写入权限覆盖 → 工作目录 → 内部路径 → allow规则 |
| **WebFetch 权限** | `WebFetchTool` | host allowlist 匹配即 allow，否则 ask |
| **Coordinator 审批** | `coordinatorHandler.ts` | 协调器工作线程：等待自动化检查完成后再决定是否弹窗 |
| **Swarm 转发** | `swarmWorkerHandler.ts` | 集群工作线程：转发权限请求到 leader 或通过 classifier |
| **交互弹窗** | `interactiveHandler.ts` | REPL 中显示 PermissionDialog，支持后台分类器自动批准 |
| **2s 宽限期** | `useCanUseTool.tsx:242-304` | 弹窗前等待投机分类器结果，若匹配则跳过弹窗直接 allow |

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
