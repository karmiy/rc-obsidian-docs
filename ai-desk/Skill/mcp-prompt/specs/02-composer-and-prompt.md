# 02 Composer 与 Prompt 兼容方案

更新时间：2026-07-06

## 目标

复用当前 SlashPromptComposer 架构，让 standalone Skill 在 UI 中继续表现为 `/skill-name`，但 expanded runtime prompt 改为新的 platform entity MCP instruction。

业务页面不应该手动拼接 Skill prompt。prompt format、reverse parsing、marker expansion 都应集中在 shared utility 中。

## Internal Marker

Standalone Skill 反解后继续使用现有 `vibeSkill` marker：

```text
[[vibeSkill|aidesk|<skillName>|id=<skillId>]]
```

示例：

```text
[[vibeSkill|aidesk|bug-fix|id=11111111-1111-1111-1111-111111111111]]
```

原因：

- standalone Skill 从用户视角仍是 `/skill`
- `vibeSkill` 已支持 `id` metadata
- 与 repo skill、user skill、Skill Set sub skill 的 skill chip 行为保持一致

Skill Set 整体 mention 仍然使用：

```text
[[promptMention|skillSet|<skillSetId>|<skillSetName>]]
```

Skill Set sub skill 仍然使用：

```text
[[vibeSkill|aidesk|<skillName>|id=<skillId>|skillSetId=<skillSetId>|skillSetName=<skillSetName>]]
```

本需求不新增 `[[promptMention|skill|...]]`。

## Expanded Prompt

Standalone Skill marker 展开为：

```text
[AI Desk @skill "<skillName>": call MCP tool get_platform_entities(id='<skillId>', kind='skill') and strictly follow the instructions provided in response.agentInstructions]
```

示例：

```text
[AI Desk @skill "bug-fix": call MCP tool get_platform_entities(id='11111111-1111-1111-1111-111111111111', kind='skill') and strictly follow the instructions provided in response.agentInstructions]
```

规则：

- display name 需要转义 `"` 和 `\`
- MCP argument value 需要转义 `'` 和 `\`
- 没有 `id` 的 malformed aidesk skill 不应生成 `kind='skill'` prompt
- set-owned skill 继续生成 `kind='skillSet', skillId='<skillId>'`

## Reverse Parsing

`expandedPromptToComposed` 必须识别新格式：

```text
[AI Desk @skill "<displayName>": call MCP tool get_platform_entities(id='<skillId>', kind='skill') and strictly follow the instructions provided in response.agentInstructions]
```

输出：

```text
[[vibeSkill|aidesk|<displayName>|id=<skillId>]]
```

同时继续识别 legacy formats：

```text
[Please call MCP tool get_platform_skill(skill_id='<skillId>', display_slash='<displayName>') to get the full skill]
```

```text
[Please call MCP tool get_platform_skill(skill_id='<skillId>', name='<displayName>') to get the full skill]
```

```text
[Please call MCP tool get_platform_skill(skill_id='<skillId>') to get the full skill]
```

```text
[Please call MCP tool get_platform_skill(skill_name='<skillName>', project_id='<projectId>') to get the full skill]
```

Legacy id-only format 没有 display name 时，UI fallback name 使用：

```text
aidesk-skill
```

## Display Restore

Agent/user message display restore 需要识别新格式：

```text
[AI Desk @skill "bug-fix": call MCP tool get_platform_entities(id='<skillId>', kind='skill') and strictly follow the instructions provided in response.agentInstructions] do it
```

展示为：

```text
/bug-fix do it
```

## Skill ID Extraction

`extractPlatformSkillIdsFromExpandedPrompt` 必须同时从以下格式提取 skill id：

- new `get_platform_entities(id='<skillId>', kind='skill')`
- legacy `get_platform_skill(skill_id='<skillId>', ...)`

该函数用于 runtime 前置逻辑和 agentmesh skill id 收集，不应只识别 legacy format。

## Workflow 行为

Workflow step prompt 使用 `WorkspaceSkillPromptComposer`，它以 `valueMode="expanded"` 读取存量 prompt。

行为要求：

- 打开旧 prompt：反解为 `/skill` chip
- 用户编辑 prompt：onChange 输出新 canonical prompt
- 用户未编辑 prompt：不强制 canonicalize，旧 prompt 保持原值
- 保存新建或编辑后的 step：使用新 canonical prompt

## 测试点

Frontend contract tests 需要覆盖：

- standalone Skill marker expands to `AI Desk @skill ... kind='skill'`
- new standalone Skill prompt reverse parses to `[[vibeSkill|aidesk|...|id=...]]`
- legacy `display_slash` prompt reverse parses to same marker
- legacy `name` prompt reverse parses to same marker
- legacy id-only prompt reverse parses with fallback name
- `agentSkillStdinToDisplayText` supports new format
- `extractPlatformSkillIdsFromExpandedPrompt` extracts ids from new and legacy formats
- Skill Set formats remain unchanged
