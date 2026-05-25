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
| JS SDK setAttributes | POST /api/v2/client/[workspaceId]/user | 同上 | `updateAttributes` - 三个独立写操作（操作0无事务 / 操作1有事务 / 操作2有事务） |
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
三个独立数据库写操作（按顺序串行）：
  0. deleteAttributes（可选，无事务）
  1. prisma.$transaction([upsert existing attributes])
  2. prisma.$transaction([create new attributeKeys + attributes])
  ↓
buildUserStateFromContact() → segments, displays, responses
  ↓
返回 { state, messages, errors }
```

#### 失败时的回滚边界（摘要）

| 失败阶段 | 回滚边界 | 数据状态 |
|---------|---------|---------|
| UpdateQueue 防抖阶段 | 本地内存，无数据库操作 | updates 在 `processUpdates` 末尾被 `clearUpdates()` 清除，**无重试机制** |
| sendUpdates 网络错误 | 请求未到达或未成功处理后端 | updates 被清除，需重新调用 `setAttributes` / `setUserId` |
| /user 接口验证失败 | 数据库未修改 | 返回错误消息，本地 config 不更新 |
| getContactWithFullData 失败 | 数据库未修改 | 返回错误 |
| createContact 失败 | 事务回滚，联系人未创建 | 返回错误 |
| updateAttributes 操作0（deleteAttributes）失败 | 两种结果：① 无需删除（`keysToDelete` 为空）→ 正常返回，后续操作继续；② `deleteMany` 抛出异常 → 异常传播，操作1和操作2**不执行** | 需区分场景：① 无影响；② 后续操作被跳过 |
| updateAttributes 操作1（已有属性更新）失败 | 事务回滚，已有属性未更新 | 操作2（新属性创建）**不会执行** |
| updateAttributes 操作2（新属性创建）失败 | 事务回滚，新属性未创建 | 操作1已提交，已有属性已更新 |
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

#### 多阶段数据库写操作

`updateAttributes` 函数内部包含**三个独立的数据库写操作**，按严格顺序串行执行，彼此不共享事务：

| 序号 | 操作 | 事务保护 | 执行条件 |
|-----|------|---------|---------|
| 0 | `deleteAttributes`（删除被移除的属性值） | ❌ 无 | `deleteRemovedAttributes=true` 且存在需要删除的属性 |
| 1 | 更新已有属性（`contactAttribute.upsert` 批量） | ✅ `$transaction` | `existingAttributes.length > 0` |
| 2 | 创建新属性（`contactAttributeKey.create` + 嵌套 `contactAttribute.create`） | ✅ `$transaction` | `validNewAttributes.length > 0` 且未超过数量限制 |

> 各操作的事务保护、执行顺序与失败级联影响详见**第 5 节**。

#### 部分失败处理策略

| 失败场景 | 处理方式 | 影响范围 |
|---------|---------|---------|
| 类型验证失败 | 跳过该属性，记录错误消息到 `messages` | 单个属性 |
| email 已存在于其他联系人 | 从 payload 中移除 email，记录警告 | 单个字段 |
| userId 已存在于其他联系人 | 从 payload 中移除 userId，记录警告 | 单个字段 |
| 属性键无效（`isSafeIdentifier`） | 跳过该新属性，记录错误到 `errors` | 单个新属性 |
| 超过属性数量限制（`MAX_ATTRIBUTE_CLASSES_PER_ENVIRONMENT`） | 跳过所有新属性，记录警告 | 所有新属性 |
| 操作0（deleteAttributes）失败 | 两种路径：① 无需删除（`keysToDelete` 为空）→ 返回 `{success: true}`，操作1和2正常执行；② `deleteMany` 抛出异常 → 异常**直接传播**，操作1和2不执行 | 场景①：无影响；场景②：后续操作被跳过，函数抛出异常 |
| 操作1（已有属性更新事务）失败 | 事务回滚，**抛出异常**，操作2不执行 | 所有已有属性更新被撤销 |
| 操作2（新属性创建事务）失败 | 事务回滚，操作1已提交 | 所有新属性被撤销，已有属性已更新 |

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

### 5. updateAttributes 双事务执行顺序与失败影响详细分析

`updateAttributes` 函数内部存在**三个独立的数据库写操作**，按严格顺序执行，彼此不共享事务。

#### 操作 0：删除被移除的属性（可选，仅 `deleteRemovedAttributes=true` 时执行）

```typescript
// apps/web/modules/ee/contacts/lib/attributes.ts:221-223
if (deleteRemovedAttributes) {
  await deleteAttributes(contactId, currentAttributes, contactAttributesParam, contactAttributeKeys);
}
```

```typescript
// apps/web/modules/ee/contacts/lib/attributes.ts:82-91
if (attributeKeyIdsToDelete.length > 0) {
  await prisma.contactAttribute.deleteMany({
    where: {
      contactId,
      attributeKeyId: { in: attributeKeyIdsToDelete },
    },
  });
}
```

| 属性 | 说明 |
|-----|------|
| **执行条件** | `deleteRemovedAttributes=true` 且存在需要删除的属性 |
| **事务保护** | ❌ 无，独立的 `prisma.contactAttribute.deleteMany` 调用 |
| **调用方** | `updateContactAttributes`（UI 层，传入 `true`）；`updateUser`（API 层，默认 `false`） |
| **保护机制** | `DEFAULT_ATTRIBUTES`（email, userId, firstName, lastName）永远不会被删除 |

**失败影响**：`deleteAttributes` 内部的 `prisma.contactAttribute.deleteMany(...)` 没有 try-catch 包裹。若该调用抛出异常（如数据库连接失败、约束冲突），异常会**直接传播到 `updateAttributes` 并终止函数**，后续操作1和操作2**不会执行**。只有在"无需删除（`keysToDelete` 为空）"的情况下，函数才会正常返回并继续执行后续操作。

#### 操作 1：更新已有属性（Transaction 1）

```typescript
// apps/web/modules/ee/contacts/lib/attributes.ts:275-300
if (existingAttributes.length > 0) {
  await prisma.$transaction(
    existingAttributes.map(({ attributeKeyId, columns }) =>
      prisma.contactAttribute.upsert({
        where: {
          contactId_attributeKeyId: { contactId, attributeKeyId },
        },
        update: { value: columns.value, valueNumber: columns.valueNumber, valueDate: columns.valueDate },
        create: { contactId, attributeKeyId, value: columns.value, valueNumber: columns.valueNumber, valueDate: columns.valueDate },
      })
    )
  );
}
```

| 属性 | 说明 |
|-----|------|
| **执行条件** | `existingAttributes.length > 0`（payload 中包含已存在的属性键） |
| **事务保护** | ✅ `prisma.$transaction` — 所有 upsert 在同一个事务中 |
| **写入表** | `ContactAttribute`（值表，不涉及键定义） |
| **约束** | `contactId_attributeKeyId` 唯一索引；若不存在则走 create |

**失败影响**：
- 事务内任意一个 upsert 失败 → 整个事务回滚 → 所有已有属性的更新被撤销
- 操作 0（删除）已提交 → 可能已删除某些属性值，但操作 1 回滚了 → 这些属性值丢失
- 操作 2（创建新属性）不会执行 → 新属性不会被创建

#### 操作 2：创建新属性（Transaction 2）

```typescript
// apps/web/modules/ee/contacts/lib/attributes.ts:354-374
await prisma.$transaction(
  preparedNewAttributes.map(({ key, dataType, columns }) =>
    prisma.contactAttributeKey.create({
      data: {
        key, name: formatSnakeCaseToTitleCase(key), type: "custom", dataType, workspaceId,
        attributes: { create: { contactId, value: columns.value, valueNumber: columns.valueNumber, valueDate: columns.valueDate } },
      },
    })
  )
);
```

| 属性 | 说明 |
|-----|------|
| **执行条件** | `validNewAttributes.length > 0` 且未超过 `MAX_ATTRIBUTE_CLASSES_PER_ENVIRONMENT` 限制 |
| **事务保护** | ✅ `prisma.$transaction` — 所有 create 在同一个事务中 |
| **写入表** | `ContactAttributeKey`（键定义表）+ `ContactAttribute`（值表，嵌套创建） |
| **前置校验** | `isSafeIdentifier()` 检查键名合法性，非法键名被跳过 |

**失败影响**：
- 事务内任意一个 create 失败 → 整个事务回滚 → 所有新属性键和值被撤销
- 操作 1（更新已有属性）已提交 → 已有属性的更新已持久化
- **典型不一致场景**：用户同时更新 firstName（已有属性）和 customScore（新属性）→ firstName 更新成功但 customScore 创建失败 → 部分更新

#### 完整执行顺序与失败场景矩阵

| 顺序 | 操作 | 事务 | 失败后是否继续 | 失败影响 |
|-----|------|------|--------------|---------|
| 0 | deleteAttributes（可选） | 无 | ✅ 继续（代码不检查返回值） | 被删除的属性值可能丢失 |
| 1 | 更新已有属性 | `$transaction` | ❌ 不继续（抛出异常） | 已有属性更新回滚，操作 2 不执行 |
| 2 | 创建新属性 | `$transaction` | 无后续 | 新属性创建回滚，操作 1 已提交 |

#### 典型故障场景与排查要点

**场景 1：部分属性更新成功，部分失败**
- **现象**：联系人 `firstName` 已更新，但 `customScore` 新属性未创建
- **根因**：操作 1 成功提交，操作 2 失败回滚
- **排查**：检查日志中是否有 `"Created new contact attribute"` 日志（`attributes.ts:349`），对比是否有 Prisma 事务异常日志。操作2失败时 `updateAttributes` 直接抛出异常，由路由层捕获并返回 500

**场景 2：UI 提交表单后属性值丢失**
- **现象**：通过 UI 删除一个属性值后刷新页面，值仍然存在
- **根因**：操作 0（deleteAttributes）失败，但代码未检查返回值（始终返回 `{success: true}`）
- **排查**：检查 `ContactAttribute` 表中 `attributeKeyId` 对应的记录是否仍存在
- **代码行**：`attributes.ts:221-223` — 无 `await` 返回值检查

**场景 3：类型不匹配导致静默跳过**
- **现象**：API 发送 `{ age: "twenty" }` 但联系人属性未更新
- **根因**：`validateAndParseAttributeValue` 类型校验失败，该属性被排除在 `existingAttributes` 之外
- **排查**：检查 `messages` 数组中是否有 `attribute_type_validation_error` 消息
- **代码行**：`attributes.ts:241-259`

### 6. sendUpdates / UpdateQueue 网络异常清理与重试边界

#### UpdateQueue 处理流程与清理时机

```typescript
// packages/js-core/src/lib/user/update-queue.ts:91-196（核心逻辑）
public async processUpdates(): Promise<void> {
  // ── 准备阶段（无 I/O）──
  // 防抖 500ms（line 191）
  // 合并 updates → currentUpdates（line 109）

  // ── 本地 language 处理（不涉及网络）──
  // 无 userId 但有 language → 本地保存 language，从 attributes 中移除（line 117-140）

  // ── 无 userId 有 attributes → 立即清除（line 142-147）──
  if (Object.keys(currentUpdates.attributes ?? {}).length > 0 && !effectiveUserId) {
    logger.error("Formbricks can't set attributes without a userId! ...");
    this.clearUpdates();  // ⚠️ 清除，不重试
  }

  // ── 网络请求（line 150-176）──
  if (effectiveUserId) {
    const result = await sendUpdates({ updates: {...} });
    // result.ok → 正常处理（line 162-170）
    // !result.ok → 仅记录日志，不抛出（line 171-174）
  }

  // ── 清除阶段（line 179-181）──
  this.clearUpdates();        // ⚠️ 无论成功失败，始终清除
  this.pendingFlush = null;
  resolve();
}
```

#### 关键代码断言：sendUpdates 永不抛出

```typescript
// packages/js-core/src/lib/user/update.ts:79-134
export const sendUpdates = async ({...}): Promise<Result<...>> => {
  try {
    const updatesResponse = await sendUpdatesToBackend({...});
    if (!updatesResponse.ok) {
      return err(updatesResponse.error);  // ✅ 返回 err，不抛出
    }
    // ... 成功处理 ...
    return ok({ hasWarnings: ... });
  } catch (e) {
    // ✅ 所有异常被捕获，返回 err()
    return err({ code: "network_error", message: "Error sending updates", ... });
  }
};
```

```typescript
// packages/js-core/src/lib/user/update.ts:29-64
export const sendUpdatesToBackend = async ({...}): Promise<Result<...>> => {
  try {
    const response = await api.createOrUpdateUser({...});
    if (!response.ok) {
      return err({...});  // ✅ HTTP 错误也返回 err，不抛出
    }
    return ok(response.data);
  } catch (e) {
    return err({ code: "network_error", ... });  // ✅ 网络异常也返回 err，不抛出
  }
};
```

**结论**：`processUpdates` 中 `await sendUpdates(...)` 这一行**永远不会 reject**。因此 `catch` 块（line 182-188）仅在 `config.update()` 等本地操作抛出时触发，与网络无关。

#### 网络异常场景分析

| 场景 | 代码路径 | updates 状态 | 重试 | 数据丢失 |
|-----|---------|-------------|------|---------|
| **网络完全中断** | `sendUpdatesToBackend` catch → `err({code:"network_error"})` → `sendUpdates` 返回 err → `processUpdates` line 171 log → line 179 `clearUpdates()` | ❌ 被清除 | ❌ 无重试 | ✅ 属性值丢失 |
| **服务端 5xx 错误** | `api.createOrUpdateUser` 返回 err → `sendUpdatesToBackend` 返回 err → 同上 | ❌ 被清除 | ❌ 无重试 | ✅ 属性值丢失 |
| **服务端 4xx 错误** | `api.createOrUpdateUser` 返回 err → 同上 | ❌ 被清除 | ❌ 无重试 | ✅ 属性值丢失 |
| **请求超时** | `api.createOrUpdateUser` 内部超时 → 抛出 → catch → `err({code:"network_error"})` → 同上 | ❌ 被清除 | ❌ 无重试 | ✅ 属性值丢失 |
| **CORS 阻止** | 浏览器拦截 → fetch 抛出 → catch → `err({code:"network_error"})` → 同上 | ❌ 被清除 | ❌ 无重试 | ✅ 属性值丢失 |
| **防抖窗口内新调用覆盖** | `updateAttributes` 合并到已有 updates → 重新计时 500ms → 最终合并发送 | ✅ 合并 | — | 无丢失（合并后一起发送） |

#### UpdateQueue 中唯一"保留" updates 的场景

```typescript
// packages/js-core/src/lib/user/update-queue.ts:182-188
catch (error: unknown) {
  this.pendingFlush = null;
  // ⚠️ 注意：这里没有调用 this.clearUpdates()
  logger.error(`Failed to process updates: ${...}`);
  reject(error as Error);
}
```

**此 catch 块仅在以下情况触发**（非网络原因）：
- `config.update()` 抛出异常（line 103-129 或 sendUpdates 内部的 config.update）
- `Object.keys(currentUpdates.attributes ?? {})` 抛出（极不可能）
- `currentUpdates.userId` 访问抛出（极不可能）

**即使触发此 catch，updates 也不会被自动重试**——`processUpdates` 返回的 Promise 会 reject，但调用方通常不会 await 这个结果：

```typescript
// packages/js-core/src/lib/user/user.ts:8
export const setUserId = async (userId: string): Promise<Result<void, ApiErrorResponse>> => {
  // ...
  void updateQueue.processUpdates();  // ⚠️ 使用 void，不等待结果
  return okVoid();
};

// packages/js-core/src/lib/user/attribute.ts:18
export const setAttributes = async (...): Promise<Result<void, NetworkError>> => {
  // ...
  void updateQueue.processUpdates();  // ⚠️ 使用 void，不等待结果
  return okVoid();
};
```

#### CommandQueue 与 UpdateQueue 的协调（唯一"等待"机制）

```typescript
// packages/js-core/src/lib/common/command-queue.ts:90-96
if (currentItem.type === CommandType.GeneralAction) {
  const updateQueue = UpdateQueue.getInstance();
  if (!updateQueue.isEmpty()) {
    console.log("🧱 Formbricks - Waiting for pending updates to complete before executing command");
    await updateQueue.processUpdates();  // ✅ 显式 await
  }
}
```

**场景**：用户先调用 `setAttributes({ plan: "pro" })` 然后提交问卷（GeneralAction）
- CommandQueue 检测到 UpdateQueue 非空
- 调用 `await updateQueue.processUpdates()` — 这次会等待结果
- 但如果网络失败，`processUpdates` 仍然清除 updates（line 179），然后 resolve
- CommandQueue 继续执行 GeneralAction，属性更新已丢失

#### waitForPendingWork 的超时保护

```typescript
// packages/js-core/src/lib/user/update-queue.ts:72-89
public async waitForPendingWork(): Promise<boolean> {
  const flush = this.pendingFlush ?? this.processUpdates();
  try {
    const succeeded = await Promise.race([
      flush.then(() => true as const),
      new Promise<false>((resolve) => {
        setTimeout(() => resolve(false), this.PENDING_WORK_TIMEOUT);  // 5秒
      }),
    ]);
    return succeeded;  // true=成功完成, false=超时
  } catch {
    return false;  // 异常也返回 false
  }
}
```

| 情况 | 返回值 | updates 状态 |
|-----|--------|-------------|
| processUpdates 在 5 秒内完成 | `true` | 已被 clearUpdates() 清除 |
| processUpdates 超过 5 秒 | `false` | 仍在处理中（但最终仍会被清除） |
| processUpdates 抛出异常 | `false` | 未被清除（catch 块中未调用 clearUpdates） |

**⚠️ 超时后的行为**：`waitForPendingWork` 返回 `false`，但 `processUpdates` 仍在后台运行。超时不会取消网络请求。请求完成后（无论成功失败），`clearUpdates()` 仍会被调用。

#### 故障排查建议

**现象：用户属性丢失，服务端未收到更新请求**
1. 检查浏览器控制台是否有 `"Failed to send updates:"` 日志（update-queue.ts:172-174）
2. 检查浏览器 Network 面板是否有 `/api/v2/client/[workspaceId]/user` 请求失败
3. 如果是"先设属性后提交问卷"场景，检查是否有 `"Waiting for pending updates to complete"` 日志（command-queue.ts:94）

**现象：用户属性丢失，但服务端收到了请求**
1. 检查服务端 `updateUser` 日志是否有属性匹配（hasChanges 判定为 false）
2. 检查 `updateAttributes` 返回的 `messages` 中是否有 `attribute_type_validation_error`、`email_already_exists`、`userid_already_exists`
3. 检查是否触发了 `attribute_limit_exceeded`（超过 `MAX_ATTRIBUTE_CLASSES_PER_ENVIRONMENT`）

**现象：UI 删除属性后刷新仍然存在**
1. 检查 `updateContactAttributes` 是否传入 `deleteRemovedAttributes: true`
2. 检查 `deleteAttributes` 中 `prisma.contactAttribute.deleteMany` 的返回值（affected count）
3. 确认属性键不在 `DEFAULT_ATTRIBUTES` 集合中（email, userId, firstName, lastName）

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
   - 联系人属性更新：三个独立数据库写操作（操作0 deleteAttributes 无事务 / 操作1 upsert 有事务 / 操作2 create 有事务），操作1失败会阻止操作2执行，操作2失败不影响操作1
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
