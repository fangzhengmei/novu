# 多租户上下文传输与隔离机制分析

## 1. 架构概览

Novu 采用三层租户隔离模型：`Organization(组织)` → `Environment(环境)` → `Tenant(租户)`。数据隔离通过**类型系统、中间件、仓储层、全局异常过滤**四层防御实现，确保跨租户数据不会泄露。

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                      请求入口层 (API Gateway)                                │
│  • JwtStrategy / ApiKeyStrategy 认证 → 注入 UserSessionData                   │
│  • CommunityUserAuthGuard 守卫 → 认证方案路由与快速失败                        │
│  • UserSession 装饰器 → 安全提取会话                                          │
└───────────────────────────────────────┬───────────────────────────────────────┘
                                        │
                                        ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│                      业务服务层 (Use Cases)                                  │
│  • Command 模式显式传递 organizationId/environmentId                          │
│  • ProcessTenant 租户解析与验证（可选降级路径）                                │
│  • TriggerMulticast/TriggerBroadcast → Job 分发                                │
└───────────────────────────────────────┬───────────────────────────────────────┘
                                        │
                                        ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│                      数据访问层 (Repositories)                               │
│  • BaseRepository<T, E, T_Enforcement> 泛型约束                                │
│  • EnforceEnvId / EnforceEnvOrOrgIds 编译时强制                                │
│  • 所有查询方法签名强制包含租户过滤字段                                         │
└───────────────────────────────────────┬───────────────────────────────────────┘
                                        │
                                        ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│                      全局异常过滤层 (AllExceptionsFilter)                    │
│  • 500 错误完全脱敏 → 生成 errorId，返回固定消息                                │
│  • 响应扁平化 → ctx 字段提升到顶层                                              │
│  • 按组织/环境上下文写入 ClickHouse 审计日志                                    │
│  • Pino Logger 按环境/组织标记日志                                              │
└───────────────────────────────────────────────────────────────────────────────┘
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
      throw new UnauthorizedException('Cannot find environment', JSON.stringify({ session }));
    }
  }
  return session;
}
```

**关键点**：
- 从 JWT Token 解码获取 `organizationId`
- 从 Header `novu-environment-id` 或 Token 获取 `environmentId`
- **双重校验**：验证 environmentId 确实属于该 organizationId，防止越权
- 验证失败立即抛 `UnauthorizedException`，快速失败

### 2.2 API Key 认证策略

**ApiKeyStrategy** (`apps/api/src/app/auth/services/passport/apikey.strategy.ts:43-73`)：
- API Key 本身已绑定到特定 `organizationId` 和 `environmentId`
- 支持组织级 Kill Switch 熔断机制
- 结果缓存于 LRU，避免重复查询

### 2.3 认证守卫路由

**CommunityUserAuthGuard** (`apps/api/src/app/auth/framework/community.user.auth.guard.ts:17-47`)：
```typescript
getAuthenticateOptions(context: ExecutionContext): IAuthModuleOptions<any> {
  const request = context.switchToHttp().getRequest();
  const authorizationHeader = request.headers.authorization;
  const authScheme = authorizationHeader?.split(' ')[0] || NONE_AUTH_SCHEME;
  request.authScheme = authScheme;

  switch (authScheme) {
    case ApiAuthSchemeEnum.BEARER:
      return { session: false, defaultStrategy: PassportStrategyEnum.JWT };
    case ApiAuthSchemeEnum.API_KEY: {
      const apiEnabled = this.reflector.get<boolean>('external_api_accessible', context.getHandler());
      if (!apiEnabled) throw new UnauthorizedException('API endpoint not accessible');
      return { session: false, defaultStrategy: PassportStrategyEnum.HEADER_API_KEY };
    }
    case NONE_AUTH_SCHEME:
      throw new UnauthorizedException('Missing authorization header');
    default:
      throw new UnauthorizedException(`Invalid authentication scheme: "${authScheme}"`);
  }
}
```

### 2.4 UserSession 装饰器

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
  organizationId: string;       // 租户隔离根键 - 只读
  environmentId: string;        // 环境级隔离 - 只读
  roles: MemberRoleEnum[];
  permissions: PermissionsEnum[];
  scheme: ApiAuthSchemeEnum;
};
```

**传播路径**：
1. Passport Strategy → `req.user`（认证层注入，后续层级只读）
2. Controller `@UserSession()` 参数注入
3. UseCase Command 构造 → 业务层（显式传递）
4. Repository 查询参数（强制类型约束）

### 3.2 Command 基类体系

**完整基类层次** (`libs/application-generic/src/commands/`)：
```
BaseCommand (create() + 验证)
├── AuthenticatedCommand (userId: string)
│   └── OrganizationCommand (organizationId: string)
├── EnvironmentLevelCommand (environmentId ✅, organizationId ❓)
├── EnvironmentLevelWithUserCommand (environmentId ✅, organizationId ❓, userId ✅)
├── OrganizationLevelCommand (environmentId ❓, organizationId ✅)
├── OrganizationLevelWithUserCommand (environmentId ❓, organizationId ✅, userId ✅)
├── EnvironmentWithUserCommand (environmentId ✅, organizationId ✅, userId ✅)  ← API 层主力
├── EnvironmentWithUserObjectCommand (user: UserSessionData)
│   ├── PaginatedListCommand
│   └── CursorBasedPaginatedCommand
├── EnvironmentWithSubscriber (envId ✅, orgId ✅, subscriberId ✅)
└── EnvironmentCommand (environmentId ✅, organizationId ✅)  ← Worker 层主力（无 userId）
```

**各基类用途与继承情况**：

| 基类 | 强制字段 | 典型使用场景 | 继承示例 |
|------|---------|-------------|---------|
| `EnvironmentWithUserCommand` | envId + orgId + userId | API 层业务操作（需审计用户） | `TriggerEventBaseCommand`, `ProcessTenantCommand`, `UpdateTenantCommand` |
| `EnvironmentCommand` | envId + orgId | Worker 内部异步任务（无需用户上下文） | `SendWebhookMessageCommand`, `UpdateSubscriberCommand`, `GetTenantCommand` |
| `EnvironmentLevelCommand` | envId（orgId 可选） | 环境级只读操作，组织上下文可选 | 极少使用 |
| `BaseCommand` | 无 | 纯工具类命令，无租户上下文 | `VerifyPayloadCommand`, `MergePreferencesCommand` |

**关键修正说明**：
- ~~所有业务 UseCase Command 必须继承 EnvironmentWithUserCommand~~ → **按场景分层继承**
- ~~核心隔离字段 envId / orgId 在所有生产路径的 Command 中均为强制非空~~ → **按基类分层强制**

**各基类强制字段分布**（实际使用场景）：

| 基类 | environmentId | organizationId | userId | 继承示例 | 占比 |
|------|--------------|----------------|--------|---------|------|
| `EnvironmentWithUserCommand` | ✅ 强制 | ✅ 强制 | ✅ 强制 | TriggerEvent, ProcessTenant, UpdateTenant | 约 60% |
| `EnvironmentCommand` | ✅ 强制 | ✅ 强制 | ❌ 无 | SendWebhook, CreateOrUpdateSubscriber, GetTenant | 约 35% |
| `OrganizationLevelCommand` | ❌ 可选 | ✅ 强制 | ❌ 无 | TierRestrictionsValidate | &lt; 1% |
| `EnvironmentLevelCommand` | ✅ 强制 | ❌ 可选 | ❌ 无 | ExecuteBridgeRequest, GetDecryptedSecretKey | &lt; 1% |
| `BaseCommand` | ❌ 无 | ❌ 无 | ❌ 无 | VerifyPayload, MergePreferences | &lt; 1% |

**触发链路隔离约束验证**：
- `TriggerEventBaseCommand extends EnvironmentWithUserCommand` → envId + orgId + userId 全部 `@IsNotEmpty()`
- `ProcessTenantCommand extends EnvironmentWithUserCommand` → 同上
- `TriggerMulticastCommand` / `TriggerBroadcastCommand` 包含 `environmentId: string; organizationId: string`（从 BaseTriggerCommand 继承）
- `mapSubscribersToJobs()` 构建 Job 时，`environmentId` 和 `organizationId` 直接从 Command 拷贝，无客户端污染路径
- 最终仓储层 `T_Enforcement` 交叉类型提供编译时最终防线

**结论**：触发链路（Trigger → ProcessTenant → Job 分发）全程满足 envId + orgId 双强制，隔离约束未受基类分层影响。

### 3.3 租户上下文 DTO

**ITenantDefine** (`packages/shared/src/types/tenant.ts:6-13`)：
```typescript
export interface ITenantPayload {
  name?: string;
  data?: CustomDataType;
}

export interface ITenantDefine extends ITenantPayload {
  identifier: string;   // 业务租户标识 - 客户端可传入
}
```

**TriggerTenantContext** (`packages/shared/src/dto/events/event.interface.ts:12`)：
```typescript
export type TriggerTenantContext = string | ITenantDefine;
```

**关键安全特性**：
- `ITenantDefine` **不包含** `_environmentId` / `_organizationId` 字段
- 客户端无法通过 tenant 对象直接注入隔离字段
- 隔离字段始终来自认证后的 `UserSessionData`

### 3.4 事件触发 DTO 示例

**Controller 层传递** (`apps/api/src/app/events/events.controller.ts:104-133`)：
```typescript
async trigger(
  @UserSession() user: UserSessionData,
  @Body() body: TriggerEventRequestDto
): Promise<TriggerEventResponseDto> {
  return await this.parseEventRequest.execute(
    ParseEventRequestMulticastCommand.create({
      userId: user._id,
      environmentId: user.environmentId,    // ✅ 从会话注入，不接受客户端
      organizationId: user.organizationId,  // ✅ 从会话注入，不接受客户端
      tenant: body.tenant,                  // 客户端传入，但受限于 env/org
      // ...
    })
  );
}
```

**关键安全设计**：
- `environmentId` 和 `organizationId` **绝不从请求 Body 中读取**
- 客户端传入的 `tenant` 仅作为业务租户标识，无隔离权限

---

## 4. 业务服务层：上下文传播与验证

### 4.1 ProcessTenant UseCase 分析

**租户处理流程** (`libs/application-generic/src/usecases/process-tenant/process-tenant.usecase.ts:19-101`)：
```typescript
@InstrumentUsecase()
public async execute(command: ProcessTenantCommand): Promise<TenantEntity | undefined> {
  const { environmentId, organizationId, userId, tenant } = command;
  let tenantEntity;

  try {
    tenantEntity = await this.getTenant(environmentId, organizationId, userId, tenant);
  } catch (e) {
    tenantEntity = null;  // 异常时静默失败
  }

  if (tenantEntity === null) {
    return undefined;     // 返回 undefined 供调用方处理
  }

  return tenantEntity;
}

private async getTenantByIdentifier({ identifier, _environmentId }) {
  return await this.tenantRepository.findOne({
    _environmentId,      // ✅ 强制环境过滤，类型系统保证
    identifier,
  });
}
```

**容错设计**：
- 租户解析失败时返回 `undefined` 而非抛出异常
- 避免因租户问题导致整个工作流中断
- 调用方需处理 `undefined` 场景

---

## 5. 触发链路：tenant 可选值降级隔离验证

### 5.1 TriggerEvent 执行链路

**代码位置**：`libs/application-generic/src/usecases/trigger-event/trigger-event.usecase.ts:126-209`

```typescript
private async getMappedCommand(command: TriggerEventCommand, workflowId: string) {
  return {
    ...command,
    tenant: this.mapTenant(command.tenant),      // 转换为 ITenantDefine | null
    actor: this.mapActor(command.actor),
    contextKeys: await this.resolveContextKeys(command, workflowId),
  };
}

private mapTenant(tenant: TriggerTenantContext): ITenantDefine | null {
  if (!tenant) return null;
  if (typeof tenant === 'string') {
    return { identifier: tenant };
  }
  return tenant;
}
```

### 5.2 ProcessTenant 降级路径分析

**关键代码片段** (`trigger-event.usecase.ts:126-153`)：
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
    // ⚠️  降级路径：仅记录日志，不中断流程
    await this.createWorkflowTrace({
      eventType: 'workflow_tenant_processing_failed',
      status: 'warning',
      rawData: { tenantIdentifier: mappedCommand.tenant.identifier },
    });
    Logger.warn(`Tenant ${mappedCommand.tenant.identifier} could not be processed`);
  }
  
  // ❗ 设计缺陷：tenantProcessed 结果未被使用
  // 后续流程继续使用原始的 mappedCommand.tenant
}

// 向下传递时使用的是 mappedCommand，而非 tenantProcessed
await this.triggerMulticast.execute(
  TriggerMulticastCommand.create({
    ...mappedCommand,  // 包含原始的 tenant
    actor: actorProcessed,
    template: storedWorkflow,
  })
);
```

### 5.3 降级路径隔离边界核验

| 阶段 | 风险点 | 隔离状态 | 说明 |
|------|--------|---------|------|
| **客户端传入** | 伪造 tenant.identifier | ✅ 安全 | identifier 仅作业务标识，无隔离权限 |
| **mapTenant 转换** | 注入伪造字段 | ✅ 安全 | ITenantDefine 类型仅允许 identifier/name/data |
| **ProcessTenant 失败** | 跳过验证使用原始数据 | ⚠️ 部分安全 | 原始 tenant 无隔离字段，但可能包含无效 data |
| **Job 创建** | 传递到下游 Worker | ✅ 安全 | JobEntity.tenant 类型为 ITenantDefine，无隔离字段 |
| **仓储查询** | 越过 EnforceEnvId | ✅ 安全 | 类型系统强制 _environmentId 必须存在 |

**隔离边界结论**：
- ✅ **横向越权防护**：`_organizationId` / `_environmentId` 始终来自 `UserSessionData`，无法通过 tenant 对象注入
- ⚠️ **数据一致性风险**：ProcessTenant 失败时，客户端传入的 `tenant.data` 可能包含无效/恶意数据，直接传递到下游模板渲染
- ✅ **类型安全防护**：TypeScript 类型系统确保 `ITenantDefine` 不会被扩展出隔离字段

### 5.4 Job 分发链路

**mapSubscribersToJobs** (`libs/application-generic/src/utils/subscribers.utils.ts:5-46`)：
```typescript
export function mapSubscribersToJobs(
  subscriberSource: SubscriberSourceEnum,
  subscribers: Array<{ subscriberId: string; topics?: SubscriberTopicPreference[] }>,
  command: BaseTriggerCommand
): IProcessSubscriberBulkJobDto[] {
  return subscribers.map((subscriber) => {
    const job: IProcessSubscriberBulkJobDto = {
      name: command.transactionId + subscriber.subscriberId,
      data: {
        environmentId: command.environmentId,   // ✅ 来自 Command，安全
        organizationId: command.organizationId, // ✅ 来自 Command，安全
        userId: command.userId,
        // ...
      },
      groupId: command.organizationId,
    };

    if (command.actor) {
      job.data.actor = command.actor;
    }
    if (command.tenant) {
      job.data.tenant = command.tenant;  // 使用原始 command.tenant
    }

    return job;
  });
}
```

**IProcessSubscriberDataDto** (`libs/application-generic/src/dtos/process-subscriber-job.dto.ts:15-34`)：
```typescript
export interface IProcessSubscriberDataDto {
  environmentId: string;      // 强制字段
  organizationId: string;     // 强制字段
  userId: string;
  // ...
  tenant?: ITenantDefine;     // 可选，仅业务字段
}
```

---

## 6. 数据访问层：编译时强制隔离

这是整个隔离体系中**最核心、最强壮**的防线。

### 6.1 强制类型约束体系

**定义** (`libs/dal/src/types/enforce.ts:1-6`)：
```typescript
export type EnforceOrgId = { _organizationId: OrganizationId };
export type EnforceEnvId = { _environmentId: EnvironmentId };
export type EnforceEnvOrOrgIds = EnforceEnvId | EnforceOrgId;
```

### 6.2 BaseRepository 泛型设计

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

### 6.3 仓储类声明示例

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

### 6.4 编译时保护效果

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

### 6.5 BaseRepositoryV2 增强

**V2 版本新增保护** (`libs/dal/src/repositories/base-repository-v2.ts:84-110`)：
- `select` 参数**必填**，避免 `SELECT *` 导致敏感字段泄露
- 返回类型自动推断为 `Pick<Entity, Keys>`，只返回请求字段
- `.lean()` 只读查询，避免 Mongoose 文档副作用
- 新增 `findById` 方法，同样强制执行 `T_Enforcement`

### 6.6 已识别绕过风险

| 方法 | 风险描述 | 影响级别 |
|------|---------|---------|
| `aggregate()` | 参数类型为 `any[]`，未应用 `T_Enforcement` 约束 | 高 |
| `bulkWrite()` | `bulkOperations` 类型为 `any`，完全绕过类型系统 | 高 |
| `OrganizationRepository` | 直接实现接口，未继承 BaseRepository | 中 |

---

## 7. 全局异常过滤链路

### 7.1 过滤器注册

**Bootstrap** (`apps/api/src/bootstrap.ts:156`)：
```typescript
app.useGlobalFilters(new AllExceptionsFilter(app.get(Logger), app.get(RequestLogRepository)));
```

全局过滤器在应用启动时注册，捕获所有未处理异常。

### 7.2 AllExceptionsFilter 核心流程

**代码位置**：`apps/api/src/exception-filter.ts:20-231`

```
异常进入 catch()
    │
    ├─▶ buildErrorResponse() ── 异常类型分派
    │     ├─▶ ThrottlerException → 429 限流
    │     ├─▶ ZodError → 400 Zod 验证失败
    │     ├─▶ CommandValidationException → 422 命令验证失败
    │     ├─▶ ValidationPipe 多错误 → 422 参数校验失败
    │     ├─▶ HttpException（非 500）→ 透传业务错误
    │     ├─▶ PayloadTooLargeError → 413 负载过大
    │     └─▶ 其他 → buildA5xxError() 脱敏处理
    │
    ├─▶ statusCode >= 500 ── logError() 记录原始异常
    │
    ├─▶ 响应扁平化 ── { ...errorDto.ctx, ...errorDto }
    │
    ├─▶ createAnalyticsLog() ── ClickHouse 审计日志
    │     └─▶ 按 organizationId/environmentId/userId 分片
    │
    └─▶ response.status(statusCode).json(finalResponse)
```

### 7.3 500 错误脱敏机制

**buildA5xxError** (`exception-filter.ts:157-164`)：
```typescript
private buildA5xxError(request: RequestWithReqId, exception: unknown) {
  const errorDto500 = this.buildErrorDto(
    request, 
    HttpStatus.INTERNAL_SERVER_ERROR, 
    ERROR_MSG_500  // "Internal server error, contact support and provide them with the errorId"
  );

  return {
    ...errorDto500,
    errorId: this.getUuid(exception),  // Sentry ID 或随机 UUID
  };
}

private getUuid(exception: unknown) {
  if (process.env.SENTRY_DSN) {
    try {
      return captureException(exception);  // 关联 Sentry 事件
    } catch (e) {
      return randomUUID();
    }
  } else {
    return randomUUID();
  }
}
```

**脱敏效果**：
- 客户端仅收到固定错误消息和 `errorId`
- 原始异常堆栈仅记录在服务端日志
- 支持通过 `errorId` 回溯 Sentry 事件

### 7.4 响应扁平化设计

**代码位置**：`exception-filter.ts:38`
```typescript
// This is for backwards compatibility for clients waiting for the context elements to appear flat
const finalResponse = { ...errorDto.ctx, ...errorDto };
```

**效果**：
```json
// 原始结构
{
  "statusCode": 400,
  "message": "Validation failed",
  "ctx": { "workflowId": "wf_123", "stepId": "step_456" }
}

// 扁平化后（ctx 字段提升到顶层）
{
  "statusCode": 400,
  "message": "Validation failed",
  "workflowId": "wf_123",
  "stepId": "step_456",
  "ctx": { "workflowId": "wf_123", "stepId": "step_456" }
}
```

**设计意图**：向后兼容旧客户端，同时保留结构化 ctx。

### 7.5 按组织与环境的日志上下文

**日志写入** (`exception-filter.ts:67-75`)：
```typescript
this.requestLogRepository
  .create(basicLog, {
    organizationId: user?.organizationId,
    environmentId: user?.environmentId,
    userId: user?._id,
  })
  .catch((err) => {
    this.logger.warn({ err }, 'Failed to log analytics to ClickHouse after retries');
  });
```

**buildLog 安全检查** (`apps/api/src/app/shared/utils/mappers.ts:24-59`)：
```typescript
export function buildLog(
  req: RequestWithReqId,
  statusCode: number,
  data: any,
  user: UserSessionData | null,
  duration: number = 0
): Omit<RequestLog, 'expires_at'> | null {
  // Skip logging when user data is incomplete to prevent orphaned log entries
  if (!user?._id || !user?.organizationId || !user?.environmentId || !user?.scheme) return null;
  // ...
  return {
    id: requestId,
    // ...
    user_id: user._id,
    organization_id: user.organizationId,
    environment_id: user.environmentId,
    auth_type: user.scheme,
    request_body: sanitizePayload(req.body),    // 敏感字段脱敏
    response_body: sanitizePayload(data),      // 敏感字段脱敏
  };
}
```

**Pino Logger 上下文配置** (`libs/application-generic/src/logging/index.ts:43-106`)：
```typescript
export function createNestLoggingModuleOptions(settings) {
  return {
    pinoHttp: {
      base: {
        pid: process.pid,
        serviceName: settings.serviceName,
        serviceVersion: settings.version,
        platform: configSet.platform,
        tenant: configSet.tenant,  // 部署级租户标识（OS/EE）
      },
      redact: {
        paths: redactFields,       // 敏感字段自动脱敏
        censor() { /* 不修改原始对象 */ }
      },
    },
  };
}
```

### 7.6 分层异常体系

| 层级 | 异常类型 | 触发场景 | HTTP 状态 | 处理方式 |
|------|---------|---------|----------|---------|
| 认证层 | `UnauthorizedException` | 无效 Token、API Key | 401 | 立即返回 |
| 权限层 | `ForbiddenException` | 权限不足 | 403 | 立即返回 |
| 限流层 | `ThrottlerException` | 超过速率限制 | 429 | 立即返回 |
| 业务层 | `BadRequestException` | 参数校验失败 | 400 | 返回错误详情 |
| 业务层 | `CommandValidationException` | 命令验证失败 | 422 | 返回约束详情 |
| 业务层 | `ServiceUnavailableException` | 组织 Kill Switch | 503 | 返回重试信息 |
| 数据层 | `DalException` | 数据库操作异常 | 500 | 包装后脱敏 |
| 未捕获 | 所有其他异常 | 未知错误 | 500 | 完全脱敏 + errorId |

---

## 8. 完整时序图：从入口到异常返回

### 8.1 正常流程时序

```
HTTP Request
     │
     ▼
1. RequestIdMiddleware ── 生成 req._nvRequestId
     │
     ▼
2. CommunityUserAuthGuard ── 解析 Authorization Header
     ├─▶ JwtStrategy.validate()
     │    ├─ 从 Token 提取 organizationId
     │    ├─ 从 Header 提取 environmentId
     │    └─ 验证 environment 归属 → 注入 req.user
     └─▶ ApiKeyStrategy.validateApiKey()
          ├─ API Key 查找用户
          └─ 检查 Kill Switch → 注入 req.user
     │
     ▼
3. Controller @UserSession() ── 提取 UserSessionData
     │
     ▼
4. TriggerEvent.execute()
     ├─▶ getMappedCommand() → mapTenant(tenant)
     ├─▶ processTenant.execute()
     │    ├─ 查询 TenantRepository（强制 _environmentId）
     │    └─ 成功返回 TenantEntity / 失败返回 undefined
     ├─▶ triggerMulticast.execute()
     │    └─▶ mapSubscribersToJobs() → 构建 Job（包含 environmentId/organizationId）
     └─▶ subscriberProcessQueueAddBulk() → 入队
     │
     ▼
5. ResponseInterceptor ── 包装成功响应
     │
     ▼
HTTP Response (201/200)
```

### 8.2 异常流程时序（ProcessTenant 降级）

```
HTTP Request (POST /v1/events/trigger)
     │
     ▼
1. 认证通过 → req.user = { organizationId: 'org_a', environmentId: 'env_b' }
     │
     ▼
2. Controller 构造 Command
     │  { ..., tenant: { identifier: 'tenant_x', data: {...} } }
     │
     ▼
3. TriggerEvent.execute()
     ├─▶ getMappedCommand() → tenant: { identifier: 'tenant_x', data: {...} }
     │
     ├─▶ processTenant.execute()
     │    ├─ tenantRepository.findOne({ _environmentId: 'env_b', identifier: 'tenant_x' })
     │    └─ ❌ 数据库超时 → catch → return undefined
     │
     ├─ 降级路径：仅记录 warning 日志，不中断
     │
     ├─▶ triggerMulticast.execute(Command)
     │    └─ Command.tenant = { identifier: 'tenant_x', data: {...} }  // 原始数据
     │
     └─▶ mapSubscribersToJobs()
          └─ job.data = {
               environmentId: 'env_b',      // ✅ 安全，来自 UserSession
               organizationId: 'org_a',     // ✅ 安全，来自 UserSession
               tenant: { identifier: 'tenant_x', data: {...} }  // 原始客户端数据
             }
     │
     ▼
4. 正常入队 → 返回 201 Created
```

### 8.3 异常流程时序（全局 500 错误）

```
HTTP Request
     │
     ▼
1. 认证通过 → req.user = { organizationId: 'org_a', environmentId: 'env_b' }
     │
     ▼
2. TriggerEvent.execute()
     └─ ❌ subscriberRepository.find() 抛出 MongoError
          │
          ▼
3. catch(e) → 记录 workflow trace → 重新 throw e
     │
     ▼
4. AllExceptionsFilter.catch(exception, host)
     ├─▶ buildErrorResponse(exception, request)
     │    └─ 非 HttpException → buildA5xxError()
     │         └─ {
              statusCode: 500,
              message: "Internal server error...",
              errorId: "sentry_abc123"
            }
     │
     ├─▶ logError() → PinoLogger.error({ err: exception, error: errorDto })
     │    └─ 日志包含完整堆栈 + organizationId + environmentId
     │
     ├─▶ 响应扁平化 → { ...ctx, ...errorDto }
     │
     ├─▶ createAnalyticsLog()
     │    ├─ buildLog() → sanitizePayload(request/response)
     │    └─ requestLogRepository.create({ organization_id: 'org_a', environment_id: 'env_b' })
     │
     └─▶ response.status(500).json(finalResponse)
          │
          ▼
HTTP Response (500)
{
  "statusCode": 500,
  "message": "Internal server error, contact support and provide them with the errorId",
  "errorId": "sentry_abc123"
}
```

---

## 9. 风险点与加固建议

### 9.1 已识别风险

| 风险点 | 位置 | 影响 | 建议 |
|--------|-----|------|------|
| ProcessTenant 结果未使用 | `trigger-event.usecase.ts:126-153` | 客户端传入的 tenant.data 可能包含恶意数据，直接传递到模板渲染 | 使用 `tenantProcessed` 替换 `mappedCommand.tenant`，确保使用数据库中存储的可信数据 |
| aggregate() 无类型约束 | `base-repository.ts:142-144` | 可构造任意聚合管道越权访问 | 增加 aggregate 方法的类型约束，或运行时检查 pipeline 包含隔离字段 |
| bulkWrite() 参数为 any | `base-repository.ts:470-472` | 完全绕过类型系统 | 添加强类型包装，每个操作强制包含隔离字段 |
| DalException 透传原始消息 | `dal.exception.ts:1` | 可能暴露 MongoDB 内部错误 | 映射数据库错误码为业务错误码，不暴露原始消息 |

### 9.2 ProcessTenant 加固方案

**当前实现（有缺陷）**：
```typescript
const tenantProcessed = await this.processTenant.execute(...);
if (!tenantProcessed) {
  // 记录日志，但不使用 tenantProcessed
}
// 继续使用 mappedCommand.tenant
```

**建议修复**：
```typescript
let finalTenant = mappedCommand.tenant;
const tenantProcessed = await this.processTenant.execute(...);
if (tenantProcessed) {
  // ✅ 使用验证后的租户数据
  finalTenant = {
    identifier: tenantProcessed.identifier,
    name: tenantProcessed.name,
    data: tenantProcessed.data,
  };
} else {
  // ⚠️  可选：租户验证失败时清空或使用最小集合
  finalTenant = { identifier: mappedCommand.tenant.identifier };
}
// 使用 finalTenant 向下传递
```

### 9.3 安全加固建议

1. **aggregate/bulkWrite 运行时检查**：在运行时验证查询条件包含 `_organizationId` 或 `_environmentId`
2. **数据库行级安全（RLS）**：MongoDB 4.0+ 支持基于角色的行级安全，作为最终防线
3. **审计日志增强**：记录所有跨租户数据访问的查询条件和返回记录数
4. **定期扫描**：使用 ESLint 规则检测绕过类型约束的 `as any` 使用
5. **ProcessTenant 硬失败选项**：增加配置项，允许在租户验证失败时硬失败而非降级

---

## 10. 关键文件索引

| 模块 | 文件路径 | 核心职责 |
|------|---------|---------|
| 认证策略 | `apps/api/src/app/auth/services/passport/jwt.strategy.ts` | JWT 认证与环境验证 |
| 认证策略 | `apps/api/src/app/auth/services/passport/apikey.strategy.ts` | API Key 认证与熔断 |
| 守卫 | `apps/api/src/app/auth/framework/community.user.auth.guard.ts` | 认证方案路由 |
| 会话装饰器 | `libs/application-generic/src/decorators/user-session.decorator.ts` | 用户会话安全提取 |
| Command 基类 | `libs/application-generic/src/commands/project.command.ts` | EnvironmentWithUserCommand / EnvironmentCommand 定义 |
| 基类根 | `libs/application-generic/src/commands/base.command.ts` | BaseCommand + CommandValidationException 定义 |
| 认证基类 | `libs/application-generic/src/commands/authenticated.command.ts` | AuthenticatedCommand / OrganizationCommand 定义 |
| 基础仓储 | `libs/dal/src/repositories/base-repository.ts` | V1 通用数据访问 |
| 基础仓储 | `libs/dal/src/repositories/base-repository-v2.ts` | V2 类型安全数据访问 |
| 强制类型 | `libs/dal/src/types/enforce.ts` | EnforceEnvId / EnforceOrgId 定义 |
| 租户处理 | `libs/application-generic/src/usecases/process-tenant/process-tenant.usecase.ts` | 租户上下文解析 |
| 事件触发 | `libs/application-generic/src/usecases/trigger-event/trigger-event.usecase.ts` | 事件流中的租户传播 |
| Job 映射 | `libs/application-generic/src/utils/subscribers.utils.ts` | mapSubscribersToJobs 实现 |
| 全局异常过滤 | `apps/api/src/exception-filter.ts` | AllExceptionsFilter 实现 |
| 启动配置 | `apps/api/src/bootstrap.ts` | 全局过滤器注册 |
| 日志构建 | `apps/api/src/app/shared/utils/mappers.ts` | buildLog 审计日志构建 |
| 日志配置 | `libs/application-generic/src/logging/index.ts` | Pino Logger 配置 |
| 会话类型 | `packages/shared/src/types/auth.ts` | UserSessionData 定义 |
| 租户类型 | `packages/shared/src/types/tenant.ts` | ITenantDefine 定义 |
