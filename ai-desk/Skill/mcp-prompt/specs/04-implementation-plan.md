# 04 Implementation Plan

更新时间：2026-07-06

## 目标

将 standalone Skill runtime prompt 从 legacy `get_platform_skill` 迁移到 `get_platform_entities(kind='skill')`，同时保持旧 workflow prompt 可运行。

## 开发顺序

### 1. Frontend Prompt Formatter

更新 centralized Skill prompt formatter。

要求：

- standalone AIDesk Skill with id expands to new canonical prompt
- set-owned Skill 仍然 expands to `kind='skillSet', skillId='<skillId>'`
- repo/user Skill prompt 不变
- malformed aidesk Skill without id 不生成 `kind='skill'`

建议保留现有 function name，内部改为新格式，减少 call-site churn。

### 2. Frontend Reverse Parsing

更新 reverse parser：

- 新增 `AI Desk @skill "...": get_platform_entities(id='...', kind='skill')`
- 保留所有 legacy `get_platform_skill` parser
- 输出 standalone Skill marker：

```text
[[vibeSkill|aidesk|<skillName>|id=<skillId>]]
```

不新增 `promptMention|skill` marker。

### 3. Frontend Display / ID Extraction

更新：

- agent stdin display restore
- `extractPlatformSkillIdsFromExpandedPrompt`

要求新旧 prompt 都能还原 `/skill` 和提取 skill id。

### 4. Python MCP Tool Schema

更新 `get_platform_entities` tool definition：

- description 从 only skillSet 改为 skill and skillSet
- `kind.enum = ["skill", "skillSet"]`
- `id` description 覆盖 skills.id 和 skill_sets.id
- `skillId` description 明确 only for `kind='skillSet'`

### 5. Python Skill Entity Bundle

在 SkillService 或相邻公共 helper 中新增 standalone Skill entity bundle builder。

要求返回：

- `metadata.kind = "skill"`
- `metadata.id`
- `metadata.name`
- `metadata.description`
- `metadata.version`
- `materialization.type = "archive"`
- `materialization.entryFile = .../SKILL.md`
- `agentInstructions`

应复用现有 Skill visibility 判断和 legacy bundle file assembly 逻辑，避免两套不同的 SKILL.md fallback。

### 6. Python Archive Support

更新 platform entity archive endpoint：

- `kind='skill'` dispatch 到 Skill archive builder
- `kind='skillSet'` 现有行为不变
- 继续使用 stateless signed token
- 重新查询 DB 并校验 permission/version

Skill archive builder 需要：

- 按 persisted Skill files 生成 zip
- 缺少 `SKILL.md` 时从 Skill content synthesize
- path 校验 no absolute path / no `..`
- duplicate path check

### 7. Python MCP Handler

更新 `handle_get_platform_entities`：

- `kind='skill'` 解析 `id` 为 Skill UUID
- 调 Skill entity bundle builder
- 创建 `kind='skill'` archive token
- 写入 `materialization.archiveUrl`
- 记录 structured logs
- `kind='skillSet'` 现有逻辑不变

`get_platform_skill` handler 保持 legacy 行为。

### 8. Backend Reference Parsers

更新所有从 prompt 文本提取 skill id 的 backend parser：

- asset usage stats
- task skill reference refresh
- system remote runtime skill materialization
- any runtime bridge parser that only recognizes `get_platform_skill`

要求同时支持：

```text
get_platform_entities(id='<skillId>', kind='skill')
get_platform_skill(skill_id='<skillId>')
```

Skill Set usage parser 仍只把 `kind='skillSet'` 计入 Skill Set reference。

### 9. Gleam Backend Parity

更新 Gleam backend：

- MCP tools schema advertises `kind = skill | skillSet`
- `get_platform_entities` handler supports `kind='skill'`
- platform entity archive route supports `kind='skill'`
- legacy `get_platform_skill` remains unchanged

如果 Python 和 Gleam 对 archive/file assembly 已有差异，优先抽出同等规则而不是引入新 contract 差异。

### 10. Tests

Frontend tests：

- new standalone Skill expansion
- new standalone Skill reverse parsing
- legacy reverse parsing still works
- display restore for new prompt
- id extraction for new prompt
- Skill Set tests unchanged

Python tests：

- MCP tool schema includes `skill`
- `get_platform_entities(kind='skill')` success
- archive download success
- permission failure
- deleted/not found failure
- legacy `get_platform_skill` unchanged
- usage extraction detects new skill prompt
- system runtime extraction detects new skill prompt

Gleam tests：

- tool schema includes `skill`
- handler accepts `kind='skill'`
- archive route accepts `kind='skill'`
- legacy tests unchanged

### 11. Verification

Focused commands before done:

```bash
cd frontend && yarn test slash-prompt-skill-marker-contract
cd frontend && yarn typecheck
cd backend && pytest tests/test_mcp_skill_set_entities.py tests/test_skill_service_usage.py tests/test_asset_usage_stats_service.py tests/test_system_remote_runtime_skills.py
cd backend_gleam && gleam test
```

If exact test command names differ, use the closest focused tests and record the command used.

Manual smoke:

- Insert standalone Skill in chat prompt, inspect expanded prompt uses `AI Desk @skill ... kind='skill'`
- Existing workflow with old `get_platform_skill` prompt opens as `/skill` chip
- Edit old workflow prompt and save, stored prompt becomes new canonical format
- Runtime calls `get_platform_entities(kind='skill')` and receives `agentInstructions`
- Archive URL downloads zip containing `SKILL.md`
- Skill Set prompt behavior remains unchanged

## Rollback

Rollback is straightforward because legacy `get_platform_skill` remains available.

If new `kind='skill'` path fails in production:

- revert frontend formatter to legacy prompt
- keep backend `kind='skill'` support harmlessly deployed
- existing new prompts may fail unless backend remains deployed, so do not remove backend support after rollout

## QA Recommendation

推荐 QA Agent `full`。

原因：

- 触达 runtime-facing prompt contract
- 触达 MCP tool schema 和 archive materialization
- 涉及 Python/Gleam backend parity
- 影响 workflow stored prompt compatibility
- 影响 task/chat runtime skill id extraction
