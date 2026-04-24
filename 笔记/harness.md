# Harness / 核验相关笔记

- 产品里已有类似 **GAN** 的对抗思路：一方生成/改代码，另一方专门「挑错」验收。工程落点是内置 **[verification agent](../src/tools/AgentTool/built-in/verificationAgent.ts)**（`subagent_type="verification"`），对生成侧产出做独立核验（构建、测试、对抗探测、`VERDICT: PASS/FAIL/PARTIAL` 等）。该能力是否对用户开放由远程特性 **`tengu_hive_evidence`**（GrowthBook）做 **A/B**，并与编译期 **`VERIFICATION_AGENT`** 一起门控；注册与提示注入见 [builtInAgents.ts](../src/tools/AgentTool/builtInAgents.ts)、[prompts.ts 中 session guidance](../src/constants/prompts.ts)（约 390–395 行）。
- Anthropic 工程博客对 harness 设计有系统阐述（长时运行应用等语境）：[Harness design for long-running apps](https://www.anthropic.com/engineering/harness-design-long-running-apps)。
- **`feature(TOKEN_BUDGET)`** 可以看作一种外层 harness：用户在输入里指定 token 目标后，系统一方面通过 [prompts.ts](../src/constants/prompts.ts) 和 [attachments.ts](../src/utils/attachments.ts) 把当前 `turn/session/budget` 进度暴露给模型，另一方面在 [query.ts](../src/query.ts) 每轮结束后检查是否达标；如果还没到预算，就自动插入一条 meta user nudge 继续跑，避免模型过早停下。
