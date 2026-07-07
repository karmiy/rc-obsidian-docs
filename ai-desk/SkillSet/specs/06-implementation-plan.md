	# 06 实施计划

更新时间：2026-07-03

## 目标

完整实现 Skill Set Management，覆盖 frontend、Python backend、Gleam backend、MCP、composer、import flow、tests 和 verification。

## 开发顺序

### 1. 数据模型与 Migration

实现：

- `SkillSet`
- `SkillSetBundleFile`
- `SkillSetInstall`
- `TaskSkillSetReference`
- `Skill.skill_set_id`

新增 indexes 和 constraints：

- `SkillSet.deleted_at`
- `SkillSetInstall unique(skill_set_id, extension_id)`
- `TaskSkillSetReference unique(task_id, skill_set_id, source_type, source_id)`
- `Skill.skill_set_id`

Migration 放在根目录：

```text
db_migrations/versions/
```

不要为 Gleam backend 新增独立 migration tree。

### 2. Backend Domain Services

新增 Skill Set service/repository 方法：

- list
- detail
- update metadata
- save content bundle
- install
- uninstall
- soft delete
- generate manifest
- build MCP file bundle

Content save 必须是 transactional。

### 3. Python Backend APIs

新增 endpoints：

```text
GET /skill-sets
GET /skill-sets/{id}
PUT /skill-sets/{id}/metadata
PUT /skill-sets/{id}/content
POST /skill-sets/{id}/install
DELETE /skill-sets/{id}/install
DELETE /skill-sets/{id}
POST /skill-sets/import-url/preview
POST /skill-sets/import-folder/preview
POST /skill-sets/import-url/analyze
POST /skill-sets/import-folder/analyze
GET /skill-sets/import-jobs/{jobId}
POST /skill-sets/import-jobs/{jobId}/cancel
POST /skill-sets/import/confirm
POST /skill-sets/{id}/sync/preview
POST /skill-sets/{id}/sync/confirm
GET /skill-sets/{id}/download
```

同步更新 OpenAPI。

### 4. Gleam Backend Parity

在 `backend_gleam/` 实现一致 contract 和 behavior。

如果某个 endpoint 暂时不能完成，需要明确记录 parity gap、原因和 follow-up，不能把 backend parity 说成完成。

### 5. Import Pipeline

实现 source acquisition：

- URL import / sync：后端让 remote daemon clone GitHub / GitLab repo 到 sandbox `source/`。
- URL 指向 repo subtree 时，Claude Code 的 source path 指向该 subtree；repo 可以完整 clone 到 sandbox 内。
- Local folder import / sync：前端读取本地 folder 后上传 source files，后端通过 remote daemon `write_file_tree` 写入 sandbox `source/`。
- 后端不对 URL source 做 `SKILL.md` / `skills/` / 标准目录结构前置判断。
- 后端不按 source file count / file size 业务限制决定是否允许 Claude 分析。
- 后端只校验 Claude Code 写出的 `output/` bundle。

实现 sandbox：

```text
<system_remote_runtime_cwd>/tmp/skill-set-imports/<job_id>/
```

实现：

- URL source：remote daemon repo clone 写入 sandbox `source/`
- local folder source：remote daemon `write_file_tree` 写入 sandbox `source/`
- system runtime Claude Code execution via remote daemon non-stream `execute_agent_prompt`
- import URL / folder preview job service：start job、poll status、cancel job
- `execute_agent_prompt` 透传稳定 `execution_id`，cancel 时调用 remote daemon cancel execution
- wait for `execute_agent_prompt` HTTP response; response means Claude CLI process has exited
- remote daemon `list_files` / `read_file` 扫描并读取 `output/`
- validation
- preview response
- confirmation persistence
- completion/failure/cancel 都 cleanup via remote daemon `remove_workspace(sandbox_root)`，只删除当前 `<job_id>` 目录

实现 local folder import / sync source acquisition：

- 复用现有 Skill local folder import 的本地 folder picker / daemon 读取能力。
- import folder 时，前端读取用户选择的 folder，上传 source file tree，并在 confirm 后保存 `origin_type = local_folder`、`origin_url = <absolute folder path>`。
- local folder sync 时，前端根据 set metadata 中保存的 `origin_url` 再次调用本地 daemon 读取 folder，并把新的 source file tree 传给 `POST /skill-sets/{id}/sync/preview`。
- 后端不直接读取 `local_folder` 的 `origin_url`，只处理前端上传的 source files；缺少 source files 时返回可展示错误。
- URL import / URL sync 和 local folder import / local folder sync 进入后端后必须共用 Claude Code converter、preview、confirm persistence。

New Skill Set dialog import UI：

- 使用 analyze job endpoints，不再用同步 preview endpoint 等待 Claude 完成。
- 前端 draft store 保存 `jobId`、analyzing/stopping、preview、metadata、selected files。
- 普通关闭 dialog 不清空 draft；review 阶段 footer Cancel 清空 draft。
- Analyzing 期间 primary 按钮 disabled 显示 `Analyzing...`；footer 左侧显示 bordered `Cancel`，点击后显示 `Canceling...` 并调用 cancel endpoint，成功后保持当前 import 页面并清空表单 / draft。

当前快速实现 note：

- Python backend job 状态使用 Redis 共享 store，避免多 BE 下 poll/cancel 打到不同 backend 后找不到 job。
- Redis job value 保存 `jobId`、`status`、`executionId`、`sandboxRoot`、`source`、`preview`、`error`、owner scope 和 timestamps，并设置 15 分钟 TTL。active job 被 poll 时，如果 TTL 低于 5 分钟才续回 15 分钟，避免每 3 秒 poll 都写 Redis。后台 `asyncio.Task` 只作为当前 BE 的本地优化，不作为 poll/cancel 正确性依赖。
- cancel 从 Redis 读取 `executionId` 和 `sandboxRoot`，任意 backend 都可以调用 daemon cancel 并清理 sandbox。worker 写 `completed/failed` 前必须尊重 Redis 里的 terminal status，不能把已经 `cancelled` 的 job 覆盖成 `failed` 或 `completed`。
- Gleam backend parity 需要补齐 analyze job/status/cancel contract。

### 6. MCP

实现 `get_platform_entities`：

```text
kind = skillSet
```

行为：

- return lightweight metadata + archive materialization descriptor
- do not return full files in MCP JSON
- generate `manifest.json` into the archive
- archive includes real utf8 and binary file contents
- include `agentInstructions`
- skillId only changes selected metadata, entry file, and instructions
- add generic `GET /api/v1/platform-entities/archive?token=...` download endpoint
- archive token must be short-lived stateless signed token so multi-BE deployments can validate it without in-memory state

### 7. Composer

扩展 `SlashPromptComposer` 内部逻辑：

- load installed Skill Sets
- load sub skills for installed sets and merge them into the existing `/` skill list
- support `@Superpowers`
- support `/Superpowers.Brainstorming` filtering/highlighting in the current slash skill menu
- add markers
- add marker expansion to MCP prompt

### 8. Frontend UI

先抽象现有 Skill detail 可复用部分：

- detail shell
- file tree/editor
- metadata form layout
- action buttons/dialogs

再实现：

- Skill Sets tab
- Set cards
- New Skill Set dialog
- `SkillSetBundlePreviewDialog`，供 import preview 和 sync preview 共用
- detail page
- Set Content save
- Set Metadata save

### 9. Usage Stats

扩展 stats 逻辑，更新：

- Skill Set users used
- Skill Set tasks used
- Skill count

复用当前 Skill usage stats patterns，但需要新增 Skill Set 自己的引用表和刷新逻辑：

- `SkillSetInstall` -> refresh `SkillSet.install_count`
- `TaskSkillSetReference` -> refresh `SkillSet.task_definition_count`
- 从 composer 展开的 `get_platform_entities(... kind='skillSet')` prompt 中解析 `setId`
- 无论用户选择 `@Superpowers` 还是 `/Superpowers.Brainstorming`，usage 都只累计到 set，不给 sub skill 单独统计 users/tasks

### 10. Tests

Backend tests：

- migrations
- list/detail
- metadata update
- content bundle save
- install/uninstall
- soft delete
- import preview/confirm
- MCP response

Frontend tests：

- Skill Sets tab
- Set card actions
- New Skill Set dialog
- detail tabs
- metadata save
- content save
- composer markers

Import tests：

- URL source without `SKILL.md` still clones to daemon sandbox and reaches Claude Code converter
- source without root `README.md` reaches Claude Code converter
- source with only `skills/` reaches Claude Code converter
- source with no `skills/` but convertible uses Claude Code converter
- ignored paths are filtered by segment: `node_modules`, `node_Modules`, `.git`, `.cache`, `__pycache__`, `dist`, `build`
- invalid paths
- binary file

### 11. Verification

Done 前必须完成：

- Python backend focused tests pass
- Gleam backend focused tests pass
- frontend typecheck/tests pass for touched code
- OpenAPI updated
- local smoke test for import preview
- local smoke test for MCP `get_platform_entities`
- manual or browser verification for Skill Set UI

### 12. QA Recommendation

这是高风险 product/runtime 改动。实现完成后建议 QA Agent `full`，因为它涉及：

- DB schema
- Python/Gleam backend parity
- runtime MCP behavior
- composer prompt expansion
- frontend settings UI
