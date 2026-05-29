# Agent Teams 工作流分析

> 分析版本: 2026-05-12
> 触发开关: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` / `--agent-teams`
> 重点对象: `TeamCreate` / `Agent(name, team_name)` / `SendMessage` / `Task*`
> 分析重点: 从 agent 看到的 prompt、工具描述、系统提醒还原协作工作流

---

## 一、总览

`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` 打开的不是普通的“多开几个 subagent”，而是一套 **team lead + teammate + shared task board + mailbox** 的协作协议。

开启后，主 agent 会被工具 prompt 引导成 team lead：

1. 复杂任务优先创建 team。
2. 把工作拆成共享任务。
3. 按 agent 能力选择 teammate。
4. 通过 `TaskUpdate(owner)` 分配任务。
5. 通过 `SendMessage` 和 teammate 沟通。
6. teammate 完成任务后回到 idle，等待下一条消息或下一项任务。
7. 收尾时 graceful shutdown，再 `TeamDelete` 清理。

普通 `Agent` 是一次性委托；Agent Teams 是长期协作编排。

---

## 二、开启后的行为变化

| 变化 | 文件位置 | 说明 |
|------|----------|------|
| 开关判断 | `src/utils/agentSwarmsEnabled.ts:14` | 外部用户需要 env/`--agent-teams`，还要通过 GrowthBook killswitch；Ant 构建默认开启 |
| API schema 增加 team 字段 | `src/utils/api.ts:86` | 未开启时会过滤 `Agent` 的 `name`、`team_name`、`mode`；开启后模型能看到这些参数 |
| Team 工具加入工具列表 | `src/tools.ts:216` | Task 工具正常存在；开启后额外加入 `TeamCreate`、`TeamDelete` |
| `Agent` 变成 teammate spawn 入口 | `src/tools/AgentTool/AgentTool.tsx:437` | 当 `teamName && name` 时，不是普通 subagent，而是调用 `spawnTeammate()` |
| teammate 可被命名寻址 | `src/tools/AgentTool/AgentTool.tsx:192` | `name` 让 teammate 可以被 `SendMessage({to: name})` 继续沟通 |
| 新增共享团队上下文 | `src/utils/messages.ts:3494` | teammate 首轮收到 team config、task list、leader 名称、按 name 通信等提醒 |
| teammate 系统提示变更 | `src/utils/swarm/teammatePromptAddendum.ts:8` | teammate 被明确告知：用户主要和 team lead 交互，协作靠 task system + messaging |
| 邮箱消息变成上下文附件 | `src/utils/attachments.ts:899` | unread teammate messages 会被注入模型上下文；结构化权限/关机消息走专门处理 |

---

## 三、经典 Agent Team 时序图

下面这个图是我代入 team lead 后会真实执行的经典场景：用户让它完成一个跨步骤需求，leader 先建团队和任务板，再把实现与验收分给不同 teammate，自己保留调度、回查、验收和对用户汇报的职责。

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant L as Team Lead
    participant T as Team / Shared Tasks
    participant W as Worker Teammate
    participant R as Reviewer Teammate
    participant H as Harness / Wakeup

    U->>L: "把这个需求做完" / "继续跑 milestone"
    L->>T: TeamCreate(team_name)
    T-->>L: team config + shared task list ready

    L->>T: TaskCreate(M1 implement)
    L->>T: TaskCreate(M1 review, blockedBy=M1 implement)
    L->>T: TaskCreate(final summarize, blockedBy=M1 review)

    L->>W: Agent(name="worker-1", team_name, prompt=implementation dispatch)
    L->>T: TaskUpdate(M1 implement, owner="worker-1", status="in_progress")
    L->>W: SendMessage(to="worker-1", assignment + expected report format)

    L->>R: Agent(name="reviewer-1", team_name, prompt=review scope)
    L->>T: TaskUpdate(M1 review, owner="reviewer-1")
    L->>R: SendMessage(to="reviewer-1", wait for implementation, then verify)

    L->>H: ScheduleWakeup(delaySeconds=1800, prompt="继续检查 team / milestone 进度")
    L-->>U: 已派出 worker 和 reviewer；30 分钟兜底回查，若 teammate 主动汇报会提前继续

    W->>T: TaskGet(M1 implement) / TaskList
    W->>W: implement / run tests / prepare result
    W->>T: TaskUpdate(M1 implement, status="completed")
    W->>L: SendMessage(to="team-lead", implementation result + changed files + test status)

    H-->>L: immediate wake because teammate message arrived
    L->>T: TaskList / inspect worker result
    L->>R: SendMessage(to="reviewer-1", implementation is ready; start review)

    R->>T: TaskGet(M1 review) / read implementation result
    R->>R: verify behavior / run e2e or acceptance checks
    alt review passes
        R->>T: TaskUpdate(M1 review, status="completed")
        R->>L: SendMessage(to="team-lead", pass report + evidence)
        H-->>L: immediate wake because reviewer message arrived
        L->>T: TaskUpdate(final summarize, status="in_progress")
        L-->>U: 汇总实现、验证结果、剩余风险
        L->>W: SendMessage(shutdown_request)
        W-->>L: SendMessage(shutdown_response approve=true)
        L->>R: SendMessage(shutdown_request)
        R-->>L: SendMessage(shutdown_response approve=true)
        L->>T: TeamDelete()
    else review finds blocker
        R->>T: TaskUpdate(M1 review, status="in_progress", blocker recorded)
        R->>L: SendMessage(to="team-lead", blocker + repro + required fix)
        H-->>L: immediate wake because reviewer message arrived
        L->>T: TaskCreate(fix blocker, owner="worker-1")
        L->>W: SendMessage(to="worker-1", fix blocker with reviewer evidence)
        L->>H: ScheduleWakeup(delaySeconds=1800, prompt="继续检查 blocker 修复进度")
        L-->>U: reviewer 发现阻塞项，已回派修复并安排回查
    end

    opt no teammate reports before timeout
        H-->>L: scheduled prompt injection
        L->>T: TaskList / check owners / decide follow-up
        L->>W: SendMessage(to="worker-1", progress check)
        L->>R: SendMessage(to="reviewer-1", progress check)
    end
```

这张图里有三条线，不能混：

| 线 | 图中动作 | 真实含义 |
|----|----------|----------|
| 状态线 | `TaskCreate` / `TaskUpdate` / `TaskList` | 共享任务板事实源，记录 owner、进度、阻塞、完成 |
| 消息线 | `SendMessage(to="worker-1")` / `SendMessage(to="team-lead")` | 真正让 teammate / leader 收到内容；普通 assistant 文本不会进对方上下文 |
| 唤醒线 | `ScheduleWakeup(delaySeconds, prompt, reason)` | leader 的外部定时回查兜底；如果 teammate 先 `SendMessage`，leader 会被提前唤醒 |

所以真实保障是“双触发”：`ScheduleWakeup` 保证 leader 到点回查，`SendMessage` 保证 teammate 有消息时即时唤醒 leader。

---

## 四、Task 工具到底解决什么

`TaskCreate` / `TaskUpdate` / `TaskList` 的目的不是“让 agent 干活”，而是给团队建立一个 **共享、结构化、可恢复的任务板**。在 Agent Team 里，它相当于一个轻量项目管理数据库；`SendMessage` 才是聊天和通知。

```text
                 ┌──────────────────────────────┐
                 │        Shared Task Board      │
                 │  id / subject / owner /       │
                 │  status / blockedBy / notes   │
                 └───────────────┬──────────────┘
                                 │
        TaskCreate               │               TaskList
   "有哪些工作单元"              │          "现在全局进度怎样"
 ┌──────────────────┐            │            ┌──────────────────┐
 │ Team Lead         │────────────┼───────────>│ Team Lead         │
 │ 拆 milestone       │            │            │ 回查/验收/收尾     │
 └──────────────────┘            │            └──────────────────┘
                                 │
        TaskUpdate               │               TaskList / TaskGet
   "谁负责、做到哪了"             │          "我该做哪一项"
 ┌──────────────────┐            │            ┌──────────────────┐
 │ Worker / Reviewer │───────────┴───────────>│ Worker / Reviewer │
 │ claim / progress  │                         │ 找任务/读要求      │
 └──────────────────┘                         └──────────────────┘
```

三个工具分别承担不同职责：

| 工具 | 真实目的 | 我作为 team lead 会在什么时候用 |
|------|----------|----------------------------------|
| `TaskCreate` | 把模糊需求拆成可分配、可阻塞、可验收的工作单元 | 建 team 后先创建 implement / review / summarize 等任务；发现 blocker 时新建修复任务 |
| `TaskUpdate` | 改变任务板事实：owner、状态、依赖、阻塞、完成 | 指派 teammate、标记 in_progress、记录 reviewer blocker、确认完成 |
| `TaskList` | 读取团队当前全局状态 | 被 `ScheduleWakeup` 拉起后先看进度；收到 teammate 消息后核对状态；收尾前确认没有 pending/in_progress |

这和 `SendMessage` 的边界很重要：

| 能力 | `Task*` | `SendMessage` |
|------|---------|---------------|
| 保存团队事实状态 | 是 | 否 |
| 表达依赖和阻塞 | 是 | 只能自然语言说明 |
| 让另一个 agent 立刻看到一段话 | 否 | 是 |
| 被 leader 定时回查时用于恢复局面 | 是，`TaskList` 是第一步 | 只能看到已投递消息，不适合作全局状态源 |
| 适合写“任务完成”结论 | 是，`TaskUpdate(status="completed")` | 适合补充证据、文件、测试结果 |

如果只靠 `SendMessage`，team lead 会有几个问题：

1. 没有一个结构化地方知道“还有哪些任务没完成”。
2. teammate 之间不知道谁已经 claim 了哪项工作。
3. 依赖关系只能散落在聊天里，reviewer 不知道什么时候该开始。
4. `ScheduleWakeup` 到点恢复时，leader 需要重新读一堆消息才能判断状态。
5. context 被压缩或 teammate idle 后，团队事实容易变成自然语言碎片。

所以经典 Agent Team 的真实分工是：

```text
TaskCreate   = 把需求变成任务图
TaskUpdate   = 更新任务图上的事实
TaskList     = 读取当前团队局面
SendMessage  = 发送指令、证据、阻塞说明
ScheduleWakeup = leader 的定时回查闹钟
```

一句话：**Task 工具管“状态”，SendMessage 管“沟通”，ScheduleWakeup 管“什么时候回来继续看状态”。**

---

## 五、Prompt 地图

| 阶段 | 文件位置 | Prompt / 工具 | 对模型形成的行为约束 |
|------|----------|---------------|----------------------|
| 是否组队 | `src/tools/TeamCreateTool/prompt.ts:5` | `TeamCreate` when-to-use | 用户提到 team/swarm/collaborate，或复杂任务适合并行时，主动建 team；拿不准时偏向建 team |
| 选 teammate 类型 | `src/tools/TeamCreateTool/prompt.ts:14` | Choosing Agent Types | read-only agent 只做研究/规划；full-capability agent 才做实现；自定义 agent 要看描述和工具限制 |
| 团队主流程 | `src/tools/TeamCreateTool/prompt.ts:37` | Team Workflow | 建队 → 建任务 → spawn teammate → assign owner → teammate 完成任务 → shutdown |
| 自动消息交付 | `src/tools/TeamCreateTool/prompt.ts:51` | Automatic Message Delivery | team lead 不需要手动查 inbox；消息自动成为新 turn 或排队 |
| idle 语义 | `src/tools/TeamCreateTool/prompt.ts:65` | Teammate Idle State | idle 是正常等待输入，不代表完成、失败或不可用 |
| 成员发现 | `src/tools/TeamCreateTool/prompt.ts:74` | Discovering Team Members | teammate 用 team config 发现成员；通信和 owner 都用 human-readable `name`，不要用 UUID |
| 任务协作 | `src/tools/TeamCreateTool/prompt.ts:93` | Task List Coordination | teammate 周期性 `TaskList`，优先低 ID，claim unassigned/unblocked 任务，完成后继续找下一项 |
| 普通 Agent 委托 | `src/tools/AgentTool/prompt.ts:202` | Agent prompt | 普通 Agent 是 fresh context；要完整 briefing；结果不直接给用户，主 agent 要汇总 |
| Team spawn 参数 | `src/tools/AgentTool/AgentTool.tsx:192` | `name` / `team_name` / `mode` schema | `name + team_name` 把 Agent 从一次性 subagent 变成 team teammate |
| 通信工具 | `src/tools/SendMessageTool/prompt.ts:23` | `SendMessage` | 普通文本对其他 agent 不可见；跨 agent 必须用 `SendMessage` |
| 通信对象 | `src/tools/SendMessageTool/prompt.ts:31` / 最新运行日志 `SendMessage` schema | `to` table | 最新日志里 `to` 只暴露 teammate name；不要把广播当成当前主流程能力 |
| 结构化响应 | `src/tools/SendMessageTool/prompt.ts:38` | Protocol responses | 收到 shutdown/plan approval request 时，用匹配的 `_response` 协议回复 |
| teammate 身份提醒 | `src/utils/messages.ts:3505` | `team_context` attachment | teammate 首轮知道自己名字、team config、task list、team-lead |
| teammate 系统提示 | `src/utils/swarm/teammatePromptAddendum.ts:8` | System prompt addendum | teammate 必须通过 `SendMessage` 沟通；用户主要和 team lead 交互 |
| Plan 后并行提示 | `src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts:470` | exit plan result | 计划批准后，如果能拆成独立任务，会提示考虑 `TeamCreate` 并行化 |
| 清理约束 | `src/tools/TeamDeleteTool/prompt.ts:1` | `TeamDelete` | 只有所有 teammate graceful shutdown 后才能清理 team/task 目录 |
| leader 回查唤醒 | 最新运行日志 / 实际 Agent Team 截屏 `ScheduleWakeup` | timed wakeup / prompt reinjection | leader 派出长跑 teammate 后，用它安排 N 分钟后带指定 prompt 回查；teammate `SendMessage` 也可提前唤醒 |
| agent 结果核验 | 最新运行日志 `Agent` tool description | Trust but verify | agent summary 只说明它声称做了什么；写代码 agent 必须检查实际变更后再汇报 |

---

## 六、经典工作流

### 1. Team lead 判断是否该组队

适合组队的典型任务：

| 场景 | 推荐动作 | 原因 |
|------|----------|------|
| 多模块 feature | `TeamCreate` + 多个实现 teammate | 前端、后端、测试可以并行 |
| 大型 refactor | `TeamCreate` + task board | 需要拆范围、保持测试、减少互相踩踏 |
| research + implementation + verification | 分配 researcher / implementer / verifier | 工具能力和上下文目标不同 |
| 用户明确要求 agents collaborate | 直接 `TeamCreate` | prompt 明确要求 proactive use |

不适合组队的任务：

| 场景 | 推荐动作 |
|------|----------|
| 单文件、小修、小问答 | 主 agent 直接做 |
| 需要强顺序依赖、不能并行 | 先自己推进或只 spawn 一个 agent |
| 任务定义不清，拆不出 owner | 先澄清/规划，再决定是否建 team |

### 2. 建队和建任务

Team lead 的第一步不是先 spawn 人，而是先建立共享事实源：

```json
{"team_name": "my-project", "description": "Working on feature X"}
```

然后用 `TaskCreate` 建任务。任务描述要足够让另一个 agent 独立完成，不要只写一句短命令。

推荐任务粒度：

| 粒度 | 好例子 | 差例子 |
|------|--------|--------|
| 明确产出 | “实现登录表单错误态，覆盖输入校验和 API 错误” | “做前端” |
| 明确边界 | “只修改 auth UI，不碰 server API” | “修一下 auth” |
| 可验证 | “运行 auth 相关测试并报告结果” | “看看有没有问题” |
| 可并行 | “补充 parser 单测，不改 parser 实现” | “研究后顺便修” |

### 3. Spawn teammate

核心触发条件是 `Agent` 同时带上 `team_name` 和 `name`：

```json
{
  "name": "tester",
  "team_name": "my-project",
  "subagent_type": "general-purpose",
  "description": "Run auth tests",
  "prompt": "You are tester in team my-project. Claim task #3, run the auth test suite, investigate failures only within test scope, update the task status, and SendMessage team-lead with a concise result."
}
```

选择 agent 类型时，TeamCreate prompt 给了明确原则：

| teammate 类型 | 应该分配 | 不应该分配 |
|---------------|----------|------------|
| read-only / Explore / Plan | 搜索、阅读、方案、风险分析 | 写代码、改文件 |
| full-capability / general-purpose | 实现、测试、修复、脚本运行 | 需要隔离评审的独立判断 |
| custom agent | 符合其 description 和 tools 的任务 | 超出工具权限的任务 |

### 4. 分配和领取任务

任务状态是团队协作的中心，不是聊天记录。

| 动作 | 工具 | 说明 |
|------|------|------|
| 创建任务 | `TaskCreate` | 默认 `pending`、无 owner |
| 指派任务 | `TaskUpdate(owner: "name")` | owner 使用 teammate name |
| 开始工作 | `TaskUpdate(status: "in_progress")` | teammate 开始前应标记 |
| 完成任务 | `TaskUpdate(status: "completed")` | 只有 fully accomplished 才能完成 |
| 表达阻塞 | 新建 blocker task 或更新依赖 | 不要把失败/半成品标 completed |
| 找下一项 | `TaskList` | 完成后必须查新任务，优先低 ID |

### 5. 通信方式

Agent Teams 的关键约束：**普通文本不是团队消息**。

Team lead 或 teammate 要让别人看到消息，必须用：

```json
{"to": "researcher", "summary": "start task 1", "message": "Please claim task #1 and report findings when done."}
```

通信原则：

| 原则 | 说明 |
|------|------|
| 用 name，不用 UUID | team config 里 `name` 是通信和 owner 的唯一稳定人类接口 |
| 不把广播当主流程 | 最新日志的 `SendMessage` schema 只暴露 teammate name；按当前工具面，默认点对点 DM |
| 不要手动查 inbox | prompt 告诉 team lead 消息会自动交付 |
| 不要复述原文 | teammate 消息已经展示给用户，team lead 汇总即可 |
| 不发 JSON 状态 | 普通协作状态用自然语言；任务状态用 `TaskUpdate` |

### 6. Idle 的正确理解

`idle` 在 Agent Teams 里是正常状态：

| 现象 | 正确理解 | 该做什么 |
|------|----------|----------|
| teammate 发完消息后 idle | 它在等输入 | 需要继续就 `SendMessage` |
| teammate 完成 task 后 idle | 它可能等下一项 | 指派新任务或让它 `TaskList` |
| 多个 idle notification | 系统会折叠，只保留最新 | 不要当错误 |
| teammate blocked 后 idle | 它等 team lead 或其他任务解除阻塞 | 新建/重排 blocker task |

---

## 七、Team lead 的理想循环

### Team lead 抽象流程

```text
1. 判断任务是否值得组队
2. TeamCreate 建团队
3. TaskCreate 拆任务
4. Agent(name, team_name, subagent_type) 创建 teammate
5. TaskUpdate(owner) 分配任务
6. 等待自动消息 / 观察 task board
7. 根据结果继续分配、拆新任务、解除 blocker
8. 汇总 teammate 结果给用户
9. SendMessage shutdown_request
10. TeamDelete 清理
```

这个循环里，主 agent 不应该把自己降级成“等所有人回报的被动收件箱”。它仍然是 coordinator：

| Team lead 职责 | 具体动作 |
|----------------|----------|
| 拆解 | 把用户目标拆成可并行、可验证的 tasks |
| 分配 | 根据工具权限和任务性质选择 teammate |
| 同步 | 用 task board 保持全局状态 |
| 决策 | teammate 返回冲突信息时，由 team lead 做取舍 |
| 整合 | teammate 输出对用户不可见时，team lead 负责摘要和最终答案 |
| 收尾 | graceful shutdown + cleanup |

从工具视角看，抽象流程可以压成这张表：

| 阶段 | leader 在想什么 | 主要工具 | 产物 |
|------|----------------|----------|------|
| 识别 | 这个需求是否需要多人协作 | 无，模型判断 | 是否进入 Agent Team |
| 建场 | 团队需要一个共享空间 | `TeamCreate` | team config + task list dir |
| 拆解 | 哪些工作可并行，哪些必须串行 | `TaskCreate` | 一组 pending tasks |
| 组队 | 哪个 teammate 适合哪项任务 | `Agent(name, team_name)` | 可寻址 teammate |
| 分配 | 谁负责哪项，当前状态是什么 | `TaskUpdate` | owner/status 写入任务板 |
| 通信 | 需要把具体指令/证据交给谁 | `SendMessage` | 对方 mailbox 收到消息 |
| 等待 | 暂时没有可做动作，但要回来查 | `ScheduleWakeup` | 未来 prompt reinjection |
| 回查 | 现在全局进度如何 | `TaskList` | 最新 task board 状态 |
| 决策 | pass、blocker、返工、继续分配 | `TaskUpdate` + `SendMessage` | 新状态 / 新消息 |
| 收尾 | 团队不再需要运行 | `SendMessage(shutdown_request)` + `TeamDelete` | teammate 退出，team 清理 |

### Team lead 具体轨迹示例

假设用户说：“把 feature-x 做完，并让另一个 agent 做验收”。我作为 team lead 的工具调用颗粒度会是这样：

#### 1. 建 team：创建共享协作空间

```json
TeamCreate({
  "team_name": "feature-x",
  "description": "Implement feature-x with separate implementation and review teammates"
})
```

这一步创建两类持久状态：

| 产物 | 用途 |
|------|------|
| `~/.claude/teams/feature-x/config.json` | 记录 team lead、members、agent name、agent id |
| `~/.claude/tasks/feature-x/` | 团队共享任务板，后续 `TaskCreate` / `TaskUpdate` / `TaskList` 都围绕它工作 |

#### 2. 建任务：把用户目标拆成可分配单元

```json
TaskCreate({
  "subject": "Implement feature-x",
  "description": "Build the feature-x behavior end to end. Keep scope limited to the agreed files. Run the relevant tests before marking complete.",
  "activeForm": "Implementing feature-x"
})
```

```json
TaskCreate({
  "subject": "Review feature-x",
  "description": "After implementation is complete, verify the user-visible behavior, inspect changed files, run acceptance checks, and report pass/fail with evidence.",
  "activeForm": "Reviewing feature-x"
})
```

```json
TaskCreate({
  "subject": "Summarize feature-x outcome",
  "description": "After implementation and review are complete, produce the final user-facing summary with tests, files, risks, and next steps.",
  "activeForm": "Summarizing feature-x"
})
```

颗粒度原则：一个 `TaskCreate` 应该对应一个可独立完成、可验收的工作单元。不要把“实现、review、总结、修 blocker”都塞进一个 task；否则 owner、blockedBy、状态流都会失真。

#### 3. Spawn worker：创建可寻址 teammate

```json
Agent({
  "name": "worker-1",
  "team_name": "feature-x",
  "subagent_type": "general-purpose",
  "description": "Implement feature-x",
  "prompt": "You are worker-1 in team feature-x. Claim the implementation task, mark it in_progress, implement feature-x only within the task scope, run the relevant tests, then mark the task completed only if fully done. SendMessage team-lead with changed files, tests, and blockers."
})
```

这里的关键不是 `subagent_type`，而是 `name + team_name`。这两个参数让它从“一次性 subagent”变成团队里的 `worker-1`，后续可以被 `SendMessage(to="worker-1")` 点名唤醒。

#### 4. 更新任务板：记录 owner 和状态

```json
TaskUpdate({
  "taskId": "1",
  "owner": "worker-1",
  "status": "in_progress"
})
```

这一步不是给 worker 发消息，而是写入共享事实：task #1 现在由 `worker-1` 负责，状态是进行中。leader、worker、reviewer 后面都能通过 `TaskList` 看到这个事实。

#### 5. Spawn reviewer：提前准备验收者

```json
Agent({
  "name": "reviewer-1",
  "team_name": "feature-x",
  "subagent_type": "general-purpose",
  "description": "Review feature-x",
  "prompt": "You are reviewer-1 in team feature-x. Wait until the implementation task is complete, then review feature-x from the user-visible behavior perspective. Use TaskList to check readiness. Report pass/fail to team-lead with evidence."
})
```

```json
TaskUpdate({
  "taskId": "2",
  "owner": "reviewer-1"
})
```

review task 此时可以只设 owner，不一定设 `in_progress`。如果 reviewer 还在等待 implement 完成，把它标成 `in_progress` 反而会让任务板看起来像已经开始验收。

#### 6. 用 SendMessage 做明确通信

如果 spawn prompt 已经足够完整，首条 `SendMessage` 可以省；但在真实长跑团队里，我通常会用它做“唤醒/补充/转交证据”，而不是当任务板用。

```json
SendMessage({
  "to": "worker-1",
  "summary": "start implementation task",
  "message": "Task #1 is assigned to you. Keep the scope tight, update TaskUpdate when complete, and SendMessage team-lead with files changed and tests run."
})
```

```json
SendMessage({
  "to": "reviewer-1",
  "summary": "wait for review",
  "message": "Task #2 is assigned to you. Wait for task #1 to complete, then review the behavior and report pass/fail with evidence."
})
```

#### 7. 安排兜底回查

```json
ScheduleWakeup({
  "delaySeconds": 1800,
  "prompt": "/change-orchestrator 继续 feature-x,检查 worker-1/reviewer-1 进度",
  "reason": "checking teammate progress after 30 minutes"
})
```

这一步不改任务板，也不发消息给 teammate。它只是告诉 harness：如果期间没人主动 `SendMessage` 唤醒我，30 分钟后把我这个 leader 拉回来继续检查。

#### 8. 收到 worker 汇报后：先读任务板，再转 reviewer

worker 完成后通常会做两件事：

```json
TaskUpdate({
  "taskId": "1",
  "status": "completed"
})
```

```json
SendMessage({
  "to": "team-lead",
  "summary": "implementation complete",
  "message": "Task #1 complete. Changed files: ... Tests: ... Notes: ..."
})
```

leader 被唤醒后，不应该只相信消息文本；先读任务板：

```json
TaskList({})
```

确认 task #1 completed 后，再让 reviewer 开始：

```json
TaskUpdate({
  "taskId": "2",
  "status": "in_progress"
})
```

```json
SendMessage({
  "to": "reviewer-1",
  "summary": "start review now",
  "message": "Task #1 is complete. Start task #2 now. Verify behavior, inspect the diff, run checks, then report pass/fail."
})
```

#### 9. reviewer pass：收尾、汇总、关停

```json
TaskList({})
```

```json
TaskUpdate({
  "taskId": "3",
  "owner": "team-lead",
  "status": "in_progress"
})
```

leader 汇总给用户后，开始 graceful shutdown：

```json
SendMessage({
  "to": "worker-1",
  "message": {
    "type": "shutdown_request",
    "reason": "feature-x implementation and review are complete"
  }
})
```

```json
SendMessage({
  "to": "reviewer-1",
  "message": {
    "type": "shutdown_request",
    "reason": "feature-x implementation and review are complete"
  }
})
```

teammate 回复 `shutdown_response` 后，leader 才清理：

```json
TeamDelete({})
```

这条链路里每个工具的职责边界是：

| 工具 | 颗粒度 | 改变了什么 |
|------|--------|------------|
| `TeamCreate` | 每个协作项目一次 | 创建 team config + task list 空间 |
| `TaskCreate` | 每个可验收工作单元一次 | 新增任务事实 |
| `Agent(name, team_name)` | 每个长期 teammate 一次 | 新增可寻址团队成员 |
| `TaskUpdate` | 每次 owner/status/blocker 变化 | 更新任务板事实 |
| `SendMessage` | 每次需要让某个 teammate 看到具体内容 | 写入对方 mailbox / 唤醒对方 |
| `TaskList` | 每次恢复上下文、回查、收尾前 | 读取团队全局状态 |
| `ScheduleWakeup` | leader 暂时无事但需要兜底回查时 | 注册未来恢复 prompt |
| `TeamDelete` | 全员 shutdown 后一次 | 清理 team + task 目录 |

---

## 八、Teammate 的理想循环

### Teammate 抽象流程

```text
1. 消化首轮注入的 team_context，确认自己的 name、team config、task list
2. 读取 TaskList，确认自己被分配的任务或可领取任务
3. TaskUpdate 标记 owner / in_progress
4. 执行任务
5. 如需沟通，用 SendMessage，不要只写普通文本
6. 完成后 TaskUpdate completed
7. TaskList 找下一项
8. 没有可做任务或被阻塞时，SendMessage team-lead
9. 收到 shutdown_request 时按协议回复
```

这里的 `team_context` 不是一个需要 teammate 主动调用的工具，而是 harness 在 teammate 首轮请求里自动塞进上下文的 meta attachment。源码里它的 `Attachment` 结构包含：

| 字段 | 含义 |
|------|------|
| `teamName` | 当前 teammate 属于哪个 team |
| `agentId` / `agentName` | 当前 teammate 的机器 ID 和人类可读名字 |
| `teamConfigPath` | `~/.claude/teams/{team-name}/config.json`，用于发现 teammate 名字 |
| `taskListPath` | `~/.claude/tasks/{team-name}/`，团队共享任务板目录 |

注入到模型时，它会变成一段 `system-reminder`，核心内容类似：

```text
# Team Coordination

You are a teammate in team "<teamName>".

Your Identity:
- Name: <agentName>

Team Resources:
- Team config: <teamConfigPath>
- Task list: <taskListPath>

Team Leader: The team lead's name is "team-lead".
Send updates and completion notifications to them.
```

所以“读取 team_context”的准确含义是：teammate 在开始工作前先从这段注入上下文里知道 **我是谁、我在哪个 team、任务板在哪里、leader 叫什么、发消息要用 name 而不是 UUID**。随后它才会用 `TaskList` / `TaskGet` 去读具体任务，用 `SendMessage(to="team-lead")` 汇报。

teammate prompt 的核心定位是：用户主要和 team lead 交互，teammate 通过任务系统和消息系统工作。因此 teammate 的好行为不是“最终回答用户”，而是“更新任务事实 + 向 team lead 汇报必要信息”。

从工具视角看，teammate 的抽象流程是：

| 阶段 | teammate 在想什么 | 主要工具 | 产物 |
|------|------------------|----------|------|
| 定位 | 我是谁，leader 是谁，任务板在哪 | 首轮 `team_context` | 身份和资源路径进入上下文 |
| 找活 | 我被分配了什么，是否有可领取任务 | `TaskList` | 当前任务候选 |
| 读要求 | 这个 task 的完整退出条件是什么 | `TaskGet` | 具体任务说明 |
| 占位 | 我开始做，别人不要重复做 | `TaskUpdate(owner/status)` | 任务板 owner / in_progress |
| 执行 | 完成任务本体 | `Read` / `Edit` / `Bash` 等 | 文件变更 / 测试结果 |
| 同步 | 我完成了、阻塞了、需要决策 | `TaskUpdate` + `SendMessage` | 任务事实 + leader 可读消息 |
| 续航 | 做完后是否还有下一项 | `TaskList` | 下一项 / idle 判断 |
| 退出 | leader 要求 shutdown | `SendMessage(shutdown_response)` | 协议化退出确认 |

### Teammate 具体轨迹示例

假设我是 `worker-1`，首轮上下文里已经被注入 `team_context`，知道自己属于 `feature-x`，leader 名字是 `team-lead`。我的完整工具调用会是这样：

#### 1. 先看任务板，不直接开干

```json
TaskList({})
```

目的：确认哪些任务分给了我、哪些任务未领取、哪些任务被依赖阻塞。teammate 不应该只凭 spawn prompt 工作，因为 team lead 可能已经在任务板里改过 owner、依赖或优先级。

如果看到 task #1 分给自己：

```json
TaskGet({
  "taskId": "1"
})
```

目的：读取完整任务描述、exit criteria、blockedBy，而不是只看列表摘要。

#### 2. Claim / 开始任务

如果 task 已经 owner=`worker-1`：

```json
TaskUpdate({
  "taskId": "1",
  "status": "in_progress"
})
```

如果 task 还没有 owner，但我是最合适的人选：

```json
TaskUpdate({
  "taskId": "1",
  "owner": "worker-1",
  "status": "in_progress"
})
```

目的：告诉整个团队“我已经开始做这个任务了”。这可以避免另一个 teammate 同时 claim 同一项。

#### 3. 执行实际工作

这里才进入普通 coding agent 的工具调用，比如：

```json
Read({
  "file_path": "/path/to/relevant/file.ts"
})
```

```json
Edit({
  "file_path": "/path/to/relevant/file.ts",
  "old_string": "...",
  "new_string": "..."
})
```

```json
Bash({
  "command": "bun test path/to/test.ts",
  "description": "Run focused feature-x tests"
})
```

这些工具只解决“任务内容本身”。它们不会自动告诉 team lead 进度，也不会更新任务板。

#### 4. 遇到阻塞：不要假装完成

如果发现需求不清、测试环境缺失、依赖另一个任务：

```json
TaskUpdate({
  "taskId": "1",
  "status": "in_progress",
  "metadata": {
    "blocker": "Need API contract from reviewer-1 before continuing"
  }
})
```

```json
SendMessage({
  "to": "team-lead",
  "summary": "blocked on API contract",
  "message": "Task #1 is blocked. I need the API contract clarified before continuing. Current state: ... Proposed options: ..."
})
```

颗粒度原则：阻塞状态写任务板，阻塞原因和取舍建议发消息。不要只发普通 assistant 文本，因为 team lead / teammate 看不到。

#### 5. 完成任务：先验证，再改 completed

只有在任务 fully accomplished 且验证通过时：

```json
TaskUpdate({
  "taskId": "1",
  "status": "completed"
})
```

然后发给 leader 一条可行动的结果消息：

```json
SendMessage({
  "to": "team-lead",
  "summary": "implementation complete",
  "message": "Task #1 completed. Changed files: src/a.ts, src/b.ts. Verification: bun test path/to/test.ts passed. Notes: reviewer should verify the user-visible flow."
})
```

注意顺序：`TaskUpdate(completed)` 是事实状态；`SendMessage` 是证据和上下文。两者都需要。

#### 5.1 如果只输出普通文本，不调用 SendMessage，会发生什么

这是 Agent Team 里最容易误解的点。先看结论：**leader LLM 实际拿到的是 mailbox 字符串，不是 teammate 的普通输出。**

如果 `worker-1` 停下来但没有 `SendMessage`，leader 下一轮上下文里看到的 teammate 消息大致是：

```xml
<teammate-message teammate_id="worker-1">
{"type":"idle_notification","from":"worker-1","timestamp":"2026-05-12T10:12:34.567Z","idleReason":"available"}
</teammate-message>
```

如果有颜色或 summary，外层可能多属性：

```xml
<teammate-message teammate_id="worker-1" color="blue" summary="...">
{"type":"idle_notification","from":"worker-1","timestamp":"2026-05-12T10:12:34.567Z","idleReason":"available"}
</teammate-message>
```

这就是 “leader 只收到 idle_notification” 的具体字符串形态。它只表示 `worker-1` 这一轮停下来了、现在 available，不表示任务完成。

如果 teammate 完成任务后只写一段普通 assistant 文本，比如：

```text
我已经完成了 task #1，修改了 src/a.ts，测试通过。
```

但没有调用 `SendMessage`，leader **不会**看到：

```xml
<teammate-message teammate_id="worker-1">
我已经完成了 task #1，修改了 src/a.ts，测试通过。
</teammate-message>
```

也就是说，普通文本不会自动变成 mailbox 消息。要让 leader LLM 拿到完成报告，teammate 必须显式调用：

```json
SendMessage({
  "to": "team-lead",
  "summary": "implementation complete",
  "message": "Task #1 completed. Changed files: ... Tests: ..."
})
```

源码里 in-process teammate runner 对这个行为写得很直白：**不会自动把 teammate 的普通 response 发给 leader**。这和 process-based teammate 保持一致；跨 agent 可见的内容必须走 mailbox，也就是 `SendMessage`。

所以 leader 能感知到的不是“任务完成了”，而只是“这个 teammate 停下来了 / idle 了”。如果 task board 还没 `completed`，leader 正确反应应该是：

```json
TaskList({})
```

然后发现 task 仍然 `in_progress` 或 `pending`，再追问：

```json
SendMessage({
  "to": "worker-1",
  "summary": "report task status",
  "message": "I see task #1 is not marked completed. If it is done, update TaskUpdate(status=completed) and SendMessage me the changed files and test results."
})
```

会不会有 harness 自动插入 `system-reminder` 来纠正它？默认不会有一个专门的“你忘了 SendMessage”的补救 reminder。已有的约束来自两个地方：

| 机制 | 是否默认存在 | 作用 |
|------|--------------|------|
| teammate system prompt addendum | 是 | 开局就告诉 teammate：普通文本对团队不可见，必须用 `SendMessage` |
| `team_context` attachment | 是，只在 teammate 首轮 | 告诉 teammate 自己是谁、leader 是谁、task list 在哪 |
| idle notification | 是 | turn 结束后通知 leader：teammate idle / available |
| 自动转发普通文本给 leader | 否 | 不会发生 |
| 忘记 `SendMessage` 后自动插入纠错 reminder | 默认否 | 不会仅因为普通文本就自动补救 |
| `TeammateIdle` / `TaskCompleted` hook | 取决于配置 | 如果用户配置了阻塞 hook，hook 可以返回反馈并阻止 idle，让 teammate 继续工作 |

也就是说，只有在额外配置了类似 `TeammateIdle` 或 `TaskCompleted` 的 hook，并且 hook 用 exit code 2 / block 决策阻止 idle 时，harness 才会把类似：

```text
TeammateIdle hook feedback:
...
```

作为 meta 消息塞回 teammate，让它继续补工具调用。没有这种 hook 时，普通文本就是 teammate 自己 transcript 里的文本，不是团队通信。

#### 6. 完成后找下一项

```json
TaskList({})
```

如果有新的 unassigned / unblocked task，teammate 可以按 team prompt 的规则领取低 ID 任务：

```json
TaskUpdate({
  "taskId": "4",
  "owner": "worker-1",
  "status": "in_progress"
})
```

如果没有可做任务：

```json
SendMessage({
  "to": "team-lead",
  "summary": "worker idle",
  "message": "I completed task #1 and found no unblocked tasks assigned to me. I am idle and available for follow-up."
})
```

#### 7. 收到 shutdown_request

如果 leader 发来：

```json
{
  "type": "shutdown_request",
  "reason": "feature-x implementation and review are complete"
}
```

teammate 应该用协议响应，而不是普通文本：

```json
SendMessage({
  "to": "team-lead",
  "message": {
    "type": "shutdown_response",
    "request_id": "<request_id>",
    "approve": true
  }
})
```

teammate 视角的职责边界：

| 工具 | 什么时候用 | 不该拿它做什么 |
|------|------------|----------------|
| `TaskList` | 开始前、完成后、被唤醒后 | 不用它发送解释性消息 |
| `TaskGet` | 开始具体 task 前 | 不用列表摘要替代完整要求 |
| `TaskUpdate` | claim、开始、阻塞、完成 | 不用它写长篇结果报告 |
| `SendMessage` | 汇报完成、阻塞、请求决策、响应 shutdown | 不用普通文本替代它 |
| `Read/Edit/Bash` | 执行实际任务 | 不用它们维护团队状态 |

---

## 九、与普通 Agent / background Agent 的区别

| 维度 | 普通 Agent | Background Agent | Agent Teams |
|------|------------|------------------|-------------|
| 生命周期 | 一次性任务 | 后台一次性任务 | 长生命周期 teammate |
| 上下文 | fresh context，需要完整 briefing | fresh context，需要完整 briefing | 有 team context、task list、mailbox |
| 寻址 | 通过 agent id/name 继续 | 通过 `SendMessage` 继续 | 通过 teammate name 作为团队成员沟通 |
| 协作状态 | 主 agent 自己记 | 主 agent 等通知 | 共享 task board |
| 用户可见性 | 结果需主 agent 汇总 | 完成通知后汇总 | teammate 消息自动交付，仍由 team lead 汇总 |
| 最佳用途 | 独立研究/执行 | 长耗时独立任务 | 多人协作、并行拆解、持续分配 |

---

## 十、常见误用

| 误用 | 问题 | 正确做法 |
|------|------|----------|
| 只 spawn teammate，不建任务 | 队友没有共享状态源 | 先 `TeamCreate` + `TaskCreate` |
| 用普通文本回复 teammate | 其他 agent 看不到 | 必须 `SendMessage` |
| owner 用 UUID | prompt 明确要求用 name | `owner: "researcher"` |
| 把 idle 当完成 | idle 只是等待输入 | 看 task status 和消息内容 |
| 假设可以广播所有消息 | 最新日志没有暴露 `to:"*"` 广播 schema | 默认点对点 DM 给具体 teammate |
| 给 read-only agent 分配实现 | 工具权限不匹配 | 研究给 read-only，实现给 full-capability |
| teammate 再 spawn teammate | roster 是扁平结构，会被拒绝 | teammate 只能 spawn 普通 subagent，不能 spawn teammate |
| 完成部分工作就 completed | TaskUpdate prompt 明确禁止 | blocked/failed 时保留 in_progress 或新建 blocker |
| TeamDelete 前不 shutdown | 工具会拒绝 active members | 先 `SendMessage` shutdown_request |

---

## 十一、实战模板

### Team lead prompt 模板

```text
We are working as team "{teamName}".
Your name is "{name}".
Claim task #{taskId}, mark it in_progress, and work only within this scope:

Scope:
- ...

Do not modify:
- ...

When done:
1. Run the relevant verification.
2. Mark task #{taskId} completed only if fully done.
3. SendMessage team-lead with summary, files changed, tests run, and blockers if any.
```

### Task description 模板

```text
Goal:
- ...

Files / areas:
- ...

Constraints:
- ...

Done means:
- ...

Verification:
- ...
```

### Teammate completion message 模板

```json
{
  "to": "team-lead",
  "summary": "task 3 completed",
  "message": "Task #3 is completed. Changed src/... and tests/.... Verification: npm test -- auth passed. No blockers."
}
```

---

## 十二、结论

开启 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` 后，Claude Code 的 agent 能力从“委托型 subagent”升级为“团队编排型 swarm”。

它的经典工作流不是代码层面的执行流，而是 prompt 设计出来的组织行为：

```text
TeamCreate 建组织
TaskCreate 建共享事实源
Agent(name, team_name) 建可寻址队友
TaskUpdate 分配和推进状态
SendMessage 做跨 agent 通信
TeamDelete 做收尾清理
```

如果只把它当作“并行调用多个 Agent”，会错过最核心的设计：**任务板是状态源，SendMessage 是通信协议，team lead 是编排者，teammate 是可持续协作的执行单元。**

---

## 十三、最新日志补充：ScheduleWakeup

最新运行日志里已经明确暴露 `ScheduleWakeup` 工具，实际 Agent Team leader 也会在正常团队工作流里调用它安排“30 分钟后回查”。当前这份反编译代码中还搜不到对应实现，只说明本地源码快照落后于日志对应版本。

因此要修正前面的判断：`ScheduleWakeup` 不是 teammate 通信工具，但它是 **Agent Team leader 的调度/回查工具**。它把“什么时候回来继续看局面”交给 harness，而不是让模型 sleep 或轮询。

| 维度 | `ScheduleWakeup` | `SendMessage` | `CronCreate` | `Monitor` / background Bash |
|------|------------------|---------------|--------------|-----------------------------|
| 核心用途 | 让当前 session 在未来带指定 prompt 恢复 | 给 agent / teammate 发消息 | 创建一次性或周期性定时任务 | 等待进程、日志、CI、服务状态 |
| 触发对象 | 当前 leader session / orchestrator skill | 另一个 agent / teammate | 调度器在未来 enqueue prompt | 后台进程输出通知 |
| 是否用于团队通信 | 否 | 是 | 否 | 否 |
| 是否持久 | 由 harness 管理当前 session 的未来唤醒 | 消息进 mailbox / agent context | 可 durable / session-only | 当前 session task |
| 典型场景 | “reviewer-1 30 分钟后回查” | “tester，继续 task #3” | “明天 9 点提醒我检查 deploy” | “build 完成/日志出现 ERROR 时通知我” |

`ScheduleWakeup` 的关键不是“模型自己等 30 分钟”，而是 **harness 在外部定时重新注入 prompt**：

| 规则 | 含义 |
|------|------|
| `delaySeconds` 被 runtime clamp 到 `[60, 3600]` | 最短 1 分钟，最长 1 小时 |
| Anthropic prompt cache TTL 是 5 分钟 | 300 秒内恢复通常能保持 cache 热 |
| 避免刚好 `300s` | 既可能丢 cache，又没有换来足够长的等待收益 |
| 活跃等待用 `60s-270s` | 适合刚启动的 build、短轮询、马上会变的状态 |
| 空闲等待用 `1200s-1800s` | 20-30 分钟，减少无意义 cache miss |
| `prompt` 是下次恢复时注入的用户输入 | 可以是 `/change-orchestrator 继续 ...` 这种 skill continuation |
| teammate 主动消息可提前唤醒 | 定时回查是兜底；`SendMessage` 到 team-lead 时可以更早恢复 leader |
| `reason` 必填 | 给用户和 telemetry 看为什么这个节奏合理 |

放到 Agent Teams 的视角，`ScheduleWakeup` 是 team lead 的“回查闹钟/兜底恢复”，不是团队通信协议：

```text
Team lead / orchestrator
  │
  ├─ TeamCreate / TaskCreate / Agent / SendMessage 分配工作
  ├─ 派出长跑 reviewer / worker
  │
  ├─ 现在没有更多可执行动作
  │
  ▼
ScheduleWakeup(
  delaySeconds: 1800,
  prompt: "/change-orchestrator 继续 feature-x,reviewer-1 进度检查",
  reason: "checking reviewer-1 progress after 30 minutes"
)
  │
  ▼
两条路径先到先恢复
  │
  ├─ 定时到点：harness 注入 prompt，leader 回查 TaskList / reviewer 状态
  └─ teammate 提前 SendMessage：harness 立即唤醒 leader 处理消息
```

因此最新的完整心智模型应该拆成三层：

| 层级 | 工具 | 作用 |
|------|------|------|
| 团队组织层 | `TeamCreate` / `TeamDelete` | 建立和销毁团队上下文 |
| 团队协作层 | `Task*` / `Agent(name, team_name)` / `SendMessage` | 分工、执行、通信、汇报 |
| leader 回查层 | `ScheduleWakeup` | leader 派出长跑 teammate 后，安排外部定时恢复作为兜底 |
| 事件等待层 | `Monitor` / background Bash / `CronCreate` | 等具体进程、日志、CI 或日历型任务 |

一句话：`ScheduleWakeup` 解决的是 **team lead 什么时候被 harness 重新拉起来回查**；`SendMessage` 解决的是 **agent 和 teammate 之间如何说话，并且可能提前唤醒 leader**。两者配合起来，才形成“30 分钟兜底 + 主动汇报即时唤醒”的真实 Agent Team 工作流。

---

## 十四、代入 agent：我会怎么真实工作

如果我是日志里的那个 Claude Code agent，我不会把 `ScheduleWakeup` 限定在 `/loop`。在真实 Agent Team leader 流里，它是派出长跑 teammate 后的回查兜底：

| 当前处境 | 我会怎么做 | 不会怎么做 |
|----------|------------|------------|
| 用户只是普通提问或简单修改 | 直接回答或直接改代码 | 不调用 `ScheduleWakeup` |
| 用户要求一次性完成复杂任务 | `TaskCreate` 跟踪，必要时 `TeamCreate` + teammate | 不为短任务安排回查 |
| 我启动了 foreground agent 且需要结果 | 等 Agent tool result | 不用 `ScheduleWakeup`，因为这不是 loop pacing |
| 我启动了 background agent | 继续做别的，等自动完成通知 | 不主动轮询，不用 `ScheduleWakeup` 查输出 |
| 我派出长跑 teammate / reviewer | `ScheduleWakeup` 安排 10-30 分钟后回查，同时等 `SendMessage` 提前唤醒 | 不用 `sleep` / Bash 轮询烧上下文 |
| 我在 `/loop` 无 interval 的动态循环里 | 当前 turn 做能做的事，然后 `ScheduleWakeup` 安排下一次 | 不把它当作 teammate 通信 |
| 我需要等具体事件出现 | 用 `Monitor` 或 Bash `run_in_background` 等事件通知 | 不用固定时间 wakeup 硬查 |
| 用户要未来某个时间提醒/重复任务 | 用 `CronCreate` | 不用 `ScheduleWakeup`，它只服务当前 loop |

真实执行时，我的内循环会长这样：

```text
收到一次 turn
  │
  ├─ 如果有用户新指令：先处理用户指令
  │
  ├─ 如果有 teammate / background agent 通知：读取并整合
  │
  ├─ 如果有任务板：TaskList / TaskGet 看最新状态
  │
  ├─ 如果有明确下一步：执行 / SendMessage / TaskUpdate
  │
  ├─ 如果所有事都完成：总结，不再唤醒
  │
  └─ 如果派出了长跑 teammate 或需要稍后回查：
       ScheduleWakeup(delaySeconds, samePrompt, reason)
```

在 Agent Teams 场景里，我会这样组合：

```text
TeamCreate
TaskCreate 拆任务
Agent(name, team_name) spawn teammate
TaskUpdate(owner) 改任务板
SendMessage(to:name) 真正通知 teammate

如果当前 turn 没有更多动作：
  - teammate 主动 SendMessage：立即唤醒 leader 处理
  - teammate 没主动汇报：ScheduleWakeup 到点兜底回查 TaskList / 状态
```

`ScheduleWakeup` 的 delay 我会按日志 prompt 这样选：

| 等待对象 | delay 选择 | reason 写法 |
|----------|------------|-------------|
| 刚启动的短测试/短构建 | `120-270` 秒 | `checking short test run while prompt cache is warm` |
| 预计 8-10 分钟的构建 | 先 `270` 秒，下一轮再判断 | `checking long build before cache window expires` |
| 没有明确事件，只是持续 babysit | `1200-1800` 秒 | `idle loop check with no near-term signal` |
| 用户说“过一小时看看” | 接近 `3600` 秒 | `checking requested status after about one hour` |
| 正好想写 `300` 秒 | 改成 `270` 或 `1200+` | 避免 5 分钟 cache TTL 边界 |

所以，站在 agent 的真实工作角度：

1. `TaskUpdate(owner)` 是“任务板状态变化”。
2. `SendMessage(to:name)` 是“让 teammate 真正收到话”。
3. `ScheduleWakeup(...)` 是“我这个 team lead/loop agent 什么时候回来继续看局面”。

这三个不是互相替代，而是三个不同层面的动作。
