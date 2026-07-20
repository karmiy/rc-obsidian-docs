# Loop 1 Backend Review Package

## Git status

```
 M toy-taro-server/app/controller/user.ts
 M toy-taro-server/app/router.ts
 M toy-taro-server/app/service/User.ts
 M toy-taro-server/typings/app/model/index.d.ts
 M toy-taro-server/typings/app/service/index.d.ts
?? toy-taro-server/app/model/groupInvite.ts
?? toy-taro-server/database/group_invite.sql
?? toy-taro-server/database/migrations/202607090003_create_group_invite.sql
?? toy-taro-server/test/app/controller/
```

## Diff stat

```
 toy-taro-server/app/controller/user.ts         | 166 ++++++++++
 toy-taro-server/app/router.ts                  |  30 ++
 toy-taro-server/app/service/User.ts            | 432 +++++++++++++++++++++++--
 toy-taro-server/typings/app/model/index.d.ts   |   2 +
 toy-taro-server/typings/app/service/index.d.ts |   4 +-
 5 files changed, 611 insertions(+), 23 deletions(-)
```

## Tracked diff

```diff
diff --git a/toy-taro-server/app/controller/user.ts b/toy-taro-server/app/controller/user.ts
index 35a666b..fe37c77 100644
--- a/toy-taro-server/app/controller/user.ts
+++ b/toy-taro-server/app/controller/user.ts
@@ -4,6 +4,7 @@ import {
   FILE_MODULE_NAME,
   FILE_NAME_PREFIX,
   SERVER_CODE,
+  ROLE,
 } from "../utils/constants";
 import { Logger } from "../utils/logger";
 import { UserEntity } from "../entity/user";
@@ -68,6 +69,86 @@ export default class UserController extends Controller {
     }
   }
 
+  public async registerCreateGroup() {
+    const { ctx } = this;
+    try {
+      const params = ctx.getParams<{
+        code?: string;
+        username?: string;
+        password?: string;
+        groupName?: string;
+      }>();
+      const { groupName = "" } = params;
+
+      if (!groupName.trim()) {
+        ctx.responseFail({
+          code: SERVER_CODE.BAD_REQUEST,
+          message: "请输入分组名称",
+        });
+        return;
+      }
+
+      const data = await ctx.service.user.registerCreateGroup({
+        ...params,
+        groupName,
+      });
+
+      ctx.responseSuccess({
+        data,
+        message: "注册成功",
+      });
+    } catch (error) {
+      const jsError = ctx.toJsError(error);
+      logger
+        .tag("[registerCreateGroup]")
+        .error("register create group error", jsError.message);
+      ctx.responseFail({
+        code: jsError.code,
+        message: jsError?.message,
+      });
+    }
+  }
+
+  public async registerJoinGroup() {
+    const { ctx } = this;
+    try {
+      const params = ctx.getParams<{
+        code?: string;
+        username?: string;
+        password?: string;
+        inviteCode?: string;
+      }>();
+      const { inviteCode = "" } = params;
+
+      if (!inviteCode.trim()) {
+        ctx.responseFail({
+          code: SERVER_CODE.BAD_REQUEST,
+          message: "请输入邀请码",
+        });
+        return;
+      }
+
+      const data = await ctx.service.user.registerJoinGroup({
+        ...params,
+        inviteCode,
+      });
+
+      ctx.responseSuccess({
+        data,
+        message: "注册成功",
+      });
+    } catch (error) {
+      const jsError = ctx.toJsError(error);
+      logger
+        .tag("[registerJoinGroup]")
+        .error("register join group error", jsError.message);
+      ctx.responseFail({
+        code: jsError.code,
+        message: jsError?.message,
+      });
+    }
+  }
+
   public async uploadAvatar() {
     // 微信开发者工具在控制台：
     // 检测头像是否还有效：
@@ -206,4 +287,89 @@ export default class UserController extends Controller {
       });
     }
   }
+
+  public async getMemberList() {
+    const { ctx } = this;
+    try {
+      const list = await ctx.service.user.getMemberList();
+      const memberList: UserEntity[] = list.map((item) => {
+        return this._userModelToEntity(item);
+      });
+
+      ctx.responseSuccess({
+        data: memberList,
+      });
+    } catch (error) {
+      const jsError = ctx.toJsError(error);
+      logger
+        .tag("[getMemberList]")
+        .error("get member list error", jsError.message);
+      ctx.responseFail({
+        code: jsError.code,
+        message: jsError?.message,
+      });
+    }
+  }
+
+  public async updateMemberRole() {
+    const { ctx } = this;
+    try {
+      const { userId, role } = ctx.getParams<{
+        userId?: string;
+        role?: ROLE;
+      }>();
+
+      if (!userId || !role) {
+        ctx.responseFail({
+          code: SERVER_CODE.BAD_REQUEST,
+          message: "缺少成员或角色",
+        });
+        return;
+      }
+
+      if (![ROLE.ADMIN, ROLE.USER].includes(role)) {
+        ctx.responseFail({
+          code: SERVER_CODE.BAD_REQUEST,
+          message: "无效角色",
+        });
+        return;
+      }
+
+      await ctx.service.user.updateMemberRole({ userId, role });
+
+      ctx.responseSuccess({
+        message: "修改成员角色成功",
+      });
+    } catch (error) {
+      const jsError = ctx.toJsError(error);
+      logger
+        .tag("[updateMemberRole]")
+        .error("update member role error", jsError.message);
+      ctx.responseFail({
+        code: jsError.code,
+        message: jsError?.message,
+      });
+    }
+  }
+
+  public async createGroupInvite() {
+    const { ctx } = this;
+    try {
+      const data = await ctx.service.user.createGroupInvite();
+
+      ctx.responseSuccess({
+        data,
+        message: "邀请码生成成功",
+      });
+    } catch (error) {
+      const jsError = ctx.toJsError(error);
+      logger
+        .tag("[createGroupInvite]")
+        .error("create group invite error", jsError.message);
+      ctx.responseFail({
+        code: jsError.code,
+        message: jsError?.message,
+      });
+    }
+  }
 }
diff --git a/toy-taro-server/app/router.ts b/toy-taro-server/app/router.ts
index aef96fc..53f5047 100644
--- a/toy-taro-server/app/router.ts
+++ b/toy-taro-server/app/router.ts
@@ -4,13 +4,43 @@ const getPath = (path: string) => `/v1/toy${path}`;
 
 export default (app: Application) => {
   const { controller, router } = app;
+  const publicUserPaths = [
+    "/user/registerCreateGroup",
+    "/user/registerJoinGroup",
+  ];
+  const authorizationConfig = app.config.authorization as {
+    ignorePaths?: string[];
+  };
+  authorizationConfig.ignorePaths = authorizationConfig.ignorePaths ?? [];
+  publicUserPaths.forEach((path) => {
+    if (!authorizationConfig.ignorePaths!.includes(path)) {
+      authorizationConfig.ignorePaths!.push(path);
+    }
+  });
 
   // user
   router.post(getPath("/user/login"), controller.user.login);
+  router.post(
+    getPath("/user/registerCreateGroup"),
+    controller.user.registerCreateGroup
+  );
+  router.post(
+    getPath("/user/registerJoinGroup"),
+    controller.user.registerJoinGroup
+  );
   router.post(getPath("/user/uploadAvatar"), controller.user.uploadAvatar);
   router.post(getPath("/user/uploadProfile"), controller.user.uploadProfile);
   router.get(getPath("/user/getUserInfo"), controller.user.getUserInfo);
   router.get(getPath("/user/getContactList"), controller.user.getContactList);
+  router.get(getPath("/user/getMemberList"), controller.user.getMemberList);
+  router.post(
+    getPath("/user/updateMemberRole"),
+    controller.user.updateMemberRole
+  );
+  router.post(
+    getPath("/user/createGroupInvite"),
+    controller.user.createGroupInvite
+  );
 
   // product
   router.get(
diff --git a/toy-taro-server/app/service/User.ts b/toy-taro-server/app/service/User.ts
index a0d9f9c..c6179b4 100644
--- a/toy-taro-server/app/service/User.ts
+++ b/toy-taro-server/app/service/User.ts
@@ -1,10 +1,23 @@
 import { Service } from "egg";
+import { createHash, randomBytes } from "crypto";
 import { Logger } from "../utils/logger";
 import { ROLE, SERVER_CODE } from "../utils/constants";
 import { JsError } from "../utils/error";
 import { UserModel } from "../model/user";
+import { GroupInviteModel } from "../model/groupInvite";
 
 const logger = Logger.getLogger("[UserService]");
+const INVITE_CODE_PREFIX = "BANYA";
+const INVITE_CODE_CHARS = "ABCDEFGHJKLMNPQRSTUVWXYZ23456789";
+const INVITE_CODE_SEGMENT_LENGTH = 4;
+const INVITE_CODE_SEGMENT_COUNT = 3;
+
+type UpdatableModel<T> = T & {
+  update(
+    fields: Record<string, any>,
+    options?: Record<string, any>
+  ): Promise<void>;
+};
 
 // 以秒表示或描述时间跨度 zeit / ms 的字符串。如 60，"2 days"，"10h"，"7d"，Expiration time，过期时间
 // 如 10 是 10s，'1000' 是 1s
@@ -14,10 +27,189 @@ const JWT_EXPIRES_IN = "7d";
  * User Service
  */
 export default class User extends Service {
+  private _signToken(user: UserModel) {
+    const { app } = this;
+    return app.jwt.sign(
+      {
+        userId: user.id,
+        openId: user.open_id,
+        username: user.username,
+        groupId: user.group_id,
+        role: user.role,
+      },
+      app.JwtSecret,
+      {
+        expiresIn: JWT_EXPIRES_IN,
+      }
+    );
+  }
+
+  private async _getOpenIdByWxCode(code: string) {
+    const { app } = this;
+    const { AppID, AppSecret } = app;
+
+    const { data, status } = await app.curl<{
+      session_key: string;
+      openid: string;
+      unionid?: string;
+      errcode?: number;
+      errmsg?: string;
+    }>(
+      `https://api.weixin.qq.com/sns/jscode2session?appid=${AppID}&secret=${AppSecret}&js_code=${code}&grant_type=authorization_code`,
+      {
+        dataType: "json",
+      }
+    );
+
+    if (status !== SERVER_CODE.OK) {
+      return Promise.reject(
+        new JsError(SERVER_CODE.INTERNAL_SERVER_ERROR, "微信登录接口请求失败")
+      );
+    }
+
+    const { openid, errcode } = data;
+    if (typeof errcode !== "undefined" || !openid) {
+      return Promise.reject(
+        new JsError(SERVER_CODE.INTERNAL_SERVER_ERROR, "无效的微信 code")
+      );
+    }
+
+    return openid;
+  }
+
+  private async _resolveRegisterIdentity(params: {
+    code?: string;
+    username?: string;
+    password?: string;
+  }) {
+    const { ctx } = this;
+    const code = params.code?.trim();
+    const username = params.username?.trim();
+    const password = params.password ?? "";
+    const openId = code ? await this._getOpenIdByWxCode(code) : undefined;
+
+    if (!openId && !username) {
+      return Promise.reject(
+        new JsError(SERVER_CODE.BAD_REQUEST, "缺少微信 code 或用户名")
+      );
+    }
+
+    if (!openId && !password) {
+      return Promise.reject(new JsError(SERVER_CODE.BAD_REQUEST, "缺少密码"));
+    }
+
+    if (openId) {
+      const existingWxUser = await ctx.model.User.findOne({
+        where: { open_id: openId },
+      });
+      if (existingWxUser) {
+        return Promise.reject(
+          new JsError(SERVER_CODE.BAD_REQUEST, "该账号已注册，请直接登录")
+        );
+      }
+    }
+
+    if (username) {
+      const existingAccountUser = await ctx.model.User.findOne({
+        where: { username },
+      });
+      if (existingAccountUser) {
+        return Promise.reject(
+          new JsError(SERVER_CODE.BAD_REQUEST, "该账号已注册，请直接登录")
+        );
+      }
+    }
+
+    return {
+      openId,
+      username,
+      password: username ? password : undefined,
+    };
+  }
+
+  private _buildUserCreateFields(params: {
+    openId?: string;
+    username?: string;
+    password?: string;
+    groupId: string;
+    role: ROLE;
+  }) {
+    const fields: {
+      open_id?: string;
+      username?: string;
+      password?: string;
+      group_id: string;
+      role: ROLE;
+    } = {
+      group_id: params.groupId,
+      role: params.role,
+    };
+
+    if (params.openId) fields.open_id = params.openId;
+    if (params.username) fields.username = params.username;
+    if (params.password) fields.password = params.password;
+
+    return fields;
+  }
+
+  private _normalizeInviteCode(inviteCode: string) {
+    return inviteCode.trim().replace(/\s+/g, "").toUpperCase();
+  }
+
+  private _hashInviteCode(inviteCode: string) {
+    const normalizedCode = this._normalizeInviteCode(inviteCode);
+    return createHash("sha256")
+      .update(`${normalizedCode}:${this.app.JwtSecret}`)
+      .digest("hex");
+  }
+
+  private _generateInviteCode() {
+    const randomLength = INVITE_CODE_SEGMENT_LENGTH * INVITE_CODE_SEGMENT_COUNT;
+    const bytes = randomBytes(randomLength);
+    const suffix = Array.from(bytes)
+      .map((byte) => INVITE_CODE_CHARS[byte % INVITE_CODE_CHARS.length])
+      .join("");
+    const segmentPattern = new RegExp(
+      `.{1,${INVITE_CODE_SEGMENT_LENGTH}}`,
+      "g"
+    );
+    const segments = suffix.match(segmentPattern) ?? [];
+    return `${INVITE_CODE_PREFIX}-${segments.join("-")}`;
+  }
+
+  private async _getCurrentAdminInfo() {
+    const { ctx } = this;
+    const { userId, groupId } = ctx.getUserInfo();
+
+    if (!userId || !groupId) {
+      return Promise.reject(
+        new JsError(SERVER_CODE.UNAUTHORIZED, "获取用户信息失败，用户未登录")
+      );
+    }
+
+    const admin = await ctx.model.User.findOne({
+      where: {
+        id: userId,
+        group_id: groupId,
+        role: ROLE.ADMIN,
+      },
+    });
+
+    if (!admin) {
+      return Promise.reject(new JsError(SERVER_CODE.FORBIDDEN, "无权限操作"));
+    }
+
+    return {
+      userId,
+      groupId,
+      admin: admin as any as UserModel,
+    };
+  }
+
   /**
    * @description 调用微信官方登录
    * @param code 微信 code
-   * @return
+   * @return {Promise<{ token: string }>} 登录 token
    */
   public async wxLogin(code: string) {
     const { app, ctx } = this;
@@ -74,18 +266,12 @@ export default class User extends Service {
       role: ROLE;
     };
 
-    const token = app.jwt.sign(
-      {
-        userId: id,
-        openId: openid,
-        groupId: group_id,
-        role,
-      },
-      app.JwtSecret,
-      {
-        expiresIn: JWT_EXPIRES_IN,
-      }
-    );
+    const token = this._signToken({
+      id,
+      open_id: openid,
+      group_id,
+      role,
+    } as any as UserModel);
 
     return {
       token,
@@ -94,7 +280,7 @@ export default class User extends Service {
 
   public async accountLogin(params: { username?: string; password?: string }) {
     const { username = "", password = "" } = params;
-    const { app, ctx } = this;
+    const { ctx } = this;
 
     const user = await ctx.model.User.findOne({
       attributes: ["id", "group_id", "role"] as any,
@@ -114,13 +300,12 @@ export default class User extends Service {
       role: ROLE;
     };
 
-    const token = app.jwt.sign(
-      { userId: id, username, groupId: group_id, role },
-      app.JwtSecret,
-      {
-        expiresIn: JWT_EXPIRES_IN,
-      }
-    );
+    const token = this._signToken({
+      id,
+      username,
+      group_id,
+      role,
+    } as any as UserModel);
 
     return {
       token,
@@ -158,6 +343,211 @@ export default class User extends Service {
     }
   }
 
+  public async registerCreateGroup(params: {
+    code?: string;
+    username?: string;
+    password?: string;
+    groupName: string;
+  }) {
+    const { ctx } = this;
+    const groupName = params.groupName.trim();
+    const identity = await this._resolveRegisterIdentity(params);
+    const existingGroup = await ctx.model.Group.findOne({
+      where: { name: groupName },
+    });
+
+    if (existingGroup) {
+      return Promise.reject(
+        new JsError(SERVER_CODE.BAD_REQUEST, "该分组名称已存在")
+      );
+    }
+
+    const transaction = await (ctx.model as any).transaction();
+    try {
+      const group = (await ctx.model.Group.create(
+        { name: groupName },
+        { transaction }
+      )) as any as { id: string };
+      const user = (await ctx.model.User.create(
+        this._buildUserCreateFields({
+          ...identity,
+          groupId: group.id,
+          role: ROLE.ADMIN,
+        }),
+        { transaction }
+      )) as any as UserModel;
+
+      await transaction.commit();
+
+      return {
+        token: this._signToken(user),
+      };
+    } catch (error) {
+      await transaction.rollback();
+      if (error instanceof JsError) {
+        return Promise.reject(error);
+      }
+      return Promise.reject(
+        new JsError(SERVER_CODE.INTERNAL_SERVER_ERROR, "注册失败")
+      );
+    }
+  }
+
+  public async registerJoinGroup(params: {
+    code?: string;
+    username?: string;
+    password?: string;
+    inviteCode: string;
+  }) {
+    const { ctx } = this;
+    const identity = await this._resolveRegisterIdentity(params);
+    const codeHash = this._hashInviteCode(params.inviteCode);
+    const transaction = await (ctx.model as any).transaction();
+
+    try {
+      const invite = (await ctx.model.GroupInvite.findOne({
+        where: {
+          code_hash: codeHash,
+          is_active: 1,
+        },
+        transaction,
+        lock: true,
+      })) as any as UpdatableModel<GroupInviteModel>;
+
+      if (!invite) {
+        throw new JsError(SERVER_CODE.BAD_REQUEST, "邀请码无效或已被使用");
+      }
+
+      const user = (await ctx.model.User.create(
+        this._buildUserCreateFields({
+          ...identity,
+          groupId: invite.group_id,
+          role: ROLE.USER,
+        }),
+        { transaction }
+      )) as any as UserModel;
+
+      await invite.update(
+        {
+          is_active: 0,
+          used_by: user.id,
+          used_at: new Date(),
+        },
+        { transaction }
+      );
+
+      await transaction.commit();
+
+      return {
+        token: this._signToken(user),
+      };
+    } catch (error) {
+      await transaction.rollback();
+      if (error instanceof JsError) {
+        return Promise.reject(error);
+      }
+      return Promise.reject(
+        new JsError(SERVER_CODE.INTERNAL_SERVER_ERROR, "注册失败")
+      );
+    }
+  }
+
+  public async createGroupInvite() {
+    const { ctx } = this;
+    const { userId, groupId } = await this._getCurrentAdminInfo();
+
+    for (let index = 0; index < 5; index++) {
+      const inviteCode = this._generateInviteCode();
+      const codeHash = this._hashInviteCode(inviteCode);
+
+      try {
+        await ctx.model.GroupInvite.create({
+          group_id: groupId,
+          code_hash: codeHash,
+          created_by: userId,
+          is_active: 1,
+        });
+
+        return {
+          inviteCode,
+        };
+      } catch (error) {
+        if ((error as any)?.name !== "SequelizeUniqueConstraintError") {
+          return Promise.reject(
+            new JsError(SERVER_CODE.INTERNAL_SERVER_ERROR, "邀请码生成失败")
+          );
+        }
+      }
+    }
+
+    return Promise.reject(
+      new JsError(SERVER_CODE.INTERNAL_SERVER_ERROR, "邀请码生成失败")
+    );
+  }
+
+  public async getMemberList() {
+    const { ctx } = this;
+    const { groupId } = await this._getCurrentAdminInfo();
+    const users = await ctx.model.User.findAll({
+      where: {
+        group_id: groupId,
+      },
+      order: [["id", "asc"]],
+    });
+
+    return users as any as UserModel[];
+  }
+
+  public async updateMemberRole(params: { userId: string; role: ROLE }) {
+    const { ctx } = this;
+    const { groupId } = await this._getCurrentAdminInfo();
+    const transaction = await (ctx.model as any).transaction();
+
+    try {
+      const groupMembers = (await ctx.model.User.findAll({
+        where: {
+          group_id: groupId,
+        },
+        order: [["id", "asc"]],
+        transaction,
+        lock: true,
+      })) as any as Array<UpdatableModel<UserModel>>;
+      const target = groupMembers.find((member) => {
+        return String(member.id) === String(params.userId);
+      });
+
+      if (!target) {
+        throw new JsError(SERVER_CODE.FORBIDDEN, "成员不存在或无权限操作");
+      }
+
+      if (target.role === ROLE.ADMIN && params.role === ROLE.USER) {
+        const adminCount = groupMembers.filter((member) => {
+          return member.role === ROLE.ADMIN;
+        }).length;
+
+        if (adminCount <= 1) {
+          throw new JsError(SERVER_CODE.BAD_REQUEST, "至少保留一名管理员");
+        }
+      }
+
+      await target.update(
+        {
+          role: params.role,
+        },
+        { transaction }
+      );
+      await transaction.commit();
+    } catch (error) {
+      await transaction.rollback();
+      if (error instanceof JsError) {
+        return Promise.reject(error);
+      }
+      return Promise.reject(
+        new JsError(SERVER_CODE.INTERNAL_SERVER_ERROR, "成员角色更新失败")
+      );
+    }
+  }
+
   async getUserListOnGroup() {
     const { ctx } = this;
     const { groupId } = ctx.getUserInfo();
diff --git a/toy-taro-server/typings/app/model/index.d.ts b/toy-taro-server/typings/app/model/index.d.ts
index 29a9906..89b5904 100644
--- a/toy-taro-server/typings/app/model/index.d.ts
+++ b/toy-taro-server/typings/app/model/index.d.ts
@@ -7,6 +7,7 @@ import ExportCollectionFolder from '../../../app/model/collectionFolder';
 import ExportCoupon from '../../../app/model/coupon';
 import ExportDiscipline from '../../../app/model/discipline';
 import ExportGroup from '../../../app/model/group';
+import ExportGroupInvite from '../../../app/model/groupInvite';
 import ExportLearningContent from '../../../app/model/learningContent';
 import ExportLuckyDraw from '../../../app/model/luckyDraw';
 import ExportLuckyDrawHistory from '../../../app/model/luckyDrawHistory';
@@ -31,6 +32,7 @@ declare module 'egg' {
     Coupon: ReturnType<typeof ExportCoupon>;
     Discipline: ReturnType<typeof ExportDiscipline>;
     Group: ReturnType<typeof ExportGroup>;
+    GroupInvite: ReturnType<typeof ExportGroupInvite>;
     LearningContent: ReturnType<typeof ExportLearningContent>;
     LuckyDraw: ReturnType<typeof ExportLuckyDraw>;
     LuckyDrawHistory: ReturnType<typeof ExportLuckyDrawHistory>;
diff --git a/toy-taro-server/typings/app/service/index.d.ts b/toy-taro-server/typings/app/service/index.d.ts
index f7a3404..a501cb1 100644
--- a/toy-taro-server/typings/app/service/index.d.ts
+++ b/toy-taro-server/typings/app/service/index.d.ts
@@ -6,7 +6,6 @@ type AnyClass = new (...args: any[]) => any;
 type AnyFunc<T = any> = (...args: any[]) => T;
 type CanExportFunc = AnyFunc<Promise<any>> | AnyFunc<IterableIterator<any>>;
 type AutoInstanceType<T, U = T extends CanExportFunc ? T : T extends AnyFunc ? ReturnType<T> : T> = U extends AnyClass ? InstanceType<U> : U;
-import ExportAi from '../../../app/service/ai';
 import ExportCheckIn from '../../../app/service/CheckIn';
 import ExportCoupon from '../../../app/service/Coupon';
 import ExportDiscipline from '../../../app/service/Discipline';
@@ -17,10 +16,10 @@ import ExportPrize from '../../../app/service/Prize';
 import ExportProduct from '../../../app/service/Product';
 import ExportTask from '../../../app/service/Task';
 import ExportUser from '../../../app/service/User';
+import ExportAi from '../../../app/service/ai';
 
 declare module 'egg' {
   interface IService {
-    ai: AutoInstanceType<typeof ExportAi>;
     checkIn: AutoInstanceType<typeof ExportCheckIn>;
     coupon: AutoInstanceType<typeof ExportCoupon>;
     discipline: AutoInstanceType<typeof ExportDiscipline>;
@@ -31,5 +30,6 @@ declare module 'egg' {
     product: AutoInstanceType<typeof ExportProduct>;
     task: AutoInstanceType<typeof ExportTask>;
     user: AutoInstanceType<typeof ExportUser>;
+    ai: AutoInstanceType<typeof ExportAi>;
   }
 }

```

## Untracked backend files

### toy-taro-server/app/model/groupInvite.ts

```
import { Application } from "egg";

export interface GroupInviteModel {
  id: string;
  group_id: string;
  code_hash: string;
  created_by: string;
  used_by?: string;
  used_at?: Date | null;
  is_active: number;
  create_time: Date;
  last_modified_time: Date;
}

export default (app: Application) => {
  const { STRING, INTEGER, DATE, TINYINT } = app.Sequelize;

  const GroupInvite = app.model.define("group_invite", {
    id: {
      type: INTEGER,
      primaryKey: true,
      autoIncrement: true,
      get() {
        return String((this as any).getDataValue("id"));
      },
    },
    group_id: {
      type: INTEGER,
      allowNull: false,
      get() {
        return String((this as any).getDataValue("group_id"));
      },
    },
    code_hash: {
      type: STRING(255),
      allowNull: false,
      unique: true,
    },
    created_by: {
      type: INTEGER,
      allowNull: false,
      get() {
        return String((this as any).getDataValue("created_by"));
      },
    },
    used_by: {
      type: INTEGER,
      allowNull: true,
      get() {
        const value = (this as any).getDataValue("used_by");
        return value ? String(value) : undefined;
      },
    },
    used_at: {
      type: DATE,
      allowNull: true,
    },
    is_active: {
      type: TINYINT,
      defaultValue: 1,
    },
    create_time: {
      type: DATE,
      defaultValue: app.Sequelize.literal("CURRENT_TIMESTAMP"),
    },
    last_modified_time: {
      type: DATE,
      defaultValue: app.Sequelize.literal("CURRENT_TIMESTAMP"),
    },
  });

  (GroupInvite as any).associate = function () {
    (app.model.GroupInvite as any).belongsTo(app.model.Group, {
      foreignKey: "group_id",
      targetKey: "id",
    });
    (app.model.GroupInvite as any).belongsTo(app.model.User, {
      as: "creator",
      foreignKey: "created_by",
      targetKey: "id",
    });
    (app.model.GroupInvite as any).belongsTo(app.model.User, {
      as: "usedUser",
      foreignKey: "used_by",
      targetKey: "id",
    });
  };

  return GroupInvite;
};

```

### toy-taro-server/database/group_invite.sql

```
-- ============================================
-- 群组邀请码 - 数据库表创建脚本
-- 数据库: weapp-toy
-- ============================================

CREATE TABLE IF NOT EXISTS `group_invite` (
  `id` INT AUTO_INCREMENT PRIMARY KEY,
  `group_id` INT NOT NULL COMMENT '所属群组ID',
  `code_hash` VARCHAR(255) NOT NULL UNIQUE COMMENT '邀请码哈希',
  `created_by` INT NOT NULL COMMENT '创建人用户ID',
  `used_by` INT NULL COMMENT '使用人用户ID',
  `used_at` TIMESTAMP NULL COMMENT '使用时间',
  `is_active` TINYINT(1) DEFAULT 1 COMMENT '是否有效(0:否 1:是)',
  `create_time` TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `last_modified_time` TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '最后修改时间',
  CONSTRAINT `group_invite_group_id_fk` FOREIGN KEY (`group_id`) REFERENCES `group`(`id`),
  CONSTRAINT `group_invite_created_by_fk` FOREIGN KEY (`created_by`) REFERENCES `user`(`id`),
  CONSTRAINT `group_invite_used_by_fk` FOREIGN KEY (`used_by`) REFERENCES `user`(`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='群组邀请码表';

```

### toy-taro-server/database/migrations/202607090003_create_group_invite.sql

```
-- Create the one-time group invite table.

-- @include ../group_invite.sql

```

### toy-taro-server/test/app/controller/user.test.ts

```
import * as assert from "assert";
import * as bufferModule from "buffer";
import mock from "egg-mock";
import { ROLE, SERVER_CODE } from "../../../app/utils/constants";

const API_PREFIX = "/v1/toy";
const TEST_GROUP_PREFIX = "loop1-test-";
const TEST_USER_PREFIX = "loop1_test_";

let app: any;
let appReady = false;

function installSlowBufferCompat() {
  const mutableBufferModule = bufferModule as any;
  if (!mutableBufferModule.SlowBuffer) {
    mutableBufferModule.SlowBuffer = mutableBufferModule.Buffer;
  }
}

function uniqueValue(prefix: string) {
  return `${prefix}${Date.now()}_${Math.random().toString(36).slice(2, 8)}`;
}

function assertEqual(actual: any, expected: any, message?: string) {
  assert.strictEqual(
    actual,
    expected,
    message ?? `expected ${actual} to equal ${expected}`
  );
}

function assertNotEqual(actual: any, expected: any, message?: string) {
  assert.notStrictEqual(
    actual,
    expected,
    message ?? `expected ${actual} not to equal ${expected}`
  );
}

function signToken(params: { userId: string; groupId: string; role: ROLE }) {
  const { userId, groupId, role } = params;
  return app.jwt.sign({ userId, groupId, role }, app.JwtSecret, {
    expiresIn: "7d",
  });
}

async function ensureGroupInviteTable() {
  await app.model.query(`
    CREATE TABLE IF NOT EXISTS \`group_invite\` (
      \`id\` INT AUTO_INCREMENT PRIMARY KEY,
      \`group_id\` INT NOT NULL COMMENT '所属群组ID',
      \`code_hash\` VARCHAR(255) NOT NULL UNIQUE COMMENT '邀请码哈希',
      \`created_by\` INT NOT NULL COMMENT '创建人用户ID',
      \`used_by\` INT NULL COMMENT '使用人用户ID',
      \`used_at\` TIMESTAMP NULL COMMENT '使用时间',
      \`is_active\` TINYINT(1) DEFAULT 1 COMMENT '是否有效(0:否 1:是)',
      \`create_time\` TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
      \`last_modified_time\` TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '最后修改时间',
      CONSTRAINT \`group_invite_group_id_fk\` FOREIGN KEY (\`group_id\`) REFERENCES \`group\`(\`id\`),
      CONSTRAINT \`group_invite_created_by_fk\` FOREIGN KEY (\`created_by\`) REFERENCES \`user\`(\`id\`),
      CONSTRAINT \`group_invite_used_by_fk\` FOREIGN KEY (\`used_by\`) REFERENCES \`user\`(\`id\`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='群组邀请码表'
  `);
}

async function cleanupTestRows() {
  await app.model.query(
    `
      DELETE gi FROM \`group_invite\` gi
      LEFT JOIN \`group\` g ON gi.group_id = g.id
      LEFT JOIN \`user\` created_user ON gi.created_by = created_user.id
      LEFT JOIN \`user\` used_user ON gi.used_by = used_user.id
      WHERE g.name LIKE :groupPrefix
        OR created_user.username LIKE :userPrefix
        OR used_user.username LIKE :userPrefix
    `,
    {
      replacements: {
        groupPrefix: `${TEST_GROUP_PREFIX}%`,
        userPrefix: `${TEST_USER_PREFIX}%`,
      },
    }
  );
  await app.model.query("DELETE FROM `user` WHERE username LIKE :userPrefix", {
    replacements: { userPrefix: `${TEST_USER_PREFIX}%` },
  });
  await app.model.query("DELETE FROM `group` WHERE name LIKE :groupPrefix", {
    replacements: { groupPrefix: `${TEST_GROUP_PREFIX}%` },
  });
}

async function createGroupAndUser(params?: { role?: ROLE; label?: string }) {
  const { role = ROLE.ADMIN, label = "member" } = params ?? {};
  const group = await app.model.Group.create({
    name: uniqueValue(`${TEST_GROUP_PREFIX}${label}-`),
  });
  const user = await app.model.User.create({
    username: uniqueValue(`${TEST_USER_PREFIX}${label}_`),
    password: "pwd",
    role,
    group_id: group.id,
  });
  return {
    group,
    user,
    token: signToken({ userId: user.id, groupId: group.id, role }),
  };
}

async function createInviteViaApi(adminToken: string) {
  const res = await app
    .httpRequest()
    .post(`${API_PREFIX}/user/createGroupInvite`)
    .set("Authorization", `Bearer ${adminToken}`)
    .expect(200);

  assertEqual(res.body.code, SERVER_CODE.OK);
  assert.ok(res.body.data.inviteCode);
  assert.deepStrictEqual(Object.keys(res.body.data).sort(), ["inviteCode"]);
  return res.body.data.inviteCode as string;
}

describe("app.controller.user register and member invite", () => {
  before(async () => {
    installSlowBufferCompat();
    app = mock.app();
    await app.ready();
    appReady = true;
    await ensureGroupInviteTable();
    await cleanupTestRows();
  });

  after(async () => {
    if (appReady) {
      await cleanupTestRows();
      await app.close();
    }
  });

  afterEach(mock.restore);

  it("registerCreateGroup creates a new group admin and returns a token", async () => {
    const username = uniqueValue(`${TEST_USER_PREFIX}register_admin_`);
    const groupName = uniqueValue(`${TEST_GROUP_PREFIX}register-`);

    const res = await app
      .httpRequest()
      .post(`${API_PREFIX}/user/registerCreateGroup`)
      .send({ username, password: "pwd", groupName })
      .expect(200);

    assertEqual(res.body.code, SERVER_CODE.OK);
    assert.ok(res.body.data.token);

    const group = await app.model.Group.findOne({ where: { name: groupName } });
    const user = await app.model.User.findOne({ where: { username } });
    const payload = app.jwt.decode(res.body.data.token);

    assert.ok(group);
    assert.ok(user);
    assertEqual(user.role, ROLE.ADMIN);
    assertEqual(user.group_id, group.id);
    assertEqual(payload.userId, user.id);
    assertEqual(payload.groupId, group.id);
  });

  it("registerCreateGroup rejects missing and duplicate group names", async () => {
    const username = uniqueValue(`${TEST_USER_PREFIX}register_missing_group_`);

    const missingGroupName = await app
      .httpRequest()
      .post(`${API_PREFIX}/user/registerCreateGroup`)
      .send({ username, password: "pwd" })
      .expect(200);

    assertEqual(missingGroupName.body.code, SERVER_CODE.BAD_REQUEST);
    assertEqual(missingGroupName.body.message, "请输入分组名称");

    const groupName = uniqueValue(`${TEST_GROUP_PREFIX}duplicate-`);

    await app
      .httpRequest()
      .post(`${API_PREFIX}/user/registerCreateGroup`)
      .send({
        username: uniqueValue(`${TEST_USER_PREFIX}register_duplicate_first_`),
        password: "pwd",
        groupName,
      })
      .expect(200);

    const duplicateGroupName = await app
      .httpRequest()
      .post(`${API_PREFIX}/user/registerCreateGroup`)
      .send({
        username: uniqueValue(`${TEST_USER_PREFIX}register_duplicate_second_`),
        password: "pwd",
        groupName,
      })
      .expect(200);

    assertEqual(duplicateGroupName.body.code, SERVER_CODE.BAD_REQUEST);
    assertEqual(duplicateGroupName.body.message, "该分组名称已存在");
  });

  it("createGroupInvite lets admins create a hashed invite without exposing group id", async () => {
    const { user, group, token } = await createGroupAndUser({
      role: ROLE.ADMIN,
      label: "invite_admin",
    });

    const inviteCode = await createInviteViaApi(token);
    const invite = await app.model.GroupInvite.findOne({
      where: { created_by: user.id },
    });

    assert.ok(/^BANYA-[A-Z0-9]{4}-[A-Z0-9]{4}-[A-Z0-9]{4}$/.test(inviteCode));
    assert.ok(invite);
    assertEqual(invite.group_id, group.id);
    assertEqual(invite.created_by, user.id);
    assertEqual(invite.is_active, 1);
    assertNotEqual(invite.code_hash, inviteCode);
    assertEqual(String(invite.code_hash).includes(inviteCode), false);
  });

  it("registerJoinGroup rejects missing invite codes", async () => {
    const res = await app
      .httpRequest()
      .post(`${API_PREFIX}/user/registerJoinGroup`)
      .send({
        username: uniqueValue(`${TEST_USER_PREFIX}missing_invite_`),
        password: "pwd",
      })
      .expect(200);

    assertEqual(res.body.code, SERVER_CODE.BAD_REQUEST);
    assertEqual(res.body.message, "请输入邀请码");
  });

  it("registerJoinGroup accepts a normalized invite and consumes it", async () => {
    const { group, token } = await createGroupAndUser({
      role: ROLE.ADMIN,
      label: "join_admin",
    });
    const inviteCode = await createInviteViaApi(token);
    const username = uniqueValue(`${TEST_USER_PREFIX}join_user_`);

    const res = await app
      .httpRequest()
      .post(`${API_PREFIX}/user/registerJoinGroup`)
      .send({
        username,
        password: "pwd",
        inviteCode: ` ${inviteCode.toLowerCase()} `,
      })
      .expect(200);

    assertEqual(res.body.code, SERVER_CODE.OK);
    assert.ok(res.body.data.token);

    const user = await app.model.User.findOne({ where: { username } });

```

