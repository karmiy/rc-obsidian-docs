# 04 MCP 与 Runtime 方案

更新时间：2026-07-03

## 目标

通过 MCP 把 Skill Set 资源提供给外部 runtime。AI Desk 不控制 Codex / Claude runtime 内部逻辑，所以 MCP response 必须返回明确的 `materialization` 下载信息和 `agentInstructions`。

MCP response 不再直接返回完整 `files` 内容。大 JSON 在 runtime 工具输出里可能被折叠或截断，导致 AI 无法完整读取 `skills/` 目录。完整 Skill Set bundle 改为通过后端签名的 archive URL 下载。

## MCP Tool

```text
get_platform_entities
```

Input：

```json
{
  "kind": "skillSet",
  "id": "skill-set-id",
  "skillId": "optional-sub-skill-id"
}
```

本期只实现：

```text
kind = skillSet
```

## 返回规则

始终返回完整 Skill Set bundle 的 materialization descriptor。

以下两种调用都指向同一个完整 archive：

```text
get_platform_entities(id='<setId>', kind='skillSet')
```

```text
get_platform_entities(id='<setId>', kind='skillSet', skillId='<skillId>')
```

`skillId` 只影响 `metadata.selectedSkill`、`materialization.entryFile` 和 `agentInstructions`。

## 返回结构

```json
{
  "metadata": {
    "kind": "skillSet",
    "id": "1",
    "name": "Superpowers",
    "version": "1.0.0",
    "description": "...",
    "selectedSkill": null
  },
  "materialization": {
    "type": "archive",
    "format": "zip",
    "archiveUrl": "https://current-api-domain/api/v1/platform-entities/archive?token=...",
    "targetDir": "./.aidesk/skill-sets/Superpowers/1.0.0",
    "parentDir": "./.aidesk/skill-sets/Superpowers",
    "entryFile": "./.aidesk/skill-sets/Superpowers/1.0.0/README.md",
    "manifestFile": "./.aidesk/skill-sets/Superpowers/1.0.0/manifest.json"
  },
  "agentInstructions": [
    "The user selected the whole Skill Set \"Superpowers\".",
    "You MUST first check whether materialization.targetDir already exists. If it exists, use the cached files there and do NOT download materialization.archiveUrl. If it does not exist, you MUST download materialization.archiveUrl and extract the zip into materialization.targetDir before using this Skill Set.",
    "You MUST preserve all relative paths from the archive.",
    "After confirming the current version exists, you SHOULD remove older version directories under materialization.parentDir.",
    "You MUST read and follow materialization.entryFile as the primary instruction source for this request.",
    "You MUST read materialization.manifestFile to understand the available sub skills and files."
  ]
}
```

选择 sub skill 时：

```json
{
  "metadata": {
    "kind": "skillSet",
    "id": "1",
    "name": "Superpowers",
    "version": "1.0.0",
    "description": "...",
    "selectedSkill": {
      "id": "333",
      "name": "brainstorming",
      "displayName": "Superpowers.brainstorming"
    }
  },
  "materialization": {
    "type": "archive",
    "format": "zip",
    "archiveUrl": "https://current-api-domain/api/v1/platform-entities/archive?token=...",
    "targetDir": "./.aidesk/skill-sets/Superpowers/1.0.0",
    "parentDir": "./.aidesk/skill-sets/Superpowers",
    "entryFile": "./.aidesk/skill-sets/Superpowers/1.0.0/skills/brainstorming/SKILL.md",
    "manifestFile": "./.aidesk/skill-sets/Superpowers/1.0.0/manifest.json"
  },
  "agentInstructions": [
    "The user selected sub skill \"Superpowers.brainstorming\" from Skill Set \"Superpowers\".",
    "You MUST first check whether materialization.targetDir already exists. If it exists, use the cached files there and do NOT download materialization.archiveUrl. If it does not exist, you MUST download materialization.archiveUrl and extract the zip into materialization.targetDir before using this skill.",
    "You MUST preserve all relative paths from the archive.",
    "After confirming the current version exists, you SHOULD remove older version directories under materialization.parentDir.",
    "You MUST read and follow materialization.entryFile as the selected skill instruction source for this request."
  ]
}
```

## manifest.json

`manifest.json` 由后端生成，并放入 archive。

它不存入 `SkillSetBundleFile`。
它不在 Set Content UI 中展示。

结构：

```json
{
  "id": "1",
  "name": "Superpowers",
  "version": "1.0.0",
  "description": "...",
  "skills": [
    {
      "id": "333",
      "name": "Brainstorming",
      "description": "...",
      "path": "skills/brainstorming/SKILL.md"
    }
  ]
}
```

本期不需要 `schemaVersion`。只有当它未来变成长期对外兼容协议时，再增加 schema version。

## Agent Instructions 规则

选择整个 set 时：

```text
Read README.md first, then use manifest.json to decide which skill instructions are relevant.
```

选择 sub skill 时：

```text
After materializing the full set, read skills/<skill>/SKILL.md and follow that selected skill.
```

Composer 发给 runtime 的 prompt 保持稳定：

```text
[Please call MCP tool get_platform_entities(...) and strictly follow the instructions provided in response.agentInstructions]
```

如果以后 materialization 规则变化，优先改后端返回的 `materialization` 和 `agentInstructions`，不要频繁改 composer prompt。

## Archive 下载接口

```text
GET /api/v1/platform-entities/archive?token=<signed-token>
```

token 为短期 stateless signed token，不依赖单个 backend pod 内存状态。payload 至少包含：

```json
{
  "purpose": "platformEntityArchive",
  "kind": "skillSet",
  "id": "skill-set-id",
  "version": "1.0.0",
  "extensionId": "requester-extension-id",
  "isSystem": false,
  "exp": 1780000000
}
```

Archive endpoint 必须：

- 验签和过期时间
- 校验 `purpose`
- 根据 `kind` dispatch，本期只支持 `skillSet`
- 按 token 内 `id/version/extensionId/isSystem` 重新查询 DB 和校验权限
- 动态生成 zip 返回

`archiveUrl` 必须由 backend 根据配置或当前 request 动态生成，不能写死域名。

## 实例化路径

Runtime 应在当前 cwd 下实例化文件：

```text
./.aidesk/skill-sets/<set-name>/<version>/
```

示例：

```text
./.aidesk/skill-sets/Superpowers/1.0.0/
  README.md
  manifest.json
  skills/
    brainstorming/
      SKILL.md
```

## 安全校验

MCP response generation 必须校验：

- no absolute paths
- no `..`
- no duplicate paths
- no missing `README.md`
- no missing `skills/<skill>/SKILL.md`
- archive 内包含后端动态生成的 `manifest.json`
- archive 内保留 utf8 和 binary 文件的真实内容
