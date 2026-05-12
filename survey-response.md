# Formbricks 答卷提交流程技术分析报告

## 一、提交校验

### 1.1 前端队列与离线存储机制

**核心文件**：`packages/surveys/src/lib/response-queue.ts`

#### 1.1.1 ResponseQueue 初始化配置

```typescript
interface QueueConfig {
  appUrl: string;
  environmentId: string;
  retryAttempts: number;
  persistOffline?: boolean;  // 关键：是否启用离线持久化
  surveyId?: string;
  onResponseCreated?: (responseId: string) => void;
  onResponseSendingFailed?: (...);
  onResponseSendingFinished?: () => void;
  onQuotaFull?: (...);
  setSurveyState?: (state: SurveyState) => void;
}
```

#### 1.1.2 答卷添加流程（真实调用顺序）

**Step 1**：调用 `ResponseQueue.add(responseUpdate)`

```typescript
add(responseUpdate: TResponseUpdate) {
  // 1. 更新 SurveyState，累加答题数据
  this.surveyState.accumulateResponse(responseUpdate);
  // 2. 可选：回调通知上层状态变更
  if (this.config.setSurveyState) {
    this.config.setSurveyState(this.surveyState);
  }
  // 3. 加入内存队列
  this.queue.push(responseUpdate);

  // 4. 离线持久化分支判断
  if (this.config.persistOffline && this.config.surveyId) {
    // 写入 IndexedDB，成功后才开始处理队列
    addPendingResponse({
      surveyId: this.config.surveyId,
      responseUpdate,
      surveyStateSnapshot: this.serializeSurveyState(),
      createdAt: Date.now(),
    }).then((dbId) => {
      if (dbId > 0) {
        this.pendingDbIds.set(responseUpdate, dbId);
      }
      this.processQueue();
    });
  } else {
    // 无离线模式：直接处理
    this.processQueue();
  }
}
```

**关键判断条件**：
- `persistOffline === true` 且 `surveyId` 存在 → 先持久化到 IndexedDB
- 否则 → 直接发送

#### 1.1.3 离线存储实现细节

**核心文件**：`packages/surveys/src/lib/offline-storage.ts`

IndexedDB 结构：
- 数据库名：`formbricks-offline`
- 对象存储：`pendingResponses`（待提交答卷）、`surveyProgress`（答题进度）

持久化数据结构：
```typescript
interface PendingResponseEntry {
  id?: number;                    // 自增主键
  surveyId: string;
  responseUpdate: TResponseUpdate; // 答卷增量更新
  surveyStateSnapshot: SerializedSurveyState;  // 完整状态快照
  createdAt: number;
}
```

#### 1.1.4 队列并发控制

```typescript
// 模块级锁（跨实例共享）
const syncingBySurvey = new Map<string, boolean>();
const requestInProgressBySurvey = new Map<string, boolean>();

async processQueue(): Promise<{ success: boolean }> {
  // 并发守卫 1：正在请求中 或 队列为空
  if (this.isRequestInProgress || this.queue.length === 0) {
    return { success: false };
  }

  // 并发守卫 2：离线且开启持久化 → 不发送
  if (this.config.persistOffline && typeof navigator !== "undefined" && !navigator.onLine) {
    return { success: false };
  }

  // 并发守卫 3：正在同步 IndexedDB → 退出
  if (this.isSyncing) {
    return { success: false };
  }

  // 并发守卫 4：离线模式且待提交数 > 1 → 转由 syncPersistedResponses 批量处理
  if (this.config.persistOffline && this.config.surveyId) {
    const pendingCount = await countPendingResponses(this.config.surveyId);
    
    // 二次检查：await 期间状态可能变化
    if (this.isSyncing || this.isRequestInProgress || this.queue.length === 0) {
      return { success: false };
    }

    if (pendingCount > 1) {
      void this.syncPersistedResponses();  // 异步启动，不等待
      return { success: false };
    }
  }

  // 正式发送
  const responseUpdate = this.queue[0];
  this.isRequestInProgress = true;
  const result = await this.sendResponseWithRetry(responseUpdate);
  // ...后续处理
}
```

### 1.2 重试机制

```typescript
private async sendResponseWithRetry(responseUpdate: TResponseUpdate) {
  let attempts = 0;
  
  while (attempts < this.config.retryAttempts) {
    const res = await this.sendResponse(responseUpdate);
    
    if (res.ok) {
      this.queue.shift();  // 发送成功，移出队列
      
      // 检查配额满情况
      if (this.isQuotaFullResponse(res.data)) {
        // 处理配额逻辑
      }
      
      return { success: true };
    }

    // reCAPTCHA 验证失败：直接返回，不重试
    if (this.isRecaptchaError(res.error)) {
      console.error("Formbricks: Recaptcha verification failed");
      return { success: false, isRecaptchaError: true };
    }

    // 指数退避：1s → 2s → 4s → 8s
    const backoffMs = 1000 * Math.pow(2, attempts);
    await delay(backoffMs);
    attempts++;
  }
  
  return { success: false, isRecaptchaError: false };
}
```

**关键判断条件**：
- reCAPTCHA 验证失败 → **立即终止重试**
- 其他错误 → 指数退避重试

---

## 二、问卷版本绑定与数据持久化

### 2.1 HTTP 请求发送

**核心文件**：`packages/surveys/src/lib/api-client.ts`

```typescript
async createResponse(responseInput: TResponseInput) {
  const fromV1 = !!responseInput.userId;
  
  // 根据 userId 存在与否决定调用 v1 或 v2 API
  return makeRequest(
    this.appUrl,
    `/api/${fromV1 ? "v1" : "v2"}/client/${this.environmentId}/responses`,
    "POST",
    responseInput
  );
}
```

**API 版本选择判断**：
- `userId` 存在 → 调用 `/api/v1/client/{envId}/responses`
- `userId` 不存在 → 调用 `/api/v2/client/{envId}/responses`

### 2.2 后端校验流程

**核心文件**：`apps/web/app/api/v2/client/[environmentId]/responses/route.ts`

#### 2.2.1 请求入口与参数解析

**POST 处理流程（真实调用顺序）**：

```
Step 1: parseAndValidateResponseInput()
  ├─ ZEnvironmentId 校验 environmentId 格式
  ├─ parseAndValidateJsonBody 解析 JSON
  └─ ZResponseInputV2 校验请求体
     ├─ surveyId: cuid2 格式
     ├─ contactId: cuid2 | null | undefined
     ├─ finished: boolean
     ├─ data: ZResponseData
     └─ recaptchaToken: string | null | undefined
  → 校验失败：返回 400 Bad Request

Step 2: getContactsDisabledResponse()
  ├─ 若 contactId 存在
  │   ├─ 查询 organizationId
  │   └─ getIsContactsEnabled() 检查企业版功能
  └─ 未启用 → 返回 403 Forbidden

Step 3: getSurvey(surveyId)
  └─ 不存在 → 返回 404 Not Found

Step 4: validateResponseSubmission() 详细校验
  ├─ checkSurveyValidity() 问卷状态校验
  │   ├─ 校验 survey.environmentId == 请求 environmentId → 不匹配 400
  │   ├─ 链接问卷 + 单次使用模式：
  │   │   ├─ singleUseId 必须存在 → 缺失 400
  │   │   ├─ meta.url 必须存在 → 缺失 400
  │   │   ├─ URL 必须包含 suId 查询参数 → 缺失 400
  │   │   └─ 加密模式下解密验证 → 不匹配 400
  │   └─ reCAPTCHA 校验：
  │       ├─ recaptchaToken 必须存在 → 缺失 400
  │       └─ verifyRecaptchaToken() → 验证失败 400
  │
  ├─ validateOtherOptionLengthForMultipleChoice()
  │   └─ 多选题"其他"选项字符限制
  │
  └─ validateResponseData() 答题数据校验
      ├─ 提取所有 survey blocks 中的 elements
      ├─ 筛选出 responseData 中存在的 elementId
      └─ 调用 @formbricks/surveys/validation 的 validateBlockResponses()
          → 校验必填、范围、正则等规则
  → 任一校验失败：返回 400 Bad Request
```

### 2.3 数据持久化实现

**核心文件**：`apps/web/app/api/v2/client/[environmentId]/responses/lib/response.ts`

#### 2.3.1 事务创建流程

```typescript
export const createResponseWithQuotaEvaluation = async (responseInput: TResponseInputV2) => {
  // 使用 Prisma 事务保证原子性
  return prisma.$transaction(async (tx) => {
    // Step 1: 创建 Response 记录
    const response = await createResponse(responseInput, tx);
    
    // Step 2: 评估配额规则
    const quotaResult = await evaluateResponseQuotas({
      surveyId: responseInput.surveyId,
      responseId: response.id,
      data: responseInput.data,
      variables: responseInput.variables,
      language: responseInput.language,
      responseFinished: response.finished,
      tx,
    });

    return {
      ...response,
      ...(quotaResult.quotaFull && { quotaFull: quotaResult.quotaFull }),
    };
  });
};
```

#### 2.3.2 Response 记录创建细节

```typescript
const buildPrismaResponseData = (
  responseInput: TResponseInputV2,
  contact: { id: string; attributes: TContactAttributes } | null,
  ttc: Record<string, number>
): Prisma.ResponseCreateInput => {
  return {
    // 问卷版本绑定：仅通过 surveyId 关联（无版本快照）
    survey: { connect: { id: responseInput.surveyId } },
    
    // 其他关联
    display: responseInput.displayId ? { connect: { id: responseInput.displayId } } : undefined,
    contact: contact?.id ? { connect: { id: contact.id } } : undefined,
    
    // 核心数据
    finished: responseInput.finished,
    data: responseInput.data,           // 答题数据 JSON
    language: responseInput.language,
    contactAttributes: contact?.attributes,
    meta: responseInput.meta as Prisma.JsonObject,  // UA、IP、来源等
    singleUseId: responseInput.singleUseId,
    variables: responseInput.variables,  // 问卷变量
    
    // TTC (Time To Complete) 处理
    ttc: ttc,
    
    // 可覆盖的创建/更新时间（用于离线同步）
    createdAt: responseInput.createdAt,
    updatedAt: responseInput.updatedAt,
  };
};
```

**关键注意：版本信息关联方式**

> ⚠️ **重要发现**：Formbricks 当前实现中 **不存在问卷版本绑定机制**
>
> - Response 表仅存储 `surveyId` 字段，无 `surveyVersionId` 或快照字段
> - 答卷与问卷的关联是**动态的、指向最新版本**的
> - 若问卷在答题过程中被修改，历史答卷将映射到新版问卷配置
> - 如需版本追溯，需额外在 `meta` 字段中存储自定义版本标识

#### 2.3.3 TTC（答题时长）计算

```typescript
// 仅在 finished = true 时计算总时长
const ttc = initialTtc ? (finished ? calculateTtcTotal(initialTtc) : initialTtc) : {};

// calculateTtcTotal 会在 ttc 对象中添加 total 字段
// 格式：{ [questionId]: durationMs, "total": totalMs }
```

### 2.4 数据写入时机

**写入数据库的准确时机**：

1. **首次提交**（responseId === null）：
   - 调用 `apiClient.createResponse()`
   - 后端 `createResponseWithQuotaEvaluation()` 执行事务
   - `prisma.response.create()` 写入数据库
   - 返回 `{ id: string, ... }`
   - 前端更新 `surveyState.responseId = id`

2. **后续更新**（responseId !== null）：
   - 调用 `apiClient.updateResponse()`
   - 后端执行 `prisma.response.update()` 合并数据
   - 相同 ID 记录被更新，不创建新记录

---

## 三、下游事件分发

### 3.1 分发触发时机与条件

**核心文件**：`apps/web/app/api/v2/client/[environmentId]/responses/route.ts`

```typescript
// 写入数据库成功后执行
const createdResponse = await createResponseForRequest({...});

// ===== 事件 1：responseCreated =====
// 触发条件：每次成功创建 Response 后（无论是否完成）
sendToPipeline({
  event: "responseCreated",
  environmentId,
  surveyId: responseData.surveyId,
  response: responseData,
});

// ===== 事件 2：responseFinished =====
// 触发条件：仅当 finished === true 时
if (responseData.finished) {
  sendToPipeline({
    event: "responseFinished",
    environmentId,
    surveyId: responseData.surveyId,
    response: responseData,
  });
}
```

**关键判断条件**：
- `responseCreated`：**每次创建都触发**（包括未完成的草稿答卷）
- `responseFinished`：**仅在 finished = true 时触发**（用户完成整个问卷）

### 3.2 sendToPipeline 实现细节

**核心文件**：`apps/web/app/lib/pipelines.ts`

```typescript
export const sendToPipeline = async ({ event, surveyId, environmentId, response }: TPipelineInput) => {
  if (!CRON_SECRET) {
    throw new Error("CRON_SECRET is not set");
  }

  // 异步调用内部 Pipeline API，不等待响应
  return fetch(`${WEBAPP_URL}/api/pipeline`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "x-api-key": CRON_SECRET,  // 内部服务鉴权
    },
    body: JSON.stringify({
      environmentId,
      surveyId,
      event,
      response,
    }),
  }).catch((error) => {
    // 静默失败：仅记录日志，不影响主流程
    logger.error(error, "Error sending event to pipeline");
  });
};
```

**设计特性**：
- **非阻塞**：`fetch` 调用无 `await`，请求发出后立即返回
- **容错性**：`.catch()` 捕获异常，不向上层抛出
- **解耦**：通过 HTTP 调用内部端点，业务逻辑与下游处理分离

---

## 四、完整端到端流程

### 4.1 在线模式（标准流程）

```
用户填写问题
    ↓
[前端] SurveyState.accumulateResponse()
    ↓
[前端] ResponseQueue.add()
    ├─ 加入内存队列
    └─ processQueue()
        └─ 并发检查通过
            ↓
[前端] sendResponseWithRetry()
    └─ 首次提交：responseId === null
        ↓
[HTTP] POST /api/v2/client/{envId}/responses
    ├─ environmentId 校验
    ├─ survey 查询
    ├─ checkSurveyValidity()
    ├─ 答题数据 validateResponseData()
    ├─ prisma.$transaction 事务创建
    │   ├─ Response 记录写入（surveyId 关联）
    │   └─ 配额评估
    ├─ sendToPipeline("responseCreated")
    └─ [finished=true] sendToPipeline("responseFinished")
        ↓
[前端] surveyState.updateResponseId(id)
```

### 4.2 离线模式（断网场景）

```
用户断网下填写问卷
    ↓
[前端] ResponseQueue.add()
    ├─ SurveyState 累加
    ├─ 加入内存队列
    └─ addPendingResponse() 写入 IndexedDB
        ↓
网络恢复 / 用户重新打开页面
    ↓
[前端] syncPersistedResponses() 启动
    ├─ getPendingResponses(surveyId) 按时间排序读取
    ├─ 逐条发送：
    │   ├─ 有 responseId → PUT /api/v2/client/{envId}/responses/{id}
    │   └─ 无 responseId → POST 创建
    │   └─ 404 重试：update 返回 404 说明服务端未收到
    │       → 重置 responseId = null
    │       → 重新 POST 创建
    ├─ 发送成功：removePendingResponse(id)
    └─ 409（已完成）：直接删除本地记录
```

---

## 五、关键设计要点总结

### 5.1 关于问卷版本

| 设计点 | 现状说明 |
|-------|---------|
| 版本绑定 | ❌ 无显式版本绑定，仅通过 surveyId 动态关联 |
| 快照存储 | ❌ 未存储答题时的问卷配置快照 |
| 追溯能力 | ⚠️ 历史答卷的字段含义依赖于当前问卷配置 |

### 5.2 事件触发矩阵

| 事件 | 触发时机 | 条件 |
|-----|---------|------|
| responseCreated | 首次写入数据库后 | 总是触发 |
| responseFinished | 写入数据库后 | 仅 `finished = true` |

### 5.3 并发控制机制

| 锁名称 | 作用域 | 用途 |
|-------|-------|------|
| requestInProgressBySurvey | 模块级 | 防止同一问卷并发发送 |
| syncingBySurvey | 模块级 | 防止离线同步与在线发送冲突 |
| Prisma 事务 | 数据库级 | 答卷创建与配额评估原子性 |

---

*报告生成时间：2026-05-12*
*基于代码版本：Formbricks 当前主分支*
