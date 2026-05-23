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

### 3.2 时间戳作为版本分界点

在 `apps/web/lib/survey/service.ts:526`：
```typescript
surveyData.updatedAt = new Date();
```

这个 `updatedAt` 就是**隐式的版本 ID**：
- 所有 `response.createdAt >= survey.updatedAt` 的响应 → 新版本响应
- 所有 `response.createdAt < survey.updatedAt` 的响应 → 旧版本响应

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

### 4.3 响应过滤机制

在 `apps/web/lib/response/utils.ts:153` 的 `buildWhereClause` 函数中：
```typescript
export const buildWhereClause = (survey: TSurvey, filterCriteria?: TResponseFilterCriteria) => {
  const whereClause: Prisma.ResponseWhereInput["AND"] = [];
  
  // 日期范围过滤
  if (filterCriteria?.createdAt) {
    const createdAt: { lte?: Date; gte?: Date } = {};
    if (filterCriteria?.createdAt?.max) createdAt.lte = filterCriteria?.createdAt?.max;
    if (filterCriteria?.createdAt?.min) createdAt.gte = filterCriteria?.createdAt?.min;
    whereClause.push({ createdAt });
  }
  // ... 其他过滤条件
};
```

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

### 5.3 回滚边界

**可以回滚的**：
- 调查状态：`inProgress` ↔ `paused` ↔ `completed`
- 问卷内容：可以修改回之前的配置（手动）
- 通过日期过滤"撤销"新版本响应的统计影响

**无法回滚的**：
- 已保存的响应数据（一旦提交永久保留）
- `survey.updatedAt` 时间戳（每次更新自动前进）
- 已产生的统计摘要（需要手动通过日期过滤修正）

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

### 6.2 设计缺点

1. **概念不直观**："隐式版本"对开发者和用户都不够直观
2. **查询复杂度**：版本分离需要在查询时动态计算时间戳
3. **无原子快照**：无法精确知道某个响应对应的问卷配置
4. **编辑风险高**：误操作可能导致数据混合，且无法一键回滚

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

---

## 七、关键数据流向图

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Draft Survey   │────▶│  Publish Action │────▶│  inProgress     │
│  (editable)     │     │  status=inProg  │     │  updatedAt=T1   │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                                         │
                                                         ▼
                                                 ┌─────────────────┐
                                                 │ Response #1     │
                                                 │ createdAt=T1+1  │◀── Link/App
                                                 └─────────────────┘
                                                         │
                                                         ▼
                                                 ┌─────────────────┐
                                                 │ Response #2     │
                                                 │ createdAt=T1+2  │◀── Link/App
                                                 └─────────────────┘
                                                         │
┌─────────────────┐     ┌─────────────────┐                │
│  Edit Survey    │────▶│ Update Action   │                │
│  (modify q's)   │     │  updatedAt=T2   │                │
└─────────────────┘     └─────────────────┘                │
                                                         │
                                                         ▼
                                                 ┌─────────────────┐
                                                 │ Response #3     │
                                                 │ createdAt=T2+1  │◀── Link/App
                                                 └─────────────────┘

版本分界点：T1 和 T2
- Response #1, #2 → 版本 1 (createdAt >= T1 and < T2)
- Response #3     → 版本 2 (createdAt >= T2)
```

---

## 八、总结

### 8.1 三个核心认知

1. **没有 SurveyVersion 表**：版本是通过 `survey.updatedAt` 时间戳隐式定义的
2. **响应不关联版本 ID**：只关联 `surveyId`，版本在查询时通过时间戳比较确定
3. **无一键回滚**：通过日期范围过滤和"复制问卷"实现版本管理

### 8.2 发布流程要点

- 发布 = 状态变更 + 时间戳更新 + 完整验证
- 两种分发渠道：Link（公开链接）和 App（SDK 触发）
- 编辑已发布问卷会产生数据一致性警告
- 推荐：复制问卷而非编辑已发布问卷

### 8.3 回滚边界

- ✅ 可以：暂停/恢复调查、手动恢复旧配置、通过日期过滤分离响应
- ❌ 不能：回滚到历史版本、撤销已提交的响应、恢复旧的 `updatedAt` 时间戳
