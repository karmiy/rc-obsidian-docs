status: DONE_WITH_CONCERNS

## Summary

Implemented backend tasks 1-6 for register/member invite:

- Added `group_invite` SQL schema and migration.
- Added `GroupInvite` Sequelize model.
- Added registration-create-group and registration-join-group service/controller flows.
- Added one-time hashed invite generation and consumption.
- Added admin-only member list, role update, and invite creation endpoints.
- Enforced current-group scoping and last-admin protection.
- Added backend controller integration tests covering the required success and failure paths.

## Tests Written And TDD Evidence

Test file:

- `toy-taro-server/test/app/controller/user.test.ts`

Covered paths:

- Create-group registration creates a new group, ADMIN user, and token.
- Admin invite generation returns a random invite code and stores only a hash.
- Join-group registration accepts normalized invite input, creates USER, consumes invite, records `used_by` and `used_at`.
- Reusing an invite fails with `邀请码无效或已被使用`.
- Member list is admin-only and returns current-group members without exposing `group_id`.
- Non-admin invite creation fails.
- Non-admin member role update fails.
- Admin member role update succeeds for same-group member.
- Cross-group member update fails.
- Last ADMIN downgrade fails.

Red evidence:

- `npm run test-local -- test/app/controller/user.test.ts`
- Initial meaningful red after fixing test harness shape: 10 failing tests.
- Failures were expected missing behavior: register route returned `401`, other new routes returned `404`.

Green evidence:

- `npm run test-local -- test/app/controller/user.test.ts`
- Result: `10 passing`.
- Final full suite: `npm run test-local`
- Result: `11 passing`.

## Files Changed

Implemented files:

- `toy-taro-server/database/group_invite.sql`
- `toy-taro-server/database/migrations/202607090003_create_group_invite.sql`
- `toy-taro-server/app/model/groupInvite.ts`
- `toy-taro-server/app/service/User.ts`
- `toy-taro-server/app/controller/user.ts`
- `toy-taro-server/app/router.ts`
- `toy-taro-server/test/app/controller/user.test.ts`
- `toy-taro-server/typings/app/model/index.d.ts`

Pre-existing worktree change left untouched:

- `toy-taro-server/typings/app/service/index.d.ts` was already modified before this task and remains as-is.

## Commands Run And Results

- `npm run test-local -- test/app/controller/user.test.ts`
  - Red before implementation: failed with missing route/auth behavior.
  - Green after implementation: `10 passing`.
- `npm run test-local`
  - Passed: `11 passing`.
- `npx eslint app/controller/user.ts app/model/groupInvite.ts app/router.ts app/service/User.ts test/app/controller/user.test.ts --ext .ts`
  - Passed with no output after scoped cleanup.
- `npm run lint`
  - Failed on unrelated pre-existing lint/prettier issues in files outside Loop 1 scope, including `app/controller/learning.ts`, `app/service/Learning.ts`, `app/service/ai.ts`, `config/config.local.ts`, and `config/config.prod.ts`.
  - No errors remained in files touched for this task under scoped ESLint.
- `npm run tsc`
  - Failed before checking app code because the repo uses TypeScript 3.0 while installed `date-fns` and `sequelize` declarations use newer TypeScript syntax.
  - Representative errors: `node_modules/date-fns/types.d.ts` `import { type ... }`, `node_modules/sequelize/types/model.d.ts` mapped-type `as` syntax.

## Spec Compliance Self-Review

- Registration create group: implemented `POST /user/registerCreateGroup`; creates group and ADMIN user; returns `{ token }`.
- Registration join group: implemented `POST /user/registerJoinGroup`; validates active one-time invite by hash; creates USER; marks invite inactive; records `used_by` and `used_at`; returns `{ token }`.
- Invite code safety: generated server-side random `BANYA-XXXX` code; normalizes input; stores SHA-256 hash only; does not expose `group_id` in API response.
- Admin-only endpoints: implemented `GET /user/getMemberList`, `POST /user/updateMemberRole`, `POST /user/createGroupInvite`; service validates current DB user is ADMIN.
- Group scoping: member list and role updates are constrained to current `group_id`; cross-group update returns `成员不存在或无权限操作`.
- Last admin protection: downgrading the final ADMIN returns `至少保留一名管理员`.
- Response/error patterns: reused `ctx.responseSuccess`, `ctx.responseFail`, `SERVER_CODE`, and `JsError`.
- Routes: register endpoints are public; member/invite endpoints require auth and service-level ADMIN check.

## Concerns

- Full `npm run tsc` is blocked by existing dependency/compiler incompatibility unrelated to this loop. The new code is exercised by `egg-bin test`, but full TypeScript verification cannot currently complete.
- Full `npm run lint` is blocked by unrelated pre-existing lint/prettier errors outside the owned write scope. Scoped ESLint on changed backend files passes.
- To stay within the requested write scope, public registration auth exceptions are appended in `router.ts` rather than changing `config/config.default.ts`.
- The controller test includes a small test-only `SlowBuffer` compatibility shim because the current Node runtime and older `egg-jwt` dependency otherwise fail while booting the mocked Egg app.

---

status: DONE

## Loop 1 Backend Review Fix Pass

### Summary

Fixed the backend review findings only:

- Increased invite-code entropy from `BANYA-XXXX` to segmented `BANYA-XXXX-XXXX-XXXX` while keeping server-side normalization and hash-only persistence.
- Made last-admin protection DB-safe by wrapping member role updates in a transaction and locking same-group user rows before checking the admin count and updating the target role.
- Added controller/auth failure-path tests for unauthenticated management endpoints, missing `groupName`, missing `inviteCode`, missing `userId` / `role`, invalid role, and duplicate group names.
- Replaced the flaky invite-code `group.id` substring assertion with the actual API contract: invite creation returns only `{ inviteCode }`, and the DB hash does not contain the plaintext invite.

### TDD Evidence

Red:

- `npm run test-local -- test/app/controller/user.test.ts`
- Result before implementation: failed with 2 expected review regressions:
  - `createGroupInvite lets admins create a hashed invite without exposing group id` rejected old `BANYA-MSX5`-style 4-character invite codes.
  - `updateMemberRole keeps one admin when two admins are downgraded concurrently` observed `remainingAdmins = 0` with the old count-then-update logic.

Green:

- `npm run test-local -- test/app/controller/user.test.ts`
- Result after implementation: `15 passing`.
- Final full suite: `npm run test-local`
- Result: `16 passing`.

### Files Changed In This Fix Pass

- `toy-taro-server/app/service/User.ts`
- `toy-taro-server/test/app/controller/user.test.ts`
- `/Users/karmiy.hong/Documents/WorkDir/obsidian-workspace/work dir/sprout-mate/register/specs/loop1-backend-report.md`

### Commands Run And Results

- `npm run test-local -- test/app/controller/user.test.ts`
  - Red before implementation: 2 failing review-regression tests.
  - Green after implementation: `15 passing`.
- `npm run test-local`
  - Passed: `16 passing`.
- `npx eslint app/controller/user.ts app/model/groupInvite.ts app/router.ts app/service/User.ts test/app/controller/user.test.ts --ext .ts`
  - Passed with no output after fixing one Prettier wrapping issue in `app/service/User.ts`.

### Reviewer Finding Responses

1. Last-admin protection race:
   - Fixed. `updateMemberRole` now runs in a transaction, locks all same-group user rows with `FOR UPDATE`, computes the admin count from the locked rows, and updates the target inside the same transaction. Added concurrent downgrade coverage.
2. Invite code entropy too low:
   - Fixed. Invite codes now use 12 random characters in three human-readable segments: `BANYA-XXXX-XXXX-XXXX`.
3. Missing controller/auth failure tests:
   - Fixed. Added tests for unauthenticated management calls, missing `groupName`, missing `inviteCode`, missing `userId`, missing `role`, and invalid role.
4. Flaky invite no-group-id substring assertion:
   - Fixed. Removed the random substring absence assertion and now tests the API response contract plus hash-not-plaintext storage.
5. Duplicate group-name behavior:
   - Kept and tested because the solution error table documents `分组名重复 | 该分组名称已存在`.
