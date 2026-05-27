# 管理员邀请流程代码分析

## 概述

本文档从代码实现角度分析 Novu 中管理员邀请成员加入组织的完整流程，包括三个核心环节：**邀请发起**、**角色绑定**、**能力校验**。

---

## 一、角色体系

### 1.1 角色枚举定义

角色枚举在 `packages/shared/src/entities/organization/member.enum.ts` 中定义：

```typescript
export enum MemberRoleEnum {
  ADMIN = 'org:admin',       // 管理员
  OWNER = 'org:owner',       // 所有者
  AUTHOR = 'org:author',     // 创作者
  VIEWER = 'org:viewer',     // 观察者
  OSS_MEMBER = 'member',     // OSS 成员（已废弃）
  OSS_ADMIN = 'admin',       // OSS 管理员（已废弃）
}
```

其中 `OSS_MEMBER` 和 `OSS_ADMIN` 是社区版（非 EE）使用的角色，已标记为 `@deprecated`。

### 1.2 角色-权限映射

权限定义在 `packages/shared/src/types/auth.ts` 中的 `ROLE_PERMISSIONS`：

| 权限 | OWNER | ADMIN | AUTHOR | VIEWER | OSS_MEMBER | OSS_ADMIN |
|------|:-----:|:-----:|:------:|:------:|:----------:|:---------:|
| WORKFLOW_READ | ✓ | ✓ | ✓ | ✓ | — | — |
| WORKFLOW_WRITE | ✓ | ✓ | ✓ | — | — | — |
| AGENT_READ | ✓ | ✓ | ✓ | ✓ | — | — |
| AGENT_WRITE | ✓ | ✓ | ✓ | — | — | — |
| WEBHOOK_READ | ✓ | ✓ | — | — | — | — |
| WEBHOOK_WRITE | ✓ | ✓ | — | — | — | — |
| ENVIRONMENT_WRITE | ✓ | ✓ | — | — | — | — |
| API_KEY_READ | ✓ | ✓ | — | — | — | — |
| API_KEY_WRITE | ✓ | ✓ | — | — | — | — |
| EVENT_WRITE | ✓ | ✓ | ✓ | — | — | — |
| INTEGRATION_READ | ✓ | ✓ | ✓ | ✓ | — | — |
| INTEGRATION_WRITE | ✓ | ✓ | ✓ | — | — | — |
| MESSAGE_READ | ✓ | ✓ | ✓ | ✓ | — | — |
| MESSAGE_WRITE | ✓ | ✓ | — | — | — | — |
| PARTNER_INTEGRATION_READ | ✓ | ✓ | — | — | — | — |
| PARTNER_INTEGRATION_WRITE | ✓ | ✓ | — | — | — | — |
| SUBSCRIBER_READ | ✓ | ✓ | ✓ | ✓ | — | — |
| SUBSCRIBER_WRITE | ✓ | ✓ | ✓ | — | — | — |
| TOPIC_READ | ✓ | ✓ | ✓ | ✓ | — | — |
| TOPIC_WRITE | ✓ | ✓ | ✓ | — | — | — |
| BILLING_WRITE | ✓ | — | — | — | — | — |
| ORG_METADATA_WRITE | ✓ | ✓ | — | — | — | — |
| NOTIFICATION_READ | ✓ | ✓ | ✓ | ✓ | — | — |
| BRIDGE_WRITE | ✓ | ✓ | ✓ | — | — | — |
| ORG_SETTINGS_WRITE | ✓ | ✓ | — | — | — | — |
| ORG_SETTINGS_READ | ✓ | ✓ | — | — | — | — |

**关键差异：**
- OWNER 比 ADMIN 多一个 `BILLING_WRITE` 权限（账单写入）
- OSS_MEMBER 和 OSS_ADMIN 的权限数组为空 `[]`，意味着在 EE 模式下需要 EE 模块的权限守卫
- API Key 认证时（`CommunityAuthService.getUserByApiKey`），直接赋予 `ALL_PERMISSIONS`

---

## 二、邀请发起

### 2.1 API 入口

`apps/api/src/app/invites/invites.controller.ts` 定义了以下端点：

| 方法 | 路径 | 说明 |
|------|------|------|
| `POST /invites` | 邀请单个成员 | 仅认证，角色硬编码为 `OSS_ADMIN` |
| `POST /invites/bulk` | 批量邀请 | 仅认证，所有邀请人角色硬编码为 `OSS_ADMIN` |
| `POST /invites/resend` | 重发邀请 | 仅认证 |
| `GET /invites/:inviteToken` | 获取邀请信息 | 无需认证 |
| `POST /invites/:inviteToken/accept` | 接受邀请 | 需认证 |

### 2.2 单个邀请核心流程

**文件：** `apps/api/src/app/invites/usecases/invite-member/invite-member.usecase.ts`

```typescript
async execute(command: InviteMemberCommand) {
  // 1. 验证组织存在
  const organization = await this.organizationRepository.findById(command.organizationId);

  // 2. 检查邮箱是否已被邀请
  const foundInvitee = await this.memberRepository.findInviteeByEmail(organization._id, command.email);
  if (foundInvitee) throw new BadRequestException('Already invited');

  // 3. 生成邀请 token
  const token = createGuid();

  // 4. 发送邀请邮件（通过 Novu 自身的工作流）
  await novu.trigger({
    workflowId: 'invite-to-organization-wBnO8NpDn',
    to: [{ subscriberId: command.email, email: command.email }],
    payload: {
      acceptInviteUrl: `${DASHBOARD_URL}/auth/invitation/${token}`,
      // ...
    },
  });

  // 5. 创建成员记录（INVITED 状态）
  await this.memberRepository.addMember(organization._id, {
    roles: [command.role as MemberRoleEnum],  // 角色在此时绑定
    memberStatus: MemberStatusEnum.INVITED,
    invite: { token, _inviterId: command.userId, email: command.email, invitationDate: new Date() },
  });
}
```

**关键数据结构：** `IAddMemberData`（见 `libs/dal`）

```typescript
{
  roles: MemberRoleEnum[];        // 角色数组，如 ['org:admin']
  memberStatus: MemberStatusEnum; // 'invited' | 'active' | 'new'
  invite?: {                      // 仅邀请时有值
    token: string;
    _inviterId: string;
    email: string;
    invitationDate: Date;
    answerDate?: Date;
  };
}
```

### 2.3 角色硬编码问题

在 `invites.controller.ts` 中：

```typescript
// POST /invites - 角色硬编码为 OSS_ADMIN
const command = InviteMemberCommand.create({
  role: MemberRoleEnum.OSS_ADMIN,  // <-- 硬编码！
  // ...
});

// POST /invites/bulk - 同样硬编码
role: MemberRoleEnum.OSS_ADMIN,   // <-- 硬编码！
```

**注意：** 这意味着在社区版中，所有被邀请的成员角色都是 `OSS_ADMIN`。EE 版可能有不同的角色选择逻辑（由 `@novu/ee-auth` 模块控制）。

### 2.4 批量邀请

**文件：** `apps/api/src/app/invites/usecases/bulk-invite/bulk-invite.usecase.ts`

批量邀请遍历每个 invitee，逐个调用 `InviteMember` usecase，单个失败不影响整体：

```typescript
for (const invitee of command.invitees) {
  try {
    await this.inviteMemberUsecase.execute(InviteMemberCommand.create({
      email: invitee.email,
      role: MemberRoleEnum.OSS_ADMIN,  // 硬编码
      // ...
    }));
    invites.push({ success: true, email: invitee.email });
  } catch (e) {
    if (e.message.includes('Already invited')) {
      invites.push({ failReason: 'Already invited', success: false, email: invitee.email });
    } else {
      invites.push({ success: false, email: invitee.email });
    }
  }
}
```

### 2.5 重发邀请

**文件：** `apps/api/src/app/invites/usecases/resend-invite/resend-invite.usecase.ts`

重发时生成新 token，更新原有成员记录的 `invite` 字段：

```typescript
await this.memberRepository.update(foundInvitee, {
  memberStatus: MemberStatusEnum.INVITED,
  invite: { token, _inviterId: command.userId, invitationDate: new Date() },
});
```

---

## 三、接受邀请与角色绑定

### 3.1 接受邀请核心流程

**文件：** `apps/api/src/app/invites/usecases/accept-invite/accept-invite.usecase.ts`

```typescript
async execute(command: AcceptInviteCommand): Promise<string> {
  // 1. 通过 token 查找成员记录
  const member = await this.memberRepository.findByInviteToken(command.token);

  // 2. 验证邀请状态
  if (member.memberStatus !== MemberStatusEnum.INVITED) {
    throw new BadRequestException('Token expired');
  }

  // 3. 将成员从 INVITED 转为 ACTIVE，绑定 userId
  await this.memberRepository.convertInvitedUserToMember(
    this.organizationId,
    command.token,
    {
      memberStatus: MemberStatusEnum.ACTIVE,
      _userId: command.userId,
      answerDate: new Date(),
    }
  );

  // 4. 生成用户 token 返回
  return this.authService.generateUserToken(user);
}
```

**数据库操作：** `community.member.repository.ts:115`

```typescript
async convertInvitedUserToMember(organizationId, token, data) {
  await this.update(
    { _organizationId: organizationId, 'invite.token': token },
    {
      memberStatus: data.memberStatus,       // 'active'
      _userId: data._userId,                 // 绑定真实用户
      'invite.answerDate': data.answerDate,  // 记录接受时间
    }
  );
}
```

**关键点：** 角色在 `convertInvitedUserToMember` 调用中**不改变**。角色是在邀请发起时就已写入成员记录的 `roles` 字段，接受邀请时仅更新状态和绑定 userId。

### 3.2 Token 生成时的角色注入

**文件：** `apps/api/src/app/auth/services/community.auth.service.ts`

`getSignedToken` 方法将成员角色注入 JWT：

```typescript
public async getSignedToken(user, organizationId?, member?, environmentId?) {
  const roles: MemberRoleEnum[] = [];
  if (member && member.roles) {
    roles.push(...member.roles);  // 从成员记录中取出角色
  }

  return this.jwtService.sign({
    _id: user._id,
    organizationId: organizationId || null,
    roles,  // <-- 角色写入 JWT
  }, { expiresIn: '30 days' });
}
```

### 3.3 切换组织时的角色刷新

**文件：** `apps/api/src/app/auth/usecases/switch-organization/switch-organization.usecase.ts`

```typescript
async execute(command: SwitchOrganizationCommand) {
  // 1. 验证用户是否属于目标组织
  const isAuthenticated = await this.authService.isAuthenticatedForOrganization(
    command.userId, command.newOrganizationId
  );

  // 2. 获取成员记录（含角色）
  const member = await this.memberRepository.findMemberByUserId(
    command.newOrganizationId, command.userId
  );

  // 3. 重新生成带角色的 token
  const token = await this.authService.getSignedToken(user, command.newOrganizationId, member);
  return token;
}
```

---

## 四、能力校验

### 4.1 权限装饰器

**文件：** `libs/application-generic/src/decorators/permissions.decorator.ts`

```typescript
export const RequirePermissions = (...permissions: PermissionsEnum[]) =>
  SetMetadata('permissions', permissions);

export const SkipPermissionsCheck = () =>
  SetMetadata('no_permissions_required', true);
```

### 4.2 权限守卫的运行机制

权限校验分为两条路径：

#### 社区版（Community）

- `CommunityUserAuthGuard`（`apps/api/src/app/auth/framework/community.user.auth.guard.ts`）仅负责认证（JWT / API Key）
- **不包含权限校验逻辑** —— 权限校验在 EE 模块中实现
- `RequirePermissions` 装饰器虽然被大量使用，但在社区版中**没有对应守卫**来读取元数据并执行校验

#### EE 版（Enterprise）

- `auth.decorator.ts` 中的 `RequireAuthentication` 会根据 `isEEAuthEnabled()` 动态选择：
  - EE 模式：使用 `@novu/ee-auth` 中的 `RequireAuthentication`（含权限校验）
  - 社区模式：使用 `CommunityUserAuthGuard`（仅认证）

```typescript
export function RequireAuthentication() {
  if (isEEAuthEnabled()) {
    const { RequireAuthentication: EERequireAuthentication } = require('@novu/ee-auth');
    return EERequireAuthentication();
  }
  return applyDecorators(UseGuards(CommunityUserAuthGuard), ApiBearerAuth(...));
}
```

### 4.3 API Key 认证的特殊处理

**文件：** `apps/api/src/app/auth/services/community.auth.service.ts:176`

```typescript
public async getUserByApiKey(apiKey: string): Promise<UserSessionData> {
  // ...
  return {
    roles: [MemberRoleEnum.OSS_ADMIN],       // 固定角色
    permissions: ALL_PERMISSIONS,            // 所有权限
    // ...
  };
}
```

通过 API Key 认证的用户自动获得**全部权限**，不受角色限制。

### 4.4 控制器中的权限使用示例

`ee.organization.controller.ts` 中展示了典型的权限使用模式：

```typescript
@Patch('/settings')
@RequirePermissions(PermissionsEnum.ORG_SETTINGS_WRITE)
async updateSettings(@UserSession() user: UserSessionData, ...) { ... }

@Get('/settings')
@RequirePermissions(PermissionsEnum.ORG_SETTINGS_READ)
async getSettings(@UserSession() user: UserSessionData) { ... }
```

只有 `OWNER` 和 `ADMIN` 角色拥有这些权限，`AUTHOR`/`VIEWER` 无法访问。

### 4.5 成员列表中的角色过滤

**文件：** `apps/api/src/app/organization/usecases/membership/get-members/get-members.usecase.ts`

```typescript
async execute(command: GetMembersCommand) {
  return (await this.membersRepository.getOrganizationMembers(command.organizationId))
    .map((member) => {
      if (!command.user.roles.includes(MemberRoleEnum.OSS_ADMIN)) {
        // 非管理员看不到已邀请的成员
        if (member.memberStatus === MemberStatusEnum.INVITED) return null;
        // 非管理员看不到成员邮箱
        if (member.user) member.user.email = '';
        if (member.invite) member.invite.email = '';
      }
      return member;
    })
    .filter(Boolean);
}
```

**这是一处基于角色的前端数据过滤逻辑：** 非 `OSS_ADMIN` 角色看不到已邀请成员和邮箱信息。

### 4.6 更改成员角色

**文件：** `apps/api/src/app/organization/usecases/membership/change-member-role/change-member-role.usecase.ts`

```typescript
async execute(command: ChangeMemberRoleCommand) {
  // 仅允许 OSS_MEMBER 和 OSS_ADMIN 角色
  if (![MemberRoleEnum.OSS_MEMBER, MemberRoleEnum.OSS_ADMIN].includes(command.role)) {
    throw new BadRequestException('Not supported role type');
  }

  // 仅允许改为 OSS_ADMIN
  if (command.role !== MemberRoleEnum.OSS_ADMIN) {
    throw new BadRequestException(`The change of role to an ${command.role} type is not supported`);
  }

  await this.memberRepository.updateMemberRoles(organization._id, command.memberId, [command.role]);
}
```

控制器端额外校验：

```typescript
// organization.controller.ts:143
if (body.role !== MemberRoleEnum.OSS_ADMIN) {
  throw new Error('Only admin role can be assigned to a member');
}
```

### 4.7 移除成员

**文件：** `apps/api/src/app/organization/usecases/membership/remove-member/remove-member.usecase.ts`

```typescript
async execute(command: RemoveMemberCommand) {
  // 不能移除自己
  if (memberToRemove._userId === command.userId) {
    throw new BadRequestException('Cannot remove self from members');
  }

  // 移除后若该成员关联了 API Key，需要转移给组织所有者
  if (isMemberAssociatedWithEnvironment) {
    const owner = await this.memberRepository.getOrganizationOwnerAccount(command.organizationId);
    await this.environmentRepository.updateApiKeyUserId(
      command.organizationId, memberToRemove._userId, owner._userId
    );
  }
}
```

### 4.8 前端权限校验

**文件：** `apps/dashboard/src/hooks/use-has-permission.tsx`

```typescript
export function useHasPermission(): CheckAuthorizationWithCustomPermissions {
  const isRbacFeatureEnabled = /* 基于 RBAC 功能开关 + 订阅等级 */;

  if (!isRbacFeatureEnabled) {
    return () => true;  // 未启用 RBAC 时，所有权限通过
  }

  return has as CheckAuthorizationWithCustomPermissions;  // 使用 Clerk 的 has 检查
}
```

前端权限校验依赖两个条件：
1. `IS_RBAC_ENABLED` 功能开关
2. 订阅等级支持 `ACCOUNT_ROLE_BASED_ACCESS_CONTROL_BOOLEAN` 功能

---

## 五、完整流程图

```
┌─────────────┐     ┌──────────────┐     ┌───────────────┐
│  管理员发起   │────▶│ 创建 Member  │────▶│ 发送邀请邮件    │
│  POST /invites│     │ roles + token│     │ (Novu 工作流)  │
└─────────────┘     └──────────────┘     └───────────────┘
                                               │
                                               ▼
┌─────────────┐     ┌──────────────┐     ┌───────────────┐
│  生成 JWT    │◀────│ 角色注入 JWT  │◀────│ 接受邀请        │
│  (含 roles)  │     │              │     │ INVITED→ACTIVE │
└─────────────┘     └──────────────┘     └───────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────┐
│               每次 API 请求时校验                      │
│                                                      │
│  社区版: CommunityUserAuthGuard 仅校验 JWT            │
│  EE 版:   EE Auth Guard 校验 JWT + RequirePermissions │
│  API Key: 自动获得 ALL_PERMISSIONS                    │
└──────────────────────────────────────────────────────┘
```

---

## 六、关键代码位置索引

| 功能 | 文件路径 | 关键行号 |
|------|---------|---------|
| 角色枚举 | `packages/shared/src/entities/organization/member.enum.ts` | L1-L14 |
| 角色-权限映射 | `packages/shared/src/types/auth.ts` | L74-L157 |
| 权限枚举 | `packages/shared/src/types/auth.ts` | L43-L70 |
| 邀请单个成员 | `apps/api/src/app/invites/usecases/invite-member/invite-member.usecase.ts` | L19-L70 |
| 批量邀请 | `apps/api/src/app/invites/usecases/bulk-invite/bulk-invite.usecase.ts` | L26-L64 |
| 接受邀请 | `apps/api/src/app/invites/usecases/accept-invite/accept-invite.usecase.ts` | L25-L52 |
| 重发邀请 | `apps/api/src/app/invites/usecases/resend-invite/resend-invite.usecase.ts` | L18-L65 |
| 获取邀请信息 | `apps/api/src/app/invites/usecases/get-invite/get-invite.usecase.ts` | L17-L50 |
| 邀请控制器 | `apps/api/src/app/invites/invites.controller.ts` | L42-L130 |
| 转换为成员(DAL) | `libs/dal/src/repositories/member/community.member.repository.ts` | L115-L130 |
| Token 生成(含角色) | `apps/api/src/app/auth/services/community.auth.service.ts` | L236-L267 |
| 生成用户 Token | `apps/api/src/app/auth/services/community.auth.service.ts` | L219-L234 |
| 切换组织 | `apps/api/src/app/auth/usecases/switch-organization/switch-organization.usecase.ts` | L14-L32 |
| 权限装饰器 | `libs/application-generic/src/decorators/permissions.decorator.ts` | L1-L12 |
| 社区认证守卫 | `apps/api/src/app/auth/framework/community.user.auth.guard.ts` | L1-L48 |
| Auth 装饰器(路由) | `apps/api/src/app/auth/framework/auth.decorator.ts` | L1-L15 |
| 更改成员角色 | `apps/api/src/app/organization/usecases/membership/change-member-role/change-member-role.usecase.ts` | L14-L34 |
| 移除成员 | `apps/api/src/app/organization/usecases/membership/remove-member/remove-member.usecase.ts` | L14-L41 |
| 获取成员列表(含过滤) | `apps/api/src/app/organization/usecases/membership/get-members/get-members.usecase.ts` | L12-L24 |
| 组织控制器(成员端点) | `apps/api/src/app/organization/organization.controller.ts` | L114-L155 |
| 前端权限 Hook | `apps/dashboard/src/hooks/use-has-permission.tsx` | L24-L45 |
| API Key 全权限 | `apps/api/src/app/auth/services/community.auth.service.ts` | L176-L197 |
| EE 认证开关 | `packages/shared/src/utils/env.ts` | L58-L69 |
| Auth 装饰器动态路由 | `apps/api/src/app/auth/framework/auth.decorator.ts` | L1-L15 |
| 社区版 JWT Strategy | `apps/api/src/app/auth/services/passport/jwt.strategy.ts` | L24-L54 |
| 权限守卫测试 | `apps/api/src/app/auth/e2e/permissions.guard.e2e.ts` | L1-L142 |
| 组织控制器动态加载 | `apps/api/src/app/organization/organization.module.ts` | L34-L40 |
| Token 生成函数 | `libs/application-generic/src/services/helper-service/helper.service.ts` | L3-L5 |
| 邮箱规范化函数 | `packages/shared/src/utils/normalizeEmail.ts` | L28-L45 |
| DAL 按邮箱查找被邀请人 | `libs/dal/src/repositories/member/community.member.repository.ts` | L145-L154 |
| DAL 按 Token 查找被邀请人 | `libs/dal/src/repositories/member/community.member.repository.ts` | L137-L143 |

---

## 七、社区版与 EE/Better-Auth 实现差异

### 7.1 认证与权限架构差异

| 维度 | 社区版 (Community) | EE 版 (Clerk/Better-Auth) |
|------|-------------------|------------------------|
| **身份提供商** | 本地 JWT + GitHub OAuth | Clerk / Better-Auth |
| **JWT 签发方** | 本地 `JwtService` | Clerk / Better-Auth 服务 |
| **JWT claims** | `_id`, `organizationId`, `roles` | `_id`, `org_id`, `org_role`, `org_permissions` |
| **权限来源** | 无（社区版不校验） | `org_permissions` claim |
| **权限守卫** | 无（仅认证） | EE 权限守卫（按订阅等级启用） |
| **角色映射** | `ROLE_PERMISSIONS`（定义但未使用） | 由 Clerk/Better-Auth 外部映射 |
| **组织控制器** | `OrganizationController` | `EEOrganizationController` |

### 7.2 切换开关逻辑

**文件：** `packages/shared/src/utils/env.ts`

```typescript
// EE 启用条件
export const isEEAuthEnabled = () =>
  process.env.NOVU_ENTERPRISE === 'true' || process.env.CI_EE_TEST === 'true';

// Auth Provider 选择
export const getEEAuthProvider = (): EEAuthProvider => {
  const provider = process.env.EE_AUTH_PROVIDER as EEAuthProvider | undefined;
  return provider || 'clerk';
};

export const isClerkEnabled = () => isEEAuthEnabled() && getEEAuthProvider() === 'clerk';
export const isBetterAuthEnabled = () => isEEAuthEnabled() && getEEAuthProvider() === 'better-auth';
```

**文件：** `apps/api/src/app/auth/framework/auth.decorator.ts`

```typescript
export function RequireAuthentication() {
  if (isEEAuthEnabled()) {
    // 加载 EE 版认证守卫（含权限校验）
    const { RequireAuthentication: EERequireAuthentication } = require('@novu/ee-auth');
    return EERequireAuthentication();
  }
  // 社区版仅认证
  return applyDecorators(UseGuards(CommunityUserAuthGuard), ApiBearerAuth(...));
}
```

### 7.3 控制器差异

**文件：** `apps/api/src/app/organization/organization.module.ts`

```typescript
function getControllers() {
  if (isClerkEnabled() || isBetterAuthEnabled()) {
    return [EEOrganizationController];  // EE 版控制器
  }
  return [OrganizationController];       // 社区版控制器
}
```

**关键差异：**
- `EEOrganizationController` 大量使用 `@RequirePermissions()` 装饰器
- `OrganizationController` 不使用 `@RequirePermissions()`，仅依赖 `@RequireAuthentication()`
- EE 版支持更多角色（OWNER/ADMIN/AUTHOR/VIEWER），社区版仅 OSS_ADMIN

---

## 八、Roles 在 JWT 和权限守卫中的实际生效路径

### 8.1 社区版：roles 在 JWT 中存活但无消费者

**JWT 构建阶段：** `apps/api/src/app/auth/services/community.auth.service.ts:236`

```typescript
public async getSignedToken(user, organizationId?, member?, environmentId?) {
  const roles: MemberRoleEnum[] = [];
  if (member && member.roles) {
    roles.push(...member.roles);  // 从成员记录中取出角色
  }

  return this.jwtService.sign({
    _id: user._id,
    organizationId: organizationId || null,
    environmentId: environmentId || null,
    roles,                        // 写入 JWT payload
  }, { expiresIn: '30 days' });
}
```

**JWT 解码阶段：** passport-jwt 自动将 JWT payload 映射为 `UserSessionData` 对象。由于 payload 中的 key 与类型字段名匹配，`session.roles` 自动获得值，`session.permissions` 为 `undefined`（JWT 中不存在此字段）。

**JwtStrategy.validate()：** `apps/api/src/app/auth/services/passport/jwt.strategy.ts:24`

```typescript
async validate(req: http.IncomingMessage, session: UserSessionData) {
  session.scheme = ApiAuthSchemeEnum.BEARER;
  // 只验证用户存在 + 是否属于组织，不处理 roles 或 permissions
  const user = await this.authService.validateUser(session);
  session.environmentId = this.resolveEnvironmentId(req, session);
  return session;  // session.roles 有值，session.permissions 为 undefined
}
```

**认证守卫：** `CommunityUserAuthGuard` 仅调用 passport 验证身份，不读取 `session.roles` 或 `session.permissions`。

**权限装饰器：** `@RequirePermissions()` 仅通过 `SetMetadata` 设置元数据，社区版没有对应的 Guard 来读取并校验这些元数据。

**实际生效链路：**

```
Member.roles → getSignedToken() → JWT payload.roles
    ↓
passport-jwt 自动映射 → UserSessionData.roles（有值）
    ↓
JwtStrategy.validate() → 透传 session，不做权限处理
    ↓
CommunityUserAuthGuard → 仅认证，不校验权限
    ↓
@RequirePermissions() → 设置元数据，无守卫读取
    ↓
唯一使用 roles 的地方：get-members.usecase.ts 中
    command.user.roles.includes(MemberRoleEnum.OSS_ADMIN)
```

**结论：** 社区版中，`session.roles` 从 JWT 到会话数据的链路是完整的（roles 确实能到达 UserSessionData），但缺少将 roles 映射为 permissions 并执行校验的守卫。唯一消费 roles 的代码是 `get-members.usecase.ts:15`，用于过滤非管理员可见的数据。

### 8.2 EE 版：完整的 roles → permissions → 校验链路

```
Clerk/Better-Auth 签发 JWT
    包含 claims: org_role, org_permissions, org_id
        ↓
EE JWT Strategy 解析 claims
    session.roles = org_role
    session.permissions = org_permissions
        ↓
EE 权限守卫（@novu/ee-auth）
    读取 @RequirePermissions() 元数据
    校验 session.permissions 是否包含所需权限
        ↓
按订阅等级分级：
  - Business tier: 严格校验，不足返回 403
  - Free/Pro tier: 跳过校验，返回 200（向后兼容）
  - API Key: 跳过校验，返回 200
```

**EE 权限守卫测试验证：** `apps/api/src/app/auth/e2e/permissions.guard.e2e.ts`

```typescript
// Business tier + 权限不足 → 403
expect(response.statusCode).to.equal(403);
expect(response.body.message).to.include('Insufficient permissions');

// Free/Pro tier + 权限不足 → 200（跳过校验）
expect(response.statusCode).to.equal(200);

// API Key + 任何权限 → 200（跳过校验）
expect(response.statusCode).to.equal(200);
```

### 8.3 UserSessionData 中 roles/permissions 的来源差异

```typescript
export type UserSessionData = {
  _id: string;
  organizationId: string;
  roles: MemberRoleEnum[];        // 来源：
                                  //   社区版: JWT payload.roles（getSignedToken 写入）
                                  //   EE 版:   JWT claim org_role
  permissions: PermissionsEnum[]; // 来源：
                                  //   社区版: undefined（JWT 中无此字段）
                                  //   EE 版:   JWT claim org_permissions
  scheme: ApiAuthSchemeEnum;
  environmentId: string;
};
```

---

## 九、邀请接受时的邮箱校验缺失

### 9.1 问题分析

**文件：** `apps/api/src/app/invites/usecases/accept-invite/accept-invite.usecase.ts`

```typescript
async execute(command: AcceptInviteCommand): Promise<string> {
  const member = await this.memberRepository.findByInviteToken(command.token);
  if (!member) throw new BadRequestException('No organization found');
  if (!member.invite) throw new BadRequestException('No active invite found for user');

  const organization = await this.organizationRepository.findById(member._organizationId);
  const user = await this.userRepository.findById(command.userId);  // 只通过 userId 查找用户

  if (member.memberStatus !== MemberStatusEnum.INVITED)
    throw new BadRequestException('Token expired');

  // ⚠️  缺失：没有校验 user.email === member.invite.email

  await this.memberRepository.convertInvitedUserToMember(
    this.organizationId,
    command.token,
    {
      memberStatus: MemberStatusEnum.ACTIVE,
      _userId: command.userId,  // 直接绑定当前登录用户
      answerDate: new Date(),
    }
  );

  return this.authService.generateUserToken(user);
}
```

### 9.2 安全隐患

**任何登录用户只要获取到邀请 token，就可以接受该邀请并加入组织。** 不需要其邮箱与邀请邮箱匹配。

**示例攻击场景：**
1. 管理员邀请 `alice@example.com`
2. 邀请邮件被拦截，token 泄露
3. 攻击者 `bob@evil.com` 登录系统
4. 攻击者调用 `POST /invites/:stolenToken/accept`
5. 攻击者成功加入组织，获得 `OSS_ADMIN` 权限

### 9.3 测试中的隐式假设

**文件：** `apps/api/src/app/invites/e2e/accept-invite.e2e.ts`

```typescript
// 测试中使用邀请邮箱对应的用户登录来接受邀请
// 但这是测试用例的约定，不是代码强制的校验
expect(member.invite && member.invite.email === invitedUserSession.user.email);
```

---

## 十、重发邀请对 invite.email 一致性的影响

### 10.1 Bug 分析

**文件：** `apps/api/src/app/invites/usecases/resend-invite/resend-invite.usecase.ts`

```typescript
async execute(command: ResendInviteCommand) {
  const organization = await this.organizationRepository.findById(command.organizationId);
  const foundInvitee = await this.memberRepository.findOne({
    _id: command.memberId,
    _organizationId: command.organizationId,
  });

  // foundInvitee.invite.email 此时是有值的（首次邀请时写入）

  const token = createGuid();

  // 发送新邀请邮件到原邮箱（使用 foundInvitee.invite.email）
  await novu.trigger({
    to: [{ subscriberId: foundInvitee.invite.email, email: foundInvitee.invite.email }],
    payload: { acceptInviteUrl: `.../${token}` },
  });

  // ⚠️  更新 invite 时丢失了 email 字段！
  await this.memberRepository.update(foundInvitee, {
    memberStatus: MemberStatusEnum.INVITED,
    invite: {
      token,
      _inviterId: command.userId,
      invitationDate: new Date(),
      // ❌ 没有包含 email: foundInvitee.invite.email
    },
  });
}
```

### 10.2 后果

| 阶段 | member.invite.email 的值 |
|------|------------------------|
| 首次邀请后 | `alice@example.com` |
| 重发邀请后 | `undefined`（被覆盖） |

后续操作会受到影响：
1. **获取邀请信息**：`GET /invites/:token` 无法返回邮箱
2. **接受邀请**：虽然当前代码不校验邮箱，但如果未来添加校验会失败
3. **数据一致性**：数据库记录不完整

### 10.3 对比：首次邀请的正确写法

**文件：** `apps/api/src/app/invites/usecases/invite-member/invite-member.usecase.ts`

```typescript
// 首次邀请时完整写入 invite 对象
await this.memberRepository.addMember(organization._id, {
  roles: [command.role as MemberRoleEnum],
  memberStatus: MemberStatusEnum.INVITED,
  invite: {
    token,
    _inviterId: command.userId,
    email: command.email,  // ✅ 包含 email
    invitationDate: new Date(),
  },
});
```

---

## 十一、接受邀请后 Token 的组织选择逻辑不稳定

### 11.1 问题分析

**文件：** `apps/api/src/app/invites/usecases/accept-invite/accept-invite.usecase.ts:51`

```typescript
return this.authService.generateUserToken(user);  // ⚠️  没有指定组织 ID！
```

**文件：** `apps/api/src/app/auth/services/community.auth.service.ts:219`

```typescript
public async generateUserToken(user: UserEntity) {
  // 查询用户所有活跃组织（按 MongoDB 默认排序，通常是创建时间升序）
  const userActiveOrganizations = await this.organizationRepository.findUserActiveOrganizations(user._id);

  if (userActiveOrganizations?.length > 0) {
    const organizationToSwitch = userActiveOrganizations[0];  // ⚠️  总是取第一个组织！

    return this.switchOrganizationUsecase.execute(
      SwitchOrganizationCommand.create({
        newOrganizationId: organizationToSwitch._id,
        userId: user._id,
      })
    );
  }

  return this.getSignedToken(user);  // 没有组织时返回无组织的 token
}
```

### 11.2 不稳定的表现

| 场景 | 邀请的组织 | 结果 Token 中的 organizationId | 是否一致 |
|------|-----------|-------------------------------|---------|
| 新用户（无组织）接受邀请 | Org-B | null | ❌ 空值 |
| 已有 1 个组织（Org-A）的用户接受邀请到 Org-B | Org-B | Org-A 的 _id | ❌ 不一致 |
| 已有 2 个组织（Org-A, Org-B）的用户接受邀请到 Org-C | Org-C | Org-A 的 _id（第一个） | ❌ 不一致 |

**关键问题：** 接受邀请后返回的 Token 中的 `organizationId` **不保证**是被邀请加入的组织。用户可能被"无声"地切换到了另一个组织。

### 11.3 根因

`generateUserToken()` 的设计目标是"生成一个带默认组织的 token"，而不是"生成指定组织的 token"。接受邀请的场景需要后者，但错误地调用了前者。

**正确的调用应该是：**

```typescript
// 直接为被邀请的组织生成 token，而不是取第一个组织
return this.authService.getSignedToken(user, this.organizationId, member);
```

---

## 十二、Invite Token 的唯一性与更新语义风险

### 12.1 Token 生成方式

**文件：** `libs/application-generic/src/services/helper-service/helper.service.ts:3`

```typescript
export function createGuid(): string {
  return uuidv1();  // 基于时间戳 + MAC 地址的 UUID v1
}
```

### 12.2 数据库层面的唯一性保障

**文件：** `libs/dal/src/repositories/member/community.member.repository.ts:137`

```typescript
async findByInviteToken(token: string) {
  return await this.findOne({ 'invite.token': token });  // 仅查询，无唯一索引约束
}
```

**风险点：**
1. **无数据库唯一索引**：`invite.token` 字段没有唯一索引，理论上可能存在重复 token（虽然 UUID v1 冲突概率极低）
2. **全局搜索而非组织内**：`findByInviteToken` 搜索整个 members 集合，而非限定在某个组织内。如果跨组织出现 token 冲突，可能返回错误的成员记录

### 12.3 更新语义：重发邀请 = 旧 Token 立即失效

**文件：** `apps/api/src/app/invites/usecases/resend-invite/resend-invite.usecase.ts:71`

```typescript
await this.memberRepository.update(foundInvitee, {
  memberStatus: MemberStatusEnum.INVITED,
  invite: {
    token,              // 新 token 覆盖旧 token
    _inviterId: command.userId,
    invitationDate: new Date(),
  },
});
```

**后果：**
- 旧邮件中的链接立即失效（即使邮件还没被看到）
- 用户点击旧链接会得到 "No invite found" 错误
- 没有给旧 token 设置宽限期或多 token 并存机制

### 12.4 并发风险

接受邀请的流程：
1. `findByInviteToken(token)` 查找成员
2. 检查 `memberStatus === MemberStatusEnum.INVITED`
3. 更新为 ACTIVE 并绑定 userId

**问题：** 这三步操作不是原子的。如果同一个 token 被并发请求，可能出现竞态条件（虽然实际影响有限，因为 `_userId` 会被最后一个请求覆盖）。

---

## 十三、邀请邮箱查重与查询的规范化口径差异

### 13.1 normalizeEmail 函数定义

**文件：** `packages/shared/src/utils/normalizeEmail.ts:28`

```typescript
export function normalizeEmail(email: string): string {
  if (typeof email !== 'string') throw new TypeError('normalize-email expects a string');

  const lowerCasedEmail = email.toLowerCase();
  const emailParts = lowerCasedEmail.split(/@/);

  if (emailParts.length !== 2) return email;

  // 规范化逻辑：转小写 + 去除 Gmail 点号 + 去除 + 后缀
  // ...
}
```

### 13.2 三处邮箱操作的规范化差异

| 操作 | 代码位置 | 是否 normalize | 说明 |
|------|---------|---------------|------|
| **查重（邀请前）** | `invite-member.usecase.ts:23` → `findInviteeByEmail` | ❌ 否 | 直接用原始 email 查询 `'invite.email': email` |
| **存储（写入 DB）** | `invite-member.usecase.ts:52` | ❌ 否 | 存储用户输入的原始 email |
| **查询（查用户）** | `get-invite.usecase.ts:33` → `findByEmail` | ✅ 是 | `normalizeEmail(invitedMember.invite.email)` |

### 13.3 问题表现：重复邀请漏洞

```
步骤 1: 邀请 Alice@Example.com  →  存储原始邮箱: "Alice@Example.com"
步骤 2: 邀请 alice@example.com  →  查重用原始邮箱查询，找不到记录
                             →  ✅ 成功！同一邮箱被重复邀请两次
```

因为查重时不做 normalize，`Alice@Example.com` 和 `alice@example.com` 被认为是不同的邮箱，可以被重复邀请到同一个组织。

### 13.4 存储与查询不匹配的副作用

`get-invite.usecase.ts:33` 中：

```typescript
// DB 中存储的是 "Alice@Example.com"（原始）
// 但查询用户时用 normalize 后的 "alice@example.com"
const invitedUser = await this.userRepository.findByEmail(normalizeEmail(invitedMember.invite.email));
```

这意味着：
- 如果用户注册时邮箱是 `alice@example.com`（规范化存储）
- 但邀请邮箱是 `Alice@Example.com`（原始存储）
- 查询时会 normalize 后查询，能正确匹配（这是正确的）

但反过来：
- 如果用户表没有做邮箱规范化存储
- 则可能出现匹配失败

---

## 十四、设计要点与注意事项

1. **角色在邀请发起时即已确定**：`InviteMember` usecase 中 `roles` 字段直接写入 Member 记录，接受邀请时不修改角色。这意味着角色选择必须在邀请发起时完成。

2. **社区版角色硬编码为 OSS_ADMIN**：在 `invites.controller.ts` 中，邀请和批量邀请的角色都硬编码为 `OSS_ADMIN`，前端无法选择角色。EE 版的角色选择逻辑在 `@novu/ee-auth` 模块中。

3. **权限守卫存在于 EE 模块**：社区版的 `CommunityUserAuthGuard` 仅做认证（JWT/API Key 身份验证），不做权限校验。`@RequirePermissions()` 装饰器在社区版中实际不生效（没有 Guard 读取元数据）。EE 版的 `@novu/ee-auth` 模块提供了完整的权限守卫，会读取 `@RequirePermissions()` 元数据并校验 `session.permissions`。社区版中唯一一处基于角色的逻辑是 `get-members.usecase.ts:15` 中对 `command.user.roles.includes(MemberRoleEnum.OSS_ADMIN)` 的检查，用于非管理员的数据可见性过滤。

4. **API Key 认证绕过角色限制**：通过 API Key 认证的用户自动获得 `ALL_PERMISSIONS`，不受 `ROLE_PERMISSIONS` 映射限制。

5. **前端权限双重门槛**：前端 `useHasPermission` 同时检查 RBAC 功能开关和订阅等级，未启用时所有权限通过。

6. **成员列表数据过滤**：非 OSS_ADMIN 看不到已邀请成员和邮箱，这是服务端的数据过滤，不是权限守卫。

7. **社区版 JWT 中 roles 确实能到达 UserSessionData**：`getSignedToken()` 将 `member.roles` 写入 JWT payload，passport-jwt 自动将 payload 映射为 `UserSessionData`，因此 `session.roles` 有值。但社区版缺少将 roles 转换为 permissions 并执行校验的守卫，`session.permissions` 始终为 `undefined`。唯一消费 `session.roles` 的代码是 `get-members.usecase.ts:15`。

8. **接受邀请不校验邮箱**：`AcceptInvite` usecase 不验证当前登录用户邮箱是否等于 `member.invite.email`，存在 token 泄露风险。

9. **重发邀请丢失 invite.email**：`ResendInvite` usecase 更新 invite 对象时未保留 email 字段，导致数据不一致。

10. **EE 权限校验分级**：Business tier 完整校验权限，Free/Pro tier 绕过权限校验，API Key 始终绕过。

11. **接受邀请后 Token 组织 ID 不稳定**：`generateUserToken()` 总是取用户的第一个组织（按创建时间排序），而非被邀请的组织。新用户甚至会得到无组织的 token（`organizationId: null`）。

12. **Invite Token 无数据库唯一约束**：`invite.token` 字段仅用 UUID v1 保证唯一性，但无数据库唯一索引。跨组织查询时理论上可能返回错误记录。重发邀请会使旧 Token 立即失效，无宽限期。

13. **邮箱规范化口径不一致**：邀请查重和存储时不做 normalize，查询用户时做 normalize。大小写不同的同一邮箱可被重复邀请到同一组织。