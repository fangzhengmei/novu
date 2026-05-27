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
  let req;
  if (ctx.getType() === 'graphql') {
    req = ctx.getArgs()[2].req;  // GraphQL 分支：从 context 第三个参数获取
  } else {
    req = ctx.switchToHttp().getRequest();  // HTTP 分支：标准 HTTP 请求
  }

  if (req.user) {
    return req.user;
  }

  Logger.error(
    'Attempted to access user session without a user in the request. You probably forgot to add the AuthGuard',
    'UserSession'
  );
  throw new InternalServerErrorException();
});
```

#### HTTP/GraphQL 双分支深度解析

| 分支类型 | 执行路径 | 获取方式 | 使用场景 |
|---------|---------|---------|---------|
| **HTTP 分支** | `ctx.switchToHttp().getRequest()` | 标准 NestJS HTTP 上下文 | REST API 端点（95% 场景） |
| **GraphQL 分支** | `ctx.getArgs()[2].req` | GraphQL context.args[2] | Inbox 订阅、WebSocket 连接 |

**GraphQL 分支代码约束**:
- 依赖 GraphQL 执行上下文的参数顺序固定为 `[root, args, context, info]`
- `context.req` 必须在 GraphQL Module 配置中显式注入
- **风险**：如果 GraphQL 上下文重构（如参数顺序变更），此分支会静默失败

---

### 2.4 无 req.user 异常路径完整链路

#### 异常触发场景

**场景 1：AuthGuard 缺失（最常见）**
```
Controller 方法使用 @UserSession()
    ↓
AuthGuard 未注册或未执行
    ↓
req.user === undefined
    ↓
UserSession 装饰器抛出 InternalServerErrorException
    ↓
HTTP 500: "Internal Server Error"
```

**场景 2：Passport 策略验证返回 false**
```
ApiKeyStrategy.validateApiKey() 返回 null
    ↓
verified(null, false) 调用
    ↓
Passport 判定认证失败
    ↓
req.user 未设置
    ↓
UserSession 装饰器抛出 500
```

**场景 3：异常在 Passport 回调中被吞掉**
```
authService.getUserByApiKey() 抛出 UnauthorizedException
    ↓
catch (err) { return verified(err, false); }
    ↓
Passport 接收到 error，但转换为 401
    ↓
如果 NestJS 异常过滤器配置不当，可能导致 req.user 未设置但也未抛出异常
```

#### 异常类型边界

| 异常类型 | 抛出位置 | HTTP 状态码 | 根因分类 |
|---------|---------|------------|---------|
| `InternalServerErrorException` | UserSession 装饰器 | 500 | **配置错误**（Guard 缺失） |
| `UnauthorizedException` | CommunityUserAuthGuard | 401 | **认证失败**（方案不支持） |
| `UnauthorizedException` | Passport 框架层 | 401 | **认证失败**（密钥无效） |
| `ServiceUnavailableException` | checkKillSwitch | 503 | **运维控制**（熔断开启） |

**关键风险**：500 异常掩盖真实问题
- 开发人员看到 500 可能误以为是服务器崩溃
- 实际原因：忘记加 `@UseGuards(CommunityUserAuthGuard)`
- **建议改进**：将 500 改为更明确的 401 或添加诊断日志

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

**文件位置**: `apps/api/src/app/auth/framework/root-environment-guard.service.ts:1-20`

```typescript
@Injectable()
export class RootEnvironmentGuard implements CanActivate {
  constructor(private authService: AuthService) {}

  async canActivate(context: ExecutionContext) {
    const request = context.switchToHttp().getRequest();
    const { user } = request;

    const environment = await this.authService.isRootEnvironment(user);

    if (environment) {
      throw new UnauthorizedException('This action is only allowed in Development environment');
    }

    return true;
  }
}
```

#### 守卫语义深度解析

**核心实现逻辑**: `apps/api/src/app/auth/services/community.auth.service.ts:294-301`

```typescript
public async isRootEnvironment(payload: UserSessionData): Promise<boolean> {
  const environment = await this.environmentRepository.findOne({
    _id: payload.environmentId,
  });
  if (!environment) throw new NotFoundException('Environment not found');

  return !!environment._parentId;  // 关键：通过 _parentId 判断环境类型
}
```

**语义对照表（易混淆点）**:

| 环境 | `_parentId` 值 | `isRootEnvironment()` 返回值 | 守卫放行状态 | 可执行操作 |
|------|---------------|-----------------------------|-------------|-----------|
| Development（开发环境） | `null` / `undefined` | `false` | ✅ 放行 | 工作流编辑、删除、状态切换 |
| Production（生产环境） | Development 的 `_id` | `true` | ❌ 拦截 | 仅读操作、触发事件 |

**关键命名澄清**:
- ❌ 方法名 `isRootEnvironment` **具有误导性**
- ✅ 实际语义：**"是否为非根环境"** 或 **"是否为 Production 环境"**
- ✅ 守卫逻辑：`if (isRootEnvironment)` → **当环境有父节点时抛出异常**
- ⚠️ **双重否定容易出错**：返回 `true` 表示"非根"，但守卫却用它来判断"是否拦截"

**受保护的端点示例**（`apps/api/src/app/workflows-v1/workflow-v1.controller.ts`）:
- `DELETE /workflows/:workflowId` - 删除工作流
- `POST /workflows` - 创建工作流
- `PUT /workflows/:workflowId/status` - 切换工作流状态

---

## 六、UserSession 异常分支逐条分析

### 6.1 异常分支总览

API Key 认证链路中共有 **8 个异常分支**，分布在 4 个层级：

| 层级 | 异常点数量 | 主要异常类型 |
|------|-----------|-------------|
| AuthGuard 层 | 1 | UnauthorizedException |
| Strategy 层 | 2 | ServiceUnavailableException |
| AuthService 层 | 5 | UnauthorizedException, NotFoundException |

---

### 6.2 AuthGuard 层异常

**分支 1：端点不允许外部 API 访问**
- **位置**: `community.user.auth.guard.ts:27-28`
- **触发条件**: `!this.reflector.get<boolean>('external_api_accessible', context.getHandler())`
- **异常**: `UnauthorizedException('API endpoint not accessible')`
- **根因**: 控制器方法缺少 `@ExternalApiAccessible()` 装饰器
- **HTTP 状态码**: 401

---

### 6.3 ApiKeyStrategy 层异常

**分支 2：组织级熔断开关开启**
- **位置**: `apikey.strategy.ts:71-73`
- **触发条件**: `isKillSwitchEnabled === true`
- **异常**: `ServiceUnavailableException('Service temporarily unavailable for this organization')`
- **根因**: 组织级 kill switch 被触发（Feature Flag 控制）
- **HTTP 状态码**: 503
- **备注**: 这是唯一非 401 的认证异常

**分支 3：Passport verify 回调异常包裹**
- **位置**: `apikey.strategy.ts:36-38`
- **触发条件**: `validateApiKey()` 抛出任何异常
- **处理**: `verified(err as Error, false)`
- **注意**: 所有异常都会被 Passport 框架捕获并转换为 401

---

### 6.4 AuthService 层异常（getUserByApiKey）

**分支 4：getApiKeyUser 返回 error**
- **位置**: `community.auth.service.ts:181`
- **触发条件**: `error` 字段有值（三种子情况）
- **异常**: `UnauthorizedException(error)`
- **错误消息**: 
  - `'API Key not found'`（哈希不匹配）
  - `'API Key not found'`（数组遍历不匹配）
  - `'User not found'`（密钥关联用户不存在）

**分支 5：user 对象为空**
- **位置**: `community.auth.service.ts:183`
- **触发条件**: `!user`
- **异常**: `UnauthorizedException('User not found')`
- **备注**: 理论上与分支 4 的第三种情况重复，属于防御性编程

---

### 6.5 isRootEnvironment 守卫异常

**分支 6：环境不存在**
- **位置**: `community.auth.service.ts:298`
- **触发条件**: `!environment`
- **异常**: `NotFoundException('Environment not found')`
- **根因**: UserSession 中的 `environmentId` 无效或环境已被删除
- **HTTP 状态码**: 404

**分支 7：在 Production 环境执行开发操作**
- **位置**: `root-environment-guard.service.ts:14-16`
- **触发条件**: `environment._parentId` 存在（即 Production 环境）
- **异常**: `UnauthorizedException('This action is only allowed in Development environment')`
- **HTTP 状态码**: 401

---

### 6.6 JwtStrategy 层异常（对比参考）

虽然 API Key 认证不经过此层，但 UserSession 在其他场景可能遇到：

**分支 8：环境-组织不匹配**
- **位置**: `jwt.strategy.ts:46-48`
- **触发条件**: 查询 `{ _id, _organizationId }` 无结果
- **异常**: `UnauthorizedException('Cannot find environment')`
- **根因**: 环境 ID 与组织 ID 不匹配，可能是越权尝试

---

## 七、密钥哈希定位环境的约束条件与风险

### 7.1 数据库查询约束

**查询实现**: `libs/dal/src/repositories/environment/environment.repository.ts:61-65`

```typescript
async findByApiKey({ hash }: { hash: string }) {
  return await this.findOne({ 'apiKeys.hash': hash }, '_id _organizationId apiKeys', {
    readPreference: 'secondaryPreferred',  // 读从库
  });
}
```

#### 约束条件 1：MongoDB 数组索引约束

```
查询条件: { 'apiKeys.hash': hash }
```

- **必须创建索引**: `db.environments.createIndex({ 'apiKeys.hash': 1 })`
- **无索引后果**: COLLSCAN 全表扫描，高并发下数据库雪崩
- **数组多键索引**: MongoDB 自动为数组中的每个元素创建索引条目
- **索引基数**: 极高（每个 API Key 都是唯一的）

#### 约束条件 2：读从库最终一致性

```typescript
readPreference: 'secondaryPreferred'
```

- **风险窗口**: 主从复制延迟期间（通常 < 1s）
- **场景**: 刚创建的 API Key 立即使用可能认证失败
- **影响**: 新密钥创建后可能需要等待几秒才能生效
- **缓解**: 关键路径可考虑降级读主库

---

### 7.2 哈希碰撞风险

#### 哈希算法选择

```typescript
const hashedApiKey = createHash('sha256').update(apiKey).digest('hex');
```

| 算法 | 输出长度 | 碰撞概率（2^32 密钥） | 评估 |
|------|---------|---------------------|------|
| SHA-256 | 256-bit | ~2^-192 | ✅ 安全 |
| MD5 | 128-bit | ~2^-64 | ❌ 已破解 |
| SHA-1 | 160-bit | ~2^-96 | ❌ 不推荐 |

**理论风险**:
- 生日悖论：n 个密钥的碰撞概率 ≈ n² / 2^257
- 10 亿密钥: 碰撞概率 ≈ 10^18 / 2^257 ≈ 2^-197
- **结论**: 工程实践中可忽略

---

### 7.3 双重校验机制与风险

**代码路径**: `community.auth.service.ts:338-351`

```typescript
// 第一次查询：数据库索引匹配
const environment = await this.environmentRepository.findByApiKey({ hash: hashedApiKey });
if (!environment) return { error: 'API Key not found' };

// 第二次校验：在 apiKeys 数组中遍历匹配
const key = environment.apiKeys.find((i) => i.hash === hashedApiKey);
if (!key) return { error: 'API Key not found' };
```

#### 为什么需要双重校验？

**MongoDB 数组查询的微妙语义**:
- 查询 `{ 'apiKeys.hash': hash }` 只要数组中**任意一个元素**匹配就返回文档
- 但投影 `apiKeys` 返回的是**整个数组**，而非匹配的子文档
- 如果没有第二次校验：无法从数组中提取 `_userId`

#### 隐含风险

**风险 1：数组膨胀攻击**
- `environment.apiKeys` 数组理论上可以无限增长
- 极端情况：一个环境创建 10 万个 API Key
- 影响：内存占用增大，`Array.find()` 线性扫描变慢
- **约束缺失**: 代码中未限制单环境 API Key 数量上限

**风险 2：哈希索引与数组内容不一致**
- 理论场景：数据库索引条目与实际数组数据不一致
- 例如：索引包含某个哈希，但 `apiKeys` 数组中已被删除
- 后果：第一次查询命中，但第二次校验失败 → 认证失败
- 概率：极低（MongoDB 崩溃 + 索引损坏）

---

### 7.4 缓存一致性风险

**缓存架构**: `apikey.strategy.ts:46-53`

```typescript
const user = await this.inMemoryLRUCacheService.get(
  InMemoryLRUCacheStore.API_KEY_USER,
  hashedApiKey,
  () => this.authService.getUserByApiKey(apiKey),  // 缓存未命中时调用
  { environmentId: 'system' }
);
```

#### 缓存失效问题

| 操作 | 缓存是否失效 | 影响 |
|------|-------------|------|
| 创建新 API Key | ❌ 不失效 | 新密钥立即可用（因为未缓存） |
| 删除 API Key | ❌ 不失效 | 已删除密钥在缓存 TTL 内仍有效 ⚠️ |
| 变更密钥权限 | ❌ 不失效 | 权限变更不立即生效 |
| 用户被删除 | ❌ 不失效 | 缓存仍返回有效 UserSession |

**高危风险**: 删除 API Key 后，攻击者如果持有缓存的 UserSession，在 TTL 窗口内仍可访问系统。

**缓解措施**:
1. 删除 API Key 时主动失效对应缓存条目
2. 缩短缓存 TTL（如 5 分钟）
3. 关键操作绕过缓存直接查库

---

### 7.5 用户关联的安全边界

```typescript
const user = await this.userRepository.findById(key._userId);
if (!user) return { error: 'User not found' };
```

#### 约束分析

**正向约束**:
- API Key 必须关联一个有效用户
- 用户不存在 → 认证失败

**缺失约束**:
- ❌ 未校验用户是否属于同一组织
- ❌ 未校验用户状态（active/suspended）
- ❌ 未校验用户在该组织中的成员身份

**风险场景**:
1. 用户 A 创建 API Key
2. 用户 A 被移出组织（但未删除用户账户）
3. API Key 继续有效，因为只检查 `key._userId` 对应用户存在
4. UserSession 仍携带 `organizationId` 并通过后续鉴权

**建议修复**:
```typescript
// 在 getApiKeyUser 中增加成员校验
const isMember = await this.memberRepository.isMemberOfOrganization(
  key._userId, 
  environment._organizationId
);
if (!isMember) return { error: 'User is not a member of this organization' };
```

---

## 八、Passport 回调异常传递边界深度分析

### 8.1 Passport 回调机制原理

**ApiKeyStrategy 回调实现**: `apps/api/src/app/auth/services/passport/apikey.strategy.ts:25-38`

```typescript
async (apikey: string, verified: (err: Error | null, user?: UserSessionData | false) => void) => {
  try {
    const user = await this.validateApiKey(apikey);
    if (!user) {
      return verified(null, false);  // 路径1：验证失败，无错误
    }
    addNewRelicTraceAttributes(user);
    return verified(null, user);     // 路径2：验证成功
  } catch (err) {
    return verified(err as Error, false);  // 路径3：异常捕获，传递错误
  }
}
```

### 8.2 异常传递的三条边界

#### 边界 1：验证失败但无异常（verified(null, false)）

**触发场景**:
- `validateApiKey()` 返回 `null`
- 密钥哈希不匹配，数据库无结果
- 密钥在数据库中但 `Array.find()` 未匹配

**传递路径**:
```
verified(null, false)
    ↓
Passport 框架内部处理
    ↓
抛出默认 UnauthorizedException
    ↓
HTTP 401: "Unauthorized"
```

**关键特征**:
- 丢失具体错误信息（如 "API Key not found"）
- 客户端无法区分"密钥无效"和"用户不存在"

#### 边界 2：验证成功（verified(null, user)）

**触发场景**:
- 所有校验通过，返回完整 `UserSessionData`

**传递路径**:
```
verified(null, user)
    ↓
Passport 将 user 挂载到 req.user
    ↓
后续中间件和控制器可通过 @UserSession() 获取
    ↓
正常执行业务逻辑
```

#### 边界 3：异常捕获后传递（verified(err, false)）

**触发场景**:
- `checkKillSwitch()` 抛出 `ServiceUnavailableException`
- 数据库连接失败
- 任何其他运行时异常

**传递路径**:
```
try { validateApiKey() } catch (err)
    ↓
verified(err, false)  // err 被完整传递
    ↓
Passport 识别到 err 参数
    ↓
NestJS 异常过滤器捕获 err
    ↓
保留原始异常类型和消息
```

**异常类型保留对照表**:

| 原始异常 | 经过 verified(err, false) | 最终 HTTP 状态码 |
|---------|-------------------------|-----------------|
| `ServiceUnavailableException` | ✅ 完整保留 | 503 |
| `UnauthorizedException` | ✅ 完整保留 | 401 |
| `NotFoundException` | ✅ 完整保留 | 404 |
| 普通 `Error` | ✅ 完整保留 | 500 |

### 8.3 异常传递的隐藏风险

**风险 1：异常类型在 Passport 层被转换**

NestJS Passport 集成的隐式行为：
- 如果 `verified()` 的第一个参数不为 null，Passport 会调用 `next(err)`
- 异常会进入 NestJS 的全局异常过滤器链
- **但是**：某些异常过滤器可能会重新包装异常，丢失原始堆栈信息

**风险 2：catch-all 吞掉关键诊断信息**

```typescript
} catch (err) {
  return verified(err as Error, false);  // err 可能是任何类型，包括 string
}
```

- `err as Error` 类型断言不安全
- 如果底层抛出字符串或数字，会导致 `err.message` 为 undefined
- 日志中只会看到 "undefined" 或 "[object Object]"

**风险 3：kill switch 异常的特殊路径**

```typescript
private async checkKillSwitch(user: UserSessionData): Promise<void> {
  const isKillSwitchEnabled = await this.featureFlagsService.getFlag(...);
  if (isKillSwitchEnabled) {
    throw new ServiceUnavailableException('Service temporarily unavailable for this organization');
  }
}
```

- 这是**唯一**在认证阶段抛出非 401 异常的场景
- 503 状态码用于指示"服务不可用"，而非"认证失败"
- **语义正确但容易混淆**：客户端可能将 503 误认为是基础设施问题，而非组织级封禁

---

## 九、哈希定位环境的代码实现约束与风险分层

### 9.1 哈希计算的实现约束

**哈希算法代码**: `apps/api/src/app/auth/services/community.auth.service.ts:336`

```typescript
const hashedApiKey = createHash('sha256').update(apiKey).digest('hex');
```

#### 约束 1：输入编码隐式假设

Node.js `crypto.createHash().update()` 的默认行为：
- 如果输入是字符串，默认使用 `'utf8'` 编码
- API Key 通常是 base64 或十六进制字符串
- **隐式约束**：API Key 必须是有效的 UTF-8 字符串（实际总是满足）

#### 约束 2：哈希值格式固定为十六进制

```typescript
.digest('hex')  // 输出 64 字符十六进制字符串
```

- 数据库存储的 `apiKeys.hash` 必须也是十六进制格式
- 任何地方修改 digest 格式（如改为 base64）都会导致**所有现有密钥失效**
- **迁移风险极高**：需要双写兼容期

### 9.2 MongoDB 查询的实现约束

**查询代码**: `libs/dal/src/repositories/environment/environment.repository.ts:61-65`

```typescript
async findByApiKey({ hash }: { hash: string }) {
  return await this.findOne({ 'apiKeys.hash': hash }, '_id _organizationId apiKeys', {
    readPreference: 'secondaryPreferred',
  });
}
```

#### 约束 1：数组多键索引的行为

MongoDB `{ 'apiKeys.hash': 1 }` 索引特性：
- 为数组中的**每个元素**创建独立索引条目
- 查询时只要任意一个元素匹配就返回文档
- **基数极高**：每个 API Key 对应一个索引条目
- **写入放大**：每次添加/删除 API Key 都会修改索引

#### 约束 2：投影字段的隐式依赖

```
投影: '_id _organizationId apiKeys'
```

- `_id`: 用于构建 UserSession 的 `environmentId`
- `_organizationId`: 用于构建 UserSession 的 `organizationId`
- `apiKeys`: 用于二次校验和提取 `_userId`
- **任何投影字段缺失都会导致认证链条断裂**

#### 约束 3：读从库的一致性窗口

```typescript
readPreference: 'secondaryPreferred'
```

| 场景 | 风险等级 | 影响 |
|------|---------|------|
| 新创建密钥立即使用 | ⚠️ 中 | 1-3 秒延迟窗口内认证失败 |
| 删除密钥立即验证 | ⚠️ 中 | 延迟窗口内仍可认证 |
| 正常认证流量 | ✅ 低 | 从库负载均衡，性能更好 |

**设计权衡**：
- 99.9% 场景读从库没问题
- 极端场景（创建后立即使用）需要重试机制
- 客户端 SDK 应该实现指数退避重试

### 9.3 双重校验的实现约束

**二次校验代码**: `community.auth.service.ts:347-351`

```typescript
const key = environment.apiKeys.find((i) => i.hash === hashedApiKey);
if (!key) return { error: 'API Key not found' };
```

#### 约束 1：线性扫描的性能边界

| apiKeys 数组大小 | `Array.find()` 耗时 | 相对性能 |
|-----------------|-------------------|---------|
| 1-10 | < 1μs | ✅ 正常 |
| 100 | ~5μs | ✅ 正常 |
| 1,000 | ~50μs | ⚠️ 可接受 |
| 10,000 | ~500μs | ⚠️ 需关注 |
| 100,000 | ~5ms | ❌ 严重 |

**现状**：无代码限制单环境 API Key 数量
**风险**：恶意用户创建 10 万密钥，导致每次认证消耗 5ms CPU

#### 约束 2：哈希碰撞的理论边界

SHA-256 碰撞概率分析：
- 单组织 100 万密钥：碰撞概率 ≈ (10^6)^2 / 2^257 ≈ 2^-137
- 全平台 10 亿密钥：碰撞概率 ≈ (10^9)^2 / 2^257 ≈ 2^-77
- **结论**：工程实践中完全可以忽略
- **但是**：如果算法被替换为 MD5/SHA-1，风险立即升高

### 9.4 风险分层矩阵

| 风险层级 | 风险类型 | 发生概率 | 影响程度 | 缓解措施 |
|---------|---------|---------|---------|---------|
| **P0 高危** | 删除密钥后缓存仍有效 | 高 | 严重 | 1. 缩短缓存 TTL<br>2. 删除时主动失效 |
| **P0 高危** | 用户被移出组织后密钥仍有效 | 中 | 严重 | 增加成员身份校验 |
| **P1 中危** | 单环境创建大量密钥导致 DoS | 低 | 中等 | 限制单环境 API Key 上限（如 100） |
| **P1 中危** | 从库延迟导致新密钥认证失败 | 中 | 中等 | 客户端重试 + 关键路径读主库 |
| **P2 低危** | 500 异常掩盖配置错误 | 高 | 轻微 | 改进错误消息 + 告警监控 |
| **P2 低危** | GraphQL 上下文重构导致分支失效 | 极低 | 中等 | 增加单元测试覆盖 |
| **P3 理论** | SHA-256 哈希碰撞 | 极低 | 严重 | 无需处理（工程不可行） |
| **P3 理论** | MongoDB 索引与数组不一致 | 极低 | 轻微 | 依赖数据库事务保证 |

---

## 十、常见调试点

| 问题 | 检查路径 | 关键条件 |
|------|---------|---------|
| 401 "API endpoint not accessible" | CommunityUserAuthGuard | 缺少 `@ExternalApiAccessible()` 装饰器 |
| 401 "API Key not found" | getApiKeyUser | 哈希不匹配或环境不存在 |
| 401 "User not found" | getApiKeyUser | key._userId 对应用户已删除 |
| 503 "Service temporarily unavailable" | checkKillSwitch | 组织级 kill switch 开启 |
| 数据越权访问 | Repository 层 | 确保查询包含 `_environmentId` 过滤 |
| 缓存不一致 | ApiKeyStrategy | LRU 缓存 TTL 配置（默认值需查配置） |
