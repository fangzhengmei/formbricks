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
| Response 创建 | `apps/web/app/api/v2/client/[workspaceId]/responses/lib/response.ts` | 32, 58 |
| 事务数据构建 | `apps/web/app/api/v1/lib/utils.ts` | 5 |
| 管道入队 | `apps/web/app/lib/pipelines.ts` | 5 |
| 管道处理 | `apps/web/modules/response-pipeline/lib/process-response-pipeline-job.ts` | 582, 720 |
| 联系人属性更新 | `apps/web/modules/ee/contacts/lib/attributes.ts` | 108 |
| 联系人属性包装 | `apps/web/modules/ee/contacts/lib/update-contact-attributes.ts` | 19 |
| ContactInfo 前端 | `packages/surveys/src/components/elements/contact-info-element.tsx` | - |

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

1. **写入顺序**：Response 同步落库 → 异步管道处理 → （可选）联系人属性更新
2. **字段映射**：ContactInfo 数组格式需手动转换为联系人属性键值对格式
3. **事务边界**：Response 创建有事务，联系人属性更新有独立事务，两者不共享
4. **失败回滚**：各层独立处理，无全局回滚机制，需业务层补偿
5. **⚠️ 核心发现**：当前代码没有自动把 ContactInfo 答案写回联系人画像的逻辑，需额外实现
