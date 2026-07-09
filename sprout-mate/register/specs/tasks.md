# 任务清单

## 后端

### 1. 新增邀请码数据库表

**文件**：

- `toy-taro-server/database/group_invite.sql`
- `toy-taro-server/database/migrations/202607090003_create_group_invite.sql`

**内容**：

- 新增 `group_invite` 表。
- 保存 `group_id`、`code_hash`、`created_by`、`used_by`、`used_at`、`is_active`。
- 为 `code_hash` 添加唯一约束。
- 添加 `group_id`、`created_by`、`used_by` 外键。

**验收**：

- 迁移后数据库存在 `group_invite` 表。
- `code_hash` 唯一。
- 表结构支持一次性邀请码失效和审计。

**验证**：

```bash
cd toy-taro-server
npm run db:migrate
npm run db:migrate:status
```

---

### 2. 新增 GroupInvite Model

**文件**：

- `toy-taro-server/app/model/groupInvite.ts`

**内容**：

- 定义 `GroupInviteModel` 接口。
- 定义 Sequelize 字段和关联关系。
- `group_id`、`created_by`、`used_by` 读取时保持和现有 model 一致，返回字符串形式。

**验收**：

- 服务端可通过 `ctx.model.GroupInvite` 访问邀请码表。
- TypeScript 类型能表达已使用和未使用的邀请码状态。

**验证**：

```bash
cd toy-taro-server
npm run tsc
```

---

### 3. 扩展 User Service 注册能力

**文件**：

- `toy-taro-server/app/service/User.ts`

**内容**：

- 新增 `registerCreateGroup`：创建 `group` 和 `ADMIN` 用户。
- 新增 `registerJoinGroup`：校验邀请码，创建 `USER` 用户，失效邀请码。
- 新增邀请码工具方法：生成随机码、规范化邀请码、hash 邀请码。
- 新增 `createGroupInvite`：管理员生成一次性邀请码。
- 新增 `updateMemberRole`：管理员调整成员角色。
- 新增最后一个管理员保护逻辑。

**验收**：

- 创建分组注册后，用户 `role = ADMIN`。
- 邀请码注册后，用户 `role = USER`。
- 邀请码被成功使用后不能再次使用。
- 非当前 group 成员不能被当前管理员更新角色。
- 不能把最后一个 `ADMIN` 降级。

**验证**：

```bash
cd toy-taro-server
npm run test-local
npm run tsc
```

---

### 4. 扩展 User Controller 接口

**文件**：

- `toy-taro-server/app/controller/user.ts`

**内容**：

- 新增 `registerCreateGroup`。
- 新增 `registerJoinGroup`。
- 新增 `getMemberList`。
- 新增 `updateMemberRole`。
- 新增 `createGroupInvite`。
- 复用现有响应格式：`ctx.responseSuccess`、`ctx.responseFail`。

**验收**：

- 参数缺失时返回 `BAD_REQUEST`。
- 无权限时返回 `FORBIDDEN`。
- 注册成功返回 `{ token }`。
- 邀请码生成成功返回 `{ inviteCode }`。

**验证**：

```bash
cd toy-taro-server
npm run lint
npm run test-local
```

---

### 5. 注册服务端路由

**文件**：

- `toy-taro-server/app/router.ts`

**内容**：

- 新增 `POST /user/registerCreateGroup`。
- 新增 `POST /user/registerJoinGroup`。
- 新增 `GET /user/getMemberList`。
- 新增 `POST /user/updateMemberRole`。
- 新增 `POST /user/createGroupInvite`。

**验收**：

- 注册类接口无需登录态。
- 成员管理和生成邀请码接口需要登录态且需要管理员权限。

**验证**：

```bash
cd toy-taro-server
npm run lint
```

---

### 6. 补充后端测试

**文件**：

- `toy-taro-server/test/app/controller/user.test.ts`
- 或按项目现有测试结构新增对应测试文件

**内容**：

- 创建分组注册成功。
- 使用邀请码注册成功。
- 邀请码二次使用失败。
- 非管理员生成邀请码失败。
- 非管理员调整成员角色失败。
- 管理员调整成员角色成功。
- 最后一个管理员不能被降级。

**验收**：

- 核心注册和权限路径均有测试覆盖。
- 异常路径返回明确错误。

**验证**：

```bash
cd toy-taro-server
npm run test-local
```

---

## 前端

### 前端实现约束

以下约束适用于所有前端任务：

- 新增页面优先复用 `Layout`、`ConfigListPanel`、`StatusWrapper`、`FloatLayout`、`FormItem`、`Button`、`Input`、`WhiteSpace`、`SafeAreaBar`。
- 管理页布局优先参考 `checkInManage/entrance`、`prizeManage/entrance`、`taskManage/categoryManage`。
- 人员管理页列表优先使用 `ConfigListPanel`，并将 `loading` 传入 `ConfigListPanel`，由其内部 `StatusWrapper` 统一处理 loading/empty。
- 如局部区域不能使用 `ConfigListPanel`，必须显式使用 `StatusWrapper`，保持 `loading`、`count`、`loadingIgnoreCount`、`size` 与现有页面一致。
- 新增 SCSS 必须 `@import '@ui/styles/index.scss';`。
- 样式优先使用 `$--color-*`、`$--text-color-*`、`$--fill-*`、`$--color-border-*`、`$--gradient-*`、`px()` 和现有 mixin。
- TS 中需要颜色时优先使用 `COLOR_VARIABLES`。
- 不允许随意写死 hex 颜色、rgba 阴影、页面背景、圆角；确需新增视觉值时先补充共享 token。

### 7. 扩展 PAGE_ID 和路由映射

**文件**：

- `toy-taro-client/src/shared/utils/constants.ts`
- `toy-taro-client/src/shared/utils/router.ts`
- `toy-taro-client/src/app.config.ts`

**内容**：

- 新增 `PAGE_ID.REGISTER`。
- 新增 `PAGE_ID.MEMBER_MANAGE`。
- 在 `pagePathMap` 中配置特殊路径。
- 在 `app.config.ts` 中新增注册页和人员管理页 subPackage。

**验收**：

- 可以通过 `navigateToPage({ pageName: PAGE_ID.REGISTER })` 打开注册页。
- 可以通过 `navigateToPage({ pageName: PAGE_ID.MEMBER_MANAGE })` 打开人员管理页。

**验证**：

```bash
cd toy-taro-client
npm run build:weapp
```

---

### 8. 扩展 User API

**文件**：

- `toy-taro-client/src/core/api/user.ts`
- `toy-taro-client/src/core/module/userModule.ts`

**内容**：

- 新增 `registerCreateGroup`。
- 新增 `registerJoinGroup`。
- 新增 `getMemberList`。
- 新增 `updateMemberRole`。
- 新增 `createGroupInvite`。
- 保持 API 返回结构与服务端一致。

**验收**：

- SDK 层可以调用所有新增接口。
- 注册接口返回 token 后能写入登录态。

**验证**：

```bash
cd toy-taro-client
npm run lint:eslint
npm test
```

---

### 9. 扩展用户实体和成员数据结构

**文件**：

- `toy-taro-client/src/core/entity/user.ts`
- `toy-taro-client/src/core/model/user.ts`

**内容**：

- 复用 `UserEntity` 作为成员数据基础。
- 如页面需要，新增 `MemberEntity` 或 `ContactEntity` 字段。
- 保持 `isAdmin` computed 能正确判断角色。

**验收**：

- 成员列表可以展示用户 ID、昵称、头像、角色。
- 角色变更后 UI 能刷新。

**验证**：

```bash
cd toy-taro-client
npm run build:weapp
```

---

### 10. 扩展 useUserAction

**文件**：

- `toy-taro-client/src/ui/viewModel/user/useUserAction.ts`

**内容**：

- 新增 `handleRegisterCreateGroup`。
- 新增 `handleRegisterJoinGroup`。
- 注册成功后复用 `bootstrap`，再 `reLaunch` 到首页。
- 保持 loading、toast、错误处理和现有登录动作一致。

**验收**：

- 注册成功后能进入首页。
- 注册失败时展示服务端错误。
- 按钮 loading 状态正确。

**验证**：

```bash
cd toy-taro-client
npm run lint:eslint
```

---

### 11. 修改登录页入口

**文件**：

- `toy-taro-client/src/ui/pages/login/index.tsx`
- `toy-taro-client/src/ui/pages/login/index.module.scss`

**内容**：

- 在默认登录面板新增注册入口文案。
- 点击跳转注册页。
- 保持当前品牌视觉和登录流程不变。

**验收**：

- 登录页能进入注册页。
- 不影响微信登录和账号密码登录。

**验证**：

```bash
cd toy-taro-client
npm run build:weapp
```

---

### 12. 新增注册页

**文件**：

- `toy-taro-client/src/ui/pages/register/index.tsx`
- `toy-taro-client/src/ui/pages/register/index.module.scss`
- `toy-taro-client/src/ui/pages/register/index.config.ts`
- `toy-taro-client/src/ui/pages/register/constants.ts`

**内容**：

- 复用登录页上半屏品牌视觉。
- 下方面板提供「创建分组」「加入分组」模式切换。
- 创建分组模式输入分组名。
- 加入分组模式输入邀请码。
- 提交时调用对应注册动作。
- 提供返回登录页入口。
- 复用 `Button`、`Input`、`FormItem`、`WhiteSpace`、`SafeAreaBar`、`FallbackImage` 等现有组件。
- 样式复用登录页结构和主题 token，不写死颜色。

**验收**：

- 创建分组注册可用。
- 邀请码加入注册可用。
- 空字段不能提交。
- 注册成功进入首页。
- 视觉与登录页保持一致，未引入新的独立页面风格。
- SCSS 中颜色、背景、阴影尽可能来自 token 或 mixin。

**验证**：

```bash
cd toy-taro-client
npm run build:weapp
```

---

### 13. 新增人员管理 ViewModel

**文件**：

- `toy-taro-client/src/ui/viewModel/user/useMemberManage.ts`

**内容**：

- 拉取成员列表。
- 更新成员角色。
- 生成邀请码。
- 管理 loading、刷新、错误提示。

**验收**：

- 页面进入时能加载成员。
- 角色更新后列表刷新。
- 生成邀请码后展示新邀请码。

**验证**：

```bash
cd toy-taro-client
npm run lint:eslint
```

---

### 14. 新增人员管理页

**文件**：

- `toy-taro-client/src/ui/pages/memberManage/entrance/index.tsx`
- `toy-taro-client/src/ui/pages/memberManage/entrance/index.module.scss`
- `toy-taro-client/src/ui/pages/memberManage/entrance/index.config.ts`

**内容**：

- 使用 `ConfigListPanel` 展示当前分组成员列表。
- 展示成员头像、昵称、角色。
- 管理员可切换成员角色。
- 提供「生成邀请码」按钮。
- 邀请码生成成功后展示弹窗或面板，支持复制。
- 使用 `loading` 透传给 `ConfigListPanel`，复用其内部 `StatusWrapper`。
- 使用 `useSyncOnPageShow` 的刷新能力，行为对齐其他「我的」管理员入口页面。
- 角色编辑和邀请码展示优先使用 `FloatLayout`，异步按钮使用 `Button loading`。
- 成员项样式使用主题 token，不写死颜色。

**验收**：

- 管理员可以查看成员。
- 管理员可以调整成员角色。
- 管理员可以生成邀请码。
- 降级最后一个管理员时提示失败。
- 首次加载展示统一 loading 状态。
- 空列表展示统一 empty 状态。
- 下拉刷新和页面返回刷新行为与现有管理页一致。
- 页面结构、间距、卡片和按钮风格与 `checkInManage/entrance`、`prizeManage/entrance`、`taskManage/categoryManage` 保持一致。

**验证**：

```bash
cd toy-taro-client
npm run build:weapp
```

---

### 15. 修改我的页面

**文件**：

- `toy-taro-client/src/ui/pages/mine/index.tsx`

**内容**：

- 管理员菜单新增「人员管理」入口。
- 仅 `isAdmin` 为 true 时展示。

**验收**：

- `ADMIN` 用户可见入口。
- `USER` 用户不可见入口。

**验证**：

```bash
cd toy-taro-client
npm run build:weapp
```

---

## 联调验证

### 16. 创建分组注册联调

**步骤**：

1. 新用户打开登录页。
2. 点击注册入口。
3. 选择「创建分组」。
4. 输入分组名。
5. 提交注册。

**验收**：

- 数据库新增 `group`。
- 数据库新增 `user`，`role = ADMIN`。
- 用户进入首页。
- 我的页面展示管理员身份。

---

### 17. 邀请码加入联调

**步骤**：

1. 管理员进入人员管理页。
2. 点击生成邀请码。
3. 新用户在注册页选择「加入分组」。
4. 输入邀请码并提交。

**验收**：

- 新用户加入管理员所在 `group`。
- 新用户 `role = USER`。
- 邀请码记录 `is_active = 0`。
- 同一个邀请码再次使用失败。

---

### 18. 权限管理联调

**步骤**：

1. 管理员进入人员管理页。
2. 将一个普通用户提升为管理员。
3. 将该用户降级回普通用户。
4. 尝试将最后一个管理员降级。

**验收**：

- 普通成员可提升为管理员。
- 管理员可降级为普通用户。
- 最后一个管理员不能被降级。
- 非管理员不能访问人员管理接口。

---

## 最终验证命令

```bash
cd toy-taro-server
npm run lint
npm run test-local
npm run tsc
```

```bash
cd toy-taro-client
npm run lint:eslint
npm test
npm run build:weapp
```
