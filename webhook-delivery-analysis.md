# Webhook 签名与重试路径分析报告

## 1. 概述

Formbricks 的 webhook 系统用于在调查响应事件发生时向外部系统推送数据。本报告详细分析其签名验证机制、投递流程、失败处理策略以及安全性考量。

## 2. 签名机制分析

### 2.1 密钥生成

**位置**: `apps/web/lib/crypto.ts:155-158`

```typescript
export const generateWebhookSecret = (): string => {
  const secretBytes = randomBytes(32); // 256 bits of entropy
  return `${WEBHOOK_SECRET_PREFIX}${secretBytes.toString("base64")}`;
};
```

**特性**:
- 使用 256 位随机熵（32 字节）
- 前缀标识: `whsec_`
- Base64 编码便于传输存储

### 2.2 签名算法

**位置**: `apps/web/lib/crypto.ts:184-193`

```typescript
export const generateStandardWebhookSignature = (
  webhookId: string,
  timestamp: number,
  payload: string,
  secret: string
): string => {
  const signedContent = `${webhookId}.${timestamp}.${payload}`;
  const secretBytes = getWebhookSecretBytes(secret);
  const signature = createHmac("sha256", secretBytes).update(signedContent).digest("base64");
  return `v1,${signature}`;
};
```

**签名规范** (Standard Webhooks):
- 签名内容: `{webhook-id}.{unix-timestamp}.{payload}`
- 算法: HMAC-SHA256
- 输出格式: `v1,{base64-signature}`

### 2.3 HTTP 头

**位置**: `apps/web/app/api/(internal)/pipeline/route.ts:140-153`

| Header | 说明 |
|--------|------|
| `content-type` | `application/json` |
| `webhook-id` | 消息唯一标识 (UUIDv7) |
| `webhook-timestamp` | Unix 时间戳（秒） |
| `webhook-signature` | HMAC 签名（仅当配置 secret 时） |

## 3. 投递流程分析

### 3.1 触发入口

**位置**: `apps/web/app/lib/pipelines.ts:5-24`

- 通过内部 API `/api/pipeline` 触发
- 需要 `CRON_SECRET` 进行身份验证
- 支持三种事件类型 (`PipelineTriggers`):
  - `responseCreated` - 响应创建时
  - `responseUpdated` - 响应更新时
  - `responseFinished` - 响应完成时

### 3.2 Webhook 筛选逻辑

**位置**: `apps/web/app/api/(internal)/pipeline/route.ts:82-91`

```typescript
const webhooks = await prisma.webhook.findMany({
  where: {
    environmentId,
    triggers: { has: event },
    OR: [{ surveyIds: { has: surveyId } }, { surveyIds: { isEmpty: true } }],
  },
});
```

筛选条件:
1. 匹配当前环境
2. webhook 配置包含当前触发事件
3. 匹配特定 survey 或全局 (空 surveyIds)

### 3.3 Payload 结构

**位置**: `apps/web/app/api/(internal)/pipeline/route.ts:120-134`

```typescript
{
  webhookId: string,
  event: PipelineTriggers,
  data: {
    ...response,
    data: resolvedResponseData, // 存储 URL 已解析
    survey: { title, type, status, createdAt, updatedAt }
  }
}
```

### 3.4 投递实现

**位置**: `apps/web/app/api/(internal)/pipeline/route.ts:119-177`

核心特性:
1. **5秒超时保护**: `AbortSignal.timeout(5000)`
2. **并行投递**: 使用 `Promise.allSettled` 并行发送
3. **URL 验证**: `validateAndResolveWebhookUrl`
4. **Pinned Dispatcher**: 防止 DNS 重新绑定攻击

## 4. 失败处理与重试策略

### 4.1 当前实现

**位置**: `apps/web/app/api/(internal)/pipeline/route.ts:174-176, 305-317`

```typescript
.catch((error) => {
  logger.error({ error, url: request.url }, `Webhook call to ${webhook.url} failed`);
});
```

### 4.2 失败处理机制

| 维度 | 当前实现 | 说明 |
|------|----------|------|
| **重试机制** | ❌ 无 | 失败后仅记录日志，不重试 |
| **死信队列** | ❌ 无 | 没有持久化失败记录 |
| **退避策略** | ❌ 无 | 没有指数退避等重试策略 |
| **错误分类** | ❌ 无 | 不区分可重试（5xx）与不可重试（4xx）错误 |
| **告警机制** | ❌ 无 | 失败无告警通知 |
| **审计记录** | ❌ 无 | 无投递状态持久化记录 |

### 4.3 关键问题分析

**问题 1: 缺少重试机制**
- 网络抖动、接收方短暂不可用等临时性错误会导致永久丢失
- 没有重试队列，无法在后续恢复

**问题 2: 缺少状态追踪**
- 无法审计 webhook 投递历史
- 用户无法查看哪些 webhook 发送失败
- 无法手动重发失败的 webhook

**问题 3: 超时设置较短**
- 5秒超时对于某些慢响应系统可能不足
- 没有分阶段超时策略

## 5. 安全性分析

### 5.1 SSRF 防护

**位置**: `apps/web/lib/utils/validate-webhook-url.ts`, `apps/web/app/api/(internal)/pipeline/route.ts:101, 156-161`

| 防护措施 | 实现 |
|----------|------|
| **协议限制** | 仅限 HTTPS（可通过环境变量放宽） |
| **内网 IP 阻止** | 验证并解析 URL，拒绝私有/保留 IP 范围 |
| **DNS 重新绑定防护** | `createPinnedDispatcher` 固定连接 IP，防止验证后 DNS 变更 |
| **重定向阻止** | `redirect: "manual"`，不跟随重定向 |

### 5.2 签名验证安全性

- ✅ 使用标准 HMAC-SHA256 算法
- ✅ 包含时间戳可防止重放攻击
- ✅ 包含唯一消息 ID 便于幂等处理
- ⚠️ 未强制要求 secret（可配置为空）
- ⚠️ 接收方需要自行实现签名验证逻辑

## 6. 关键代码路径索引

| 功能 | 文件位置 | 行号 |
|------|----------|------|
| 密钥生成 | `apps/web/lib/crypto.ts` | 155-158 |
| 签名生成 | `apps/web/lib/crypto.ts` | 184-193 |
| 投递主逻辑 | `apps/web/app/api/(internal)/pipeline/route.ts` | 119-177 |
| Webhook 筛选 | `apps/web/app/api/(internal)/pipeline/route.ts` | 82-91 |
| URL 验证 | `apps/web/lib/utils/validate-webhook-url.ts` | - |
| 测试端点 | `apps/web/modules/integrations/webhooks/lib/webhook.ts` | 169-252 |

## 7. 改进建议

### 7.1 重试机制增强

**建议实现**:
```typescript
// 建议的重试策略配置
interface WebhookRetryConfig {
  maxRetries: number;        // 建议: 5
  initialDelay: number;      // 建议: 1000ms
  maxDelay: number;          // 建议: 32000ms
  backoffMultiplier: number; // 建议: 2 (指数退避)
}
```

**重试条件**:
- 网络错误 (ECONNRESET, ETIMEDOUT, etc.)
- 5xx 响应 (500, 502, 503, 504)
- 429 Too Many Requests (需处理 Retry-After 头)

### 7.2 投递状态持久化

建议新增 `WebhookDelivery` 模型:
```prisma
model WebhookDelivery {
  id            String   @id @default(cuid())
  webhookId     String
  event         PipelineTriggers
  responseId    String?
  status         DeliveryStatus // pending, success, failed, retrying
  attemptCount  Int      @default(0)
  lastAttemptAt DateTime?
  nextAttemptAt DateTime?
  errorMessage  String?
  createdAt     DateTime @default(now())
  webhook       Webhook  @relation(fields: [webhookId], references: [id], onDelete: Cascade)
}
```

### 7.3 后台队列处理

建议使用 BullMQ 或类似队列系统:
- 分离投递逻辑与响应流程
- 支持延迟任务调度
- 内置重试与死信队列
- 支持并发控制

### 7.4 用户体验改进

- Webhook 详情页展示投递历史与状态
- 失败告警通知（邮件/站内信）
- 手动重发失败 Webhook 按钮
- Webhook 活动日志

### 7.5 签名验证强化

- 强制要求配置 secret（或至少强烈建议）
- 提供签名验证示例代码
- 时间戳窗口验证（建议: ±5 分钟）
- 文档化签名验证最佳实践

## 8. 总结

### 现状评估
- ✅ 签名机制遵循 Standard Webhooks 规范，设计合理
- ✅ SSRF 防护措施完善（多纵深防护）
- ❌ **严重缺失重试机制**，可靠性不足
- ❌ 缺少投递状态追踪与审计
- ❌ 缺少失败告警与手动重发能力

### 优先级建议
1. **高优先级**: 实现基础重试机制（至少 3 次重试 + 指数退避）
2. **中优先级**: 增加投递状态持久化模型
3. **中优先级**: 实现后台队列处理
4. **低优先级**: 用户界面展示与告警机制

当前系统在安全性方面设计良好，但可靠性方面存在明显不足，建议优先增强重试与状态追踪能力。
