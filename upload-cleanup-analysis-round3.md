# 问卷附件回收机制深度分析（Round 3）

## 一、问题 1：问卷题目变更后，当前题目集合过滤是否漏删历史答案附件

### 1.1 代码证据

**证据 1.1**: 旧版清理逻辑基于当前 survey.blocks
- 文件: `apps/web/lib/response/service.ts:579-588`
```typescript
const findAndDeleteUploadedFilesInResponse = async (response: TResponse, survey: TSurvey): Promise<void> => {
  const elements = getElementsFromBlocks(survey.blocks);  // ← 当前问卷的 blocks

  const fileUploadElements = new Set(
    elements.filter((element) => element.type === TSurveyElementTypeEnum.FileUpload).map((q) => q.id)
  );

  const fileUrls = Object.entries(response.data)
    .filter(([elementId]) => fileUploadElements.has(elementId))  // ← 只匹配当前存在的题目
    .flatMap(([, elementResponse]) => elementResponse as string[]);
  // ...
};
```

**证据 1.2**: API v2 版清理逻辑基于当前 survey.questions
- 文件: `apps/web/modules/api/v2/management/responses/[responseId]/lib/utils.ts:8-23`
```typescript
export const findAndDeleteUploadedFilesInResponse = async (
  responseData: Response["data"],
  questions: Survey["questions"],  // ← 当前问卷的 questions
  workspaceId?: string
): Promise<Result<void, ApiErrorResponseV2>> => {
  const fileUploadQuestions = new Set(
    questions
      .filter((question: { type: string; id: string }) => question.type === TSurveyQuestionTypeEnum.FileUpload)
      .map((q: { type: string; id: string }) => q.id)
  );

  const fileUrls = Object.entries(responseData)
    .filter(([questionId]) => fileUploadQuestions.has(questionId))  // ← 只匹配当前存在的题目
    .flatMap(([, questionResponse]) => questionResponse as string[]);
  // ...
};
```

**证据 1.3**: 删除答案时传入的是当前 survey 的 questions
- 文件: `apps/web/modules/api/v2/management/responses/[responseId]/lib/response.ts:92-139`
```typescript
export const deleteResponse = async (responseId: string) => {
  const deletedResponse = await prisma.response.delete({...});
  // ...
  const questionsResponse = await getSurveyQuestions(deletedResponse.surveyId);  // ← 当前问卷
  // ...
  await findAndDeleteUploadedFilesInResponse(
    deletedResponse.data,
    questionsResponse.data.questions,  // ← 当前问卷的 questions
    questionsResponse.data.workspaceId
  );
};
```

### 1.2 可复现场景

**场景 A：删除 FileUpload 题目**
1. 问卷包含 FileUpload 题目 `q_file_001`
2. 用户提交答案，上传文件，存入 `response.data["q_file_001"] = ["/storage/.../file.pdf"]`
3. 编辑问卷，删除题目 `q_file_001`
4. 删除该答案
5. **结果**: `getSurveyQuestions()` 返回的 questions 中已无 `q_file_001`，`fileUploadQuestions` 集合不包含该 ID，文件 URL 被过滤掉，**文件永久残留**

**场景 B：修改题目 ID**
1. 问卷包含 FileUpload 题目 `q_file_001`
2. 用户提交答案
3. 编辑问卷，将题目 ID 改为 `q_file_new_id`
4. 删除该答案
5. **结果**: 历史答案的 key 是 `q_file_001`，但当前题目 ID 是 `q_file_new_id`，匹配失败，**文件永久残留**

**场景 C：将 FileUpload 改为其他类型**
1. 问卷包含 FileUpload 题目 `q_file_001`
2. 用户提交答案
3. 编辑问卷，将题目类型改为 `OpenText`
4. 删除该答案
5. **结果**: 题目仍在，但 `type !== FileUpload`，被过滤掉，**文件永久残留**

### 1.3 影响范围

- **影响所有已删除/修改的 FileUpload 题目对应的历史答案**
- 问卷编辑越频繁，残留文件越多
- 无法通过常规删除操作清理，只能通过 S3 手动清理或全表扫描

---

## 二、问题 2：management 端 GET → PUT 路径转换是否混存相对/绝对路径

### 2.1 代码证据

**证据 2.1**: GET 时将相对路径转绝对路径
- 文件: `apps/web/modules/api/v2/management/responses/[responseId]/route.ts:54-57`
```typescript
return responses.successResponse({
  ...response,
  data: { ...response.data, data: resolveStorageUrlsInObject(response.data.data) },  // ← 转绝对路径
});
```

- 文件: `apps/web/app/api/v1/management/responses/[responseId]/route.ts:52-56`
```typescript
return {
  response: responses.successResponse({
    ...result.response,
    data: resolveStorageUrlsInObject(result.response.data),  // ← 转绝对路径
  }),
};
```

**证据 2.2**: `resolveStorageUrlsInObject` 递归转换
- 文件: `apps/web/modules/storage/utils.ts:268-281`
```typescript
export const resolveStorageUrlsInObject = <T>(obj: T): T => {
  if (typeof obj !== "object" || obj === null) return obj;
  if (Array.isArray(obj)) {
    return obj.map((item) => resolveStorageUrlsInObject(item)) as T;
  }
  const result: Record<string, unknown> = {};
  for (const [key, value] of Object.entries(obj)) {
    if (typeof value === "string") {
      result[key] = resolveStorageUrlAuto(value);  // ← 单个字符串转绝对路径
    } else {
      result[key] = resolveStorageUrlsInObject(value);
    }
  }
  return result as T;
};
```

**证据 2.3**: PUT 时直接写入数据库，**无反向转换**
- 文件: `apps/web/modules/api/v2/management/responses/[responseId]/route.ts:218`
```typescript
const response = await updateResponseWithQuotaEvaluation(params.responseId, body);  // ← body.data 是绝对路径
```

- 文件: `apps/web/modules/api/v2/management/responses/[responseId]/lib/response.ts:141-155`
```typescript
export const updateResponse = async (
  responseId: string,
  responseInput: z.infer<typeof ZResponseUpdateSchema>,
  tx?: Prisma.TransactionClient
): Promise<Result<Response, ApiErrorResponseV2>> => {
  const prismaClient = tx ?? prisma;
  const updatedResponse = await prismaClient.response.update({
    where: { id: responseId },
    data: responseInput,  // ← 直接写入，不做路径转换
  });
  return ok(updatedResponse);
};
```

**证据 2.4**: `validateFileUploads` 只验证扩展名，不验证路径格式
- 文件: `apps/web/modules/storage/utils.ts:99-114`
```typescript
export const validateFileUploads = (data?: TResponseData, questions?: TSurveyQuestion[]): boolean => {
  if (!data) return true;
  for (const key of Object.keys(data)) {
    const question = questions?.find((q) => q.id === key);
    if (!question || question.type !== TSurveyQuestionTypeEnum.FileUpload) continue;
    const fileUrls = data[key];
    if (!Array.isArray(fileUrls) || !fileUrls.every((url) => typeof url === "string")) return false;
    for (const fileUrl of fileUrls) {
      if (!validateSingleFile(fileUrl, question.allowedFileExtensions)) return false;
    }
  }
  return true;
};
```

**证据 2.5**: `validateSingleFile` 兼容两种路径
- 文件: `apps/web/modules/storage/utils.ts:88-97`
```typescript
export const validateSingleFile = (fileUrl: string, allowedFileExtensions?: TAllowedFileExtension[]): boolean => {
  const fileName = getOriginalFileNameFromUrl(fileUrl);  // ← 兼容相对/绝对路径
  if (!fileName) return false;
  const extension = extractFileExtension(fileName);
  if (!extension) return false;
  return !allowedFileExtensions || allowedFileExtensions.includes(extension as TAllowedFileExtension);
};
```

**证据 2.6**: `getOriginalFileNameFromUrl` 分支处理
- 文件: `apps/web/modules/storage/url-helpers.ts:12-29`
```typescript
export const getOriginalFileNameFromUrl = (fileURL: string): string => {
  try {
    const lastSegment = fileURL.startsWith("/storage/")
      ? (fileURL.split("/").pop() ?? "")  // ← 相对路径：直接 split
      : (new URL(fileURL).pathname.split("/").pop() ?? "");  // ← 绝对路径：用 URL 解析
    // ...
  } catch {
    return "";
  }
};
```

### 2.2 可复现场景

**场景：Management 端 GET 后 PUT 回写**
1. 数据库中存储：`"/storage/ws-abc/private/file--fid--uuid.pdf"`（相对路径）
2. 调用 GET /api/v2/management/responses/{id}，返回绝对路径：
   ```json
   { "data": { "q_file_001": ["https://app.formbricks.com/storage/ws-abc/private/file--fid--uuid.pdf"] } }
   ```
3. 前端修改其他字段后，调用 PUT 将整个 data 回写
4. 数据库中存储变为：`"https://app.formbricks.com/storage/ws-abc/private/file--fid--uuid.pdf"`（绝对路径）

### 2.3 对删除行为的影响

| 路径类型 | 删除解析行为 | 结果 |
|---------|-------------|------|
| **相对路径**<br>`/storage/...` | `new URL(fileUrl)` 抛 `TypeError: Invalid URL` → catch 吞掉 | ❌ 文件残留 |
| **绝对路径**<br>`https://app.com/storage/...` | `new URL(fileUrl)` 成功解析 → `pathname = /storage/...` | ✅ 正常删除 |

**反直觉结果**: 经过 GET→PUT 转换的绝对路径**反而能被正确删除**，而原始相对路径会失败。这意味着：
- 新创建的答案（相对路径）→ 删除失败
- 经过 Management 端编辑过的答案（绝对路径）→ 删除成功

### 2.4 影响范围

- **Management 端编辑过的答案** 数据中混入绝对路径
- 删除行为不一致，部分答案能删到文件，部分不能
- 数据库中路径格式不统一，增加后续维护难度

---

## 三、问题 3：答案更新与批量删除流程为何可能留下孤儿文件

### 3.1 答案更新（PUT）不清理旧文件

**证据 3.1**: Management 端 PUT 更新答案，无旧文件清理
- 文件: `apps/web/modules/api/v2/management/responses/[responseId]/route.ts:117-252`
```typescript
export const PUT = authenticatedApiClient({
  handler: async ({ parsedInput, auditLog }) => {
    // ... 验证 ...
    const response = await updateResponseWithQuotaEvaluation(params.responseId, body);  // ← 直接更新
    // ...
    return responses.successResponse({
      ...response,
      data: { ...response.data, data: resolveStorageUrlsInObject(response.data.data) },
    });
  },
});
```

**证据 3.2**: 客户端 PUT 更新答案，同样无旧文件清理
- 文件: `apps/web/app/api/v1/client/[workspaceId]/responses/[responseId]/lib/put-response-handler.ts:164-205`
```typescript
const getUpdatedResponse = async (req, responseId, responseUpdateInput) => {
  const updatedResponse = await updateResponseWithQuotaEvaluation(responseId, responseUpdateInput);
  return { updatedResponse };
};
```

**证据 3.3**: `updateResponseWithQuotaEvaluation` 只更新数据库
- 文件: `apps/web/modules/api/v2/management/responses/[responseId]/lib/response.ts:177-213`
```typescript
export const updateResponseWithQuotaEvaluation = async (responseId, responseInput) => {
  const txResponse = await prisma.$transaction(async (tx) => {
    const responseResult = await updateResponse(responseId, responseInput, tx);  // ← 更新 data
    // ... 配额评估 ...
    return responseResult;
  });
  return txResponse;
};
```

### 3.2 可复现场景（答案更新）

**场景：用户更新答案中的文件**
1. 用户提交答案，上传 `file1.pdf`，存入 `response.data["q_file_001"] = ["/storage/.../file1.pdf"]`
2. 用户更新答案，删除 `file1.pdf`，上传 `file2.pdf`
3. PUT 后数据库中变为：`response.data["q_file_001"] = ["/storage/.../file2.pdf"]`
4. **结果**: `file1.pdf` 仍在 S3 中，但已不在答案中引用，**成为孤儿文件**

### 3.3 批量删除的两种路径

#### 路径 A：UI 批量删除（逐行调用单个删除）

**证据 3.4**: SelectedRowSettings 组件逐行调用 deleteAction
- 文件: `apps/web/modules/ui/components/data-table/components/selected-row-settings.tsx:64-77`
```typescript
const handleDelete = async () => {
  const rowsToBeDeleted = table.getFilteredSelectedRowModel().rows.map((row) => row.id);
  const CHUNK_SIZE = 5;
  for (let i = 0; i < rowsToBeDeleted.length; i += CHUNK_SIZE) {
    const chunk = rowsToBeDeleted.slice(i, i + CHUNK_SIZE);
    if (type === "response") {
      await Promise.all(chunk.map((rowId) => deleteAction(rowId, { decrementQuotas })));  // ← 逐行调用
    } else {
      await Promise.all(chunk.map((rowId) => deleteAction(rowId)));
    }
  }
};
```

- `deleteAction` 是 `deleteResponseAction`，内部调用 `deleteResponse()`
- `deleteResponse()` 内部调用 `findAndDeleteUploadedFilesInResponse()`
- **理论上**每个答案的文件都会被清理，但受限于问题 1 和问题 2 的缺陷

#### 路径 B：删除整个问卷（deleteMany 无清理）

**证据 3.5**: `deleteResponsesAndDisplaysForSurvey` 直接 deleteMany，无文件清理
- 文件: `apps/web/app/(app)/workspaces/[workspaceId]/surveys/[surveyId]/(analysis)/summary/lib/survey.ts:7-37`
```typescript
export const deleteResponsesAndDisplaysForSurvey = async (surveyId: string) => {
  const [deletedResponsesCount, deletedDisplaysCount] = await prisma.$transaction([
    prisma.response.deleteMany({ where: { surveyId } }),  // ← 批量删除，无清理
    prisma.display.deleteMany({ where: { surveyId } }),
  ]);
  return { deletedResponsesCount: deletedResponsesCount.count, deletedDisplaysCount: deletedDisplaysCount.count };
};
```

**证据 3.6**: 删除问卷时调用上述函数
- 文件: `apps/web/app/(app)/workspaces/[workspaceId]/surveys/[surveyId]/(analysis)/summary/actions.ts:96-98`
```typescript
const { deletedResponsesCount, deletedDisplaysCount } = await deleteResponsesAndDisplaysForSurvey(
  parsedInput.surveyId
);
```

### 3.4 可复现场景（批量删除）

**场景 A：UI 批量删除 100 个答案**
1. 选中 100 个答案，点击删除
2. 前端分 20 批（每批 5 个）调用 `deleteResponseAction`
3. 每个 `deleteResponse` 内部调用 `findAndDeleteUploadedFilesInResponse`
4. **结果**: 受限于问题 1（题目变更漏删）和问题 2（相对路径解析失败），部分文件残留

**场景 B：删除整个问卷**
1. 在问卷详情页点击"删除所有响应"
2. 调用 `deleteResponsesAndDisplaysForSurvey`
3. `prisma.response.deleteMany({ where: { surveyId } })` 一次性删除所有答案
4. **结果**: **完全不清理任何文件**，所有附件永久残留

### 3.5 影响范围

| 操作 | 清理行为 | 残留风险 |
|------|---------|---------|
| 单个删除答案（Management API） | 调用 findAndDeleteUploadedFilesInResponse | ⚠️ 中（受问题 1/2 限制） |
| 单个删除答案（UI） | 调用 deleteResponseAction → deleteResponse | ⚠️ 中 |
| UI 批量删除答案 | 循环调用单个 deleteResponseAction | ⚠️ 中（每个都有问题 1/2 风险） |
| 更新答案（PUT） | 无清理逻辑 | ❌ 高（旧文件全部残留） |
| 删除问卷所有响应 | 直接 deleteMany，无清理 | ❌ 极高（全部残留） |
| 删除整个问卷 | 级联删除（取决于 Prisma schema） | ⚠️ 取决于外键配置 |

---

## 四、综合失败边界矩阵

| # | 场景 | 触发条件 | 代码路径 | 结果 | 影响范围 |
|---|------|---------|---------|------|---------|
| 1 | 题目删除后删答案 | FileUpload 题目被删除 | `findAndDeleteUploadedFilesInResponse` 过滤时找不到题目 ID | ❌ 文件残留 | 所有编辑过的问卷 |
| 2 | 题目 ID 变更后删答案 | 修改了 FileUpload 题目的 ID | 同上，历史答案 key 不匹配 | ❌ 文件残留 | 所有编辑过的问卷 |
| 3 | 题目类型变更后删答案 | FileUpload → 其他类型 | 同上，type 不匹配 | ❌ 文件残留 | 所有编辑过的问卷 |
| 4 | 相对路径删答案 | 数据库中是相对路径 | `new URL("/storage/...")` 抛异常 | ❌ 文件残留 | 所有未经过 Management 端编辑的答案 |
| 5 | 绝对路径删答案 | 数据库中是绝对路径（GET→PUT 后） | `new URL("https://...")` 成功 | ✅ 正常删除 | 经过 Management 端编辑的答案 |
| 6 | 更新答案替换文件 | PUT 更新答案中的文件 | 直接覆盖 data，无旧文件清理 | ❌ 旧文件残留 | 所有更新过文件的答案 |
| 7 | UI 批量删除答案 | 选中多个答案删除 | 循环调用单个 deleteResponse | ⚠️ 部分残留（受 #1-#4 影响） | 批量操作 |
| 8 | 删除问卷所有响应 | 点击"删除所有响应" | `deleteMany` 无清理 | ❌ 全部文件残留 | 问卷级操作 |
| 9 | 并行删除部分失败 | 批量删除中某几个失败 | Promise.all 中某个 reject，catch 吞掉 | ❌ 失败的文件残留 | 并发场景 |

---

## 五、代码位置索引

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| `findAndDeleteUploadedFilesInResponse`（旧版） | `apps/web/lib/response/service.ts` | 579-606 |
| `findAndDeleteUploadedFilesInResponse`（v2） | `apps/web/modules/api/v2/management/responses/[responseId]/lib/utils.ts` | 8-42 |
| `deleteResponse`（v2） | `apps/web/modules/api/v2/management/responses/[responseId]/lib/response.ts` | 92-139 |
| GET 转绝对路径（v2） | `apps/web/modules/api/v2/management/responses/[responseId]/route.ts` | 54-57 |
| GET 转绝对路径（v1） | `apps/web/app/api/v1/management/responses/[responseId]/route.ts` | 52-56 |
| PUT 直接写入（v2） | `apps/web/modules/api/v2/management/responses/[responseId]/lib/response.ts` | 141-155 |
| `resolveStorageUrlsInObject` | `apps/web/modules/storage/utils.ts` | 268-281 |
| `getOriginalFileNameFromUrl` | `apps/web/modules/storage/url-helpers.ts` | 12-29 |
| `validateFileUploads` | `apps/web/modules/storage/utils.ts` | 99-114 |
| UI 批量删除 | `apps/web/modules/ui/components/data-table/components/selected-row-settings.tsx` | 64-77 |
| `deleteResponsesAndDisplaysForSurvey` | `apps/web/app/(app)/workspaces/[workspaceId]/surveys/[surveyId]/(analysis)/summary/lib/survey.ts` | 7-37 |
| `deleteResponseAction` | `apps/web/modules/analysis/components/SingleResponseCard/actions.ts` | 145-174 |
