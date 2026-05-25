# 问卷附件回收安全边界深度分析（Round 4）

## 一、附件 URL 信任链与 workspace 归属校验缺口

### 1.1 删除调用链溯源

```
Response.data["q_file_001"] = ["/storage/ws-abc/private/file--fid--uuid.pdf"]
                        ↓ （URL 被完全信任，无校验）
findAndDeleteUploadedFilesInResponse()
                        ↓
const [, storageId, accessType, fileName] = pathname.split("/").filter(Boolean);
                        ↓ （storageId 直接从 URL 提取，无归属校验）
deleteFile(storageId, accessType, fileName, workspaceId)
                        ↓
deleteFileFromS3(`${storageId}/${accessType}/${fileName}`)  // ← 直接用 storageId
```

### 1.2 代码证据：无 workspace 归属校验

**证据 1.1**: API v2 版清理逻辑 - storageId 直接从 URL 提取
- 文件: `apps/web/modules/api/v2/management/responses/[responseId]/lib/utils.ts:25-33`
```typescript
const deletionPromises = fileUrls.map(async (fileUrl) => {
  try {
    const { pathname } = new URL(fileUrl);
    const [, storageId, accessType, fileName] = pathname.split("/").filter(Boolean);

    if (!storageId || !accessType || !fileName) {
      throw new Error(`Invalid file path: ${pathname}`);
    }
    return deleteFile(storageId, accessType as "private" | "public", fileName, workspaceId);
    //                                                                 ↑ 只是 fallback
  } catch (error) {
    logger.error({ error, fileUrl }, "Failed to delete file");
  }
});
```

**证据 1.2**: 旧版清理逻辑 - 同样无校验
- 文件: `apps/web/lib/response/service.ts:590-599`
```typescript
const deletionPromises = fileUrls.map(async (fileUrl) => {
  try {
    const { pathname } = new URL(fileUrl);
    const [, storageId, accessType, fileName] = pathname.split("/").filter(Boolean);
    if (!storageId || !accessType || !fileName) {
      throw new Error(`Invalid file path: ${pathname}`);
    }
    return deleteFile(storageId, accessType as "private" | "public", fileName, survey.workspaceId);
    //                                                                      ↑ 只是 fallback
  } catch (error) {
    logger.error(error, `Failed to delete file ${fileUrl}`);
  }
});
```

**证据 1.3**: `deleteFile` 的第四个参数只是 fallbackId
- 文件: `apps/web/modules/storage/service.ts:105-118`
```typescript
// Deletes a file from S3. Tries the primary ID path first; if the file is not found and a
// fallbackId is provided, retries with the fallback path (backwards compat for old environmentId paths).
export const deleteFile = async (
  primaryId: string,      // ← 从 URL 提取的 storageId，无校验
  accessType: TAccessType,
  fileName: string,
  fallbackId?: string     // ← workspaceId，仅在 primary 路径找不到时尝试
) => {
  const result = await deleteFileFromS3(`${primaryId}/${accessType}/${fileName}`);
  // ↑ 直接用 primaryId（storageId）构建 S3 key

  if (!result.ok && result.error.code === StorageErrorCode.FileNotFoundError && fallbackId) {
    return await deleteFileFromS3(`${fallbackId}/${accessType}/${fileName}`);
  }

  return result;
};
```

### 1.3 信任链问题总结

| 环节 | 校验行为 | 风险 |
|------|---------|------|
| URL 来源 | 完全信任 `Response.data` 中的 URL，不校验是否由本系统生成 | 注入恶意路径 |
| storageId 提取 | 直接从 URL pathname 分割提取 | 可伪造其他 workspace ID |
| workspace 归属校验 | **无校验**！不验证 `storageId === workspaceId` | 跨租户误删 |
| fallbackId 用途 | 仅用于兼容旧的 environmentId 路径 | 不是安全校验 |

---

## 二、跨租户误删攻击场景

### 2.1 攻击前提

1. **攻击者需要能控制某个答案的 `data` 字段**
   - 通过 Management API 的 PUT 操作（需认证，但如果 API Key 泄露）
   - 或通过客户端 API 的某种注入漏洞

2. **需要知道或猜测目标 workspace ID**
   - Workspace ID 格式：`ws_` 前缀 + 随机字符串（可通过公开信息或枚举）

3. **需要知道或猜测目标文件名**
   - 文件名格式：`{originalName}--fid--{UUID}.{extension}`
   - UUID 增加了猜测难度，但并非不可能

### 2.2 可复现攻击路径

**场景 A：通过 Management API PUT 注入**

```
1. 攻击者拥有自己的 workspace（ws-attacker）和 API Key
2. 在自己的 workspace 中创建一个问卷并提交答案
3. 调用 PUT /api/v2/management/responses/{responseId}，注入恶意 data：
   {
     "data": {
       "q_file_001": [
         "/storage/ws-victim/private/target-file--fid--uuid.pdf"
       ]
     }
   }
4. 调用 DELETE /api/v2/management/responses/{responseId} 删除答案
5. 系统尝试删除：`ws-victim/private/target-file--fid--uuid.pdf`
6. 如果文件存在 → ✅ 成功删除受害者的文件
```

**代码验证**：Management API 的 PUT 操作只验证文件扩展名，不验证路径归属
- 文件: `apps/web/modules/storage/utils.ts:88-114`
```typescript
export const validateSingleFile = (fileUrl: string, allowedFileExtensions?: TAllowedFileExtension[]): boolean => {
  const fileName = getOriginalFileNameFromUrl(fileUrl);  // ← 只提取文件名
  if (!fileName) return false;
  const extension = extractFileExtension(fileName);
  if (!extension) return false;
  return !allowedFileExtensions || allowedFileExtensions.includes(extension as TAllowedFileExtension);
  // ↑ 只验证扩展名，不验证 workspace 归属！
};
```

**场景 B：客户端答案更新注入**（如果存在漏洞）

```
1. 受害者提交答案，上传文件：
   /storage/ws-victim/private/secret--fid--1234.pdf
2. 攻击者通过某种方式控制了该答案的 data 字段
3. 将 fileUrl 改为指向其他文件（同 workspace 内的横向移动）
4. 删除答案时，删除的是攻击者指定的文件
```

### 2.3 影响范围评估

| 攻击场景 | 可行性 | 影响 | 防护状态 |
|---------|--------|------|---------|
| 跨 workspace 删除 | ⚠️ 中高（需 API Key + 猜测文件名） | 其他租户文件被删 | ❌ 无防护 |
| 同 workspace 横向 | ⚠️ 中（需控制 data 字段） | 同 workspace 内任意文件被删 | ❌ 无防护 |
| 删除系统文件 | ❌ 低（路径被 storageId 限制） | 无法跳出 workspace 目录 | ✅ 路径分割限制 |

---

## 三、v1 与 v2 答案更新语义对比

### 3.1 架构复用关系

**证据 3.1**: v2 client API 完全复用 v1 的 PUT handler
- 文件: `apps/web/app/api/v2/client/[workspaceId]/responses/[responseId]/route.ts:1-3`
```typescript
import { OPTIONS, PUT } from "@/app/api/v1/client/[workspaceId]/responses/[responseId]/route";
export { OPTIONS, PUT };
```

### 3.2 更新语义对比

| 维度 | v1 Client API | v2 Client API | v2 Management API |
|------|--------------|--------------|------------------|
| 实现代码 | `put-response-handler.ts` | **复用 v1** | `updateResponseWithQuotaEvaluation` |
| data 更新方式 | 直接覆盖 | **直接覆盖** | 直接覆盖 |
| 旧文件清理 | ❌ 无 | ❌ **无** | ❌ 无 |
| workspace 校验 | ✅ `survey.workspaceId === workspaceId` | ✅ **同 v1** | ✅ API Key 权限校验 |
| 文件验证 | `validateResponse` | ✅ **同 v1** | `validateFileUploads` |

### 3.3 附件替换与残留差异边界

**共同点（所有路径都不清理旧文件）**：

```typescript
// v1/v2 Client API: apps/web/app/api/v1/client/[workspaceId]/responses/[responseId]/lib/put-response-handler.ts:170
const updatedResponse = await updateResponseWithQuotaEvaluation(responseId, responseUpdateInput);
// ↑ 直接更新 data，不对比新旧文件差异

// v2 Management API: apps/web/modules/api/v2/management/responses/[responseId]/lib/response.ts:148
const updatedResponse = await prismaClient.response.update({
  where: { id: responseId },
  data: responseInput,  // ↑ 直接覆盖，不清理旧文件
});
```

**差异点**：

| 场景 | v1/v2 Client API | v2 Management API |
|------|-----------------|------------------|
| 用户更新答案中的文件 | 用户重新上传 → 旧文件残留 | 管理员 PUT 新 URL → 旧文件残留 |
| 路径格式 | 相对路径（新上传） | 可能是绝对路径（GET→PUT 回写） |
| 删除时解析结果 | ❌ 相对路径解析失败 → 新文件也删不掉 | ✅ 绝对路径解析成功 → 新文件能删掉 |
| 最终残留 | 旧文件 + 新文件（都删不掉） | 旧文件残留，新文件能删掉 |

**反直觉结果**：经过 Management API 编辑的答案，虽然混存了绝对路径，但**删除时新文件反而能被正确删掉**，只有旧文件残留。而未编辑过的答案，新旧文件都删不掉。

---

## 四、孤儿文件累积场景与量化分析

### 4.1 累积场景全景

| # | 场景 | 触发频率 | 每个操作残留文件数 | 累积速度 |
|---|------|---------|-------------------|---------|
| 1 | 更新答案替换文件 | 高（用户频繁修改） | N（被替换的文件数） | 快 |
| 2 | 删除问卷所有响应 | 中（问卷生命周期结束） | N × 平均附件数 | 极快 |
| 3 | 题目变更后删除答案 | 中（问卷迭代） | N（历史答案附件数） | 中 |
| 4 | 相对路径解析失败 | 极高（所有正常删除） | N（所有文件） | 极快 |
| 5 | 批量删除 UI 操作失败 | 中（网络/并发问题） | K（失败的答案数 × 平均附件数） | 中 |
| 6 | display/survey 查询失败 | 低（异常情况） | N（当前答案的附件数） | 慢 |

### 4.2 典型场景量化

假设：
- 每个 FileUpload 题目平均上传 1.2 个文件
- 活跃问卷有 1000 份答案
- 10% 的答案会被更新（平均替换 1 个文件）
- 50% 的问卷会被最终删除所有响应

**单问卷生命周期的孤儿文件估算**：

```
1. 正常提交：1000 × 1.2 = 1200 个文件
2. 更新答案残留：1000 × 10% × 1 = 100 个文件
3. 删除所有响应残留：1200 + 100 = 1300 个文件（全部残留）

总残留：1300 个文件 / 问卷
```

**考虑相对路径解析失败的情况**：

即使问卷不删除所有响应，只是逐个删除答案：
```
逐个删除 1000 份答案：
- 每份答案的相对路径都解析失败 → 1200 个文件全部残留
- 加上更新残留的 100 个 → 总计 1300 个
```

**结论**：无论逐个删除还是批量删除问卷所有响应，**几乎所有附件最终都会成为孤儿文件**。

---

## 五、综合安全矩阵与修复建议

### 5.1 安全问题矩阵

| # | 问题 | 严重程度 | 影响 | 修复优先级 |
|---|------|---------|------|-----------|
| 1 | 无 workspace 归属校验 | 🔴 高危 | 跨租户误删 | P0 |
| 2 | 相对路径解析失败 | 🔴 高危 | 所有文件删除失败 | P0 |
| 3 | 更新答案不清理旧文件 | 🟠 中高 | 大量孤儿文件 | P1 |
| 4 | 删除问卷所有响应不清理 | 🟠 中高 | 批量残留 | P1 |
| 5 | 题目变更后漏删 | 🟡 中 | 历史答案附件残留 | P2 |
| 6 | 异常分支提前 return | 🟡 中 | 部分文件残留 | P2 |

### 5.2 修复建议

**建议 1：增加 workspace 归属校验（P0）**

```typescript
// 删除前校验 storageId === workspaceId
const deletionPromises = fileUrls.map(async (fileUrl) => {
  try {
    const { pathname } = new URL(fileUrl);
    const [, storageId, accessType, fileName] = pathname.split("/").filter(Boolean);
    
    // 新增：归属校验
    if (storageId !== workspaceId) {
      logger.warn({ storageId, workspaceId, fileUrl }, "Attempted to delete file from different workspace");
      return;  // 跳过，不删除
    }
    
    return deleteFile(storageId, accessType as "private" | "public", fileName);
  } catch (error) {
    logger.error({ error, fileUrl }, "Failed to delete file");
  }
});
```

**建议 2：修复相对路径解析（P0）**

```typescript
// 兼容相对路径和绝对路径
let pathname: string;
if (fileUrl.startsWith("/storage/")) {
  pathname = fileUrl;
} else {
  pathname = new URL(fileUrl).pathname;
}
```

**建议 3：更新答案时清理旧文件（P1）**

```typescript
// 更新前对比新旧 data，删除被替换的文件
const oldFiles = extractFileUrls(oldResponse.data, fileUploadQuestions);
const newFiles = extractFileUrls(newData, fileUploadQuestions);
const filesToDelete = oldFiles.filter(f => !newFiles.includes(f));

await deleteFiles(filesToDelete, workspaceId);
```

**建议 4：删除问卷时批量清理（P1）**

```typescript
// deleteResponsesAndDisplaysForSurvey 中增加文件清理
export const deleteResponsesAndDisplaysForSurvey = async (surveyId: string) => {
  // 1. 先查询所有答案的附件
  const responses = await prisma.response.findMany({
    where: { surveyId },
    select: { data: true }
  });
  
  // 2. 清理所有附件
  await deleteAllFilesFromResponses(responses);
  
  // 3. 再批量删除答案
  return prisma.$transaction([...]);
};
```

**建议 5：不依赖题目 ID 过滤（P2）**

```typescript
// 直接从 data 中提取所有 /storage/ 路径，不依赖题目类型
const fileUrls = Object.values(responseData)
  .flat()
  .filter((val): val is string => 
    typeof val === "string" && val.startsWith("/storage/")
  );
```

---

## 六、代码位置索引

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| `findAndDeleteUploadedFilesInResponse`（v2） | `apps/web/modules/api/v2/management/responses/[responseId]/lib/utils.ts` | 25-33 |
| `findAndDeleteUploadedFilesInResponse`（旧版） | `apps/web/lib/response/service.ts` | 590-599 |
| `deleteFile`（storage service） | `apps/web/modules/storage/service.ts` | 105-118 |
| v2 client API 复用 v1 | `apps/web/app/api/v2/client/[workspaceId]/responses/[responseId]/route.ts` | 1-3 |
| `validateSingleFile`（仅验证扩展名） | `apps/web/modules/storage/utils.ts` | 88-97 |
| `validateFileUploads` | `apps/web/modules/storage/utils.ts` | 99-114 |
| `updateResponseWithQuotaEvaluation` | `apps/web/modules/api/v2/management/responses/[responseId]/lib/response.ts` | 177-213 |
| `deleteResponsesAndDisplaysForSurvey` | `apps/web/app/(app)/workspaces/[workspaceId]/surveys/[surveyId]/(analysis)/summary/lib/survey.ts` | 7-37 |
