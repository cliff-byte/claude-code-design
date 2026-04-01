# Prompt 提取索引

从 Claude Code 源码中提取的全部 prompt 文本，共 **8 个文件**，按用途分类。

所有 prompt 原文以 `text` 代码块呈现，避免 prompt 内部的 Markdown 标记干扰文档渲染。

## 文件索引

| 文件 | 内容 |
|------|------|
| [system-prompts.md](system-prompts.md) | 系统提示词 — AI 的角色定义、行为规范、工具使用指导、输出风格等 24 个模块 |
| [tool-prompts.md](tool-prompts.md) | 36 个工具的 prompt — 每个工具给 AI 看的描述和使用指导 |
| [agent-prompts.md](agent-prompts.md) | 6 个内置 Agent 的系统提示词 — 通用/探索/规划/验证/指南/状态栏 |
| [compact-prompts.md](compact-prompts.md) | 上下文压缩提示词 — 对话摘要生成的完整指导 |
| [memory-prompts.md](memory-prompts.md) | 记忆系统提示词 — 记忆类型定义、自动提取、会话记忆更新 |
| [classifier-prompts.md](classifier-prompts.md) | 权限分类器提示词 — Auto 模式下 AI 评估工具调用风险 |
| [summary-prompts.md](summary-prompts.md) | 摘要提示词 — Agent 进度摘要 (3-5词) 和工具结果摘要 (30字符) |
| [magic-docs-prompts.md](magic-docs-prompts.md) | MagicDocs 提示词 — 自动文档更新的编辑规则和哲学 |

## 阅读建议

- **了解 Claude Code 如何定义自己的角色** → 从 `system-prompts.md` 开始
- **了解 AI 如何决定用哪个工具** → 看 `tool-prompts.md` 中各工具的 Description
- **了解多 Agent 的分工** → 看 `agent-prompts.md` 中不同 Agent 的能力限制
- **了解"智能遗忘"如何工作** → 看 `compact-prompts.md` 的摘要生成指导
- **了解 AI 如何学习和记忆** → 看 `memory-prompts.md` 的记忆类型和提取规则
