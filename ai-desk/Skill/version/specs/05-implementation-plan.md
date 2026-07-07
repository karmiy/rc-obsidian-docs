# 05 实施计划

更新时间：2026-07-06

## 目标

实现 standalone Skill version，覆盖 Python backend、Gleam backend、frontend type/UI、tests 和 verification。

## 开发顺序

### 1. Python Version Helper

新增集中 helper：

- `get_skill_version(config)`
- `with_skill_version(config, version)`
- `bump_skill_patch(config)`
- `bump_skill_minor(config)`

要求：

- helper 覆盖 `None`、非 dict、缺失 version、非法 version
- helper 保留 config 中其他 key
- 不允许在业务代码散落 fallback 字符串逻辑

### 2. Python Schema / Response

更新：

- `SkillSummaryRead.version`
- `SkillDetailRead.version`

所有 list/detail/sync/create/import response 均返回 version。

### 3. Python Create / Import

更新：

- manual create
- URL import single
- URL import batch
- runtime import via POST `/skills`
- local folder import via POST `/skills`
- debug seed skill create 如走同一表，也应写入或被 response helper fallback

规则：

- 创建成功后 version 为 `1.0.0`
- 客户端传入 `config.version` 时后端覆盖为 `1.0.0`
- 保留客户端传入的 `config.origin`

### 4. Python Update

更新 `PUT /skills/{skill_id}`：

- update 成功前基于 DB 当前 config bump PATCH
- 写回 config
- 旧数据首次保存后为 `1.0.1`
- 保留 origin/debug metadata

### 5. Python Sync

更新 URL origin sync：

- hash 相同：返回 `changed=false`，version 不变
- hash 不同：写入 remote bundle，bump MINOR
- 更新 origin root/source metadata 时必须和 version merge 在同一个 config 上完成

Local folder sync 建议新增后端可识别 sync path：

```text
POST /skills/{skill_id}/sync-local
```

该 endpoint 接收前端通过 local daemon 读取后的 bundle，并执行：

- 计算 current hash
- 计算 local source bundle hash
- no-op 不 bump
- changed bump MINOR

如果暂不新增 endpoint，则必须明确记录 gap：local folder sync 仍会走普通 update，因此只能 bump PATCH，尚未满足 sync minor 语义。

### 6. Gleam Backend Parity

更新 `backend_gleam/src/http/runtime_db_routes.gleam`：

- settings skill select columns 返回 `version`
- create settings skill 写 `config.version = "1.0.0"`
- update settings skill bump PATCH
- 旧数据 response fallback `1.0.0`

如果 Gleam 暂无 URL sync endpoint，不需要新增；但已有 create/update/list/detail contract 必须与 Python 对齐。

### 7. Frontend Types / UI

更新：

- `frontend/src/types/skill.ts`
- `frontend/src/components/settings/skills/settingsSkillsTypes.ts`
- `detailToViewModel`
- Settings Skills list/detail 展示 version

前端规则：

- 使用 response `version`
- 不从 `config.version` 做 fallback
- 保存后使用后端返回值刷新 UI

### 8. OpenAPI

检查 openapi 是否已有 skills spec。

如果存在：

- `SkillSummaryRead` 增加 required `version`
- `SkillDetailRead` 增加 required `version`
- sync response nested skill 自动包含 version

如果不存在：

- 在实现说明中记录当前 skills contract 由 Pydantic schema 和 frontend type 对齐

### 9. Tests

Python backend tests：

- `get_skill_version(None) == "1.0.0"`
- invalid version fallback
- patch bump
- minor bump
- create writes/returns `1.0.0`
- existing skill without version response returns `1.0.0`
- update bumps patch and preserves origin
- URL sync changed bumps minor and preserves origin
- URL sync noop does not bump and does not write DB

Gleam backend tests：

- list/detail old config returns `1.0.0`
- create returns `1.0.0`
- update bumps patch
- config merge preserves origin

Frontend tests:

- list/detail renders returned version
- save refreshes version from API response
- sync noop message does not fake a new version

### 10. Verification

Focused commands before done:

```bash
cd backend && pytest tests/test_skill_service.py
cd backend_gleam && gleam test
cd frontend && yarn test <relevant skill tests>
cd frontend && yarn typecheck
```

If exact test files differ, use the closest focused tests for touched modules and report the command used.

Manual/browser smoke:

- create Skill shows `1.0.0`
- save metadata shows `1.0.1`
- save content/file shows next patch
- sync URL changed shows next minor
- sync URL unchanged keeps version
- old Skill without config version displays `1.0.0`

## QA Recommendation

推荐 QA Agent `standard`。

原因：

- 改动影响 Settings Skills 产品行为
- 涉及 Python/Gleam backend parity
- 涉及 API contract 和 frontend display
- 不新增 DB schema，不改变 runtime execution path，风险低于 full

如果实现同时新增 local folder sync endpoint 或 MCP response version，则建议升级到 `full`，因为会触达 runtime-facing contract。

