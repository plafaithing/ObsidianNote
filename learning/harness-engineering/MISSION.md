# Mission: Harness Engineering 方法论与落地实践

> ✅ 已定稿 · 2026-07-13
> 学什么：围绕基础模型搭的工程脚手架（harness）——把"会聊天的模型"变成"能可靠干活的 agent"
> 为什么学：从"会用 Claude Code"进阶到"读透 harness 设计 + 能定制 + 售前能给方案"

## Why（为什么学）
- 用户是 Claude Code 基础使用者，一直把 harness 当黑盒用。想从"会用"进阶到"读透设计 + 能动手定制"。
- 现实驱动是售前：客户要定制 agent 时，得知道 harness 工程从哪下手、哪些层要决策，不能只会调 API。
- 具体改变：从"被 harness 当黑盒驱使"→"能拆解 harness 各层、能定制、能在售前给出方案"。

## Success looks like
- 能拆解一个 harness 由哪几层组成，讲清每层干什么、为什么这么设计
- 能读 Claude Agent SDK 源码，指出 agent loop / 工具 / 上下文管理各层在代码里怎么实现
- 能说出"读透一个陌生 harness"该按什么顺序看哪些东西
- 能动手做至少一次 Claude Code 定制（改 CLAUDE.md / 配 hooks / 写权限规则 / 调 subagent），并说清改的是哪层、效果是什么
- 能对比 Claude Agent SDK 与至少一个第三方框架（如 LangGraph）的设计差异，讲清各自取舍
- 售前场景：面对"帮我定制一个 agent"，能给出 harness 设计的切入点和决策框架

## Constraints
- 起点：基础薄弱，只基础使用过 Claude Code；system prompt / tool use 循环 / context window 等概念从零讲起，不假设前置知识
- 主样本：Claude Agent SDK（Claude Code 开源内核）+ Claude Code 运行时配置面；第三方框架作对比补充
- 方法论为主，落地为实践验证（非纯工程课，也非纯理论）
- 语言：中文；课程 HTML，归档 md
- 工作区：`learning/harness-engineering/`
- 前置：subagents 课已掌握"派发"能力（编排层会衔接，不重复教派发）

## Out of scope（暂不深入，除非后期需要）
- 完整从零自建 agent 框架（造轮子）——目标是读透 + 定制，不是重写
- 评估体系（evals）——基础阶段先不上，售前真需要再补
- 第三方框架（LangGraph/AutoGen 等）源码级精读——仅作对比，理解与 Agent SDK 的差异
- 复杂多 agent 编排框架的工程实现——编排「概念 + 模式 + 决策框架」in scope，框架源码实现 out of scope

## 样本分工

| 样本 | 读什么 | 用途 |
|------|--------|------|
| Claude Code（运行时/配置面） | system prompt、`settings.json`/hooks/`CLAUDE.md`/权限/subagents | 读透设计 + 定制实战（第一战场，门槛低） |
| Claude Agent SDK（开源实现面） | agent loop、工具、上下文管理源码（Python/TS） | 读透内核实现 + 售前自建基座（中后期精读） |
| 第三方框架（LangGraph 等） | 设计理念、拓扑、与 Agent SDK 的差异 | 对比补充，凸显设计取舍 |

> 关键事实：Claude Agent SDK 官方文档原话——"the same tools, agent loop, and context management that power Claude Code"。即 Agent SDK 就是 Claude Code 的开源内核，读它 ≈ 读 Claude Code 实现。
