# 02 数据模型与 API 方案

更新时间：2026-07-06

## 目标

在不新增数据库列的前提下，为 standalone Skill 暴露稳定 version 字段，并保持旧数据兼容。

## 数据模型

### skills.config

继续使用现有 `skills.config JSONB` 字段保存 Skill metadata。

新增字段：

```json
{
  "version": "1.0.0"
}
```

完整示例：

```json
{
  "version": "1.0.0",
  "origin": {
    "type": "github",
    "source_url": "https://github.com/org/repo",
    "root_path": "skills/example",
    "source_name": "example"
  }
}
```

设计原则：

- `config.version` 归 AIDesk backend 维护
- 前端请求体中的 `config.version` 不作为可信版本来源
- 写入 version 时必须保留 `config` 中已有的其他字段，例如 `origin`、`debugSeedBatchId`
- 旧数据 `config is null` 或缺少 `version` 时，逻辑版本为 `1.0.0`
- 非法 version 字符串也按 `1.0.0` 处理

不新增独立 DB column 的原因：

- 当前 Skill 已经用 `config` 保存 origin/import metadata
- version 只是当前 Skill asset 的轻量 metadata
- 旧数据无需 backfill migration
- 减少 Python/Gleam schema migration 风险

## 后端 Helper

必须新增集中 helper，禁止在代码中散落：

```python
config.get("version") or "1.0.0"
```

Python backend helper 建议放在 Skill service 旁边或独立 utility 中：

- `get_skill_version(config: object) -> str`
- `with_skill_version(config: object, version: str) -> dict`
- `bump_skill_patch(config: object) -> dict`
- `bump_skill_minor(config: object) -> dict`

规则：

- 合法格式：`major.minor.patch`，三个 segment 都是非负整数
- `get_skill_version(None) == "1.0.0"`
- `get_skill_version({"version": "bad"}) == "1.0.0"`
- `bump_skill_patch(None)` 写入 `1.0.1`
- `bump_skill_minor(None)` 写入 `1.1.0`
- `bump_skill_minor({"version": "1.0.7"})` 写入 `1.1.0`

Gleam backend 需要实现同等 helper 或同等 SQL/JSONB merge 规则。

## API Response

后端 response 必须直接返回 `version` 字段，前端不需要从 `config` 推断。

更新 schema：

- `SkillSummaryRead.version: str`
- `SkillDetailRead.version: str`

TypeScript 更新：

- `SkillSummary.version: string`
- `SkillDetail.version: string`
- `MockSkillSummary.version?: string`

所有 response 中的 `version` 都来自后端 `get_skill_version(skill.config)`。

## API Request

现有 request body 不需要新增必填字段。

### POST /skills

创建 Skill 时：

- 如果 `payload.config` 为空，写入 `{"version": "1.0.0"}`
- 如果 `payload.config` 有 origin 等字段，merge 写入 `version = "1.0.0"`
- 不信任客户端传入的 `config.version`

### PUT /skills/{skill_id}

普通保存 Skill 时：

- 基于 DB 当前 `skill.config` bump PATCH
- 更新 name、description、content、visibility、team_id、files 后写回 config
- 如果 DB 当前无 version，则本次保存后为 `1.0.1`

当前 PUT 是 full replace editable fields，因此 metadata 和 content/file 保存都会经过该接口。只要保存成功，就 bump PATCH。

### POST /skills/{skill_id}/sync

URL origin sync 时：

- 先保留当前 bundle hash 比较行为
- 如果 current hash == remote hash，返回 `changed=false`，不写 DB，不 bump version
- 如果有变化，更新 bundle 并 bump MINOR
- 如果当前无 version，则 sync changed 后为 `1.1.0`

### Local Folder Sync

当前 local folder sync 由前端读取本地 folder 后调用普通 update，因此会被识别为 PATCH。

如果需要满足“sync 整个 skill 就加第二位”，应新增一种后端可识别的 sync path：

推荐方案：

```text
POST /skills/{skill_id}/sync-local
```

或在已有 update payload 中增加后端可信的 operation mode：

```json
{
  "versionBump": "minor"
}
```

更推荐独立 endpoint，因为普通保存和 sync 是不同业务语义，独立 endpoint 更容易做 noop diff、日志和测试。

## OpenAPI

如果仓库存在 `/skills` OpenAPI spec，需要同步补充：

- `SkillSummaryRead.version`
- `SkillDetailRead.version`
- sync response 中 nested skill 的 version

如果当前没有独立 skills OpenAPI 文件，应在实现说明中记录 contract 由 Pydantic schema 和 frontend api types 对齐。

