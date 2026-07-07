# 01 产品与 UI 方案

更新时间：2026-07-01

## 目标

在现有 Settings -> Skills 页面中增加 Skill Sets 管理能力。Skill Set 是一组 skills 的集合，用户可以安装/卸载整个 set，也可以进入 set detail 查看和编辑 set bundle。

## 页面入口

Skills 页面增加顶部 tab：

- `Skills`
- `Skill Sets`

进入 `Skill Sets` tab 后：

- 搜索框用于搜索 set name / description。
- 过滤和排序样式沿用现有 Skills 页面。
- 右上角按钮变为 `New Skill Set`。

## Skill Set Card 样式

Card 布局参考现有 Skill item。

Card 展示：

- set icon
- set name
- description
- creator
- `{x} users used`
- `{x} tasks used`
- `{x} skills`

Card 不展示：

- `installed` tag
- `2 updated` tag

Card 右侧 action 和现有 Skill item 一致：

- 未安装：`Install`
- 已安装：`Installed`
- 已安装 hover：`Uninstall`

点击 card 非按钮区域进入 Skill Set detail。

## New Skill Set 弹框

Dialog 提供两个 import 入口：

- `Import from URL`
- `Import from Folder`

URL / Folder 导入都必须是 preview-first：

1. 用户输入 URL 或选择 folder。
2. 后端生成 import preview。
3. New Skill Set dialog 显示解析完成状态，并展示 metadata form，让用户确认或修改 `name`、`description`、`visibility` 等字段。
4. metadata form 可以使用后端解析出的结果预填，例如从 source README、repo name、source metadata 推断 `name` 和 `description`。
5. Dialog 同时显示识别到的 skill 数量和文件数量，并提供 `Review files` 按钮打开二级 preview 弹框。
6. 二级 preview 弹框复用 `Set Content` 的 file tree/editor 组件，但内容只读。
7. file tree 每个节点左侧有 checkbox，默认全选。
8. 用户取消勾选的文件或目录不进入 confirm payload，也不保存入库。
9. 用户确认后才保存入库，入库 metadata 以 form 当前值为准。

二级 preview 弹框行为：

- 左侧是可勾选 file tree。
- 右侧显示当前选中文件内容，read-only。
- 勾选目录等于勾选目录下全部文件。
- 取消勾选目录等于排除目录下全部文件。
- 必须保留 `README.md` 和至少一个 `skills/<skill>/SKILL.md`，否则 confirm 按钮 disabled 并提示无效。
- 空 preview bundle 可以打开预览，让用户看到 source 无有效内容。

## Skill Set 详情页

Detail 页面复用现有 Skill detail 的页面骨架：

- 顶部 back button
- icon + title
- visibility chip
- tabs

Tabs 改为：

- `Set Content`
- `Set Metadata`

## Set Content Tab

`Set Content` 复用现有 Skill Content 的 file tree / editor 体验，但数据源是 Skill Set bundle。

展示结构：

```text
README.md
assets/
  avatar.png
skills/
  brainstorming/
    SKILL.md
    references/
    scripts/
  systematic-debugging/
    SKILL.md
```

数据来源：

- `skills/` 外文件来自 `SkillSetBundleFile`
- `skills/` 下文件来自 `Skill` / `SkillFile`
- `manifest.json` 不在 UI 中展示

Save 行为：

- `Set Content` 的 Save 只保存完整 content file tree。
- 不保存 metadata 字段。
- 不做单文件保存接口。

## Set Metadata Tab

`Set Metadata` 复用现有 Skill Metadata 的表单风格。

字段：

- name
- description
- creator
- created
- updated
- version
- visibility
- origin

Actions：

- `Sync from origin`
- `Save`
- `Download`
- `Delete`

Save 行为：

- `Set Metadata` 的 Save 只保存 metadata 字段。
- 不保存 Set Content 文件。

Sync 行为：

- 点击 `Sync from origin` 后，重新从 origin 生成 sync preview。
- 如果 origin 是 URL，前端直接请求后端 sync preview，由后端按 `origin_url` 下载并转换。
- 如果 origin 是 local folder，前端先按 metadata 中保存的 folder path 调用本地 daemon 重新读取该文件夹，再把读取到的 source file tree 提交给后端 sync preview；后端不直接读取用户机器上的本地路径。
- Sync preview 不直接更新 DB。
- Sync preview 使用和 import preview 相同的 preview 组件。
- Sync preview 主弹框也展示 metadata form，默认用重新解析出的 metadata 预填；用户确认前可以修改。
- 用户可以点击 `Review files` 打开二级只读 file tree 弹框。
- 用户取消勾选的文件或目录不进入 sync confirm payload。
- 用户确认后，后端用用户勾选后的 bundle 全量替换当前 Skill Set 的旧内容。
- Sync replacement 是物理替换：旧的 `SkillSetBundleFile`、owned `Skill`、owned `SkillFile` 数据删除后重新写入。
- Sync confirm 成功后统一 bump MINOR，例如 `1.0.31 -> 1.1.0`。

## 组件复用

实现前需要抽象现有 Skill detail 中可复用能力：

- detail shell / header / tabs
- file tree
- file editor
- add/delete file dialog
- metadata field/action layout
- visibility selector
- delete/sync confirmation dialog pattern
- `SkillSetBundlePreviewDialog`：import preview 和 sync preview 共用；内部复用 Set Content 的只读 file tree/editor，并支持 checkbox 选择文件

Skill 和 Skill Set 分别提供自己的 data adapter，不在组件里写业务判断。

## Usage Stats 统计

Set card usage tags 参考现有 Skill：

- `{x} users used`：参考 Skill 的 `install_count` 更新链路；Skill Set 可由 `SkillSetInstall` 聚合或冗余到 `SkillSet.install_count`。
- `{x} tasks used`：参考 Skill 的 `task_definition_count` / usage stats 更新链路；通过 `TaskSkillSetReference` 记录 task 对 set 的引用，再刷新 `SkillSet.task_definition_count`。
- set 内 sub skill 不单独统计 users/tasks。用户无论选择 `@Superpowers` 还是 `/Superpowers.Brainstorming`，都只累计到 `Superpowers` 这个 set。
- `{x} skills`：统计 `skills where skill_set_id = set_id and deleted_at is null`。

## Delete / Uninstall 语义

- `Uninstall`：只取消当前用户安装关系，不删除 set 内容。
- `Delete`：删除 Skill Set 资产本身，采用 soft delete。

Delete 后：

- Skill Set list 不再显示。
- Composer 不再能选择。
- MCP 不再能返回该 set。
