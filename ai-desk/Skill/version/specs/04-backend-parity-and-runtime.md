# 04 双后端与 Runtime 影响

更新时间：2026-07-06

## 目标

Skill version 是 Settings Skills 的产品契约，必须保持 Python backend 和 Gleam backend 行为一致。Runtime 和 daemon 不应承担版本计算逻辑。

## Python Backend

主要改动文件：

- `backend/app/services/skill_service.py`
- `backend/app/schemas/skill.py`
- `backend/app/api/v1/skills.py`
- `backend/tests/test_skill_service.py` 或现有 skill service 测试文件

Python backend 负责：

- 创建时写 `config.version = "1.0.0"`
- 更新时 bump PATCH
- URL sync changed 时 bump MINOR
- URL sync noop 时不写 DB
- response 中返回 `version`
- 日志中如果需要记录版本，只记录 bounded metadata，不泄漏 raw source URL 以外的敏感内容

## Gleam Backend

当前 Gleam backend 在 `backend_gleam/src/http/runtime_db_routes.gleam` 中实现 Settings skills：

- list settings skills
- create settings skill
- get settings skill
- update settings skill
- install/uninstall/delete
- workspace bundle

Gleam backend 需要同步：

- select response 增加 `version`
- create SQL 对 `config` merge `version = "1.0.0"`
- update SQL 对 DB 当前 `config.version` bump PATCH
- 不要求实现 URL import/sync，如果当前 Gleam 没有对应 endpoint；但已有 Settings CRUD contract 必须 parity

Gleam SQL 建议使用 Postgres JSONB merge：

```sql
coalesce(config, '{}'::jsonb) || jsonb_build_object('version', $version)
```

update bump 需要基于当前 DB config 计算，不应依赖客户端传入 config。

如果在 Gleam 中实现 helper 比 SQL regex 更清晰，可以先读取当前 row 的 config，再在 Gleam 中计算 next config 后 update。

## OpenAPI / Contract

如果 `/skills` endpoints 有 OpenAPI 文件，需要同步 version 字段。

Contract 规则：

- `version` 是 response required 字段
- `version` 不作为 create/update request 的必填字段
- 旧数据 response 也必须有 `version = "1.0.0"`

## Workspace Bundle

现有 AIDesk workspace bundle 依赖：

- skill id
- name
- description
- updated_at
- files

本期不要求把 version 写入 materialized `SKILL.md` 或 workspace bundle payload。

原因：

- runtime 使用 Skill 的行为不依赖 version
- bundle freshness 当前由 `updated_at` / bundle hash 决定
- 写入 version 到 runtime file tree 可能造成不必要的 runtime 行为变化

如果未来希望 runtime cache 按 version 管理 standalone Skill，可以另开设计。

## MCP

`get_platform_skill` 返回 full skill bundle。

本期不改变 MCP tool input/output 的核心语义。可选地在 MCP metadata 中增加 version，但不是本期必须项，除非前端或 runtime 明确需要展示。

要求：

- MCP 读取 Skill 时不能自己推断版本
- 如返回 version，必须使用 backend helper
- 不要让 daemon 读取 DB 或计算 version

## Daemon Boundary

daemon 不参与 Skill version 计算。

local folder import/sync 中：

- 前端可以通过本地 daemon 读取用户机器上的 folder
- 后端负责判断这是普通保存还是 sync，并负责 bump version
- daemon 只提供文件读取能力，不理解 Skill version、origin 或 DB 状态

