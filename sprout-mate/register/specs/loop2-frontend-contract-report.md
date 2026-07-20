# Loop 2 Frontend Contract Report

## status

DONE_WITH_CONCERNS

## tests and TDD evidence

TDD was applied to the route, API, module, and UI-adjacent hook behavior in scope.

- RED: `npm test -- src/shared/utils/__tests__/router-register-member.spec.ts --runInBand`
  - Failed with `Property 'REGISTER' does not exist on type 'typeof PAGE_ID'` and `Property 'MEMBER_MANAGE' does not exist on type 'typeof PAGE_ID'`.
- GREEN: same command passed after adding `PAGE_ID` and route mappings.

- RED: `npm test -- src/core/api/__tests__/user-register-member.spec.ts --runInBand`
  - Failed with missing `UserApi.registerCreateGroup`, `registerJoinGroup`, `getMemberList`, `updateMemberRole`, and `createGroupInvite`.
- GREEN: same command passed after adding the API contract methods.

- RED: `npm test -- src/core/module/__tests__/user-register-member.spec.ts --runInBand`
  - Failed with missing `UserModule.registerCreateGroup`, `registerJoinGroup`, `getMemberList`, `updateMemberRole`, and `createGroupInvite`.
- GREEN: same command passed after adding module wrappers, token persistence, and member refresh behavior.

- RED: `npm test -- src/ui/viewModel/user/useUserAction.register.spec.ts --runInBand`
  - Failed with missing `handleRegisterCreateGroup` and `handleRegisterJoinGroup` returned from `useUserAction`.
- GREEN: same command passed after adding the registration actions.

- Full GREEN test run: `npm test -- --runInBand`
  - Result: 7 suites passed, 26 tests passed.

No page-rendering TDD was added because Loop 2 explicitly forbids creating or modifying the register page and member management page. The UI-adjacent hook behavior was covered with mocked React/Taro/bootstrap dependencies.

## files changed

- `toy-taro-client/src/shared/utils/constants.ts`
- `toy-taro-client/src/shared/utils/router.ts`
- `toy-taro-client/src/app.config.ts`
- `toy-taro-client/src/core/api/user.ts`
- `toy-taro-client/src/core/module/userModule.ts`
- `toy-taro-client/src/core/entity/user.ts`
- `toy-taro-client/src/core/model/user.ts`
- `toy-taro-client/src/ui/viewModel/user/useUserAction.ts`
- `toy-taro-client/src/shared/utils/__tests__/router-register-member.spec.ts`
- `toy-taro-client/src/core/api/__tests__/user-register-member.spec.ts`
- `toy-taro-client/src/core/module/__tests__/user-register-member.spec.ts`
- `toy-taro-client/src/ui/viewModel/user/useUserAction.register.spec.ts`

## commands run and results

- `npm test -- src/shared/utils/__tests__/router-register-member.spec.ts --runInBand`
  - RED before implementation: missing `PAGE_ID` members.
  - GREEN after implementation: 2 tests passed.

- `npm test -- src/core/api/__tests__/user-register-member.spec.ts --runInBand`
  - RED before implementation: missing `UserApi` methods.
  - GREEN after implementation: 5 tests passed.

- `npm test -- src/core/module/__tests__/user-register-member.spec.ts --runInBand`
  - RED before implementation: missing `UserModule` wrappers.
  - GREEN after implementation: 6 tests passed.

- `npm test -- src/ui/viewModel/user/useUserAction.register.spec.ts --runInBand`
  - RED before implementation: missing hook actions.
  - GREEN after implementation: 3 tests passed.

- `npm test -- --runInBand`
  - PASS: 7 test suites, 26 tests.

- `npm run build:weapp`
  - FAIL: Taro cannot resolve `src/ui/pages/memberManage/entrance/index`.
  - This is expected for this loop boundary because `app.config.ts` now declares the Loop 14 page path, but Loop 2 is not allowed to create the member management page files. The output stopped on the missing member management page; register page files are also owned by a later task.

- `npm run lint:eslint`
  - FAIL: ESLint crashed before reaching changed files with `TypeError: Cannot read properties of undefined (reading 'members')`.
  - Crash occurred in rule `@typescript-eslint/no-duplicate-enum-values` while linting existing `toy-taro-client/src/core/constants.ts`.

## spec compliance self-review

- Added `PAGE_ID.REGISTER` and `PAGE_ID.MEMBER_MANAGE`.
- Added router mappings:
  - `REGISTER` -> `/ui/pages/register/index`
  - `MEMBER_MANAGE` -> `/ui/pages/memberManage/entrance/index`
- Added `app.config.ts` subpackages for register and member management paths.
- Added `UserApi` methods:
  - `registerCreateGroup`
  - `registerJoinGroup`
  - `getMemberList`
  - `updateMemberRole`
  - `createGroupInvite`
- Added `UserModule` wrappers:
  - Registration wrappers persist returned token through `UserStorageManager`.
  - Member list loads through `UserApi.getMemberList`, refreshes the existing `CONTACT` store, and stops loading in `finally`.
  - Role updates refresh the member list after success.
  - Invite creation returns `{ inviteCode }`.
- Added `MemberEntity` as a `UserEntity` alias, preserving existing contact/user behavior.
- Kept `UserModel.isAdmin` behavior and made `role` observable so computed admin state remains reactive when the user model role changes.
- Added `handleRegisterCreateGroup` and `handleRegisterJoinGroup` to `useUserAction`.
  - Both use Taro login code.
  - Both call the new module registration wrappers.
  - Both reuse the existing loading/error handling flow.
  - Both bootstrap after registration, show success toast, and relaunch home.
- Did not create or modify register page, member management page, login page, mine page, backend files, or `STATES.md`.
- Did not add page styles or hard-coded style values.

## concerns

- `npm run build:weapp` cannot pass until later loops add the declared register/member page files. I did not create stubs because the user explicitly prohibited page creation in this loop.
- `npm run lint:eslint` is blocked by an existing ESLint rule crash in `src/core/constants.ts`; I did not modify lint config or that core constants file because it is outside the owned write scope.
- Existing backend changes were present in the worktree before this frontend work began and were left untouched.

---

## Loop 2 Frontend Contract Fix - Auth Public Paths

## status

DONE

## reviewer finding response

- Fixed the frontend auth interceptor so `/user/registerCreateGroup` and `/user/registerJoinGroup` are treated as public request paths alongside `/user/login`.
- Kept protected behavior intact: routes such as `/user/getMemberList` still reject without an auth token.
- Used an explicit public path list with suffix matching so existing absolute request URLs such as `/v1/toy/user/login` continue to work.

## tests and TDD evidence

- RED: `npm test -- src/core/httpRequest/__tests__/interceptors.spec.ts --runInBand`
  - Initial test setup run exposed a Jest alias mock resolution error; fixed the test harness without changing production code.
  - Behavior RED then failed as expected: both registration routes rejected with `登录已过期`.
- GREEN: `npm test -- src/core/httpRequest/__tests__/interceptors.spec.ts --runInBand`
  - PASS: 1 suite, 3 tests.
- Full GREEN run: `npm test -- --runInBand`
  - PASS: 8 suites, 29 tests.

## files changed

- `toy-taro-client/src/core/httpRequest/interceptors.ts`
- `toy-taro-client/src/core/httpRequest/__tests__/interceptors.spec.ts`
- `/Users/karmiy.hong/Documents/WorkDir/obsidian-workspace/work dir/sprout-mate/register/specs/loop2-frontend-contract-report.md`

## scope notes

- Did not create register/member page files.
- Did not touch backend files or `STATES.md`.
- Did not change `app.config.ts`; the known Loop 3 page-file build blocker remains outside this fix.
