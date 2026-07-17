# Harness 各层速查 — 读透一个 harness 该看什么

> 把一个 harness 拆成层，每层回答四个问题：**是什么 / 在 Claude Code·Agent SDK 里对应什么 / 关键决策点 / 对应课程**。
> "读透一个陌生 harness"就按这个表从上往下看。

## 0. 心智模型（不是层，是总图）
- **是什么**：harness = 包裹基础模型的工程脚手架，把一问一答的模型变成能干活的 agent。
- **关键判断**：裸模型 → augmented LLM（+检索+工具+记忆）→ workflow（预定义路径）/ agent（自主循环）。
- **课程**：L01 harness 是什么 · L02 agentic loop · L03 workflow vs agent

## 1. 系统提示词层
- **是什么**：塑造 agent 行为的"角色设定 + 规则 + 约束 + 上下文"。
- **CC / SDK 对应**：Claude Code 的 system prompt（见 Piebald 仓库，各版本可读）；Agent SDK 可 `modifying system prompts`。
- **关键决策**：身份、能力边界、do/don't、输出格式、few-shot。
- **课程**：L04 作用结构写法 · L05 拆 CC system prompt

## 2. 工具层
- **是什么**：agent 能调用的能力 + 调用循环。
- **CC / SDK 对应**：内置工具 Read/Write/Edit/Bash/Glob/Grep/WebSearch…；Agent SDK `allowed_tools`、custom tools、MCP、tool search。
- **关键决策**：工具粒度、描述信息量（ACI）、命名、参数防呆、错误处理。
- **课程**：L06 机制与定义 · L07 循环与写好工具 · L08 读 SDK 工具实现

## 3. 上下文管理层
- **是什么**：决定什么进上下文、什么被压缩/丢弃、何时注入。
- **CC / SDK 对应**：CC 的上下文压缩；Agent SDK 的 context management（与 CC 同源）；sessions（resume/fork）。
- **关键决策**：窗口预算、压缩策略、消息历史裁剪、检索注入。
- **课程**：L09 窗口压缩原则 · L10 记忆与状态注入 · L11 读 SDK 上下文实现

## 4. 权限与安全层
- **是什么**：约束 agent 能做什么、对什么资源做，防越界。
- **CC / SDK 对应**：CC 权限规则（allow/deny/ask）；Agent SDK `permission_mode`、`allowed_tools`、permissions 配置。
- **关键决策**：最小权限、危险操作确认、审计、hooks 介入。
- **课程**：L12 权限模型 · L13 hooks · L14 安全设计

## 5. 编排层
- **是什么**：组织 agent 工作流的拓扑——单 agent loop 还是多 agent 协作。
- **CC / SDK 对应**：CC 的 subagents（`.claude/agents/`）；Agent SDK 的 `agents`/subagents、Agent 工具。
- **关键决策**：单/多 agent、用哪种模式（派发/orchestrator-workers/路由/并行/评估器-优化器）。
- **课程**：L15 编排全景 · L16 分发类 · L17 评估器-优化器 · L18 售前编排决策

## 6. 可观测性与错误恢复（运维面）
- **是什么**：让 agent 可被观察、可恢复。
- **CC / SDK 对应**：Agent SDK 的 OpenTelemetry、cost/usage tracking、checkpointing、Todo Lists。
- **关键决策**：日志/trace 粒度、重试/降级、中断恢复、成本上限。
- **课程**：L19 错误恢复与可观测性（选学）

## 7. 定制面（Claude Code 专属入口）
- **是什么**：不改源码就能调 harness 行为的配置入口。
- **CC / SDK 对应**：`CLAUDE.md`（记忆）、`settings.json`、hooks、权限规则、`.claude/agents/`、skills、commands、plugins。
- **关键决策**：每个定制改的是上面哪一层、带来什么效果、副作用。
- **课程**：L20 定制面全景 · L21-23 实战

## 8. 实现内核（Agent SDK 源码）
- **是什么**：harness 各层的真实代码实现。
- **怎么看**：Agent SDK = CC 开源内核，按 agent loop / 工具系统 / 上下文管理 三块读源码。
- **课程**：L24 架构总览 · L25 读 loop+工具源码 · L26 读上下文源码 · L27 跑自定义 agent

## 9. 对比与售前
- **是什么**：把样本抽象成方法论，迁移到选型与售前。
- **关键决策**：Agent SDK vs LangGraph（循环式 vs 图式）vs AutoGen（对话式）；自建 vs SDK vs CC；POC 怎么快搭。
- **课程**：L28-29 框架对比 · L30-33 售前方法论

---

### 一句话读透法
> 拿到一个 harness，先找它的 **agentic loop**（心跳），再顺着 loop 看它**调什么工具、谁管上下文、谁管权限、怎么编排、怎么观察**——五问下去，骨架就清楚了。
