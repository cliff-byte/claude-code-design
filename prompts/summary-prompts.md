# 摘要生成提示词

Claude Code 有两个摘要生成机制：**Agent 进度摘要**（AgentSummary）用于在 UI 中显示子 Agent 当前正在做什么，**工具使用摘要**（ToolUseSummary）用于在 SDK 中为已完成的工具调用批次生成可读标签。

---

## 1. Agent 进度摘要提示词（Agent Summary）

此提示词每 30 秒通过 fork 子 Agent 对话生成一次，产出 3-5 个词的进度描述，用于 UI 显示。fork 时复用主 Agent 的 prompt cache 以节省开销。工具被拒绝使用（`canUseTool` 返回 deny），模型只需生成纯文本回复。

### 摘要请求提示词

```text
Describe your most recent action in 3-5 words using present tense (-ing). Name the file or function, not the branch. Do not use tools.
${previousSummary ? '\nPrevious: "' + previousSummary + '" -- say something NEW.\n' : ''}

Good: "Reading runAgent.ts"
Good: "Fixing null check in validate.ts"
Good: "Running auth module tests"
Good: "Adding retry logic to fetchUser"

Bad (past tense): "Analyzed the branch diff"
Bad (too vague): "Investigating the issue"
Bad (too long): "Reviewing full branch diff and AgentTool.tsx integration"
Bad (branch name): "Analyzed adam/background-summary branch diff"
```

> 注：如果存在上一次摘要（`previousSummary`），会插入 `Previous: "${previousSummary}" -- say something NEW.` 以引导模型生成不重复的新进度描述。

---

## 2. 工具使用摘要提示词（Tool Use Summary）

此提示词由 Haiku 模型处理，为已完成的工具调用批次生成简短标签（类似 git commit subject），显示在移动端 SDK UI 中。约 30 字符会截断。

### 系统提示词

```text
Write a short summary label describing what these tool calls accomplished. It appears as a single-line row in a mobile app and truncates around 30 characters, so think git-commit-subject, not sentence.

Keep the verb in past tense and the most distinctive noun. Drop articles, connectors, and long location context first.

Examples:
- Searched in auth/
- Fixed NPE in UserService
- Created signup endpoint
- Read config.json
- Ran failing tests
```

### 用户提示词格式

```text
${lastAssistantText ? "User's intent (from assistant's last message): " + lastAssistantText.slice(0, 200) + '\n\n' : ''}Tools completed:

Tool: ${tool.name}
Input: ${truncatedInput}
Output: ${truncatedOutput}

[... 多个工具结果 ...]

Label:
```

> 注：每个工具的 input 和 output 被截断到 300 字符以控制 prompt 大小。如果有最近一条 assistant 文本，会作为意图上下文前置。
