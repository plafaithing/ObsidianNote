# Mission: 子 agent (subagent)

> ✅ 已定稿 · 2026-06-26

## Why
用户重度使用 Claude Code 做开发，但一直只用"单线程"方式——让 CC 自己读文件、改代码，上下文容易塞满、大活儿干得慢。想解锁"子 agent"这个能力：会判断什么活儿该派出去、能正确派子 agent 去搜索或实现，把 CC 从单兵变成可调度的协作体。

## Success looks like
- 能说出"什么任务适合派子 agent、什么不适合"，并讲清理由
- 能亲手派一个子 agent 去做一次搜索/实现任务，并拿到结果
- 理解子 agent 的核心边界：上下文隔离、并行不冲突、成本权衡

## Constraints
- 起点：从未主动派过子 agent，从零讲起
- 语言：中文
- 课程：HTML（teach skill 默认）；归档：md（Obsidian 复习）
- 工作区：`D:\workplace\doc\workplace\learning\subagents`

## Out of scope（暂不深入，除非后续要）
- superpowers 的 subagent-driven-development 完整工作流（派发-实现-两阶段 review）
- 自己定义 / 定制 subagent 类型
- 多 agent 高级编排
