# 问卷文件上传与清理机制分析

## 一、文件上传通道

### 1.1 上传 API 端点

Formbricks 提供两套文件上传 API，分别用于公共文件和私有文件（问卷答案附件）。

#### 公共文件上传
- **路由**: `POST /api/v1/management/storage` [apps/web/app/api/v1/management/storage/route.ts:17-89](file:///d:/fz/0508-3/solo-dogfeeding/code/64-formbricks/apps/web/app/api/v1/management/storage/route.ts#L17-L89)
- **用途**: 上传公共资源（如品牌背景图、Logo 等）
- **访问类型**: `public`
- **权限**: 需要认证
- **最大文件大小**: 5MB

#### 私有文件上传（问卷答案附件）
- **路由**: `POST /api/v1/client/[workspaceId]/storage` [apps/web/app/api/v1/client/[workspaceId]/storage/route.ts:32-158](file:///d:/fz/0508-3/solo-dogfeeding/code/64-formbricks/apps/web/app/api/v1/client/[workspaceId]/storage/route.ts#L32-L158)
- **用途**: 上传问卷答案中的文件附件
- **访问类型**: `private`
- **权限**: 无需认证（公开问卷场景）
- **最大文件大小**: 标准版 10MB，企业版可升级

### 1.2 上传流程

```
客户端                      服务端                         S3
   |                            |                            |
   | 1. 请求预签名 URL         |                            |
   |--------------------------->|                            |
   |                            | 2. 生成预签名 URL        |
   |                            |--------------------------->|
   |                            | 3. 返回 signedUrl + fileUrl  |
   |                            |<---------------------------|
   | 4. 返回 signedUrl + fileUrl|                            |
   |<---------------------------|                            |
   |                            |                            |
   | 5. 直传文件到 S3          |                            |
   |------------------------------------------------------>|
   |                            |                            |
   | 6. 将 fileUrl 存入答案   |                            |
   |    存入 Response.data      |                            |
   |--------------------------->|                            |
```

### 1.3 核心服务函数

**获取预签名 URL: `getSignedUrlForUpload` [apps/web/modules/storage/service.ts:16-66](file:///d:/fz/0508-3/solo-dogfeeding/code/64-formbricks/apps/web/modules/storage/service.ts#L16-L66)

```typescript
// 文件命名规则: {originalName}--fid--{UUID}.{extension}
const updatedFileName = `${fileNameWithoutExtension}--fid--${randomUUID()}.${fileExtension}`;

// 返回的 fileUrl 格式:
/storage/{workspaceId}/{accessType}/{encodedFileName}
```

**S3 预签名 URL 生成**: `getSignedUploadUrl` [packages/storage/src/service.ts:28-85](file:///d:/fz/0508-3/solo-dogfeeding/code/64-formbricks/packages/storage/src/service.ts#L28-L85)

- 预签名 URL 有效期：2分钟
- 支持文件大小限制通过 `content-length-range` 条件

---

## 二、文件与答案的绑定关系

### 2.1 数据库模型

**Response 模型 [packages/database/schema.prisma:158-188](file:///d:/fz/0508-3/solo-dogfeeding/code/64-formbricks/packages/database/schema.prisma#L158-L188)

```prisma
model Response {
  id        String @id @default(cuid())
  surveyId  String
  data      Json   @default("{}")  // 答案数据，包含文件 URL
  // ... 其他字段
}
```

**关键设计特点**:

1. **无独立 File/Attachment 表**: 文件与答案的关联是**隐式**的，没有独立的数据库表来跟踪文件引用。

2. **文件 URL 存储格式**:
   ```json
   {
     "question-id-123": [
       "/storage/workspace-abc/private/report--fid--uuid-123.pdf",
       "/storage/workspace-abc/private/image--fid--uuid-456.jpg"
     ],
     "other-question": "text answer"
   }
   ```

3. **文件命名约定**: `{originalName}--fid--{UUID}.{extension}`
   - `--fid--` 是分隔符，后跟 UUID 确保唯一性
   - 存储时对文件名进行安全过滤 [apps/web/modules/storage/utils.ts:24-60](file:///d:/fz/0508-3/solo-dogfeeding/code/64-formbricks/apps/web/modules/storage/utils.ts#L24-L60)

### 2.2 绑定关系的建立

1. **客户端上传文件，获得 `fileUrl`
2. **用户提交答案时，将 `fileUrl` 存入 `Response.data` 中对应问题 ID 的数组里
3. **绑定完成**：通过问题类型（`FileUpload`）和问题 ID 来识别文件归属

### 2.3 文件 URL 解析

`getOriginalFileNameFromUrl` [apps/web/modules/storage/url-helpers.ts:12-29](file:///d:/fz/0508-3/solo-dogfeeding/code/64-formbricks/apps/web/modules/storage/url-helpers.ts#L12-L29)

```typescript
// 从 URL 中提取原始文件名
// 例如: "/storage/ws-abc/private/report--fid--uuid-123.pdf
// 解析后: "report.pdf"
```

---

## 三、答案删除时的附件清理机制

### 3.1 单个答案删除流程

`deleteResponse` 函数有两套实现：

#### 旧版实现 [apps/web/lib/response/service.ts:608-670](file:///d:/fz/0508-3/solo-dogfeeding/code/64-formbricks/apps/web/lib/response/service.ts#L608-L670)

```typescript
export const deleteResponse = async (responseId: string, decrementQuotas: boolean = false) => {
  // 1. 事务删除 Response 记录
  const txResponse = await prisma.$transaction(async (tx) => {
    const responsePrisma = await tx.response.delete({...});
    // 删除关联的 display、配额等
    return response;
  });

  // 2. 事务提交后，清理上传的文件
  const survey = await getSurvey(txResponse.surveyId);
  if (survey) {
    await findAndDeleteUploadedFilesInResponse(txResponse, survey);
  }
};
```

#### API v2 实现 [apps/web/modules/api/v2/management/responses/[responseId]/lib/response.ts:92-139](file:///d:/fz/0508-3/solo-dogfeeding/code/64-formbricks/apps/web/modules/api/v2/management/responses/[responseId]/lib/response.ts#L92-L139)

```typescript
export const deleteResponse = async (responseId: string) => {
  // 1. 删除 Response 记录
  const deletedResponse = await prisma.response.delete({...});

  // 2. 删除 display
  if (deletedResponse.displayId) {
    await deleteDisplay(deletedResponse.displayId);
  }

  // 3. 获取问卷问题
  const surveyQuestionsResult = await getSurveyQuestions(deletedResponse.surveyId);

  // 4. 清理上传的文件
  await findAndDeleteUploadedFilesInResponse(
    deletedResponse.data,
    surveyQuestionsResult.data.questions,
    surveyQuestionsResult.data.workspaceId
  );
};
```

### 3.2 文件清理核心逻辑

`findAndDeleteUploadedFilesInResponse` 同样有两套实现：

#### 旧版 [apps/web/lib/response/service.ts:579-606](file:///d:/fz/0508-3/solo-dogfeeding/code/64-formbricks/apps/web/lib/response/service.ts#L579-L606)

```typescript
const findAndDeleteUploadedFilesInResponse = async (response: TResponse, survey: TSurvey) => {
  // 1. 找出所有 FileUpload 类型的问题 ID
  const fileUploadElements = new Set(
    elements.filter(e => e.type === TSurveyElementTypeEnum.FileUpload).map(q => q.id)
  );

  // 2. 从 response.data 中提取这些问题的文件 URL
  const fileUrls = Object.entries(response.data)
    .filter(([elementId]) => fileUploadElements.has(elementId))
    .flatMap(([, elementResponse]) => elementResponse as string[]);

  // 3. 解析 URL 并删除文件
  const deletionPromises = fileUrls.map(async (fileUrl) => {
    const { pathname } = new URL(fileUrl);
    const [, storageId, accessType, fileName] = pathname.split("/").filter(Boolean);
    return deleteFile(storageId, accessType, fileName, survey.workspaceId);
  });

  await Promise.all(deletionPromises);
};
```

#### API v2 版 [apps/web/modules/api/v2/management/responses/[responseId]/lib/utils.ts:8-42](file:///d:/fz/0508-3/solo-dogfeeding/code/64-formbricks/apps/web/modules/api/v2/management/responses/[responseId]/lib/utils.ts#L8-L42)

逻辑与旧版类似，增加了错误处理和 Result 返回类型。

### 3.3 S3 文件删除

`deleteFile` [packages/storage/src/service.ts:211-242](file:///d:/fz/0508-3/solo-dogfeeding/code/64-formbricks/packages/storage/src/service.ts#L211-L242)

```typescript
export const deleteFile = async (fileKey: string) => {
  const deleteObjectCommand = new DeleteObjectCommand({
    Bucket: S3_BUCKET_NAME,
    Key: fileKey,
  });
  await s3Client.send(deleteObjectCommand);
};
```

---

## 四、设计缺陷与风险

### 4.1 无定时清理孤儿文件的 CRON 任务

**当前背景任务列表** [packages/jobs/src/definitions.ts:8-24](file:///d:/fz/0508-3/solo-dogfeeding/code/64-formbricks/packages/jobs/src/definitions.ts#L8-L24)

```typescript
export const backgroundJobDefinitions = {
  [JOB_NAMES.responsePipeline]: {...},
  [JOB_NAMES.surveyScheduling]: {...},
  [JOB_NAMES.testLog]: {...},
} as const;
```

**问题**: 没有 `storage-cleanup` 或 `orphan-file-cleanup` 任务。

### 4.2 孤儿文件产生场景

1. **用户上传文件但未提交答案**：
   - 文件已上传到 S3，但 `fileUrl` 未存入任何 Response
   - 这些文件永远不会被清理

2. **批量删除答案**：
   - 未找到批量删除答案时调用 `findAndDeleteUploadedFilesInResponse` 的代码
   - 可能存在文件残留风险

3. **删除问卷/工作区时**：
   - `deleteFilesByWorkspaceId` [apps/web/modules/storage/service.ts:122-136](file:///d:/fz/0508-3/solo-dogfeeding/code/64-formbricks/apps/web/modules/storage/service.ts#L122-L136) 可按前缀批量删除
   - 但需要确认删除工作区时显式调用

4. **更新答案时替换文件**：
   - 用户重新上传文件覆盖旧答案，旧文件成为孤儿

### 4.3 事务一致性问题

**删除流程：
```
事务开始
  ├─ 删除 Response 记录
  └─ 删除关联数据（display、配额等）
事务提交
  └─ 异步删除 S3 文件 ← 可能失败
```

**风险**: 如果 S3 删除失败，文件成为孤儿，且没有重试机制。

### 4.4 两套实现不一致

- `apps/web/lib/response/service.ts`（旧版）
- `apps/web/modules/api/v2/management/responses/[responseId]/lib/`（API v2 版）

两套实现逻辑相似但独立维护，存在行为可能不一致。

---

## 五、代码位置索引

| 功能 | 文件路径 |
|------|---------|
| 公共文件上传 API | `apps/web/app/api/v1/management/storage/route.ts` |
| 私有文件上传 API | `apps/web/app/api/v1/client/[workspaceId]/storage/route.ts` |
| 存储服务层 | `apps/web/modules/storage/service.ts` |
| S3 存储核心 | `packages/storage/src/service.ts` |
| 旧版答案删除 | `apps/web/lib/response/service.ts` |
| API v2 答案删除 | `apps/web/modules/api/v2/management/responses/[responseId]/lib/response.ts` |
| 文件清理工具（旧版） | `apps/web/lib/response/service.ts:579-606` |
| 文件清理工具（v2） | `apps/web/modules/api/v2/management/responses/[responseId]/lib/utils.ts` |
| 背景任务定义 | `packages/jobs/src/definitions.ts` |
| 数据库模型 | `packages/database/schema.prisma` |
| 存储类型定义 | `packages/types/storage.ts` |

---

## 六、改进建议

### 6.1 增加孤儿文件清理 CRON 任务

1. 新增 `storage-cleanup` 定时任务，定期扫描 S3，对比数据库中的文件引用，删除孤儿文件。

### 6.2 引入 File 表显式追踪

新增 `ResponseFile` 表，显式追踪文件与答案的关联：

```prisma
model ResponseFile {
  id          String   @id @default(cuid())
  responseId  String
  response    Response @relation(fields: [responseId], references: [id], onDelete: Cascade)
  fileUrl     String
  fileName    String
  createdAt   DateTime @default(now())
  @@index([responseId])
}
```

### 6.3 确保批量删除时清理文件

在批量删除答案的 API 中，遍历所有被删除的答案，调用 `findAndDeleteUploadedFilesInResponse`。

### 6.4 统一清理逻辑

合并两套 `findAndDeleteUploadedFilesInResponse` 实现，避免维护成本。

### 6.5 增加删除重试机制

S3 文件删除失败时，将失败记录，定时重试。
