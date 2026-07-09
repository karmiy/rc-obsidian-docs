# 01 Product Scope

更新时间：2026-07-09

## 背景

Flow Task 和 Chat Task 在 Local runtime 下支持 `Use worktree` 选项。

当前行为：

- 勾选 `Use worktree`：创建 `workspace-{taskId}`，并把选中的项目以 Git worktree 形式放入该 workspace。
- 未勾选 `Use worktree`：直接使用用户真实项目目录作为 runtime CWD。

新需求是统一 Local runtime 的 CWD 形态：即使用户未勾选 `Use worktree`，也创建 `workspace-{taskId}` 作为 runtime CWD。

## 目标

Local runtime 下，未勾选 `Use worktree` 时：

- 创建 task workspace 目录：

```text
<worktree-root>/workspace-{taskId}
```

- 在 workspace 下为每个选中项目创建同名 symlink：

```text
workspace-{taskId}/Fiji -> /real/local/path/Fiji
workspace-{taskId}/ai-desk -> /real/local/path/ai-desk
```

- 在 workspace 根目录写入 `AGENTS.md`，说明 workspace 内项目入口是 symlink。
- 后续 runtime CWD 使用 workspace 根目录，而不是用户真实项目目录。
- 勾选 `Use worktree` 的现有行为保持不变。

## 非目标

本期不做：

- 不改变 remote runtime / remote daemon 的 workspace 或 worktree 创建逻辑。
- 不改变 AgentMesh runtime 的 repo mount 行为。
- 不把 task、project、workspace 等 AI Desk 业务概念写进 daemon。
- 不改变 backend remote worktree creation 的分支策略。
- 不迁移历史 task 的已持久化 workspace path。
- 不删除或重命名现有 `local_worktree_enabled` 字段。

## Runtime Scope

本需求只影响 Local runtime：

```text
FE -> local daemon -> local runtime CLI
```

Remote runtime 保持现状：

```text
FE -> BE remote runtime bridge -> remote daemon -> remote runtime CLI
```

原因：

- remote daemon 运行在服务器侧，不能访问用户本机真实项目目录。
- remote Git 项目当前由后端创建 server-side repo/worktree。
- remote 代码路径不使用 `local_worktree_enabled` 作为是否创建 remote worktree 的开关。

## 成功标准

- Local Flow Task 未勾选 `Use worktree` 时，runtime CWD 是 `workspace-{taskId}`。
- Local Chat Task 未勾选 `Use worktree` 时，runtime CWD 是 `workspace-{taskId}`。
- workspace 下每个项目入口都是指向真实本地项目目录的 symlink。
- workspace 根目录存在 `AGENTS.md`。
- 勾选 `Use worktree` 的 Local task 仍创建 Git worktree。
- Remote runtime 和 AgentMesh runtime 行为无变化。

