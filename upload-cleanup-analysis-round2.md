# 问卷附件链路证据化分析（Round 2）

## 一、链路全景与路径格式演变

```
上传接口返回 fileUrl → 客户端提交答案 → 服务端写入 Response.data → 读取时转绝对路径 → 删除时解析
     [相对路径]          [相对路径]          [相对路径]            [绝对路径]       [BUG! 相对路径解析失败]
```

---

## 二、逐段证据化分析

### 2.1 上传接口返回 fileUrl —— 相对路径

**证据 1**: `getSignedUrlForUpload` 返回相对路径
- 文件: `apps/web/modules/storage/service.ts:53-58`
```typescript
// Return relative path - can be resolved to absolute URL at runtime when needed
return ok({
  signedUrl: signedUrlResult.data.signedUrl,
  presignedFields: signedUrlResult.data.presignedFields,
  fileUrl: `/storage/${workspaceId}/${accessType}/${encodeURIComponent(updatedFileName)}`,
});
```

**路径格式示例**:
```
/storage/ws-abc123/private/report--fid--550e8400-e29b-41d4-a716-446655440000.pdf
```

---

### 2.2 客户端提交答案 —— 相对路径透传

**证据 2**: `ApiClient.uploadFile` 直接返回相对路径
- 文件: `packages/surveys/src/lib/api-client.ts:164-221`
```typescript
const json = (await response.json()) as TUploadFileResponse;
const { data } = json;
const { signedUrl, fileUrl, presignedFields } = data as {
  signedUrl: string;
  presignedFields: Record<string, string>;
  fileUrl: string;
};
// ... 直传 S3 后直接返回 fileUrl
return fileUrl;  // 相对路径
```

**证据 3**: 客户端组件将 fileUrl 存入答案数据
- 文件: `packages/surveys/src/components/elements/file-upload-element.tsx:105-106`
```typescript
const urls = convertToStringArray(files);  // 提取 file.url → 相对路径
onChange({ [element.id]: urls });  // 存入 responseData
```

**证据 4**: 提交答案时透传 data
- 文件: `packages/surveys/src/lib/api-client.ts:76-90`
```typescript
async createResponse(responseInput) {
  return makeRequest(
    this.appUrl,
    `/api/${fromV1 ? "v1" : "v2"}/client/${this.workspaceId}/responses`,
    "POST",
    responseInput  // data 字段是相对路径
  );
}
```

---

### 2.3 服务端写入 Response.data —— 相对路径落库

**证据 5**: v1 API 直接透传 data 到 Prisma
- 文件: `apps/web/app/api/v1/lib/utils.ts:31`
```typescript
const buildPrismaResponseData = (...) => {
  return {
    // ...
    data: data,  // 原样写入，未转换
    // ...
  };
};
```

**证据 6**: v2 API 同样直接透传
- 文件: `apps/web/app/api/v2/client/[workspaceId]/responses/lib/response.ts:73`
```typescript
const buildPrismaResponseData = (...) => {
  return {
    // ...
    data: data,  // 原样写入
    // ...
  };
};
```

**数据库中的真实存储格式**（`Response.data` 字段）:
```json
{
  "file-upload-question-id-123": [
    "/storage/ws-abc123/private/report--fid--550e8400-e29b-41d4-a716-446655440000.pdf",
    "/storage/ws-abc123/private/image--fid--123e4567-e89b-12d3-a456-426614174000.jpg"
  ],
  "other-question": "text answer"
}
```

---

### 2.4 读取时转绝对路径（仅在 API 输出层）

**证据 7**: Management API 返回前调用 `resolveStorageUrlsInObject`
- 文件: `apps/web/modules/api/v2/management/responses/route.ts:48`
```typescript
data: workspaceResponses.map((r) => ({
  ...r,
  data: resolveStorageUrlsInObject(r.data)  // 相对路径 → 绝对路径
}))
```

**证据 8**: `resolveStorageUrlAuto` 实现
- 文件: `apps/web/modules/storage/utils.ts:250-254`
```typescript
const STORAGE_URL_PATTERN = /^\/storage\/[^/]+\/(public|private)\/.+/;

const isStorageUrl = (value: string): boolean => STORAGE_URL_PATTERN.test(value);

export const resolveStorageUrlAuto = (url: string): string => {
  if (!isStorageUrl(url)) return url;
  const accessType = url.includes("/private/") ? "private" : "public";
  return resolveStorageUrl(url, accessType);  // 拼接 WEBAPP_URL + 相对路径
};
```

**证据 9**: `resolveStorageUrl` 实现
- 文件: `apps/web/modules/storage/utils.ts:225-243`
```typescript
export const resolveStorageUrl = (url: string, accessType: "public" | "private" = "public"): string => {
  if (!url) return "";
  // Already absolute URL - return as-is
  if (url.startsWith("http://") || url.startsWith("https://")) {
    return url;
  }
  // Relative path - resolve with base URL
  if (url.startsWith("/storage/")) {
    const baseUrl = accessType === "public" ? getPublicDomain() : WEBAPP_URL;
    return `${baseUrl}${url}`;
  }
  return url;
};
```

**转换后输出给前端的格式**:
```json
{
  "file-upload-question-id-123": [
    "https://app.formbricks.com/storage/ws-abc123/private/report--fid--550e8400-e29b-41d4-a716-446655440000.pdf"
  ]
}
```

---

## 三、删除答案时的路径解析 —— 核心 BUG

### 3.1 两套删除实现的 URL 解析逻辑

**证据 10**: 旧版 `findAndDeleteUploadedFilesInResponse`
- 文件: `apps/web/lib/response/service.ts:590-602`
```typescript
const deletionPromises = fileUrls.map(async (fileUrl) => {
  try {
    const { pathname } = new URL(fileUrl);  // ← 问题所在！
    const [, storageId, accessType, fileName] = pathname.split("/").filter(Boolean);
    if (!storageId || !accessType || !fileName) {
      throw new Error(`Invalid file path: ${pathname}`);
    }
    return deleteFile(storageId, accessType as "private" | "public", fileName, survey.workspaceId);
  } catch (error) {
    logger.error(error, `Failed to delete file ${fileUrl}`);  // 静默吞掉
  }
});
```

**证据 11**: API v2 版 `findAndDeleteUploadedFilesInResponse`
- 文件: `apps/web/modules/api/v2/management/responses/[responseId]/lib/utils.ts:25-36`
```typescript
const deletionPromises = fileUrls.map(async (fileUrl) => {
  try {
    const { pathname } = new URL(fileUrl);  // ← 同样的问题！
    const [, storageId, accessType, fileName] = pathname.split("/").filter(Boolean);
    if (!storageId || !accessType || !fileName) {
      throw new Error(`Invalid file path: ${pathname}`);
    }
    return deleteFile(storageId, accessType as "private" | "public", fileName, workspaceId);
  } catch (error) {
    logger.error({ error, fileUrl }, "Failed to delete file");  // 静默吞掉
  }
});
```

### 3.2 相对路径 vs 绝对路径的行为差异

| 路径类型 | `new URL(fileUrl)` 行为 | 结果 |
|---------|------------------------|------|
| **相对路径**<br>`/storage/ws-abc/private/file.pdf` | 抛出 `TypeError: Invalid URL`<br>（`new URL()` 要求绝对 URL 或 base 参数） | ❌ 解析失败，文件残留 |
| **绝对路径**<br>`https://app.com/storage/ws-abc/private/file.pdf` | 成功解析，`pathname` 为 `/storage/ws-abc/private/file.pdf` | ✅ 正常删除 |

### 3.3 `new URL()` 规范验证

```javascript
// 相对路径 - 失败
new URL("/storage/ws-abc/private/file.pdf")
// → Uncaught TypeError: Invalid URL

// 相对路径 + base - 成功
new URL("/storage/ws-abc/private/file.pdf", "https://example.com")
// → URL { href: "https://example.com/storage/ws-abc/private/file.pdf", ... }

// 绝对路径 - 成功
new URL("https://app.com/storage/ws-abc/private/file.pdf")
// → URL { pathname: "/storage/ws-abc/private/file.pdf", ... }
```

---

## 四、失败分支与后果分析

### 4.1 场景 1: 数据库中是相对路径（99% 正常场景）

**触发条件**: 正常流程，Response.data 中是相对路径
**代码路径**: `fileUrl` → `new URL(fileUrl)` → 抛异常 → catch 吞掉
**后果**:
- 异常仅打日志 `Failed to delete file`
- 文件永久残留在 S3
- 用户看不到任何错误提示
- 无重试机制

### 4.2 场景 2: 数据库中碰巧是绝对路径（边缘场景）

**触发条件**: 旧数据、人工修改、或某些 API 直接写入绝对路径
**代码路径**: `fileUrl` → `new URL(fileUrl)` → 成功 → 解析 pathname → 删除
**后果**: ✅ 文件正常删除

### 4.3 场景 3: S3 侧删除失败

**触发条件**: 权限问题、文件已被删除、S3 服务不可用
**代码路径**: `deleteFile` → S3 返回错误 → 抛异常 → catch 吞掉
**后果**:
- 仅打日志
- 该文件残留
- 无重试机制

### 4.4 场景 4: 删除 Response 后、清理文件前失败

**API v2 代码路径** [apps/web/modules/api/v2/management/responses/[responseId]/lib/response.ts:92-118]:
```typescript
export const deleteResponse = async (responseId: string) => {
  // 1. 删除 Response（成功）
  const deletedResponse = await prisma.response.delete({...});

  // 2. 删除 display（可能失败，直接 return error）
  if (deletedResponse.displayId) {
    const deleteDisplayResult = await deleteDisplay(deletedResponse.displayId);
    if (!deleteDisplayResult.ok) {
      return deleteDisplayResult;  // ← 直接返回，文件清理不执行
    }
  }

  // 3. 获取 survey questions（可能失败，直接 return error）
  const surveyQuestionsResult = await getSurveyQuestions(deletedResponse.surveyId);
  if (!surveyQuestionsResult.ok) {
    return { ok: false, error: surveyQuestionsResult.error };  // ← 直接返回，文件清理不执行
  }

  // 4. 清理文件（如果前面都成功才执行）
  await findAndDeleteUploadedFilesInResponse(...);
};
```

**后果**:
- Response 已从数据库删除
- 但 display 删除失败或 survey 查询失败时，文件清理不执行
- 文件永久残留，且 Response 已删除，无法再次触发清理

### 4.5 场景 5: 批量删除答案

**问题**: 未找到批量删除 API 调用 `findAndDeleteUploadedFilesInResponse` 的代码
**后果**: 批量删除时所有附件都残留

---

## 五、测试覆盖问题

**证据 12**: 测试 mock 了 `findAndDeleteUploadedFilesInResponse`
- 文件: `apps/web/modules/api/v2/management/responses/[responseId]/lib/tests/response.test.ts:45-47`
```typescript
vi.mock("../utils", () => ({
  findAndDeleteUploadedFilesInResponse: vi.fn(),
}));
```

**证据 13**: mock 数据中的 `fileUrl` 是假值
- 文件: `apps/web/modules/api/v2/management/responses/[responseId]/lib/tests/__mocks__/response.mock.ts:7`
```typescript
export const responseInput: Omit<Response, "id"> = {
  data: { file: "fileUrl" },  // 假值，不是真实路径格式
  // ...
};
```

**后果**: 单元测试无法发现相对路径解析失败的问题。

---

## 六、修复建议

### 6.1 立即修复: 兼容相对路径解析

修改 `findAndDeleteUploadedFilesInResponse` 中的 URL 解析：

```typescript
// 修复前
const { pathname } = new URL(fileUrl);

// 修复后 - 处理相对路径
let pathname: string;
if (fileUrl.startsWith("/storage/")) {
  pathname = fileUrl;
} else {
  try {
    pathname = new URL(fileUrl).pathname;
  } catch (error) {
    logger.error({ error, fileUrl }, "Invalid file URL");
    return;
  }
}
```

### 6.2 增加重试机制

文件删除失败时，将失败记录写入队列，定时重试。

### 6.3 清理前置

将文件清理移到事务内或事务前，避免 Response 删除后无法重试。

### 6.4 批量删除支持

在批量删除 API 中增加文件清理逻辑。

### 6.5 增加测试覆盖

用真实路径格式（相对路径 + 绝对路径）测试 `findAndDeleteUploadedFilesInResponse`。

---

## 七、代码位置索引

| 功能 | 文件位置 |
|------|---------|
| 上传返回相对路径 | `apps/web/modules/storage/service.ts:53-58` |
| 客户端透传 | `packages/surveys/src/lib/api-client.ts:221` |
| 服务端落库（v1） | `apps/web/app/api/v1/lib/utils.ts:31` |
| 服务端落库（v2） | `apps/web/app/api/v2/client/[workspaceId]/responses/lib/response.ts:73` |
| 读取时转绝对路径 | `apps/web/modules/storage/utils.ts:250-281` |
| 旧版删除清理（BUG） | `apps/web/lib/response/service.ts:590-602` |
| v2 版删除清理（BUG） | `apps/web/modules/api/v2/management/responses/[responseId]/lib/utils.ts:25-36` |
| 删除响应主流程 | `apps/web/modules/api/v2/management/responses/[responseId]/lib/response.ts:92-139` |
| URL 解析工具 | `apps/web/modules/storage/utils.ts:225-243` |
