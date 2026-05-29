# 报告：Anthropic Claude Code `cache_edits` 最可能做法推测 by chatgpt in web

## 0. 结论先行

我现在对 Claude Code `cache_edits` 的最可能判断是：

> **它不是把 `A B C` 重新变成真正的 `A C`，也不是重算 `C`。更可能是：Claude Code 在客户端保持原始消息结构不变，把旧 `tool_result` 标上 `cache_reference`；当这些工具结果过期或太多时，通过 `cache_edits.delete(reference)` 告诉 Anthropic 服务端从“已缓存前缀的有效视图”里删除这些工具结果。服务端后续让模型不再直接读取被删的 `B`，但保留后面 `C` 已经吸收过 `B` 的工作状态。**

用模型推理抽象就是：

```text
原始上下文：
A B C

B = 大量旧 tool_result
C = assistant 基于 B 形成的后续判断、计划、代码修改状态

cache_edits 后的有效状态更像：
A [hole] C_from_ABC

而不是：
A C_from_AC
```

所以 `cache_edits` 的目标不是“消除 B 的影响”，而是：

```text
删除 B 的直接可见原始上下文
保留 C 对 B 的消化结果
同时尽量不破坏 prompt cache
```

这点对 coding agent 非常合理。

---

## 1. 已知事实：官方公开能力 vs `cache_edits`

Anthropic 官方公开的 prompt caching 文档讲的是 `cache_control` / cache breakpoint：稳定内容放在 prompt 前面，cache 前缀按 `tools → system → messages` 顺序建立；官方文档也明确说 cache hit 需要 prompt segment 100% identical。公开文档没有把 `cache_edits` 描述成普通用户可用的标准 API 字段。([Claude Platform][1])

这意味着：普通 prompt cache 逻辑下，如果你把历史里的 `B` 删除，原始前缀从 `A B C` 变成 `A C`，cache 从删除点开始就会断。`cache_edits` 的价值正是绕过这个问题：不是客户端本地改写历史，而是把“删除某个已引用块”的意图交给 Anthropic first-party 服务端处理。

社区逆向分析里提到的三件套是：

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_abc",
  "cache_reference": "toolu_abc",
  "content": "..."
}
```

然后后续插入：

```json
{
  "type": "cache_edits",
  "edits": [
    {
      "type": "delete",
      "cache_reference": "toolu_abc"
    }
  ]
}
```

同时依赖内部 beta header，例如 `anthropic-beta: claude-code-20250219`；响应里可能出现 `cache_deleted_input_tokens`，表示释放/删除的缓存 token 数。这个信息来自 Reddit 逆向帖，不是官方文档，所以只能当作非官方证据。([Reddit][2])

---

## 2. Claude Code 里它处在哪个机制里

现在看起来 Claude Code 至少有几种不同层级的压缩/裁剪机制，不能混为一谈。

GitHub issue 里有用户基于源码路径整理出三个机制：time-based microcompact、cached microcompact、session memory compact。其中 **cached microcompact** 被描述为使用 `cache_edits` API 删除旧 tool results；time-based microcompact 则是旧工具结果内容被清空；session memory compact 则是更大粒度的会话记忆压缩。([GitHub][3])

第三方源码分析也给出类似判断：`CACHED_MICROCOMPACT` 会在 API 层排队 `cache_edits`，本地 messages 保持不变，目标是移除大工具输出但保留 cache prefix。([GitHub][4])

另一个源码摘录显示，Claude Code 会把 pending cache edits “pin” 到特定 user message 位置，并在后续请求中把之前 pin 过的 edits 重新发送，以保证 cache hit 的位置稳定。([Gist][5])

这个“pin 到原位置”的细节非常关键：它说明 `cache_edits` 不是一次性本地删除，而更像一个 **服务端 cache view 的 edit log**。后续请求仍然需要把同样的编辑脚本放在同样的位置，否则 cache key / cache view 可能不一致。

---

## 3. 最可能的端到端流程

我推测 Claude Code 的流程大致是这样：

### 3.1 初始阶段：正常写入可缓存前缀

Claude Code 正常构造 messages：

```text
system/tools
user message
assistant tool_use
user tool_result(B)
assistant response(C)
...
```

对于大工具结果，例如 `Bash`、`Read`、`Grep`、`Glob`、`WebSearch`、`Edit`、`Write` 等，Claude Code 在 API 层给对应 `tool_result` 加上 `cache_reference`。社区源码分析和博客都提到 microcompact 会针对这些高体积工具结果。([Stefano Straus][6])

此时服务端缓存的是完整前缀：

```text
A B C
```

这里 `C` 已经是基于 `B` 推理出来的状态，例如“我读完了日志，根因是 X；已经修改文件 Y；下一步要跑测试”。

### 3.2 触发阶段：判断哪些 tool result 可以删

Claude Code 的 cached microcompact 可能按数量、时间、上下文增长、工具类型等策略挑选旧工具结果。GitHub issue 里提到 cached microcompact 是 count-based trigger/keep threshold，配置来自 GrowthBook。([GitHub][3])

它最可能不是按 attention score 选 token，而是按 **结构化 block** 选：

```text
删除单位：整个 tool_result
引用键：tool_use_id / cache_reference
```

这和学术里的 H2O / SnapKV 那种 token-level eviction 不一样。Claude Code 更工程化：它知道哪些块是工具输出，知道哪些输出价值衰减快，知道 user message 不能随便删。

### 3.3 请求阶段：插入 `cache_edits`

当决定删除某些旧工具结果时，Claude Code 不直接把本地历史改成：

```text
A C
```

而是在某个 user message 中插入：

```text
cache_edits(delete B_reference)
```

并把这个 edit block pin 住，后续继续在同一位置发送。第三方源码摘录里能看到 `consumePendingCacheEdits()`、`getPinnedCacheEdits()`、`pinCacheEdits()` 这样的流程。([Gist][5])

### 3.4 服务端阶段：应用删除脚本

最可能的服务端语义是：

```text
已缓存原始前缀：
A B C

收到 cache_edits：
delete B_reference

服务端有效视图：
A [deleted B] C
```

这里要强调：**这不等于服务端把 raw prompt 重新变成 `A C` 后再做普通 cache lookup。**

如果它这么做，普通 exact-prefix cache 仍然会从 `B` 处断掉，和这个功能的动机冲突。官方 prompt caching 文档说 cache hit 要求 segment 100% identical，所以 `cache_edits` 如果存在，必然是在普通 raw-prefix matching 之上加了一层内部协议。([Claude Platform][1])

---

## 4. 从 KV cache 角度，最可能发生了什么

把上下文抽象成：

```text
A B C
```

完整 prefill 后得到：

```text
KV_A
KV_B
KV_C_from_ABC
```

`cache_edits.delete(B)` 之后，最可能不是：

```text
重新计算 KV_C_from_AC
```

而是：

```text
保留 KV_A
删除/屏蔽 KV_B
保留 KV_C_from_ABC
```

也就是未来 token attention 的可见集合变成：

```text
A + C_from_ABC
```

而不是：

```text
A + C_from_AC
```

这个差别很重要。`C_from_ABC` 已经吸收过 `B` 的信息。对 coding agent 来说，这往往是好事：旧工具结果 `B` 的原始细节不再占上下文，但后续 assistant 状态 `C` 仍然保留了“我看过 B 后得出的结论”。

### 4.1 为什么不是严格的 `A C`

Transformer 的 `C` 不是只由 `C` 自己决定。`C` 的 hidden state / K / V 是在它前面有 `A+B` 时算出来的。因此：

```text
KV_C_from_ABC ≠ KV_C_from_AC
```

如果要真正得到 `A C`，理论上至少要重算 `C`，或者做 RoPE 位置修正、局部重算、attention repair。那会显著增加服务端复杂度，且不符合 `cache_edits` 作为快速微压缩机制的定位。

### 4.2 位置编码最可能保持原位置

如果原始位置是：

```text
A: pos 0-99
B: pos 100-999
C: pos 1000-1099
```

删除 `B` 后，最可能不是把 `C` 重排成 pos 100-199，而是保留：

```text
A: pos 0-99
C: pos 1000-1099
```

后续新 token 也继续接在原始 logical length 后面：

```text
D: pos 1100
```

这就是：

```text
A [hole] C
```

而不是：

```text
A C
```

这个推测也和 KV compaction 论文里的“logical KV length 和 physical cache size 解耦”思想一致：物理 KV 条目可以减少，但逻辑长度仍然保留，以便后续 token 的位置/RoPE 相位保持一致。Fast KV Compaction 论文明确把这种解耦称作一个实用的 KV compaction 原语。([arXiv][7])

---

## 5. 和学术研究最接近的是哪类

### 5.1 最接近底层动作：post-prefill KV dropping

`post-prefill KV dropping` 不是一篇论文标题，而是一类方法。EMNLP 2024 的综述论文把 token dropping 分成 during prefill 和 after prefill；after prefill 的定义就是：先生成完整 KV cache，再从中移除不重要 token。论文还指出，这通常比 during-prefill dropping 效果更好，因为重要性判断能利用完整 attention 信息。

这和 Claude Code 的抽象最像：

```text
先完整处理 A B C
让 B 有机会影响 C
之后删掉 B 的直接 KV / cache block
```

H2O 是经典代表：它观察到少数 token 贡献了大部分 attention score，提出动态保留 recent tokens 和 heavy-hitter tokens 的 KV cache eviction 策略。

但 Claude Code 和 H2O 的差别也很明显：

```text
H2O：按 token attention 分数删
Claude Code：按 tool_result block / cache_reference 删
```

所以 Claude Code 更像 **结构化 post-prefill block dropping**，不是普通 token-level dropping。

### 5.2 最接近“B 被 C 吸收后删 B”：Breadcrumbs Reasoning

Breadcrumbs Reasoning 的做法是周期性用特殊 beacon token 计算过去 KV entries 的压缩表示，然后 evict 原来的 cache entries，只保留 beacon 的 KV。论文算法里明确写到：输入 compression token 后更新 cache，然后 drop 最近 token 的 KV entries，但保留 beacon entry。([arXiv][8])

这和 Claude Code 的精神很像：

```text
Breadcrumbs:
旧 reasoning tokens B
→ beacon token C 吸收 B
→ 删除 B 的 KV

Claude Code:
旧 tool_result B
→ 后续 assistant 状态 C 吸收 B
→ 删除 B 的 cache/KV
```

区别是：Breadcrumbs 需要模型训练出 compression beacon 能力；Claude Code 则更像使用已有 assistant 文本状态 + 服务端 cache edit。

### 5.3 长对话场景近亲：EpiCache

EpiCache 面向长多轮对话，指出 KV cache 会随 dialogue length 线性增长，并提出 episodic KV compression / episode-specific eviction。它还特别批评普通 post-prefill eviction 的峰值内存问题，以及 query-dependent eviction 不适合多轮未来问题。([arXiv][9])

这和 Claude Code 的长期 session 问题接近，但 EpiCache 不是针对 tool_result，也不是 cache-prefix-preserving edit log。它更像“长对话按 episode 管理 KV”的研究近亲。

### 5.4 为什么 Fast KV Compaction 不是最像

Fast KV Compaction 的动机接近：长上下文部署里需要 compaction，且文本摘要有损。但它的做法是构造短的 compact K/V 来替代原始大段 KV，目标是匹配 attention output 和 attention mass。([arXiv][10])

Claude Code `cache_edits` 更像：

```text
B 直接删掉
靠 C_from_ABC 保留状态
```

Fast KV Compaction 更像：

```text
B 压成 B'_latent
未来还能 attend 到 B'_latent
```

所以它更像未来高级版，不像当前 `cache_edits` 的最可能实现。

---

## 6. 为什么这个设计适合 Claude Code

Claude Code 的工作负载有几个特点：

1. **工具输出极长**：`Read`、`Grep`、`Bash`、`WebSearch` 输出动辄几千到几万 token。
2. **工具输出价值衰减快**：很多日志、grep 结果、测试输出只服务于当下一个判断。
3. **后续 assistant 状态已经消化了工具输出**：例如根因、改动文件、已跑测试、剩余问题。
4. **缓存经济性极重要**：普通删除会破坏 exact prefix cache，而官方 prompt caching 又要求 100% identical prompt segment 才能 hit。([Claude Platform][1])
5. **结构化工具块比自然语言历史更适合删**：用户消息、安全约束、任务目标不能随便删，但旧工具结果可以。

所以 `cache_edits` 的产品目标应该是：

```text
长 session 不爆上下文
旧工具输出不继续污染注意力
不破坏已经建立的 cache prefix
不强迫每次 full compact
让模型继续基于已形成的工作状态推进
```

这解释了为什么它主要针对 `tool_result`，而不是随意删除 user messages 或 assistant messages。

---

## 7. 最可能的服务端实现形态

我把可能性从高到低排一下。

### 方案 A：cache block 级逻辑删除 / attention mask

最可能。

服务端缓存 prefix 时，内部不是只有一个 raw token hash，而是还记录 block 边界和 `cache_reference`。收到 `cache_edits.delete(ref)` 后，服务端把对应 block 的 KV page 标记为 deleted，后续 attention 不再直接读取这些 K/V。

语义：

```text
未来 token 不 attend B
但可以 attend C_from_ABC
```

优点：快，符合 microcompact。
缺点：不是严格 `A C`，但这正是 agent compaction 想要的。

### 方案 B：删除 KV page，但保留 logical positions

也很可能。

物理上释放/跳过 `B` 的 KV page，但保留 sequence logical length。这样 `C` 和后续 token 的 RoPE/position 不重排，语义是 `A [hole] C`。

这和“physical cache size 减少，但 logical KV length 保留”的 KV compaction 思路一致。([arXiv][7])

### 方案 C：局部重算后缀 C

可能性较低。

如果要把上下文真的变成 `A C`，就要重算 `C` 或做复杂 correction。这个代价和复杂度较高，也不符合“删除旧工具结果但保 cache”的设计动机。

### 方案 D：只做计费/context accounting 删除，模型内部仍可见 B

可能性低到中等。

如果只是账面删除，但模型仍可 attend B，那能省计费/上下文显示，但不能减少真实 attention memory，也无法解释“旧 tool result 被 stripped 后模型无法直接回查原始材料”的用户观察。GitHub issue 中有人报告 agent 会基于 internalized summaries 自信回答，因为源材料被剥离。([GitHub][3])

---

## 8. 风险和副作用

### 8.1 上下文透明度问题

用户可能以为 Claude Code 仍然持有原始工具输出，但实际上旧 tool result 已经从有效上下文中删除。GitHub issue 的抱怨重点就是：这些机制静默运行，没有 UI 通知，也不触发常规 hooks。([GitHub][3])

### 8.2 “我看过”不等于“我还能引用”

删除 B 后，模型还可能记得 C 里总结过的结论，但不能可靠回查 B 的原始细节。

这对 coding agent 通常可接受，因为可以重新 `grep`、重新 `read`、重新跑测试。但对会计、法务、审计、医学这种证据驱动任务很危险。

### 8.3 C 的压缩质量决定成败

如果 C 写得好：

```text
B: 50000 token 测试日志
C: 失败测试、错误栈、根因、文件路径、修复计划
```

删 B 没问题。

如果 C 写得差：

```text
B: 50000 token 证据
C: “我看完了，没问题”
```

删 B 就会造成不可审计的幻觉风险。

### 8.4 兼容层很难复刻

OpenRouter、Bedrock、自建 Anthropic-compatible proxy 很可能不能支持这一点。因为它不是普通 API 字段，而是 Anthropic first-party 服务端对缓存结构的内部能力。Reddit 逆向帖也明确说这看起来不是公开 API、也不在 public beta headers 里。([Reddit][2])

---

## 9. 对自建 agent harness 的启发

如果你要在 OpenClaw / 自建 coding agent 里学这个设计，不应该直接假设兼容层支持 `cache_edits`。更实际的做法是模拟它的产品语义：

```text
1. 工具结果结构化存档
2. 后续 assistant 必须写显式状态摘要
3. 旧工具结果按工具类型、时间、数量、引用状态裁剪
4. 裁剪后保留 provenance：文件、命令、时间、路径、hash
5. 需要时重新拉取原始证据
```

也就是：

```text
Claude Code first-party:
删 B 的服务端 cache block，保 C_from_ABC

自建通用 harness:
把 B 外部化存储，C 显式摘要，必要时重新 retrieve B
```

不要依赖隐状态残留。要让 C 明文承载关键状态。

---

## 10. 最终判断

`cache_edits` 最可能不是一种“神奇无损 KV 变换”，而是一种 **结构化、服务端支持的 cached-prefix edit log**：

```text
客户端：
标记 tool_result → 生成 delete edits → pin edits

服务端：
命中原始 cached prefix → 应用 delete refs → 删除/屏蔽旧 tool_result 的直接可见 cache/KV → 保留后续状态

模型语义：
A [hole] C_from_ABC
```

它和学术界最接近的是：

```text
post-prefill KV dropping
+
structured block-level eviction
+
agent context microcompaction
```

但它的工程创新点在于：

```text
不是纯 token eviction
不是纯文本摘要
不是普通 prefix cache
而是在 first-party prompt cache 上做引用级删除
```

这就是为什么它很适合 Claude Code：coding agent 的长工具输出非常多，而后续 assistant 状态通常已经足够承载推进任务所需的信息。

---

## 遗留表

| 问题                     | 当前判断                                                       |
| ---------------------- | ---------------------------------------------------------- |
| `cache_edits` 是否公开 API | 目前没有看到官方公开文档支持                                             |
| 最可能的本质                 | 服务端 cached-prefix edit log / block-level cache deletion    |
| 模型看到的是不是严格 `A C`       | 不是，更像 `A [hole] C_from_ABC`                                |
| C 的位置编码是否重排            | 大概率不重排，继续使用原 logical position                              |
| 和学术最接近方向               | post-prefill KV dropping、Breadcrumbs Reasoning、EpiCache    |
| 最大风险                   | 用户以为原始工具结果还在，但模型只能依赖后续摘要/状态                                |
| 自建 harness 可复刻部分       | 工具结果结构化存档 + 显式状态摘要 + 外部检索；不能指望通用 provider 支持 `cache_edits` |

[1]: https://platform.claude.com/docs/en/build-with-claude/prompt-caching "Prompt caching - Claude API Docs"
[2]: https://www.reddit.com/r/ClaudeCode/comments/1s9qz8b/i_found_an_extremely_valuable_undocumented_api/ "I found an extremely valuable undocumented API feature in the Claude Code source called cache_edits : r/ClaudeCode"
[3]: https://github.com/anthropics/claude-code/issues/42542 "[BUG] Silent context degradation — tool results cleared without notification on 1M context sessions this issue documents three separate mechanisms (microcompact, cached microcompact, session memory compact) · Issue #42542 · anthropics/claude-code · GitHub"
[4]: https://github.com/carlosduplar/claude-code-optimizer/blob/master/docs/prompt-caching.md "claude-code-optimizer/docs/prompt-caching.md at master · carlosduplar/claude-code-optimizer · GitHub"
[5]: https://gist.github.com/Houstoten/144e4ae9c520a281551d0cb92c488e04 "Claude Code internals: AutoDream, Buddy companion, prompt cache economics, microcompact · GitHub"
[6]: https://straus.it/blog/claude-code-source-leak-anatomy/ "Anatomy of the Claude Code Source Leak: What 512K Lines of TypeScript Reveal | Stefano Straus"
[7]: https://arxiv.org/html/2602.16284v1?utm_source=chatgpt.com "Fast KV Compaction via Attention Matching"
[8]: https://arxiv.org/html/2510.13797v1 "Breadcrumbs Reasoning: Memory-Efficient Reasoning with Compression Beacons"
[9]: https://arxiv.org/html/2509.17396v1 "EpiCache: Episodic KV Cache Management for Long Conversational Question Answering"
[10]: https://arxiv.org/abs/2602.16284?utm_source=chatgpt.com "[2602.16284] Fast KV Compaction via Attention Matching"
