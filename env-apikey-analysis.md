# 环境 API Key 认证与上下文拼装分析

## 概述

本文档深入分析 Novu 平台中环境维度 API Key 的完整处理流程，涵盖**密钥识别**、**上下文注入**、**租户隔离**三个核心环节。API Key 认证是外部系统与 Novu 交互的主要认证方式，其处理流程涉及多层中间件协作。

---

## 一、密钥识别：认证入口与策略选择

### 1.1 认证入口：CommunityUserAuthGuard

**文件位置**: `apps/api/src/app/auth/framework/community.user.auth.guard.ts:8-47`

这是 API 请求的第一道认证关卡，负责识别认证方案并选择相应策略：

```typescript
export class CommunityUserAuthGuard extends AuthGuard([PassportStrategyEnum.JWT, PassportStrategyEnum.HEADER_API_KEY]) {
  getAuthenticateOptions(context: ExecutionContext): IAuthModuleOptions<any> {
    const request = context.switchToHttp().getRequest();
    const authorizationHeader = request.headers.authorization;
    const authScheme = authorizationHeader?.split(' ')[0] || NONE_AUTH_SCHEME;
    
    switch (authScheme) {
      case ApiAuthSchemeEnum.API_KEY: {
        // 关键检查：端点是否允许外部 API 访问
        const apiEnabled = this.reflector.get<boolean>('external_api_accessible', context.getHandler());
        if (!apiEnabled) throw new UnauthorizedException('API endpoint not accessible');

        return {
          session: false,
          defaultStrategy: PassportStrategyEnum.HEADER_API_KEY,
        };
      }
      // ... 其他认证方案
    }
  }
}
```

**关键条件**:
- 请求头必须包含 `Authorization: ApiKey <your-key>`
- 目标端点必须标记 `@ExternalApiAccessible()` 装饰器
- 装饰器通过 `SetMetadata('external_api_accessible', true)` 标记端点权限

### 1.2 API Key 策略：ApiKeyStrategy

**文件位置**: `apps/api/src/app/auth/services/passport/apikey.strategy.ts:16-75`

Passport 策略层，负责执行具体的密钥验证逻辑：

```typescript
export class ApiKeyStrategy extends PassportStrategy(HeaderAPIKeyStrategy) {
  constructor(
    private readonly authService: AuthService,
    private readonly featureFlagsService: FeatureFlagsService,
    private readonly inMemoryLRUCacheService: InMemoryLRUCacheService
  ) {
    super(
      { header: HttpRequestHeaderKeysEnum.AUTHORIZATION, prefix: `${ApiAuthSchemeEnum.API_KEY} ` },
      true,
      async (apikey: string, verified: (err: Error | null, user?: UserSessionData | false) => void) => {
        const user = await this.validateApiKey(apikey);
        // ... 验证逻辑
      }
    );
  }

  private async validateApiKey(apiKey: string): Promise<UserSessionData | null> {
    // 哈希计算：sha256 哈希作为缓存键和数据库查询键
    const hashedApiKey = createHash('sha256').update(apiKey).digest('hex');

    // LRU 缓存：命中则直接返回，未命中调用 authService
    const user = await this.inMemoryLRUCacheService.get(
      InMemoryLRUCacheStore.API_KEY_USER,
      hashedApiKey,
      () => this.authService.getUserByApiKey(apiKey),
      { environmentId: 'system' }
    );

    // 组织级熔断开关检查
    if (user) {
      await this.checkKillSwitch(user);
    }

    return user;
  }
}
```

**识别流程要点**:
1. **哈希前置**: API Key 从不明文存储，使用 SHA-256 哈希作为查询键
2. **两级缓存**: 内存 LRU 缓存 → 数据库查询
3. **熔断检查**: 验证通过后检查组织级 kill switch 标志

---

## 二、上下文注入：UserSessionData 构建与传播

### 2.1 会话数据构建：CommunityAuthService.getUserByApiKey

**文件位置**: `apps/api/src/app/auth/services/community.auth.service.ts:176-197`

这是构建用户会话的核心方法：

```typescript
@Instrument()
public async getUserByApiKey(apiKey: string): Promise<UserSessionData> {
  const { environment, user, error } = await this.getApiKeyUser({ apiKey });

  if (error) throw new UnauthorizedException(error);
  if (!user) throw new UnauthorizedException('User not found');

  return {
    _id: user._id,
    firstName: user.firstName,
    lastName: user.lastName || undefined,
    email: user.email,
    profilePicture: user.profilePicture || undefined,
    roles: [MemberRoleEnum.OSS_ADMIN],        // 社区版默认角色
    permissions: ALL_PERMISSIONS,             // 社区版默认全权限
    organizationId: environment?._organizationId || '',  // 租户标识
    environmentId: environment?._id || '',               // 环境标识
    scheme: ApiAuthSchemeEnum.API_KEY,         // 认证方案标记
  };
}
```

**UserSessionData 结构定义**: `packages/shared/src/types/auth.ts:9-20`

```typescript
export type UserSessionData = {
  _id: string;                    // 用户 ID
  organizationId: string;         // 租户隔离键
  environmentId: string;          // 环境隔离键
  roles: MemberRoleEnum[];
  permissions: PermissionsEnum[];
  scheme: ApiAuthSchemeEnum;      // 认证来源：BEARER | API_KEY | KEYLESS
  // ... 用户基本信息
};
```

### 2.2 环境-用户关联查询

**文件位置**: `apps/api/src/app/auth/services/community.auth.service.ts:331-359`

```typescript
private async getApiKeyUser({ apiKey }: { apiKey: string }): Promise<{
  environment?: EnvironmentEntity;
  user?: UserEntity;
  error?: string;
}> {
  const hashedApiKey = createHash('sha256').update(apiKey).digest('hex');

  // 步骤1：通过哈希查找环境
  const environment = await this.environmentRepository.findByApiKey({ hash: hashedApiKey });
  if (!environment) return { error: 'API Key not found' };

  // 步骤2：在环境的 apiKeys 数组中匹配具体密钥
  const key = environment.apiKeys.find((i) => i.hash === hashedApiKey);
  if (!key) return { error: 'API Key not found' };

  // 步骤3：通过 key._userId 查找关联用户
  const user = await this.userRepository.findById(key._userId);
  if (!user) return { error: 'User not found' };

  return { environment, user };
}
```

**数据模型关联**: `libs/dal/src/repositories/environment/environment.entity.ts:7-16`

```typescript
export interface IApiKey {
  key: EncryptedSecret | string;  // 加密存储的密钥值（向后兼容）
  hash?: string;                  // SHA-256 哈希，用于快速查询
  _userId: string;                // 创建/拥有该密钥的用户
}
```

### 2.3 会话注入：UserSession 装饰器

**文件位置**: `libs/application-generic/src/decorators/user-session.decorator.ts:1-20`

```typescript
export const UserSession = createParamDecorator((data, ctx) => {
  const req = ctx.switchToHttp().getRequest();
  
  // Passport 验证成功后将 user 对象挂载到 req.user
  if (req.user) {
    return req.user;
  }

  throw new InternalServerErrorException('No user in request - forgot AuthGuard?');
});
```

**控制器层使用示例**: `apps/api/src/app/events/events.controller.ts:104-133`

```typescript
@ExternalApiAccessible()
@Post('/trigger')
async trigger(
  @UserSession() user: UserSessionData,  // 注入会话
  @Body() body: TriggerEventRequestDto
): Promise<TriggerEventResponseDto> {
  // 会话数据直接透传给 UseCase
  const result = await this.parseEventRequest.execute(
    ParseEventRequestMulticastCommand.create({
      userId: user._id,
      environmentId: user.environmentId,    // 环境隔离
      organizationId: user.organizationId,  // 租户隔离
      // ... 其他参数
    })
  );
}
```

---

## 三、租户隔离：查询层的强制边界

### 3.1 隔离原则

**核心设计**: 所有跨租户数据查询必须同时携带 `organizationId` 和 `environmentId`，通过在 Repository 层强制过滤实现隔离。

### 3.2 环境维度查询模式

**文件位置**: `libs/dal/src/repositories/environment/environment.repository.ts:61-65`

```typescript
async findByApiKey({ hash }: { hash: string }) {
  return await this.findOne({ 'apiKeys.hash': hash }, '_id _organizationId apiKeys', {
    readPreference: 'secondaryPreferred',  // 读从库
  });
}
```

**返回投影优化**: 仅返回 `_id`、`_organizationId`、`apiKeys` 三个必要字段，减少数据传输。

### 3.3 业务实体查询隔离

以 NotificationTemplate 为例，`libs/dal/src/repositories/notification-template/notification-template.repository.ts`:

**模式1：环境隔离** (大多数查询)
```typescript
async findByTriggerIdentifier(environmentId: string, identifier: string, ...) {
  const requestQuery: NotificationTemplateQuery = {
    _environmentId: environmentId,  // 强制环境过滤
    'triggers.identifier': identifier,
  };
  // ...
}
```

**模式2：租户+环境双重隔离**
```typescript
async findPublishable(environmentId: string, organizationId: string) {
  const items = await this.MongooseModel.find({
    _environmentId: environmentId,     // 环境隔离
    _organizationId: organizationId,   // 租户隔离
    type: ResourceTypeEnum.BRIDGE,
    origin: ResourceOriginEnum.NOVU_CLOUD,
  });
  // ...
}
```

**模式3：ID+环境联合查询**
```typescript
async findById(id: string, environmentId: string, ...) {
  const query = this.MongooseModel.findOne({
    _id: id,
    _environmentId: environmentId,  // 防止 ID 越权访问
  });
  // ...
}
```

### 3.4 API Key 的租户边界

**关键约束**:
- 每个 API Key 属于且仅属于一个 `Environment`
- 每个 Environment 属于且仅属于一个 `Organization`
- API Key 的 `_userId` 指向创建该密钥的用户
- 会话的 `organizationId` 和 `environmentId` 继承自关联的 Environment

---

## 四、完整调用链路图

```
HTTP Request
    ↓
[Header: Authorization: ApiKey xxx]
    ↓
CommunityUserAuthGuard
    ├─ 解析 authScheme = "ApiKey"
    ├─ 检查 @ExternalApiAccessible() 元数据
    └─ 选择 HEADER_API_KEY 策略
        ↓
ApiKeyStrategy.validateApiKey
    ├─ SHA-256(apiKey) → hashedKey
    ├─ LRU Cache 查找 (key: hashedKey)
    │   ├─ 命中 → 直接返回 UserSessionData
    │   └─ 未命中 → 调用 authService.getUserByApiKey
    └─ KillSwitch 检查
        ↓
CommunityAuthService.getUserByApiKey
    ├─ getApiKeyUser()
    │   ├─ EnvironmentRepository.findByApiKey(hash)
    │   │   └─ MongoDB 查询: { "apiKeys.hash": hash }
    │   ├─ 匹配 environment.apiKeys[] 数组
    │   └─ UserRepository.findById(key._userId)
    └─ 构建 UserSessionData
        ├─ organizationId = environment._organizationId
        ├─ environmentId = environment._id
        ├─ roles/permissions (社区版默认全集)
        └─ scheme = API_KEY
            ↓
Passport 挂载 req.user = UserSessionData
    ↓
Controller @UserSession() 注入
    ├─ userId: user._id
    ├─ organizationId: user.organizationId  [租户隔离键]
    └─ environmentId: user.environmentId    [环境隔离键]
        ↓
UseCase Command 透传
    ↓
Repository 查询层强制过滤
    └─ { _organizationId, _environmentId }
```

---

## 五、关键安全机制

### 5.1 密钥存储安全

| 层级 | 存储方式 | 用途 |
|------|---------|------|
| 数据库 | `apiKeys.key` (EncryptedSecret) | 用于需要展示密钥前缀的场景 |
| 数据库 | `apiKeys.hash` (SHA-256) | 用于快速查询验证 |
| 缓存 | key = SHA-256(apiKey) | 内存缓存从不存储明文 |
| 传输 | HTTPS + Authorization Header | 传输过程加密 |

### 5.2 权限模型差异

- **社区版 (OSS)**: `roles: [OSS_ADMIN]`, `permissions: ALL_PERMISSIONS`
- **企业版 (EE)**: 精细化 RBAC，基于 JWT claims 中的 `org_permissions` 数组校验

### 5.3 环境类型保护

**文件位置**: `apps/api/src/app/auth/framework/root-environment-guard.service.ts`

```typescript
@Injectable()
export class RootEnvironmentGuard implements CanActivate {
  async canActivate(context: ExecutionContext) {
    const { user } = context.switchToHttp().getRequest();
    const isRootEnv = await this.authService.isRootEnvironment(user);
    
    if (isRootEnv) {
      throw new UnauthorizedException('This action is only allowed in Development environment');
    }
    return true;
  }
}
```

**用途**: 防止在生产环境执行开发操作（如工作流编辑、测试发送等）

---

## 六、常见调试点

| 问题 | 检查路径 | 关键条件 |
|------|---------|---------|
| 401 "API endpoint not accessible" | CommunityUserAuthGuard | 缺少 `@ExternalApiAccessible()` 装饰器 |
| 401 "API Key not found" | getApiKeyUser | 哈希不匹配或环境不存在 |
| 401 "User not found" | getApiKeyUser | key._userId 对应用户已删除 |
| 503 "Service temporarily unavailable" | checkKillSwitch | 组织级 kill switch 开启 |
| 数据越权访问 | Repository 层 | 确保查询包含 `_environmentId` 过滤 |
| 缓存不一致 | ApiKeyStrategy | LRU 缓存 TTL 配置（默认值需查配置） |
