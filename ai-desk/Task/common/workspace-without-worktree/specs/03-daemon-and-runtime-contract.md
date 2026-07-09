# 03 Daemon 与 Runtime 契约

更新时间：2026-07-09

## 目标

新增一个通用 daemon filesystem capability，用于创建 symlink。Local no-worktree workspace 由 FE 编排，daemon 只执行通用文件系统操作。

## Daemon Boundary

daemon 必须保持 business-agnostic。

daemon 可以做：

- 创建目录。
- 写文件。
- 创建 symlink。
- 按调用方提供的路径执行 filesystem 操作。

daemon 不可以做：

- 根据 task id 计算 workspace 路径。
- 根据 project id 查询或推导项目真实路径。
- 判断 Flow Task、Chat Task、`local_worktree_enabled` 等业务状态。
- 调用 AI Desk backend 获取 task/project 数据。

## Existing APIs

复用现有 daemon API：

```http
POST /api/v1/workspace
POST /api/v1/file/write
DELETE /api/v1/workspace
```

用途：

- `POST /api/v1/workspace`：创建 workspace root。
- `POST /api/v1/file/write`：写入 `AGENTS.md`。
- `DELETE /api/v1/workspace`：清理 task workspace。

## New Symlink API

新增通用接口：

```http
POST /api/v1/file/symlink
```

请求：

```json
{
  "link_path": "/Users/user/.aidesktop/workspace-task-id/Fiji",
  "target_path": "/Users/user/code/Fiji"
}
```

响应：

```json
{
  "success": true,
  "data": {
    "link_path": "/Users/user/.aidesktop/workspace-task-id/Fiji",
    "target_path": "/Users/user/code/Fiji",
    "created": true
  }
}
```

## Validation Rules

daemon 需要校验：

- `link_path` 必填。
- `target_path` 必填。
- `link_path` 和 `target_path` 都必须在 `allowed_working_dirs` 允许范围内。
- `target_path` 必须存在。
- `link_path` parent directory 必须存在，或由 daemon 安全创建。
- 如果 `link_path` 不存在，创建 symlink。
- 正常 task 创建流程中 `link_path` 应位于全新的 `workspace-{taskId}` 内，因此不应已存在。
- 如果 `link_path` 意外已存在，daemon 不应覆盖；返回失败，错误信息需要可行动。

## Security Scope

如果 daemon security grant 开启，新接口应使用现有 file write scope：

```text
file.write
```

原因：

- 创建 symlink 是 filesystem mutation。
- 不引入新的 AI Desk 业务 scope。

## Runtime Contract

Local runtime execution payload 的 `cwd` 必须是 workspace root：

```text
<worktree-root>/workspace-{taskId}
```

runtime CLI 不需要知道 symlink 是如何创建的。它只看到一个普通 workspace root。

## Remote Runtime Contract

本需求不改变 remote runtime：

- FE 仍通过 backend remote runtime bridge 调 remote daemon。
- backend remote services 继续准备 server-side workspace/worktree。
- remote daemon 不尝试 symlink 到用户本机目录。
- AgentMesh 继续使用自己的 repo mount / pod runtime 机制。
