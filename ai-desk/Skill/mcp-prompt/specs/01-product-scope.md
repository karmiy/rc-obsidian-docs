# 01 Product Scope

更新时间：2026-07-06

## 背景

Standalone Skill 当前发给 runtime 的 prompt 使用 legacy MCP tool：

```text
[Please call MCP tool get_platform_skill(skill_id='<skillId>', display_slash='<skillName>') to get the full skill]
```

Skill Set 已经改为通过 `get_platform_entities` 返回 `materialization` 和 `agentInstructions`。该方式让 AI 先调用 MCP tool，再严格按照 response 中的 instructions 去物化和读取文件，比直接返回大段 `files` 更稳定。

本需求将 standalone Skill 也统一到同一套 platform entity MCP contract。

## 目标

Standalone Skill 选择后，runtime prompt 使用新的 canonical format：

```text
[AI Desk @skill "bug-fix": call MCP tool get_platform_entities(id='<skillId>', kind='skill') and strictly follow the instructions provided in response.agentInstructions]
```

目标包括：

- 独立 Skill 和 Skill Set 共享 `get_platform_entities` 的 runtime-facing 方案
- MCP response 通过 `agentInstructions` 告诉 AI 如何 materialize 和读取 Skill
- 前端能反解新旧 prompt，继续显示为 `/bug-fix` chip
- 已存量 workflow 的旧 prompt 保持可运行
- 用户重新编辑含旧 prompt 的 step 后，保存为新的 canonical prompt

## 非目标

本期不做：

- 删除 legacy `get_platform_skill`
- 对历史 workflow 批量迁移 DB 数据
- 在新 prompt 的 MCP call 参数里加入 `displayName` 或 `display_slash`
- 通过 skill name fallback 查找 stale skill id
- 改变 repo skill 或 user local skill 的 prompt format

## Canonical Prompt

新 standalone Skill prompt 固定为：

```text
[AI Desk @skill "<displayName>": call MCP tool get_platform_entities(id='<skillId>', kind='skill') and strictly follow the instructions provided in response.agentInstructions]
```

规则：

- `<displayName>` 只用于人读和前端 reverse parsing
- MCP tool 参数只包含 `id` 和 `kind`
- `id` 是 skills table row UUID
- `kind` 固定为 `skill`
- 如果 `id` 查不到或无权限，MCP 应失败，不按 name fallback

## Legacy Compatibility

旧 prompt 仍然必须被识别：

```text
[Please call MCP tool get_platform_skill(skill_id='<skillId>', display_slash='<skillName>') to get the full skill]
```

以及历史变体：

```text
[Please call MCP tool get_platform_skill(skill_id='<skillId>', name='<skillName>') to get the full skill]
```

```text
[Please call MCP tool get_platform_skill(skill_id='<skillId>') to get the full skill]
```

```text
[Please call MCP tool get_platform_skill(skill_name='<skillName>', project_id='<projectId>') to get the full skill]
```

Legacy prompt 的运行路径继续由 `get_platform_skill` 保底。新保存或新发送时使用新的 canonical prompt。

## 保存与升级策略

不做全局 save-time canonicalize。

原因：

- 当前 workflow prompt editor 已经通过 expanded prompt reverse parsing 恢复 chip
- 用户编辑某个 step prompt 时，onChange 会把 marker 重新 expand 成 canonical prompt
- 没编辑 prompt 只改其它字段时，保留旧 prompt 可以接受，因为 legacy MCP tool 仍然存在

因此升级策略是 opportunistic migration：

- 新插入 Skill：写入新 canonical prompt
- 旧 prompt 被编辑后：写入新 canonical prompt
- 旧 prompt 未被编辑：继续保留旧格式并保持可运行

## 成功标准

- 新选择 standalone Skill 后，发给 runtime 的 prompt 是 `AI Desk @skill ... get_platform_entities(... kind='skill')`
- 旧 workflow prompt 能在 composer 中反解成 `/skill` chip
- 旧 workflow prompt 重新编辑后保存为新 canonical prompt
- 新 prompt 调 MCP 后能返回 `agentInstructions`
- Runtime 可按 `agentInstructions` 下载 archive、读取 `SKILL.md`
- Legacy `get_platform_skill` 继续可用
