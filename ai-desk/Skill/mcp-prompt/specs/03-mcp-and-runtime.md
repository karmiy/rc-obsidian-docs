# 03 MCP 与 Runtime 契约

更新时间：2026-07-06

## 目标

让独立 Skill 通过 `get_platform_entities(kind='skill')` 返回与 Skill Set 同风格的物化描述和 agent instructions。

Runtime 不再依赖 `get_platform_skill` 返回大段 `files` 内容来理解新 prompt。历史 prompt 仍可继续走 `get_platform_skill`。

## MCP Tool

```text
get_platform_entities
```

输入：

```json
{
  "kind": "skill",
  "id": "skill-id"
}
```

本需求完成后，该 tool 支持：

```text
kind = skill
kind = skillSet
```

`kind='skill'` 不接受 `skillId` 参数。

## 返回结构

独立 Skill 返回：

```json
{
  "metadata": {
    "kind": "skill",
    "id": "11111111-1111-1111-1111-111111111111",
    "name": "bug-fix",
    "description": "...",
    "version": "1.0.0"
  },
  "materialization": {
    "type": "archive",
    "format": "zip",
    "archiveUrl": "https://current-api-domain/api/v1/platform-entities/archive?token=...",
    "targetDir": "./.aidesk/skills/bug-fix/1.0.0",
    "parentDir": "./.aidesk/skills/bug-fix",
    "entryFile": "./.aidesk/skills/bug-fix/1.0.0/SKILL.md"
  },
  "agentInstructions": [
    "The user selected Skill \"bug-fix\".",
    "You MUST first check whether materialization.targetDir already exists. If it exists, use the cached files there and do NOT download materialization.archiveUrl. If it does not exist, you MUST download materialization.archiveUrl and extract the zip into materialization.targetDir before using this Skill.",
    "You MUST preserve all relative paths from the archive.",
    "After confirming the current version exists, you SHOULD remove older cached versions of the same Skill under materialization.parentDir. Only remove older version directories for this same Skill. Do NOT remove the current version, newer versions, other Skills, Skill Sets, or any project files outside materialization.parentDir.",
    "You MUST read and follow materialization.entryFile as the selected skill instruction source for this request."
  ]
}
```

说明：

- `agentInstructions` 是给 runtime AI 执行的指令，实际返回文本建议继续使用英文，和当前 Skill Set 保持一致。
- 文档中的中文用于说明产品和工程规则，不要求 MCP response 字段名翻译。

## 版本

`metadata.version` 应使用持久化 Skill version。

如果旧数据没有独立 Skill version，后端应使用现有 Skill version 的 fallback 策略。

实例化路径包含 version，便于缓存复用和清理旧版本：

```text
./.aidesk/skills/<skill-name>/<version>/
```

## Archive 内容

Archive 应包含完整的独立 Skill 文件，并保留相对路径。

必需文件：

```text
SKILL.md
```

如果持久化 files 中没有 `SKILL.md`，但 Skill content 存在，后端应按 legacy `get_platform_skill` 当前逻辑合成 `SKILL.md`。

Archive 示例：

```text
SKILL.md
docs/context.md
scripts/helper.py
```

## Archive 下载接口

现有 endpoint：

```text
GET /api/v1/platform-entities/archive?token=<signed-token>
```

Token payload 支持 `kind='skill'`：

```json
{
  "purpose": "platformEntityArchive",
  "kind": "skill",
  "id": "skill-id",
  "version": "1.0.0",
  "extensionId": "requester-extension-id",
  "isSystem": false,
  "exp": 1780000000
}
```

Archive endpoint 按 kind 分发：

- `skill`：构建独立 Skill archive
- `skillSet`：构建 Skill Set archive

Endpoint 必须基于 token claims 重新查询 DB 状态并校验权限。Token 本身不等于授权。

## 权限规则

`kind='skill'` 使用与 legacy `get_platform_skill` 相同的可见性规则：

- admin/system MCP 可以读取
- workspace/public 可见 Skill 可以读取
- private Skill 只能由 owner 读取
- deleted Skill 必须失败

如果 Skill id stale、不可见或不存在，返回 MCP error。不按 display name fallback。

## Legacy Tool

`get_platform_skill` 继续保留，用于已有历史 prompt。

Legacy response shape 保持不变：

```json
{
  "skillId": "...",
  "skillName": "...",
  "projectId": null,
  "files": {
    "SKILL.md": "..."
  }
}
```

本需求不把 legacy tool 改成返回 materialization。原因是旧 prompt 直接要求 agent 调用 `get_platform_skill`，并且可能依赖 `files` shape。

## 后端可观测性

需要增加或复用 structured logs，覆盖：

- `get_platform_entities kind=skill` 请求接收
- 权限或 id 校验失败
- archive URL 生成
- archive 成功生成
- archive 生成被拒绝或失败

已有 platform entity archive 日志中会 redacted identifiers 的位置，新日志也应保持同样规则。

## Gleam Parity

Gleam backend 的 MCP tool listing 和 handler contract 必须同步更新：

- `get_platform_entities` schema enum 包含 `skill`
- handler 接受 `kind='skill'`
- archive route 接受 `kind='skill'`
- tests 断言 `skill` 和 `skillSet` 都被 advertised

如果 Gleam 无法在同一 work item 中完整构建 Skill archive，必须在 merge 前明确记录 parity gap。优先目标是 full parity。

## 测试

Backend tests 需要覆盖：

- MCP tools list 中 `get_platform_entities` enum 包含 `skill` 和 `skillSet`
- `get_platform_entities(kind='skill')` 返回 materialization 和 agentInstructions
- unknown/deleted/private Skill 返回受控错误
- archive token 使用 `kind='skill'` 时可以下载 zip
- archive 内容包含 `SKILL.md`
- legacy `get_platform_skill` 仍返回旧 `files` shape
- usage/reference extraction 能识别新的 `kind='skill'` prompt
- system remote runtime skill id extraction 能识别新的 prompt
