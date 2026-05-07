# Agent Loop 流程图

> 分析版本: 2026-04-29
> 涉及文件: `src/query.ts`
> 重点对象: `query()` / `queryLoop()` / `State.transition`

---

## 一、总览

`query()` 是外层 generator：它调用 `queryLoop()`，等 loop 正常返回后把已经消费的命令标记为 `completed`。

真正的 agent loop 在 `queryLoop()` 内部。它用一个 `while (true)` 反复执行：

1. 读取并投影当前上下文
2. 做 tool result budget / snip / microcompact / context collapse / autocompact
3. 调模型流式响应
4. 根据模型输出判断是否需要工具 follow-up
5. 没有工具调用时，处理恢复、Stop Hook、Token Budget，然后终止或重试
6. 有工具调用时，执行工具、注入附件、检查最大轮次，然后进入下一轮

`State.transition` 只记录“上一轮为什么继续”，用于测试和防止某些恢复路径无限循环；最终退出原因由 `Terminal.reason` 返回。

---

## 二、Agent Loop ASCII 图

```text
                         +----------------------+
                         | 用户输入 / 上下文     |
                         +----------+-----------+
                                    |
                                    v
                         +----------------------+
                         | query()              |
                         +----------+-----------+
                                    |
                                    v
                         +----------------------+
                         | 0. init State        |
                         +----------+-----------+
                                    |
                                    v
  retry states return    +----------------------+
  to this loop head --->  | while(true)          |
                         +----------+-----------+
                                    |
                                    v
                         +----------------------+
                         | preProcess           |
                         | 1 tool result budget |
                         | 2 snip               |
                         | 3 microcompact       |
                         | 4 context collapse   |
                         | 5 autocompact        |
                         +----------+-----------+
                                    |
                                    +==> END 1.blocking_limit
                                    |
                                    v
                         +----------------------+
                         | callModel            |
                         | stream response      |
                         +----------+-----------+
                                    |
                                    +--> fallback: retry callModel
                                    +==> END 2.image_error
                                    +==> END 3.model_error
                                    +==> ABORT 4.aborted_streaming
                                    |
                                    v
                         +----------------------+
                         | needsFollowUp ?      |
                         +-----+-----------+----+
                               | no        | yes
                               |           |
                               v           v
                    +----------------+   +----------------------+
                    | 无工具调用      |   | 工具调用              |
                    +-------+--------+   +----------+-----------+
                            |                       |
                            |                       +==> ABORT 16.aborted_tools
                            |                       +==> ABORT 17.hook_stopped
                            |                       |
                            |                       v
                            |            +----------------------+
                            |            | 注入附件 / 摘要       |
                            |            | refresh tools        |
                            |            +----------+-----------+
                            |                       |
                            |                       +==> END 18.max_turns
                            |                       +--> RETRY 19.next_turn
                            |
                            v
                    +----------------------+
                    | 结束 / 恢复判断       |
                    +----------+-----------+
                               |
                               +-- prompt too long
                               |   +--> RETRY 5.collapse_drain_retry
                               |   +--> RETRY 6.reactive_compact_retry
                               |   +==> END 7/8.prompt_too_long
                               |
                               +-- max output tokens
                               |   +--> RETRY 9.escalate_to_64k
                               |   +--> RETRY 10.recovery_retry
                               |
                               +-- API error
                               |   +==> DONE 11.completed
                               |
                               +-- stop hook
                               |   +==> ABORT 12.stop_hook_prevented
                               |   +--> RETRY 13.stop_hook_blocking
                               |
                               +-- token budget
                               |   +--> RETRY 14.budget_continuation
                               |
                               +==> DONE 15.completed
```

---

## 三、状态编号对照

| 编号 | 状态 / reason | 类型 | 代码含义 |
|---:|---|---|---|
| 0 | `undefined` | 初始 | 第一轮进入 `while (true)` 前，`state.transition` 为空 |
| 1 | `blocking_limit` | 终止 | proactive 阶段发现上下文已到硬阻断上限，直接返回 |
| 2 | `image_error` | 终止 | 图片大小或缩放错误；或媒体错误 reactive compact 后仍失败 |
| 3 | `model_error` | 终止 | 模型调用/运行时抛出未处理异常 |
| 4 | `aborted_streaming` | 终止 | 模型流式响应期间用户中断 |
| 5 | `collapse_drain_retry` | 继续 | prompt too long 后先 drain context collapse，再重试 |
| 6 | `reactive_compact_retry` | 继续 | prompt too long 或媒体错误触发 reactive compact 后重试 |
| 7/8 | `prompt_too_long` | 终止 | prompt 过长，collapse / reactive compact 均无法恢复 |
| 9 | `max_output_tokens_escalate` | 继续 | 首次撞输出上限时把 `maxOutputTokensOverride` 提升到 64k |
| 10 | `max_output_tokens_recovery` | 继续 | 注入 “继续输出” meta message，最多恢复 3 次 |
| 11 | `completed` after API error | 终止 | 最后一条是 API 错误，如限流、鉴权、权限等；跳过 Stop Hook |
| 12 | `stop_hook_prevented` | 终止 | Stop Hook 明确阻止继续 |
| 13 | `stop_hook_blocking` | 继续 | Stop Hook 返回 blocking error，作为消息注入后重试 |
| 14 | `token_budget_continuation` | 继续 | Token Budget 判断还应继续，注入 nudge 后重试 |
| 15 | `completed` | 终止 | 无工具调用、无错误、Hook 通过，正常结束 |
| 16 | `aborted_tools` | 终止 | 工具执行期间用户中断 |
| 17 | `hook_stopped` | 终止 | 工具/Hook 附件要求停止 continuation |
| 18 | `max_turns` | 终止 | 工具调用后下一轮会超过 `maxTurns` |
| 19 | `next_turn` | 继续 | 工具结果、附件、上下文刷新后进入下一轮模型调用 |

---

## 四、关键判断点

### `needsFollowUp`

`needsFollowUp` 是 agent loop 的主分叉。

- 流式响应中只要出现 `tool_use` block，就设为 `true`
- `true`：执行工具，拿到 tool results 后继续下一轮
- `false`：不再执行工具，进入恢复 / Stop Hook / Token Budget / 完成判断

### `state.transition`

`transition` 不是终止原因，而是“上一轮为什么继续”。典型用途：

- `collapse_drain_retry` 防止 413 后反复 drain
- `reactive_compact_retry` 标记已经做过 reactive compact
- `max_output_tokens_recovery` 记录恢复次数
- `stop_hook_blocking` 表示 hook error 已注入，下一轮让模型处理
- `next_turn` 表示工具调用的正常递归轮次

### 工具调用之后再拼接消息

工具路径会先完整执行当前批次工具，再把 `toolResults` 和附件统一追加到下一轮消息里。这样可以避免 `tool_result` 和普通 user message 交错插入，导致 API 校验失败。

---

## 五、压缩与恢复顺序

每轮模型调用前的上下文处理顺序：

```text
原始 messages
  -> getMessagesAfterCompactBoundary
  -> applyToolResultBudget              # 1. 工具结果专项预算
  -> snip compact                       # 2. 删除式中段裁剪
  -> microcompact                       # 3. 旧 tool_result 轻量清理
  -> context collapse projection        # 4. 分段归档投影
  -> autocompact                        # 5. 重型兜底摘要
  -> callModel
```

模型返回 prompt-too-long 后的恢复顺序：

```text
withheld prompt-too-long
  -> contextCollapse.recoverFromOverflow()
  -> reactiveCompact.tryReactiveCompact()
  -> 失败后暴露错误并 Terminal: prompt_too_long
```

输出 token 受限后的恢复顺序：

```text
withheld max_output_tokens
  -> 可用时升级 max output 到 64k
  -> 仍失败则注入恢复 meta message
  -> 最多 3 次
  -> 失败后暴露错误，作为 API error completed
```
