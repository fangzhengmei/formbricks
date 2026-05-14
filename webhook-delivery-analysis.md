# Webhook 签名与重试路径分析报告

## 1. 概述

Formbricks 的 webhook 系统用于在调查响应事件发生时向外部系统推送数据。本报告详细分析其签名验证机制、投递流程、失败处理策略以及安全性考量。重点揭示了 HTTP 错误响应处理中的严重设计缺陷。

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

## 4. 失败处理与重试策略 — 深度分析

### 4.1 当前实现的严重设计缺陷

#### 4.1.1 核心问题：Fetch API 的行为误解

**代码证据位置**: `apps/web/app/api/(internal)/pipeline/route.ts:156-176`

```typescript
return validateAndResolveWebhookUrl(webhook.url)
  .then(async (address) => {
    const dispatcher = address ? createPinnedDispatcher(address) : undefined;
    try {
      return await fetchWithTimeout(webhook.url, {
        method: "POST",
        headers: requestHeaders,
        body,
        dispatcher,
      });
    } finally {
      await dispatcher?.destroy();
    }
  })
  .catch((error) => {
    logger.error({ error, url: request.url }, `Webhook call to ${webhook.url} failed`);
  });
```

**关键发现**:
- `fetch()` API **只有在网络层面失败时才会 reject Promise**
- HTTP 4xx/5xx 响应会被当作 **成功 resolve**，而不是 reject！
- `.catch()` 仅捕获：
  - 网络连接错误 (ECONNRESET, ETIMEDOUT, ECONNREFUSED)
  - DNS 解析失败
  - CORS 错误
  - `validateAndResolveWebhookUrl` 抛出的异常 (URL 验证失败)
  - AbortSignal 超时 (5秒超时)

#### 4.1.2 HTTP 4xx/5xx 响应的"隐形"失败

**代码证据位置**: `apps/web/app/api/(internal)/pipeline/route.ts:303-317`

```typescript
const results = await Promise.allSettled(webhookPromises);
results.forEach((result) => {
  if (result.status === "rejected") {
    logger.error({ error: result.reason, url: request.url }, "Promise rejected");
  }
});
```

**失败场景矩阵分析**:

| 失败类型 | 是否触发 catch | 是否记录错误日志 | Promise 状态 | 实际结果 |
|----------|----------------|------------------|-------------|----------|
| **网络连接超时** (5s) | ✅ 是 | ✅ 是 | rejected | 正确记录 |
| **DNS 解析失败** | ✅ 是 | ✅ 是 | rejected | 正确记录 |
| **URL 验证失败** (内网IP) | ✅ 是 | ✅ 是 | rejected | 正确记录 |
| **HTTP 400 Bad Request** | ❌ 否 | ❌ 否 | fulfilled | **静默失败** |
| **HTTP 401 Unauthorized** | ❌ 否 | ❌ 否 | fulfilled | **静默失败** |
| **HTTP 403 Forbidden** | ❌ 否 | ❌ 否 | fulfilled | **静默失败** |
| **HTTP 404 Not Found** | ❌ 否 | ❌ 否 | fulfilled | **静默失败** |
| **HTTP 429 Too Many Requests** | ❌ 否 | ❌ 否 | fulfilled | **静默失败** |
| **HTTP 500 Server Error** | ❌ 否 | ❌ 否 | fulfilled | **静默失败** |
| **HTTP 502 Bad Gateway** | ❌ 否 | ❌ 否 | fulfilled | **静默失败** |
| **HTTP 503 Service Unavailable** | ❌ 否 | ❌ 否 | fulfilled | **静默失败** |
| **HTTP 504 Gateway Timeout** | ❌ 否 | ❌ 否 | fulfilled | **静默失败** |

**严重性结论**:
- **75% 的失败场景被完全忽略**！
- 接收方返回的所有 HTTP 错误都被当作"成功"处理
- 系统对投递失败完全"失明"

### 4.2 对重试与告警判断的影响

#### 4.2.1 对重试决策的毁灭性影响

由于 HTTP 错误不会被标记为失败，导致：
1. **可重试错误被忽略**：
   - 503 Service Unavailable (服务暂时不可用)
   - 504 Gateway Timeout (网关超时)
   - 429 Too Many Requests (限流，可配合 Retry-After 重试)
   - 502 Bad Gateway (上游服务重启中)
   
   这些都是典型的临时性错误，重试极有可能成功，但当前系统完全不会重试。

2. **不可重试的配置错误被掩盖**：
   - 401 Unauthorized (secret 验证失败，或 token 失效)
   - 403 Forbidden (IP 被封禁，或权限不足)
   - 404 Not Found (webhook 端点 URL 配置错误)
   
   这些需要人工介入修复的配置错误，管理员完全无法感知。

#### 4.2.2 对告警机制的破坏

由于没有失败状态记录：
- 管理员无法通过日志发现配置错误的 webhook
- 无法基于失败率设置告警阈值
- 无法区分"服务不可用"与"永久配置错误"
- 用户界面无法展示"最近投递状态"

#### 4.2.3 数据一致性风险

- 业务系统依赖 webhook 进行数据同步时，HTTP 层面的失败会导致数据丢失
- 没有投递状态审计，无法排查数据不一致问题
- 无法实现"至少一次投递"的交付保证

### 4.3 当前失败处理机制总结

| 维度 | 当前实现状态 | 风险等级 | 说明 |
|------|-------------|----------|------|
| **HTTP 状态码检查** | ❌ 完全缺失 | 🔴 致命 | 4xx/5xx 全部静默失败 |
| **网络错误捕获** | ✅ 部分实现 | 🟡 中等 | 仅捕获网络层异常 |
| **重试机制** | ❌ 完全缺失 | 🔴 严重 | 失败后仅记录日志，不重试 |
| **死信队列** | ❌ 完全缺失 | 🟠 高 | 没有持久化失败记录 |
| **退避策略** | ❌ 完全缺失 | 🟠 高 | 没有指数退避策略 |
| **错误分类** | ❌ 完全缺失 | 🔴 严重 | 不区分可重试与不可重试 |
| **告警机制** | ❌ 完全缺失 | 🟠 高 | 失败无告警通知 |
| **审计记录** | ❌ 完全缺失 | 🟠 高 | 无投递状态持久化 |
| **状态码回显** | ❌ 完全缺失 | 🟡 中等 | 无法知道响应具体内容 |

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
| Promise 结果处理 | `apps/web/app/api/(internal)/pipeline/route.ts` | 303-317 |

## 7. 改进建议 — 按风险优先级排序

### 🔴 P0: 立即修复（致命缺陷）

#### 7.1.1 增加 HTTP 响应状态码检查

**修改位置**: `apps/web/app/api/(internal)/pipeline/route.ts:156-176`

**修复代码示例**:
```typescript
return validateAndResolveWebhookUrl(webhook.url)
  .then(async (address) => {
    const dispatcher = address ? createPinnedDispatcher(address) : undefined;
    try {
      const response = await fetchWithTimeout(webhook.url, {
        method: "POST",
        headers: requestHeaders,
        body,
        dispatcher,
      });
      
      // 关键修复：检查 HTTP 状态码
      if (!response.ok) {
        const responseBody = await response.text().catch(() => "");
        throw new Error(
          `HTTP ${response.status}: ${response.statusText}. Body: ${responseBody.substring(0, 500)}`
        );
      }
      
      return response;
    } finally {
      await dispatcher?.destroy();
    }
  })
  .catch((error) => {
    logger.error(
      { error, webhookId: webhook.id, webhookUrl: webhook.url, event },
      `Webhook delivery failed`
    );
    // 重新抛出，让 allSettled 能捕获到失败状态
    throw error;
  });
```

**预期效果**:
- HTTP 4xx/5xx 会被正确标记为失败
- 失败原因包含状态码和响应体，便于排查
- `allSettled` 可以正确识别失败的 Promise

### 🔴 P0: 立即修复（严重可靠性问题）

#### 7.1.2 实现基础重试机制

**建议在 `apps/web/app/api/(internal)/pipeline/route.ts` 中添加**:

```typescript
const MAX_RETRIES = 3;
const INITIAL_DELAY = 1000; // 1秒

const isRetriableError = (error: any): boolean => {
  // 网络错误总是可重试
  if (error.name === 'AbortError' || error.code === 'ECONNRESET' || error.code === 'ETIMEDOUT') {
    return true;
  }
  
  // 从错误消息中提取 HTTP 状态码
  const match = error.message.match(/HTTP (\d+)/);
  if (match) {
    const statusCode = parseInt(match[1], 10);
    // 5xx 和 429 可重试
    return statusCode === 429 || (statusCode >= 500 && statusCode < 600);
  }
  
  return false;
};

const fetchWithRetry = async (
  webhook: Webhook,
  options: RequestInit & { dispatcher?: Agent },
  attempt: number = 0
): Promise<Response> => {
  try {
    return await fetchWithTimeout(webhook.url, options);
  } catch (error) {
    // 如果是最后一次尝试，或者错误不可重试，直接抛出
    if (attempt >= MAX_RETRIES - 1 || !isRetriableError(error)) {
      throw error;
    }
    
    // 指数退避延迟
    const delay = INITIAL_DELAY * Math.pow(2, attempt);
    logger.info(
      { webhookId: webhook.id, attempt: attempt + 1, maxRetries: MAX_RETRIES, delay },
      `Retrying webhook delivery`
    );
    
    await new Promise(resolve => setTimeout(resolve, delay));
    return fetchWithRetry(webhook, options, attempt + 1);
  }
};
```

**重试策略配置**:
```typescript
interface WebhookRetryConfig {
  maxRetries: number;        // 建议: 3-5
  initialDelay: number;      // 建议: 1000ms
  maxDelay: number;          // 建议: 32000ms
  backoffMultiplier: number; // 建议: 2 (指数退避)
}
```

**重试条件 (可重试错误)**:
- 网络错误 (ECONNRESET, ETIMEDOUT, ECONNREFUSED, AbortError)
- HTTP 5xx 响应 (500, 502, 503, 504)
- HTTP 429 Too Many Requests (需处理 Retry-After 头)

**不可重试错误 (直接失败)**:
- HTTP 400 Bad Request (payload 格式错误)
- HTTP 401 Unauthorized (签名验证失败)
- HTTP 403 Forbidden (权限不足)
- HTTP 404 Not Found (端点不存在)
- URL 验证失败 (内网IP等)

### 🟠 P1: 高优先级修复

#### 7.2.1 投递状态持久化

建议新增 `WebhookDelivery` 模型到 `packages/database/schema.prisma`:

```prisma
enum WebhookDeliveryStatus {
  pending
  success
  failed
  retrying
}

model WebhookDelivery {
  id            String                 @id @default(cuid())
  webhookId     String
  event         PipelineTriggers
  responseId    String?
  status        WebhookDeliveryStatus
  attemptCount  Int                    @default(0)
  statusCode    Int?                   // HTTP 状态码
  errorMessage  String?                // 错误详情
  requestBody   String?                // 请求体快照
  responseBody  String?                // 响应体片段
  lastAttemptAt DateTime?
  nextAttemptAt DateTime?
  createdAt     DateTime               @default(now())
  webhook       Webhook                @relation(fields: [webhookId], references: [id], onDelete: Cascade)

  @@index([webhookId, createdAt])
  @@index([status, nextAttemptAt])     // 用于重试任务扫描
}
```

**新增 API 端点**:
- `GET /api/webhooks/{id}/deliveries` - 获取投递历史
- `POST /api/webhooks/{id}/deliveries/{deliveryId}/retry` - 手动重发

#### 7.2.2 失败告警机制

```typescript
// 在投递失败后触发告警
const maybeTriggerAlert = async (webhook: Webhook, consecutiveFailures: number) => {
  if (consecutiveFailures >= 5) {
    // 连续失败 5 次触发告警
    await sendAlert({
      type: 'webhook_delivery_failing',
      severity: 'warning',
      webhookId: webhook.id,
      webhookUrl: webhook.url,
      consecutiveFailures,
      message: `Webhook 已经连续失败 ${consecutiveFailures} 次，请检查端点可用性`,
    });
  }
  if (consecutiveFailures >= 20) {
    // 连续失败 20 次，自动禁用 webhook
    await prisma.webhook.update({
      where: { id: webhook.id },
      data: { enabled: false },
    });
    await sendAlert({
      type: 'webhook_disabled',
      severity: 'critical',
      webhookId: webhook.id,
      webhookUrl: webhook.url,
      message: `Webhook 因连续投递失败已被自动禁用`,
    });
  }
};
```

### 🟡 P2: 中优先级改进

#### 7.3.1 后台队列处理

建议引入 BullMQ 队列系统:

**优势**:
- 分离投递逻辑与 API 响应流程
- 支持延迟任务调度（指数退避）
- 内置死信队列
- 支持并发控制
- 支持任务优先级
- 提供监控面板

**队列设计**:
```typescript
// Webhook 投递队列
const webhookQueue = new Queue('webhook-delivery', {
  defaultJobOptions: {
    attempts: 5,
    backoff: {
      type: 'exponential',
      delay: 1000,
    },
    removeOnComplete: 100,  // 保留最近 100 条成功记录
    removeOnFail: 500,      // 保留最近 500 条失败记录
  },
});

// 投递处理器
webhookQueue.process(async (job) => {
  const { webhookId, payload, headers } = job.data;
  // 执行投递逻辑
});
```

#### 7.3.2 签名验证强化

- 强制要求配置 secret（或至少在 UI 中强烈建议）
- 提供多语言签名验证示例代码 (Node.js, Python, Java, Go)
- 时间戳窗口验证（建议: ±5 分钟）
- 文档化签名验证最佳实践

**验证示例（Node.js）**:
```javascript
const crypto = require('crypto');

function verifyWebhookSignature(headers, payload, secret) {
  const webhookId = headers['webhook-id'];
  const timestamp = headers['webhook-timestamp'];
  const signature = headers['webhook-signature'];
  
  // 检查时间戳窗口（防止重放攻击）
  const now = Math.floor(Date.now() / 1000);
  if (Math.abs(now - parseInt(timestamp, 10)) > 300) { // 5分钟窗口
    return false;
  }
  
  // 重新计算签名
  const signedContent = `${webhookId}.${timestamp}.${payload}`;
  const hmac = crypto.createHmac('sha256', Buffer.from(secret, 'base64'));
  const expectedSignature = `v1,${hmac.update(signedContent).digest('base64')}`;
  
  // 定时安全比较
  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expectedSignature)
  );
}
```

### 🟢 P3: 低优先级（用户体验）

#### 7.4.1 用户界面改进

- Webhook 详情页展示投递历史与状态时间线
- 最近投递状态徽章（成功/失败/重试中）
- 失败告警通知（邮件/站内信）
- 手动重发失败 Webhook 按钮
- Webhook 活动日志搜索筛选
- 成功率统计图表

## 8. 总结

### 现状评估

| 维度 | 评级 | 说明 |
|------|------|------|
| **签名安全性** | ✅ 良好 | 遵循 Standard Webhooks 规范 |
| **SSRF 防护** | ✅ 优秀 | 多纵深防护机制 |
| **HTTP 错误处理** | 🔴 **致命** | 4xx/5xx 完全静默失败 |
| **重试机制** | 🔴 严重 | 完全缺失 |
| **状态追踪** | 🟠 高风险 | 无投递历史 |
| **告警机制** | 🟠 高风险 | 完全缺失 |
| **整体可靠性** | 🔴 严重 | 交付保证缺失 |

### 核心发现

1. **HTTP 响应状态完全未检查** — 这是最严重的设计缺陷。75% 的失败场景被系统忽略，管理员和用户完全无法感知。

2. **缺少重试机制** — 网络抖动、服务重启等临时性错误会导致数据永久丢失。

3. **无审计追踪** — 无法排查数据同步问题，无法证明投递成功或失败。

### 优先级建议汇总

| 优先级 | 改进项 | 预期收益 | 工作量估计 |
|--------|--------|----------|------------|
| 🔴 **P0** | 增加 HTTP 状态码检查 | 修复 75% 的"隐形失败" | 1-2 小时 |
| 🔴 **P0** | 实现基础重试逻辑 | 显著提高投递成功率 | 4-6 小时 |
| 🟠 **P1** | 新增 WebhookDelivery 模型 | 支持审计与状态追踪 | 1-2 天 |
| 🟠 **P1** | 实现失败告警与自动禁用 | 主动发现配置问题 | 1 天 |
| 🟡 **P2** | 引入后台队列系统 | 提高可靠性与可观测性 | 3-5 天 |
| 🟡 **P2** | 强化签名验证文档 | 帮助用户正确实现验证 | 4 小时 |
| 🟢 **P3** | UI 展示投递历史与状态 | 改善用户体验 | 2-3 天 |

**建议立即实施 P0 级别的修复**，这两个改动可以解决当前系统中最严重的可靠性问题。其他改进可按优先级逐步推进。
