# 02 Local Workspace Behavior

更新时间：2026-07-09

## 目标

定义 Local runtime 在 `Use worktree` 关闭时的 workspace 准备流程、目录结构和持久化规则。

## 目录结构

用户选择项目：

```text
Fiji
ai-desk
```

真实本地路径：

```text
/Users/user/code/Fiji
/Users/user/code/ai-desk
```

目标 workspace：

```text
<worktree-root>/workspace-{taskId}/
  Fiji -> /Users/user/code/Fiji
  ai-desk -> /Users/user/code/ai-desk
  AGENTS.md
```

`Fiji` 和 `ai-desk` 是 symlink entry，不是提前创建的真实目录。

## AGENTS.md

文件路径：

```text
<worktree-root>/workspace-{taskId}/AGENTS.md
```

内容：

```md
# agents.md

Projects in this workspace are symbolic links to their real local directories. Inspect project files from the real directories pointed to by those links.
```

## Local No-Worktree Flow

1. FE 通过现有 task creation API 创建 task，并持久化 `local_worktree_enabled=false`。
2. FE 通过现有 local project path mapping 解析每个选中项目的真实本地路径。
3. FE 使用现有 worktree root config 计算 workspace root：

```text
<resolved-worktree-root>/workspace-{taskId}
```

4. FE 调 local daemon 创建 workspace 目录。
5. FE 对每个选中项目调用 daemon symlink API：

```text
link_path   = <workspace>/<project-entry-name>
target_path = <real-local-project-path>
```

6. FE 调 daemon file write API 写入 `AGENTS.md`。
7. FE 将 workspace root 持久化为该 task 的 Local runtime CWD。
8. 后续 local runtime stream 继续走现有 `daemonService.executeAgentStream(...)`。

## CWD Rule

Local runtime 的 CWD 规则调整为：

- 有 Git worktree：CWD 为 `workspace-{taskId}`。
- 无 Git worktree：CWD 仍为 `workspace-{taskId}`，workspace 内项目入口为 symlink。
- 纯 local-only project 的现有限制和兼容性需要保持，不在本 spec 中扩大支持范围。

## Project Entry Name

项目 symlink 名称应复用现有 multi-project/worktree entry 命名策略。

要求：

- 用户能在 workspace 下看到稳定、可读的项目目录名。
- 产品侧预期同一个 task 内不会选择两个同名项目；本需求不新增同名冲突处理。
- 如果底层已有 sanitize 规则，继续复用；不要为 no-worktree 分支发明新的命名规则。
- 不允许由 daemon 根据 project id 或 task id 推导名称。

## Missing Path Handling

如果选中项目没有配置本地路径，或路径不存在：

- 必须继续走现有 Local runtime preparation / LocalPathDialog 机制。
- 不创建半成品 workspace。
- 不继续启动 runtime。

## Cleanup

删除或清理 task workspace 时：

- `workspace-{taskId}` 需要被移除。
- symlink entry 和 `AGENTS.md` 随 workspace 一起移除。
- 删除 workspace 目录只应删除 symlink entry 本身，不能 follow symlink 删除真实项目目录。
