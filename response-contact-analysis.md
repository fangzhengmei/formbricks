# 问卷答案落库与联系人画像属性回写字段链路分析

## 概述

本文档从代码实现角度梳理问卷答案（Response）落库后，将答案中的属性写回联系人画像（Contact Profile）的完整技术链路。包括**写入顺序**、**字段映射关系**和**失败回滚与事务处理机制**三个核心部分。

---

## 一、写入顺序（Write Order）

整个链路分为**同步阶段**、**异步管道阶段**和**联系人属性更新阶段**三个部分。

### 1. 同步阶段：Response 落库（HTTP 请求线程）

#### 入口
- API 路由：`apps/web/app/api/v2/client/[workspaceId]/responses/route.ts`
- 支持 POST（创建）和 PUT（更新）

#### 核心事务处理
```typescript
// apps/web/app/api/v2/client/[workspaceId]/responses/lib/response.ts:32
export const createResponseWithQuotaEvaluation = async (
  responseInput: TResponseInputV2
): Promise<TResponseWithQuotaFull> => {
  const txResponse = await prisma.$transaction(async (tx) => {
    // 步骤1: 创建 Response 记录
    const response = await createResponse(responseInput, tx);
    
    // 步骤2: 评估配额
    const quotaResult = await evaluateResponseQuotas({...});
    
    return { ...response, ...(quotaResult.quotaFull && { quotaFull: quotaResult.quotaFull }) };
  });
  return txResponse;
};
```

#### createResponse 内部逻辑
```typescript
// apps/web/app/api/v2/client/[workspaceId]/responses/lib/response.ts:58
const createResponse = async (
  responseInput: TResponseInputV2,
  tx: PrismaClient | PrismaTransactionClient
) => {
  // 获取联系人信息（如果有 contactId）
  const contact = responseInput.contactId
    ? await getContactForResponse(responseInput.contactId, responseInput.workspaceId)
    : null;

  // 构建 Prisma 数据
  const data = buildPrismaResponseData(responseInput, contact, ttc);
  // 注意: contactAttributes 是联系人当前属性的**快照**，只读不写
  
  return tx.response.create({ data });
};
```

#### 关键要点
1. **事务原子性**：Response 创建和配额评估在同一个 `prisma.$transaction` 中
2. **联系人属性快照**：如果有 `contactId`，会**读取**当前联系人属性并保存到 `response.contactAttributes` 字段中
3. **⚠️ 重要发现**：此阶段**不会**把问卷答案中的 ContactInfo 字段写回联系人画像，只是**读取快照**

#### 触发异步管道
```typescript
// apps/web/app/api/v2/client/[workspaceId]/responses/route.ts
// Response 创建/更新成功后，调用 sendToPipeline 加入队列
await sendToPipeline({
  event: response.finished ? "responseFinished" : "responseCreated",
  responseId: response.id,
  surveyId: response.surveyId,
  workspaceId: responseInput.workspaceId,
  response: {...},
});
```

### 2. 异步阶段：Response Pipeline（BullMQ 队列）

#### 任务入队
```typescript
// apps/web/app/lib/pipelines.ts:5
export const sendToPipeline = async (job: TResponsePipelineJobData): Promise<void> => {
  const producer = getBackgroundJobProducer();
  await producer.enqueueResponsePipeline(job);
};
```

#### 任务处理入口
```typescript
// apps/web/modules/response-pipeline/lib/process-response-pipeline-job.ts:720
export const processResponsePipelineJob: JobHandler<TResponsePipelineJobData> = async (data, context) => {
  // 根据事件类型执行不同副作用
  if (data.event === "responseFinished") {
    await runResponseFinishedSideEffects({...});
  }
  if (data.event === "responseCreated") {
    await runResponseCreatedSideEffects({...});
  }
};
```

#### 副作用处理顺序
| 事件类型 | 处理内容 |
|---------|---------|
| `responseCreated` | 1. 计量事件记录（Stripe）<br>2. PostHog 事件捕获<br>3. 遥测事件发送 |
| `responseFinished` | 1. Webhook 分发<br>2. 集成处理（Google Sheets, Slack, Airtable, Notion）<br>3. 连接器管道<br>4. 后续动作（Follow-ups）<br>5. 通知邮件<br>6. 调查自动完成 |

#### ⚠️ 关键发现
**在异步管道的标准处理流程中，没有自动更新联系人属性的逻辑。** 需要通过以下方式之一实现：
- 配置 Webhook 由外部系统处理
- 使用集成（如 Make.com, Zapier）
- 前端 SDK 主动调用更新接口

### 3. 联系人属性更新阶段（需显式触发）

#### 核心更新函数
```typescript
// apps/web/modules/ee/contacts/lib/attributes.ts:108
export const updateAttributes = async (
  contactId: string,
  userId: string,
  workspaceId: string,
  contactAttributesParam: TContactAttributesInput,
  deleteRemovedAttributes: boolean = false
): Promise<{
  success: boolean;
  messages?: TAttributeUpdateMessage[];
  errors?: TAttributeUpdateMessage[];
}> => {
  // 1. 输入验证
  // 2. 检查 email/userId 唯一性
  // 3. 删除移除的属性（仅当 deleteRemovedAttributes = true）
  // 4. 更新现有属性（事务批量 upsert）
  // 5. 创建新属性（事务批量创建）
};
```

#### 包装函数（UI 层使用）
```typescript
// apps/web/modules/ee/contacts/lib/update-contact-attributes.ts:19
export const updateContactAttributes = async (
  contactId: string,
  attributes: TContactAttributesInput
): Promise<UpdateContactAttributesResult> => {
  const contact = await getContact(contactId);
  // 调用 updateAttributes，deleteRemovedAttributes = true
  const updateResult = await updateAttributes(contactId, userId, workspaceId, attributes, true);
  const updatedAttributes = await getContactAttributes(contactId);
  return { updatedAttributes, messages: updateResult.messages };
};
```

### 4. Response 更新流程（responseUpdated 和 responseFinished）

#### v1 Response 更新路由
```typescript
// apps/web/app/api/v1/client/[workspaceId]/responses/[responseId]/route.ts
export const PUT = withV1ApiWrapper({
  handler: putResponseHandler,
});
```

#### putResponseHandler 处理流程
```typescript
// apps/web/app/api/v1/client/[workspaceId]/responses/[responseId]/lib/put-response-handler.ts:207
export const putResponseHandler = async ({ req, props }: THandlerParams<TPutRouteParams>) => {
  // 1. 验证输入
  const validatedUpdateInput = await getValidatedResponseUpdateInput(req);
  
  // 2. 获取现有 Response
  const existingResponse = await getResponse(responseId);
  
  // 3. 获取 Survey 并验证
  const survey = await getSurvey(existingResponse.surveyId);
  
  // 4. 更新 Response（带配额评估）
  const updatedResponse = await updateResponseWithQuotaEvaluation(responseId, responseUpdateInput);
  
  // 5. 发送 responseUpdated 事件到管道
  await sendToPipeline({ event: "responseUpdated", ... });
  
  // 6. 如果 finished=true，额外发送 responseFinished 事件
  if (updatedResponse.finished) {
    await sendToPipeline({ event: "responseFinished", ... });
  }
};
```

#### updateResponseWithQuotaEvaluation 事务处理
```typescript
// apps/web/app/api/v1/client/[workspaceId]/responses/[responseId]/lib/response.ts:7
export const updateResponseWithQuotaEvaluation = async (
  responseId: string,
  responseInput: TResponseUpdateInput
): Promise<TResponseWithQuotaFull> => {
  const txResponse = await prisma.$transaction(async (tx) => {
    // 步骤1: 更新 Response 记录
    const response = await updateResponse(responseId, responseInput, tx);
    
    // 步骤2: 评估配额
    const quotaResult = await evaluateResponseQuotas({...});
    
    return { ...response, ...(quotaResult.quotaFull && { quotaFull: quotaResult.quotaFull }) };
  });
  return txResponse;
};
```

#### updateResponse 数据合并逻辑
```typescript
// apps/web/lib/response/service.ts:507
export const updateResponse = async (
  responseId: string,
  responseInput: TResponseUpdateInput,
  tx?: Prisma.TransactionClient
): Promise<TResponse> => {
  // 1. 获取当前 Response
  const currentResponse = await prismaClient.response.findUnique({...});
  
  // 2. 合并 data 对象（浅合并）
  const data = {
    ...currentResponse.data,
    ...responseInput.data,
  };
  
  // 3. 合并 ttc 对象
  const mergedTtc = responseInput.ttc
    ? { ...currentTtc, ...responseInput.ttc }
    : currentTtc;
  
  // 4. 合并 variables 对象
  const variables = {
    ...currentResponse.variables,
    ...responseInput.variables,
  };
  
  // 5. 更新数据库
  const responsePrisma = await prismaClient.response.update({
    where: { id: responseId },
    data: {
      finished: responseInput.finished,
      endingId: responseInput.endingId,
      data,
      ttc: responseInput.finished ? calculateTtcTotal(mergedTtc) : mergedTtc,
      language: responseInput.language,
      variables,
    },
    select: responseSelection,
  });
  
  return response;
};
```

#### 触发入口与数据形态

| 触发方式 | 事件类型 | 数据形态 | 入库位置 |
|---------|---------|---------|---------|
| PUT /api/v1/client/[workspaceId]/responses/[responseId] | responseUpdated | `TResponseUpdateInput` - 包含 data, finished, ttc, variables, language, endingId | `prisma.response.update` - 合并更新 |
| PUT /api/v1/client/[workspaceId]/responses/[responseId]（finished=true） | responseFinished | 同上 | 同上 |
| POST /api/v2/client/[workspaceId]/responses（finished=true） | responseCreated + responseFinished | `TResponseInputV2` - 包含完整 Response 数据 | `prisma.response.create` - 新建记录 |

### 5. /user 接口（v1 和 v2 版本）

#### v1 /user 接口路由
```typescript
// apps/web/modules/ee/contacts/api/v1/client/[workspaceId]/user/route.ts:40
export const POST = withV1ApiWrapper({
  handler: async ({ req, props }: THandlerParams<{ params: Promise<{ workspaceId: string }> }>) => {
    // 1. 解析输入
    const { userId, attributes } = jsonInput;
    
    // 2. 检查企业版许可证
    const isContactsEnabled = await getIsContactsEnabled(organizationId);
    
    // 3. 调用 updateUser
    const { state, messages, errors } = await updateUser(workspaceId, userId, deviceType, attributes);
    
    // 4. 返回用户状态
    return { response: responses.successResponse({ state, messages, errors }, true) };
  },
});
```

#### v2 /user 接口路由
```typescript
// apps/web/app/api/v2/client/[workspaceId]/user/route.ts:1
import { OPTIONS, POST } from "@/modules/ee/contacts/api/v1/client/[workspaceId]/user/route";
export { POST, OPTIONS };
```

**⚠️ 关键发现**：v2 /user 接口只是转发到 v1 的实现，两者使用相同的后端逻辑。

#### updateUser 核心逻辑
```typescript
// apps/web/modules/ee/contacts/api/v1/client/[workspaceId]/user/lib/update-user.ts:132
export const updateUser = async (
  workspaceId: string,
  userId: string,
  device: "phone" | "desktop",
  attributes?: TContactAttributesInput
): Promise<{ state: TJsPersonState; messages?: string[]; errors?: string[] }> => {
  // 1. 获取或创建联系人
  let contactData = await getContactWithFullData(workspaceId, userId);
  if (!contactData) {
    contactData = await createContact(workspaceId, userId);
  }
  
  // 2. 处理属性更新
  if (attributes && Object.keys(attributes).length > 0) {
    // 检查是否有变化
    const hasChanges = Object.entries(attributes).some(
      ([key, value]) => value !== contactAttributes[key]
    );
    
    if (hasChanges) {
      // 调用 updateAttributes 更新联系人属性
      const { success, messages, errors } = await updateAttributes(
        contactData.id, userId, workspaceId, attributes
      );
    }
  }
  
  // 3. 构建用户状态（包含 segments, displays, responses 等）
  const userState = await buildUserStateFromContact(contactData, workspaceId, userId, device);
  
  return {
    state: {
      data: { ...userState, language },
      expiresAt: new Date(Date.now() + 1000 * 60 * 30), // 30 分钟
    },
    messages,
    errors,
  };
};
```

#### getContactWithFullData 一次查询获取所有数据
```typescript
// apps/web/modules/ee/contacts/api/v1/client/[workspaceId]/user/lib/update-user.ts:11
const getContactWithFullData = async (workspaceId: string, userId: string) => {
  return prisma.contact.findFirst({
    where: {
      workspaceId,
      attributes: {
        some: {
          attributeKey: { key: "userId", workspaceId },
          value: userId,
        },
      },
    },
    select: {
      id: true,
      attributes: { select: { attributeKey: { select: { key: true } }, value: true } },
      responses: { select: { surveyId: true } },
      displays: { select: { surveyId: true, createdAt: true }, orderBy: { createdAt: "desc" } },
    },
  });
};
```

#### 触发入口与数据形态

| 触发方式 | HTTP 方法 | 数据形态 | 入库位置 |
|---------|----------|---------|---------|
| JS SDK setUserId | POST /api/v2/client/[workspaceId]/user | `{ userId: string, attributes?: Record<string, string | number> }` | `prisma.contact.findFirst` → `prisma.contact.create` / `updateAttributes` |
| JS SDK setAttributes | POST /api/v2/client/[workspaceId]/user | 同上 | `updateAttributes` - 两个独立事务 |
| 直接 API 调用 | POST /api/v1/client/[workspaceId]/user | 同上 | 同上 |

### 6. JS SDK UpdateQueue 触发链路

#### UpdateQueue 单例设计
```typescript
// packages/js-core/src/lib/user/update-queue.ts:7
export class UpdateQueue {
  private static instance: UpdateQueue | null = null;
  private updates: TUpdates | null = null;
  private debounceTimeout: NodeJS.Timeout | null = null;
  private pendingFlush: Promise<void> | null = null;
  private readonly DEBOUNCE_DELAY = 500; // 500ms 防抖
  private readonly PENDING_WORK_TIMEOUT = 5000; // 5秒超时
}
```

#### 触发入口 1：setUserId
```typescript
// packages/js-core/src/lib/user/user.ts:8
export const setUserId = async (userId: string): Promise<Result<void, ApiErrorResponse>> => {
  const updateQueue = UpdateQueue.getInstance();
  
  // 如果是不同的 userId，先清理之前的状态
  if (currentUserId && currentUserId !== userId) {
    tearDown();
  }
  
  // 更新队列中的 userId
  updateQueue.updateUserId(userId);
  
  // 触发处理
  void updateQueue.processUpdates();
  
  return okVoid();
};
```

#### 触发入口 2：setAttributes
```typescript
// packages/js-core/src/lib/user/attribute.ts:18
export const setAttributes = async (
  attributes: Record<string, string | number | Date>
): Promise<Result<void, NetworkError>> => {
  // 规范化值：Date → ISO 字符串
  const normalizedAttributes: Record<string, string | number> = {};
  for (const [key, value] of Object.entries(attributes)) {
    if (value instanceof Date) {
      normalizedAttributes[key] = value.toISOString();
    } else {
      normalizedAttributes[key] = value;
    }
  }
  
  const updateQueue = UpdateQueue.getInstance();
  updateQueue.updateAttributes(normalizedAttributes);
  void updateQueue.processUpdates();
  
  return okVoid();
};
```

#### UpdateQueue 数据合并
```typescript
// packages/js-core/src/lib/user/update-queue.ts:23
public updateUserId(userId: string): void {
  if (!this.updates) {
    this.updates = { userId, attributes: {} };
  } else {
    this.updates = { ...this.updates, userId };
  }
}

public updateAttributes(attributes: TAttributes): void {
  const userId = this.updates?.userId ?? config.get().user.data.userId ?? "";
  
  if (!this.updates) {
    this.updates = { userId, attributes };
  } else {
    this.updates = {
      ...this.updates,
      userId,
      attributes: { ...this.updates.attributes, ...attributes },
    };
  }
}
```

#### processUpdates 处理流程
```typescript
// packages/js-core/src/lib/user/update-queue.ts:91
public async processUpdates(): Promise<void> {
  // 1. 防抖处理（500ms）
  // 2. 获取 userId（从 updates 或 config）
  // 3. 特殊处理：如果只有 language 而没有 userId，本地保存 language
  // 4. 如果有 attributes 但没有 userId，记录错误并清除
  // 5. 调用 sendUpdates 发送到后端
  const result = await sendUpdates({
    updates: {
      userId: effectiveUserId,
      attributes: currentUpdates.attributes ?? {},
    },
  });
  
  // 6. 更新本地 config
  config.update({
    ...config.get(),
    user: { ...userState },
    filteredSurveys,
  });
  
  // 7. 清除队列
  this.clearUpdates();
}
```

#### sendUpdates 发送到后端
```typescript
// packages/js-core/src/lib/user/update.ts:67
export const sendUpdates = async ({ updates }: { updates: TUpdates }) => {
  // 调用 sendUpdatesToBackend
  const updatesResponse = await sendUpdatesToBackend({ appUrl, workspaceId, updates });
  
  // 更新本地状态
  config.update({
    ...config.get(),
    user: { ...userState },
    filteredSurveys,
  });
};

// packages/js-core/src/lib/user/update.ts:9
export const sendUpdatesToBackend = async ({ appUrl, workspaceId, updates }) => {
  // 调用 ApiClient.createOrUpdateUser
  const api = new ApiClient({ appUrl, workspaceId, isDebug });
  const response = await api.createOrUpdateUser({
    userId: updates.userId,
    attributes: updates.attributes,
  });
  return response;
};
```

#### ApiClient.createOrUpdateUser
```typescript
// packages/js-core/src/lib/common/api.ts:70
async createOrUpdateUser(userUpdateInput: {
  userId: string;
  attributes?: Record<string, string | number>;
}): Promise<Result<CreateOrUpdateUserResponse, ApiErrorResponse>> {
  return makeRequest(
    this.appUrl,
    `/api/v2/client/${this.workspaceId}/user`,  // ⚠️ 调用 v2 接口
    "POST",
    { userId: userUpdateInput.userId, attributes: userUpdateInput.attributes },
    this.isDebug
  );
}
```

#### CommandQueue 与 UpdateQueue 的协作
```typescript
// packages/js-core/src/lib/common/command-queue.ts:90
if (currentItem.type === CommandType.GeneralAction) {
  // 执行 GeneralAction 前，先等待 UpdateQueue 完成
  const updateQueue = UpdateQueue.getInstance();
  if (!updateQueue.isEmpty()) {
    console.log("🧱 Formbricks - Waiting for pending updates to complete before executing command");
    await updateQueue.processUpdates();
  }
}
```

#### 完整触发链路图

```
setUserId(userId)
  ↓
UpdateQueue.updateUserId(userId)
  ↓
UpdateQueue.processUpdates()
  ↓
防抖 500ms
  ↓
sendUpdates({ updates: { userId, attributes } })
  ↓
sendUpdatesToBackend({ appUrl, workspaceId, updates })
  ↓
ApiClient.createOrUpdateUser({ userId, attributes })
  ↓
POST /api/v2/client/[workspaceId]/user
  ↓
转发到 POST /api/v1/client/[workspaceId]/user
  ↓
updateUser(workspaceId, userId, device, attributes)
  ↓
getContactWithFullData() → findOrCreateContact()
  ↓
updateAttributes(contactId, userId, workspaceId, attributes)
  ↓
两个独立事务：
  1. prisma.$transaction([upsert existing attributes])
  2. prisma.$transaction([create new attributeKeys + attributes])
  ↓
buildUserStateFromContact() → segments, displays, responses
  ↓
返回 { state, messages, errors }
```

#### 失败时的回滚边界

| 失败阶段 | 回滚边界 | 数据状态 |
|---------|---------|---------|
| UpdateQueue 防抖阶段 | 本地内存，无数据库操作 | updates 保留在内存中，下次调用时重试 |
| sendUpdates 网络错误 | 未发送到后端 | updates 被清除，需要用户重新调用 |
| /user 接口验证失败 | 数据库未修改 | 返回错误消息，本地状态不更新 |
| getContactWithFullData 失败 | 数据库未修改 | 返回错误 |
| createContact 失败 | 事务回滚，联系人未创建 | 返回错误 |
| updateAttributes 事务1失败 | 现有属性未更新 | 事务回滚，新属性可能已创建 |
| updateAttributes 事务2失败 | 新属性未创建 | 现有属性可能已更新 |
| buildUserStateFromContact 失败 | 属性已更新，但用户状态未返回 | 返回错误，但数据库已更新 |

---

## 二、字段映射（Field Mapping）

### 1. ContactInfo 问题数据格式

#### 前端数据结构（contact-info-element.tsx）
ContactInfo 问题的答案在前端以**数组**格式存储：
```typescript
// packages/surveys/src/components/elements/contact-info-element.tsx
const fieldIds = ["firstName", "lastName", "email", "phone", "company"];

// 示例值
["John", "Doe", "john@example.com", "13800138000", "Acme Corp"]
```

#### 后端存储格式
在 `response.data` 中保持原始数组格式：
```json
{
  "responseId": "clx...",
  "data": {
    "question-1": ["John", "Doe", "john@example.com", "13800138000", "Acme Corp"]
  },
  "contactAttributes": {
    "firstName": "旧值",
    "lastName": "旧值",
    "email": "old@example.com"
  }
}
```

### 2. 联系人属性格式

#### 键值对格式
```typescript
// TContactAttributesInput
{
  firstName: "John",
  lastName: "Doe",
  email: "john@example.com",
  phone: "13800138000",
  company: "Acme Corp",
  custom_attribute: "value"
}
```

#### 数据库存储
- `ContactAttributeKey` 表：存储属性定义（key, name, type, dataType）
- `ContactAttribute` 表：存储属性值（value, valueNumber, valueDate）
- 通过 `attributeKeyId` 关联

### 3. 字段映射关系（需手动转换）

| ContactInfo 数组索引 | 字段含义 | 联系人属性键 | 数据类型 |
|---------------------|---------|-------------|---------|
| 0 | 名字 | firstName | string |
| 1 | 姓氏 | lastName | string |
| 2 | 邮箱 | email | string |
| 3 | 电话 | phone | string |
| 4 | 公司 | company | string |

### 4. 转换逻辑示例
```typescript
// 从 Response data 中提取 ContactInfo 并转换为联系人属性格式
const convertContactInfoToAttributes = (contactInfoArray: string[]): TContactAttributesInput => {
  const [firstName, lastName, email, phone, company] = contactInfoArray;
  return {
    firstName: firstName || "",
    lastName: lastName || "",
    email: email || "",
    phone: phone || "",
    company: company || "",
  };
};
```

### 5. 特殊字段处理

#### email 和 userId 的唯一性约束
```typescript
// apps/web/modules/ee/contacts/lib/attributes.ts:153-155
const [existingEmailAttribute, existingUserIdAttribute] = await Promise.all([
  emailValue ? hasEmailAttribute(emailValue, workspaceId, contactId) : Promise.resolve(null),
  userIdValue ? hasUserIdAttribute(userIdValue, workspaceId, contactId) : Promise.resolve(null),
]);

// 如果已存在，则跳过更新并记录警告
if (emailExists) {
  const { email: _email, ...rest } = contactAttributes;
  contactAttributes = rest;
  ignoreEmailAttribute = true;
}
```

---

## 三、失败回滚与事务处理（Failure Rollback & Transactions）

### 1. Response 创建事务

#### 事务范围
```typescript
// apps/web/app/api/v2/client/[workspaceId]/responses/lib/response.ts:32-49
const txResponse = await prisma.$transaction(async (tx) => {
  const response = await createResponse(responseInput, tx);
  const quotaResult = await evaluateResponseQuotas({...});
  return { ...response, ...(quotaResult.quotaFull && { quotaFull: quotaResult.quotaFull }) };
});
```

#### 事务边界
- **包含**：Response 创建、配额评估
- **不包含**：联系人属性更新、异步管道

#### 回滚机制
- 任何一步抛出异常，Prisma 自动回滚整个事务
- Response 不会写入数据库，配额也不会更新

### 2. 异步管道处理

#### 事务特性
- **无全局事务**：各处理步骤独立
- **独立错误处理**：每个副作用都有自己的 try-catch

#### 错误处理示例
```typescript
// apps/web/modules/response-pipeline/lib/process-response-pipeline-job.ts:608-619
if (integrations.length > 0) {
  try {
    await handleIntegrations(integrations, data, survey);
  } catch (error) {
    logger.error({...}, "Response pipeline integration handling failed");
    // 只记录日志，不影响其他步骤，不重试
  }
}
```

#### 重试机制
- BullMQ 队列自带重试功能
- 仅对特定错误重试（如数据库连接池耗尽）
- 其他错误直接标记为失败

### 3. 联系人属性更新事务

#### 多阶段事务设计
`updateAttributes` 函数内部有**两个独立的 Prisma 事务**：

##### 事务1：更新现有属性
```typescript
// apps/web/modules/ee/contacts/lib/attributes.ts:276-299
if (existingAttributes.length > 0) {
  await prisma.$transaction(
    existingAttributes.map(({ attributeKeyId, columns }) =>
      prisma.contactAttribute.upsert({
        where: {
          contactId_attributeKeyId: { contactId, attributeKeyId },
        },
        update: {
          value: columns.value,
          valueNumber: columns.valueNumber,
          valueDate: columns.valueDate,
        },
        create: {
          contactId,
          attributeKeyId,
          value: columns.value,
          valueNumber: columns.valueNumber,
          valueDate: columns.valueDate,
        },
      })
    )
  );
}
```

##### 事务2：创建新属性
```typescript
// apps/web/modules/ee/contacts/lib/attributes.ts:354-374
await prisma.$transaction(
  preparedNewAttributes.map(({ key, dataType, columns }) =>
    prisma.contactAttributeKey.create({
      data: {
        key,
        name: formatSnakeCaseToTitleCase(key),
        type: "custom",
        dataType,
        workspaceId,
        attributes: {
          create: {
            contactId,
            value: columns.value,
            valueNumber: columns.valueNumber,
            valueDate: columns.valueDate,
          },
        },
      },
    })
  )
);
```

#### 部分失败处理策略

| 失败场景 | 处理方式 | 影响范围 |
|---------|---------|---------|
| 类型验证失败 | 跳过该属性，记录错误消息 | 单个属性 |
| email 已存在 | 跳过 email 更新，记录警告 | 单个字段 |
| userId 已存在 | 跳过 userId 更新，记录警告 | 单个字段 |
| 属性键无效 | 跳过该属性，记录错误 | 单个属性 |
| 超过属性数量限制 | 跳过所有新属性，记录警告 | 所有新属性 |
| 现有属性更新事务失败 | 抛出异常，回滚该事务 | 所有现有属性更新 |
| 新属性创建事务失败 | 抛出异常，回滚该事务 | 所有新属性创建 |

#### 事务边界问题
⚠️ **现有属性更新**和**新属性创建**是两个独立事务，可能出现：
1. 现有属性更新成功
2. 新属性创建失败
3. 结果：部分更新，部分回滚

**解决方案**：业务层需要处理这种部分成功的情况，通过 `messages` 和 `errors` 返回详细信息。

### 4. 跨系统一致性问题

#### 典型场景
1. Response 创建成功（事务1提交）
2. 异步管道任务入队成功
3. 联系人属性更新失败（独立事务）

#### 数据不一致表现
- `response.contactAttributes` 保存的是更新前的旧值
- 联系人画像中的属性没有更新

#### 解决方案
| 方案 | 实现方式 | 适用场景 |
|-----|---------|---------|
| 同步调用 | Response 创建后立即调用 `updateAttributes`，用同一个外层事务包裹 | 需要强一致性 |
| 队列重试 | 联系人属性更新失败时，重新入队重试 | 可容忍短暂不一致 |
| Webhook 补偿 | 通过 Webhook 通知外部系统，由外部系统保证最终一致性 | 微服务架构 |
| 读取时合并 | 读取 Response 时，动态获取最新的联系人属性覆盖快照 | 读多写少场景 |

---

## 四、关键代码位置索引

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| Response 创建（v2） | `apps/web/app/api/v2/client/[workspaceId]/responses/lib/response.ts` | 22, 92 |
| Response 创建路由（v2） | `apps/web/app/api/v2/client/[workspaceId]/responses/route.ts` | 188 |
| Response 更新（v1） | `apps/web/app/api/v1/client/[workspaceId]/responses/[responseId]/lib/response.ts` | 7 |
| Response 更新路由（v1） | `apps/web/app/api/v1/client/[workspaceId]/responses/[responseId]/lib/put-response-handler.ts` | 207 |
| Response 更新服务 | `apps/web/lib/response/service.ts` | 507 |
| 事务数据构建 | `apps/web/app/api/v1/lib/utils.ts` | 5 |
| 管道入队 | `apps/web/app/lib/pipelines.ts` | 5 |
| 管道处理 | `apps/web/modules/response-pipeline/lib/process-response-pipeline-job.ts` | 582, 720 |
| 联系人属性更新 | `apps/web/modules/ee/contacts/lib/attributes.ts` | 108 |
| 联系人属性包装 | `apps/web/modules/ee/contacts/lib/update-contact-attributes.ts` | 19 |
| /user 接口（v1） | `apps/web/modules/ee/contacts/api/v1/client/[workspaceId]/user/route.ts` | 40 |
| /user 接口（v2） | `apps/web/app/api/v2/client/[workspaceId]/user/route.ts` | 1 |
| updateUser 服务 | `apps/web/modules/ee/contacts/api/v1/client/[workspaceId]/user/lib/update-user.ts` | 132 |
| ContactInfo 前端 | `packages/surveys/src/components/elements/contact-info-element.tsx` | - |
| JS SDK UpdateQueue | `packages/js-core/src/lib/user/update-queue.ts` | 7, 91 |
| JS SDK setUserId | `packages/js-core/src/lib/user/user.ts` | 8 |
| JS SDK setAttributes | `packages/js-core/src/lib/user/attribute.ts` | 18 |
| JS SDK sendUpdates | `packages/js-core/src/lib/user/update.ts` | 9, 67 |
| JS SDK ApiClient | `packages/js-core/src/lib/common/api.ts` | 51, 70 |
| JS SDK CommandQueue | `packages/js-core/src/lib/common/command-queue.ts` | 25, 90 |

---

## 五、架构建议

### 1. 实现自动回写
如需实现问卷答案自动回写联系人画像，建议在 `processResponsePipelineJob` 中添加处理逻辑：

```typescript
// 在 runResponseFinishedSideEffects 中添加
await handleContactAttributeSync(data.response, survey, workspaceId);
```

### 2. 事务增强
如需强一致性，可将联系人属性更新纳入 Response 创建事务：

```typescript
const txResponse = await prisma.$transaction(async (tx) => {
  const response = await createResponse(responseInput, tx);
  
  // 提取 ContactInfo 并更新联系人属性
  const contactAttributes = extractContactInfo(response.data, survey);
  if (Object.keys(contactAttributes).length > 0 && responseInput.contactId) {
    await updateAttributesInTx(responseInput.contactId, "", responseInput.workspaceId, contactAttributes, false, tx);
  }
  
  const quotaResult = await evaluateResponseQuotas({...});
  return { ...response, ...(quotaResult.quotaFull && { quotaFull: quotaResult.quotaFull }) };
});
```

### 3. 幂等性设计
添加幂等键（Idempotency Key）防止重复更新：
- 使用 `responseId + questionId` 作为幂等键
- 记录已处理的更新，避免重复执行

---

## 总结

### 核心发现

1. **写入顺序**：
   - Response 创建：同步落库 → 异步管道处理 → （可选）联系人属性更新
   - Response 更新：PUT 更新 → 触发 responseUpdated 事件 → 如果 finished=true 额外触发 responseFinished
   - JS SDK 属性更新：setUserId/setAttributes → UpdateQueue 防抖 → sendUpdates → POST /api/v2/client/[workspaceId]/user → updateUser → updateAttributes

2. **字段映射**：
   - ContactInfo 数组格式需手动转换为联系人属性键值对格式
   - 映射关系：[0]→firstName, [1]→lastName, [2]→email, [3]→phone, [4]→company
   - JS SDK 发送的属性格式：`Record<string, string | number>`

3. **事务边界**：
   - Response 创建：有事务，包含 Response 创建 + 配额评估
   - Response 更新：有事务，包含 Response 更新 + 配额评估
   - 联系人属性更新：两个独立事务（更新现有属性 / 创建新属性），可能部分成功
   - Response 事务与联系人属性事务不共享

4. **失败回滚**：
   - 各层独立处理，无全局回滚机制
   - Response 事务失败：自动回滚，Response 不写入
   - 联系人属性事务失败：部分回滚，可能出现部分属性已更新
   - JS SDK 网络错误：updates 被清除，需重新调用
   - 需业务层补偿（重试、幂等性、补偿机制）

5. **v1 vs v2 /user 接口**：
   - v2 只是转发到 v1 的实现，两者使用相同的后端逻辑
   - JS SDK 的 ApiClient.createOrUpdateUser 调用的是 v2 接口

6. **JS SDK UpdateQueue 机制**：
   - 单例设计，500ms 防抖
   - 支持合并多次调用（userId + attributes）
   - CommandQueue 执行 GeneralAction 前会等待 UpdateQueue 完成
   - 5秒超时保护

7. **⚠️ 核心发现**：当前代码没有自动把 ContactInfo 答案写回联系人画像的逻辑，需额外实现
