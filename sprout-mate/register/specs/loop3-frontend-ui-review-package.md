# Loop 3 Frontend UI Review Package

## Re-review Update After NEEDS_FIX

Reviewer finding fixed:

- Member management now uses the shared `useSyncOnPageShow` path instead of manual `useDidShow` and custom refresh props.
- `useSyncOnPageShow` now includes `PAGE_ID.MEMBER_MANAGE` and calls `sdk.modules.user.syncMemberList()` through the existing `syncContext` / `syncApi` pattern.
- `toy-taro-client/src/ui/pages/memberManage/entrance/index.tsx` now calls `useSyncOnPageShow()` normally and passes `scrollViewRefreshProps` directly to `ConfigListPanel`.
- Added `toy-taro-client/src/ui/hooks/useSyncOnPageShow.spec.ts` coverage for member-management shared sync.

Re-verification:

- `cd toy-taro-client && npx eslint src/ui/hooks/useSyncOnPageShow.ts src/ui/pages/memberManage/entrance/index.tsx src/ui/hooks/useSyncOnPageShow.spec.ts --ext .ts,.tsx` passed.
- `cd toy-taro-client && npm test -- --runInBand` passed 10 suites / 33 tests.
- `cd toy-taro-client && npm run build:weapp` passed; only existing Browserslist, Tailwind content, and bundle-size warnings were reported.
- `git diff --check` passed.

## Git status (UI scope)

```
 M toy-taro-client/src/app.config.ts
 M toy-taro-client/src/ui/pages/login/index.module.scss
 M toy-taro-client/src/ui/pages/login/index.tsx
 M toy-taro-client/src/ui/pages/mine/index.tsx
 M toy-taro-client/src/ui/viewModel/user/index.ts
 M toy-taro-client/src/ui/viewModel/user/useUserAction.ts
?? toy-taro-client/src/ui/pages/memberManage/
?? toy-taro-client/src/ui/pages/register/
?? toy-taro-client/src/ui/viewModel/user/useMemberManage.spec.ts
?? toy-taro-client/src/ui/viewModel/user/useMemberManage.ts
?? toy-taro-client/src/ui/viewModel/user/useUserAction.register.spec.ts
```

## Diff stat (UI scope)

```
 toy-taro-client/src/app.config.ts                  |  8 +++++
 .../src/ui/pages/login/index.module.scss           | 19 ++++++++++
 toy-taro-client/src/ui/pages/login/index.tsx       | 12 +++++--
 toy-taro-client/src/ui/pages/mine/index.tsx        |  5 +++
 toy-taro-client/src/ui/viewModel/user/index.ts     |  1 +
 .../src/ui/viewModel/user/useUserAction.ts         | 42 +++++++++++++++++++---
 6 files changed, 80 insertions(+), 7 deletions(-)
```

## Tracked diff (UI scope)

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
diff --git a/toy-taro-client/src/ui/pages/login/index.module.scss b/toy-taro-client/src/ui/pages/login/index.module.scss
index 661fa97..2dd678b 100644
--- a/toy-taro-client/src/ui/pages/login/index.module.scss
+++ b/toy-taro-client/src/ui/pages/login/index.module.scss
@@ -125,4 +125,23 @@
       margin-left: px(4);
     }
   }
+
+  .registerEntry {
+    display: flex;
+    align-items: center;
+    justify-content: center;
+    margin-top: px(16);
+    font-size: px(13);
+    line-height: 1.45;
+  }
+
+  .registerText {
+    color: $--text-color-secondary;
+  }
+
+  .registerAction {
+    margin-left: px(4);
+    color: $--color-red;
+    font-weight: 700;
+  }
 }
diff --git a/toy-taro-client/src/ui/pages/login/index.tsx b/toy-taro-client/src/ui/pages/login/index.tsx
index e6f7ff6..256b5a1 100644
--- a/toy-taro-client/src/ui/pages/login/index.tsx
+++ b/toy-taro-client/src/ui/pages/login/index.tsx
@@ -1,11 +1,12 @@
 import { Fragment, useState } from 'react';
 import { Image, Text, View } from '@tarojs/components';
+import { navigateToPage } from '@shared/utils/router';
 import { Button, FallbackImage, Icon, Input, SafeAreaBar, WhiteSpace } from '@ui/components';
 import { FormItem } from '@ui/container';
-import { useUserAction } from '@ui/viewModel';
-import { COLOR_VARIABLES } from '@/shared/utils/constants';
 import banyaLogo from '@ui/images/brand/banya-logo.png';
 import growthHero from '@ui/images/brand/growth-hero.jpg';
+import { useUserAction } from '@ui/viewModel';
+import { COLOR_VARIABLES, PAGE_ID } from '@/shared/utils/constants';
 import { LOGIN_PAGE_STATUS } from './constants';
 import styles from './index.module.scss';
 
@@ -92,6 +93,13 @@ export default function () {
             >
               账号密码登录
             </Button>
+            <View
+              className={styles.registerEntry}
+              onClick={() => navigateToPage({ pageName: PAGE_ID.REGISTER })}
+            >
+              <Text className={styles.registerText}>还没有成长空间？</Text>
+              <Text className={styles.registerAction}>去注册</Text>
+            </View>
           </Fragment>
         ) : null}
       </View>
diff --git a/toy-taro-client/src/ui/pages/mine/index.tsx b/toy-taro-client/src/ui/pages/mine/index.tsx
index 71a60a6..34b8046 100644
--- a/toy-taro-client/src/ui/pages/mine/index.tsx
+++ b/toy-taro-client/src/ui/pages/mine/index.tsx
@@ -55,6 +55,11 @@ function Mine() {
     );
     if (isAdmin) {
       list.push(
+        {
+          title: '人员管理',
+          emoji: '👥',
+          onClick: () => navigateToPage({ pageName: PAGE_ID.MEMBER_MANAGE }),
+        },
         {
           title: '商品录入',
           emoji: '🎁',
diff --git a/toy-taro-client/src/ui/viewModel/user/index.ts b/toy-taro-client/src/ui/viewModel/user/index.ts
index 8b94f67..10639cf 100644
--- a/toy-taro-client/src/ui/viewModel/user/index.ts
+++ b/toy-taro-client/src/ui/viewModel/user/index.ts
@@ -1,3 +1,4 @@
 export * from './useUserInfo';
 export * from './useUserAction';
 export * from './useContactList';
+export * from './useMemberManage';
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

## Untracked UI files

### toy-taro-client/src/ui/pages/memberManage/entrance/index.config.ts

```
export default definePageConfig({
  navigationBarTitleText: '人员管理',
  disableScroll: true,
});

```

### toy-taro-client/src/ui/pages/memberManage/entrance/index.module.scss

```
@import '@ui/styles/index.scss';

.memberItem {
  display: flex;
  align-items: center;
  width: 100%;
  min-width: 0;
}

.avatar {
  flex-shrink: 0;
  width: px(42);
  height: px(42);
  border-radius: 50%;
  overflow: hidden;
  background: $--fill-fallback;
}

.memberInfo {
  flex: 1;
  min-width: 0;
  margin-left: px(10);
  display: flex;
  flex-direction: column;
}

.memberName {
  color: $--text-color-base;
  font-size: px(14);
  line-height: 1.35;
  font-weight: 700;
  @include single-line-ellipsis;
}

.memberMeta {
  margin-top: px(3);
  color: $--text-color-secondary;
  font-size: px(12);
  line-height: 1.35;
  font-weight: 400;
  @include single-line-ellipsis;
}

.roleTag {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  min-width: px(58);
  height: px(26);
  padding: 0 px(9);
  border-radius: px(13);
  border: px(1) solid $--color-border-base;
  color: $--text-color-secondary;
  background: $--fill-default-glass;
  font-size: px(12);
  font-weight: 700;

  &.isAdmin {
    border-color: transparent;
    color: $--text-color-base-inverse;
    background: $--gradient-primary;
  }
}

.floatContent {
  padding: px(16);
}

.inviteCodeBox {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: px(112);
  @include glass-card(px(18), 0.86);
}

.inviteLabel {
  color: $--text-color-secondary;
  font-size: px(12);
  line-height: 1.4;
}

.inviteCode {
  margin-top: px(8);
  color: $--text-color-base;
  font-size: px(24);
  line-height: 1.2;
  font-weight: 850;
  letter-spacing: 0;
}

.roleMember {
  display: flex;
  align-items: center;
  padding: px(12);
  border-radius: px(16);
  background: $--fill-default-glass;
  border: px(1) solid $--color-border-base;
}

.roleAvatar {
  flex-shrink: 0;
  width: px(46);
  height: px(46);
  border-radius: 50%;
  overflow: hidden;
  background: $--fill-fallback;
}

.roleOptions {
  display: flex;
  flex-direction: column;
  gap: px(12);
}

.roleOption {
  width: 100%;
}

.roleDesc {
  display: flex;
  align-items: center;
  margin-top: px(7);
  color: $--text-color-secondary;
  font-size: px(12);
  line-height: 1.4;

  text {
    margin-left: px(5);
  }
}

```

### toy-taro-client/src/ui/pages/memberManage/entrance/index.tsx

```
import { Fragment, useCallback, useMemo, useState } from 'react';
import { Text, View } from '@tarojs/components';
import { setClipboardData, useDidShow } from '@tarojs/taro';
import clsx from 'clsx';
import { COLOR_VARIABLES } from '@shared/utils/constants';
import { showToast } from '@shared/utils/operateFeedback';
import { ROLE } from '@core';
import { Button, FallbackImage, FloatLayout, Icon, WhiteSpace } from '@ui/components';
import { ConfigListPanel } from '@ui/container';
import { useSyncOnPageShow } from '@ui/hooks';
import { useMemberManage } from '@ui/viewModel';
import styles from './index.module.scss';

const ROLE_OPTIONS = [
  {
    value: ROLE.ADMIN,
    label: '管理员',
    desc: '可邀请成员和调整权限',
  },
  {
    value: ROLE.USER,
    label: '普通成员',
    desc: '可参与任务和奖励',
  },
];

const ROLE_LABEL: Record<ROLE, string> = {
  [ROLE.ADMIN]: '管理员',
  [ROLE.USER]: '普通成员',
};

export default function () {
  const { scrollViewRefreshProps } = useSyncOnPageShow({ enableSyncOnPageShow: false });
  const {
    memberList,
    loading,
    inviteCode,
    isRoleActionLoading,
    isInviteActionLoading,
    isRefreshing,
    refreshMembers,
    handleUpdateMemberRole,
    handleCreateInvite,
    handleClearInviteCode,
  } = useMemberManage();
  const [showInviteModal, setShowInviteModal] = useState(false);
  const [showRoleModal, setShowRoleModal] = useState(false);
  const [editMemberId, setEditMemberId] = useState<string>();

  useDidShow(() => {
    refreshMembers();
  });

  const currentMember = useMemo(() => {
    return memberList.find(item => item.id === editMemberId);
  }, [editMemberId, memberList]);

  const memberScrollViewProps = useMemo(() => {
    return {
      ...scrollViewRefreshProps,
      refresherTriggered: isRefreshing,
      onRefresherRefresh: refreshMembers,
    };
  }, [isRefreshing, refreshMembers, scrollViewRefreshProps]);

  const handleOpenInvite = useCallback(() => {
    setShowInviteModal(true);
  }, []);

  const handleCloseInvite = useCallback(() => {
    setShowInviteModal(false);
    handleClearInviteCode();
  }, [handleClearInviteCode]);

  const handleOpenRole = useCallback((id: string) => {
    setEditMemberId(id);
    setShowRoleModal(true);
  }, []);

  const handleCloseRole = useCallback(() => {
    setShowRoleModal(false);
    setEditMemberId(undefined);
  }, []);

  const handleRoleChange = useCallback(
    async (role: ROLE) => {
      if (!currentMember || role === currentMember.role) {
        return;
      }
      const success = await handleUpdateMemberRole({
        userId: currentMember.id,
        role,
      });
      if (success) {
        handleCloseRole();
      }
    },
    [currentMember, handleCloseRole, handleUpdateMemberRole],
  );

  const handleGenerateInvite = useCallback(async () => {
    await handleCreateInvite();
  }, [handleCreateInvite]);

  const handleCopyInvite = useCallback(async () => {
    if (!inviteCode) {
      return;
    }
    try {
      await setClipboardData({ data: inviteCode });
      await showToast({ title: '邀请码已复制' });
    } catch {
      await showToast({ title: '复制失败' });
    }
  }, [inviteCode]);

  const renderContent = useCallback(
    ({ index }: { index: number }) => {
      const member = memberList[index];
      const isAdmin = member.role === ROLE.ADMIN;
      return (
        <View className={styles.memberItem}>
          <FallbackImage src={member.avatar} className={styles.avatar} />
          <View className={styles.memberInfo}>
            <Text className={styles.memberName}>{member.nickName}</Text>
            <Text className={styles.memberMeta}>ID：{member.id}</Text>
          </View>
          <View className={clsx(styles.roleTag, { [styles.isAdmin]: isAdmin })}>
            <Text>{ROLE_LABEL[member.role]}</Text>
          </View>
        </View>
      );
    },
    [memberList],
  );

  return (
    <Fragment>
      <ConfigListPanel
        title='人员管理'
        addButtonText='邀请成员'
        list={memberList}
        loading={loading}
        renderContent={renderContent}
        scrollViewProps={memberScrollViewProps}
        onAdd={handleOpenInvite}
        onEdit={handleOpenRole}
        deletable={false}
      />
      <FloatLayout visible={showInviteModal} onClose={handleCloseInvite} title='邀请成员'>
        <View className={styles.floatContent}>
          <View className={styles.inviteCodeBox}>
            <Text className={styles.inviteLabel}>邀请码</Text>
            <Text className={styles.inviteCode}>{inviteCode || '待生成'}</Text>
          </View>
          <WhiteSpace size='medium' />
          <Button
            size='large'
            width='100%'
            icon='add'
            loading={isInviteActionLoading}
            disabled={isInviteActionLoading}
            onClick={handleGenerateInvite}
          >
            生成邀请码
          </Button>
          {inviteCode ? (
            <Fragment>
              <WhiteSpace size='small' />
              <Button
                size='large'
                width='100%'
                type='plain'
                disabled={isInviteActionLoading}
                onClick={handleCopyInvite}
              >
                复制邀请码
              </Button>
            </Fragment>
          ) : null}
        </View>
      </FloatLayout>
      <FloatLayout visible={showRoleModal} onClose={handleCloseRole} title='成员权限'>
        <View className={styles.floatContent}>
          {currentMember ? (
            <View className={styles.roleMember}>
              <FallbackImage src={currentMember.avatar} className={styles.roleAvatar} />
              <View className={styles.memberInfo}>
                <Text className={styles.memberName}>{currentMember.nickName}</Text>
                <Text className={styles.memberMeta}>{ROLE_LABEL[currentMember.role]}</Text>
              </View>
            </View>
          ) : null}
          <WhiteSpace size='medium' />
          <View className={styles.roleOptions}>
            {ROLE_OPTIONS.map(item => {
              const isCurrent = currentMember?.role === item.value;
              return (
                <View key={item.value} className={styles.roleOption}>
                  <Button
                    size='large'
                    width='100%'
                    type={isCurrent ? 'primary' : 'plain'}
                    icon={isCurrent ? 'check' : undefined}
                    loading={isRoleActionLoading && !isCurrent}
                    disabled={isCurrent || isRoleActionLoading}
                    onClick={() => handleRoleChange(item.value)}
                  >
                    {item.label}
                  </Button>
                  <View className={styles.roleDesc}>
                    <Icon
                      name={isCurrent ? 'check' : 'uncheck'}
                      size={12}
                      color={
                        isCurrent ? COLOR_VARIABLES.COLOR_RED : COLOR_VARIABLES.TEXT_COLOR_SECONDARY
                      }
                    />
                    <Text>{item.desc}</Text>
                  </View>
                </View>
              );
            })}
          </View>
        </View>
      </FloatLayout>
    </Fragment>
  );
}

```

### toy-taro-client/src/ui/pages/register/constants.ts

```
export enum REGISTER_MODE {
  CREATE_GROUP = 'CREATE_GROUP',
  JOIN_GROUP = 'JOIN_GROUP',
}

export const REGISTER_MODE_OPTIONS = [
  {
    value: REGISTER_MODE.CREATE_GROUP,
    label: '创建分组',
  },
  {
    value: REGISTER_MODE.JOIN_GROUP,
    label: '加入分组',
  },
];

```

### toy-taro-client/src/ui/pages/register/index.config.ts

```
export default definePageConfig({
  navigationBarTitleText: '注册',
  disableScroll: true,
});

```

### toy-taro-client/src/ui/pages/register/index.module.scss

```
@import '@ui/styles/index.scss';

.registerWrapper {
  position: relative;
  height: 100%;
  overflow: hidden;
  @include soft-page-background;

  .visual {
    position: absolute;
    top: 0;
    right: 0;
    bottom: px(250);
    left: 0;
    overflow: hidden;
  }

  .heroImage {
    width: 100%;
    height: 100%;
  }

  .visualMask {
    position: absolute;
    top: 0;
    right: 0;
    bottom: 0;
    left: 0;
    background: linear-gradient(
        180deg,
        $--color-white-opacity-1 0%,
        rgba($--fill-body, 0.38) 58%,
        $--fill-body 100%
      ),
      linear-gradient(90deg, rgba($--color-white, 0.86) 0%, $--color-white-opacity-3 100%);
  }

  .brand {
    position: absolute;
    left: px(24);
    right: px(24);
    top: px(70);
    display: flex;
    flex-direction: column;
    align-items: flex-start;
  }

  .logo {
    width: px(58);
    height: px(58);
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: px(20);
    background: $--color-white-opacity-7;
    border: px(1) solid $--color-white-opacity-8;
    box-shadow: $--box-shadow-menu;
  }

  .logoImage {
    width: px(46);
    height: px(46);
  }

  .brandName {
    margin-top: px(18);
    color: $--text-color-base;
    font-size: px(30);
    line-height: 1.12;
    font-weight: 850;
  }

  .brandDesc {
    width: px(250);
    margin-top: px(10);
    color: $--text-color-secondary;
    font-size: px(14);
    line-height: 1.58;
  }

  .container {
    position: absolute;
    box-sizing: border-box;
    padding: px(20);
    @include glass-card(px(28), 0.78);
    width: calc(100vw - px(28));
    left: px(14);
    bottom: px(24);
    bottom: calc(constant(safe-area-inset-bottom) + #{px(24)});
    bottom: calc(env(safe-area-inset-bottom) + #{px(24)});
  }

  .panelHeader {
    margin-bottom: px(12);
  }

  .panelTitle {
    display: block;
    color: $--text-color-base;
    font-size: px(21);
    line-height: 1.2;
    font-weight: 850;
  }

  .panelDesc {
    display: block;
    min-height: px(38);
    margin-top: px(6);
    color: $--text-color-secondary;
    font-size: px(13);
    line-height: 1.45;
  }

  .modeTabs {
    min-height: px(120);
  }

  .modeTabsHeader {
    width: 100%;
  }

  .modeTabItem {
    flex: 1;
  }

  .modePanel {
    padding-top: px(10);
  }

  .actionButton {
    width: 100%;
  }

  .back {
    display: flex;
    align-items: center;
    margin-top: px(16);
    color: $--color-red;
    font-size: px(14);

    text {
      margin-left: px(4);
    }
  }
}

```

### toy-taro-client/src/ui/pages/register/index.tsx

```
import { useCallback, useMemo, useState } from 'react';
import { Image, Text, View } from '@tarojs/components';
import { COLOR_VARIABLES, PAGE_ID } from '@shared/utils/constants';
import { showToast } from '@shared/utils/operateFeedback';
import { navigateToPage } from '@shared/utils/router';
import {
  Button,
  FallbackImage,
  Icon,
  Input,
  SafeAreaBar,
  TabPanel,
  Tabs,
  WhiteSpace,
} from '@ui/components';
import { FormItem } from '@ui/container';
import banyaLogo from '@ui/images/brand/banya-logo.png';
import growthHero from '@ui/images/brand/growth-hero.jpg';
import { useUserAction } from '@ui/viewModel';
import { REGISTER_MODE, REGISTER_MODE_OPTIONS } from './constants';
import styles from './index.module.scss';

export default function () {
  const [mode, setMode] = useState(REGISTER_MODE.CREATE_GROUP);
  const [groupName, setGroupName] = useState('');
  const [inviteCode, setInviteCode] = useState('');
  const { isActionLoading, handleRegisterCreateGroup, handleRegisterJoinGroup } = useUserAction();
  const currentModeIndex = useMemo(() => {
    return REGISTER_MODE_OPTIONS.findIndex(item => item.value === mode);
  }, [mode]);

  const submitText = mode === REGISTER_MODE.CREATE_GROUP ? '创建成长空间' : '加入成长空间';
  const panelDesc =
    mode === REGISTER_MODE.CREATE_GROUP
      ? '为你的家庭或班级创建一个新的成长空间。'
      : '输入管理员分享的邀请码，加入已有成长空间。';

  const handleModeChange = useCallback((index: number) => {
    const nextMode = REGISTER_MODE_OPTIONS[index]?.value;
    if (nextMode) {
      setMode(nextMode);
    }
  }, []);

  const handleSubmit = useCallback(async () => {
    if (isActionLoading) {
      return;
    }

    if (mode === REGISTER_MODE.CREATE_GROUP) {
      const value = groupName.trim();
      if (!value) {
        await showToast({ title: '请输入分组名称' });
        return;
      }
      await handleRegisterCreateGroup({ groupName: value });
      return;
    }

    const value = inviteCode.trim();
    if (!value) {
      await showToast({ title: '请输入邀请码' });
      return;
    }
    await handleRegisterJoinGroup({ inviteCode: value });
  }, [
    groupName,
    handleRegisterCreateGroup,
    handleRegisterJoinGroup,
    inviteCode,
    isActionLoading,
    mode,
  ]);

  return (
    <View className={styles.registerWrapper}>
      <View className={styles.visual}>
        <FallbackImage className={styles.heroImage} src={growthHero} />
        <View className={styles.visualMask} />
        <View className={styles.brand}>
          <View className={styles.logo}>
            <Image className={styles.logoImage} src={banyaLogo} mode='aspectFit' />
          </View>
          <Text className={styles.brandName}>伴芽</Text>
          <Text className={styles.brandDesc}>先确定成长空间，再开始温柔地安排任务与奖励。</Text>
        </View>
      </View>
      <View className={styles.container}>
        <View className={styles.panelHeader}>
          <Text className={styles.panelTitle}>注册成长空间</Text>
          <Text className={styles.panelDesc}>{panelDesc}</Text>
        </View>
        <Tabs
          className={styles.modeTabs}
          classes={{
            headerContainer: styles.modeTabsHeader,
            headerItem: styles.modeTabItem,
          }}
          variant='contained'
          current={currentModeIndex}
          lazy={false}
          onChange={handleModeChange}
        >
          <TabPanel label={REGISTER_MODE_OPTIONS[0].label} className={styles.modePanel}>
            <FormItem title='分组名称' required>
              <Input
                placeholder='例如：小芽的家庭'
                value={groupName}
                onInput={e => setGroupName(e.detail.value)}
              />
            </FormItem>
          </TabPanel>
          <TabPanel label={REGISTER_MODE_OPTIONS[1].label} className={styles.modePanel}>
            <FormItem title='邀请码' required>
              <Input
                placeholder='请输入管理员分享的邀请码'
                value={inviteCode}
                onInput={e => setInviteCode(e.detail.value.toUpperCase())}
              />
            </FormItem>
          </TabPanel>
        </Tabs>
        <WhiteSpace size='medium' />
        <Button
          className={styles.actionButton}
          size='large'
          loading={isActionLoading}
          disabled={isActionLoading}
          onClick={handleSubmit}
        >
          {submitText}
        </Button>
        <View
          className={styles.back}
          onClick={() => navigateToPage({ pageName: PAGE_ID.LOGIN, isRedirect: true })}
        >
          <Icon name='arrow-left' color={COLOR_VARIABLES.COLOR_RED} /> 返回登录
        </View>
      </View>
      <SafeAreaBar inset='bottom' />
    </View>
  );
}

```

### toy-taro-client/src/ui/viewModel/user/useMemberManage.spec.ts

```
import { showToast } from '@shared/utils/operateFeedback';
import { ROLE, sdk, STORE_NAME } from '@core';
import { useStoreList, useStoreLoadingStatus } from '../base';
import { useMemberManage } from './useMemberManage';

const mockSetInviteCode = jest.fn();
const mockSetRoleActionLoading = jest.fn();
const mockSetInviteActionLoading = jest.fn();
const mockSetRefreshing = jest.fn();

let stateIndex = 0;

jest.mock('react', () => ({
  useCallback: (callback: unknown) => callback,
  useMemo: (factory: () => unknown) => factory(),
  useState: jest.fn((initial: unknown) => {
    const setters = [
      mockSetInviteCode,
      mockSetRoleActionLoading,
      mockSetInviteActionLoading,
      mockSetRefreshing,
    ];
    const setter = setters[stateIndex] ?? jest.fn();
    stateIndex += 1;
    return [typeof initial === 'function' ? (initial as () => unknown)() : initial, setter];
  }),
}));

jest.mock(
  '@shared/utils/operateFeedback',
  () => ({
    showToast: jest.fn(),
  }),
  { virtual: true },
);

jest.mock(
  '@core',
  () => ({
    STORE_NAME: {
      CONTACT: 'CONTACT',
    },
    ROLE: {
      ADMIN: 'ADMIN',
      USER: 'USER',
    },
    sdk: {
      modules: {
        user: {
          getMemberList: jest.fn(),
          updateMemberRole: jest.fn(),
          createGroupInvite: jest.fn(),
        },
      },
    },
  }),
  { virtual: true },
);

jest.mock('../base', () => ({
  useStoreList: jest.fn(),
  useStoreLoadingStatus: jest.fn(),
}));

describe('useMemberManage', () => {
  const members = [
    { id: '1', name: '管理员', role: ROLE.ADMIN },
    { id: '2', name: '成员', role: ROLE.USER },
  ];

  beforeEach(() => {
    jest.clearAllMocks();
    stateIndex = 0;
    jest.mocked(useStoreList).mockReturnValue(members as never);
    jest.mocked(useStoreLoadingStatus).mockReturnValue(false);
    jest.mocked(showToast).mockResolvedValue(undefined);
    jest.mocked(sdk.modules.user.getMemberList).mockResolvedValue(members as never);
    jest.mocked(sdk.modules.user.updateMemberRole).mockResolvedValue(undefined);
    jest.mocked(sdk.modules.user.createGroupInvite).mockResolvedValue({
      inviteCode: 'BANYA-8F6Q',
    });
  });

  it('exposes contact-store members and refreshes the member list', async () => {
    const { memberList, loading, refreshMembers } = useMemberManage();

    expect(useStoreList).toHaveBeenCalledWith(STORE_NAME.CONTACT);
    expect(useStoreLoadingStatus).toHaveBeenCalledWith(STORE_NAME.CONTACT);
    expect(memberList).toBe(members);
    expect(loading).toBe(false);

    await refreshMembers();

    expect(mockSetRefreshing).toHaveBeenNthCalledWith(1, true);
    expect(sdk.modules.user.getMemberList).toHaveBeenCalled();
    expect(mockSetRefreshing).toHaveBeenLastCalledWith(false);
  });

  it('updates a member role with loading and success feedback', async () => {
    const { handleUpdateMemberRole } = useMemberManage();

    await handleUpdateMemberRole({ userId: '2', role: ROLE.ADMIN });

    expect(mockSetRoleActionLoading).toHaveBeenNthCalledWith(1, true);
    expect(sdk.modules.user.updateMemberRole).toHaveBeenCalledWith({
      userId: '2',
      role: ROLE.ADMIN,
    });
    expect(showToast).toHaveBeenCalledWith({ title: '角色已更新' });
    expect(mockSetRoleActionLoading).toHaveBeenLastCalledWith(false);
  });

  it('stores and returns the generated invite code', async () => {
    const { handleCreateInvite } = useMemberManage();

    const inviteCode = await handleCreateInvite();

    expect(mockSetInviteActionLoading).toHaveBeenNthCalledWith(1, true);
    expect(sdk.modules.user.createGroupInvite).toHaveBeenCalled();
    expect(mockSetInviteCode).toHaveBeenCalledWith('BANYA-8F6Q');
    expect(showToast).toHaveBeenCalledWith({ title: '邀请码已生成' });
    expect(mockSetInviteActionLoading).toHaveBeenLastCalledWith(false);
    expect(inviteCode).toBe('BANYA-8F6Q');
  });
});

```

### toy-taro-client/src/ui/viewModel/user/useMemberManage.ts

```
import { useCallback, useState } from 'react';
import { showToast } from '@shared/utils/operateFeedback';
import { ROLE, sdk, STORE_NAME } from '@core';
import type { ContactModel } from '@core/model';
import { useStoreList, useStoreLoadingStatus } from '../base';

interface UpdateMemberRoleParams {
  userId: string;
  role: ROLE;
}

function getErrorMessage(error: unknown, fallback: string) {
  return error instanceof Error ? error.message : fallback;
}

export function useMemberManage() {
  const memberList = useStoreList(STORE_NAME.CONTACT) as ContactModel[];
  const loading = useStoreLoadingStatus(STORE_NAME.CONTACT);
  const [inviteCode, setInviteCode] = useState('');
  const [isRoleActionLoading, setIsRoleActionLoading] = useState(false);
  const [isInviteActionLoading, setIsInviteActionLoading] = useState(false);
  const [isRefreshing, setIsRefreshing] = useState(false);

  const refreshMembers = useCallback(async () => {
    try {
      setIsRefreshing(true);
      await sdk.modules.user.getMemberList();
    } catch (error) {
      await showToast({ title: getErrorMessage(error, '成员列表刷新失败') });
    } finally {
      setIsRefreshing(false);
    }
  }, []);

  const handleUpdateMemberRole = useCallback(async (params: UpdateMemberRoleParams) => {
    try {
      setIsRoleActionLoading(true);
      await sdk.modules.user.updateMemberRole(params);
      await showToast({ title: '角色已更新' });
      return true;
    } catch (error) {
      await showToast({ title: getErrorMessage(error, '角色更新失败') });
      return false;
    } finally {
      setIsRoleActionLoading(false);
    }
  }, []);

  const handleCreateInvite = useCallback(async () => {
    try {
      setIsInviteActionLoading(true);
      const result = await sdk.modules.user.createGroupInvite();
      setInviteCode(result.inviteCode);
      await showToast({ title: '邀请码已生成' });
      return result.inviteCode;
    } catch (error) {
      await showToast({ title: getErrorMessage(error, '邀请码生成失败') });
      return '';
    } finally {
      setIsInviteActionLoading(false);
    }
  }, []);

  const handleClearInviteCode = useCallback(() => {
    setInviteCode('');
  }, []);

  return {
    memberList,
    loading,
    inviteCode,
    isRoleActionLoading,
    isInviteActionLoading,
    isRefreshing,
    refreshMembers,
    handleUpdateMemberRole,
    handleCreateInvite,
    handleClearInviteCode,
  };
}

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
