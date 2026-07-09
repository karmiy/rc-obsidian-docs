# 04 Implementation Plan

更新时间：2026-07-09

## 目标

让 Local runtime 在 `Use worktree` 关闭时也使用 task workspace root 作为 CWD，并通过 symlink 暴露用户真实项目目录。

## 开发顺序

### 1. Daemon Symlink API

在 daemon 中新增通用 symlink endpoint：

```http
POST /api/v1/file/symlink
```

要求：

- 请求体包含 `link_path` 和 `target_path`。
- 路径展开和 allowlist 校验复用现有 file/workspace 规则。
- 正常调用时 `link_path` 位于全新 workspace 内，不应存在。
- 如果 `link_path` 已存在，报错，不自动覆盖。
- daemon security cases 增加该 endpoint，scope 使用 `file.write`。

### 2. Frontend Daemon Client

在 `DaemonService` 增加 `createSymlink` 方法。

要求：

- 仅封装 daemon API。
- 不在 service 内计算 task workspace 或 project entry name。
- 错误信息保留 daemon 返回内容，方便 UI/日志定位。

### 3. Local No-Worktree Workspace Preparation

更新 `TaskStore` Local runtime workspace 准备逻辑。

要求：

- `local_worktree_enabled=false` 时不再把真实项目目录作为 task CWD。
- 计算 `workspace-{taskId}`。
- 调 daemon 创建 workspace root。
- 为每个选中 Git 项目创建 symlink entry。
- 调 daemon 写入 `AGENTS.md`。
- 将 workspace root 写入 local workspace path cache / sessionDB。

### 4. Flow Task Creation

更新 Flow Task 创建后的 local setup 分支。

要求：

- 勾选 `Use worktree`：保持现有 worktree 创建流程。
- 未勾选 `Use worktree`：执行 no-worktree workspace preparation。
- local setup 失败时，保持当前 task-created-but-local-setup-failed 的提示策略。

### 5. Chat Task Creation

更新 Chat Task 创建后的 local setup 分支。

要求：

- 勾选 `Use worktree`：保持现有 worktree 创建流程。
- 未勾选 `Use worktree`：执行 no-worktree workspace preparation。
- 多项目 local task 不应再因为未勾选 worktree 而直接使用某一个真实项目目录作为 CWD。
- UI 不再因为选择多个 Git 项目而强制勾选 `Use worktree`。

### 6. Cleanup

更新 task local workspace cleanup。

要求：

- Local worktree workspace 和 Local no-worktree workspace 都需要删除 `workspace-{taskId}`。
- no-worktree cleanup 只删除 workspace root；删除 symlink entry 不会删除 symlink target。
- 继续避免删除非 task workspace 路径。

### 7. Tests

Daemon tests：

- 创建 symlink 成功。
- `target_path` 不存在时报错。
- `link_path` 已存在时报错。
- allowlist 校验覆盖 `link_path` 和 `target_path`。

Frontend tests：

- Local no-worktree task 创建 workspace root。
- Local no-worktree task 写入 symlink project entries。
- Local no-worktree task 写入 `AGENTS.md`。
- Local no-worktree task CWD 持久化为 workspace root。
- Local worktree enabled 行为不变。
- Remote runtime 创建流程不受 `local_worktree_enabled` 改动影响。

### 8. Verification

建议本地验证：

```bash
cd ai-desk-daemon/daemon && go test ./internal/daemonapp
cd frontend && yarn typecheck
```

如果存在相关 focused frontend test，优先运行 focused test 后再运行 typecheck。

手动 smoke：

- 创建 Local Flow Task，选择多个 Git 项目，关闭 `Use worktree`。
- 确认 workspace root 为 `workspace-{taskId}`。
- 确认 workspace 下项目入口是 symlink。
- 确认 `AGENTS.md` 存在且内容正确。
- 启动 Local runtime，确认 CWD 是 workspace root。
- 删除 task 或触发 cleanup，确认真实项目目录仍存在。

## Rollback

如果 no-worktree workspace preparation 出现问题：

- 保留 daemon symlink API 无害部署。
- 前端可回退到旧逻辑：`local_worktree_enabled=false` 时使用真实项目目录作为 CWD。
- 已创建的 no-worktree workspace 可通过 existing workspace cleanup 删除。

## QA Recommendation

推荐 QA Agent `standard`。

原因：

- 影响 Local Flow Task 和 Chat Task 的 runtime CWD。
- 新增 daemon filesystem mutation API。
- 涉及 task workspace cleanup。
- 不影响 remote runtime、DB schema、auth、backend remote daemon execution。
