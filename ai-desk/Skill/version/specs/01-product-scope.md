# 01 产品范围

更新时间：2026-07-06

## 目标

为现有 standalone Skill 增加 AIDesk 维护的版本号，让用户和系统能够区分 Skill 的本地保存、metadata 修改和 origin sync 后的版本变化。

Skill 当前没有独立 version 字段。新增版本能力后：

- 新建 Skill 默认版本为 `1.0.0`
- 已有 DB 中没有版本信息的 Skill 视为 `1.0.0`
- 保存 metadata 或内容文件时 bump PATCH，例如 `1.0.0 -> 1.0.1`
- sync origin 且实际内容有变化时 bump MINOR，例如 `1.0.7 -> 1.1.0`
- sync origin 但内容无变化时不 bump version

## 非目标

本期不做：

- Skill 历史版本表
- 回滚到旧版本
- 每个文件独立版本
- migration backfill 所有旧 Skill 的 config
- 改变 Skill Set 的 version 存储方式
- 改变 runtime 使用 Skill 的 prompt/MCP 语义

本期只维护当前版本号。历史内容仍由现有 Git/DB/审计能力或未来单独设计处理。

## UI 展示

Settings -> Skills 中应展示 Skill version。

推荐展示位置：

- Skill list/card metadata 区展示 `v1.0.0`
- Skill detail metadata tab 展示 `Version`
- sync 成功后 UI 使用接口返回的新 version 刷新

前端不得从 `config.version ?? "1.0.0"` 自行散落读取。前端只消费 API 返回的 `version` 字段。`config` 仍可保留用于 origin/source 展示。

## 用户可见行为

用户创建 Skill：

- 保存后返回 `version = "1.0.0"`

用户编辑 Skill 内容或 supporting files：

- 保存成功后返回 patch 后版本
- 如果当前 Skill 是旧数据且没有版本，保存后返回 `1.0.1`

用户编辑 Skill metadata：

- 保存成功后返回 patch 后版本
- metadata 包括 name、description、visibility、team scope 等当前 update 接口会更新的字段

用户 sync URL origin：

- 后端先比较当前 Skill bundle 和 origin bundle
- 如果 bundle hash 无变化，返回 `changed=false` 和当前 version，不写 DB
- 如果 bundle hash 有变化，写入新内容并 bump MINOR

用户 sync local folder origin：

- 当前 local folder sync 由前端通过本地 daemon 读取 folder 后走普通 update
- 如果产品上把这个动作视为 sync，则需要新增显式 sync update path 或 update 参数，让后端区分它和普通保存
- 本期建议将 local folder sync 也走后端 sync 语义：内容有变化时 bump MINOR，无变化时不 bump

## 兼容性

旧 Skill 的 `skills.config` 可能为 `null`，也可能包含：

```json
{
  "origin": {
    "type": "github",
    "source_url": "..."
  }
}
```

新增版本后，规范结构为：

```json
{
  "version": "1.0.0",
  "origin": {
    "type": "github",
    "source_url": "..."
  }
}
```

`version` 是 AIDesk persisted Skill 自身版本，不是 origin/source 的版本，因此放在 `config.version`，不放在 `config.origin.version`。

