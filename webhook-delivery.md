# Webhook 投递链路分析报告

> ⚠️ **重要声明**：本报告基于代码库实际现状整理，明确区分"已上线实现"与"文档/规划信息"。每节均标注证据来源，可追溯验证。

---

## 一、签名生成流程

### 1.1 密钥生成

**【已上线实现】** ✅

webhook 密钥遵循 Standard Webhooks 规范，格式为 `whsec_{base64_encoded_random_bytes}`。

**证据来源**：
- 代码位置：`apps/web/lib/crypto.ts:155-158`
- 数据库模型：`packages/database/zod/webhooks.ts` 包含 `secret` 字段

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

---

### 1.2 签名算法

**【已上线实现】** ✅

签名采用 HMAC-SHA256 算法，遵循 Standard Webhooks 规范。

**证据来源**：
- 代码位置：`apps/web/lib/crypto.ts:184-193`
- 调用位置：`apps/web/app/api/(internal)/pipeline/route.ts:138-144`

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

---

### 1.3 请求头结构

**【已上线实现】** ✅

每个 webhook 请求包含以下标准头：

| 请求头 | 说明 | 示例 |
|--------|------|------|
| `webhook-id` | 唯一消息标识符（UUID v7） | `018c5d5e-...` |
| `webhook-timestamp` | Unix 时间戳（秒） | `1704547200` |
| `webhook-signature` | HMAC-SHA256 签名（仅当配置密钥时） | `v1,abc123...` |
| `content-type` | 内容类型 | `application/json` |

**证据来源**：
- 代码位置：`apps/web/app/api/(internal)/pipeline/route.ts:127-145`

---

## 二、时间戳防重放机制

### 2.1 时间戳生成

**【已上线实现】** ✅

每个 webhook 请求在发送前生成当前 Unix 时间戳（秒级精度）：

**证据来源**：
- 代码位置：`apps/web/app/api/(internal)/pipeline/route.ts:129`

```typescript
const webhookTimestamp = Math.floor(Date.now() / 1000);
```

---

### 2.2 接收方验证流程

**【规范建议，非服务端实现】** ⚠️

根据 Standard Webhooks 规范，接收方应执行以下验证步骤：

1. **提取头信息**：获取 `webhook-id`、`webhook-timestamp`、`webhook-signature`
2. **时间窗口验证**：验证时间戳是否在可接受范围内（通常 5 分钟）
3. **签名验证**：使用相同算法计算签名并比对

**证据来源**：
- 文档位置：`docs/xm-and-surveys/core-features/integrations/webhooks.mdx`
- 说明：Formbricks 服务端仅负责生成签名和时间戳，不负责接收方的重放验证。接收方需要自行实现时间窗口检查。

---

### 2.3 防重放原理说明

**【规范建议】**

- **时间戳唯一性**：每个请求都有独立的发送时间
- **时间窗口限制**：接收方只接受最近时间（如 5 分钟内）的请求，旧请求自动失效
- **消息 ID 唯一性**：`webhook-id`（UUID v7）确保每条消息唯一，接收方可以记录已处理的 ID 进一步防重放

---

## 三、失败重试与退避策略

### 3.1 队列架构现状

**【关键结论：当前未使用 BullMQ 队列】** ❌

代码库中不存在 BullMQ 相关的实现，webhook 投递采用**同步 HTTP 调用**模式。

**证据核查结果**：
- ❌ 不存在 `@formbricks/jobs` 包
- ❌ 不存在 `packages/jobs/` 目录
- ❌ 不存在 `apps/web/modules/response-pipeline/` 目录
- ❌ 不存在 `apps/web/instrumentation-jobs.ts` 文件
- ❌ 不存在 `enqueueResponsePipelineEvents()` 函数
- ✅ 实际使用内部 HTTP 路由：`POST /api/(internal)/pipeline`

**文档与代码差异说明**：
> `docs/development/technical-handbook/background-job-processing.mdx` 中描述的 BullMQ 方案属于**规划文档或未合入的设计稿**，在当前代码库中**未实际实现**。

---

### 3.2 当前失败处理实现

**【已上线实现】** ✅

当前仅在**单个 webhook 级别**捕获异常并记录日志，**无任何重试机制**。

**证据来源**：
- 代码位置：`apps/web/app/api/(internal)/pipeline/route.ts:147-157`

```typescript
return validateWebhookUrl(webhook.url)
  .then(() =>
    fetchWithTimeout(webhook.url, {
      method: "POST",
      headers: requestHeaders,
      body,
    })
  )
  .catch((error) => {
    logger.error({ error, url: request.url }, `Webhook call to ${webhook.url} failed`);
  });
```

**失败处理逻辑**：
1. **URL 验证失败**：被 catch 捕获，记录错误日志
2. **网络请求失败**：被 catch 捕获，记录错误日志
3. **超时（5秒）**：触发 Error("Timeout")，被 catch 捕获，记录错误日志
4. **HTTP 非 2xx 状态码**：**不会触发 catch**，仅静默忽略（fetch 仅在网络错误时 reject）

⚠️ **重要**：无任何退避或重试逻辑，投递失败即永久丢失。

---

### 3.3 并发控制机制

**【已上线实现】** ✅

使用 `Promise.allSettled` 并行投递所有匹配的 webhook，单个失败不影响其他 webhook 投递。

**证据来源**：
- 代码位置：`apps/web/app/api/(internal)/pipeline/route.ts:285, 293`

```typescript
// responseFinished 事件：并行执行 webhook + 邮件
const results = await Promise.allSettled([...webhookPromises, ...emailPromises]);

// 其他事件：仅并行执行 webhook
const results = await Promise.allSettled(webhookPromises);
```

---

### 3.4 请求超时保护

**【已上线实现】** ✅

使用 `Promise.race` 实现 5 秒超时保护，防止请求挂起耗尽连接池。

**证据来源**：
- 代码位置：`apps/web/app/api/(internal)/pipeline/route.ts:101-106`

```typescript
const fetchWithTimeout = (url: string, options: RequestInit, timeout: number = 5000): Promise<Response> => {
  return Promise.race([
    fetch(url, { ...options, redirect: redirectMode }),
    new Promise<never>((_, reject) => setTimeout(() => reject(new Error("Timeout")), timeout)),
  ]);
};
```

---

### 3.5 退避策略现状

**【关键结论：无退避策略】** ❌

当前代码库中**不存在任何形式的退避实现**：
- 无固定间隔重试
- 无指数退避
- 无抖动策略

---

## 四、完整投递链路

### 4.1 触发入口

**【已上线实现】** ✅

当用户提交问卷响应时，响应处理程序调用 `sendToPipeline` 触发 webhook 投递。

**证据来源**：
- 触发位置：`apps/web/app/api/v1/client/[environmentId]/responses/[responseId]/lib/put-response-handler.ts:257-271`
- 封装位置：`apps/web/app/lib/pipelines.ts:5-25`

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

---

### 4.2 触发事件类型

**【已上线实现】** ✅

支持三种触发事件：
- `responseCreated` - 响应创建时
- `responseUpdated` - 响应更新时
- `responseFinished` - 响应完成时

---

### 4.3 完整投递流程图

**【已上线实现】** ✅

```
用户提交响应
    ↓
[API 处理程序] put-response-handler.ts
    ↓
[调用] sendToPipeline()  ────────────┐
    │                                │
    ↓                                │
[内部 HTTP POST] /api/pipeline      │
    │  ← x-api-key 认证              │
    │                                │
    ├────────────────────────────────┘
    │  ⚠️  火并忘模式：发送方不等待响应，仅 catch 网络错误
    ↓
[查询匹配 webhook] prisma.webhook.findMany()
    条件：environmentId 匹配 + triggers 包含事件 + surveyIds 匹配或为空
    ↓
[加载问卷信息] getSurvey()
    ↓
[对每个 webhook 构造 Promise]
    ├─ 生成 webhook-id (UUID v7)
    ├─ 生成 webhook-timestamp (秒级)
    ├─ 构造 JSON payload
    ├─ 如配置密钥，生成签名
    └─ validateWebhookUrl() → fetchWithTimeout(5s)
    ↓
[并发执行] Promise.allSettled()
    ├─ 成功：无额外操作
    └─ 失败：仅记录 error 日志，无重试
    ↓
[继续执行其他管道逻辑]
    ├─ responseFinished: 处理集成、发送邮件、更新问卷状态等
    ├─ responseCreated: 计费埋点、遥测
    └─ responseUpdated: 无额外逻辑
```

---

### 4.4 Webhook 匹配逻辑

**【已上线实现】** ✅

**证据来源**：
- 代码位置：`apps/web/app/api/(internal)/pipeline/route.ts:81-90`

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

**匹配条件**：
1. `environmentId` 匹配 - 必须属于当前环境
2. `triggers` 包含当前事件 - 事件类型匹配
3. `surveyIds` 包含当前问卷 ID 或为空 - 全局或特定问卷配置

---

### 4.5 Payload 结构

**【已上线实现】** ✅

**证据来源**：
- 代码位置：`apps/web/app/api/(internal)/pipeline/route.ts:110-125`

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

**【已上线实现】** ✅

**证据来源**：
- 代码位置：`apps/web/app/api/(internal)/pipeline/route.ts:95-100, 147`

```typescript
const redirectMode: RequestRedirect = DANGEROUSLY_ALLOW_WEBHOOK_INTERNAL_URLS ? "follow" : "manual";
```

- 默认模式：`redirect: "manual"` - 禁止重定向
- 目的：防止通过 3xx 重定向访问内网资源（如云元数据服务）
- 投递前调用 `validateWebhookUrl` 验证 URL 安全性，禁止访问内网 IP 段
- 自托管用户可通过环境变量启用重定向跟随

---

### 5.2 超时保护

**【已上线实现】** ✅

5 秒超时防止挂起请求耗尽连接池。

---

### 5.3 并行隔离

**【已上线实现】** ✅

使用 `Promise.allSettled` 并行投递，单个失败不影响其他 webhook。

---

## 六、可观测性

### 6.1 日志记录

**【已上线实现】** ✅

失败时仅记录结构化日志：

```typescript
logger.error(
  { error, url: request.url },
  `Webhook call to ${webhook.url} failed`
);
```

---

### 6.2 测试端点

**【已上线实现】** ✅

提供 webhook 测试功能，支持：
- URL 可访问性验证
- 签名生成测试
- 超时测试

**证据来源**：
- 代码位置：`apps/web/modules/integrations/webhooks/lib/webhook.ts:164-240`

---

## 七、关键入口文件汇总

| 文件路径 | 职责 | 状态 |
|----------|------|------|
| `apps/web/lib/crypto.ts` | 签名生成、密钥管理 | ✅ 已实现 |
| `apps/web/app/api/(internal)/pipeline/route.ts` | 核心投递逻辑 | ✅ 已实现 |
| `apps/web/app/lib/pipelines.ts` | 触发入口封装（fire-and-forget） | ✅ 已实现 |
| `apps/web/modules/integrations/webhooks/lib/webhook.ts` | Webhook CRUD 与测试 | ✅ 已实现 |
| `@formbricks/jobs` 包 | BullMQ 队列实现 | ❌ 不存在（仅文档提及） |
| `response-pipeline` 模块 | 后台任务处理器 | ❌ 不存在（仅文档提及） |
| `instrumentation-jobs.ts` | Worker 启动注册 | ❌ 不存在（仅文档提及） |

---

## 八、总结与差距分析

### 当前实现状态总结

| 功能 | 实现状态 | 说明 |
|------|----------|------|
| Standard Webhooks 签名 | ✅ 完整实现 | HMAC-SHA256 + 时间戳 + 消息 ID |
| 时间戳防重放 | ⚠️ 半实现 | 服务端生成时间戳，接收方需自行验证 |
| 失败重试 | ❌ 未实现 | 仅 catch 记录日志，无重试机制 |
| 退避策略 | ❌ 未实现 | 无任何退避逻辑 |
| SSRF 防护 | ✅ 完整实现 | 重定向禁止 + URL 验证 |
| 超时保护 | ✅ 完整实现 | 5 秒超时 |
| 并发隔离 | ✅ 完整实现 | Promise.allSettled |
| 异步队列 | ❌ 未实现 | 同步 HTTP 调用，无 BullMQ |

### 文档与代码差距

**背景文档中描述但未实际实现的功能**：

1. ❌ BullMQ 队列系统（`background-jobs` 队列）
2. ❌ 3 次重试 + 指数退避策略
3. ❌ 响应管道异步处理器（`processResponsePipelineJob`）
4. ❌ 运行时启动失败自动重试
5. ❌ 结构化失败元数据记录（jobId、attempt 等）

### 风险提示

⚠️ **当前架构存在的问题**：
1. **同步阻塞**：webhook 投递在请求线程中执行，慢端点会拖慢 API 响应
2. **火并忘模式**：`sendToPipeline` 不 await 结果，仅 catch 网络错误，API 路由层面的投递失败无法感知
3. **无重试机制**：网络抖动、接收方临时不可用都会导致永久投递失败
4. **HTTP 状态码忽略**：fetch 仅在网络错误时 reject，4xx/5xx 状态码不会被捕获
5. **可观测性不足**：仅记录日志，无失败统计、告警机制
