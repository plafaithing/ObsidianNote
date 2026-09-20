# RESOURCES — Harness Engineering

> 知识来源清单。课程论断应能追回到这里的资源。
> 标注：`[主样本]` / `[对比补充]` / `[方法论一手]` / `[拆解参考]`
> ⚠️ 外部 URL 经 web-search 检索获得，深读时以官方站点实际内容为准；本环境 WebSearch 不可用，用 web-search-prime / web-reader 核实。

## 方法论一手（Anthropic 官方，高可信）
1. `[方法论一手]` **Building Effective AI Agents** — workflow vs agent、agentic loop、编排模式（routing / parallelization / orchestrator-workers / evaluator-optimizer）。**编排层总纲**。
   https://www.anthropic.com/engineering/building-effective-agents
2. `[方法论一手]` **Effective context engineering for AI agents** — 上下文管理（窗口、压缩、记忆注入）。**上下文层**。
   https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
3. `[方法论一手]` **Writing effective tools for AI agents** — 工具定义、工具循环、工具设计原则。**工具层**。
   https://www.anthropic.com/engineering/writing-tools-for-agents
4. `[方法论一手]` **Prompting best practices（Claude Platform Docs）** — 系统提示词写法：清晰直接 / XML 结构化 / 给例子 / 给角色 + agentic 场景下调控各层的 prompt 片段。**系统提示词层**。
   https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices

## 主样本：Claude Agent SDK（Claude Code 开源内核）
4. `[主样本]` **Agent SDK overview 文档** — "the same tools, agent loop, and context management that power Claude Code"。入口。
   https://code.claude.com/docs/en/agent-sdk/overview
5. `[主样本]` **claude-agent-sdk-typescript** — TS 版源码，可读可跑。
   https://github.com/anthropics/claude-agent-sdk-typescript
6. `[主样本]` **claude-agent-sdk-demos** — 多种用法 demo。
   https://github.com/anthropics/claude-agent-sdk-demos
7. `[主样本]` **anthropic-sdk-python** — 底层 SDK（Agent SDK Python 版基座）。
   https://github.com/anthropics/anthropic-sdk-python
7b. `[主样本]` **Agent SDK custom-tools 文档** — tool() 四部分、Zod schema、createSdkMcpServer、全名 mcp__server__tool、readOnlyHint 注解、可用性 vs 权限两层、isError 错误处理、structuredContent。**L08 工具实现核心来源**。
    https://code.claude.com/docs/en/agent-sdk/custom-tools
7c. `[主样本]` **Agent SDK agent-loop 文档** — loop 五步、turns/messages、内置工具=CC 同款、并行/串行调度、tool search 延迟加载、工具被拒作为 tool_result、context 窗口与压缩。**L08 + L09 共用**。
    https://code.claude.com/docs/en/agent-sdk/agent-loop

## 主样本：Claude Code 运行时 / 配置面
8. `[主样本]` **Claude Code system prompts 仓库（Piebald-AI）** — 各版本 system prompt + token 计数，**读系统提示词层**。
   https://github.com/Piebald-AI/claude-code-system-prompts
9. `[主样本]` **Claude Code 官方文档** — `settings.json` / hooks / 权限 / subagents / `CLAUDE.md` 机制。
   https://docs.claude.com/en/docs/claude-code

## 拆解参考（第三方深读，中可信，需核实）
10. `[拆解参考]` **Claude Code Harness and Environment Engineering** — 实现层指南，覆盖 in-process harness layer（`settings.json`…）。
    https://hidekazu-konishi.com/entry/claude_code_harness_and_environment_engineering_guide.html
11. `[拆解参考]` **Inside the Agent Harness: How Codex and Claude Code actually work** — Codex vs CC 对比深读。
    https://medium.com/jonathans-musings/inside-the-agent-harness-how-codex-and-claude-code-actually-work-63593e26c176
12. `[拆解参考]` **Found from Claude Code: Chapter 1 — The Harness Paradigm** — harness 范式入门，适合基础薄弱起步。
    https://kenhuangus.substack.com/p/found-from-claude-code-chapter-1
13. `[拆解参考]` **What I learned by rebuilding a CC-style harness on raw SDK**（reddit） — 用裸 SDK 重建 CC harness 的实战心得。
    https://www.reddit.com/r/ClaudeCode/comments/1ueuh0h/

## 对比补充（第三方框架，待选型与核实）
14. `[对比补充]` **LangGraph** — 状态图式编排，与 Agent SDK 的循环式对比。待核实入口与版本。
15. `[对比补充]` **AutoGen / 其他** — 多 agent 对话式编排。待核实。

## 待补充
- 深读以上资源后的笔记沉淀到 `reference/`
- 用户实际售前 / 定制案例
- 术语表（`reference/术语表.md`）：harness / agentic loop / tool use / context window / orchestrator-workers / …
