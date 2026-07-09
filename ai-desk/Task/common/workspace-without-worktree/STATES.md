# Workspace Without Worktree Loop State

更新时间：2026-07-09

## Discovery

Specs:

- `specs/01-product-scope.md`
- `specs/02-local-workspace-behavior.md`
- `specs/03-daemon-and-runtime-contract.md`
- `specs/04-implementation-plan.md`

Tasks:

1. Daemon symlink API: add `POST /api/v1/file/symlink`, validation, security case, Go tests.
2. Frontend daemon client: add `daemonService.createSymlink`.
3. Local no-worktree workspace preparation: create `workspace-{taskId}`, symlink selected project entries, write `AGENTS.md`, persist CWD.
4. Flow Task creation: call no-worktree workspace preparation when Local runtime and `Use worktree` is off.
5. Chat Task creation: no longer force `Use worktree` for multiple Git projects; call no-worktree workspace preparation when off.
6. Cleanup: remove `workspace-{taskId}` for Local no-worktree workspaces without touching symlink targets.
7. Verification: focused daemon tests, focused frontend tests/typecheck, spec review.

## Plan

Round 1 plan tasks:

- P1 Daemon symlink API, because FE no-worktree setup depends on it.
- P2 Frontend local workspace orchestration, independent after the daemon client contract is known.

Round 2 plan tasks:

- P3 Cleanup and UI forced-worktree adjustment.
- P4 Verification review and fixes.

## Status

| Task | Owner | Status | Notes |
| --- | --- | --- | --- |
| P1 Daemon symlink API | worker Leibniz | completed | Added `POST /api/v1/file/symlink`, security route/cases, focused Go tests. |
| P2 Frontend local workspace orchestration | worker Gibbs/main | completed | Local no-worktree now prepares `workspace-{taskId}`, symlinks selected projects, writes `AGENTS.md`, persists workspace root. |
| P3 Cleanup/UI adjustment | worker/main | completed | Local no-worktree cleanup removes only safe task workspace dirs; Chat Task UI no longer forces worktree for multi-project selection; Flow Task create-only path prepares local workspace for Git projects regardless of worktree checkbox. |
| P4 Verification | verifier/main | completed | Verifier found Flow Task create-only gap; main fixed it and reran focused checks. |

## Verification Log

- Codegraph check: `.understand-anything/meta.json` is already at current `HEAD` (`2d988acd1140bf6be11b72b0e33af92b40aeb2aa`). Current feature edits are uncommitted, so the commit-based incremental graph update has no new committed range to analyze yet.
- Pass: `go test ./internal/daemonapp -count=1 -run 'TestHandleFileSymlink|TestDaemonSecurityPolicyAllowsRemoteFileSymlinkWithWriteGrant|TestDaemonSecurityCaseRunnerCoversAllFlows|TestDaemonSecurityPolicyMergesDefaultRoutesIntoStalePolicyFile'` from `ai-desk-daemon/daemon`.
- Pass: `node test/agent-task-setup-preferences.test.cjs && node test/task-local-no-worktree-workspace.test.cjs` from `frontend`.
- Pass after verifier fix: `node test/create-task-worktree-ui.test.cjs && node test/agent-task-setup-preferences.test.cjs && node test/task-local-no-worktree-workspace.test.cjs && node test/task-workspace-path-guard.test.cjs` from `frontend`.
- Pass: `node frontend/test/task-workspace-path-guard.test.cjs` from frontend worker.
- Pass: `npm --prefix frontend run type-check`.
- Pass after verifier fix: `npm --prefix frontend run type-check`.
- Pass: `cd frontend && yarn type-check` from frontend worker.
- Pass: `git diff --check -- ai-desk-daemon/daemon/internal/daemonapp frontend/src frontend/test`.
- Pass after verifier fix: `git diff --check -- ai-desk-daemon/daemon/internal/daemonapp frontend/src frontend/test`.
- Pass: frontend worker dev server smoke `curl -I http://127.0.0.1:3000/tasks` returned `200 OK`.
- Not green, likely existing unrelated failure: `npm run test:frontend:unit:cjs` stops at `frontend/test/agent-preset-selector-catalog-contract.test.cjs` with `RuntimeSelector groups should be labeled AgentFarm, Remote Runtime, Local Runtime, and Shared Runtime`.
- Verifier finding resolved: `CreateTaskModal` now calls `taskStore.ensureLocalWorktreeForTask(task.id)` for created Git tasks even when `Use worktree` is unchecked, so `TaskStore` can prepare the no-worktree symlink workspace during create-only Flow Task creation.

## Implementation Memory

- Scope is Local runtime only.
- Remote daemon runtime and AgentMesh runtime must remain unchanged.
- Daemon must stay business-agnostic.
- `AGENTS.md` content is English.
- `link_path` should be inside a new `workspace-{taskId}` and should not already exist.
- If `link_path` exists, daemon should fail instead of overwriting.
- Chat Task multi Git project selection should not force `Use worktree`.
