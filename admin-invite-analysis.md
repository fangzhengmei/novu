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

---

## 七、设计要点与注意事项

1. **角色在邀请发起时即已确定**：`InviteMember` usecase 中 `roles` 字段直接写入 Member 记录，接受邀请时不修改角色。这意味着角色选择必须在邀请发起时完成。

2. **社区版角色硬编码为 OSS_ADMIN**：在 `invites.controller.ts` 中，邀请和批量邀请的角色都硬编码为 `OSS_ADMIN`，前端无法选择角色。EE 版的角色选择逻辑在 `@novu/ee-auth` 模块中。

3. **权限守卫存在于 EE 模块**：社区版的 `CommunityUserAuthGuard` 仅做认证，不做权限校验。`RequirePermissions` 装饰器在社区版中实际不生效（没有守卫读取元数据）。

4. **API Key 认证绕过角色限制**：通过 API Key 认证的用户自动获得 `ALL_PERMISSIONS`，不受 `ROLE_PERMISSIONS` 映射限制。

5. **前端权限双重门槛**：前端 `useHasPermission` 同时检查 RBAC 功能开关和订阅等级，未启用时所有权限通过。

6. **成员列表数据过滤**：非 OSS_ADMIN 看不到已邀请成员和邮箱，这是服务端的数据过滤，不是权限守卫。

7. **Token 中包含角色数组**：`getSignedToken` 将 member.roles 写入 JWT payload，但社区版 JWT strategy 不解析 roles 到 UserSessionData（仅 EE 版完整解析）。