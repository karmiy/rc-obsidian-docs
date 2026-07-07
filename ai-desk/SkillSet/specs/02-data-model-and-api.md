# 02 数据模型与 API 方案

更新时间：2026-07-03

## 目标

新增 Skill Set 持久化模型和后端 API。Skill Set 作为独立资产存在，拥有 set-level files 和 owned sub skills。

## 数据表

### SkillSet

保存 set metadata。

字段：

- `id`
- `name`
- `description`
- `version`
- `visibility`
- `creator`
- `origin_type`
- `origin_url`
- `install_count`
- `task_definition_count`
- `created_at`
- `updated_at`
- `last_synced_at`
- `deleted_at`

`version` 由后端维护，初始值 `1.0.0`。

不单独保存 `origin_root_path`。如果导入来源本身包含子目录，后端应把它保留在规范化后的 `origin_url` / origin metadata 中，避免一个来源被拆成两个可能 drift 的字段。

`origin_type` 取值：

- `url`：`origin_url` 保存用户输入的 source URL。
- `local_folder`：`origin_url` 保存用户通过本地 folder picker 选择的绝对文件夹路径。

`local_folder` 的 `origin_url` 只作为 origin metadata 和后续 sync 的本地读取线索。生产环境 backend / remote daemon 不应假设自己能直接访问用户机器上的这个路径；local folder sync 必须由前端通过本地 daemon 重新读取该路径，上传 source file tree 后，再由后端走统一 import converter。

### SkillSetBundleFile

保存 set-level files，不保存 sub skill files。

字段：

- `id`
- `skill_set_id`
- `path`
- `mime_type`
- `encoding`: `utf8` / `base64`
- `content_text`
- `content_base64`
- `size_bytes`
- `sha256`
- `created_at`
- `updated_at`

示例：

- `README.md`
- `assets/avatar.png`
- `references/overview.md`

不包含：

- `manifest.json`
- `skills/<skill>/**`

### SkillSetInstall

保存用户个人安装关系。

字段：

- `id`
- `skill_set_id`
- `extension_id`
- `account_id`
- `created_at`
- `updated_at`

约束：

- `unique(skill_set_id, extension_id)`

### TaskSkillSetReference

保存 task 对 Skill Set 的引用，用于统计 set 级别的 `{x} tasks used`。

字段：

- `id`
- `task_id`
- `skill_set_id`
- `source_type`
- `source_id`
- `created_at`
- `updated_at`

约束：

- `unique(task_id, skill_set_id, source_type, source_id)`

索引：

- `task_id`
- `skill_set_id`

语义：

- 用户选择 `@Superpowers`：写入 `skill_set_id = Superpowers`
- 用户选择 `/Superpowers.Brainstorming`：同样只写入 `skill_set_id = Superpowers`
- `SkillSet.task_definition_count` 来自 `count(distinct task_id)` 聚合
- set 内 sub skill 不单独统计 users/tasks
- 该表独立于现有 `task_skill_references`，因为 Skill Set usage 只按 set 维度聚合

### Skill

现有 `Skill` 表新增：

- `skill_set_id nullable`

语义：

- `skill_set_id is null`：standalone skill，显示在普通 Skills tab。
- `skill_set_id is not null`：set sub skill，只显示在对应 Skill Set 中。

不新增：

- `catalog_visibility`
- `skill_set_source_path`

Sub skill bundle path 由 `skill.name` 推导：

```text
skills/<skill.name>/SKILL.md
skills/<skill.name>/references/*
skills/<skill.name>/scripts/*
```

`skill.name` 必须是 safe path segment。

## API 接口

### List Skill Sets

```text
GET /skill-sets
```

返回 card 数据：

- id
- name
- description
- version
- visibility
- creator
- installed
- install_count / users used
- task_definition_count / tasks used
- skill_count
- updated_at

只返回 `deleted_at is null`。

### Skill Set Detail

```text
GET /skill-sets/{id}
```

返回：

- metadata
- content file tree

content tree 由后端拼接：

- 非 `skills/` 路径来自 `SkillSetBundleFile`
- `skills/<skill>/**` 路径来自 `Skill` / `SkillFile`

不返回 `manifest.json`。

### Update Metadata

```text
PUT /skill-sets/{id}/metadata
```

只更新 metadata 字段：

- name
- description
- visibility
- origin metadata，如需要

不更新文件。

### Save Content Bundle

```text
PUT /skill-sets/{id}/content
```

Request body 包含完整可见 file tree：

```json
{
  "files": [
    {
      "path": "README.md",
      "encoding": "utf8",
      "content": "..."
    },
    {
      "path": "skills/brainstorming/SKILL.md",
      "encoding": "utf8",
      "content": "..."
    }
  ]
}
```

后端按 path 拆分：

- `skills/<skill>/*` -> `Skill` / `SkillFile`
- 其他路径 -> `SkillSetBundleFile`

Save 是 bundle-level 且必须在 transaction 内完成。不做单文件 save API。

这里的 diff 只用于用户在 `Set Content` 手动保存时的后端内部优化：可以按 path/hash 判断哪些文件需要更新，避免大 set 每次全量重写。它不适用于 sync confirm。

### Sync Preview

```text
POST /skill-sets/{id}/sync/preview
```

重新从 origin 生成标准 bundle，返回 preview，不更新 DB。

行为按 `origin_type` 区分：

- `url`：后端使用 `origin_url` 下载 source tree，然后走统一 import converter。
- `local_folder`：前端先根据 DB 返回的 `origin_url` 调用本地 daemon 重新读取用户机器上的 folder，得到 source file tree 后提交给该接口；后端不直接读取本地路径，只处理上传的 source files。

`local_folder` sync preview request 需要携带重新读取到的 source files：

```json
{
  "originPath": "/Users/name/path/to/superpowers",
  "sourceFiles": [
    {
      "path": "README.md",
      "encoding": "utf8",
      "content": "..."
    },
    {
      "path": "skills/brainstorming/SKILL.md",
      "encoding": "utf8",
      "content": "..."
    }
  ]
}
```

如果 `origin_type = local_folder` 但 request 没有携带 `sourceFiles`，后端应拒绝 sync preview，并返回可展示的错误，提示客户端需要重新读取本地 folder。

### Sync Confirm

```text
POST /skill-sets/{id}/sync/confirm
```

Request body 携带用户在 preview 中勾选后的 paths：

```json
{
  "syncJobId": "job-id",
  "metadata": {
    "name": "Superpowers",
    "description": "...",
    "visibility": "public"
  },
  "selectedPaths": [
    "README.md",
    "skills/brainstorming/SKILL.md"
  ]
}
```

确认后不做 diff 合并，直接把 selected bundle 当成新的完整 set 内容，全量物理替换当前 set 内容：

- 物理删除旧 `SkillSetBundleFile`
- 物理删除旧 owned `Skill` 及其 `SkillFile`
- 根据 selected bundle 重新创建 `SkillSetBundleFile`
- 根据 selected bundle 重新创建 owned `Skill` / `SkillFile`
- 保存用户在 preview metadata form 中确认后的 metadata
- 更新 `last_synced_at`
- version 统一 bump MINOR，例如 `1.0.31 -> 1.1.0`

### Import Analyze Jobs

New Skill Set 的 URL / Folder import 使用异步 analyze job，避免前端被长时间 HTTP 请求阻塞：

```text
POST /skill-sets/import-url/analyze
POST /skill-sets/import-folder/analyze
GET /skill-sets/import-jobs/{jobId}
POST /skill-sets/import-jobs/{jobId}/cancel
```

Start response：

```json
{
  "jobId": "skill_set_import_xxx",
  "status": "queued",
  "preview": null,
  "error": null,
  "createdAt": "2026-07-03T00:00:00Z",
  "updatedAt": "2026-07-03T00:00:00Z"
}
```

Status 语义：

- `queued` / `analyzing`：`preview = null`
- `completed`：`preview` 为 `SkillSetImportPreviewRead`
- `failed`：`error` 为可展示错误
- `cancelled`：前端清空 draft，可重新 import

Cancel 行为：

- cancel remote daemon execution
- remove 当前 import job sandbox
- job 标记为 `cancelled`

旧同步 `/import-url/preview`、`/import-folder/preview` endpoint 保留兼容；New Skill Set dialog 使用 analyze job endpoints。

### Install

```text
POST /skill-sets/{id}/install
```

为当前用户创建 `SkillSetInstall`。

### Uninstall

```text
DELETE /skill-sets/{id}/install
```

删除当前用户的 `SkillSetInstall`。

不删除 set 内容。

### Delete

```text
DELETE /skill-sets/{id}
```

Soft delete：

- 设置 `SkillSet.deleted_at`
- 设置 owned sub skills 的 `Skill.deleted_at`
- 删除 `SkillSetInstall`
- 如果未来支持 sub skill 单独 install，再删除 owned sub skills 对应的 `SkillInstall`
- 保留 `SkillSetBundleFile`
- 保留 `SkillFile`

## 校验

Path validation：

- path 不能为空
- path 不能是绝对路径
- path 不能包含 `..`
- path 不能重复
- skill name 不能包含 path separator
- binary file 必须使用 `base64`

Import / sync preview 的后端校验对象是 Claude Code 写出的 `output/` bundle，不是原始 source。

后端不应该因为原始 source 没有 `SKILL.md`、没有 `skills/` 或目录结构非标准而提前拒绝。URL source 应先 clone 到 daemon sandbox，让 Claude Code 负责分析与转换；local folder source 由前端读取后上传，后端写入 daemon sandbox。

对于前端上传的 local folder source，以及 Claude Code 输出的 `output/`，仍需执行路径安全校验和 ignored path 过滤。过滤按 path segment 生效：

- `node_modules` 大小写不敏感
- 以 `.` 开头的 segment
- 以 `__` 开头的 segment
- 常见 generated/cache output，例如 `dist`、`build`、`coverage`、`out`、`target`

Skill Set 必须有：

- `README.md`
- 至少一个 `skills/<skill>/SKILL.md`

## Backend Parity 要求

后端改动要求：

- Python backend implementation
- Gleam backend parity
- migration 只放根目录 `db_migrations/versions/`
- 更新 OpenAPI
- structured logs / local verification commands
