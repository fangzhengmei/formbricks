# 问卷发布流程：版本快照、响应通道与回滚边界

## 一、核心概念澄清

Formbricks 没有传统意义上的"版本快照表"（SurveyVersion），而是采用**基于时间戳的隐式版本机制**。理解这一点是关键。

### 1.1 为什么"冻结编辑态"难以理解

直觉上我们期望：
- 发布时创建一个独立的版本记录
- 响应与特定版本 ID 关联
- 可以随时回滚到历史版本

但 Formbricks 的实现更为简洁：
- 使用 `survey.updatedAt` 作为版本分界点
- 响应通过 `response.createdAt` 与 `survey.updatedAt` 的时间先后关系隐式关联到"版本"
- 没有独立的版本回滚机制，而是通过日期过滤实现版本分离

### 1.2 可落地的关键结论（必读）

经过代码级验证，以下两点是理解机制的核心：

| 认知修正 | 代码证据 | 实际影响 |
|---------|---------|---------|
| **响应查询是手动按 `createdAt` 过滤，而非自动按 `survey.updatedAt` 分桶** | `buildWhereClause` 接收 `survey` 参数但完全不使用 `survey.updatedAt` | 版本关联是"概念上的"，代码中没有自动分桶逻辑 |
| **草稿自动保存和正式发布都会推进 `updatedAt`** | `updateSurveyInternal:526` 无条件设置 `surveyData.updatedAt = new Date()` | 版本边界模糊，无法区分"发布版本"和"草稿保存版本" |

---

## 二、发布流程全景

### 2.1 发布状态机

调查状态定义在 `packages/database/schema.prisma:226-231`：
```prisma
enum SurveyStatus {
  draft      // 草稿态
  inProgress // 发布中
  paused     // 已暂停
  completed  // 已完成
}
```

### 2.2 完整发布调用链

```
前端触发 (survey-menu-bar.tsx:445)
    ↓
handleSurveyPublish()  [设置 status = "inProgress"]
    ↓
updateSurveyAction()  (apps/web/modules/survey/editor/actions.ts:244)
    ├─ 权限检查 (owner/manager 或 readWrite)
    ├─ Spam Protection 权限检查
    ├─ Follow-ups 权限检查
    ├─ 外部 URL 权限检查
    └─ 审计日志
        ↓
updateSurvey()  (apps/web/modules/survey/editor/lib/survey.ts:11)
    ↓
updateSurveyInternal()  (apps/web/lib/survey/service.ts:273)
    ├─ ZSurvey 完整验证（草稿跳过）
    ├─ 图片/媒体验证
    ├─ 语言更新逻辑
    ├─ Triggers 更新 (handleTriggerUpdates)
    ├─ Segment 处理（app 类型自动创建私有 segment）
    ├─ FollowUps 处理
    ├─ 清除 isDraft 标记 (stripIsDraftFromBlocks)
    ├─ 调度时间规范化 (normalizeSurveyScheduling)
    ├─ Prisma 更新（自动更新 updatedAt）
    └─ 调度调和 (reconcilePersistedSurveySchedulingIfDue)
        ↓
返回更新后的 Survey 对象（包含新的 updatedAt）
```

### 2.3 关键代码位置

| 阶段 | 文件 | 行号 |
|------|------|------|
| 前端发布按钮点击 | `apps/web/modules/survey/editor/components/survey-menu-bar.tsx` | 445-493 |
| Action 层验证 | `apps/web/modules/survey/editor/actions.ts` | 244-322 |
| 核心发布逻辑 | `apps/web/lib/survey/service.ts` | 273-556 |
| 调度自动化 | `apps/web/modules/survey/scheduling/lib/survey-scheduling.ts` | 181-229 |

---

## 三、版本快照生成机制（冻结编辑态）

### 3.1 "冻结"的本质

发布时没有创建独立的快照记录，"冻结"体现在：

1. **状态锁**：`status` 从 `draft` → `inProgress`
2. **时间戳锚点**：Prisma 自动更新 `survey.updatedAt` 为当前时间
3. **编辑警告**：后续编辑时弹出数据一致性警告

### 3.2 时间戳作为版本分界点（关键修正）

在 `apps/web/lib/survey/service.ts:526`：
```typescript
// ⚠️  重要：这行代码在 skipValidation 判断之外
// 无论是否是草稿保存，都会执行
surveyData.updatedAt = new Date();
```

这个 `updatedAt` 就是**隐式的版本 ID**：
- 所有 `response.createdAt >= survey.updatedAt` 的响应 → 新版本响应
- 所有 `response.createdAt < survey.updatedAt` 的响应 → 旧版本响应

### 3.2.1 草稿自动保存也会推进 `updatedAt`

**两个更新入口（`service.ts:558-565`）：**
```typescript
// 正式更新（发布时调用）
export const updateSurvey = async (updatedSurvey: TSurvey): Promise<TSurvey> => {
  return updateSurveyInternal(updatedSurvey);  // skipValidation = false
};

// 草稿保存（自动保存调用）
export const updateSurveyDraft = async (updatedSurvey: TSurvey): Promise<TSurvey> => {
  return updateSurveyInternal(updatedSurvey, true);  // skipValidation = true
};
```

**`updateSurveyInternal` 中的时间戳推进（无条件执行）：**
```typescript
export const updateSurveyInternal = async (
  updatedSurvey: TSurvey,
  skipValidation = false
): Promise<TSurvey> => {
  if (!skipValidation) {
    validateInputs([updatedSurvey, ZSurvey]);  // 仅正式发布验证
  }
  
  // ... 其他逻辑 ...
  
  // ⚠️  关键：无论 skipValidation 是 true 还是 false，以下代码都会执行
  surveyData.updatedAt = new Date();  // 无条件推进时间戳
  surveyData.publishOn = normalizedScheduling.publishOn;
  surveyData.closeOn = normalizedScheduling.closeOn;
  
  data = { ...surveyData };
  return prisma.survey.update({ where: { id: surveyId }, data });
};
```

### 3.2.2 草稿自动保存触发频率

在 `survey-menu-bar.tsx:281-333`：
- 每 10 秒自动保存一次（如果有修改）
- 问卷内容变化、问题移动、选项修改都会触发

**实际时间戳推进场景：**
```
T0: 创建问卷，updatedAt = T0
T0+10s: 自动保存草稿，updatedAt = T0+10s  ← 不是"发布版本"
T0+20s: 自动保存草稿，updatedAt = T0+20s  ← 不是"发布版本"
T0+30s: 自动保存草稿，updatedAt = T0+30s  ← 不是"发布版本"
T0+35s: 点击发布，updatedAt = T0+35s       ← 才是"发布版本"
T0+45s: 编辑后自动保存，updatedAt = T0+45s  ← 又是"非发布版本"
```

### 3.2.3 对版本边界的影响

| 场景 | updatedAt 推进 | 是否是"有效版本分界点" |
|------|---------------|----------------------|
| 首次发布 | ✅ 是 | ✅ 是（status 从 draft → inProgress） |
| 草稿自动保存 | ✅ 是（每10秒） | ❌ 否（问卷可能不完整） |
| 编辑已发布问卷（手动保存） | ✅ 是 | ⚠️ 可能是（取决于是否是有意修改） |
| 暂停/恢复调查 | ✅ 是 | ❌ 否（只是状态变更） |
| 修改问卷设置（非内容） | ✅ 是 | ❌ 否（不影响响应数据结构） |

**核心问题**：`updatedAt` 被过度使用了，它既是"最后修改时间"又是"版本分界点"，两者语义冲突。

### 3.3 草稿与发布的验证差异

**草稿保存** (`updateSurveyDraft`)：
- 跳过 Zod 完整验证 (`skipValidation = true`)
- 允许不完整的问卷配置
- 每 10 秒自动保存 (`survey-menu-bar.tsx:281-333`)

**正式发布** (`updateSurvey`)：
- 完整 Zod 验证 (`ZSurvey.safeParse`)
- 问题/区块/结束页 ID 唯一性检查
- 多语言标签完整性验证
- 触发条件有效性检查

### 3.4 发布时的数据清洗

在 `survey-menu-bar.tsx:384-403`：
```typescript
// 清除元素的 isDraft 标记
localSurvey.blocks = localSurvey.blocks.map((block) => ({
  ...block,
  elements: block.elements.map((element) => {
    const { isDraft, ...rest } = element;
    return rest;
  }),
}));
```

---

## 四、响应通道关联逻辑

### 4.1 两种分发渠道

调查类型定义在 `packages/types/surveys/types.ts:849`：
```typescript
export const ZSurveyType = z.enum(["link", "app"]);
```

#### 4.1.1 Link 类型（链接问卷）

**分发方式**：
- 公开分享链接 (`/s/[slug]`)
- 嵌入网页 (iframe)
- 邮件嵌入
- 个人链接（关联联系人）

**响应提交流程**：
```
用户访问链接
    ↓
survey-renderer.tsx 渲染问卷
    ↓
前端提交响应 (packages/surveys/src/lib/response.ts)
    ↓
/api/v2/client/[workspaceId]/responses
    ↓
createResponseWithQuotaEvaluation()  (apps/web/app/api/v2/client/[workspaceId]/responses/lib/response.ts:22)
    ├─ 配额评估 (evaluateResponseQuotas)
    ├─ 联系人关联
    └─ 响应创建（记录 createdAt）
```

#### 4.1.2 App 类型（应用内问卷）

**分发方式**：
- 通过 JavaScript SDK 在应用/网站中触发
- 基于 Action Class 触发（代码事件或无代码配置）
- 支持定向人群细分 (Segment)

**响应提交流程**：
```
SDK 监听触发事件 (action class)
    ↓
满足 Segment 过滤条件
    ↓
显示问卷 (packages/js-core)
    ↓
用户提交响应
    ↓
同 Link 类型的响应创建逻辑
```

### 4.2 响应与版本的隐式关联

**响应表结构** (`schema.prisma:158-190`)：
```prisma
model Response {
  id          String   @id @default(cuid())
  createdAt   DateTime @default(now())
  surveyId    String   // 关联到调查
  // ... 没有 versionId 字段！
}
```

**关联逻辑**：
- 响应只存储 `surveyId`，不存储版本 ID
- 版本关联是**查询时动态计算**的，通过比较 `response.createdAt` 和 `survey.updatedAt`

### 4.3 响应过滤机制（关键修正）

#### 4.3.1 `buildWhereClause` 不使用 `survey.updatedAt` 自动分桶

在 `apps/web/lib/response/utils.ts:153` 的 `buildWhereClause` 函数中：
```typescript
export const buildWhereClause = (survey: TSurvey, filterCriteria?: TResponseFilterCriteria) => {
  const whereClause: Prisma.ResponseWhereInput["AND"] = [];
  
  // ⚠️  重要：survey 参数传入但完全未使用
  // 没有任何代码使用 survey.updatedAt 来自动分桶
  
  // 日期范围过滤（仅来自用户手动选择）
  if (filterCriteria?.createdAt) {
    const createdAt: { lte?: Date; gte?: Date } = {};
    if (filterCriteria?.createdAt?.max) createdAt.lte = filterCriteria?.createdAt?.max;
    if (filterCriteria?.createdAt?.min) createdAt.gte = filterCriteria?.createdAt?.min;
    whereClause.push({ createdAt });
  }
  // ... 其他过滤条件（finished, contactAttributes, data, tags 等）
};
```

#### 4.3.2 `TResponseFilterCriteria` 类型定义

在 `packages/types/responses.ts:202-302` 中：
```typescript
export const ZResponseFilterCriteria = z.object({
  finished: z.boolean().optional(),
  responseIds: z.array(ZId).optional(),
  createdAt: z
    .object({
      min: z.date().optional(),
      max: z.date().optional(),
    })
    .optional(),
  // ⚠️  重要：没有 surveyUpdatedAt 或 versionId 字段
  contactAttributes: z.record(...).optional(),
  data: z.record(...).optional(),
  tags: z.array(...).optional(),
  meta: z.record(...).optional(),
  hiddenFields: z.record(...).optional(),
});
```

#### 4.3.3 实际查询流程（`surveySummary.ts:1051-1144`）

```typescript
const whereClause: Prisma.ResponseWhereInput = {
  surveyId,  // 只按 surveyId 过滤
  ...buildWhereClause(survey, filterCriteria),  // 只加用户手动选择的过滤条件
};

const responses = await prisma.response.findMany({
  where: whereClause,
  // ⚠️  没有任何版本分桶逻辑
  // 所有该 surveyId 下的响应都会返回，除非用户手动选了日期范围
});
```

#### 4.3.4 可落地结论

| 假设（错误） | 实际（正确） |
|-------------|-------------|
| 系统自动按 `survey.updatedAt` 分桶显示不同版本响应 | 用户必须手动选择日期范围才能分离不同版本 |
| 每个响应自动关联到发布时的版本 | 响应只关联 `surveyId`，版本关联是查询时的人为概念 |
| 查看统计时自动排除旧版本响应 | 统计默认包含所有响应，除非手动设置日期过滤 |

---

## 五、回滚边界与版本管理

### 5.1 编辑已发布问卷的警告

当 `responseCount > 0` 时，在 `survey-menu-bar.tsx:588-594` 显示警告：
```typescript
{responseCount > 0 && (
  <Alert variant="warning" size="small">
    <AlertTitle>{t("workspace.surveys.edit.caution_text")}</AlertTitle>
    <AlertButton onClick={() => setIsCautionDialogOpen(true)}>
      {t("common.learn_more")}
    </AlertButton>
  </Alert>
)}
```

警告对话框内容 (`edit-public-survey-alert-dialog/index.tsx:67-77`)：
> **Edit a published survey?**
> 
> This may cause data inconsistencies in the survey summary. We recommend duplicating the survey instead.
> 
> Here is what happens if you do:
> - Older and newer responses get mixed which can lead to misleading data summaries.
> - Responses before the change may not or only partially be included in the survey summary.
> - All data, including past responses, remain available as download on the survey summary page.

### 5.2 版本分离的实现方式

**没有版本回滚按钮**，但可以通过以下方式实现版本分离：

1. **日期范围过滤**（`CustomFilter.tsx`）：
   - 用户手动选择时间范围
   - 可选：过去 7 天、过去 30 天、本月、上月、自定义范围等
   - 本质是用 `response.createdAt` 过滤不同时间段的响应

2. **下载所有响应**：
   - 所有历史响应永久保留
   - 可导出为 CSV/Excel
   - 用户可在外部进行版本分析

### 5.3 回滚边界（关键修正）

#### 5.3.1 可以回滚的

| 操作 | 回滚方式 | 代码位置 |
|------|---------|---------|
| 调查状态 | `inProgress` ↔ `paused` ↔ `completed` | `survey-menu-bar.tsx:541-586` |
| 问卷内容 | 手动修改回之前的配置 | `updateSurveyInternal` |
| 统计视图 | 通过日期过滤"撤销"新版本响应的统计影响 | `surveySummary.ts:1001-1019` |

#### 5.3.2 无法回滚的

| 操作 | 不可回滚原因 | 代码证据 |
|------|-------------|---------|
| 已保存的响应数据 | 一旦提交永久保留 | `Response` 模型无软删除 |
| `survey.updatedAt` 时间戳 | 每次更新自动前进，无法后退 | `service.ts:526` 无条件 `new Date()` |
| 历史 `updatedAt` 记录 | 数据库只存当前值，不存历史 | `Survey` 模型无版本历史表 |
| 已产生的统计摘要 | 需手动通过日期过滤修正 | 无版本分桶逻辑 |

#### 5.3.3 版本边界模糊导致的回滚判断问题

**问题 1：无法确定哪些时间戳是"发布版本"**

```
T1: 草稿自动保存（updatedAt = T1）
T2: 草稿自动保存（updatedAt = T2）
T3: 正式发布（updatedAt = T3）
T4: 编辑后自动保存（updatedAt = T4）
T5: 编辑后手动保存（updatedAt = T5）

响应 R1 (createdAt = T3.5) → 应该关联 T3 版本
响应 R2 (createdAt = T4.5) → 应该关联 T4 还是 T5 版本？

// ⚠️  代码无法区分 T1-T5 中哪些是"有意发布"的版本
```

**问题 2：回滚判断需要用户自己记忆时间点**

没有版本历史列表，用户必须：
1. 自己记住"我在 5 月 20 日下午 3 点修改了问题 3"
2. 手动在日期过滤器中选择 5 月 20 日 3 点作为分界点
3. 无法一键"回滚到修改前的版本"

**问题 3：草稿自动保存可能产生"无效版本边界"**

```
T1: 发布版本（问题："您的年龄？"）
T2: 自动保存草稿（问题改成："您的年龄？（必填）"）← 只是加了必填
T3: 自动保存草稿（问题改成："您的年龄段？"）← 这才是真正修改

响应 R1 (T1.5) → 对应 T1 的问题
响应 R2 (T2.5) → 对应 T2 的问题（只是加了必填，问题相同）
响应 R3 (T3.5) → 对应 T3 的问题（问题完全不同）

// ⚠️  T2 产生了一个不必要的版本边界，但代码无法区分
```

#### 5.3.4 回滚判断的可落地操作建议

| 场景 | 推荐操作 | 原理 |
|------|---------|------|
| 只是修复错别字/样式 | 直接编辑，不分离版本 | 不影响响应数据结构 |
| 修改了问题选项/顺序 | 用日期过滤分离修改前后的响应 | 问题变化导致数据不可比 |
| 新增/删除了问题 | 用日期过滤分离，或复制问卷 | 响应字段结构变化 |
| 需要精确对比版本差异 | **复制问卷创建新版本** | 新旧问卷响应天然隔离 |

### 5.4 推荐的版本管理实践

系统推荐"复制问卷"而非编辑已发布问卷：
1. 旧问卷保持不变，继续收集响应
2. 复制问卷创建新版本，修改后发布
3. 新旧问卷的响应天然隔离，无需时间戳判断

---

## 六、代码设计权衡分析

### 6.1 设计优点

1. **简洁性**：不需要额外的版本表，减少数据冗余
2. **灵活性**：不需要预先定义版本，任何编辑都自动产生"新版本"
3. **性能**：响应写入不需要额外的版本 ID 查询
4. **存储效率**：不需要存储多份问卷配置副本

### 6.2 设计缺点（修正版）

| 设计问题 | 代码证据 | 实际影响 |
|---------|---------|---------|
| **概念不直观** | 无 `SurveyVersion` 表 | "隐式版本"对开发者和用户都不够直观 |
| **查询无自动分桶** | `buildWhereClause` 不使用 `survey.updatedAt` | 用户必须手动选择日期范围才能分离版本 |
| **`updatedAt` 语义过载** | 草稿自动保存也推进 `updatedAt` | 无法区分"发布版本"和"草稿保存版本" |
| **无原子快照** | 只存当前 `updatedAt`，无历史 | 无法精确知道某个响应对应的问卷配置 |
| **无版本历史** | 无版本历史表 | 无法一键回滚，需要用户手动记忆时间点 |
| **编辑风险高** | 误操作导致数据混合 | 无法一键回滚，只能通过日期过滤修正 |

### 6.3 关键优化点

在 `surveySummary.ts:1001-1019` 中使用游标分页避免大查询：
```typescript
// 使用游标分页替代 count + offset
while (hasMore) {
  const batch = await getResponsesForSummary(surveyId, batchSize, 0, filterCriteria, cursor);
  responses.push(...batch);
  if (batch.length < batchSize) {
    hasMore = false;
  } else {
    cursor = batch[batch.length - 1].id;
  }
}
```

### 6.4 潜在改进方向（基于代码分析）

如果需要改进版本管理机制，可以考虑：

1. **新增 `publishedAt` 字段**：仅在正式发布时更新，作为清晰的版本分界点
2. **新增 `SurveyVersion` 表**：存储每次发布的问卷配置快照和版本号
3. **在 `Response` 中增加 `versionId` 字段**：响应提交时关联到当前发布版本
4. **草稿保存不更新 `updatedAt`**：仅正式发布时更新，避免语义过载
5. **提供版本回滚功能**：一键切换到历史版本，自动按版本分桶显示统计

---

## 七、关键数据流向图（修正版）

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Draft Survey   │────▶│ Auto-save every │────▶│  updatedAt      │
│  (editing)      │     │ 10s (skipVal)   │     │  = T0+10s       │◀── 无意义推进
└─────────────────┘     └─────────────────┘     └─────────────────┘
          │                        ▲
          │                        │
          ▼                        │
┌─────────────────┐                │
│  Publish Action │                │
│  status=inProg  │                │
│  (full valid)   │                │
└─────────────────┘                │
          │                        │
          ▼                        │
┌─────────────────┐                │
│  inProgress     │                │
│  updatedAt=T1   │                │
└─────────────────┘                │
          │                        │
          ▼                        │
┌─────────────────┐                │
│ Response #1     │                │
│ createdAt=T1+1  │◀── Link/App    │
└─────────────────┘                │
          │                        │
          ▼                        │
┌─────────────────┐                │
│ Response #2     │                │
│ createdAt=T1+2  │◀── Link/App    │
└─────────────────┘                │
          │                        │
          ▼                        │
┌─────────────────┐                │
│  Edit Survey    │                │
│  (modify q's)   │                │
└─────────────────┘                │
          │                        │
          ├────────────────────────┘
          │
          ▼
┌─────────────────┐     ┌─────────────────┐
│ Update Action   │────▶│ updatedAt=T2    │
│  (manual save)  │     │ (full valid)    │
└─────────────────┘     └─────────────────┘
          │
          ▼
┌─────────────────┐
│ Response #3     │
│ createdAt=T2+1  │◀── Link/App
└─────────────────┘

版本分界点：T1 和 T2（但中间有 T0+10s 等无意义分界点）
- Response #1, #2 → 版本 1 (createdAt >= T1 and < T2)
- Response #3     → 版本 2 (createdAt >= T2)
- T0+10s 等自动保存产生的时间戳 → 无法从代码层面区分是否是有效版本
```

⚠️  **关键提醒**：`T0+10s` 这种自动保存产生的 `updatedAt` 推进，没有任何代码标记它是"草稿保存"还是"正式发布"，因此无法精确划定版本边界。

---

## 八、总结

### 8.1 四个核心认知（修正版）

1. **没有 SurveyVersion 表**：版本是通过 `survey.updatedAt` 时间戳**隐式定义**的
2. **响应不关联版本 ID**：只关联 `surveyId`，版本关联是**查询时的人为概念**
3. **无一键回滚**：通过日期范围过滤和"复制问卷"实现版本管理
4. **草稿自动保存也推进 updatedAt**：这是最容易被忽略的关键细节

### 8.2 发布流程要点（修正版）

- 发布 = 状态变更 + 时间戳更新 + 完整验证
- 两种分发渠道：Link（公开链接）和 App（SDK 触发）
- 编辑已发布问卷会产生数据一致性警告
- **推荐：复制问卷而非编辑已发布问卷**（从根本上避免版本边界模糊问题）

### 8.3 回滚边界（修正版）

| ✅ 可以回滚 | ❌ 无法回滚 |
|-----------|-----------|
| 暂停/恢复调查 | 已提交的响应数据 |
| 手动恢复旧配置 | `survey.updatedAt` 时间戳 |
| 通过日期过滤分离响应 | 历史 `updatedAt` 记录 |
| | 已产生的统计摘要（需手动修正） |

### 8.4 可落地的版本管理建议

基于代码分析，给出以下操作建议：

1. **发布后尽量不要编辑**：如果只是修复错别字可以，但修改问题结构会破坏数据一致性
2. **重要修改请复制问卷**：复制后新旧问卷响应天然隔离，是最可靠的版本管理方式
3. **如需日期过滤，记录修改时间**：如果确实需要编辑已发布问卷，记下修改时间，后续用这个时间点过滤响应
4. **不要依赖 updatedAt 做精确版本分界**：它会被草稿自动保存频繁推进，只能作为粗略参考
5. **导出数据做离线分析**：如果需要精确的版本对比，导出所有响应后在外部按时间分段分析
