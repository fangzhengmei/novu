# Webhook 回调机制深度分析

本文档详细分析 Novu 平台向用户回调 webhook 的完整机制，**所有事件触发点均基于 `sendWebhookMessage.execute()` 真实调用点核实**，包括事件流转、签名生成、失败重试与去重机制，以及接收方如何验证身份与回溯历史投递。

---

## 1. 架构概述

Novu 存在 **两条独立的签名链路**，用于不同的通信场景：

```
┌───────────────────────────────────────────────────────────────────────────┐
│                              出站 Webhook (Svix)                          │
│  业务事件 → SendWebhookMessage → Svix API → svix-signature → 用户回调端点   │
│     (MESSAGE_SENT, WORKFLOW_DELETED 等 17 种事件)                          │
└───────────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────────────┐
│                         入站 Bridge (novu-signature)                      │
│  Novu 平台 → ExecuteFrameworkRequest → novu-signature → 用户 Bridge 端点   │
│     (工作流发现、执行、预览等内部通信)                                      │
└───────────────────────────────────────────────────────────────────────────┘
```

**核心代码位置**：
- `libs/application-generic/src/webhooks/` - 核心 webhook 逻辑
- `apps/api/src/app/outbound-webhooks/` - Webhook Portal API
- `packages/shared/src/webhooks/` - 事件类型定义
- `packages/framework/src/handler.ts` - Bridge 签名验证
- `libs/application-generic/src/utils/hmac.ts` - novu-signature 签名生成

---

## 2. 事件触发矩阵（基于真实调用点）

> **⚠️ 重要说明**：本矩阵完全基于 `sendWebhookMessage.execute()` 方法的真实调用点构建，每个事件都标注了精确的文件和行号作为代码证据。

### 2.1 事件类型定义

**文件**：`packages/shared/src/webhooks/webhook-event.enum.ts:1-33`

```typescript
export enum WebhookEventEnum {
  // Workflow
  WORKFLOW_CREATED = 'workflow.created',
  WORKFLOW_UPDATED = 'workflow.updated',
  WORKFLOW_DELETED = 'workflow.deleted',
  WORKFLOW_PUBLISHED = 'workflow.published',

  // Message
  MESSAGE_SENT = 'message.sent',
  MESSAGE_FAILED = 'message.failed',
  MESSAGE_DELIVERED = 'message.delivered',
  MESSAGE_SEEN = 'message.seen',
  MESSAGE_READ = 'message.read',
  MESSAGE_UNREAD = 'message.unread',
  MESSAGE_ARCHIVED = 'message.archived',
  MESSAGE_UNARCHIVED = 'message.unarchived',
  MESSAGE_SNOOZED = 'message.snoozed',
  MESSAGE_UNSNOOZED = 'message.unsnoozed',
  MESSAGE_DELETED = 'message.deleted',

  // Preference
  PREFERENCE_UPDATED = 'preference.updated',

  // Email Inbound
  EMAIL_RECEIVED = 'email.received',
}
```

### 2.2 完整触发矩阵

| # | 事件类型 | 触发场景 | 代码位置 | Payload 字段 |
|---|---------|---------|---------|-------------|
| **1** | `WORKFLOW_CREATED` | 创建新工作流 | `libs/application-generic/src/usecases/upsert-workflow/upsert-workflow.usecase.ts:125` | `object` |
| **2** | `WORKFLOW_UPDATED` | 更新工作流（v1 API） | `libs/application-generic/src/usecases/upsert-workflow/upsert-workflow.usecase.ts:114` | `object`, `previousObject` |
| **3** | `WORKFLOW_UPDATED` | 部分更新工作流（v2 API） | `apps/api/src/app/workflows-v2/usecases/patch-workflow/patch-workflow.usecase.ts:57` | `object`, `previousObject` |
| **4** | `WORKFLOW_DELETED` | 删除工作流（删除实体后触发） | `apps/api/src/app/workflows-v1/usecases/delete-workflow/delete-workflow.usecase.ts:50` | `object` |
| **5** | `WORKFLOW_PUBLISHED` | 工作流同步/发布到环境 | `apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts:150` | `object`, `previousObject` |
| **6** | `MESSAGE_SENT` | **Email** 发送成功 | `apps/worker/src/app/workflow/usecases/send-message/send-message-email.usecase.ts:487` | `object`, `providerResponseId` |
| **7** | `MESSAGE_FAILED` | **Email** 发送失败 | `apps/worker/src/app/workflow/usecases/send-message/send-message-email.usecase.ts:553` | `object`, `error` |
| **8** | `MESSAGE_SENT` | **SMS** 发送成功 | `apps/worker/src/app/workflow/usecases/send-message/send-message-sms.usecase.ts:356` | `object`, `providerResponseId` |
| **9** | `MESSAGE_FAILED` | **SMS** 发送失败 | `apps/worker/src/app/workflow/usecases/send-message/send-message-sms.usecase.ts:381` | `object`, `error` |
| **10** | `MESSAGE_SENT` | **Push** 发送成功 | `apps/worker/src/app/workflow/usecases/send-message/send-message-push.usecase.ts:644` | `object`, `providerResponseId`, `deviceToken` |
| **11** | `MESSAGE_FAILED` | **Push** 发送失败 | `apps/worker/src/app/workflow/usecases/send-message/send-message-push.usecase.ts:726` | `object`, `error.push.reason`, `error.push.deviceToken`, `error.message` |
| **12** | `MESSAGE_SENT` | **Chat** 发送成功 | `apps/worker/src/app/workflow/usecases/send-message/send-message-chat.usecase.ts:805` | `object`, `providerResponseId`, `channelData` |
| **13** | `MESSAGE_SENT` | ⚠️ **Chat 发送失败**（错误使用 MESSAGE_SENT） | `apps/worker/src/app/workflow/usecases/send-message/send-message-chat.usecase.ts:854` | `object`, `error.message` |
| **14** | `MESSAGE_SENT` | **In-App** 发送成功（第1个事件） | `apps/worker/src/app/workflow/usecases/send-message/send-message-in-app.usecase.ts:307` | `object`, `providerResponseId` |
| **15** | `MESSAGE_DELIVERED` | **In-App** 投递成功（第2个事件，仅此一处主动触发！） | `apps/worker/src/app/workflow/usecases/send-message/send-message-in-app.usecase.ts:319` | `object`, `providerResponseId` |
| **16** | `MESSAGE_SEEN` / `MESSAGE_READ` / `MESSAGE_UNREAD` | Widget 单个标记消息状态 | `apps/api/src/app/widgets/usecases/mark-message-as/mark-message-as.usecase.ts:184` | `object` |
| **17** | `MESSAGE_SEEN` / `MESSAGE_READ` / `MESSAGE_UNREAD` | Widget 批量标记消息状态 | `apps/api/src/app/widgets/usecases/mark-all-messages-as/mark-all-messages-as.usecase.ts:71` | `object` |
| **18** | `MESSAGE_SEEN` / `MESSAGE_READ` / `MESSAGE_UNREAD` | Widget 按 mark 标记状态 | `apps/api/src/app/widgets/usecases/mark-message-as-by-mark/mark-message-as-by-mark.usecase.ts:80` | `object` |
| **19** | `MESSAGE_SEEN` | Inbox 批量标记已看（100条一批） | `apps/api/src/app/inbox/usecases/mark-notifications-as-seen/mark-notifications-as-seen.usecase.ts:185` | `object` |
| **20** | `MESSAGE_READ` / `MESSAGE_UNREAD` | Inbox 批量标记已读/未读 | `apps/api/src/app/inbox/usecases/mark-many-notifications-as/mark-many-notifications-as.usecase.ts:149` | `object` |
| **21** | `MESSAGE_ARCHIVED` / `MESSAGE_UNARCHIVED` | Inbox 批量归档/取消归档 | `apps/api/src/app/inbox/usecases/mark-many-notifications-as/mark-many-notifications-as.usecase.ts:149` | `object` |
| **22** | `MESSAGE_SNOOZED` / `MESSAGE_UNSNOOZED` | Inbox 批量稍后/取消稍后 | `apps/api/src/app/inbox/usecases/mark-many-notifications-as/mark-many-notifications-as.usecase.ts:149` | `object` |
| **23** | `MESSAGE_READ` / `MESSAGE_UNREAD` | Inbox 按条件标记已读/未读 | `apps/api/src/app/inbox/usecases/update-all-notifications/update-all-notifications.usecase.ts:170` | `object` |
| **24** | `MESSAGE_ARCHIVED` / `MESSAGE_UNARCHIVED` | Inbox 按条件归档/取消归档 | `apps/api/src/app/inbox/usecases/update-all-notifications/update-all-notifications.usecase.ts:170` | `object` |
| **25** | `MESSAGE_SNOOZED` / `MESSAGE_UNSNOOZED` | Inbox 按条件稍后/取消稍后 | `apps/api/src/app/inbox/usecases/update-all-notifications/update-all-notifications.usecase.ts:170` | `object` |
| **26** | `MESSAGE_DELETED` | Inbox 批量删除消息 | `apps/api/src/app/inbox/usecases/delete-many-notifications/delete-many-notifications.usecase.ts:127` | `object` |
| **27** | `MESSAGE_DELETED` | Inbox 按条件删除消息 | `apps/api/src/app/inbox/usecases/delete-all-notifications/delete-all-notifications.usecase.ts:154` | `object` |
| **28** | `PREFERENCE_UPDATED` | 通知偏好更新 | `apps/api/src/app/inbox/usecases/update-preferences/update-preferences.usecase.ts:78` | `object`, `subscriberId` |
| **29** | `EMAIL_RECEIVED` | 入站邮件到达 | `libs/application-generic/src/usecases/inbound-domain-route-delivery/inbound-domain-route-delivery.usecase.ts:121` | `object` (domain, route, mail) |

### 2.3 各渠道 MESSAGE_SENT / MESSAGE_FAILED / MESSAGE_DELIVERED 对比表

| 渠道 | 发送成功 | 发送失败 | 主动触发 MESSAGE_DELIVERED |
|-----|---------|---------|--------------------------|
| **Email** | ✅ `MESSAGE_SENT`（L487） | ✅ `MESSAGE_FAILED`（L553） | ❌ 依赖第三方回调 |
| **SMS** | ✅ `MESSAGE_SENT`（L356） | ✅ `MESSAGE_FAILED`（L381） | ❌ 依赖第三方回调 |
| **Push** | ✅ `MESSAGE_SENT`（L644） | ✅ `MESSAGE_FAILED`（L726） | ❌ 依赖第三方回调 |
| **Chat** | ✅ `MESSAGE_SENT`（L805） | ⚠️ **`MESSAGE_SENT`**（L854，错误！） | ❌ 依赖第三方回调 |
| **In-App** | ✅ `MESSAGE_SENT`（L307） | ❌ 无失败 webhook | ✅ `MESSAGE_DELIVERED`（L319，仅此一处） |

> **⚠️ 关键发现**：
> 1. **Chat 发送失败事件类型不一致**：Chat 渠道发送失败时使用 `MESSAGE_SENT` 而非 `MESSAGE_FAILED`，与其他渠道不一致。接收方必须检查 `eventType === 'message.sent' && data.error` 才能判定 Chat 发送失败。
> 2. **MESSAGE_DELIVERED 仅 In-App 主动触发**：其他渠道的 `MESSAGE_DELIVERED` 依赖第三方 webhook 回调（通过 `apps/webhook` 服务），不由 `sendWebhookMessage` 主动触发。
> 3. **In-App 成功触发两个事件**：In-App 发送成功后会连续触发 `MESSAGE_SENT` 和 `MESSAGE_DELIVERED` 两个事件。

---

## 3. 各链路详细调用链与代码证据

### 3.1 消息状态变更（归档/稍后/删除）完整链路

#### 3.1.1 调用链总览

所有消息状态变更操作最终都会汇聚到 `MarkManyNotificationsAs` 或对应批量用例，由其统一触发 webhook。

```
单个操作入口
├─→ MarkNotificationAs.execute()               [mark-notification-as.usecase.ts:22]
│   └─→ MarkManyNotificationsAs.execute()      [mark-many-notifications-as.usecase.ts:42]
│       └─→ buildEventTypes()                  [mark-many-notifications-as.usecase.ts:81-97]
│       └─→ processWebhooksInBatches()         [mark-many-notifications-as.usecase.ts:99]
│           └─→ sendWebhookMessage.execute()   [mark-many-notifications-as.usecase.ts:149]

批量操作入口
├─→ MarkManyNotificationsAs.execute()          [直接调用]
│   └─→ ...（同上）

按条件操作入口
├─→ UpdateAllNotifications.execute()           [update-all-notifications.usecase.ts]
│   └─→ sendWebhookMessage.execute()           [update-all-notifications.usecase.ts:170]

删除操作入口
├─→ DeleteManyNotifications.execute()          [delete-many-notifications.usecase.ts]
│   └─→ sendWebhookMessage.execute()           [delete-many-notifications.usecase.ts:127]
└─→ DeleteAllNotifications.execute()           [delete-all-notifications.usecase.ts]
    └─→ sendWebhookMessage.execute()           [delete-all-notifications.usecase.ts:154]
```

#### 3.1.2 事件类型映射逻辑（精确代码）

**文件**：`apps/api/src/app/inbox/usecases/mark-many-notifications-as/mark-many-notifications-as.usecase.ts:81-97`

```typescript
const eventTypes: WebhookEventEnum[] = [];

// 已读/未读
if (command.read !== undefined) {
  const eventType = command.read ? WebhookEventEnum.MESSAGE_READ : WebhookEventEnum.MESSAGE_UNREAD;
  eventTypes.push(eventType);
}

// 归档/取消归档
if (command.archived !== undefined) {
  const eventType = command.archived ? WebhookEventEnum.MESSAGE_ARCHIVED : WebhookEventEnum.MESSAGE_UNARCHIVED;
  eventTypes.push(eventType);
}

// 稍后/取消稍后（注意：null 表示取消稍后）
if (command.snoozedUntil !== undefined) {
  // do not change to !== null, as null is a indication of unsnooze
  const eventType = command.snoozedUntil ? WebhookEventEnum.MESSAGE_SNOOZED : WebhookEventEnum.MESSAGE_UNSNOOZED;
  eventTypes.push(eventType);
}

await this.processWebhooksInBatches(eventTypes, updatedMessages, command, environment);
```

#### 3.1.3 批量处理策略

**文件**：`apps/api/src/app/inbox/usecases/mark-many-notifications-as/mark-many-notifications-as.usecase.ts:113-145`

```typescript
private async processWebhooksInBatches(
  eventTypes: WebhookEventEnum[],
  messages: MessageEntity[],
  command: MarkManyNotificationsAsCommand,
  environment: EnvironmentEntity
): Promise<void> {
  const BATCH_SIZE = 100;
  const messageChunks = this.chunkArray(messages, BATCH_SIZE);

  for (const messageChunk of messageChunks) {
    const webhookPromises: Promise<{ eventId: string } | undefined>[] = [];

    for (const eventType of eventTypes) {
      webhookPromises.push(...this.buildWebhookPromises(eventType, messageChunk, command, environment));
    }

    await Promise.all(webhookPromises);  // 每批并发发送
  }
}
```

### 3.2 Snooze/Unsnooze（稍后提醒）完整链路

#### 3.2.1 用户主动 Snooze

**文件**：`apps/api/src/app/inbox/usecases/snooze-notification/snooze-notification.usecase.ts:51-93`

```typescript
public async execute(command: SnoozeNotificationCommand): Promise<InboxNotificationDto> {
  // 1. 验证 snooze 时长
  // 2. 查找通知

  await this.messageRepository.withTransaction(async () => {
    scheduledJob = await this.createScheduledUnsnoozeJob(notification, snoozeDurationMs); // 创建定时唤醒 Job
    snoozedNotification = await this.markNotificationAsSnoozed(command); // → MarkNotificationAs → MarkManyNotificationsAs → MESSAGE_SNOOZED
    await this.enqueueJob(scheduledJob, snoozeDurationMs); // 加入延迟队列
  });

  return snoozedNotification;
}

// 调用 markNotificationAsSnoozed → 最终触发 MESSAGE_SNOOZED
private async markNotificationAsSnoozed(command: SnoozeNotificationCommand) {
  return this.markNotificationAs.execute(
    MarkNotificationAsCommand.create({
      environmentId: command.environmentId,
      organizationId: command.organizationId,
      subscriberId: command.subscriberId,
      notificationId: command.notificationId,
      snoozedUntil: command.snoozeUntil,  // 非 null 值 → MESSAGE_SNOOZED
      contextKeys: command.contextKeys,
    })
  );
}
```

#### 3.2.2 用户主动 Unsnooze

**文件**：`apps/api/src/app/inbox/usecases/unsnooze-notification/unsnooze-notification.usecase.ts:27-103`

```typescript
async execute(command: UnsnoozeNotificationCommand): Promise<InboxNotificationDto> {
  // 1. 查找 snoozed 通知

  await this.messageRepository.withTransaction(async () => {
    scheduledJob = await this.jobRepository.findOneAndDelete(...); // 删除定时 Job
    unsnoozedNotification = await this.markNotificationAs.execute(
      MarkNotificationAsCommand.create({
        ...
        snoozedUntil: null,  // null 值 → MESSAGE_UNSNOOZED
      })
    );
  });

  return unsnoozedNotification;
}
```

#### 3.2.3 定时自动唤醒（ProcessUnsnoozeJob）

**注意**：`ProcessUnsnoozeJob` 用例本身不直接调用 `sendWebhookMessage.execute()`，而是通过更新消息状态触发后续流程。

### 3.3 工作流删除链路

**文件**：`apps/api/src/app/workflows-v1/usecases/delete-workflow/delete-workflow.usecase.ts:39-58`

```typescript
async execute(command: DeleteWorkflowCommand): Promise<void> {
  const workflowEntity = await this.getWorkflowByIdsUseCase.execute(...);

  // 先删除所有关联实体
  await this.deleteRelatedEntities(command, workflowEntity);

  // 删除完成后触发 webhook
  await this.sendWebhookMessage.execute({
    eventType: WebhookEventEnum.WORKFLOW_DELETED,
    objectType: WebhookObjectTypeEnum.WORKFLOW,
    payload: { object: workflowEntity },  // 包含已删除工作流的完整信息
    organizationId: command.organizationId,
    environmentId: command.environmentId,
  });
}
```

### 3.4 工作流发布/同步链路

**文件**：`apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts:135-160`

```typescript
// 执行工作流同步（upsert 到目标环境）
const upsertedWorkflow = await this.upsertWorkflowUseCase.execute(
  UpsertWorkflowCommand.create(...)
);

// 更新源工作流的发布信息
await this.notificationTemplateRepository.updatePublishFields(
  sourceWorkflow._id,
  command.user.environmentId,
  command.user._id,
  command.session
);

// 触发 WORKFLOW_PUBLISHED 事件
if (this.sendWebhookMessage) {
  await this.sendWebhookMessage.execute({
    eventType: WebhookEventEnum.WORKFLOW_PUBLISHED,
    objectType: WebhookObjectTypeEnum.WORKFLOW,
    payload: {
      object: upsertedWorkflow,        // 目标环境的新工作流状态
      previousObject: sourceWorkflow,  // 源环境的原始工作流状态
    },
    organizationId: command.user.organizationId,
    environmentId: command.user.environmentId,  // 注意：是源环境 ID
  });
}
```

### 3.5 入站邮件链路

**文件**：`libs/application-generic/src/usecases/inbound-domain-route-delivery/inbound-domain-route-delivery.usecase.ts:112-132`

```typescript
async deliverToWebhook(params: {
  environmentId: string;
  organizationId: string;
  domain: RoutableDomain;
  route: DomainRouteEntity;
  mail: InboundDomainRouteMailInput;
}): Promise<{ latencyMs: number; skipped: boolean }> {
  const started = Date.now();
  const payload = this.buildDomainRouteWebhookPayload(params.domain, params.route, params.mail);

  const result = await this.sendWebhookMessage.execute({
    environmentId: params.environmentId,
    organizationId: params.organizationId,
    eventType: WebhookEventEnum.EMAIL_RECEIVED,
    objectType: WebhookObjectTypeEnum.EMAIL_INBOUND,
    payload: { object: payload as unknown as Record<string, unknown> },
  });

  return {
    latencyMs: Date.now() - started,
    skipped: result === undefined,
  };
}
```

**入站邮件 Payload 结构**：
```typescript
{
  domain: { id, name, data },       // 域名配置
  route: { address, data },         // 路由配置
  mail: {                           // 邮件内容
    from, to, subject, html, text,
    headers, attachments, messageId,
    inReplyTo, references, date, cc
  }
}
```

### 3.6 Chat 发送失败不一致问题详细分析

**文件**：`apps/worker/src/app/workflow/usecases/send-message/send-message-chat.usecase.ts:825-873`

```typescript
private async handleMessageSendError(
  command: SendMessageChatCommand,
  message: MessageEntity,
  error: Error,
  channelData: ChatProviderData,
  redactedChannelData: Record<string, unknown>
): Promise<{ status: SendMessageStatus; errorMessage: DetailEnum }> {
  // ... 更新消息状态为 FAILED ...

  // ⚠️ 错误：使用 MESSAGE_SENT 而非 MESSAGE_FAILED
  await this.sendWebhookMessage.execute({
    eventType: WebhookEventEnum.MESSAGE_SENT,  // ❌ 应该是 MESSAGE_FAILED
    objectType: WebhookObjectTypeEnum.MESSAGE,
    payload: {
      object: messageWebhookMapper(message, command.subscriberId, {
        channelData: redactedChannelData,
      }),
      error: {
        message: this.getErrorMessage(error) || 'Error while sending chat with provider',
      },
    },
    organizationId: command.organizationId,
    environmentId: command.environmentId,
  });

  return {
    status: SendMessageStatus.FAILED,  // 返回状态是 FAILED，但事件类型是 SENT
    errorMessage: DetailEnum.PROVIDER_ERROR,
  };
}
```

**接收方兼容处理**：
```typescript
if (event.type === 'message.sent') {
  if (event.data.error) {
    // Chat 发送失败（特殊情况）
    handleChatFailure(event.data);
  } else {
    // 发送成功（所有渠道）
    handleSuccess(event.data);
  }
} else if (event.type === 'message.failed') {
  // Email/SMS/Push 发送失败
  handleOtherFailure(event.data);
}
```

---

## 4. 事件投递流程

### 4.1 SendWebhookMessage 核心逻辑

**文件**：`libs/application-generic/src/webhooks/usecases/send-webhook-message/send-webhook-message.usecase.ts:20-80`

```typescript
async execute(command: SendWebhookMessageCommand): Promise<{ eventId: string } | undefined> {
  // 1. 检查 Svix 客户端是否可用
  if (!this.svix) return;

  // 2. 获取环境配置，检查 webhookAppId（优先使用传入的 environment，减少 DB 查询）
  const environment = command.environment || await this.environmentRepository.findOne({ _id: command.environmentId });
  const appId = environment.webhookAppId;
  if (!appId) return;

  // 3. 生成唯一事件 ID
  const eventId = `evt_${generateObjectId()}`;

  // 4. 构造标准 payload
  const webhookPayload: WrapperDto<any> = {
    id: eventId,
    type: command.eventType,
    object: command.objectType,
    data: command.payload,
    timestamp: new Date().toISOString(),
    environmentId: environment.identifier,
  };

  // 5. 通过 Svix 发送
  await this.svix.message.create(appId, {
    eventType: command.eventType,
    eventId,
    payload: webhookPayload,
  });

  return { eventId };
}
```

### 4.2 可靠性语义与错误分支深度分析

**文件**：`libs/application-generic/src/webhooks/usecases/send-webhook-message/send-webhook-message.usecase.ts:20-80`

`sendWebhookMessage.execute()` 方法有 **5 个执行分支**，每个分支的返回值、日志行为和对调用方的影响各不相同。

#### 4.2.1 完整执行流程图

```
                              ┌─────────────────────────────┐
                              │ sendWebhookMessage.execute() │
                              └─────────────┬───────────────┘
                                            │
                        ┌───────────────────┴───────────────────┐
                        │ 分支 1: 检查 SVIX_CLIENT 是否存在     │
                        └───────────────────┬───────────────────┘
                                            │
                        ┌───────────────────┴───────────────────┐
                        │ 分支 2: 检查 Environment 是否存在     │
                        └───────────────────┬───────────────────┘
                                            │
                        ┌───────────────────┴───────────────────┐
                        │ 分支 3: 检查 webhookAppId 是否存在     │
                        └───────────────────┬───────────────────┘
                                            │
                        ┌───────────────────┴───────────────────┐
                        │ 分支 4: 调用 Svix API 创建消息        │
                        └───────────────────┬───────────────────┘
                                            │
                        ┌───────────────────┴───────────────────┐
                        │ 分支 5: 成功返回 { eventId }           │
                        └───────────────────────────────────────┘
```

#### 4.2.2 各分支详细分析

| 分支 | 触发条件 | 返回值 | 日志级别 | 日志内容 | 是否抛出 | 调用方可见性 |
|-----|---------|--------|---------|---------|---------|-------------|
| **分支 1** | `!this.svix`（SVIX_CLIENT 未注入） | `undefined` | `debug` | `Outbound webhook client not available – webhooks are disabled for this instance.` | ❌ 不抛出 | ✅ 静默跳过 |
| **分支 2** | `!environment`（环境不存在） | 抛出 `Error` | - | - | ✅ **抛出异常** | ⚠️ **错误暴露，可能中断业务流程** |
| **分支 3** | `!appId`（webhookAppId 为空） | `undefined` | `debug` | `Webhook app ID not found for environment xxx` | ❌ 不抛出 | ✅ 静默跳过 |
| **分支 4** | `svix.message.create()` 抛出异常 | `undefined` | `error` | `Failed to send webhook xxx. Error: xxx, Event ID: xxx` | ❌ 不抛出 | ✅ 静默跳过（仅日志） |
| **分支 5** | Svix API 调用成功 | `{ eventId: string }` | `debug` | `Attempting to send webhook...` + `Successfully sent webhook...` | ❌ 不抛出 | ✅ 可通过返回值确认 |

##### 分支 1: 无 SVIX_CLIENT（L22-26）

**代码**：
```typescript
if (!this.svix) {
  this.logger.debug('Outbound webhook client not available – webhooks are disabled for this instance.');
  return;
}
```

**说明**：
- `SVIX_CLIENT` 用 `@Optional()` 装饰器注入，未配置 `SVIX_API_KEY` 环境变量时返回 `null`
- 仅记录 `debug` 级日志，生产环境默认日志级别（`info`）下**不可见**
- 静默返回 `undefined`，调用方无感知

**典型场景**：社区版部署、开发环境未配置 Svix

##### 分支 2: 环境不存在（L37-39）

**代码**：
```typescript
if (!environment) {
  throw new Error(`Environment not found for id ${command.environmentId}`);
}
```

**说明**：
- **唯一会向外抛出异常的分支**
- 无内部日志，异常直接向上传播
- 调用方如果没有 `try-catch`，会**导致整个业务流程中断**

**风险场景**：
- 并发删除环境导致的竞态条件
- 数据库不一致导致环境 ID 无效
- 手动传入错误的 `environmentId`

##### 分支 3: 缺少 webhookAppId（L43-47）

**代码**：
```typescript
if (!appId) {
  this.logger.debug(`Webhook app ID not found for environment ${command.environmentId}`);
  return;
}
```

**说明**：
- 环境存在但未初始化 webhook 配置（未调用过 `/v2/outbound-webhooks/portal/token`）
- 仅记录 `debug` 级日志，生产环境不可见
- 静默返回 `undefined`，调用方无感知

**典型场景**：新环境尚未配置 webhook

##### 分支 4: Svix API 调用异常（L74-78）

**代码**：
```typescript
catch (error: any) {
  this.logger.error(
    `Failed to send webhook ${command.eventType} for application ${appId}. Error: ${error.message}, Event ID: ${eventId}`,
    error.stack
  );
}
```

**说明**：
- Svix API 调用失败（网络问题、权限问题、配额超限等）
- 记录 `error` 级日志，包含完整 `error.stack`，生产环境可见
- catch 块没有 `return` 语句，函数隐式返回 `undefined`
- 不向外抛出异常，调用方无感知

**典型场景**：
- Svix 服务不可用
- API 配额耗尽
- 签名密钥配置错误
- 网络分区

##### 分支 5: 成功路径（L65-73）

**代码**：
```typescript
await this.svix.message.create(appId, {
  eventType: command.eventType,
  eventId,
  payload: webhookPayload,
});

this.logger.debug(`Successfully sent webhook ${command.eventType}. Event ID: ${eventId}`);
return { eventId };
```

**说明**：
- 返回 `{ eventId: string }`，调用方可通过返回值确认发送成功
- 记录两条 `debug` 级日志（尝试发送 + 发送成功）

#### 4.2.3 各调用方错误处理模式分析

基于 29 个真实调用点的分析，调用方分为 **4 种错误处理模式**：

| 模式 | 调用方数量 | try-catch | 检查返回值 | 依赖存在性检查 | 行为特征 |
|-----|-----------|-----------|-----------|---------------|---------|
| **A: Fire-and-Forget** | 25/29 | ❌ 无 | ❌ 不检查 | ❌ 不检查 | 环境不存在时中断流程，其他情况静默 |
| **B: 有 try-catch** | 1/29 | ✅ 有 | ❌ 不检查 | ❌ 不检查 | 所有异常都捕获，记录额外日志后继续 |
| **C: 检查返回值** | 1/29 | ❌ 无 | ✅ 检查 `result === undefined` | ❌ 不检查 | 通过返回值区分是否跳过 |
| **D: 检查依赖存在** | 1/29 | ❌ 无 | ❌ 不检查 | ✅ `if (this.sendWebhookMessage)` | 依赖不存在时完全不调用 |
| **E: 传入 environment** | 1/29 | ❌ 无 | ❌ 不检查 | ❌ 不检查 | 避免环境不存在异常 |

##### 模式 A: Fire-and-Forget（绝大多数调用方）

**代表调用方**：
- Email 发送成功/失败: `send-message-email.usecase.ts:487, 553`
- SMS 发送成功/失败: `send-message-sms.usecase.ts:356, 381`
- Push 发送成功: `send-message-push.usecase.ts:644`
- In-App 发送成功/送达: `send-message-in-app.usecase.ts:307, 319`
- Chat 发送成功/失败: `send-message-chat.usecase.ts:805, 854`
- 工作流删除: `delete-workflow.usecase.ts:50`
- 工作流更新/创建: `patch-workflow.usecase.ts:57`, `upsert-workflow.usecase.ts:114, 125`
- 偏好更新: `update-preferences.usecase.ts:78`
- Widget 各种标记: `mark-message-as.usecase.ts:184` 等
- 消息批量状态变更: `mark-many-notifications-as.usecase.ts:149`

**代码特征**：
```typescript
await this.sendWebhookMessage.execute({ ... });
// 无 try-catch，不检查返回值，继续执行
```

**风险**：
- 如果遇到「分支 2：环境不存在」，会直接抛出异常，**中断整个业务流程**
- 其他分支（无 SVIX_CLIENT、无 webhookAppId、Svix 异常）均静默跳过，调用方无感知

##### 模式 B: 有 try-catch（Push 发送失败路径）

**调用方**：`send-message-push.usecase.ts:726`

**代码**：
```typescript
try {
  await this.sendWebhookMessage.execute({ ... });
} catch (err) {
  Logger.error(
    { jobId: command.jobId },
    `Error sending webhook message for jobId ${command.jobId} ${err.message || err.toString()}`,
    LOG_CONTEXT
  );
}
// 继续执行，返回 { success: false, error: e }
```

**行为**：
- 捕获所有异常（包括分支 2 的环境不存在）
- 记录额外的错误日志（包含 `jobId`）
- 不向外抛出，继续执行
- **Push 发送失败是唯一有 try-catch 保护的调用点**

##### 模式 C: 检查返回值（入站邮件）

**调用方**：`inbound-domain-route-delivery.usecase.ts:121`

**代码**：
```typescript
const result = await this.sendWebhookMessage.execute({ ... });

return {
  latencyMs: Date.now() - started,
  skipped: result === undefined,  // 明确标识是否跳过
};
```

**行为**：
- 保存返回值，通过 `result === undefined` 区分是否成功发送
- 不捕获异常，环境不存在时仍会中断
- **入站邮件是唯一检查返回值的调用方**

##### 模式 D: 检查依赖存在性（工作流发布）

**调用方**：`sync-to-environment.usecase.ts:149`

**代码**：
```typescript
if (this.sendWebhookMessage) {  // 先检查依赖是否注入
  await this.sendWebhookMessage.execute({ ... });
}
```

**行为**：
- 先检查 `this.sendWebhookMessage` 是否存在
- 不存在时完全不调用，避免潜在的运行时错误
- 不捕获异常，环境不存在时仍会中断

##### 模式 E: 传入 environment 对象（批量标记）

**调用方**：`mark-many-notifications-as.usecase.ts:149`

**代码**：
```typescript
private sendWebhookEvents(...) {
  return updatedMessages.map((message) =>
    this.sendWebhookMessage.execute({
      ...,
      environment: environment,  // 直接传入已查询的 environment 对象
    })
  );
}
```

**行为**：
- 通过 `command.environment` 传入已查询的环境对象
- 避免 `sendWebhookMessage` 内部再次查询数据库
- **消除了分支 2（环境不存在）的可能性**，因为环境已在上游验证存在

#### 4.2.4 关键风险点总结

| 风险 | 影响 | 涉及调用方 | 建议 |
|-----|------|-----------|------|
| **环境不存在异常未被捕获** | 可能导致业务流程意外中断 | 模式 A（25 个调用方） | 为关键路径添加 try-catch，或统一在调用前验证环境 |
| **debug 级日志生产环境不可见** | 无 SVIX_CLIENT、无 webhookAppId 时调用方无法感知 | 分支 1、3 | 考虑提升为 `info` 级，或添加 metrics 指标 |
| **Svix 失败仅记录日志不通知** | 业务方不知道 webhook 发送失败 | 分支 4 | 结合 Svix Portal 告警，或实现失败回调 |
| **批量操作中单个失败不影响其他** | 部分消息的 webhook 可能未发送但不报错 | 模式 A 批量场景 | 记录每个事件的 eventId，支持事后审计 |

### 4.3 Payload 结构

**文件**：`libs/application-generic/src/webhooks/dtos/webhook-payload.dto.ts:4-40`

```typescript
export class WrapperDto<T> {
  id: string;                    // 事件唯一标识 evt_xxx
  type: WebhookEventEnum;        // 事件类型
  object: WebhookObjectTypeEnum; // 对象类型
  data: T;                       // 事件数据
  timestamp: string;             // ISO 时间戳
  environmentId: string;         // 环境标识
}
```

### 4.3 消息 Mapper

**文件**：`libs/application-generic/src/webhooks/mappers/message.mapper.ts:5-79`

`messageWebhookMapper` 函数负责将内部 `MessageEntity` 映射为对外暴露的 `MessageWebhookResponseDto`，包含字段：
- 基础字段：`_id`, `_templateId`, `_environmentId`, `_organizationId`, `_notificationId`
- 状态字段：`status`, `seen`, `read`, `archived`, `snoozedUntil`
- 时间字段：`createdAt`, `updatedAt`, `deliveredAt`, `lastSeenDate`, `lastReadDate`
- 其他：`subscriberId`, `transactionId`, `channel`, `providerId`, `errorId`, `errorText`, `contextKeys`

---

## 5. 两条签名链路深度对比

### 5.1 链路概览

| 维度 | Svix 出站签名 | Framework novu-signature 验签 |
|-----|--------------|-----------------------------|
| **使用场景** | Novu 向用户回调业务事件（17 种 webhook 事件） | Novu 调用用户 Bridge 端点（工作流发现、执行、预览） |
| **签名头部** | `svix-signature` | `novu-signature` |
| **签名格式** | `t=<timestamp>,v1=<signature>` | `t=<unix-ms>,v1=<hex-hmac>` |
| **签名算法** | HMAC-SHA256 | HMAC-SHA256 |
| **签名密钥** | Svix 为每个 endpoint 生成的 Signing Secret | 用户环境的 `secretKey`（Novu 平台存储，加密保存） |
| **签名生成方** | Svix 服务自动生成 | Novu 平台（`buildNovuSignatureHeader`） |
| **签名验证方** | 用户代码（使用 Svix SDK） | Novu Framework Handler（用户 Bridge 端） |
| **时间戳容忍度** | Svix 默认 5 分钟 | Novu Framework 固定 5 分钟 |
| **重放防护** | ✅ 时间戳 + 事件 ID | ✅ 时间戳 |
| **恒时比较** | ✅ Svix SDK 实现 | ✅ `timingSafeEqual` 手动实现 |
| **开关控制** | `SVIX_API_KEY` 环境变量 | `strictAuthentication` 开关 |

### 5.2 链路一：Svix 出站签名（用户接收 Novu 事件）

**签名生成**：由 Svix 服务在投递时自动生成，使用用户在 Svix Portal 中配置的 Signing Secret。

**签名验证**（用户端实现）：
```javascript
import { Webhook } from 'svix';

const webhook = new Webhook(process.env.SVIX_WEBHOOK_SECRET);

app.post('/webhook', (req, res) => {
  const payload = JSON.stringify(req.body);
  const headers = req.headers;

  try {
    const verified = webhook.verify(payload, headers);
    res.status(200).send('OK');
  } catch (err) {
    res.status(400).send('Invalid signature');
  }
});
```

**Svix 客户端初始化**：`libs/application-generic/src/webhooks/services/svix-provider.service.ts:1-18`

```typescript
export const SvixProviderService: Provider<SvixClient> = {
  provide: 'SVIX_CLIENT',
  useFactory: (): SvixClient => {
    const apiKey = process.env.SVIX_API_KEY;
    if (!apiKey) return null;  // 未配置则禁用 webhook 功能
    return new Svix(apiKey);
  },
};
```

### 5.3 链路二：novu-signature 验签（Novu 调用用户 Bridge）

#### 5.3.1 签名生成（Novu 平台端）

**文件**：`libs/application-generic/src/utils/hmac.ts:6-12`

```typescript
export function buildNovuSignatureHeader(secretKey: string, payload: unknown): string {
  const timestamp = Date.now();
  const publicKey = `${timestamp}.${JSON.stringify(payload)}`;
  const hmac = createHmac('sha256', secretKey).update(publicKey).digest('hex');

  return `t=${timestamp},v1=${hmac}`;
}
```

**调用点**：`libs/application-generic/src/usecases/execute-bridge-request/execute-framework-request.usecase.ts:140-148`

```typescript
private async buildRequestHeaders(command: ExecuteBridgeRequestCommand) {
  const novuSignatureHeader = await this.buildRequestSignature(command);

  return {
    [HttpRequestHeaderKeysEnum.BYPASS_TUNNEL_REMINDER]: 'true',
    [HttpRequestHeaderKeysEnum.CONTENT_TYPE]: 'application/json',
    [HttpHeaderKeysEnum.NOVU_SIGNATURE]: novuSignatureHeader,  // 注入签名
  };
}

private async buildRequestSignature(command: ExecuteBridgeRequestCommand) {
  const secretKey = await this.getDecryptedSecretKey.execute(
    GetDecryptedSecretKeyCommand.create({ environmentId: command.environmentId })
  );

  return buildNovuSignatureHeader(secretKey, command.event || {});
}
```

#### 5.3.2 签名验证（用户 Bridge 端）

**文件**：`packages/framework/src/handler.ts:371-396`

```typescript
private async validateHmac(payload: unknown, hmacHeader: string | null): Promise<void> {
  // 1. 检查是否启用 HMAC（受 strictAuthentication 控制）
  if (!this.hmacEnabled) return;

  // 2. 检查签名头部是否存在
  if (!hmacHeader) throw new SignatureNotFoundError();

  // 3. 检查签名密钥是否配置
  if (!this.client.secretKey) throw new SigningKeyNotFoundError();

  // 4. 解析签名头部
  const parsed = parseSignatureHeader(hmacHeader);
  if (!parsed.v1 || parsed.t === undefined) throw new SignatureInvalidError();

  // 5. 检查时间戳（防重放攻击，5分钟容忍度）
  const now = Date.now();
  if (parsed.t < now - SIGNATURE_TIMESTAMP_TOLERANCE || parsed.t > now + SIGNATURE_TIMESTAMP_TOLERANCE) {
    throw new SignatureExpiredError();
  }

  // 6. 计算本地签名（使用 Web Crypto API，跨平台兼容）
  const localHash = await createHmacSubtle(
    this.client.secretKey,
    `${parsed.t}.${JSON.stringify(payload)}`
  );

  // 7. 恒定时间比较（防时序攻击）
  if (!timingSafeEqual(localHash, parsed.v1)) {
    throw new SignatureMismatchError();
  }
}
```

**时间戳容忍度**：`packages/framework/src/constants/api.constants.ts:6-7`
```typescript
export const SIGNATURE_TIMESTAMP_TOLERANCE_MINUTES = 5;
export const SIGNATURE_TIMESTAMP_TOLERANCE = SIGNATURE_TIMESTAMP_TOLERANCE_MINUTES * 60 * 1000; // 5分钟
```

#### 5.3.3 核心加密工具

**文件**：`packages/framework/src/utils/crypto.utils.ts:11-60`

##### HMAC 生成（跨平台兼容 Web Crypto API）
```typescript
export const createHmacSubtle = async (secretKey: string, data: string): Promise<string> => {
  const encoder = new TextEncoder();
  const keyData = encoder.encode(secretKey);
  const dataBuffer = encoder.encode(data);

  const cryptoKey = await crypto.subtle.importKey(
    'raw', keyData,
    { name: 'HMAC', hash: { name: 'SHA-256' } },
    false, ['sign']
  );

  const signature = await crypto.subtle.sign('HMAC', cryptoKey, dataBuffer);
  return Array.from(new Uint8Array(signature))
    .map((byte) => byte.toString(16).padStart(2, '0'))
    .join('');
};
```

##### 恒定时间比较（防时序攻击）
```typescript
export const timingSafeEqual = (a: string, b: string): boolean => {
  if (typeof a !== 'string' || typeof b !== 'string') return false;
  if (a.length !== b.length) return false;

  let mismatch = 0;
  for (let i = 0; i < a.length; i += 1) {
    mismatch |= a.charCodeAt(i) ^ b.charCodeAt(i);
  }
  return mismatch === 0;
};
```

##### 签名头部解析
**文件**：`packages/framework/src/handler.ts:415-439`

```typescript
function parseSignatureHeader(header: string): ParsedSignatureHeader {
  const fields: Record<string, string> = {};
  for (const rawPart of header.split(',')) {
    const part = rawPart.trim();
    if (!part) continue;
    const eqIdx = part.indexOf('=');
    if (eqIdx <= 0) continue;
    const key = part.slice(0, eqIdx);
    const value = part.slice(eqIdx + 1);
    if (key && value && !(key in fields)) fields[key] = value;
  }
  return { t: Number.isFinite(Number(fields.t)) ? Number(fields.t) : undefined, v1: fields.v1 };
}
```

#### 5.3.4 签名错误类型

**文件**：`packages/framework/src/errors/signature.errors.ts:4-57`

| 错误类型 | 场景 | HTTP 状态码 |
|---------|------|------------|
| `SignatureNotFoundError` | 请求缺少 `novu-signature` 头部 | 401 |
| `SignatureInvalidError` | 签名格式不正确 | 401 |
| `SignatureExpiredError` | 签名时间戳超出 5 分钟容忍范围 | 401 |
| `SignatureMismatchError` | 签名验证失败 | 401 |
| `SigningKeyNotFoundError` | 服务端未配置签名密钥 | 500 |
| `SignatureVersionInvalidError` | 签名版本不支持 | 401 |

---

## 6. strictAuthentication 开关深度分析

### 6.1 开关定义

**文件**：`packages/framework/src/types/config.types.ts:15-25`

```typescript
export type ClientOptions = {
  /**
   * Explicitly use HMAC signature verification.
   * Setting this to `false` will enable Novu to communicate with your Bridge API
   * without requiring a valid HMAC signature.
   * This is useful for local development and testing.
   *
   * In production you must specify an `secretKey` and set this to `true`.
   *
   * Defaults to true.
   */
  strictAuthentication?: boolean;
};
```

### 6.2 开关优先级与默认值

**文件**：`packages/framework/src/client.ts:53-95`

```typescript
function isRuntimeInDevelopment() {
  return ['development', undefined, 'dev'].includes(process.env.NODE_ENV);
}

private buildOptions(providedOptions?: ClientOptions) {
  const builtConfiguration: Required<ClientOptions> = {
    apiUrl: resolveApiUrl(providedOptions?.apiUrl),
    secretKey: resolveSecretKey(providedOptions?.secretKey),
    strictAuthentication: !isRuntimeInDevelopment(),  // 默认：开发环境 false，生产 true
    verbose: isRuntimeInDevelopment(),
  };

  // 优先级 1：显式传入的参数
  if (providedOptions?.strictAuthentication !== undefined) {
    builtConfiguration.strictAuthentication = providedOptions.strictAuthentication;
  }
  // 优先级 2：环境变量 NOVU_STRICT_AUTHENTICATION_ENABLED
  else if (process.env.NOVU_STRICT_AUTHENTICATION_ENABLED !== undefined) {
    builtConfiguration.strictAuthentication = process.env.NOVU_STRICT_AUTHENTICATION_ENABLED === 'true';
  }
  // 优先级 3：默认值（!isRuntimeInDevelopment()）

  return builtConfiguration;
}
```

**优先级总结**：
1. **最高**：显式传入 `strictAuthentication` 参数
2. **其次**：环境变量 `NOVU_STRICT_AUTHENTICATION_ENABLED`
3. **最低**：默认值（`NODE_ENV` 为 `development`/`dev`/`undefined` 时为 `false`，否则为 `true`）

### 6.3 开关影响范围

**文件**：`packages/framework/src/handler.ts:80-88`

```typescript
constructor(options: INovuRequestHandlerOptions<Input, Output>) {
  this.handler = options.handler;
  this.client = options.client ? options.client : new Client();
  this.workflows = options.workflows || [];
  this.agents = options.agents || [];
  this.http = initApiClient(this.client.secretKey, this.client.apiUrl);
  this.frameworkName = options.frameworkName;
  this.hmacEnabled = this.client.strictAuthentication;  // ⚠️ 直接关联
  this.client.addAgents(this.agents);
}
```

**当 `strictAuthentication = false` 时**：
- `this.hmacEnabled = false`
- `validateHmac()` 方法直接 `return`，跳过所有签名验证
- 即使请求不带 `novu-signature` 头部也能通过
- **仅用于开发/测试环境**

**当 `strictAuthentication = true` 时**：
- `this.hmacEnabled = true`
- 所有请求必须携带有效的 `novu-signature` 头部
- 缺少头部、签名过期、签名不匹配都会抛出相应错误

### 6.4 生产环境强制启用

**文件**：`apps/api/src/app/environments-v1/novu-bridge-client.ts:127-132`

Novu 托管的 Bridge 端点强制启用严格认证：

```typescript
const novuRequestHandler = new NovuRequestHandler({
  frameworkName,
  workflows,
  client: new Client({ secretKey, strictAuthentication: true, verbose: false }),  // 强制 true
  handler: this.novuHandler.handler,
});
```

### 6.5 测试环境禁用

**文件**：`apps/dashboard/tests/utils/test-bridge-server.ts:14`

```typescript
this.client = new Client({ strictAuthentication: false, secretKey, apiUrl });
```

**文件**：`apps/api/e2e/test-bridge-server.ts:9`

```typescript
public client = new Client({ strictAuthentication: false });
```

---

## 7. 失败重试与去重机制

### 7.1 出站 Webhook 重试（Svix 自动处理）

Svix 作为专业的 webhook 服务提供完善的重试机制：

**重试策略**：指数退避（Exponential Backoff）
**重试次数**：Svix 默认最多重试 25 次，时间跨度约 3 天
**成功判定**：接收方返回 2xx 状态码视为成功

### 7.2 内部 Webhook Filter 重试策略

**文件**：`apps/worker/src/app/workflow/usecases/webhook-filter-backoff-strategy/webhook-filter-backoff-strategy.usecase.ts:11-36`

用于工作流执行中的 webhook filter 步骤重试：

```typescript
public async execute(command: WebhookFilterBackoffStrategyCommand): Promise<number> {
  const { attemptsMade, eventError: error, eventJob } = command;

  // 记录重试执行详情
  await this.createExecutionDetails.execute(
    CreateExecutionDetailsCommand.create({
      ...CreateExecutionDetailsCommand.getDetailsFromJob(job),
      detail: DetailEnum.WEBHOOK_FILTER_FAILED_RETRY,
      source: ExecutionDetailsSourceEnum.WEBHOOK,
      status: ExecutionDetailsStatusEnum.PENDING,
      isTest: false,
      isRetry: true,
      raw: JSON.stringify({ message: JSON.parse(error?.message).message, attempt: attemptsMade }),
    })
  );

  // 指数退避 + 随机抖动
  return Math.round(Math.random() * 2 ** attemptsMade * 1000); // 毫秒
}
```

**重试间隔计算**：`随机因子 * 2^重试次数 * 1000ms`
- 第 1 次重试：0-2 秒
- 第 2 次重试：0-4 秒
- 第 3 次重试：0-8 秒
- ...

### 7.3 Bridge 请求重试

**文件**：`libs/application-generic/src/usecases/execute-bridge-request/execute-framework-request.usecase.ts:116-118`

```typescript
retry: {
  limit: retriesLimit,  // 默认 DEFAULT_RETRIES_LIMIT = 3
},
```

### 7.4 去重机制

#### 事件 ID 去重
**文件**：`libs/application-generic/src/webhooks/usecases/send-webhook-message/send-webhook-message.usecase.ts:49`

```typescript
const eventId = `evt_${generateObjectId()}`;
```

- 每个 webhook 事件生成全局唯一的 `eventId`
- 格式：`evt_` + ObjectId（24 位十六进制）
- Svix 使用 `eventId` 作为幂等键，确保同一事件不会被重复投递

#### 幂等性保证
- 发送时明确指定 `eventId` 给 Svix：`svix.message.create(appId, { eventId, ... })`
- Svix 确保对于相同的 `eventId`，即使多次调用 API 也只会投递一次

---

## 8. 接收方身份验证与历史投递回溯

### 8.1 Webhook Portal 访问管理

**文件**：`apps/api/src/app/outbound-webhooks/outbound-webhooks.controller.ts:13-57`

用户通过 Svix App Portal 管理 webhook 配置，Novu 提供 API 生成访问令牌。

#### 初始化 Webhook Portal
**API**：`POST /v2/outbound-webhooks/portal/token`

**文件**：`apps/api/src/app/outbound-webhooks/usecases/create-webhook-portal-token/create-webhook-portal.usecase.ts:16-55`

```typescript
async execute(command: CreateWebhookPortalCommand): Promise<CreateWebhookPortalResponseDto> {
  // 在 Svix 创建应用
  const app = await this.svix.application.create({
    name: organization.name,
    uid: generateWebhookAppId(command.organizationId, command.environmentId),
    metadata: {
      environmentId: command.environmentId,
      organizationId: command.organizationId,
    },
  });

  // 保存 webhookAppId 到环境配置
  await this.environmentRepository.updateOne(
    { _id: command.environmentId },
    { $set: { webhookAppId: app.uid } }
  );

  return { appId: app.uid! };
}
```

#### 获取 Portal 访问令牌
**API**：`GET /v2/outbound-webhooks/portal/token`

**文件**：`apps/api/src/app/outbound-webhooks/usecases/get-webhook-portal-token/get-webhook-portal-token.usecase.ts:15-50`

```typescript
async execute(command: GetWebhookPortalTokenCommand): Promise<GetWebhookPortalTokenResponseDto> {
  const svixResponse = await this.svix.authentication.appPortalAccess(
    environment.webhookAppId,
    {}
  );

  return {
    url: svixResponse.url,
    token: svixResponse.token,
    appId: environment.webhookAppId,
  };
}
```

### 8.2 App ID 生成规则

**文件**：`libs/application-generic/src/webhooks/utils/app-id.ts:7-9`

```typescript
export function generateWebhookAppId(organizationId: OrganizationId, environmentId: EnvironmentId): string {
  return `o-${organizationId}-e-${environmentId}`;
}
```

### 8.3 企业版 vs 社区版差异

**文件**：`apps/api/src/app/outbound-webhooks/outbound-webhooks.module.ts:14-50`

- **企业版**：完整的 Svix 集成，包含所有 webhook 功能
- **社区版**：使用 `NoopSendWebhookMessage` 空实现，webhook 功能被禁用

### 8.4 历史投递回溯

#### 通过 Event ID 追踪
每个 webhook payload 包含唯一的 `id` 字段（即 `eventId`），可用于：
1. 去重处理：接收方可以记录已处理的 `eventId`，避免重复处理
2. 问题排查：通过 `eventId` 在 Svix Portal 中查询具体的投递记录
3. 审计追踪：完整记录每次回调的时间和内容

#### Svix Portal 功能
用户通过 Svix App Portal 可以：
- 查看所有 webhook 事件的投递历史
- 检查每次投递的状态（成功/失败/重试中）
- 查看请求和响应的详细内容
- 手动重发失败的事件
- 配置回调端点 URL 和订阅的事件类型

#### Payload 中的追踪字段
```typescript
{
  "id": "evt_65abc123def456...",      // 事件唯一标识
  "type": "message.sent",              // 事件类型
  "timestamp": "2024-01-15T10:30:00.000Z", // 事件发生时间
  "environmentId": "prod-env-123",     // 环境标识
  "object": "message",                 // 对象类型
  "data": { ... }                      // 业务数据
}
```

---

## 9. 入站 Webhook 处理（平台接收第三方回调）

### 9.1 入口端点

**文件**：`apps/webhook/src/webhooks/webhooks.controller.ts:12-46`

```typescript
@Controller('/webhooks')
export class WebhooksController {
  @Post('/organizations/:organizationId/environments/:environmentId/email/:providerOrIntegrationId')
  public emailWebhook(...) { ... }

  @Post('/organizations/:organizationId/environments/:environmentId/sms/:providerOrIntegrationId')
  public smsWebhook(...) { ... }
}
```

### 9.2 处理流程

**文件**：`apps/webhook/src/webhooks/usecases/webhook/webhook.usecase.ts:24-141`

```typescript
async execute(command: WebhookCommand): Promise<IWebhookResult[]> {
  // 1. 查找集成配置
  const integration = await this.integrationRepository.findOne(query);

  // 2. 创建对应 provider handler
  this.createProvider(integration, command.type);

  // 3. 从回调中提取消息 ID
  const messageIdentifiers = this.provider.getMessageId(body);

  // 4. 逐条处理事件
  for (const messageIdentifier of messageIdentifiers) {
    // 查找消息
    const message = await this.messageRepository.findOne({ identifier: messageIdentifier });

    // 解析事件类型（delivered, bounced, clicked 等）
    const event = this.provider.parseEventBody(body, messageIdentifier);

    // 创建执行详情
    await this.createExecutionDetails.execute({ message, webhook, webhookEvent, channel });
  }
}
```

### 9.3 Provider 接口要求

**文件**：`packages/stateless/src/lib/provider/provider.interface.ts`

支持 webhook 的 provider 必须实现：
- `getMessageId(body)` - 从回调中提取消息 ID
- `parseEventBody(body, messageId)` - 解析事件类型和详情

> **注意**：第三方回调触发的 `MESSAGE_DELIVERED` 等投递事件不通过 `sendWebhookMessage.execute()`，而是直接更新消息状态并创建执行详情。

---

## 10. 关键配置项

### 环境变量
- `SVIX_API_KEY` - Svix API 密钥，未配置时 webhook 功能禁用
- `NOVU_ENTERPRISE` - 是否为企业版，决定是否启用完整 webhook 功能
- `STORE_NOTIFICATION_CONTENT` - 是否存储通知内容
- `NOVU_STRICT_AUTHENTICATION_ENABLED` - Bridge 签名验证开关（默认生产环境 true）
- `NODE_ENV` - 影响 `strictAuthentication` 默认值

### 数据库字段
- `environment.webhookAppId` - 环境对应的 Svix 应用 ID
- `environment.bridge.url` / `environment.echo.url` - Bridge 端点 URL
- `environment.apiKeys[].key` - 环境 API 密钥（加密存储）

---

## 11. 接收方集成指南

### 11.1 验证 Svix 签名（接收 Novu 出站事件）

使用 Svix 官方 SDK 验证签名：

```javascript
import { Webhook } from 'svix';

const webhook = new Webhook(process.env.SVIX_WEBHOOK_SECRET);

app.post('/webhook', (req, res) => {
  const payload = JSON.stringify(req.body);
  const headers = req.headers;

  try {
    const verified = webhook.verify(payload, headers);

    // 处理不同事件类型
    const { type, data } = verified;
    switch (type) {
      case 'message.sent':
        // ⚠️ 注意：Chat 失败时也是 message.sent，但有 error 字段
        if (data.error) {
          console.log('Chat 消息发送失败:', data.error.message);
        } else {
          console.log('消息发送成功:', data.object._id, data.object.channel);
        }
        break;
      case 'message.failed':
        // Email/SMS/Push 失败（注意 Chat 不会触发这个）
        console.log('消息发送失败:', data.error.message);
        break;
      case 'message.delivered':
        // 仅 In-App 主动触发，其他渠道依赖第三方回调
        console.log('消息已投递:', data.object._id);
        break;
      case 'message.archived':
        console.log('消息已归档:', data.object._id);
        break;
      case 'message.snoozed':
        console.log('消息已稍后提醒:', data.object._id, data.object.snoozedUntil);
        break;
      case 'message.deleted':
        console.log('消息已删除:', data.object._id);
        break;
      case 'workflow.deleted':
        console.log('工作流已删除:', data.object._id, data.object.name);
        break;
      case 'workflow.published':
        console.log('工作流已发布:', data.object.name);
        break;
      case 'email.received':
        console.log('收到入站邮件:', data.object.mail.subject, data.object.mail.from);
        break;
      case 'preference.updated':
        console.log('偏好已更新:', data.subscriberId, data.object.workflow);
        break;
      // ... 其他事件类型
    }

    res.status(200).send('OK');
  } catch (err) {
    res.status(400).send('Invalid signature');
  }
});
```

### 11.2 去重处理（接收方实现）

```javascript
const processedEventIds = new Set();

app.post('/webhook', (req, res) => {
  const eventId = req.body.id;

  if (processedEventIds.has(eventId)) {
    return res.status(200).send('Already processed');
  }

  processedEventIds.add(eventId);
  // 处理事件...
});
```

### 11.3 Bridge 端签名验证（使用 Novu Framework）

```typescript
import { Client, NovuHandler, NovuRequestHandler } from '@novu/framework/nest';

// 生产环境：强制启用严格认证
const client = new Client({
  secretKey: process.env.NOVU_SECRET_KEY,
  strictAuthentication: process.env.NODE_ENV === 'production',
});

const handler = new NovuRequestHandler({
  frameworkName: 'express',
  workflows: [myWorkflow],
  client,
  handler: ({ step, payload }) => {
    // 工作流逻辑
  },
});

// 所有请求会自动验证 novu-signature 头部
app.post('/bridge', handler.createHandler());
```

### 11.4 回溯历史

1. 通过 Novu API 获取 Svix Portal 访问令牌
   ```bash
   GET /v2/outbound-webhooks/portal/token
   ```
2. 登录 Svix Portal 查看完整投递历史
3. 使用 `eventId` 搜索特定事件
4. 查看每次投递的请求/响应详情和重试记录

---

## 12. 已知问题与注意事项

### 12.1 ⚠️ Chat 发送失败事件类型不一致

**问题代码**：`apps/worker/src/app/workflow/usecases/send-message/send-message-chat.usecase.ts:854`

**问题描述**：Chat 渠道发送失败时，使用的事件类型是 `MESSAGE_SENT` 而非 `MESSAGE_FAILED`，与 Email/SMS/Push 等其他渠道不一致。

**接收方处理建议**：
```typescript
if (event.type === 'message.sent') {
  if (event.data.error) {
    // Chat 发送失败（特殊情况）
    handleChatFailure(event.data);
  } else {
    // 发送成功（所有渠道）
    handleSuccess(event.data);
  }
} else if (event.type === 'message.failed') {
  // Email/SMS/Push 发送失败
  handleOtherFailure(event.data);
}
```

### 12.2 MESSAGE_DELIVERED 仅 In-App 主动触发

**问题描述**：`MESSAGE_DELIVERED` 事件仅在 In-App 渠道发送成功后由 `sendWebhookMessage` 主动触发（`send-message-in-app.usecase.ts:319`）。其他渠道的 `MESSAGE_DELIVERED` 依赖第三方 webhook 回调（通过 `apps/webhook` 服务），不经过 `sendWebhookMessage` 路径。

### 12.3 In-App 成功触发两个事件

**问题描述**：In-App 发送成功后会连续触发 `MESSAGE_SENT` 和 `MESSAGE_DELIVERED` 两个事件，接收方需要注意处理重复。

### 12.4 两条签名链路使用不同密钥

- **Svix 出站**：使用 Svix Signing Secret（在 Svix Portal 中获取）
- **Bridge 入站**：使用 Novu 环境 Secret Key（在 Novu 管理后台获取）

不要混淆这两个密钥。

### 12.5 社区版不支持出站 Webhook

社区版使用 `NoopSendWebhookMessage` 空实现，所有 `sendWebhookMessage.execute()` 调用直接返回 `undefined`，不会发送任何 webhook。

---

## 13. 核心文件索引（按调用点排序）

| # | 文件路径 | 功能 | 调用行号 |
|---|---------|-----|---------|
| 1 | `apps/worker/src/app/workflow/usecases/send-message/send-message-in-app.usecase.ts` | In-App 发送（MESSAGE_SENT + MESSAGE_DELIVERED） | 307, 319 |
| 2 | `apps/worker/src/app/workflow/usecases/send-message/send-message-chat.usecase.ts` | Chat 发送（成功 + 失败） | 805, 854 |
| 3 | `apps/worker/src/app/workflow/usecases/send-message/send-message-email.usecase.ts` | Email 发送（成功 + 失败） | 487, 553 |
| 4 | `apps/worker/src/app/workflow/usecases/send-message/send-message-sms.usecase.ts` | SMS 发送（成功 + 失败） | 356, 381 |
| 5 | `apps/worker/src/app/workflow/usecases/send-message/send-message-push.usecase.ts` | Push 发送（成功 + 失败） | 644, 726 |
| 6 | `apps/api/src/app/inbox/usecases/mark-many-notifications-as/mark-many-notifications-as.usecase.ts` | 批量标记（已读/归档/稍后） | 149 |
| 7 | `apps/api/src/app/inbox/usecases/update-all-notifications/update-all-notifications.usecase.ts` | 按条件更新（已读/归档/稍后） | 170 |
| 8 | `apps/api/src/app/inbox/usecases/delete-many-notifications/delete-many-notifications.usecase.ts` | 批量删除 | 127 |
| 9 | `apps/api/src/app/inbox/usecases/delete-all-notifications/delete-all-notifications.usecase.ts` | 按条件删除 | 154 |
| 10 | `apps/api/src/app/inbox/usecases/mark-notifications-as-seen/mark-notifications-as-seen.usecase.ts` | 批量标记已看 | 185 |
| 11 | `apps/api/src/app/inbox/usecases/update-preferences/update-preferences.usecase.ts` | 偏好更新 | 78 |
| 12 | `apps/api/src/app/widgets/usecases/mark-message-as/mark-message-as.usecase.ts` | Widget 单个标记 | 184 |
| 13 | `apps/api/src/app/widgets/usecases/mark-all-messages-as/mark-all-messages-as.usecase.ts` | Widget 批量标记 | 71 |
| 14 | `apps/api/src/app/widgets/usecases/mark-message-as-by-mark/mark-message-as-by-mark.usecase.ts` | Widget 按 mark 标记 | 80 |
| 15 | `libs/application-generic/src/usecases/upsert-workflow/upsert-workflow.usecase.ts` | 工作流创建/更新（v1） | 114, 125 |
| 16 | `apps/api/src/app/workflows-v2/usecases/patch-workflow/patch-workflow.usecase.ts` | 工作流更新（v2） | 57 |
| 17 | `apps/api/src/app/workflows-v1/usecases/delete-workflow/delete-workflow.usecase.ts` | 工作流删除 | 50 |
| 18 | `apps/api/src/app/workflows-v2/usecases/sync-to-environment/sync-to-environment.usecase.ts` | 工作流发布 | 150 |
| 19 | `libs/application-generic/src/usecases/inbound-domain-route-delivery/inbound-domain-route-delivery.usecase.ts` | 入站邮件 | 121 |
| 20 | `libs/application-generic/src/webhooks/usecases/send-webhook-message/send-webhook-message.usecase.ts` | Webhook 发送核心逻辑 | 42 |
