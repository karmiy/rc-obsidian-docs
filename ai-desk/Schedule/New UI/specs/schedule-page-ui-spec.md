# Schedule Page New UI Implementation Specs

Design mock:

- [scheduler-redesign.html](../scheduler-redesign.html)
- GitHub Pages: https://karmiy.github.io/rc-obsidian-docs/ai-desk/Schedule/New%20UI/scheduler-redesign.html

## Scope

重构 AI Desk Scheduler page。UI 按设计稿实现；开发时优先复用 AI Desk 已有组件。已有组件不满足但属于通用控件的，先封装成可复用组件，不在 Scheduler 页面里写一次性样式。

需要覆盖：

- schedule list / empty state
- create panel
- edit/detail panel
- pause / resume / run now
- frequency config，包括 custom
- notifications switch
- previous runs 跳转 free chat task
- save dirty patch

## UI Rule

UI 不重新定义。以设计稿为准。

开发时重点检查并复用：

- button / icon button
- search input
- select trigger
- popover/menu
- switch
- runtime selector
- intelligence selector
- sidebar task unread local cursor

## Data Model

Schedule 本体状态和 run/task 状态必须分开。

```ts
type ScheduleStatus = "enabled" | "disabled" | "archived";
type LatestRunStatus = "not_started" | "running" | "succeeded" | "failed" | "paused" | null;
type LatestTaskStatus = string | null;
```

不要再用 latest run/task status 覆盖 `schedule.status`。

推荐返回：

```json
{
  "status": "enabled",
  "latestRunStatus": "running",
  "latestTaskStatus": "running"
}
```

## API Reuse

优先复用现有接口：

- `GET /api/v1/schedules`
- `GET /api/v1/schedules/{id}`
- `POST /api/v1/schedules`
- `DELETE /api/v1/schedules/{id}`
- `POST /tasks/{task_id}/actions` with `archive`
- schedule detail 里的 `runs`
- run result 里的 `taskId` / `goalId`
- sidebar unread localStorage cursor

## API Gaps

需要补或调整：

### Pause Schedule

新增薄接口：

```http
POST /api/v1/schedules/{id}/pause
```

行为：

- 改 schedule 本体为 disabled/paused 语义。
- 同步 Temporal schedule pause/disable。
- 不 pause 已经创建出来的 task。

### Resume Schedule

新增薄接口：

```http
POST /api/v1/schedules/{id}/resume
```

行为：

- 改 schedule 本体为 enabled。
- 同步 Temporal schedule resume/enable。
- 不 resume 历史 task。

### Run Now

如果现有 manual/event trigger 不适合前端直接调用，补薄接口：

```http
POST /api/v1/schedules/{id}/run
```

行为：

- 立即创建一次 schedule run。
- 正常创建 free chat task / agent goal。
- run result 需要能拿到 `taskId`。

### Edit Save

现有 full patch 如果不好用，补 partial patch：

```http
PATCH /api/v1/schedules/{id}
```

要求：

- 支持只更新 dirty fields。
- 不因为历史 latest run/task 已 started 就拒绝编辑 schedule config。
- 保存后重新 register/update Temporal schedule。

### Previous Runs Contract

每次 schedule 触发后都会创建 run。run result 必须暴露可跳转字段：

```json
{
  "taskId": "...",
  "goalId": "...",
  "taskTitle": "..."
}
```

前端 Previous runs：

- list 展示 schedule runs，每条 run 绑定一个 free chat task。
- 点击 run row/title 跳转对应 free chat task。
- hover run item 时显示单条 `...` icon button，不显示跳转箭头。
- 单条 `...` menu：
  - Mark as read / Mark as unread
  - Archive / Unarchive
- Previous runs section 右侧 `...` menu：
  - Mark all as read
  - Archive all
- read/unread：复用 sidebar localStorage read cursor。
- archive/unarchive：复用现有 task archive/unarchive 能力；缺少 unarchive 时补薄 action。
- batch 操作第一版可以前端循环调用；后续可补 batch API。

### Notifications

设计稿里 Notifications 是 switch。

行为：

- enabled：schedule 到点触发并创建 task 后，调用现有 bot/Glip notification。
- notification 内容包含 schedule 名称和 task link。
- disabled：不发通知。
- 不需要 All runs / Failures only。

## Frequency Contract

简单 repeat 可转 cron：

- Hourly
- Daily
- Weekdays
- Weekly

Custom 通过扩展 `triggerConfig` JSON，不新增 DB column。

示例：

```json
{
  "kind": "custom",
  "frequency": "weekly",
  "interval": 2,
  "daysOfWeek": ["MO", "TU", "WE", "TH", "FR"],
  "time": "09:00",
  "timezone": "Asia/Shanghai",
  "anchorDate": "2026-07-20"
}
```

后端需要：

- validate / normalize custom triggerConfig
- calculate next fire time
- register/update Temporal schedule
- Temporal callback 时按 `interval + anchorDate` 做 due check

建议实现：

- Temporal 负责按候选 calendar time 触发。
- AI Desk 后端判断这次是否符合 custom interval。
- 符合才创建 schedule run/task。

## Frontend Mapping

Create payload：

- `name`: title
- `description`: optional
- `prompt`: prompt textarea
- `triggerType`: cron/custom/manual/event 等现有枚举兼容
- `triggerConfig`: frequency config
- `taskConfig.runtime`: Runtime selector
- `taskConfig.intelligence`: Intelligence selector
- `taskConfig.projectId`: Project selector
- `taskConfig.notificationsEnabled`: switch

Edit payload：

- 只提交 dirty fields。
- 保存成功后刷新当前 schedule detail 和 list。

## Acceptance

- Create 不产生假的 list item；保存成功后才进入 list。
- Edit 后能保存 title/prompt/project/runtime/intelligence/frequency/notifications。
- Pause/resume 只控制 schedule，不控制历史 task。
- Run now 会创建一条 run，并能从 Previous runs 跳到 free chat task。
- Schedule status 和 latest run/task status 分开展示。
- Previous runs row 点击进入 free chat task；row hover 显示单条 `...` menu。
- Previous runs 支持单条 read/unread、archive/unarchive 和批量 mark read/archive。
- Notifications enabled 时，触发后能给用户发 bot/Glip 消息。
- Custom 支持 `Every = N` 和 weekdays 多选。
- 每 2 周工作日 9 点这类 custom 能正确触发。
- UI 与设计稿一致，并复用/封装 AI Desk 公共组件。
