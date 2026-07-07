# 03 版本计算规则

更新时间：2026-07-06

## 目标

Standalone Skill 的 version 规则与 Skill Set 保持一致：

- create/import 默认 `1.0.0`
- save metadata / save file / save content bump PATCH
- sync changed bump MINOR
- sync noop 不变

## Version 格式

格式：

```text
<major>.<minor>.<patch>
```

示例：

```text
1.0.0
1.0.1
1.1.0
```

本期不定义 MAJOR bump。所有 backend helper 必须允许未来扩展 MAJOR，但当前业务不会触发。

## 默认版本

以下情况逻辑版本均为 `1.0.0`：

- `skills.config is null`
- `skills.config` 不是 object
- `skills.config.version` 缺失
- `skills.config.version` 为空字符串
- `skills.config.version` 不符合 `x.y.z`

默认版本只在 response 和 bump 计算中生效，不要求 migration 立即写回所有旧数据。

## PATCH bump

触发场景：

- 用户手动保存 Skill metadata
- 用户手动保存 `SKILL.md`
- 用户新增、修改、删除 supporting files
- Runtime/local folder/manual import 后再次普通保存

计算：

```text
1.0.0 -> 1.0.1
1.0.31 -> 1.0.32
bad -> 1.0.1
missing -> 1.0.1
```

由于当前 Skill update API 是 full replace，后端可以在 PUT 成功时统一 bump PATCH，不需要在本期细分 metadata-only 和 content-only endpoint。

## MINOR bump

触发场景：

- URL origin sync 发现 remote bundle 和当前 bundle 不同，并成功写入 DB
- Local folder sync 如果改成独立 sync endpoint，发现 source bundle 和当前 bundle 不同，并成功写入 DB

计算：

```text
1.0.0 -> 1.1.0
1.0.31 -> 1.1.0
1.7.2 -> 1.8.0
bad -> 1.1.0
missing -> 1.1.0
```

## Sync Noop

sync no-op 不更新 version。

判断规则沿用现有 Skill sync：

- 计算当前 Skill bundle hash
- 计算 origin materialized bundle hash
- hash 相同则返回 `changed=false`
- 不更新 `skills.updated_at`
- 不更新 `skills.config.version`
- UI 显示 “Already up to date” 或当前后端返回 message

这样可以避免用户反复点击 sync 产生没有内容差异的新版本。

## Config Merge

写入 version 时必须保留原有 config。

输入：

```json
{
  "origin": {
    "type": "github",
    "source_url": "https://github.com/org/repo"
  },
  "debugSeedBatchId": "batch-1"
}
```

PATCH 后：

```json
{
  "version": "1.0.1",
  "origin": {
    "type": "github",
    "source_url": "https://github.com/org/repo"
  },
  "debugSeedBatchId": "batch-1"
}
```

禁止用新 config 覆盖掉 `origin`。

## Client Config

前端创建 Skill 时可能传入 `config.origin`：

- runtime import
- local folder import
- URL import 由后端生成 origin

后端必须接受这些 origin 字段，但应覆盖 `config.version` 为 `1.0.0`。客户端传入的 version 不可信。

## Skill Set Sub Skill

本期 version 只面向 standalone Skill。

`skill_set_id is not null` 的 sub skill 不单独维护 `config.version`。它的版本来自 parent Skill Set 的 `skill_sets.version`。原因：

- Skill Set sync/save 是 bundle-level 操作
- sub skill 不应显示或独立 bump users/tasks/version
- 避免 parent version 和 child version 出现 drift

后端 helper 在 list/detail standalone Skill 时使用 `config.version`。Skill Set detail 中的 sub skill summary 不需要新增 child version。

