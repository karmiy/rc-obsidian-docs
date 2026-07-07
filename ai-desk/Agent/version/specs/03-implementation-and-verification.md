# 03 实施与验收

更新时间：2026-07-07

## Feature Boundary Contract

- Feature name: Agent asset version
- FFS key: 无
- Risk level: medium
- Entry points: `/agents`、`/admin/agents`、Gleam Agent catalog routes
- Existing behavior: Agent asset 没有 version 字段
- New behavior: create 返回 `1.0.0`；editable PATCH 递增 patch
- Public API/schema changes: `AgentRead.version`
- Rollback plan: UI 停止展示并停止使用该字段；如果只做 code rollback，DB column 可保留
- Observability needed: 现有 Agents API structured DB-access logs 基本够用；update log 可选记录 `previous_version` 和 `next_version`
- Local verification: backend focused tests、Gleam tests、frontend type/test checks

## 实施计划

### 1. Migration 与 Python Model

- 新增 `agents.version`。
- 更新 `Agent` SQLAlchemy model。
- migration 保持 additive 和 reversible。

### 2. Python Service 与 Schema

- 在 `agent_service.py` 附近或单独 Agent utility 中新增集中 version helper。
- `AgentRead` 增加 `version`。
- 更新 `_to_read` 和 admin `_agent_read`。
- `AgentService.create` 使用 DB/model default，返回 `1.0.0`。
- `AgentService.update` 只在 payload 包含至少一个 service 接受的 editable field 时 bump patch。
- update 失败路径必须保持 DB 当前 version 不变；version bump 和字段更新必须同事务提交。

Editable fields：

```text
name, description, instructions, skills, environment, environment_visibility, avatar_base64, visibility
```

### 3. Gleam Backend Parity

Gleam backend 必须和 Python backend 在同一个 work item 内同步改动。需要新增等价的 Agent version helper/policy，并覆盖 create/update/read contract。

同步更新已有 Agent catalog persistence path，包括：

- `backend_gleam/src/persistence/sql/agent_catalog_repo.gleam`
- `backend_gleam/src/http/runtime_db_routes.gleam` 中仍服务 public Agent API 的 legacy inline Agent SQL
- 如果 debug seed 会创建 Agent row，也要补齐 version/default

Gleam helper 规则必须与 Python helper 一致：

- 默认 `1.0.0`
- 非法 version fallback 到 `1.0.0`
- update 只 bump patch
- create 返回 `1.0.0`

Gleam 尽量用 pure policy/helper 处理 patch bump；如现有路径必须用 SQL expression，也要集中成 helper。不要新增依赖。

### 4. Frontend

- `Agent` 类型增加 `version: string`。
- `/agents` detail 的 Agent metadata tab 中展示 `v{agent.version}`，位置在 `Visibility` 上面单独一行。
- 每次保存后继续使用后端 response 更新本地状态。
- 前端不计算 version。

### 5. Tests

Python backend：

- helper default/invalid/patch cases
- create returns `1.0.0`
- update instructions bumps patch
- failed update preserves current version
- update skills bumps patch
- update environment bumps patch
- update metadata/avatar bumps patch
- install/uninstall/delete do not bump version
- admin list returns version

Gleam backend：

- helper default/invalid/patch cases
- list/detail include version
- create returns `1.0.0`
- update bumps patch
- failed update preserves current version
- install/uninstall preserve version

Frontend：

- type coverage compiles with `Agent.version`
- detail view renders returned version above `Visibility` in Agent metadata tab
- save flow refreshes displayed version from API response

## Acceptance Matrix

| Case | Entry point | Verification | Status |
| --- | --- | --- | --- |
| Create Agent | `POST /agents` | backend test | pending |
| Save Instructions | `PATCH /agents/{id}` | backend test + UI test/smoke | pending |
| Save Skills | `PATCH /agents/{id}` | backend test + UI test/smoke | pending |
| Save Variable | `PATCH /agents/{id}` | backend test + UI test/smoke | pending |
| Save Agent metadata | `PATCH /agents/{id}` | backend test + UI test/smoke | pending |
| Failed update | `PATCH /agents/{id}` | backend test | pending |
| Install/uninstall | install endpoints | backend test | pending |
| Admin list | `GET /admin/agents` | backend test | pending |
| Existing row | migration/default | migration or model test | pending |
| Python/Gleam parity | both backends | focused tests | pending |

## 建议验证命令

如果实现时测试文件名变化，使用对应 touched module 的 focused tests。

```bash
cd backend && venv/bin/python -m pytest tests/test_agent_service_permissions.py tests/test_agents_observability.py
cd backend && venv/bin/python -m pytest tests/test_alembic_migrations.py
cd backend_gleam && gleam test
cd frontend && yarn typecheck
```

Manual smoke：

- create Agent shows `v1.0.0`
- save Instructions shows `v1.0.1`
- add/remove Skill shows next patch
- save Variable shows next patch
- save Agent metadata/avatar shows next patch
- install/uninstall does not change version

## QA Recommendation

推荐 QA Agent `standard`。

原因：本需求改变 product API contract、frontend display、DB schema、Python/Gleam backend parity；不改变 runtime execution、auth、task execution 或 daemon 行为。因此除非实现扩大到 runtime-facing contract，否则不需要 `full`。
