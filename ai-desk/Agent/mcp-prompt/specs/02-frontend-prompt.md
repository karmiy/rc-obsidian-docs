# 02 Frontend Prompt Contract

更新时间：2026-07-07

## 现有边界

当前 Agent mention 相关逻辑集中在：

- `frontend/src/utils/slashPromptAgentInstructions.ts`
- `frontend/src/utils/aiDeskAgentStreamDisplay.ts`
- `frontend/src/services/task-acp-chat/common/TaskACPRuntimeUserDisplay.ts`
- `frontend/src/components/common/slash-prompt/useAgentMentionToolProvider.ts`
- `frontend/test/slash-prompt-agent-instructions-contract.test.cjs`
- `frontend/test/task-acp-runtime-user-display.test.cjs`

业务组件不应手写 Agent MCP prompt。

## Internal Marker

Agent mention 的 composed marker 不变：

```text
[[promptMention|agent|<agentId>|<agentName>]]
```

示例：

```text
[[promptMention|agent|11290346-ed9c-4bbc-8b56-1e8aa28fceb0|Jira actions agent]]
```

不要新增 `promptMention|skill` 或其它 Agent marker。这样可以最小化对 SlashPromptComposer、workflow editor、task chat editor 的影响。

## Expansion

`formatAgentMentionInstruction` 输出新 canonical prompt：

```text
[AI Desk @agent "<agentName>": call MCP tool get_platform_entities(id='<agentId>', kind='agent') and strictly follow the instructions provided in response.agentInstructions]
```

实现要求：

- display name 需要转义 `"` 和 `\`
- MCP value 需要转义 `'` 和 `\`
- 空 name fallback 仍使用 `Agent`
- 只改集中 formatter，不在 provider 或页面中拼字符串

## Reverse Parsing

`expandedAgentPromptToComposed` 必须识别新格式：

```text
[AI Desk @agent "<agentName>": call MCP tool get_platform_entities(id='<agentId>', kind='agent') and strictly follow the instructions provided in response.agentInstructions]
```

输出：

```text
[[promptMention|agent|<agentId>|<agentName>]]
```

同时继续识别旧格式：

```text
[AI Desk @agent "<agentName>": call MCP tool get_agent(agent_id='<agentId>') ...]
```

旧格式 parser 应继续容忍 `get_agent(...)` 后面的 instruction text 变化，避免历史 prompt 因文案微调失效。

## ID Extraction

`extractAgentMentionIdsFromPromptValue` 必须同时支持：

- composed marker：`[[promptMention|agent|...]]`
- new expanded prompt：`get_platform_entities(id='<agentId>', kind='agent')`
- legacy expanded prompt：`get_agent(agent_id='<agentId>')`

该函数用于 runtime context 和相关 metadata 收集，不应只依赖 composed marker。

## Display Restore

所有将 runtime prompt 还原成用户可读文本的路径，需要隐藏新 MCP instruction。

需要重点更新/验证：

- `expandedAgentPromptToComposed` in `frontend/src/utils/slashPromptAgentInstructions.ts`
  - 负责把新旧 expanded Agent prompt 还原为 `[[promptMention|agent|...]]`
- `aiDeskAgentUserTextToDisplay` in `frontend/src/utils/aiDeskAgentStreamDisplay.ts`
  - 已通过 `expandedAgentPromptToComposed(...)` 做 Agent mention display restore；新格式支持应从这里自然生效
- `taskACPRuntimePromptToVisibleUserText` in `frontend/src/services/task-acp-chat/common/TaskACPRuntimeUserDisplay.ts`
  - ACP navigator tooltip、aria label、copy text 使用该函数；测试必须覆盖新 `get_platform_entities(kind='agent')` prompt

要求：

- 新 prompt 展示为 Agent mention/chip，而不是暴露 `[AI Desk @agent ...]`
- 旧 prompt 展示行为不变
- `@agent` display prefix matches the Agent UI identity; internal marker remains `promptMention|agent`

## Frontend Tests

需要覆盖：

- Agent marker expands to new `AI Desk @agent ... kind='agent'`
- New prompt reverse parses to `[[promptMention|agent|...]]`
- Legacy `AI Desk @agent ... get_agent(...)` still reverse parses
- Parser tolerates legacy instruction text changes
- ID extraction supports composed, new expanded, and legacy expanded prompt
- `aiDeskAgentUserTextToDisplay` hides new expanded MCP instruction
- `taskACPRuntimePromptToVisibleUserText` hides new expanded MCP instruction in ACP navigator/copy text
