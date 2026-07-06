# Q&A — 子 agent

> 教学中用户提出的问题与回答，持续整理。方便后续查看复习。

---

## SDD 是什么？
SDD = Subagent-Driven Development（子 agent 驱动开发）。把计划拆成独立任务，每个派一个**全新**子 agent 实现，实现后过两道 review（规格符合 → 代码质量），通过才进下一个。核心公式：fresh subagent per task + 两阶段 review。主会话当控制器，子 agent 当执行者。详见 [选学课程 0006 · SDD 工作流](lessons/0006-sdd-workflow.html)。

## Claude 原生 subagent 和 superpowers 的 SDD 冲突吗？
不冲突。两者不同层：原生 subagent 是**能力/基础设施**（Agent 工具 + agent 类型 + `.claude/agents/` 自定义）；SDD 是建在原生 Agent 工具之上的**工作流方法论**。SDD 不另起炉灶，就是调用原生工具。类比：原生是引擎，SDD 是驾驶手册。

## 为什么这个环境 WebSearch 不可用？
环境模型是 glm-5.2（非 Anthropic Claude），像是兼容 Claude Code 协议的第三方部署。WebSearch 依赖 Anthropic 搜索后端，此部署未接入。靠本地资料 + 子 agent。具体部署配置无法查证。

## 派子 agent 时，主会话（main）在干什么？
做协调活，不读代码：派之前理解需求/核对 prompt/构造指令/决定派哪种；子 agent 跑时主会话等；返回后整合讲解。代码分析是子 agent 独立做的。即"控制器 vs 执行者"分工。

## 子 agent 的结果能直接输出到文档吗？
能。两种方式：(A) 主会话拿到返回后写文件；(B) 让子 agent 自己写——但 `Explore` 是只读不能写，要用 `general-purpose` 等可写 agent，并在 prompt 输出格式里明确"写入 X 文件"。

## 调用子 agent 必须用特定 prompt 触发吗？模型能自己判断吗？
模型能自主判断，不需要特定 prompt 触发。但是否倾向于派受五方面影响：模型倾向、系统提示、skill、CLAUDE.md 指令、用户直接要求。**不写也能派，写了更可靠一致**。不是"必须写明否则无法拆"。

## 子 agent 的 token 消耗如何判断/控制？
子 agent 有独立上下文，消耗 = 读的文件 + 思考 + 输出，总成本 = 主会话 + 所有子 agent。
- **判断**（黑盒，靠预估）：任务规模、广度、子 agent 数量、输出长度。
- **控制**：①广度提示（medium 而非 very thorough）②约束输出（简洁/不贴源码/限长）③限定范围（指定文件目录）④选对 agent（只读用 Explore）⑤模型选择（自定义 agent 配 haiku 降本）⑥拆分粒度。
- 子 agent 内部 token 无实时仪表盘，靠 prompt 预防。

## 子 agent 执行过长 / 循环 / 强制停止怎么办？
两种模式：
- **同步等待**（默认）：主会话阻塞，用户 **Esc** 可中断。
- **后台运行**（`run_in_background`）：主会话继续，`TaskOutput` 查进度、`TaskStop` 强制停。

循环风险存在，缓解：prompt 明确无歧义、给退出条件（"找不到就返回未找到"）、限制广度。强制停止：后台用 TaskStop，同步用 Esc。
（本环境 glm-5.2 部署的具体中断/超时机制未完全确定，上述为 Claude Code 通用机制。）

## Explore 的广度有哪些程度词？
描述里明确给了两个：`medium`（中等）和 `very thorough`（非常彻底）。实际广度是自然语言提示，非硬枚举，可用任何程度词：quick/brief（浅）→ medium（中）→ thorough（彻底）→ very thorough（最彻底）。描述只明确背书 medium 和 very thorough，其余为合理推断。广度也是控制 token 的手段（广度小=读得少）。

## CC 内置了哪些 subagent？区别是什么？
当前环境可用的 agent 类型（系统提示给出，是当前部署真实可用，非凭记忆）：
- **Explore**：只读广度搜索，扫文件要结论不贴文件堆。只读。
- **Plan**：软件架构师，设计实现计划。只读。
- **general-purpose**：通用多步任务/复杂搜索，工具全开。可写。
- **claude**：通用回退，没指定类型时的默认。可写。
- **claude-code-guide**：回答 Claude Code/SDK/API 相关问题。只读（有 WebFetch/WebSearch）。
- **statusline-setup**：配置状态栏。仅可改状态栏。
- **stock-deep-analyzer:investor-panel**：股票分析（插件提供，非内置）。

三个关键区分：① **只读**(Explore/Plan/claude-code-guide) vs **可写**(general-purpose/claude)——决定能否让子 agent 自己写文件；② 专用 vs 通用回退；③ 带 `插件名:` 前缀的是插件提供的，不带的是 CC 自带，另可在 `.claude/agents/` 自定义。

选型速查：搜代码要结论→Explore；规划实现→Plan；要写文件/改代码/多步→general-purpose；问 CC 本身→claude-code-guide；无合适→claude。

注：此列表是当前环境实际可用，CC 内置类型随版本变；看自己环境完整列表看 Agent 工具的 available agent types。想核对官方全集可派 `claude-code-guide`。

## 我要自己写 `Agent(...)` 这种代码来调度子 agent 吗？
不用。`Agent(subagent_type:..., prompt:...)` 是**主会话（模型）**运行时调用的工具，不是用户写的代码。用户一侧永远是自然语言：说需求（如"并行分析 X 和 Y"），主会话负责判断拆分、给每个子 agent 写 prompt、在一条消息里发起多个 Agent 调用（并发）、收结果整合。用户是"下需求的人"，主会话是"调度员"。

想让模型更倾向并行/拆分，在 CLAUDE.md 写自然语言偏好即可（如"大项目多模块分析优先并行派 Explore"），仍非写代码。当然你也可以用自然语言主动指挥（"派 3 个分别看 A/B/C"），但那是描述需求，不是写 `Agent(...)`。

实证（第 5 课实操）：用户一句话"看 raft1 的选举和日志复制咋实现"，主会话拆成 2 个 Explore 并行派发、统一收口整合，全程用户没写任何调度代码。

## SDD 里的 implementer 是什么概念？是内置 agent 类型吗？
不是内置类型，是 SDD 流程里的一个**角色**——指"负责实现任务的那个子 agent"。它实际由 `general-purpose`（可写、全工具）这类 agent + SDD 的 `implementer-prompt` 模板扮演。

区分两个层次：
- **角色**（implementer / spec 审查者 / 质量审查者）= SDD 流程分工，靠 prompt 模板定义"演什么"。skill 提供 `implementer-prompt.md` / `spec-reviewer-prompt.md` / `code-quality-reviewer-prompt.md` 三个模板。
- **agent 类型**（general-purpose / Explore / Plan…）= 系统的"演员"，决定工具权限和模型。

类比剧组：角色是"演什么"，agent 类型是"哪个演员演"。implementer 要写代码 → 由可写的 general-purpose 扮演；审查角色只读够 → 可用 Explore 类 + 审查 prompt 扮演。（具体选型 skill 没硬性规定，按"角色需要什么权限"选。）

要点：implementer 是 SDD 术语，不是 CC 通用概念，出了 SDD 就没这个固定东西。下一节自造 subagent 后，可把 implementer 从"临时角色"固化成自定义 agent 类型（`.claude/agents/implementer.md`，配 tools=可写、model=便宜快模型），届时 CC 能自动派它。
