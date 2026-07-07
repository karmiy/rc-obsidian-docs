# 01 产品范围

更新时间：2026-07-07

## 目标

为 `/agents` 页面保存的 Agent 资产增加 AIDesk 维护的 `version`。

这个版本号用于标识 Agent 定义是否发生过保存变更，覆盖这些配置区：

- Instructions
- Skills
- Variable
- Agent metadata

## 用户可见行为

- 新建 Agent 默认版本为 `1.0.0`。
- 旧 Agent 数据补齐或默认视为 `1.0.0`。
- 每次成功保存可编辑配置时，只递增第三位 patch：
  - `1.0.0 -> 1.0.1`
  - `1.0.9 -> 1.0.10`
  - `2.3.4 -> 2.3.5`
- 本需求不定义 Agent 的第二位 minor 或第一位 major 递增规则。

## 需要递增版本的保存

这些 tab 的保存当前都走 `PATCH /api/v1/agents/{agentId}`；保存成功后都应递增 patch：

- Instructions tab: `instructions`
- Skills tab: `skills`
- Variable tab: `environment`, `environmentVisibility`
- Agent metadata tab: `name`, `description`, `visibility`, `avatarBase64`

版本递增由后端负责。前端不计算下一版本。

## 不递增版本的场景

- `POST /agents` 创建时返回 `1.0.0`，不会立即变成 `1.0.1`。
- `POST /agents/{agentId}/install`
- `DELETE /agents/{agentId}/install`
- `DELETE /agents/{agentId}` 软删除
- `install_count`、`task_definition_count` 等 usage/count 刷新
- list/detail/admin 等只读查询

## UI 展示

在 `/agents` 详情页展示后端返回的 Agent version。

固定展示位置：

- Agent metadata tab 内。
- 放在 `Visibility` 上面。
- 单独一行，label 为 `Version`，value 展示为 `v{agent.version}`。

UI 只消费 API 返回的 `agent.version`。不要从 `updatedAt`、本地状态或前端保存次数推导版本。

## 非目标

本需求不做：

- Agent 历史版本表
- 回滚到旧版本
- 每个 tab 或字段单独维护版本
- minor/major 递增规则
- runtime/daemon 版本逻辑
- 基于版本号改变 task execution 行为

本期 version 只是 persisted Agent 当前态的轻量 metadata。
