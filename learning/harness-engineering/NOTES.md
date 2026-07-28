# NOTES — Harness Engineering 教学

> 教学工作区偏好与备忘。新会话开始时先读这里。

## 工作区
- 根：`learning/`（Obsidian 笔记库内）
- 当前主题目录：`learning/harness-engineering/`
- 总索引：`learning/INDEX.md`

## 用户偏好（沿用教学工作区约定）
- **课程格式**：HTML（teach skill 默认，Tufte 风格，内联 CSS 保证 Obsidian 兼容；参考 `rag` 主题样式与 callout/quiz 组件）
- **归档格式**：md，放 `archive/`
- **交流方式**：开放对话为主，不强制结构化多选项；提问给方向但留自由发挥空间
- **语言**：中文

## 出题规则（用户强调，2026-07-27）
- **不重复**：随堂题 / 模块测验 / 期中 / 期末**四层共享一个去重池**。同概念可多次出现，但必须换场景/换角度/换考法（识别 vs 机制 vs 决策 vs 排错 vs 应用）。出题前先扫已有题查重。
- **变种巩固**：用户对某题"始终不懂"时，不要原题重出——出同概念不同场景的**变种题**（换数据/角色/提问方向），多角度巩固直到能独立答对。
- 详见 `tasks/lessons.md` L001。

## 用户当前水平（harness 主题）
- Claude Code 基础使用者，但 harness 整体概念**基础薄弱**
- system prompt / tool use 循环 / context window 等概念不熟 → **从零讲起，不假设前置**
- 前置已学：subagents 课（掌握"派发"能力）→ 编排层可衔接，不重复教派发

## 学习路径判断
- **近期层定制** = 改 Claude Code（`CLAUDE.md`/hooks/`settings`/权限/subagents），门槛低、第一战场
- **远期层定制** = 用 Agent SDK 自建 agent harness，售前场景真正要走的路
- 方法论抽象自 Claude Code / Agent SDK 样本，迁移到售前

## 真实场景（教学案例库，持续积累）
- 售前：客户要定制 agent，需要知道 harness 从哪下手、哪些层决策
- 待积累：用户实际遇到的 Claude Code 使用 / 定制痛点

## Mission
- ✅ 已定稿（2026-07-13），见 `MISSION.md`
- 落点：主看 Claude Agent SDK（Claude Code 开源内核）+ Claude Code 运行时配置面；第三方框架作对比补充，突出差异点
