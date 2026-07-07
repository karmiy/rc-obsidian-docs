# 05 Composer 集成方案

更新时间：2026-07-01

## 目标

让用户可以在 composer 中选择 installed Skill Sets，并在现有 `/` skill 选择体验里搜索 Skill Set 下的 sub skills。实现方式应复用当前 persisted skills 使用的 `SlashPromptComposer` 架构。

## 范围

支持两种入口：

- `@Superpowers`：使用整个 Skill Set
- `/Superpowers.Brainstorming`：在现有 skill slash list 中过滤并选择 set 内某个 skill

## 归属边界

installed skill sets 的加载、过滤、marker 创建和 prompt expansion 都应该封装在 `SlashPromptComposer` 现有体系内。

业务页面不应该手动拼 Skill Set prompt。

遵循当前 persisted skill 行为：

- UI 显示友好的 tag
- internal marker 保存实体 identity
- 最终发送消息前，把 marker 展开成 MCP instruction

## @ Skill Set 入口

搜索来源：

- installed Skill Sets only

展示项：

```text
Superpowers
```

Composer tag：

```text
Superpowers
```

Internal marker：

```text
[[vibeSkillSet|aidesk|Superpowers|setId=<setId>]]
```

Expanded prompt：

```text
[Please call MCP tool get_platform_entities(id='<setId>', kind='skillSet') and strictly follow the instructions provided in response.agentInstructions]
```

## / Set Sub Skill 入口

`/` 入口不做一套独立的 sub skill 菜单。它应该把 installed Skill Sets 下的 sub skills merge 到当前 slash composer / slash menu 的 skill list 中；从用户视角看，它仍然是在选一个 skill。

搜索来源：

- standalone installed skills
- installed Skill Sets 下的 sub skills

过滤行为：

```text
/Superpowers.Brainstorming
```

应该可以过滤出 `Superpowers` 下的 `Brainstorming`，并按当前 skill 搜索一样高亮匹配片段。

搜索匹配需要增强：

- 继续支持按 skill name 搜索，例如 `/Brainstorming`
- 新增支持按 set qualified name 搜索，例如 `/Superpowers.Brainstorming`
- `Superpowers` 和 `Brainstorming` 两段都参与匹配和高亮
- 结果项仍按 skill item 样式展示，额外展示 set context，例如 `Superpowers`

选择后的 composer tag：

```text
Brainstorming
```

Internal marker：

```text
[[vibeSkillSetSkill|aidesk|Brainstorming|setId=<setId>|skillId=<skillId>]]
```

Expanded prompt：

```text
[Please call MCP tool get_platform_entities(id='<setId>', kind='skillSet', skillId='<skillId>') and strictly follow the instructions provided in response.agentInstructions]
```

只调用一次 MCP。`skillId` 只是额外参数。

## 搜索与重名处理

不同 set 里可能有同名 skill，因此：

- 搜索结果需要展示 set context，例如 `Superpowers / Brainstorming`
- 选中后的 tag 可以只显示 `Brainstorming`
- marker 必须保存 `setId` 和 `skillId`

## Copy/Paste

Marker 需要像当前 `[[vibeSkill|...]]` 一样支持 composer copy/paste。

复制出的 marker 必须包含足够的信息用于之后展开：

- set id
- sub skill 的 skill id
- display name

## 测试点

需要覆盖：

- `@Superpowers` search 可以列出 installed set
- uninstalled set 不出现在 search
- `/Superpowers.Brainstorming` 可以找到 sub skill
- 选中 sub skill 后 tag 显示 `Brainstorming`
- marker 保留 `setId` 和 `skillId`
- marker 展开为 `get_platform_entities(... kind='skillSet')`
- 带 skillId 的 marker 展开为一次 MCP call，并带上 `skillId`
