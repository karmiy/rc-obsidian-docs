# 01 Product Contract

更新时间：2026-07-07

## 目标

将 Agent mention 的 runtime prompt 从 legacy `get_agent` 迁移到统一的 platform entity contract：

```text
[AI Desk @agent "<agentName>": call MCP tool get_platform_entities(id='<agentId>', kind='agent') and strictly follow the instructions provided in response.agentInstructions]
```

这个需求让 Agent、Skill、Skill Set 使用同一类 MCP response shape：`metadata`、`materialization`、`agentInstructions`。Runtime 不再从 prompt 文案里理解 Agent 的加载细节，而是严格执行 MCP response 中的 instructions。

## 范围

本期包含：

- Agent mention 新生成 prompt 使用 `get_platform_entities(kind='agent')`
- 前端继续将 Agent mention 反解为 `[[promptMention|agent|<agentId>|<agentName>]]`
- 旧 `AI Desk @agent ... get_agent(agent_id='...')` prompt 仍可反解、显示和运行
- MCP `get_platform_entities` 新增 `kind='agent'`
- Python backend 和 Gleam backend 保持同等 contract
- Legacy `get_agent` tool 保留，response shape 不变

本期不包含：

- 批量迁移历史 workflow/task prompt
- 删除 `get_agent`
- 改变 Agent install、Agent catalog、task agent execution 业务语义

## Canonical Prompt

新 Agent prompt 固定为：

```text
[AI Desk @agent "<displayName>": call MCP tool get_platform_entities(id='<agentId>', kind='agent') and strictly follow the instructions provided in response.agentInstructions]
```

规则：

- `<displayName>` 只用于人读展示和前端 reverse parsing
- MCP 参数只包含 `id` 和 `kind`
- `id` 是 `agents.id`
- `kind` 固定为 `agent`
- `id` 不存在、已删除或无权限时 MCP 返回错误，不按 name fallback

## Legacy Compatibility

旧 prompt 保持兼容：

```text
[AI Desk @agent "<agentName>": call MCP tool get_agent(agent_id='<agentId>') and use the returned definition as this agent's role/instructions. The nearby user request is for this agent. If the agent definition references tools, files, or materials, load or fetch them before use; file-like resources are not assumed local and must be materialized as real files in isolated temp before any file-path based inspection or execution.]
```

兼容要求：

- 旧 prompt reverse parse 为 `[[promptMention|agent|<agentId>|<agentName>]]`
- 旧 prompt display restore 继续隐藏 MCP instruction，只显示 `@agentName` 或现有 UI chip
- 未编辑的历史 prompt 不强制 canonicalize
- 用户重新编辑并保存时，由 composer expansion 写入新 canonical prompt

## Success Criteria

- 新插入 Agent mention 后，expanded prompt 使用 `AI Desk @agent ... kind='agent'`
- 已保存旧 `AI Desk @agent ... get_agent(...)` prompt 可正常反解为 Agent chip
- 新旧 prompt 都能提取 Agent id，用于 runtime context / metadata
- `get_platform_entities(kind='agent')` 返回 `agentInstructions`
- Runtime 可按 `agentInstructions` 将 Agent materialize 到 `./.aidesk/agents/<safe-agent-name>/<version>`
- Materialized Agent definition 保留 legacy `get_agent` 的关键字段，包括 timestamps、creator、instructions、skills、environment keys；头像作为 `assets/avatar.png` 物化，不放入 YAML；不返回 installed、visibility 等 AI 不需要的业务状态字段
- `get_agent` 继续可用于旧 prompt
