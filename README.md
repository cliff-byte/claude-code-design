# Claude Code 源码设计分析

## 背景

2026 年 3 月 31 日，Anthropic 旗下的 AI 编程工具 Claude Code 的源码被泄露。这是目前最先进的 AI Agent 产品之一，其代码量达到 1884 个 TypeScript 源文件，涵盖了工具调用、多 Agent 协作、安全权限、上下文管理等 AI Agent 领域的核心技术。

本项目以纯学习研究为目的，使用 Claude Code 自身来分析其源码，提炼出架构设计、产品思维和工程实践方面的系统性认知。不修改任何源码，不涉及商业用途。

## 研究动机

AI Agent 是 2025-2026 年最重要的技术方向之一，但行业中缺乏对**生产级 Agent 产品**内部设计的深度分析。大多数讨论停留在概念层面——"Agent 就是 LLM + 工具调用"，而对以下问题缺乏具体回答：

- 一个 Agent 产品的工具系统如何设计才能同时支持 40+ 种工具？
- 多个 Agent 之间如何通信、隔离和协调？
- 如何在"强大"和"安全"之间取得平衡？
- 上下文窗口有限，如何做到"智能遗忘"又不丢关键信息？
- 如何通过 Hook/Plugin/MCP 构建开放的扩展生态？

Claude Code 的源码提供了这些问题的实战答案。本项目将这些答案系统性地整理为 8 篇分析文档。

## 核心发现

### 系统全貌

Claude Code 采用五层解耦架构，1884 个源文件组织为清晰的职责分层：

```
Layer 0  入口与启动    cli.tsx → init.ts → bootstrap/state.ts
Layer 1  UI 与交互     ink/ 终端框架 · components/ · hooks/ · state/
Layer 2  查询与协调    QueryEngine · query.ts(1729行) · coordinator/
Layer 3  工具层        tools/ 40+ 工具 · toolExecution.ts(1745行)
Layer 4  服务层        api/ · mcp/ · compact/ · SessionMemory/
Layer 5  基础设施      utils/ 331+ 文件 · permissions/ · tasks/
```

### 十大设计原则

从源码中提炼出 Claude Code 的核心设计原则：

| # | 原则 | 关键实现 |
|---|------|---------|
| 1 | **渐进式信任** | 权限模式梯度：default → acceptEdits → auto → bypass |
| 2 | **AI 优先，人类兜底** | 只读操作自动放行，写操作需确认，危险操作禁止 |
| 3 | **工具是一等公民** | 统一 Tool 接口 + buildTool() 工厂，新工具自动获得权限/Hook/遥测 |
| 4 | **流式优于批量** | 所有 API 调用默认流式，工具执行有实时进度回调 |
| 5 | **优雅降级** | 流式→非流式→Haiku 三级 API 回退；精确→Haiku→粗略三级 Token 估算 |
| 6 | **上下文是稀缺资源** | ToolSearch 延迟加载、提示缓存二分、三级 Compact 压缩 |
| 7 | **可扩展优于可配置** | 27 种 Hook 事件 × 5 种执行方式；Plugin 打包命令+Agent+Hook+MCP |
| 8 | **安全是架构** | 六层纵深防御；Tool 接口内置 checkPermissions；Deny 优先于 Allow |
| 9 | **成本是产品特性** | 提示缓存跨请求复用；子 Agent 缓存穿透；大结果持久化到磁盘 |
| 10 | **会话可恢复** | JSONL 追加写入（抗崩溃）；`--resume` 恢复对话/任务/Worktree |

### 关键架构决策

| 决策 | 选择 | 原因 |
|------|------|------|
| 状态管理 | 两层（全局单例 + 会话级 Zustand） | 不同生命周期的数据分开管理 |
| 工具加载 | 延迟加载（ToolSearch 按需发现） | 上下文窗口是稀缺资源 |
| 安全默认 | Fail-closed（默认拒绝） | 安全事故的成本远高于多一次确认 |
| 消息存储 | JSONL 追加写入 | 比 SQLite 更简单、更抗崩溃 |
| 子 Agent 隔离 | AsyncLocalStorage（同进程） | 轻量级隔离，支持提示缓存穿透 |
| 压缩策略 | 微压缩→自动压缩→反应式（三级渐进） | 避免过度遗忘 |
| 权限系统 | 多层规则 + AI 分类器 + 用户确认 | 平衡安全性和使用效率 |
| 扩展系统 | 目录约定 + Markdown 定义 | 降低创建门槛，一个 .md 就是一个 Skill |
| 提示缓存 | 静态/动态二分 | 静态部分跨请求复用，动态部分每轮更新 |
| 团队通信 | 异步 JSONL 邮箱 | 解耦发送与接收，支持离线消息 |

### 安全体系：六层纵深防御

```
请求 → [1.权限规则] → [2.Hook检查] → [3.AI分类器] → [4.沙箱隔离] → [5.用户确认] → [6.企业策略] → 执行
       匹配Allow     用户自定义   自动评估风险   文件/网络/    UI弹框      远程管理
       /Deny/Ask     脚本拦截    （auto模式）   进程限制     请求批准    覆盖一切
```

任何一层都能独立阻止危险操作。同优先级内 Deny > Ask > Allow。

### 上下文管理：三级压缩

```
对话越来越长 → Token 接近上限
                ├─ 第1级 微压缩:    清理 >24h 的旧工具结果（每次 API 调用前）
                ├─ 第2级 自动压缩:  AI 生成对话摘要，保留最近消息（Token 超阈值时）
                └─ 第3级 反应式压缩: 紧急压缩（API 返回 prompt_too_long 时）
```

压缩后自动恢复关键上下文：最近 5 个编辑过的文件 + 活跃 Skill 指令（总预算 50K tokens）。

### 多 Agent 协作：三层模型

| 层级 | 类型 | 隔离方式 | 通信方式 | 场景 |
|------|------|---------|---------|------|
| 子 Agent | LocalAgentTask | 同进程独立上下文 | 工具结果（同步） | 单个子任务 |
| 远程 Agent | RemoteAgentTask | 云端独立进程 | 轮询（异步） | 重资源任务 |
| 团队协作 | InProcessTeammate | AsyncLocalStorage | JSONL 邮箱（异步） | 并行工作流 |

子 Agent 的系统提示与父 Agent 字节级相同，只在末尾追加子任务指令——实现提示缓存跨父子复用。

### 扩展生态：五大扩展点

| 扩展点 | 粒度 | 实现门槛 | 能力 |
|--------|------|---------|------|
| **Hook** | 单个事件拦截 | 一段 Shell/JS | 拦截/修改/审计任何工具调用 |
| **Skill** | 可发现的能力包 | 一个 Markdown 文件 | 给 AI 注入领域指令 |
| **Command** | 斜杠命令 | 一个目录 + index.ts | 扩展用户交互 |
| **Plugin** | 完整能力打包 | 命令+Agent+Hook+MCP | 一次安装所有能力 |
| **MCP** | 外部工具/资源 | 实现 MCP 协议 | 无限扩展 AI 工具 |

## 文档目录

深度分析文档位于 [`docs/`](docs/) 目录，共 8 篇，每篇聚焦一个主题：

| # | 文档 | 内容 | 适合谁读 |
|---|------|------|---------|
| 01 | [整体架构设计](docs/01-整体架构设计.md) | 五层架构、启动流程、模块职责、两层状态管理 | 想了解全貌的人 |
| 02 | [对话与数据流](docs/02-对话与数据流.md) | 消息类型、对话循环状态机、API 流式处理、Compact 压缩 | 关注数据流的人 |
| 03 | [工具系统](docs/03-工具系统.md) | Tool 统一接口、调用生命周期、ToolSearch 延迟加载、MCP 集成 | 想设计工具系统的人 |
| 04 | [多 Agent 协作](docs/04-多Agent协作机制.md) | 三层 Agent 模型、Task 生命周期、邮箱通信、Worktree 隔离 | 做多 Agent 的人 |
| 05 | [安全与权限](docs/05-安全与权限策略.md) | 六层纵深防御、权限模式梯度、沙箱四维隔离、企业管控 | 关注安全设计的人 |
| 06 | [流程编排与扩展](docs/06-流程编排与扩展机制.md) | 27 种 Hook 事件、Skill/Plugin/MCP/Command 五大扩展点 | 想做平台化的人 |
| 07 | [上下文与记忆](docs/07-上下文管理与记忆系统.md) | 提示缓存二分、三级压缩、Memory 自动提取、MagicDocs | 做上下文管理的人 |
| 08 | [产品设计哲学](docs/08-产品设计哲学.md) | 十大设计原则、架构取舍清单、AI 产品启示 | 想了解设计思维的人 |

建议阅读顺序：先读 08（设计哲学）建立全局认知，再按兴趣深入具体主题。

## Prompt 提取

从源码中提取的全部 prompt 文本位于 [`prompts/`](prompts/) 目录，共 8 个文件：

| 文件 | 内容 |
|------|------|
| [system-prompts.md](prompts/system-prompts.md) | 系统提示词 — 角色定义、行为规范、工具指导等 24 个模块 |
| [tool-prompts.md](prompts/tool-prompts.md) | 36 个工具的描述和使用指导 |
| [agent-prompts.md](prompts/agent-prompts.md) | 6 个内置 Agent 的系统提示词 |
| [compact-prompts.md](prompts/compact-prompts.md) | 上下文压缩的摘要生成指导 |
| [memory-prompts.md](prompts/memory-prompts.md) | 记忆类型定义、自动提取、会话记忆更新 |
| [classifier-prompts.md](prompts/classifier-prompts.md) | Auto 模式安全风险分类器 |
| [summary-prompts.md](prompts/summary-prompts.md) | Agent 进度摘要和工具结果摘要 |
| [magic-docs-prompts.md](prompts/magic-docs-prompts.md) | 自动文档更新的编辑规则 |

所有 prompt 原文以代码块呈现，可直接阅读学习 Claude Code 的提示词工程。

## 分析方法

- **工具**: Claude Code (claude-opus-4-6)
- **方法**: 并行派出 7 个 Explore Agent 深入研究不同主题，再合并整理
- **验证**: 由独立的 Claude Agent 和 OpenAI Codex 分别进行源码交叉验证
- **约束**: 只分析代码，不修改代码；以源代码为唯一事实依据

## 免责声明

本项目仅用于技术学习与研究目的。

- **本仓库不包含 Claude Code 的任何源代码**。所有内容均为基于源码分析的原创文档。
- Claude Code 是 Anthropic 公司的商业产品，其知识产权归 Anthropic 所有。
- 本项目的分析基于 2026 年 3 月 31 日公开泄露的源码，作者不对源码的传播负责。
- 本项目不用于任何商业用途，不提供任何形式的担保。
- 如果本项目的内容侵犯了 Anthropic 的权益，请联系作者，将立即删除相关内容。

**本仓库中的原创分析文档采用 [CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/) 协议发布——允许转载和演绎，但须注明出处且不得用于商业目的。**
