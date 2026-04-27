# 延迟加载工具（Deferred Tools / Tool Search）

当 MCP 工具或内部标记 `shouldDefer` 的工具数量很多时，工具定义可能占掉大量上下文 token。延迟加载机制把这类工具从初始 API 请求中剔除，只保留名字列表；模型需要时通过 `ToolSearchTool` 按需解锁，从而突破工具数量限制。

---

## 1. 哪些工具会被延迟

判定函数 [`isDeferredTool()`](../src/tools/ToolSearchTool/prompt.ts#L62)：

```typescript
export function isDeferredTool(tool: Tool): boolean {
  if (tool.alwaysLoad === true) return false      // 白名单：永远直接加载
  if (tool.isMcp === true) return true            // MCP 工具默认延迟
  if (tool.name === TOOL_SEARCH_TOOL_NAME) return false  // ToolSearchTool 本身不延迟
  // 特殊例外：FORK_SUBAGENT 时的 AgentTool、KAIROS 时的 BriefTool/SendUserFileTool
  return tool.shouldDefer === true                // 内部标记延迟的工具
}
```

**内部标记 `shouldDefer: true` 的工具有 26 个**：
`ReadMcpResourceTool`、`TaskStopTool`、`CronListTool`、`CronDeleteTool`、`CronCreateTool`、`SendMessageTool`、`TaskCreateTool`、`RemoteTriggerTool`、`TaskListTool`、`TeamCreateTool`、`LSPTool`、`EnterPlanModeTool`、`ConfigTool`、`WebSearchTool`、`TaskUpdateTool`、`EnterWorktreeTool`、`WebFetchTool`、`ExitPlanModeV2Tool`、`AskUserQuestionTool`、`NotebookEditTool`、`TaskGetTool`、`ExitWorktreeTool`、`TodoWriteTool`、`TaskOutputTool`、`ListMcpResourcesTool`、`TeamDeleteTool`。

预计算延迟工具集合（避免每轮重复调用 `isDeferredTool`，它内部有 GrowthBook 查询）：
[`query.ts`](../src/query.ts) 中在过滤前一次性构建 `deferredToolNames` Set。

---

## 2. 是否启用工具搜索

[`isToolSearchEnabled()`](../src/utils/toolSearch.ts#L385) 做最终判断，条件：
- 模型支持 `tool_reference`（Haiku 不支持）
- `ToolSearchTool` 在可用工具列表中（没被 `disallowedTools` 禁掉）
- 工具搜索模式不是 `standard`（由 `ENABLE_TOOL_SEARCH` 环境变量控制）
- `tst-auto` 模式下，延迟工具描述字符数/Token 数超过阈值（默认上下文窗口的 10%）

乐观预检 [`isToolSearchEnabledOptimistic()`](../src/utils/toolSearch.ts#L270)：
- 用于决定是否在基础工具列表中保留 `ToolSearchTool`
- 当 `ENABLE_TOOL_SEARCH` 未设置且 `ANTHROPIC_BASE_URL` 指向第三方代理时自动禁用（代理通常不支持 `tool_reference`）

---

## 3. 动态过滤工具列表

每轮 API 请求前，在 [`claude.ts#L1168-L1186`](../src/services/api/claude.ts#L1168) 中过滤：

```typescript
const discoveredToolNames = extractDiscoveredToolNames(messages)

filteredTools = tools.filter(tool => {
  if (!deferredToolNames.has(tool.name)) return true      // 非延迟工具：始终保留
  if (toolMatchesName(tool, TOOL_SEARCH_TOOL_NAME)) return true  // ToolSearchTool：始终保留
  return discoveredToolNames.has(tool.name)                // 延迟工具：只有发现过的才保留
})
```

- **第一轮**：`discoveredToolNames` 为空，API 只收到非延迟工具 + ToolSearchTool
- **后续轮**：已发现的延迟工具被加入，模型可以直接调用

如果本轮没有任何延迟工具且没有正在连接的 MCP 服务器，工具搜索自动关闭（`useToolSearch = false`），避免不必要的开销。

---

## 4. API 请求构建：defer_loading 标记

对过滤后的每个工具调用 [`toolToAPISchema()`](../src/utils/api.ts#L119)：

```typescript
deferLoading: willDefer(tool)
// willDefer(t) = useToolSearch && deferredToolNames.has(t.name)
```

在 [`api.ts#L224-225`](../src/utils/api.ts#L224)：

```typescript
if (options.deferLoading) {
  schema.defer_loading = true
}
```

生成的 schema 仍包含完整的 `name`、`description`、`input_schema`，只是额外附加 `defer_loading: true`。Anthropic API 据此对模型隐藏 `input_schema`，模型知道工具存在但不知道参数怎么传。

**服务端内部推测（tool_reference 展开）**：

客户端在初始请求里就带了所有 deferred 工具的完整 schema，服务端持有这些信息。当它在 tool_result 里收到 `tool_reference` block 时，会将其展开为对应工具的完整 schema JSON 呈现给模型，类似：

```json
// 模型实际看到的 tool_result 内容（推测）
{
  "name": "WebSearchTool",
  "description": "...",
  "input_schema": { "type": "object", "properties": { ... } }
}
```

这样模型当轮就能读到完整参数定义，可以在同一轮后续调用该工具。

客户端的 `extractDiscoveredToolNames()` 在**下一轮**把已发现的工具加入 `filteredTools`（不带 `defer_loading`），是第二层保障——绕过展开机制，让模型直接从 tool list 里看到 schema，更稳定高效。两层机制配合：当轮靠服务端展开，下一轮靠客户端完整加载。

**Beta header**：
- 1P/Foundry: [`advanced-tool-use-2025-11-20`](../src/constants/betas.ts#L13)
- Vertex/Bedrock: [`tool-search-tool-2025-10-19`](../src/constants/betas.ts#L14)

Bedrock 不走 betas 数组，而是塞进 `extraBodyParams`。

---

## 5. 模型怎么知道有哪些延迟工具

### 5.1 `<available-deferred-tools>` 消息

当 `isDeferredToolsDeltaEnabled()` 为 false 时，在 [`claude.ts#L1372-1386`](../src/services/api/claude.ts#L1372) 中 prepend 一条用户消息：

```typescript
const deferredToolList = tools
  .filter(t => deferredToolNames.has(t.name))
  .map(formatDeferredToolLine)   // 返回 tool.name，只有名字
  .sort()
  .join('\n')

messagesForAPI = [
  createUserMessage({
    content: `<available-deferred-tools>\n${deferredToolList}\n</available-deferred-tools>`,
    isMeta: true,
  }),
  ...messagesForAPI,
]
```

模型从这里看到**所有延迟工具的名字列表**，但没有 description 或参数信息。

### 5.2 ToolSearchTool 的 prompt

[`getPrompt()`](../src/tools/ToolSearchTool/prompt.ts#L119) 返回的描述中明确告知：

> "Until fetched, only the name is known — there is no parameter schema, so the tool cannot be invoked."

模型由此理解：这些工具存在但不可调用，需要通过 `ToolSearchTool` 获取完整定义。

---

## 6. ToolSearchTool：模糊匹配

### 6.1 查询语法

- `select:ToolA,ToolB` — 直接按名字选择（精确匹配，不走搜索）
- `slack send` — 关键词搜索，空格分隔
- `+slack send` — `+` 前缀表示"必需"，结果必须包含 `slack`

### 6.2 搜索打分 [`searchToolsWithKeywords()`](../src/tools/ToolSearchTool/ToolSearchTool.ts#L186)

**工具名解析** [`parseToolName()`](../src/tools/ToolSearchTool/ToolSearchTool.ts#L132)：
- MCP: `mcp__slack__send_message` → `['slack', 'send', 'message']`
- 普通: `WebSearchTool` → `['web', 'search', 'tool']`

**打分权重**：

| 匹配类型 | MCP 分数 | 普通工具分数 |
|---------|---------|-------------|
| 名字部分精确包含查询词 | 12 | 10 |
| 名字部分子串包含 | 6 | 5 |
| 完整名字包含（fallback） | 3 | 3 |
| `searchHint` 中单词边界匹配 | 4 | 4 |
| 描述中单词边界匹配 | 2 | 2 |

`searchHint` 是工具上 curate 的短描述，信号比完整 prompt 更强。结果按分降序取前 `max_results`（默认 5）。

### 6.3 返回格式

[`mapToolResultToToolResultBlockParam()`](../src/tools/ToolSearchTool/ToolSearchTool.ts#L444)：

```typescript
{
  type: 'tool_result',
  tool_use_id: toolUseID,
  content: [
    { type: 'tool_reference', tool_name: 'mcp__slack__send_message' },
    { type: 'tool_reference', tool_name: 'mcp__slack__post_message' },
  ]
}
```

匹配的工具名以 `tool_reference` beta 内容块形式返回。

---

## 7. 发现机制：extractDiscoveredToolNames

[`extractDiscoveredToolNames()`](../src/utils/toolSearch.ts#L545) 扫描消息历史：

```typescript
for (const msg of messages) {
  // 1. 压缩边界快照
  if (msg.type === 'system' && msg.subtype === 'compact_boundary') {
    const carried = msg.compactMetadata?.preCompactDiscoveredTools
    for (const name of carried) discoveredTools.add(name)
    continue
  }

  // 2. user 消息中的 tool_result 块
  if (msg.type !== 'user') continue
  const content = msg.message?.content
  for (const block of content) {
    if (block.type === 'tool_result' && Array.isArray(block.content)) {
      for (const item of block.content) {
        if (item.type === 'tool_reference' && typeof item.tool_name === 'string') {
          discoveredTools.add(item.tool_name)
        }
      }
    }
  }
}
```

两个数据来源：
1. **ToolSearchTool 历史调用结果**：`tool_result.content` 中的 `tool_reference` 块
2. **压缩边界快照**：`compactMetadata.preCompactDiscoveredTools`，防止压缩后丢失发现状态

提取出的 `Set<string>` 就是下一轮过滤延迟工具时的白名单。

---

## 8. 压缩边界持久化

当对话触发 compaction 时，已发现的工具集合被保存到压缩边界消息中：

```typescript
// compact boundary 上的快照
msg.compactMetadata.preCompactDiscoveredTools = [...discoveredToolNames]
```

这样即使长对话被压缩，工具发现状态也不会丢失。`extractDiscoveredToolNames` 读取边界时会把这些名字重新加入已发现集合。

---

## 9. 完整流程图

```
[初始化]
  ↓
判定延迟工具集合（isDeferredTool）
  ↓
构建 API 请求
  ├─ filteredTools：非延迟工具 + ToolSearchTool + 已发现的延迟工具
  ├─ 对延迟工具 schema 附加 defer_loading: true
  ├─ prepend <available-deferred-tools>（只列名字）
  └─ 加 tool search beta header
  ↓
模型收到 → 看到延迟工具名字但无法调用（缺 schema）
  ↓
模型调用 ToolSearchTool("slack send")
  ↓
ToolSearchTool 内部模糊匹配 → 返回 tool_reference 块
  ↓
消息历史中留下 tool_reference
  ↓
下一轮 extractDiscoveredToolNames() 提取
  ↓
该延迟工具加入 filteredTools → 模型现在可以直接调用
```

---

## 10. 为什么其他厂商模型无法复现这套机制

即使在 prompt 里完整描述一个工具的名字和 schema，其他厂商模型也不会调用它。这不是"模型不知道"，而是更底层的约束。

**两层限制叠加：**

**第一层：训练数据分布分离。** 模型生成 tool_use 输出的触发条件是"context 开头正式 tool 定义里存在该工具"，与"prompt 文字描述"是两条完全独立的数据分布。模型没有见过"从 prompt 里读 schema 然后生成 function call"的训练样本，这两条路径在训练时就是分离的。

**第二层：解码层硬约束。** 大多数厂商的 function calling 通过 special token（如 DeepSeek 的 `<｜tool▁calls▁begin｜>`）或 constrained decoding 实现，解码时校验 tool name 必须在注册表里，否则直接截断。prompt 根本触达不到这一层。

**实验验证：** Claude 自身也有同样的约束。尝试调用一个完全不在 tool list 里的工具（如 `DeleteDatabaseTool`），无法产生对应的 tool_use 输出块——不是"选择不做"，而是根本无法生成那个结构化输出。deferred tools 虽然名字可见（出现在 system reminder 里），但不先通过 ToolSearchTool 解锁，直接调用同样失败。

**结论：** deferred tools 机制是 Anthropic 独占的，依赖三个缺一不可的条件：`defer_loading` API 字段（服务端协议扩展）、`tool_reference` beta content type（服务端展开逻辑）、模型针对这套机制的专项训练。纯靠 prompt 工程在其他模型上无法复现。

---

## 11. 关键文件速查

| 文件 | 作用 |
|------|------|
| [`src/tools/ToolSearchTool/prompt.ts`](../src/tools/ToolSearchTool/prompt.ts) | `isDeferredTool()`、prompt 模板、header 常量 |
| [`src/utils/toolSearch.ts`](../src/utils/toolSearch.ts) | `extractDiscoveredToolNames()`、`isToolSearchEnabled()`、模式判定 |
| [`src/services/api/claude.ts`](../src/services/api/claude.ts) | 动态过滤逻辑、`defer_loading` 标记、`<available-deferred-tools>` 注入 |
| [`src/utils/api.ts`](../src/utils/api.ts) | `toolToAPISchema()`，生成带 `defer_loading` 的 schema |
| [`src/tools/ToolSearchTool/ToolSearchTool.ts`](../src/tools/ToolSearchTool/ToolSearchTool.ts) | 模糊匹配搜索、返回 `tool_reference` 块 |
| [`src/constants/betas.ts`](../src/constants/betas.ts) | `advanced-tool-use-2025-11-20`、`tool-search-tool-2025-10-19` |
