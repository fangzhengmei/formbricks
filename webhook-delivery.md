# Webhook 投递链路分析报告

## 概述

Formbricks 的 webhook 系统采用 **Standard Webhooks 规范** 实现，提供了签名验证、防重放攻击、以及失败重试机制。本文档详细分析从事件触发到最终投递的完整链路。

---

## 一、签名生成流程

### 1.1 密钥生成

webhook 密钥遵循 Standard Webhooks 规范，格式为 `whsec_{base64_encoded_random_bytes}`。

**关键代码位置**：`apps/web/lib/crypto.ts:155-158`

```typescript
export const generateWebhookSecret = (): string => {
  const secretBytes = randomBytes(32); // 256 bits of entropy
  return `${WEBHOOK_SECRET_PREFIX}${secretBytes.toString("base64")}`;
};
```

**生成步骤**：
1. 使用 `randomBytes(32)` 生成 256 位随机熵
2. 添加 `whsec_` 前缀
3. Base64 编码后返回

### 1.2 签名算法

签名采用 HMAC-SHA256 算法，遵循 Standard Webhooks 规范。

**关键代码位置**：`apps/web/lib/crypto.ts:184-193`

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

**签名生成步骤**：
1. 构造待签名字符串：`{webhook-id}.{webhook-timestamp}.{body}`
2. 解码密钥：去除 `whsec_` 前缀，Base64 解码得到原始字节
3. 使用 HMAC-SHA256 计算签名
4. 格式化为：`v1,{base64_signature}`

### 1.3 请求头结构

每个 webhook 请求包含以下标准头：

| 请求头 | 说明 | 示例 |
|--------|------|------|
| `webhook-id` | 唯一消息标识符（UUID v7） | `018c5d5e-...` |
| `webhook-timestamp` | Unix 时间戳（秒） | `1704547200` |
| `webhook-signature` | HMAC-SHA256 签名（仅当配置密钥时） | `v1,abc123...` |
| `content-type` | 内容类型 | `application/json` |

**关键代码位置**：`apps/web/app/api/(internal)/pipeline/route.ts:127-145`

---

## 二、时间戳防重放机制

### 2.1 时间戳生成

每个 webhook 请求在发送前生成当前 Unix 时间戳（秒级精度）：

**关键代码位置**：`apps/web/app/api/(internal)/pipeline/route.ts:129`

```typescript
const webhookTimestamp = Math.floor(Date.now() / 1000);
```

### 2.2 接收方验证流程

根据 Standard Webhooks 规范，接收方应执行以下验证步骤：

1. **提取头信息**：获取 `webhook-id`、`webhook-timestamp`、`webhook-signature`
2. **时间窗口验证**：验证时间戳是否在可接受范围内（通常 5 分钟）
3. **签名验证**：使用相同算法计算签名并比对

### 2.3 防重放原理

- **时间戳唯一性**：每个请求都有独立的发送时间
- **时间窗口限制**：只接受最近时间（如 5 分钟内）的请求，旧请求自动失效
- **消息 ID 唯一性**：`webhook-id`（UUID v7）确保每条消息唯一，接收方可以记录已处理的 ID 进一步防重放

---

## 三、失败重试与退避策略

### 3.1 队列架构

webhook 投递通过 **BullMQ 队列** 异步处理，采用 Redis 作为存储后端。

**队列配置**：
- 队列名称：`background-jobs`
- 前缀：`formbricks:jobs`
- 任务名称：`response-pipeline.process`

### 3.2 重试配置

**关键配置位置**：`docs/development/technical-handbook/background-job-processing.mdx:273-274`

```typescript
// 默认重试策略
{
  attempts: 3,           // 最多重试 3 次
  backoff: {
    type: "exponential", // 指数退避
    delay: 1000          // 初始延迟 1000ms
  }
}
```

### 3.3 退避算法

采用 **指数退避** 策略，重试间隔按指数增长：

| 重试次数 | 延迟时间 |
|----------|----------|
| 第 1 次重试 | 1 秒 |
| 第 2 次重试 | 2 秒 |
| 第 3 次重试 | 4 秒 |

### 3.4 失败分支处理

**关键代码位置**：`apps/web/app/api/(internal)/pipeline/route.ts:155-157`

```typescript
.catch((error) => {
  logger.error({ error, url: request.url }, `Webhook call to ${webhook.url} failed`);
});
```

**失败处理逻辑**：

1. **提前失败重试**：webhook 投递失败会导致 BullMQ 任务失败，触发队列级重试
2. **最终失败降级**：在最后一次重试失败后，记录错误日志但不中断其他管道流程（如集成通知、邮件发送等）
3. **日志记录**：每次失败都记录结构化日志，包含：
   - `jobId` - 任务 ID
   - `attempt` - 当前重试次数
   - `environmentId` - 环境 ID
   - `surveyId` - 问卷 ID
   - `responseId` - 响应 ID

### 3.5 请求超时保护

**关键代码位置**：`apps/web/app/api/(internal)/pipeline/route.ts:101-106`

```typescript
const fetchWithTimeout = (url: string, options: RequestInit, timeout: number = 5000): Promise<Response> => {
  return Promise.race([
    fetch(url, { ...options, redirect: redirectMode }),
    new Promise<never>((_, reject) => setTimeout(() => reject(new Error("Timeout")), timeout)),
  ]);
};
```

- 默认超时：**5 秒**
- 使用 `Promise.race` 实现超时取消
- 超时触发 Error("Timeout") 并被捕获记录

---

## 四、完整投递链路

### 4.1 触发入口

当用户提交问卷响应时，响应处理程序调用 `sendToPipeline` 触发 webhook 投递。

**关键代码位置**：`apps/web/app/api/v1/client/[environmentId]/responses/[responseId]/lib/put-response-handler.ts:253-268`

```typescript
sendToPipeline({
  event: "responseUpdated",
  environmentId: survey.environmentId,
  surveyId: survey.id,
  response: responseData,
});

if (updatedResponse.finished) {
  sendToPipeline({
    event: "responseFinished",
    environmentId: survey.environmentId,
    surveyId: survey.id,
    response: responseData,
  });
}
```

### 4.2 触发事件类型

支持三种触发事件：
- `responseCreated` - 响应创建时
- `responseUpdated` - 响应更新时
- `responseFinished` - 响应完成时

### 4.3 投递流程

```
用户提交响应
    ↓
[响应处理程序] → sendToPipeline()
    ↓
[内部 API] POST /api/pipeline
    ↓
[验证] x-api-key 认证
    ↓
[查询匹配 webhook]
    environmentId 匹配
    triggers 包含当前事件
    surveyIds 匹配或为空
    ↓
[并行投递] Promise.allSettled()
    ├─ webhook 1
    │   ├─ 验证 URL（SSRF 防护）
    │   ├─ 生成 UUID v7 消息 ID
    │   ├─ 生成时间戳
    │   ├─ 构造 payload
    │   ├─ 生成签名（如有密钥）
    │   └─ fetchWithTimeout 发送
    ├─ webhook 2
    └─ ...
    ↓
[结果处理]
    成功：无额外操作
    失败：记录错误日志，不中断其他流程
```

### 4.4 Webhook 匹配逻辑

**关键代码位置**：`apps/web/app/api/(internal)/pipeline/route.ts:81-90`

```typescript
const getWebhooksForPipeline = async (environmentId: string, event: PipelineTriggers, surveyId: string) => {
  const webhooks = await prisma.webhook.findMany({
    where: {
      environmentId,
      triggers: { has: event },
      OR: [{ surveyIds: { has: surveyId } }, { surveyIds: { isEmpty: true } }],
    },
  });
  return webhooks;
};
```

匹配条件：
1. `environmentId` 匹配 - 必须属于当前环境
2. `triggers` 包含当前事件 - 事件类型匹配
3. `surveyIds` 包含当前问卷 ID 或为空 - 全局或特定问卷配置

### 4.5 Payload 结构

**关键代码位置**：`apps/web/app/api/(internal)/pipeline/route.ts:110-125`

```typescript
{
  webhookId: "webhook_123",
  event: "responseFinished",
  data: {
    id: "response_456",
    surveyId: "survey_789",
    data: { /* 响应数据 */ },
    finished: true,
    survey: {
      title: "问卷名称",
      type: "web",
      status: "inProgress",
      createdAt: "...",
      updatedAt: "..."
    }
  }
}
```

---

## 五、安全防护措施

### 5.1 SSRF 防护

**关键代码位置**：`apps/web/app/api/(internal)/pipeline/route.ts:95-100`

```typescript
const redirectMode: RequestRedirect = DANGEROUSLY_ALLOW_WEBHOOK_INTERNAL_URLS ? "follow" : "manual";
```

- 默认模式：`redirect: "manual"` - 禁止重定向
- 目的：防止通过 3xx 重定向访问内网资源（如云元数据服务）
- 自托管用户可通过环境变量启用重定向跟随

### 5.2 URL 验证

投递前调用 `validateWebhookUrl` 验证 URL 安全性，防止访问内网 IP 段。

**关键代码位置**：`apps/web/app/api/(internal)/pipeline/route.ts:147`

### 5.3 超时保护

5 秒超时防止挂起请求耗尽连接池。

### 5.4 并行控制

使用 `Promise.allSettled` 并行投递，单个失败不影响其他 webhook。

---

## 六、可观测性

### 6.1 日志记录

失败时记录结构化日志：

```typescript
logger.error(
  { error, url: request.url },
  `Webhook call to ${webhook.url} failed`
);
```

### 6.2 测试端点

提供 webhook 测试功能，支持：
- URL 可访问性验证
- 签名生成测试
- 超时测试

**关键代码位置**：`apps/web/modules/integrations/webhooks/lib/webhook.ts:164-240`

---

## 七、关键入口文件汇总

| 文件路径 | 职责 |
|----------|------|
| `apps/web/lib/crypto.ts` | 签名生成、密钥管理 |
| `apps/web/app/api/(internal)/pipeline/route.ts` | 核心投递逻辑 |
| `apps/web/app/lib/pipelines.ts` | 触发入口封装 |
| `apps/web/modules/integrations/webhooks/lib/webhook.ts` | Webhook CRUD 与测试 |
| `apps/web/modules/response-pipeline/lib/process-response-pipeline-job.ts` | BullMQ 任务处理器 |
| `apps/web/instrumentation-jobs.ts` | Worker 启动注册 |

---

## 八、流程图总结

```
┌─────────────────┐
│  用户提交响应   │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────┐
│  sendToPipeline({event, ...})   │  ←-- 响应处理程序调用
└────────┬────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│  POST /api/pipeline             │  ←-- 内部 API（x-api-key 认证）
└────────┬────────────────────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌───────┐  ┌─────────┐
│ 查询  │  │ 加载    │
│ Webhook│  │ 问卷信息│
└───┬───┘  └────┬────┘
    │            │
    └─────┬──────┘
          ▼
┌─────────────────────────────────┐
│  为每个 webhook:                │
│  1. 生成 webhook-id (UUID v7)  │
│  2. 生成 webhook-timestamp     │
│  3. 构造 JSON payload          │
│  4. 如配置密钥，生成签名       │
│  5. validateWebhookUrl()       │
│  6. fetchWithTimeout (5s)      │
└────────┬────────────────────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌───────┐  ┌─────────┐
│ 成功  │  │  失败   │
└───┬───┘  └────┬────┘
    │            │
    │            ▼
    │    ┌──────────────┐
    │    │ BullMQ 重试  │  ←-- 指数退避 1s/2s/4s
    │    └──────┬───────┘
    │           │
    │           ▼
    │    ┌──────────────┐
    │    │  3 次后放弃  │
    │    └──────┬───────┘
    │           │
    └─────┬─────┘
          ▼
┌─────────────────────────────────┐
│  Promise.allSettled 汇总结果    │
│  所有失败都记录日志但不中断     │
└─────────────────────────────────┘
```
