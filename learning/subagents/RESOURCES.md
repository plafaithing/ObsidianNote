# RESOURCES — 子 agent

> 知识来源清单。课程中的论断应能追回到这里的资源。
> 本环境 WebSearch 不可用，以下以**本地一手资料**为主，外部官方文档供用户自行查阅。

## 本地一手资料（高可信）

1. **superpowers:subagent-driven-development** — 子 agent 哲学（隔离上下文、fresh subagent per task）、SDD 工作流（派发-实现-两阶段 review）。
   路径：`~/.claude/plugins/cache/superpowers-marketplace/superpowers/5.1.0/skills/subagent-driven-development/`
2. **superpowers:dispatching-parallel-agents** — 并行派发：独立任务一个 agent 一个问题域、并发、review 整合、何时该/不该并行。同上插件目录。
3. **系统自带 Agent 工具文档** — Agent 工具行为、可用 agent 类型（`claude` / `Explore` / `Plan` / `general-purpose` 等）。来源：本会话系统提示。
4. **SDD prompt 模板**（后续课用）— `implementer-prompt.md` / `spec-reviewer-prompt.md` / `code-quality-reviewer-prompt.md`，在 subagent-driven-development skill 目录下。

## 外部官方文档（未在线核实，供用户参考）

- Claude Code 官方文档：https://docs.claude.com/en/docs/claude-code （subagents 章节）
- Anthropic 工程博客：https://www.anthropic.com/engineering （agent 相关文章）

> ⚠️ 以上 URL 未经本环境在线核实，以官方站点实际内容为准。

## 待补充

- 用户实际派发子 agent 的真实案例（教学中逐步积累）
