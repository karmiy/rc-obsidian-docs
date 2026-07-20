# AI Desk Installed Skill Native Discovery - Loop States

更新时间：2026-07-09

## Source Specs

- `00-overview.md`
- `01-local-materialization.md`
- `02-sync-flow.md`
- `03-workspace-runtime-discovery.md`
- `04-implementation-plan.md`

## Task List

| ID | Task | Dependency | Status | Notes |
| --- | --- | --- | --- | --- |
| T1 | BE installed manifest API | none | done | `GET /api/v1/skills/installed-manifest`; Python service/API/schema/OpenAPI/tests |
| T2 | BE installed bundle download API | T1 data model / skill query understanding | done | `POST /api/v1/skills/installed-bundles:download`; reuse SKILL.md fallback and safe relative path validation |
| T3 | BE skills updated event | existing skill install/update/uninstall paths | done | event pattern `/users/~/skills/updated`; publish after successful install/uninstall/version updates |
| T4 | MCP skill agentInstructions compatibility | existing MCP response builders | done | Python BE + Gleam BE wording parity for `kind='skill'` only |
| T5 | Daemon file batch APIs | existing daemon file APIs | done | `write-tree-batch` + `delete-batch`; security policy/cases updated |
| T6 | FE local skill sync service | T1/T2/T5 contracts | done | manifest diff, bundle download, daemon batch writes, manifest write |
| T7 | FE app startup sync trigger | T6 | done | daemon-ready startup trigger in `App.tsx` |
| T8 | FE skills updated socket trigger | T3/T6 | done | user-scoped event subscription in `App.tsx` |
| T9 | FE task workspace provider skills symlink ensure | T5/T6 | done | task preparation hook before runtime start |
| T10 | Cross-module verification and docs/OpenAPI | T1-T9 | done | focused tests, typecheck/build, API docs |
| T11 | FE skills updated subscription lifecycle race fix | T8 | done | Extracted app-level subscription coordinator; stale effect cleanup can no longer remove the active user-scoped Skill channel |
| T12 | Installed public Skill detail read-only permission fix | T1 | done | Applied the existing creator/admin rule to every Skill detail mutation entry point and handled save failures without an uncaught runtime error |
| T13 | Browser-local installed Skill sync disable switch | T6/T9 | done | Skip installed Skill sync and provider symlink preparation before any BE/daemon call when the browser-local switch is enabled |
| T14 | Gleam Skill version event installer fan-out parity | T3 | done | Version-changing PUT/sync/sync-local paths publish to every extension in `skill_installs`, matching Python BE behavior |

## Plan Queue

### Loop 1

| ID | Priority | Execution Mode | Status | Rationale |
| --- | --- | --- | --- | --- |
| T1/T2/T3/T4 | P0 | sub-agent worker | done | BE APIs/events/MCP instructions share skill service and route context; assigned to worker Russell. |
| T5 | P0 | main-thread takeover | done | Worker was interrupted after no response; main thread implemented daemon batch API and tests. |

### Deferred

None.

### Loop 3

| ID | Priority | Execution Mode | Status | Rationale |
| --- | --- | --- | --- | --- |
| T11 | P0 | sub-agent worker + independent verifier | done | Runtime logs proved duplicate subscribe followed by stale unsubscribe removed `/users/{extensionId}/skills/updated`; the coordinator fix preserves existing EventBus channels and daemon-ready sync behavior. |

### Loop 4

| ID | Priority | Execution Mode | Status | Rationale |
| --- | --- | --- | --- | --- |
| T12 | P0 | sub-agent worker + independent verifier | done | Gleam correctly returns 403 for a non-owner update; FE now enforces the same read-only boundary and handles rejected save Promises through UI errors. |

### Loop 5

| ID | Priority | Execution Mode | Status | Rationale |
| --- | --- | --- | --- | --- |
| T13 | P1 | focused FE implementation + independent verifier | done | Two browser environments may point at the same local daemon; one browser now has an explicit local opt-out so it cannot mutate the shared installed Skill directory. |

### Loop 6

| ID | Priority | Execution Mode | Status | Rationale |
| --- | --- | --- | --- | --- |
| T14 | P0 | sub-agent worker + independent verifier | done | Redis subscription state is correct, but Gleam published a version event only to the editing actor; one shared fan-out invariant now covers PUT, sync, and sync-local version changes. |

## Implementation Memory

- FE orchestrates sync; daemon remains generic and business-agnostic.
- `~/.aidesktop/skills` root is created by writing `manifest.json` through daemon `file/write`.
- Local manifest `issues` are per-sync lifecycle logs only and do not participate in next diff.
- Provider skills symlinks are directory-level links from workspace outer root to `~/.aidesktop/skills`.
- Existing slash UI/prompt marker remains unchanged; only MCP skill `agentInstructions` gains workspace cache priority.
- FE Loop 2 likely files:
  - new `frontend/src/services/installedSkillLocalSync.ts` for local manifest parse, dirName normalize, diff, bundle download, daemon batch calls, manifest write, single-flight.
  - `frontend/src/services/DaemonService.ts` client methods from daemon worker: `writeFileTreeBatch`, `deleteBatch`.
  - `frontend/src/App.tsx` daemon-ready startup trigger and `/users/~/skills/updated` event subscription.
  - `frontend/src/stores/TaskStore.ts` workspace preparation hook after workspace/project entry ready and before runtime start.
- Existing `frontend/src/services/taskWorkspaceAideskSkills.ts` is old workspace-local bundle materialization. New service should stay separate because specs require user-level inventory symlink, not per-workspace file copy.
- Loop 1 BE slice plan: keep installed manifest/bundle in Python `SkillService`/`skills.py`; do not add FE sync or daemon batch APIs in this worker.
- T3 publish should happen after successful DB commit from existing service methods; version events should fan out through `skill_installs` rows only.
- T4 Gleam parity is scoped to `backend_gleam/src/http/runtime_mcp_routes.gleam` and focused tests; avoid touching existing backend_gleam config changes.
- Loop 1 BE slice implemented 2026-07-09:
  - Python `GET /api/v1/skills/installed-manifest` returns current user installed standalone skills only.
  - Python `POST /api/v1/skills/installed-bundles:download` validates requested ids against current user installs, returns safe relative file trees with `SKILL.md` fallback.
  - `/users/~/skills/updated` registered and guarded so the subscription extensionId must equal current user.
  - install/uninstall publish current-user events; PUT/sync/sync-local publish version events to installed extension ids only when version-changing path succeeds.
  - Python and Gleam MCP skill entity instructions now prefer workspace .{agent}/skills cache before archive materialization.
  - Verification: Python focused tests passed; Gleam format/build passed; Gleam full test command is blocked by missing `GLEAM_TEST_DATABASE_URL` for existing DB integration tests.
- Loop 2 implementation completed 2026-07-09:
  - Daemon added generic `POST /api/v1/file/write-tree-batch` and `POST /api/v1/file/delete-batch`, reusing existing `writeFileTree` and delete semantics with per-item failed lists.
  - Daemon security policy/cases include the new file write APIs under `file.write`.
  - FE added `installedSkillLocalSyncService` with single-flight sync, name normalization, local manifest diff, BE bundle download, daemon batch write/delete, per-sync `issues`, and absolute daemon file paths resolved from daemon home dir.
  - FE app triggers sync on daemon-ready startup and current-user `/users/~/skills/updated` socket events.
- FE task workspace preparation ensures `.codex/skills`, `.claude/skills`, `.cursor/skills`, `.github/skills`, and `.agents/skills` symlink to the user installed skills root only for AI Desk managed task workspace roots.
- T11 root cause evidence (2026-07-10): FE refresh issued two 101-byte Skill subscription POSTs followed by one 55-byte Skill unsubscribe DELETE; Redis `SISMEMBER` then returned `0`, while the WebSocket remained active. The subscription lifecycle must move out of `App.tsx` local effect variables into a persistent, idempotent coordinator with active/pending scope ownership.
- T11 implementation boundary: keep `InstalledSkillLocalSyncService` responsible for manifest/materialization only; add a separate subscription coordinator and a thin React hook; `App.tsx` should only invoke the hook and retain the existing daemon-ready startup sync trigger.
- T11 acceptance: repeated ensure for the same extension creates one subscription; stale cleanup cannot unsubscribe the current scope; scope change unsubscribes the previous callback once and subscribes the new scope; socket event triggers `skill_change_socket`; focused race test and frontend type-check pass.
- T11 verification (2026-07-10):
  - `node test/installed-skill-sync-coordinator.test.cjs` passed.
  - `node test/installed-skill-local-sync.test.cjs` passed.
  - `yarn type-check` passed.
  - Live Gleam/Redis/browser flow passed: the current session remained subscribed to `/users/{extensionId}/skills/updated`; installing `azure-skills/azure-deploy` added it to the local manifest and materialized its directory; uninstalling it removed both entries while the subscription remained active.
  - Verification passed:
    - `PYTHONPATH=backend pytest backend/tests/test_installed_skill_local_sync.py backend/tests/test_skill_repository_skill_set_filter.py backend/tests/test_skill_change_events.py backend/tests/test_events_subscriptions_observability.py backend/tests/test_skill_service_usage.py::test_mcp_platform_skill_entity_bundle_uses_archive_materialization -q`
    - `go test ./internal/daemonapp -run 'TestHandleFileTreeBatch|TestHandleFileDeleteBatch|TestWriteFileTree|TestDaemonSecurityCaseRunnerCoversAllFlows|TestDaemonSecurityPolicyMergesDefaultRoutesIntoStalePolicyFile'`
    - `node test/installed-skill-local-sync.test.cjs && yarn type-check`
    - `cd backend_gleam && gleam format --check src/http/runtime_mcp_routes.gleam test/mcp_routes_test.gleam && gleam build --warnings-as-errors`
- T12 root cause evidence (2026-07-10): `SettingsSkillsSkillDetail` already computed the backend-aligned creator/admin `canModifySkill` rule, but used it only for origin sync/source visibility. Content save, file mutation, metadata save, visibility, and delete remained interactive for read-only viewers; rejected save Promises were not caught by the detail surface.
- T12 implementation boundary: frontend only. Keep Python/Gleam authorization unchanged; make the Skill detail read-only for non-creator/non-admin users, hide mutation controls, and convert save failures to UI error toasts.
- T12 verification (2026-07-10):
  - `node test/settings-skill-detail-readonly.test.cjs` passed.
  - `node test/settings-skills-local-origin-sync-contract.test.cjs` passed.
  - `node test/settings-skill-detail-save-model.test.cjs` passed.
  - `node test/settings-skills-local-origin-sync.test.cjs` passed.
  - `yarn type-check` passed.
  - Independent verifier approved the scoped diff; manual browser acceptance remains with the user by request.
- T13 boundary: localStorage key `ai-desk-disable-installed-skill-local-sync`; value `true` disables startup/socket/manual sync and task workspace provider symlink preparation before any BE or daemon operation. Removing the key restores the existing behavior.
- T13 verification (2026-07-10):
  - `node test/installed-skill-local-sync.test.cjs` passed, including real localStorage key simulation and zero API/daemon call assertions.
  - `node test/installed-skill-sync-coordinator.test.cjs` passed.
  - `yarn type-check` passed.
  - Independent verifier approved the scoped diff; no browser verification was run by user request.
- T14 root cause evidence (2026-07-10): Skill `080199e1-19f1-4aa3-99d4-cc33fc1a49c8` is version `1.0.4`; `skill_installs` contains both owner `8966222234270333` and installer `8967481597427794`; the installer's active sessions subscribe to `/users/8967481597427794/skills/updated`. Gleam `update_settings_skill` publishes only to the request actor extension, while Python queries every installed extension. Gleam sync and sync-local changed branches also omit the version event.
- T14 implementation boundary: add a primary-routed `skill_installs` query plan through Skill repo/service, one aggregate-observed installer fan-out helper, and call it only after successful version-changing PUT/sync/sync-local paths. Keep install/uninstall events and the existing socket payload unchanged.
- T14 verification (2026-07-10):
  - Focused Gleam EUnit `skill_version_event_fanout_test`: 2 passed.
  - `gleam build --warnings-as-errors` passed.
  - `gleam format --check` passed for the six scoped Gleam files.
  - Python parity test `backend/tests/test_skill_change_events.py`: 5 passed.
  - Local Gleam container rebuilt after restart and `/health/ready` returned HTTP 200 healthy.
  - Full `gleam test`: 1337 passed, 67 existing DB integration failures caused by missing `GLEAM_TEST_DATABASE_URL`.
  - Independent verifier approved: all three version-changing entries fan out, unchanged sync does not publish, install/uninstall and event payload remain unchanged, and aggregate logs contain no extension IDs.
