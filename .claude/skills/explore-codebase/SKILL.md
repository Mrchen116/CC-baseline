---
name: explore-codebase
description: "Deep-dive learning mode for reading and understanding a codebase. Use when the user asks 'how does X work', 'explain the mechanism of Y', 'walk me through Z', or any question that requires tracing data flow across multiple files. Outputs: sequence diagram first, then code map, then focused Q&A with tables."
---

# Explore Codebase

Read a repository to learn how something works. Sequence diagram → code map → focused answers.

## Trigger

User asks anything that requires understanding a mechanism across multiple files:
- "给我讲下 X 是怎么工作的"
- "画个时序图"
- "Y 的数据流是什么"
- "Z 的写入/读取逻辑在哪"
- 任何涉及 "怎么从文件中恢复/重建/加载" 的问题

## Core Workflow

```
Question → [Read key files] → [Sequence diagram] → [Code map] → [Focused Q&A loop]
```

---

## Step 1: Read Key Files

根据用户问题，定位核心代码路径。优先读：

1. **入口文件** — command handler / API endpoint / CLI parser
2. **核心逻辑文件** — 数据转换、状态管理、持久化层
3. **类型定义** — 理解数据结构

> 如果用户给了笔记/文档文件（如 `笔记/xxx.md`），先读它获取高层 overview，再读代码验证。

---

## Step 2: Sequence Diagram

**必须**画一个 ASCII 时序图。格式：

```
┌─────────┐     ┌──────────────┐     ┌─────────────────┐
│  Actor  │     │   Module A   │     │    Module B     │
└────┬────┘     └──────┬───────┘     └────────┬────────┘
     │                 │                      │
     │  action         │                      │
     │────────────────>│                      │
     │                 │  internal call       │
     │                 │─────────────────────>│
     │                 │                      │
     │  response       │                      │
     │<────────────────│                      │
```

**画图原则：**
- 横向是模块/角色，纵向是时间
- 标注**数据转换的关键节点**（如 "Lite → Full"、"deserialize"、"build chain"）
- 不要画到函数级，画到**模块级 + 关键操作名**

---

## Step 3: Code Map

时序图之后，**立即**给一张完整的代码地图表。格式：

| 阶段 | 文件 | 函数/类 | 说明 |
|------|------|--------|------|
| 入口 | `src/commands/resume/resume.tsx:224` | `call` | `/resume` slash command 入口 |
| 列表加载 | `src/utils/sessionStorage.ts:4074` | `loadSameRepoMessageLogs()` | 加载同仓库会话列表 |
| ... | ... | ... | ... |

**表格必须包含：**
- **阶段名** — 用中文，简短（如"反序列化"、"状态重建"）
- **文件路径:行号** — 精确到行，用户可直接跳转
- **函数/类名** — 精确的符号名
- **一句话说明** — 这个函数在这个阶段做了什么

如果阶段多，按阶段分组，每组一个表格。

---

## Step 4: Focused Q&A Loop

用户会基于时序图和代码地图继续追问。回答格式：

### 简短文字（1-3 句）

直接回答核心问题，不要铺垫。

### 多用表格

任何涉及"有哪些时机/场景/字段/选项"的问题，**必须用表格**。

表格必须包含文件位置列：

| 时机 | 文件位置 | 说明 |
|------|---------|------|
| Session 退出 | `sessionStorage.ts:458` | cleanup handler 刷元数据到尾部 |
| Compaction 后 | `compact.ts:713` | 防止元数据被挤出 tail 窗口 |

### 如果问题是"为什么有多个 X"

回答结构：
1. **机制解释**（1 句）— JSONL 是 append-only
2. **写入时机表** — 什么时候会追加新条目
3. **读取策略** — 怎么处理多个条目（last-wins / 合并 / 忽略）

### 如果问题是"除了 A 还在哪里消费"

回答结构：
1. **搜索** — grep 整个 src 目录
2. **表格** — 每个消费点一行

| 消费方 | 文件位置 | 用途 |
|-------|---------|------|
| `/resume` picker | `sessionStorage.ts:4762` | 展示"最近做了什么" |
| SDK listSessions | `listSessionsImpl.ts:117` | 生成 `SessionInfo.summary` |

---

## Tone & Style

- **不给模糊答案。** 如果不知道具体行号，去 grep，不要猜。
- **不啰嗦。** 时序图和代码地图已经给了全景，文字回答只补充细节。
- **中文为主。** 用户用中文问，用中文答。代码符号、文件名保持英文。
- **用户说"我没法找"时** — 立刻意识到是代码地图不够精确，补充文件路径和行号。

## Example Output Flow

**User**: `/resume` 怎么从文件中重建历史记录？画个时序图

**Assistant**:
1. 读 `src/commands/resume/resume.tsx`、`src/utils/sessionStorage.ts`、`src/utils/conversationRecovery.ts`
2. 输出时序图（模块级，4 个阶段）
3. 输出代码地图表（5-8 行，精确到行号）

**User**: 你没给我代码入口

**Assistant**:
1. 补充完整代码地图（按阶段分组，每个函数带文件:行号）

**User**: lastPrompt 除了 resume 还什么时候消费？

**Assistant**:
1. grep 搜索 `lastPrompt` 消费点
2. 表格输出，每行带文件位置

---

## Anti-patterns

❌ 不给时序图，直接丢代码片段  
❌ 代码地图只有文件名，没有行号  
❌ 回答用长段落，不用表格  
❌ 说"大概在 X 文件里"而不去确认  
❌ 把内部思考过程输出给用户
