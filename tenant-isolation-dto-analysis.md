# 多租户上下文传输与隔离机制分析

## 1. 架构概览

Novu 采用三层租户隔离模型：`Organization(组织)` → `Environment(环境)` → `Tenant(租户)`。数据隔离通过类型系统、中间件、仓储层三层防御实现，确保跨租户数据不会泄露。

```
┌─────────────────────────────────────────────────────────────┐
│                    请求入口层 (API Gateway)                 │
│  • JwtStrategy / ApiKeyStrategy 认证                        │
│  • CommunityUserAuthGuard 守卫                              │
│  • HttpRequestHeaderKeysEnum.NOVU_ENVIRONMENT_ID 头提取     │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    业务服务层 (Use Cases)                   │
│  • UserSessionData 上下文传播                                │
│  • Command 对象显式传递 organizationId/environmentId         │
│  • ProcessTenant 租户解析与验证                              │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    数据访问层 (Repositories)                │
│  • BaseRepository<T, E, T_Enforcement> 泛型约束              │
│  • EnforceEnvId / EnforceEnvOrOrgIds 编译时强制              │
│  • 所有查询方法签名强制包含租户过滤字段                       │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 请求入口：上下文注入机制

### 2.1 认证策略与上下文提取

**JWT 认证策略** (`apps/api/src/app/auth/services/passport/jwt.strategy.ts:24-54`)：
```typescript
async validate(req: http.IncomingMessage, session: UserSessionData) {
  session.scheme = ApiAuthSchemeEnum.BEARER;
  const user = await this.authService.validateUser(session);
  const environmentId = this.resolveEnvironmentId(req, session);
  session.environmentId = environmentId;
  
  // 边界验证：确保 environment 属于该 organization
  if (session.environmentId) {
    const environment = await this.environmentRepository.findOne(
      { _id: session.environmentId, _organizationId: session.organizationId },
      '_id'
    );
    if (!environment) {
      throw new UnauthorizedException('Cannot find environment');
    }
  }
  return session;
}
```

**关键点**：
- 从 JWT Token 解码获取 `organizationId`
- 从 Header `novu-environment-id` 或 Token 获取 `environmentId`
- **双重校验**：验证 environmentId 确实属于该 organizationId，防止越权

### 2.2 API Key 认证策略

**ApiKeyStrategy** (`apps/api/src/app/auth/services/passport/apikey.strategy.ts:43-73`)：
- API Key 本身已绑定到特定 `organizationId` 和 `environmentId`
- 支持组织级 Kill Switch 熔断机制
- 结果缓存于 LRU，避免重复查询

### 2.3 UserSession 装饰器

**用户会话提取** (`libs/application-generic/src/decorators/user-session.decorator.ts:1-20`)：
```typescript
export const UserSession = createParamDecorator((data, ctx) => {
  const req = ctx.getType() === 'graphql' 
    ? ctx.getArgs()[2].req 
    : ctx.switchToHttp().getRequest();
  
  if (req.user) return req.user;
  
  Logger.error('Attempted to access user session without a user in the request');
  throw new InternalServerErrorException();
});
```

**设计意图**：
- 强制检查 `req.user` 存在性，避免未认证请求进入业务逻辑
- 同时支持 HTTP 和 GraphQL 上下文
- 缺失时抛 `InternalServerErrorException`，属于**快速失败**设计

---

## 3. DTO 传输：跨层对象的租户边界

### 3.1 核心会话对象 UserSessionData

**定义** (`packages/shared/src/types/auth.ts:9-20`)：
```typescript
export type UserSessionData = {
  _id: string;
  organizationId: string;       // 租户隔离根键
  environmentId: string;        // 环境级隔离
  roles: MemberRoleEnum[];
  permissions: PermissionsEnum[];
  scheme: ApiAuthSchemeEnum;
};
```

**传播路径**：
1. Passport Strategy → `req.user`
2. Controller `@UserSession()` 参数注入
3. UseCase Command 构造 → 业务层
4. Repository 查询参数

### 3.2 租户上下文 DTO

**ITenantDto** (`packages/shared/src/dto/tenant/tenant.dto.ts:1-21`)：
```typescript
export interface ITenantDto {
  _id?: TenantId;
  identifier: string;           // 业务租户标识
  name?: string;
  _environmentId: EnvironmentId;  // 强制环境绑定
  _organizationId: OrganizationId; // 强制组织绑定
}
```

**TriggerTenantContext** (`packages/shared/src/dto/events/event.interface.ts:12`)：
```typescript
export type TriggerTenantContext = string | ITenantDefine;
```
- 支持两种形式：租户标识符字符串，或完整租户定义对象
- 在 `ProcessTenant` UseCase 中统一解析为 `TenantEntity`

### 3.3 事件触发 DTO 示例

**TriggerEventRequestDto** 中的租户字段：
```typescript
export class TriggerEventRequestDto {
  tenant?: TriggerTenantContext;  // 可选租户上下文
  // ... 其他字段
}
```

**Controller 层传递** (`apps/api/src/app/events/events.controller.ts:104-133`)：
```typescript
async trigger(
  @UserSession() user: UserSessionData,
  @Body() body: TriggerEventRequestDto
): Promise<TriggerEventResponseDto> {
  return await this.parseEventRequest.execute(
    ParseEventRequestMulticastCommand.create({
      userId: user._id,
      environmentId: user.environmentId,    // 从会话注入，不接受客户端
      organizationId: user.organizationId,  // 从会话注入，不接受客户端
      tenant: body.tenant,                  // 客户端传入，但受限于 env/org
      // ...
    })
  );
}
```

**关键安全设计**：
- `environmentId` 和 `organizationId` **绝不从请求 Body 中读取**，只来自认证后的 `UserSessionData`
- 客户端传入的 `tenant` 仅作为业务租户标识，最终查询会强制带上 `_environmentId` 过滤

---

## 4. 业务服务层：上下文传播与验证

### 4.1 Command 模式的显式传递

所有 UseCase Command 必须包含租户上下文字段：
```typescript
class ProcessTenantCommand {
  environmentId: string;
  organizationId: string;
  userId: string;
  tenant: ITenantDefine;
}
```

**设计原则**：
- 禁止通过隐式上下文（如 AsyncLocalStorage）传递租户信息
- 每个 UseCase 调用必须显式传入，确保可追溯性

### 4.2 ProcessTenant UseCase

**租户处理流程** (`libs/application-generic/src/usecases/process-tenant/process-tenant.usecase.ts:19-101`)：
```typescript
public async execute(command: ProcessTenantCommand): Promise<TenantEntity | undefined> {
  try {
    return await this.getTenant(
      command.environmentId,
      command.organizationId,
      command.userId,
      command.tenant
    );
  } catch (e) {
    return undefined;  // 租户处理失败时返回 undefined，不中断主流程
  }
}

private async getTenantByIdentifier({ identifier, _environmentId }) {
  return await this.tenantRepository.findOne({
    _environmentId,  // 强制环境过滤
    identifier,
  });
}
```

**容错设计**：
- 租户解析失败时返回 `undefined` 而非抛出异常
- 避免因租户问题导致整个工作流中断
- 调用方需处理 `undefined` 情况

### 4.3 TriggerEvent 中的租户传播

**关键代码** (`libs/application-generic/src/usecases/trigger-event/trigger-event.usecase.ts:126-150`)：
```typescript
if (mappedCommand.tenant) {
  const tenantProcessed = await this.processTenant.execute(
    ProcessTenantCommand.create({
      environmentId,
      organizationId,
      userId,
      tenant: mappedCommand.tenant,
    })
  );
  
  if (!tenantProcessed) {
    await this.createWorkflowTrace({
      eventType: 'workflow_tenant_processing_failed',
      status: 'warning',
      rawData: { tenantIdentifier: mappedCommand.tenant.identifier },
    });
    Logger.warn(`Tenant ${mappedCommand.tenant.identifier} could not be processed`);
  }
}
```

---

## 5. 数据访问层：编译时强制隔离

这是整个隔离体系中**最核心、最强壮**的防线。

### 5.1 强制类型约束体系

**定义** (`libs/dal/src/types/enforce.ts:1-6`)：
```typescript
export type EnforceOrgId = { _organizationId: OrganizationId };
export type EnforceEnvId = { _environmentId: EnvironmentId };
export type EnforceEnvOrOrgIds = EnforceEnvId | EnforceOrgId;
```

### 5.2 BaseRepository 泛型设计

**核心签名** (`libs/dal/src/repositories/base-repository.ts:39`)：
```typescript
export class BaseRepository<T_DBModel, T_MappedEntity, T_Enforcement> {
  async findOne(
    query: FilterQuery<T_DBModel> & T_Enforcement,  // 交叉类型强制
    select?: ProjectionType<T_MappedEntity>,
    options?: {...}
  ): Promise<T_MappedEntity | null>;
  
  async find(
    query: FilterQuery<T_DBModel> & T_Enforcement,  // 所有方法都应用
    ...
  ): Promise<T_MappedEntity[]>;
  
  async update(
    query: FilterQuery<T_DBModel> & T_Enforcement,  // 写入操作同样强制
    updateBody: UpdateQuery<T_DBModel>,
    ...
  ): Promise<{ matched: number; modified: number }>;
}
```

### 5.3 仓储类声明示例

不同集合根据数据敏感度选择不同的强制级别：

```typescript
// 严格环境级隔离（如 Tenant、Message）
export class TenantRepository extends BaseRepository<
  TenantDBModel, TenantEntity, EnforceEnvId
> { ... }

// 环境或组织级隔离（如 Subscriber、NotificationTemplate）
export class SubscriberRepository extends BaseRepository<
  SubscriberDBModel, SubscriberEntity, EnforceEnvOrOrgIds
> { ... }

// 无强制隔离（如 Organization 自身）
export class OrganizationRepository implements IOrganizationRepository { ... }
```

### 5.4 编译时保护效果

**正确代码（编译通过）**：
```typescript
// ✅ 包含 _environmentId，满足 EnforceEnvId
const tenant = await this.tenantRepository.findOne({
  _environmentId: environmentId,
  identifier: 'customer-a',
});
```

**错误代码（编译失败）**：
```typescript
// ❌ 缺少 _environmentId，TypeScript 报错
const tenant = await this.tenantRepository.findOne({
  identifier: 'customer-a',  // Error: Property '_environmentId' is missing
});
```

### 5.5 BaseRepositoryV2 增强

**V2 版本新增保护** (`libs/dal/src/repositories/base-repository-v2.ts:84-110`)：
- `select` 参数**必填**，避免 `SELECT *` 导致敏感字段泄露
- 返回类型自动推断为 `Pick<Entity, Keys>`，只返回请求字段
- `.lean()` 只读查询，避免 Mongoose 文档副作用
- 新增 `findById` 方法，同样强制执行 `T_Enforcement`

---

## 6. 错误处理路径：边界防守策略

### 6.1 分层异常体系

| 层级 | 异常类型 | 触发场景 | 处理方式 |
|------|---------|---------|---------|
| 认证层 | `UnauthorizedException` | 无效 Token、API Key | 立即返回 401 |
| 权限层 | `ForbiddenException` | 权限不足 | 立即返回 403 |
| 业务层 | `BadRequestException` | 参数校验失败 | 返回 400 + 错误详情 |
| 数据层 | `DalException` | 数据库操作异常 | 包装后向上抛出 |
| 熔断层 | `ServiceUnavailableException` | 组织级 Kill Switch 触发 | 返回 503 |

### 6.2 认证边界快速失败

**CommunityUserAuthGuard** (`apps/api/src/app/auth/framework/community.user.auth.guard.ts:42-45`)：
```typescript
case NONE_AUTH_SCHEME:
  throw new UnauthorizedException('Missing authorization header');
default:
  throw new UnauthorizedException(`Invalid authentication scheme: "${authScheme}"`);
```

**JwtStrategy 边界校验** (`jwt.strategy.ts:37-49`)：
```typescript
if (session.environmentId) {
  const environment = await this.environmentRepository.findOne(
    { _id: session.environmentId, _organizationId: session.organizationId },
    '_id'
  );
  if (!environment) {
    throw new UnauthorizedException('Cannot find environment');
  }
}
```

### 6.3 组织级熔断机制

**Kill Switch** (`apps/api/src/app/events/events.controller.ts:71-83`)：
```typescript
private async checkKillSwitch(user: UserSessionData): Promise<void> {
  const isKillSwitchEnabled = await this.featureFlagsService.getFlag({
    key: FeatureFlagsKeysEnum.IS_ORG_KILLSWITCH_FLAG_ENABLED,
    organization: { _id: user.organizationId },
    environment: { _id: user.environmentId },
    component: 'trigger',
  });
  
  if (isKillSwitchEnabled) {
    throw new ServiceUnavailableException(
      'Service temporarily unavailable for this organization'
    );
  }
}
```

**应用场景**：
- 特定组织出现异常流量时可快速熔断
- 不影响其他租户正常服务
- 支持按环境粒度控制

### 6.4 异常信息脱敏

**DalException 定义** (`libs/dal/src/shared/exceptions/dal.exception.ts:1`)：
```typescript
export class DalException extends Error {}
```

**使用模式**：
```typescript
try {
  result = await this.MongooseModel.insertMany(data, { ordered });
} catch (e: unknown) {
  if (e instanceof Error) {
    throw new DalException(e.message);  // 保留原始错误信息
  }
  throw new DalException('An unknown error occurred');
}
```

**注意**：当前实现直接透传原始错误消息。生产环境应考虑：
- 数据库错误码映射为业务错误码
- 避免暴露 MongoDB 内部错误细节

### 6.5 租户处理容错边界

**ProcessTenant 异常吞噬** (`process-tenant.usecase.ts:25-33`)：
```typescript
try {
  tenantEntity = await this.getTenant(environmentId, organizationId, userId, tenant);
} catch (e) {
  tenantEntity = null;  // 静默失败
}

if (tenantEntity === null) {
  return undefined;     // 向上返回 undefined
}
```

**设计考量**：
- 租户是可选上下文，不应因租户问题阻断主流程
- 失败时记录 warning 日志，便于排查
- 调用方需处理 `undefined` 场景，采用无租户模式继续

---

## 7. 风险点与潜在漏洞

### 7.1 已识别风险

1. **Aggregate 方法绕过隔离**
   ```typescript
   async aggregate(query: any[], options = {}): Promise<any> {
     return await this.MongooseModel.aggregate(query)
       .read(options.readPreference || 'primary');
   }
   ```
   - `aggregate` 方法**未应用** `T_Enforcement` 约束
   - 调用方可构造任意聚合管道，存在越权风险
   - **建议**：增加 aggregate 方法的类型约束，或在运行时检查 pipeline

2. **bulkWrite 无类型约束**
   ```typescript
   async bulkWrite(bulkOperations: any, ordered = false): Promise<any> {
     return this.MongooseModel.bulkWrite(bulkOperations, { ordered });
   }
   ```
   - `bulkOperations` 类型为 `any`，完全绕过类型系统
   - **建议**：为 bulkWrite 添加强类型包装

3. **BaseRepository 无默认 readPreference**
   - V1 版本默认读主库，在高并发场景可能有性能问题
   - V2 版本已修复，支持 `defaultReadPreference` 配置

### 7.2 边界模糊点

1. **OrganizationRepository 无强制隔离**
   - 该仓储直接实现接口而非继承 BaseRepository
   - 需手动确保所有查询包含正确过滤条件

2. **异步任务中的上下文传递**
   - JobEntity 中保存了 `_environmentId` 和 `_organizationId`
   - Worker 消费时需重新构建上下文，需确保不被篡改

3. **错误消息中的租户信息**
   - 部分日志和异常消息可能包含租户标识符
   - 需确保不会在跨租户响应中泄露

---

## 8. 最佳实践总结

### 8.1 DTO 设计原则

✅ **强制字段**：所有跨层传输对象必须包含 `_organizationId` / `_environmentId`  
✅ **只读注入**：租户身份信息只在认证层设置，后续层级不修改  
❌ **禁止透传**：绝不将客户端传入的 organizationId/environmentId 直接用于查询  

### 8.2 仓储使用规范

✅ **优先使用 V2**：新代码使用 `BaseRepositoryV2`，必填 `select` 参数  
✅ **显式过滤**：即使类型系统已强制，代码中仍应清晰可见过滤条件  
❌ **避免 any**：不要使用 `as any` 绕过类型检查  

### 8.3 错误处理模式

✅ **快速失败**：认证/权限检查在最外层完成  
✅ **信息隐藏**：数据层异常向上传递时包装，不暴露底层细节  
✅ **容错边界**：非核心路径（如租户解析）失败时，采用降级策略而非中断  

### 8.4 安全加固建议

1. 为 `aggregate` 和 `bulkWrite` 增加运行时租户字段检查
2. 实现数据库层面的行级安全（RLS）作为最终防线
3. 增加审计日志，记录所有跨租户数据访问
4. 定期扫描代码中绕过类型约束的 `as any` 使用

---

## 9. 关键文件索引

| 模块 | 文件路径 | 核心职责 |
|------|---------|---------|
| 认证策略 | `apps/api/src/app/auth/services/passport/jwt.strategy.ts` | JWT 认证与环境验证 |
| 认证策略 | `apps/api/src/app/auth/services/passport/apikey.strategy.ts` | API Key 认证与熔断 |
| 守卫 | `apps/api/src/app/auth/framework/community.user.auth.guard.ts` | 认证方案路由 |
| 会话装饰器 | `libs/application-generic/src/decorators/user-session.decorator.ts` | 用户会话安全提取 |
| 基础仓储 | `libs/dal/src/repositories/base-repository.ts` | V1 通用数据访问 |
| 基础仓储 | `libs/dal/src/repositories/base-repository-v2.ts` | V2 类型安全数据访问 |
| 强制类型 | `libs/dal/src/types/enforce.ts` | EnforceEnvId / EnforceOrgId 定义 |
| 租户处理 | `libs/application-generic/src/usecases/process-tenant/process-tenant.usecase.ts` | 租户上下文解析 |
| 事件触发 | `libs/application-generic/src/usecases/trigger-event/trigger-event.usecase.ts` | 事件流中的租户传播 |
| 会话类型 | `packages/shared/src/types/auth.ts` | UserSessionData 定义 |
