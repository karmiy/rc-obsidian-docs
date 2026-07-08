# 03 MCP Backend Plan

更新时间：2026-07-07

## MCP Tool Contract

`get_platform_entities` 新增：

```json
{
  "kind": "agent",
  "id": "agent-id"
}
```

完成后支持：

```text
kind = agent
kind = skill
kind = skillSet
```

`skillId` 仍只允许用于 `kind='skillSet'`。

## Agent Response Shape

`kind='agent'` 返回：

```json
{
  "metadata": {
    "kind": "agent",
    "id": "11111111-1111-1111-1111-111111111111",
    "name": "Bug Fixer",
    "description": "...",
    "version": "1.0.0"
  },
  "materialization": {
    "type": "archive",
    "format": "zip",
    "archiveUrl": "https://current-api-domain/api/v1/platform-entities/archive?token=...",
    "targetDir": "./.aidesk/agents/Bug Fixer/1.0.0",
    "parentDir": "./.aidesk/agents/Bug Fixer",
    "entryFile": "./.aidesk/agents/Bug Fixer/1.0.0/AGENT.md"
  },
  "agentInstructions": [
    "The user selected Agent \"Bug Fixer\".",
    "You MUST first check whether materialization.targetDir already exists. If it exists, use the cached files there and do NOT download materialization.archiveUrl. If it does not exist, you MUST download materialization.archiveUrl and extract the zip into materialization.targetDir before using this Agent.",
    "You MUST preserve all relative paths from the archive.",
    "After confirming the current version exists, you SHOULD remove older cached versions of the same Agent under materialization.parentDir. Only remove older version directories for this same Agent. Do NOT remove the current version, newer versions, Skills, Skill Sets, other Agents, or any project files outside materialization.parentDir.",
    "You MUST read and follow materialization.entryFile as the selected agent instruction source for this request.",
    "The YAML frontmatter in materialization.entryFile contains the structured Agent definition.",
    "If the agent definition references skills, tools, files, or materials, load or fetch them before use. File-like resources are not assumed local and must be materialized as real files in an isolated temp directory before any file-path based inspection or execution."
  ]
}
```

说明：

- Agent 和 Skill 一样走 archive materialization，避免 runtime 只拿到不完整 inline snippet。
- `targetDir` 使用 `./.aidesk/agents/<safe-agent-name>/<version>`。
- `<safe-agent-name>` 使用与 Skill 相同的 safe materialization 规则：trim 后将 `\`、`/`、换行、引号等不安全字符替换为 `-`。例如 `Hello/agent` 变成 `Hello-agent`。
- `metadata.version` 使用 Agent version；没有 version 的旧数据 fallback 到 `1.0.0`。

## Archive 内容

Agent archive 必须提供一个适合 AI 直接阅读、同时保留结构化 metadata 的入口文件。

必需文件：

```text
AGENT.md
```

可选文件：

```text
assets/avatar.png
```

当 Agent 有头像时，后端必须把存储的 base64/data URL 头像作为真实图片文件写入 `assets/avatar.png`，不要把头像 base64 放进 `AGENT.md` YAML。

`AGENT.md` 采用 Claude subagent 风格：YAML frontmatter + Markdown body。

frontmatter 应尽量覆盖 legacy `get_agent` payload 的关键字段：

```md
---
kind: agent
id: 121a02bf-23a2-43c5-bef8-2fbff033ee6d
name: Hello agent
description: say hello
version: 1.0.0
creatorExtensionId: "8967481597427794"
createdAt: "2026-07-07T01:57:25.701689+00:00"
updatedAt: "2026-07-07T02:15:04.041601+00:00"
skills:
  - d5549c86-3673-4b6c-9b5c-6fe9b1b1c229
  - 02e2b0f5-b0dd-4ab0-8246-a473b5b60d9c
environmentKeys:
  - T
---
# Hello agent

please use [AI Desk @skill "solution-helper": call MCP tool get_platform_entities(id='d5549c86-3673-4b6c-9b5c-6fe9b1b1c229', kind='skill') and strictly follow the instructions provided in response.agentInstructions] to say hello。
```

字段要求：

- 不泄漏 `environment` values，只返回 `environmentKeys`
- 不返回 `avatarBase64`；有头像时在 archive 中提供 `assets/avatar.png`
- 保留 `createdAt`、`updatedAt`、`creatorExtensionId`
- 保留 `skills` 原始列表
- 不返回 `visibility`、`installed`、`installCount`、`taskDefinitionCount` 等 AI 不需要的业务状态字段
- Markdown body 保留 `instructions` 原文，里面可能包含 Skill 的 platform entity prompt
- `AGENT.md` 是 `materialization.entryFile`

## Archive 下载接口

现有 endpoint：

```text
GET /api/v1/platform-entities/archive?token=<signed-token>
```

Token payload 支持 `kind='agent'`：

```json
{
  "purpose": "platformEntityArchive",
  "kind": "agent",
  "id": "agent-id",
  "version": "1.0.0",
  "extensionId": "requester-extension-id",
  "isSystem": false,
  "exp": 1780000000
}
```

Archive endpoint 按 kind 分发：

- `agent`：构建 Agent archive
- `skill`：构建 standalone Skill archive
- `skillSet`：构建 Skill Set archive

Endpoint 必须基于 token claims 重新查询 DB 状态并校验权限和 version。Token 本身不等于授权。

## Legacy Tool

`get_agent` 保留，返回旧 shape：

```json
{
  "agent": {
    "id": "...",
    "name": "...",
    "description": "...",
    "instructions": "...",
    "skills": [],
    "environmentKeys": []
  }
}
```

不要把 legacy `get_agent` 改成 `metadata/materialization/agentInstructions`，否则旧 prompt 语义会被动变化。

## Python Backend

主要改动：

- `backend/app/api/v1/mcp.py`
  - `get_platform_entities` tool schema enum 增加 `agent`
  - `handle_get_platform_entities` 支持 `kind='agent'`
  - `kind='agent'` 不接受 `skillId`
  - structured log 复用现有 `mcp.get_platform_entities.*`
- Agent service 层新增或复用 bundle builder
  - 复用 `AgentService.get(...)` 权限规则
  - 复用 `get_agent` 对 environment 的处理，只暴露 `environmentKeys`
  - 返回 archive materialization fields：`archiveUrl`、`targetDir`、`parentDir`、`entryFile`
  - 使用 shared `safe_materialization_name` 生成 Agent cache path
  - 不泄漏 environment values
- platform entity archive endpoint
  - dispatch `kind='agent'`
  - 生成包含 `AGENT.md` 的 zip
  - zip path 使用统一 archive path 校验，拒绝 absolute path、`..` 和 duplicate path

权限规则与 `get_agent` 一致：

- system MCP 可读
- workspace Agent 可读
- private Agent 仅 owner 可读
- deleted Agent 不可读

## Gleam Backend Parity

必须同步更新：

- `backend_gleam/src/http/core/mcp_dispatch.gleam`
  - tool schema description / enum 包含 `agent`
- `backend_gleam/src/http/runtime_mcp_routes.gleam`
  - `get_platform_entities(kind='agent')`
  - 查询 `agents` 表并执行同等权限过滤
  - response shape 与 Python 一致
- `backend_gleam/src/http/runtime_platform_entity_routes.gleam`
  - archive route 支持 `kind='agent'`
  - safe materialization name 规则与 Python 一致
  - zip 内容包含 `AGENT.md`
- 现有 `get_agent` 保持 legacy 行为

不要只改 Python。Python 和 Gleam response 字段名、权限、environment value redaction 必须一致。

## Backend Reference Parsers

所有提取 Agent mention id 的 backend parser 需要识别新 prompt：

```text
get_platform_entities(id='<agentId>', kind='agent')
```

仍要识别旧 prompt：

```text
get_agent(agent_id='<agentId>')
```

需要重点检查：

- task/chat runtime context preparation
- remote runtime agentmesh metadata
- asset usage / reference tracking
- workflow prompt reference collection

## Tests

Python:

- MCP tools list advertises `kind='agent'`
- `get_platform_entities(kind='agent')` success
- private/deleted/not found Agent returns controlled MCP error
- response redacts environment values and only returns `environmentKeys`
- response returns archive materialization under `./.aidesk/agents/<safe-agent-name>/<version>`
- safe materialization converts `Hello/agent` to `Hello-agent`
- archive download returns zip containing `AGENT.md`
- `AGENT.md` frontmatter preserves legacy `get_agent` metadata fields
- `AGENT.md` body preserves legacy `get_agent.agent.instructions`
- legacy `get_agent` response unchanged
- Agent id extraction supports new and legacy prompt

Gleam:

- tool schema includes `agent`
- handler accepts `kind='agent'`
- visibility filtering matches Python
- response shape matches Python
- archive route supports `kind='agent'`
- safe materialization name matches Python
- legacy `get_agent` unchanged

Frontend:

- see `02-frontend-prompt.md`

## Verification Commands

Use focused tests first:

```bash
cd frontend && node test/slash-prompt-agent-instructions-contract.test.cjs
cd backend && venv/bin/python -m pytest tests/test_mcp_get_agent.py
cd backend_gleam && gleam test
```

Add or run additional touched-module tests when parser/reference code changes.

## QA Recommendation

推荐 QA Agent `full`。

原因：该需求影响 runtime-facing prompt contract、MCP tool schema、platform entity archive materialization、Python/Gleam parity、stored prompt compatibility，以及 task/workflow runtime context。
