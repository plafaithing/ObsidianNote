# Q&A — Harness Engineering

> 用户在学习中提出的问题（尤其概念 / 机制类），持续追加。问答事实存这里，里程碑洞察存 `learning-records/`。
> 格式：**Q** 问题 → **A** 回答 → 关联课程 / 延伸。

---

## L02 · agentic loop 代码细节

### Q1：run_tool 是大模型处理的函数吗？
**A：不是。** `run_tool` 是 **harness 这边（你的代码）** 的函数，大模型完全不执行工具。
- 模型只输出"我要调 Read 工具，参数是 `file.py`"——这叫 `tool_call`（API 里的 `tool_use` 内容块）。
- 真正去读文件、跑命令的是 harness 侧的 `run_tool`。
- 核心分工：**模型决策，harness 执行**。Claude Code 里 Read/Bash/Edit 等内置工具，就是 harness 侧的 run_tool 实现。
- 关联：L06 tool use 机制、L07 工具循环。

### Q2：messages 里的"旧历史"是什么？变量存储还是全量打过来？tool_call 和 result 为什么都要放？
**A：** 三个小问：

1. **旧历史是什么**：本次会话从头到现在的所有消息——第一条用户问题、之前每一轮模型要调的工具、harness 返回的结果、模型的中间文字。即前面所有执行记录。
2. **全量打过来**：对，**全量**。Anthropic Messages API 是**无状态**的——服务器不替你记历史，每次 `messages.create` 都要把完整对话重新发一遍，不是服务器侧有个变量存着。这正是 agent 跑久了上下文会爆的根源。
3. **为什么 tool_call 和 result 都要放**：模型要"看见"自己刚调了什么、拿到什么结果，才能决定下一步。API 规范要求**成对**：
   - 模型的 `tool_use`（assistant 角色）——"我要调这个工具"
   - harness 的 `tool_result`（user 角色）——"工具返回了这个"
   - 只放 result，模型不知道是哪个调用返回的；只放 tool_call，模型以为还没执行会重复调。

   历史结构示意（每轮都是 assistant 的 tool_use + user 的 tool_result 成对）：
   ```
   messages=[
     {"role":"user","content":"当前目录有哪些文件？"},                                    # 第1条问题
     {"role":"assistant","content":[{"type":"tool_use","name":"Glob",...}]},            # 模型第1轮：调 Glob
     {"role":"user","content":[{"type":"tool_result",...}]},                            # harness 返回 Glob 结果
     {"role":"assistant","content":[{"type":"tool_use","name":"Read",...}]},            # 模型第2轮：调 Read
     {"role":"user","content":[{"type":"tool_result",...}]},                            # harness 返回 Read 结果
     ...
   ]
   ```
- 关联：L09 上下文管理（历史膨胀与压缩）、L06 tool use 机制、L11 读 SDK 上下文实现。

### Q3：stop_reason 是模型自行修改吗？
**A：不是模型"修改"某个变量。** `stop_reason` 是 **API 每次响应里返回给你的字段**，由模型这一轮的输出决定：
- 模型这轮想调工具 → 响应 `stop_reason = "tool_use"` → while 继续
- 模型这轮输出最终回答（不调工具）→ `stop_reason = "end_turn"` → 循环退出
- （其他可能值：`max_tokens` 超长截断、`stop_sequence` 命中停止序列）

机制：模型的行为决定 stop_reason 的值，harness 每轮从 `response.stop_reason` 读出来判断要不要再来一圈。模型通过"输出工具调用 / 输出文本"**间接控制**循环的继续与终止——这就是"模型掌控循环走向"在代码里的落点。
- 关联：L02 agentic loop、L19 错误恢复（max_tokens 等异常 stop_reason 的处理）。

### Q4：ClaudeAgentOptions 是把前面那段 while 接口化了吗？
**A：方向对，但要分清两个角色。**
- 把 while 循环封装掉的，是 `query()` 函数——它替你转循环、执行内置工具、维护消息历史。
- `ClaudeAgentOptions` 不是"封装 while 的东西"，而是递给 `query()` 的**配置单**：告诉这个封装好的 loop 用哪些工具、什么权限、挂不挂 hooks 等。

对照：
- 手写版：while + run_tool + 拼 messages，全自己写
- Agent SDK：`query()` = 发动机（转循环）；`ClaudeAgentOptions` = 配置单（装什么工具、限什么权限）

**关键连接**：ClaudeAgentOptions 的字段几乎一一对应 harness 的一层（L01 总图）：
- `allowed_tools` → 工具层 + 权限层
- `permission_mode` → 权限层（如 acceptEdits）
- `hooks` → hooks 层（PreToolUse/PostToolUse）
- `agents` → 编排层（自定义 subagent）
- `mcp_servers` → 工具层（接外部工具）
- `setting_sources` / `CLAUDE.md` → 记忆层

即 ClaudeAgentOptions 是 **harness 各层暴露成可配置参数的入口**。这也是 L20 定制实战的伏笔：定制 Claude Code / 用 Agent SDK 搭 agent，本质就是填这张配置单。
- 关联：L02 agentic loop、L01 harness 七层、L20 CC 定制面全景、L24 Agent SDK 架构总览。

### Q5：回喂 tool_result 时，是否包含新的提示词？
**A：不包含。** 回喂这一步只是**在 messages 末尾追加两条**（assistant 的 `tool_use` + user 的 `tool_result`），其余一律不变：

| 部分 | 回喂时 | 说明 |
|------|--------|------|
| system prompt | 不变 | 每次请求都带（API 无状态要求），但整个 loop 里是同一个，不重新生成；且在单独的 `system` 参数里，不在 messages 数组 |
| user 新指令 | 没有 | 用户没说话，user 消息里只有 tool_result 块 |
| tools 清单 | 不变 | 同一套工具定义 |
| messages 末尾 | 追加两条 | 模型上步的 tool_use + 工具返回的 tool_result |

L02 代码 `messages=[...旧历史, tool_call, result]` 里的 `tool_call` 容易让人误以为是"新提示词"，其实它是**模型自己上一步的输出**——因为 API 无状态，模型不记得自己刚说了啥，得把它的话放回历史，它才知道"我刚才调了 list_files，现在结果回来了"。`result` 才是真正"喂回"的内容。

**注意"上步输出"不是说它已在历史里**：模型本轮的 tool_use 是从 response 里返回的，返回时还没进 messages。harness 要把它作为一条 **assistant 消息追加**进 messages（连同 user 的 tool_result 一起追加）。即 `messages=[...旧历史, tool_call, result]` 里后两条都是**本轮新增**，都不属于旧历史——代码注释"历史里追加这一轮"即此意。

"提示词"三种理解辨析：
- 指 **system prompt** → 每次带但不变，不是"喂回"对象
- 指 **user 新指令** → 回喂时没有
- 指 **给模型的输入** → tool_result 算新信息，但它是"工具结果/事实数据"，不是"指令"

口诀：「回喂不喂新提示，只追 tool_use 配 tool_result；system 每次带但不变」
- 关联：L02 agentic loop、L06 tool use 机制、L09 上下文管理（历史膨胀）。

---
