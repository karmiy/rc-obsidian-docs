# Loop 2 Frontend Contract Review Package

## Git status

```
 M toy-taro-client/src/app.config.ts
 M toy-taro-client/src/core/api/user.ts
 M toy-taro-client/src/core/entity/user.ts
 M toy-taro-client/src/core/httpRequest/interceptors.ts
 M toy-taro-client/src/core/model/user.ts
 M toy-taro-client/src/core/module/userModule.ts
 M toy-taro-client/src/shared/utils/constants.ts
 M toy-taro-client/src/shared/utils/router.ts
 M toy-taro-client/src/ui/viewModel/user/useUserAction.ts
?? toy-taro-client/src/core/api/__tests__/
?? toy-taro-client/src/core/httpRequest/__tests__/interceptors.spec.ts
?? toy-taro-client/src/core/module/__tests__/
?? toy-taro-client/src/shared/utils/__tests__/router-register-member.spec.ts
?? toy-taro-client/src/ui/viewModel/user/useUserAction.register.spec.ts
```

## Diff stat

```
 toy-taro-client/src/app.config.ts                  |  8 +++
 toy-taro-client/src/core/api/user.ts               | 54 ++++++++++++++++-
 toy-taro-client/src/core/entity/user.ts            |  2 +
 .../src/core/httpRequest/interceptors.ts           | 10 +++-
 toy-taro-client/src/core/model/user.ts             |  1 +
 toy-taro-client/src/core/module/userModule.ts      | 67 +++++++++++++++++++++-
 toy-taro-client/src/shared/utils/constants.ts      |  2 +
 toy-taro-client/src/shared/utils/router.ts         |  2 +
 .../src/ui/viewModel/user/useUserAction.ts         | 42 ++++++++++++--
 9 files changed, 178 insertions(+), 10 deletions(-)
```

## Tracked diff

```diff
diff --git a/toy-taro-client/src/app.config.ts b/toy-taro-client/src/app.config.ts
index 8b7f944..5abf020 100644
--- a/toy-taro-client/src/app.config.ts
+++ b/toy-taro-client/src/app.config.ts
@@ -56,6 +56,14 @@ export default defineAppConfig({
       root: 'ui/pages/login/',
       pages: ['index'],
     },
+    {
+      root: 'ui/pages/register/',
+      pages: ['index'],
+    },
+    {
+      root: 'ui/pages/memberManage/',
+      pages: ['entrance/index'],
+    },
     {
       root: 'ui/pages/checkout/',
       pages: ['index'],
diff --git a/toy-taro-client/src/core/api/user.ts b/toy-taro-client/src/core/api/user.ts
index a42567a..508e094 100644
--- a/toy-taro-client/src/core/api/user.ts
+++ b/toy-taro-client/src/core/api/user.ts
@@ -1,5 +1,5 @@
-import { uploadFile } from '@tarojs/taro';
-import { ContactEntity, UserEntity } from '../entity';
+import { ROLE } from '../constants';
+import { ContactEntity, MemberEntity, UserEntity } from '../entity';
 import { httpRequest } from '../httpRequest';
 import { mock, MOCK_API_NAME } from '../mock';
 
@@ -13,6 +13,23 @@ export type LoginResponse = {
   token: string;
 };
 
+export type RegisterCreateGroupParams = LoginParams & {
+  groupName: string;
+};
+
+export type RegisterJoinGroupParams = LoginParams & {
+  inviteCode: string;
+};
+
+export type UpdateMemberRoleParams = {
+  userId: string;
+  role: ROLE;
+};
+
+export type CreateGroupInviteResponse = {
+  inviteCode: string;
+};
+
 export class UserApi {
   @mock({ name: MOCK_API_NAME.GET_USER_INFO, enable: false })
   static async getUserInfo(): Promise<UserEntity> {
@@ -51,4 +68,37 @@ export class UserApi {
       url: '/user/getContactList',
     });
   }
+
+  static async registerCreateGroup(params: RegisterCreateGroupParams): Promise<LoginResponse> {
+    return httpRequest.post<LoginResponse>({
+      url: '/user/registerCreateGroup',
+      data: params,
+    });
+  }
+
+  static async registerJoinGroup(params: RegisterJoinGroupParams): Promise<LoginResponse> {
+    return httpRequest.post<LoginResponse>({
+      url: '/user/registerJoinGroup',
+      data: params,
+    });
+  }
+
+  static async getMemberList(): Promise<MemberEntity[]> {
+    return httpRequest.get<MemberEntity[]>({
+      url: '/user/getMemberList',
+    });
+  }
+
+  static async updateMemberRole(params: UpdateMemberRoleParams): Promise<void> {
+    return httpRequest.post<void>({
+      url: '/user/updateMemberRole',
+      data: params,
+    });
+  }
+
+  static async createGroupInvite(): Promise<CreateGroupInviteResponse> {
+    return httpRequest.post<CreateGroupInviteResponse>({
+      url: '/user/createGroupInvite',
+    });
+  }
 }
diff --git a/toy-taro-client/src/core/entity/user.ts b/toy-taro-client/src/core/entity/user.ts
index 60686dc..36f0e50 100644
--- a/toy-taro-client/src/core/entity/user.ts
+++ b/toy-taro-client/src/core/entity/user.ts
@@ -10,3 +10,5 @@ export interface UserEntity {
 }
 
 export type ContactEntity = UserEntity;
+
+export type MemberEntity = UserEntity;
diff --git a/toy-taro-client/src/core/httpRequest/interceptors.ts b/toy-taro-client/src/core/httpRequest/interceptors.ts
index ffd61f1..fb5438e 100644
--- a/toy-taro-client/src/core/httpRequest/interceptors.ts
+++ b/toy-taro-client/src/core/httpRequest/interceptors.ts
@@ -6,6 +6,13 @@ import { ERROR_MESSAGE, SERVER_ERROR_CODE } from '../constants';
 
 const logger = Logger.getLogger('[HttpRequestInterceptor]');
 
+const PUBLIC_PATHS = ['/user/login', '/user/registerCreateGroup', '/user/registerJoinGroup'];
+
+const isPublicPath = (url: string) => {
+  const requestPath = url.split('?')[0].split('#')[0];
+  return PUBLIC_PATHS.some(path => requestPath.endsWith(path));
+};
+
 const requestInterceptor = (chain: Chain) => {
   const requestParams = chain.requestParams;
   const { url, header, data } = requestParams;
@@ -13,8 +20,7 @@ const requestInterceptor = (chain: Chain) => {
   // 自动携带 token
   const Authorization = UserStorageManager.getInstance().getUserAuth();
 
-  // 登录接口不需要校验
-  const isIgnorePath = url.includes('login');
+  const isIgnorePath = isPublicPath(url);
   logger.info('requestInterceptor', {
     url,
     data: isIgnorePath ? '[filtered]' : data,
diff --git a/toy-taro-client/src/core/model/user.ts b/toy-taro-client/src/core/model/user.ts
index 3a16038..dc2ed68 100644
--- a/toy-taro-client/src/core/model/user.ts
+++ b/toy-taro-client/src/core/model/user.ts
@@ -12,6 +12,7 @@ export class UserModel {
   @observable
   avatarUrl?: string;
 
+  @observable
   role: ROLE;
 
   @observable
diff --git a/toy-taro-client/src/core/module/userModule.ts b/toy-taro-client/src/core/module/userModule.ts
index 35d3298..00125f5 100644
--- a/toy-taro-client/src/core/module/userModule.ts
+++ b/toy-taro-client/src/core/module/userModule.ts
@@ -1,6 +1,13 @@
 import { eventCenter } from '@tarojs/taro';
 import { JsError } from '@shared/utils/utils';
-import { LoginParams, UserApi } from '../api';
+import {
+  CreateGroupInviteResponse,
+  LoginParams,
+  RegisterCreateGroupParams,
+  RegisterJoinGroupParams,
+  UpdateMemberRoleParams,
+  UserApi,
+} from '../api';
 import { AbstractModule, UserStorageManager } from '../base';
 import {
   ERROR_MESSAGE,
@@ -9,6 +16,7 @@ import {
   SERVER_ERROR_CODE,
   STORE_NAME,
 } from '../constants';
+import { MemberEntity } from '../entity';
 import { storeManager } from '../storeManager';
 
 export class UserModule extends AbstractModule {
@@ -74,6 +82,26 @@ export class UserModule extends AbstractModule {
     }
   }
 
+  async registerCreateGroup(params: RegisterCreateGroupParams) {
+    try {
+      const { token } = await UserApi.registerCreateGroup(params);
+      UserStorageManager.getInstance().setUserAuth(token);
+    } catch (error) {
+      this._logger.error('registerCreateGroup failed', error.message);
+      throw error;
+    }
+  }
+
+  async registerJoinGroup(params: RegisterJoinGroupParams) {
+    try {
+      const { token } = await UserApi.registerJoinGroup(params);
+      UserStorageManager.getInstance().setUserAuth(token);
+    } catch (error) {
+      this._logger.error('registerJoinGroup failed', error.message);
+      throw error;
+    }
+  }
+
   async uploadAvatar(tempUrl: string) {
     // 不再支持 getUserProfile 获取昵称头像了 https://developers.weixin.qq.com/community/develop/doc/00022c683e8a80b29bed2142b56c01
     // 可以单独做个配置页面，用 chooseMedia 和 button 组件的 open-type="chooseAvatar" 让用户选头像， 做个弹框让用户输入昵称
@@ -127,4 +155,41 @@ export class UserModule extends AbstractModule {
     storeManager.refresh(STORE_NAME.CONTACT, contactList);
     storeManager.stopLoading(STORE_NAME.CONTACT);
   }
+
+  async getMemberList(): Promise<MemberEntity[]> {
+    storeManager.startLoading(STORE_NAME.CONTACT);
+    try {
+      const memberList = await UserApi.getMemberList();
+      storeManager.refresh(STORE_NAME.CONTACT, memberList);
+      return memberList;
+    } catch (error) {
+      this._logger.error('getMemberList error', error.message);
+      throw error;
+    } finally {
+      storeManager.stopLoading(STORE_NAME.CONTACT);
+    }
+  }
+
+  async syncMemberList() {
+    return this.getMemberList();
+  }
+
+  async updateMemberRole(params: UpdateMemberRoleParams) {
+    try {
+      await UserApi.updateMemberRole(params);
+      await this.getMemberList();
+    } catch (error) {
+      this._logger.error('updateMemberRole error', error.message);
+      throw error;
+    }
+  }
+
+  async createGroupInvite(): Promise<CreateGroupInviteResponse> {
+    try {
+      return await UserApi.createGroupInvite();
+    } catch (error) {
+      this._logger.error('createGroupInvite error', error.message);
+      throw error;
+    }
+  }
 }
diff --git a/toy-taro-client/src/shared/utils/constants.ts b/toy-taro-client/src/shared/utils/constants.ts
index dd9c170..958695d 100644
--- a/toy-taro-client/src/shared/utils/constants.ts
+++ b/toy-taro-client/src/shared/utils/constants.ts
@@ -28,6 +28,7 @@ export const CONTENT_TYPE_LABELS: Record<ContentType, string> = {
 
 export enum PAGE_ID {
     LOGIN = 'login',
+    REGISTER = 'register',
     HOME = 'home',
     SHOP_CART = 'shopCart',
     CHECKOUT = 'checkout',
@@ -36,6 +37,7 @@ export enum PAGE_ID {
     TASK_CATEGORY_MANAGE = 'taskCategoryManage',
     TASK_FLOW_MANAGE = 'taskFlowManage',
     MINE = 'mine',
+    MEMBER_MANAGE = 'memberManage',
     COUPON = 'coupon',
     COUPON_MANAGE = 'couponManage',
     EXCHANGE_RECORD = 'exchangeRecord',
diff --git a/toy-taro-client/src/shared/utils/router.ts b/toy-taro-client/src/shared/utils/router.ts
index 97ac1c2..617f917 100644
--- a/toy-taro-client/src/shared/utils/router.ts
+++ b/toy-taro-client/src/shared/utils/router.ts
@@ -13,6 +13,8 @@ interface NavigateOptions {
 }
 
 const pagePathMap = new Map<PAGE_ID, string>([
+  [PAGE_ID.REGISTER, 'register'],
+  [PAGE_ID.MEMBER_MANAGE, 'memberManage/entrance'],
   [PAGE_ID.TASK_MANAGE, 'taskManage/entrance'],
   [PAGE_ID.TASK_CATEGORY_MANAGE, 'taskManage/categoryManage'],
   [PAGE_ID.TASK_FLOW_MANAGE, 'taskManage/taskFlowManage'],
diff --git a/toy-taro-client/src/ui/viewModel/user/useUserAction.ts b/toy-taro-client/src/ui/viewModel/user/useUserAction.ts
index e4c9467..8cac943 100644
--- a/toy-taro-client/src/ui/viewModel/user/useUserAction.ts
+++ b/toy-taro-client/src/ui/viewModel/user/useUserAction.ts
@@ -13,10 +13,10 @@ const logger = Logger.getLogger('[useUserAction]');
 export function useUserAction() {
   const [isActionLoading, setIsActionLoading] = useState(false);
 
-  const handleLoginSuccess = useCallback(async () => {
+  const handleLoginSuccess = useCallback(async (successTitle = '登录成功') => {
     try {
       await bootstrap();
-      await showToast({ title: '登录成功' });
+      await showToast({ title: successTitle });
       navigateToPage({ pageName: PAGE_ID.HOME, isRelaunch: true });
     } catch (error) {
       logger.tag('[handleLoginSuccess]').error('reload sdk failed', error);
@@ -25,12 +25,12 @@ export function useUserAction() {
   }, []);
 
   const handleLogin = useCallback(
-    async (options: { handler: () => Promise<void>; logName: string }) => {
-      const { handler, logName } = options;
+    async (options: { handler: () => Promise<void>; logName: string; successTitle?: string }) => {
+      const { handler, logName, successTitle } = options;
       try {
         setIsActionLoading(true);
         await handler();
-        await handleLoginSuccess();
+        await handleLoginSuccess(successTitle);
       } catch (error) {
         logger.tag('[handleLogin]').error(`${logName} login failed`, error);
         showToast({ title: error.message ?? '登录失败' });
@@ -41,6 +41,36 @@ export function useUserAction() {
     [handleLoginSuccess],
   );
 
+  const handleRegisterCreateGroup = useCallback(
+    async (params: { groupName: string }) => {
+      const { groupName } = params;
+      await handleLogin({
+        handler: async () => {
+          const { code } = await taroLogin();
+          await sdk.modules.user.registerCreateGroup({ code, groupName });
+        },
+        logName: 'registerCreateGroup',
+        successTitle: '注册成功',
+      });
+    },
+    [handleLogin],
+  );
+
+  const handleRegisterJoinGroup = useCallback(
+    async (params: { inviteCode: string }) => {
+      const { inviteCode } = params;
+      await handleLogin({
+        handler: async () => {
+          const { code } = await taroLogin();
+          await sdk.modules.user.registerJoinGroup({ code, inviteCode });
+        },
+        logName: 'registerJoinGroup',
+        successTitle: '注册成功',
+      });
+    },
+    [handleLogin],
+  );
+
   const handleWxLogin = useCallback(async () => {
     handleLogin({
       handler: async () => {
@@ -120,6 +150,8 @@ export function useUserAction() {
     isActionLoading,
     handleWxLogin,
     handleAccountLogin,
+    handleRegisterCreateGroup,
+    handleRegisterJoinGroup,
     handleChooseAvatar,
     handleUpdateNickName,
     handleLogout,

```

## Untracked frontend files

### toy-taro-client/src/core/api/__tests__/user-register-member.spec.ts

```
import { ROLE } from '../../constants';
import { httpRequest } from '../../httpRequest';
import { UserApi } from '../user';

jest.mock('@tarojs/taro', () => ({
  uploadFile: jest.fn(),
}));

jest.mock('../../httpRequest', () => ({
  httpRequest: {
    get: jest.fn(),
    post: jest.fn(),
    postFormDataFile: jest.fn(),
  },
}));

jest.mock('../../mock', () => ({
  MOCK_API_NAME: {
    GET_USER_INFO: 'GET_USER_INFO',
    USER_LOGIN: 'USER_LOGIN',
    UPLOAD_AVATAR: 'UPLOAD_AVATAR',
    UPLOAD_PROFILE: 'UPLOAD_PROFILE',
    GET_CONTACT_LIST: 'GET_CONTACT_LIST',
  },
  mock:
    () =>
    (_target: unknown, _propertyKey: string, descriptor: PropertyDescriptor) =>
      descriptor,
}));

describe('UserApi register/member contract', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('posts registerCreateGroup with identity and group name', async () => {
    jest.mocked(httpRequest.post).mockResolvedValue({ token: 'token-create' });

    const result = await UserApi.registerCreateGroup({
      code: 'wx-code',
      groupName: '伴芽家庭',
    });

    expect(httpRequest.post).toHaveBeenCalledWith({
      url: '/user/registerCreateGroup',
      data: {
        code: 'wx-code',
        groupName: '伴芽家庭',
      },
    });
    expect(result).toEqual({ token: 'token-create' });
  });

  it('posts registerJoinGroup with identity and invite code', async () => {
    jest.mocked(httpRequest.post).mockResolvedValue({ token: 'token-join' });

    const result = await UserApi.registerJoinGroup({
      username: 'parent',
      password: 'pass',
      inviteCode: 'BANYA-8F6Q',
    });

    expect(httpRequest.post).toHaveBeenCalledWith({
      url: '/user/registerJoinGroup',
      data: {
        username: 'parent',
        password: 'pass',
        inviteCode: 'BANYA-8F6Q',
      },
    });
    expect(result).toEqual({ token: 'token-join' });
  });

  it('gets member list from the admin member endpoint', async () => {
    const members = [{ id: '1', name: '管理员', role: ROLE.ADMIN }];
    jest.mocked(httpRequest.get).mockResolvedValue(members);

    const result = await UserApi.getMemberList();

    expect(httpRequest.get).toHaveBeenCalledWith({
      url: '/user/getMemberList',
    });
    expect(result).toBe(members);
  });

  it('posts member role updates with user id and role', async () => {
    jest.mocked(httpRequest.post).mockResolvedValue(undefined);

    await UserApi.updateMemberRole({
      userId: '2',
      role: ROLE.ADMIN,
    });

    expect(httpRequest.post).toHaveBeenCalledWith({
      url: '/user/updateMemberRole',
      data: {
        userId: '2',
        role: ROLE.ADMIN,
      },
    });
  });

  it('posts invite creation and returns the plaintext invite code', async () => {
    jest.mocked(httpRequest.post).mockResolvedValue({ inviteCode: 'BANYA-8F6Q' });

    const result = await UserApi.createGroupInvite();

    expect(httpRequest.post).toHaveBeenCalledWith({
      url: '/user/createGroupInvite',
    });
    expect(result).toEqual({ inviteCode: 'BANYA-8F6Q' });
  });
});

```

### toy-taro-client/src/core/httpRequest/__tests__/interceptors.spec.ts

```
const mockGetUserAuth = jest.fn();

jest.mock(
  '@shared/utils/logger',
  () => ({
    Logger: {
      getLogger: jest.fn(() => ({
        info: jest.fn(),
        error: jest.fn(),
      })),
    },
  }),
  { virtual: true },
);

jest.mock(
  '@shared/utils/utils',
  () => ({
    JsError: class JsError extends Error {
      code: string | number;

      constructor(code: string | number, message: string) {
        super(message);
        this.code = code;
      }
    },
  }),
  { virtual: true },
);

jest.mock('../../base', () => ({
  UserStorageManager: {
    getInstance: jest.fn(() => ({
      getUserAuth: mockGetUserAuth,
    })),
  },
}));

import { ERROR_MESSAGE, SERVER_ERROR_CODE } from '../../constants';
import { interceptors } from '../interceptors';

const requestInterceptor = interceptors[0];

const createRequestChain = (url: string) => ({
  requestParams: {
    url,
    header: {},
    data: { sample: true },
  },
  proceed: jest.fn().mockResolvedValue({ ok: true }),
});

describe('requestInterceptor auth public paths', () => {
  beforeEach(() => {
    jest.clearAllMocks();
    mockGetUserAuth.mockReturnValue(undefined);
  });

  it.each(['/user/registerCreateGroup', '/user/registerJoinGroup'])(
    'allows %s without an auth token',
    async url => {
      const chain = createRequestChain(url);

      await expect(requestInterceptor(chain as any)).resolves.toEqual({ ok: true });

      expect(chain.proceed).toHaveBeenCalledWith({
        url,
        header: {},
        data: { sample: true },
      });
    },
  );

  it('rejects protected routes without an auth token', async () => {
    const chain = createRequestChain('/user/getMemberList');

    await expect(requestInterceptor(chain as any)).rejects.toMatchObject({
      code: SERVER_ERROR_CODE.LOGIN_EXPIRED,
      message: ERROR_MESSAGE.LOGIN_EXPIRED,
    });

    expect(chain.proceed).not.toHaveBeenCalled();
  });
});

```

### toy-taro-client/src/core/module/__tests__/user-register-member.spec.ts

```
import { ROLE, STORE_NAME } from '../../constants';
import { UserApi } from '../../api';
import { UserStorageManager } from '../../base';
import { storeManager } from '../../storeManager';
import { UserModule } from '../userModule';

const mockSetUserAuth = jest.fn();

jest.mock('@tarojs/taro', () => ({
  eventCenter: {
    on: jest.fn(),
    off: jest.fn(),
  },
}));

jest.mock('@shared/utils/utils', () => ({
  JsError: class JsError extends Error {
    code: number;

    constructor(code: number, message: string) {
      super(message);
      this.code = code;
    }
  },
}), { virtual: true });

jest.mock('../../api', () => ({
  UserApi: {
    registerCreateGroup: jest.fn(),
    registerJoinGroup: jest.fn(),
    getMemberList: jest.fn(),
    updateMemberRole: jest.fn(),
    createGroupInvite: jest.fn(),
  },
}));

jest.mock('../../base', () => ({
  AbstractModule: class AbstractModule {
    protected _sdk: unknown;

    constructor(sdk: unknown) {
      this._sdk = sdk;
    }

    protected get _logger() {
      return {
        info: jest.fn(),
        error: jest.fn(),
      };
    }
  },
  UserStorageManager: {
    getInstance: jest.fn(() => ({
      init: jest.fn(),
      dispose: jest.fn(),
      getUserAuth: jest.fn(() => 'auth-token'),
      getUserInfo: jest.fn(),
      setUserAuth: mockSetUserAuth,
      isAdmin: false,
    })),
  },
}));

jest.mock('../../storeManager', () => ({
  storeManager: {
    startLoading: jest.fn(),
    stopLoading: jest.fn(),
    refresh: jest.fn(),
    emitUpdate: jest.fn(),
    get: jest.fn(),
  },
}));

describe('UserModule register/member contract', () => {
  const createModule = () => new UserModule({} as never);

  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('stores the returned token after creating a group during registration', async () => {
    jest.mocked(UserApi.registerCreateGroup).mockResolvedValue({ token: 'token-create' });

    await createModule().registerCreateGroup({
      code: 'wx-code',
      groupName: '伴芽家庭',
    });

    expect(UserApi.registerCreateGroup).toHaveBeenCalledWith({
      code: 'wx-code',
      groupName: '伴芽家庭',
    });
    expect(UserStorageManager.getInstance().setUserAuth).toHaveBeenCalledWith('token-create');
  });

  it('stores the returned token after joining a group during registration', async () => {
    jest.mocked(UserApi.registerJoinGroup).mockResolvedValue({ token: 'token-join' });

    await createModule().registerJoinGroup({
      username: 'parent',
      password: 'pass',
      inviteCode: 'BANYA-8F6Q',
    });

    expect(UserApi.registerJoinGroup).toHaveBeenCalledWith({
      username: 'parent',
      password: 'pass',
      inviteCode: 'BANYA-8F6Q',
    });
    expect(UserStorageManager.getInstance().setUserAuth).toHaveBeenCalledWith('token-join');
  });

  it('loads members into the contact store with loading lifecycle', async () => {
    const members = [
      { id: '1', name: '管理员', role: ROLE.ADMIN },
      { id: '2', name: '成员', role: ROLE.USER },
    ];
    jest.mocked(UserApi.getMemberList).mockResolvedValue(members);

    const result = await createModule().getMemberList();

    expect(storeManager.startLoading).toHaveBeenCalledWith(STORE_NAME.CONTACT);
    expect(UserApi.getMemberList).toHaveBeenCalled();
    expect(storeManager.refresh).toHaveBeenCalledWith(STORE_NAME.CONTACT, members);
    expect(storeManager.stopLoading).toHaveBeenCalledWith(STORE_NAME.CONTACT);
    expect(result).toBe(members);
  });

  it('stops member loading when member list refresh fails', async () => {
    const error = new Error('network failed');
    jest.mocked(UserApi.getMemberList).mockRejectedValue(error);

    await expect(createModule().getMemberList()).rejects.toBe(error);

    expect(storeManager.startLoading).toHaveBeenCalledWith(STORE_NAME.CONTACT);
    expect(storeManager.stopLoading).toHaveBeenCalledWith(STORE_NAME.CONTACT);
    expect(storeManager.refresh).not.toHaveBeenCalled();
  });

  it('refreshes the member list after updating a member role', async () => {
    const members = [{ id: '2', name: '成员', role: ROLE.ADMIN }];
    jest.mocked(UserApi.updateMemberRole).mockResolvedValue(undefined);
    jest.mocked(UserApi.getMemberList).mockResolvedValue(members);

    await createModule().updateMemberRole({
      userId: '2',
      role: ROLE.ADMIN,
    });

    expect(UserApi.updateMemberRole).toHaveBeenCalledWith({
      userId: '2',
      role: ROLE.ADMIN,
    });
    expect(storeManager.refresh).toHaveBeenCalledWith(STORE_NAME.CONTACT, members);
  });

  it('returns the invite code generated for the current group', async () => {
    jest.mocked(UserApi.createGroupInvite).mockResolvedValue({ inviteCode: 'BANYA-8F6Q' });

    const result = await createModule().createGroupInvite();

    expect(UserApi.createGroupInvite).toHaveBeenCalled();
    expect(result).toEqual({ inviteCode: 'BANYA-8F6Q' });
  });
});

```

### toy-taro-client/src/shared/utils/__tests__/router-register-member.spec.ts

```
import { navigateTo } from '@tarojs/taro';
import { PAGE_ID } from '../constants';
import { navigateToPage } from '../router';

jest.mock(
  '@core/entity',
  () => ({
    ContentType: {
      WORD: 'WORD',
      SENTENCE: 'SENTENCE',
      POEM: 'POEM',
      ESSAY: 'ESSAY',
    },
  }),
  { virtual: true },
);

jest.mock('@tarojs/taro', () => ({
  getCurrentPages: jest.fn(() => []),
  navigateTo: jest.fn(),
  redirectTo: jest.fn(),
  reLaunch: jest.fn(),
  switchTab: jest.fn(),
}));

describe('register/member route contract', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('navigates to the register page package path', () => {
    navigateToPage({ pageName: PAGE_ID.REGISTER });

    expect(navigateTo).toHaveBeenCalledWith({
      url: '/ui/pages/register/index',
    });
  });

  it('navigates to the member management entrance package path', () => {
    navigateToPage({ pageName: PAGE_ID.MEMBER_MANAGE });

    expect(navigateTo).toHaveBeenCalledWith({
      url: '/ui/pages/memberManage/entrance/index',
    });
  });
});

```

### toy-taro-client/src/ui/viewModel/user/useUserAction.register.spec.ts

```
import { login as taroLogin } from '@tarojs/taro';
import { showToast } from '@shared/utils/operateFeedback';
import { navigateToPage } from '@shared/utils/router';
import { sdk } from '@core';
import { bootstrap } from '@ui/bootstrap';
import { useUserAction } from './useUserAction';

const mockSetActionLoading = jest.fn();

jest.mock('react', () => ({
  useState: jest.fn(() => [false, mockSetActionLoading]),
  useCallback: (callback: unknown) => callback,
}));

jest.mock('@tarojs/components', () => ({}), { virtual: true });

jest.mock('@tarojs/taro', () => ({
  login: jest.fn(),
}));

jest.mock('@shared/utils/constants', () => ({
  PAGE_ID: {
    HOME: 'home',
  },
}), { virtual: true });

jest.mock('@shared/utils/logger', () => ({
  Logger: {
    getLogger: jest.fn(() => ({
      tag: jest.fn(() => ({
        error: jest.fn(),
      })),
      error: jest.fn(),
    })),
  },
}), { virtual: true });

jest.mock('@shared/utils/operateFeedback', () => ({
  showToast: jest.fn(),
}), { virtual: true });

jest.mock('@shared/utils/router', () => ({
  navigateToPage: jest.fn(),
}), { virtual: true });

jest.mock('@core', () => ({
  sdk: {
    modules: {
      user: {
        registerCreateGroup: jest.fn(),
        registerJoinGroup: jest.fn(),
      },
    },
  },
}), { virtual: true });

jest.mock('@ui/bootstrap', () => ({
  bootstrap: jest.fn(),
  unBootstrap: jest.fn(),
}), { virtual: true });

describe('useUserAction registration actions', () => {
  beforeEach(() => {
    jest.clearAllMocks();
    jest.mocked(taroLogin).mockResolvedValue({ code: 'wx-code' } as never);
    jest.mocked(showToast).mockResolvedValue(undefined);
    jest.mocked(bootstrap).mockResolvedValue(undefined);
    jest.mocked(sdk.modules.user.registerCreateGroup).mockResolvedValue(undefined);
    jest.mocked(sdk.modules.user.registerJoinGroup).mockResolvedValue(undefined);
  });

  it('creates a group with taro login, bootstraps, toasts, and relaunches home', async () => {
    const { handleRegisterCreateGroup } = useUserAction();

    await handleRegisterCreateGroup({ groupName: '伴芽家庭' });

    expect(mockSetActionLoading).toHaveBeenNthCalledWith(1, true);
    expect(taroLogin).toHaveBeenCalled();
    expect(sdk.modules.user.registerCreateGroup).toHaveBeenCalledWith({
      code: 'wx-code',
      groupName: '伴芽家庭',
    });
    expect(bootstrap).toHaveBeenCalled();
    expect(showToast).toHaveBeenCalledWith({ title: '注册成功' });
    expect(navigateToPage).toHaveBeenCalledWith({
      pageName: 'home',
      isRelaunch: true,
    });
    expect(mockSetActionLoading).toHaveBeenLastCalledWith(false);
  });

  it('joins a group with taro login before bootstrapping into home', async () => {
    const { handleRegisterJoinGroup } = useUserAction();

    await handleRegisterJoinGroup({ inviteCode: 'BANYA-8F6Q' });

    expect(taroLogin).toHaveBeenCalled();
    expect(sdk.modules.user.registerJoinGroup).toHaveBeenCalledWith({
      code: 'wx-code',
      inviteCode: 'BANYA-8F6Q',
    });
    expect(bootstrap).toHaveBeenCalled();
    expect(navigateToPage).toHaveBeenCalledWith({
      pageName: 'home',
      isRelaunch: true,
    });
    expect(mockSetActionLoading).toHaveBeenLastCalledWith(false);
  });

  it('shows the server error and does not relaunch when joining fails', async () => {
    jest
      .mocked(sdk.modules.user.registerJoinGroup)
      .mockRejectedValue(new Error('邀请码无效或已被使用'));
    const { handleRegisterJoinGroup } = useUserAction();

    await handleRegisterJoinGroup({ inviteCode: 'BANYA-OLD' });

    expect(showToast).toHaveBeenCalledWith({ title: '邀请码无效或已被使用' });
    expect(bootstrap).not.toHaveBeenCalled();
    expect(navigateToPage).not.toHaveBeenCalled();
    expect(mockSetActionLoading).toHaveBeenLastCalledWith(false);
  });
});

```

