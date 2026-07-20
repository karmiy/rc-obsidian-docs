# Register Feature Loop State

## Branch

- Source branch: `master`
- Working branch: `codex/register-member-invite`
- Loop source: `/Users/karmiy.hong/Documents/WorkDir/obsidian-workspace/work dir/ai-engineer/loop.md`
- Specs:
  - `requirements.md`
  - `solution.md`
  - `tasks.md`

## Global Constraints

- Implement only the specs in this directory; stop and report a block instead of inventing fallback behavior.
- Use TDD for behavior changes: write the failing test, observe RED, implement, observe GREEN.
- Keep changes incremental and reviewable.
- Do not touch unrelated user changes; existing untracked `.codegraph/` is out of scope.
- Frontend must reuse existing components and layout patterns where possible.
- Frontend styles must use `@ui/styles/index.scss`, token variables, `COLOR_VARIABLES`, `px()`, and existing mixins instead of hard-coded colors and ad hoc styling.
- Member management page must align with `checkInManage/entrance`, `prizeManage/entrance`, and `taskManage/categoryManage`, including `ConfigListPanel` and `StatusWrapper` loading/empty behavior.
- Backend must enforce admin checks and group scoping; frontend visibility is not security.
- One group must retain at least one `ADMIN`.
- Invite codes must be random, hashed in storage, one-time use, and must not expose `group_id`.
- Active invite generation must reuse the current group's existing unused invite and return it without inserting duplicate rows.
- Invite codes must not be stored in plaintext; store a server-decryptable ciphertext only when needed to return an existing active invite.
- Successful invite registration must physically delete the consumed invite row.
- `group.name` is not globally unique; creating the same visible group name creates separate group rows.
- Account-password registration usernames must contain English letters and numbers only.
- Create-group registration and invite-join registration both first collect the target space information, then switch the current page area to the shared WeChat/account-password registration choices.
- Member role modal option taps update local selection only; the API call and loading state belong to the explicit save button.

## Discovery

| Task | Source | Status | Notes |
|------|--------|--------|-------|
| 1 | tasks.md backend | completed | Add `group_invite` SQL and migration |
| 2 | tasks.md backend | completed | Add Sequelize GroupInvite model |
| 3 | tasks.md backend | completed | Extend User service registration, invite, member role logic |
| 4 | tasks.md backend | completed | Extend User controller |
| 5 | tasks.md backend | completed | Register backend routes |
| 6 | tasks.md backend | completed | Add backend tests |
| 7 | tasks.md frontend | completed | Add PAGE_ID, router, app config |
| 8 | tasks.md frontend | completed | Extend User API and module |
| 9 | tasks.md frontend | completed | Extend user/member data structures |
| 10 | tasks.md frontend | completed | Extend useUserAction |
| 11 | tasks.md frontend | completed | Add login page register entry |
| 12 | tasks.md frontend | completed | Add register page |
| 13 | tasks.md frontend | completed | Add useMemberManage |
| 14 | tasks.md frontend | completed | Add member management page |
| 15 | tasks.md frontend | completed | Add Mine menu entry |
| 16 | tasks.md verification | completed | Create group register covered by backend/frontend tests |
| 17 | tasks.md verification | completed | Invite join covered by backend/frontend tests |
| 18 | tasks.md verification | completed | Role management covered by backend/frontend tests |
| 19 | user feedback 2026-07-09 | completed | Join-invite page switches to account creation content instead of failing on already-registered wx identity |
| 20 | user feedback 2026-07-09 | completed | Create invite reuses an existing unused invite for the group |
| 21 | user feedback 2026-07-09 | completed | Consumed invite row is physically deleted |
| 22 | user feedback 2026-07-10 | completed | Member role modal saves only after explicit confirmation; role options do not show loading |
| 23 | user feedback 2026-07-10 | completed | Account-password registration usernames accept only English letters and numbers |
| 24 | user feedback 2026-07-10 | completed | Create-group registration allows duplicate visible group names and removes the unique DB constraint |
| 25 | user feedback 2026-07-10 | completed | Create-group flow uses the same WeChat/account-password registration step as invite-join flow, carrying `groupName` to the backend |

## Plan

### Loop 0: Baseline

- Status: completed
- Goal: confirm branch and existing test/build health before implementation.
- Tasks: run backend and frontend baseline checks that are practical in this repo.

### Loop 1: Backend persistence and business contract

- Status: completed
- Plan tasks: 1, 2, 3, 4, 5, 6.
- Dependency reason: frontend registration and member management depend on backend API contracts.
- Execution owner: backend worker subagent.
- Verification owner: backend reviewer subagent.
- Expected deliverable: tested backend endpoints for create-group registration, invite registration, member list, role update, invite generation.

### Loop 2: Frontend API and routing contract

- Status: completed
- Plan tasks: 7, 8, 9, 10.
- Dependency reason: pages need route constants, SDK APIs, and view-model actions.
- Execution owner: frontend contract worker subagent.
- Verification owner: frontend reviewer subagent.
- Expected deliverable: client API/module/types/actions compile and follow existing login/bootstrap patterns.

### Loop 3: Frontend screens and menu entry

- Status: completed
- Plan tasks: 11, 12, 13, 14, 15.
- Dependency reason: requires frontend API and routing contract from Loop 2.
- Execution owner: frontend UI worker subagent.
- Verification owner: frontend reviewer subagent.
- Expected deliverable: login entry, register page, member management page, and Mine entry, using existing components/status/token style constraints.

### Loop 4: Integration verification

- Status: completed
- Plan tasks: 16, 17, 18.
- Dependency reason: requires backend and frontend implementation.
- Execution owner: main thread.
- Verification owner: final reviewer subagent plus local verification commands.
- Expected deliverable: final backend/frontend verification and spec compliance review.

## Progress Log

- Created loop state from specs.
- Baseline complete: `toy-taro-client npm test -- --runInBand` passed 3 suites / 10 tests; `toy-taro-server npm run test-local` passed 1 test.
- Loop 1 started: backend worker owns server-side tasks 1-6.
- Loop 2 started while Loop 1 continues: frontend contract worker owns client-side tasks 7-10 only.
- Loop 1 completed: backend reviewer approved after fixes; local `npm run test-local` passed 16 tests and scoped ESLint passed.
- Loop 2 completed: frontend reviewer approved after auth-interceptor fix; local `npm test -- --runInBand` passed 8 suites / 29 tests. Build remains deferred until Loop 3 page files exist.
- Loop 3 started: frontend UI worker owns tasks 11-15.
- Loop 3 completed: frontend reviewer approved after shared `useSyncOnPageShow` member-management refresh fix; local `npm test -- --runInBand` passed 10 suites / 33 tests, `npm run build:weapp` passed, scoped ESLint passed, and `git diff --check` passed.
- Loop 4 started: main thread owns final backend/frontend integration verification.
- Loop 4 completed: backend tests passed 19 tests, frontend tests passed 10 suites / 33 tests, WeApp build passed, DB migration tests passed, scoped ESLint passed, `git diff --check` passed, and final reviewer approved with no remaining findings.

### Loop 5: Invite registration UX and invite lifecycle correction

- Status: completed
- Plan tasks: 19, 20, 21.
- Dependency reason: backend invite lifecycle determines frontend registration behavior and error handling.
- Execution owner: main thread with TDD.
- Verification owner: reviewer subagent plus local verification commands.
- Expected deliverable: repeated invite generation returns the existing active invite, successful invite registration physically deletes the invite row, and the register page switches from invite input to微信/账号密码 registration content while carrying the invite code.

- Loop 5 started: user clarified invite join should guide account registration, repeated invite generation should reuse unused invite DB row, and invite consumption should physically delete the row.
- Loop 5 completed: backend now stores invite hashes plus decryptable ciphertext for returning an existing active invite, generation reuses the active invite instead of inserting duplicates, successful invite registration physically deletes the consumed row, and the register join page now switches from invite input to微信/账号密码 registration content while carrying the invite code.
- Loop 5 verification: backend targeted user tests passed 20 tests; full backend tests passed 21 tests; DB migration parser tests passed; server scoped ESLint passed; frontend register/action tests passed 7 tests; full frontend tests passed 11 suites / 37 tests; scoped frontend ESLint passed; `npm run build:weapp` passed with existing warnings; `git diff --check` passed; reviewer subagent returned no findings.

### Loop 6: Register method parity, role-save confirmation, and group-name uniqueness correction

- Status: completed
- Plan tasks: 22, 23, 24, 25.
- Dependency reason: latest UX feedback changed both frontend registration flow and backend registration persistence assumptions.
- Execution owner: main thread with TDD.
- Verification owner: main thread local verification commands.
- Expected deliverable: member role modal saves through a confirmation button, create-group and invite-join flows share the same registration-method step, account usernames are alphanumeric only, and group names are allowed to repeat.

- Loop 6 started: user clarified role changes should be selected then saved, account usernames should reject Chinese/special characters, group creation should not check duplicate names, and create-group registration should use the same WeChat/account-password registration step as invite join while sending `groupName`.
- Loop 6 completed: role modal now keeps pending selection locally and loads only the save button; register flow now has a shared account-method step for both create and join modes; account username sanitization/validation exists on frontend and backend; duplicate group-name checks and unique DB constraints were removed with a migration to drop historical indexes.
- Loop 6 verification: frontend targeted tests passed 3 suites / 13 tests; backend targeted user tests passed 20 tests; scoped frontend ESLint passed; scoped backend ESLint passed; DB migration parser tests passed; full frontend tests passed 12 suites / 43 tests; full backend tests passed 21 tests; `npm run build:weapp` passed with existing warnings; `git diff --check` passed.
