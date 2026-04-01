# Prompt 提取索引

从 Claude Code 源码中提取的全部 prompt 定义文件，共 **51 个文件**，按用途分为 8 个类别。

> 这些文件是 TypeScript 源码，prompt 文本以模板字符串或函数返回值的形式定义在代码中。

## 目录结构

```
prompts/
├── system/          # 系统提示词 — AI 的角色定义和行为规范
├── tools/           # 工具提示词 — 36 个工具的描述和使用指导
├── agents/          # 内置 Agent — 6 种专用 Agent 的系统提示
├── compact/         # 压缩提示词 — 对话摘要生成指导
├── memory/          # 记忆提示词 — 记忆提取和会话记忆更新
├── classifier/      # 分类器提示词 — Auto 模式的安全风险评估
├── summary/         # 摘要提示词 — Agent 进度摘要和工具结果摘要
└── magic-docs/      # MagicDocs — 自动文档更新指导
```

## 分类说明

### system/ — 系统提示词（2 个文件）

Claude Code 的"总指挥"，定义了 AI 的角色、能力边界和行为规范。

| 文件 | 说明 |
|------|------|
| `prompts.ts` | 核心系统提示词构建，包含角色定义、任务指导、工具使用规范、输出风格等 |
| `systemPromptSections.ts` | 系统提示词的分段缓存机制，区分可缓存（静态）和不可缓存（动态）部分 |

### tools/ — 工具提示词（36 个文件）

每个工具的 `prompt.ts` 定义了该工具给 AI 看的描述和使用指导。

| 文件 | 工具 | 说明 |
|------|------|------|
| `BashTool.ts` | Bash | Shell 命令执行指导，含 Git 安全协议、后台执行说明（最大，~160 行） |
| `PowerShellTool.ts` | PowerShell | PowerShell 版本特定语法指导（~145 行） |
| `FileReadTool.ts` | Read | 文件读取指导，含 PDF、图片、Notebook 支持说明 |
| `FileEditTool.ts` | Edit | 文件编辑指导，含精确匹配和替换规则 |
| `FileWriteTool.ts` | Write | 文件写入指导 |
| `GrepTool.ts` | Grep | 正则搜索指导，含 ripgrep 语法说明 |
| `GlobTool.ts` | Glob | 文件模式匹配指导 |
| `AgentTool.ts` | Agent | 子 Agent 创建指导，含后台执行和 Worktree 隔离 |
| `SendMessageTool.ts` | SendMessage | Agent 间消息传递指导 |
| `SkillTool.ts` | Skill | 自定义能力包调用指导 |
| `ToolSearchTool.ts` | ToolSearch | 工具延迟发现指导 |
| `TaskCreateTool.ts` | TaskCreate | 任务创建指导 |
| `TaskUpdateTool.ts` | TaskUpdate | 任务状态更新指导 |
| `TaskGetTool.ts` | TaskGet | 任务查询指导 |
| `TaskListTool.ts` | TaskList | 任务列表指导 |
| `TaskStopTool.ts` | TaskStop | 任务停止指导 |
| `TodoWriteTool.ts` | TodoWrite | 待办事项管理指导 |
| `WebSearchTool.ts` | WebSearch | 网络搜索指导 |
| `WebFetchTool.ts` | WebFetch | 网页内容获取指导 |
| `MCPTool.ts` | MCP | MCP 协议工具指导 |
| `EnterPlanModeTool.ts` | EnterPlanMode | 进入计划模式指导 |
| `ExitPlanModeTool.ts` | ExitPlanMode | 退出计划模式指导 |
| `EnterWorktreeTool.ts` | EnterWorktree | Git Worktree 创建指导 |
| `ExitWorktreeTool.ts` | ExitWorktree | Worktree 退出指导 |
| `AskUserQuestionTool.ts` | AskUserQuestion | 向用户提问指导 |
| `BriefTool.ts` | Brief | 简化输出指导 |
| `ConfigTool.ts` | Config | 配置查询指导 |
| `LSPTool.ts` | LSP | 语言服务器协议工具指导 |
| `NotebookEditTool.ts` | NotebookEdit | Jupyter Notebook 编辑指导 |
| `TeamCreateTool.ts` | TeamCreate | 团队创建指导 |
| `TeamDeleteTool.ts` | TeamDelete | 团队删除指导 |
| `ListMcpResourcesTool.ts` | ListMcpResources | MCP 资源列表指导 |
| `ReadMcpResourceTool.ts` | ReadMcpResource | MCP 资源读取指导 |
| `ScheduleCronTool.ts` | ScheduleCron | 定时任务调度指导 |
| `RemoteTriggerTool.ts` | RemoteTrigger | 远程触发器指导 |
| `SleepTool.ts` | Sleep | 延时等待指导 |

### agents/ — 内置 Agent 提示词（6 个文件）

预定义的专用 Agent，每个有独立的系统提示词和工具限制。

| 文件 | Agent 类型 | 说明 |
|------|-----------|------|
| `generalPurposeAgent.ts` | 通用 Agent | 代码搜索和多步骤研究任务 |
| `exploreAgent.ts` | 探索 Agent | 只读模式快速代码库探索，强调并行工具调用 |
| `planAgent.ts` | 规划 Agent | 只读规划模式，禁止文件修改，输出实现方案 |
| `verificationAgent.ts` | 验证 Agent | 对抗性验证和质量检查，含详细验证策略（~130 行） |
| `claudeCodeGuideAgent.ts` | 使用指南 Agent | Claude Code 功能问答 |
| `statuslineSetup.ts` | 状态栏配置 Agent | 终端状态栏设置 |

### compact/ — 压缩提示词（1 个文件）

| 文件 | 说明 |
|------|------|
| `prompt.ts` | 对话压缩摘要生成的完整指导（~335 行），定义了 9 个摘要部分的结构 |

### memory/ — 记忆提示词（2 个文件）

| 文件 | 说明 |
|------|------|
| `extractMemories-prompts.ts` | 后台自动记忆提取 Agent 的指导，包含记忆类型识别和去重逻辑 |
| `sessionMemory-prompts.ts` | 会话记忆更新的模板和指导（~325 行），定义了笔记格式 |

### classifier/ — 分类器提示词（1 个文件）

| 文件 | 说明 |
|------|------|
| `yoloClassifier.ts` | Auto 模式的 AI 安全分类器（~200+ 行），评估工具调用的风险等级 |

注：分类器引用了外部化的 `.txt` 模板文件（`auto_mode_system_prompt.txt` 等），这些文件在编译时被内联，源码中不存在独立副本。

### summary/ — 摘要提示词（2 个文件）

| 文件 | 说明 |
|------|------|
| `agentSummary.ts` | 后台 Agent 进度摘要生成（每 30 秒），输出 3-5 词简述 |
| `toolUseSummaryGenerator.ts` | 工具批次结果摘要（Git 风格简述，如"Searched in auth/"） |

### magic-docs/ — MagicDocs 提示词（1 个文件）

| 文件 | 说明 |
|------|------|
| `prompts.ts` | MagicDoc 自动更新 Agent 的指导，定义了文档编辑规则和更新哲学 |
