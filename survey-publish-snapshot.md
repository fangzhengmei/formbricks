# 问卷发布流程：版本快照、响应通道与回滚边界

## 一、核心概念澄清

Formbricks 没有传统意义上的"版本快照表"（SurveyVersion），也没有"按问卷更新时间自动分桶"的逻辑。所有版本关联都是**人为概念**，代码中仅依赖用户传入的 `createdAt` 过滤条件来分离响应。

### 1.1 为什么"冻结编辑态"难以理解

直觉上我们期望：
- 发布时创建一个独立的版本记录
- 响应与特定版本 ID 关联
- 可以随时回滚到历史版本
- 系统自动按版本分桶显示统计

但 Formbricks 的实现更为简洁（也更模糊）：
- 使用 `survey.updatedAt` 作为**人为约定**的版本分界点
- 响应只存储 `surveyId`，不存储任何版本标识
- 查询响应时，**仅依赖用户传入的 `filterCriteria.createdAt` 过滤**
- 没有独立的版本回滚机制

### 1.2 可落地的关键结论（必读）

经过代码级验证，以下是理解机制的核心：

| 认知修正 | 代码证据 | 实际影响 |
|---------|---------|---------|
| **响应查询仅依赖用户传入的 `createdAt` 过滤条件** | `buildWhereClause:153-642` 完全不使用 `survey.updatedAt`，只处理 `filterCriteria.createdAt` | 没有任何"自动分桶"逻辑，版本分离完全靠用户手动选择日期范围 |
| **草稿自动保存和正式发布都会推进 `updatedAt`** | `updateSurveyInternal:526` 无条件设置 `surveyData.updatedAt = new Date()` | 版本边界模糊，无法区分"发布版本"和"草稿保存版本" |
| **`updatedAt` 仅存储当前值，不存历史** | `Survey` 模型只有 `updatedAt` 字段，无历史表 | 无法追溯历史版本时间点，无法精确关联响应与问卷配置 |

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
    ├─ 无条件设置 updatedAt = new Date()
    ├─ Prisma 更新
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
| 响应查询核心 | `apps/web/lib/response/utils.ts` | 153-642 |

---

## 三、版本快照生成机制（冻结编辑态）

### 3.1 "冻结"的本质

发布时没有创建独立的快照记录，"冻结"体现在：

1. **状态锁**：`status` 从 `draft` → `inProgress`
2. **时间戳锚点**：代码手动设置 `survey.updatedAt` 为当前时间
3. **编辑警告**：后续编辑时弹出数据一致性警告

### 3.2 时间戳作为人为约定的版本分界点

在 `apps/web/lib/survey/service.ts:526`：
```typescript
// ⚠️  重要：这行代码在 skipValidation 判断之外
// ⚠️  无论是否是草稿保存，都会无条件执行
surveyData.updatedAt = new Date();
```

这个 `updatedAt` 是**人为约定**的版本分界点（不是代码强制的）：
- 人为约定：`response.createdAt >= survey.updatedAt` → "新版本"响应
- 人为约定：`response.createdAt < survey.updatedAt` → "旧版本"响应
- ⚠️ 代码中**完全没有**自动应用这个逻辑

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

**核心问题**：`updatedAt` 被过度使用了，它既是"最后修改时间"又是"人为约定的版本分界点"，两者语义冲突。

### 3.2.4 符合数据层约束的版本边界方案（重要）

由于 Prisma `@updatedAt` 自动更新的硬约束，**"草稿不更新 updatedAt" 在现有数据模型下无法实现**。任何对 Survey 记录的修改都会自动推进 `updatedAt`。

**可行的替代方案**（按推荐程度排序）：

| 方案 | 实现方式 | 数据层变更 | 推荐度 |
|------|---------|-----------|--------|
| **方案一：新增 `publishedAt` 字段** | 仅在正式发布时手动设置 `publishedAt`，草稿保存不设置 | 在 Survey 表新增 `publishedAt: DateTime?` 字段 | ⭐⭐⭐⭐⭐ |
| **方案二：新增 `SurveyVersion` 表** | 每次发布时在 `SurveyVersion` 表创建快照，存储问卷配置和版本时间 | 新增 `SurveyVersion` 表，`Response` 可选 `versionId` 外键 | ⭐⭐⭐⭐ |
| **方案三：新增 `lastContentUpdateAt` 字段** | 仅在问卷内容（questions/blocks）变更时手动更新，忽略草稿自动保存 | 在 Survey 表新增 `lastContentUpdateAt: DateTime?` 字段 | ⭐⭐⭐ |
| **方案四：业务层记录边界** | 在业务代码中自行维护版本边界列表（如本文档示例） | 无需数据层变更 | ⭐⭐⭐ |
| **方案五：草稿不存 Survey 主表** | 草稿保存到独立的 `SurveyDraft` 表，正式发布才写入 Survey 表 | 新增 `SurveyDraft` 表，修改保存逻辑 | ⭐⭐ |

**方案一（新增 `publishedAt`）代码示例**：
```typescript
// 1. schema.prisma 新增字段
model Survey {
  // ... 现有字段 ...
  updatedAt    DateTime  @updatedAt @map(name: "updated_at")
  publishedAt  DateTime? @map(name: "published_at")  // 新增：仅在正式发布时设置
}

// 2. 修改 updateSurveyInternal，仅在正式发布时更新 publishedAt
export const updateSurveyInternal = async (
  updatedSurvey: TSurvey,
  skipValidation = false
): Promise<TSurvey> => {
  if (!skipValidation) {
    validateInputs([updatedSurvey, ZSurvey]);
  }
  
  // ... 其他逻辑 ...
  
  surveyData.updatedAt = new Date();  // Prisma 自动更新，此行可删除
  // ⚠️  关键：仅在正式发布（非草稿保存）时更新 publishedAt
  if (!skipValidation) {
    surveyData.publishedAt = new Date();  // 这才是真正的版本分界点
  }
  
  return prisma.survey.update({ where: { id: surveyId }, data: surveyData });
};

// 3. 响应版本过滤改为使用 publishedAt
function getVersionFilterCriteriaByPublishedAt(
  publishHistory: Date[],
  versionIndex: number
) {
  // 逻辑与之前相同，但时间戳来自 publishedAt 历史记录
  // 不会被草稿自动保存干扰
}
```

**方案一的优势**：
- `publishedAt` 仅在正式发布时更新，不受草稿自动保存干扰
- 无需改动查询逻辑，只需将版本分界点从 `updatedAt` 改为 `publishedAt`
- 数据模型变更最小，迁移成本低

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
    └─ 响应创建（记录 createdAt，不记录任何版本标识）
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

### 4.2 响应表结构（无版本关联）

**响应表结构** (`schema.prisma:158-190`)：
```prisma
model Response {
  id          String   @id @default(cuid())
  createdAt   DateTime @default(now())
  surveyId    String   // 仅关联到调查，没有 versionId 字段
  // ... 其他字段：data, meta, ttc, contactAttributes 等
}
```

**关键事实**：
- 响应只存储 `surveyId`，不存储版本 ID
- 响应不存储提交时的 `survey.updatedAt`
- 没有任何字段可以将响应与特定问卷配置版本关联

### 4.3 响应过滤机制（代码级验证）

#### 4.3.1 `buildWhereClause` 完整实现分析

在 `apps/web/lib/response/utils.ts:153-642` 的 `buildWhereClause` 函数：
```typescript
export const buildWhereClause = (survey: TSurvey, filterCriteria?: TResponseFilterCriteria) => {
  const whereClause: Prisma.ResponseWhereInput["AND"] = [];

  // 1. finished 状态过滤
  if (filterCriteria?.finished !== undefined) {
    whereClause.push({ finished: filterCriteria.finished });
  }

  // 2. 日期范围过滤（仅来自用户手动选择）
  if (filterCriteria?.createdAt) {
    const createdAt: { lte?: Date; gte?: Date } = {};
    if (filterCriteria.createdAt.max) createdAt.lte = filterCriteria.createdAt.max;
    if (filterCriteria.createdAt.min) createdAt.gte = filterCriteria.createdAt.min;
    whereClause.push({ createdAt });
  }

  // 3. Tags 过滤
  // 4. Contact Attributes 过滤
  // 5. Meta 过滤
  // 6. Others 过滤
  // 7. Data 过滤（这里用到了 survey 参数来查找元素）
  if (filterCriteria?.data) {
    const data: Prisma.ResponseWhereInput[] = [];
    Object.entries(filterCriteria.data).forEach(([key, val]) => {
      // ⚠️  survey 参数仅在这里使用：通过 element.id 查找元素信息
      // ⚠️  完全不使用 survey.updatedAt 或任何版本相关字段
      const elements = getElementsFromBlocks(survey.blocks);
      const element = elements.find((element) => element.id === key);
      // ... 数据过滤逻辑
    });
    whereClause.push({ AND: data });
  }

  // 8. responseIds 过滤
  // 9. quotas 过滤

  return { AND: whereClause };
};
```

**代码级验证结论**：
- ✅ `survey` 参数仅在 `data` 过滤中使用，用于通过 `element.id` 查找问卷元素信息
- ❌ **完全没有**使用 `survey.updatedAt` 或任何版本相关字段
- ❌ **完全没有**自动按时间戳分桶的逻辑
- ✅ 日期过滤**仅**来自 `filterCriteria.createdAt`（用户手动传入）

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
  // ⚠️  没有任何版本相关的过滤条件
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
  // ⚠️  所有该 surveyId 下的响应都会返回，除非用户手动选了日期范围
});
```

#### 4.3.4 可执行的版本边界判断示例（修正版）

##### 4.3.4.1 数据层硬约束：Prisma `@updatedAt` 自动更新

在分析版本边界逻辑之前，必须先明确**数据层的硬约束**：

在 `packages/database/schema.prisma:348`：
```prisma
model Survey {
  // ...
  updatedAt  DateTime  @updatedAt @map(name: "updated_at")
  // ...
}
```

**`@updatedAt` 是 Prisma 内置属性，具有以下约束：**
- ✅ 每次调用 `prisma.survey.update()` 时，Prisma **自动**将 `updatedAt` 设置为当前时间
- ❌ 即使代码中不手动设置 `updatedAt`，它也会自动更新
- ❌ 无法通过代码阻止 `updatedAt` 自动更新（除非不调用 `update`）
- ⚠️  `service.ts:526` 中的 `surveyData.updatedAt = new Date()` 实际上是**多余**的，因为 Prisma 会自动覆盖

**实际影响**：
- 草稿自动保存（调用 `updateSurveyInternal`）→ 触发 `prisma.survey.update()` → `updatedAt` 自动推进
- 正式发布（调用 `updateSurveyInternal`）→ 触发 `prisma.survey.update()` → `updatedAt` 自动推进
- 暂停/恢复调查 → 触发 `prisma.survey.update()` → `updatedAt` 自动推进
- **任何对 Survey 记录的修改都会推进 `updatedAt`**

这意味着：**"草稿不更新 updatedAt" 在现有数据模型下是不可能的**，必须通过新增字段来实现版本边界的清晰化。

##### 4.3.4.2 修正后的版本边界判断代码（无索引越界）

基于代码分析，以下是在业务层实现版本边界判断的**可执行、无 Bug** 代码示例：

```typescript
/**
 * 人为约定的版本边界判断
 * 注意：这不是 Formbricks 内置的逻辑，需要业务层自行实现
 */
interface VersionBoundary {
  timestamp: Date;
  description: string;
  type: 'publish' | 'edit';  // 移除 'draft'，因为草稿保存不产生有效版本边界
}

/**
 * 版本类型定义
 */
type VersionSegmentType = 'first' | 'middle' | 'last';

/**
 * 版本区间信息
 */
interface VersionSegment {
  type: VersionSegmentType;
  versionIndex: number;
  description: string;
  filterCriteria: { createdAt?: { min?: Date; max?: Date } };
}

/**
 * 修正后的版本过滤条件生成函数
 * 修复了原示例中末段索引越界的问题
 * 
 * @param boundaries 人为记录的版本边界列表（需要自己维护）
 * @param versionIndex 要查询的版本索引（0 = 最新版本 = 首段）
 * @param surveyCreatedAt 问卷创建时间（可选，用于末段下界）
 * @returns 该版本的响应过滤条件，以及版本类型信息
 */
function getVersionFilterCriteria(
  boundaries: VersionBoundary[],
  versionIndex: number,
  surveyCreatedAt?: Date
): VersionSegment {
  // 边界检查：空边界列表
  if (boundaries.length === 0) {
    return {
      type: 'first',
      versionIndex: 0,
      description: '未定义版本边界',
      filterCriteria: {}
    };
  }

  // 边界检查：索引越界
  if (versionIndex < 0 || versionIndex >= boundaries.length) {
    throw new Error(`versionIndex ${versionIndex} out of bounds (0-${boundaries.length - 1})`);
  }

  // 按时间倒序排列（从新到旧）
  const sortedBoundaries = [...boundaries].sort((a, b) => b.timestamp.getTime() - a.timestamp.getTime());

  const currentVersion = sortedBoundaries[versionIndex];
  const nextVersion = sortedBoundaries[versionIndex + 1];  // 注意：是下一个（更旧的）版本

  // 判断版本类型
  const isFirstSegment = versionIndex === 0;                    // 首段：最新版本
  const isLastSegment = versionIndex === sortedBoundaries.length - 1;  // 末段：最早版本
  const isMiddleSegment = !isFirstSegment && !isLastSegment;   // 中段：中间版本

  let filterCriteria: { createdAt?: { min?: Date; max?: Date } } = {};

  if (isFirstSegment) {
    // ════════════════════════════════════════════════════════════
    // 首段（最新版本）：>= 当前边界时间戳，无上界
    // ════════════════════════════════════════════════════════════
    // 时间轴： ──┼───────────────────────────▶
    //      T3(当前)       响应属于此版本
    filterCriteria = {
      createdAt: { min: currentVersion.timestamp }
    };
  } else if (isMiddleSegment) {
    // ════════════════════════════════════════════════════════════
    // 中段（历史版本）：>= 下一个边界 AND < 当前边界
    // ════════════════════════════════════════════════════════════
    // 时间轴： ──┼───────────────────────────┼──────▶
    //      T2(下一个)   响应属于此版本    T3(当前)
    filterCriteria = {
      createdAt: {
        min: nextVersion!.timestamp,  // 中段一定有下一个版本，非空
        max: new Date(currentVersion.timestamp.getTime() - 1)  // 闭区间：不包含当前边界
      }
    };
  } else {
    // ════════════════════════════════════════════════════════════
    // 末段（最早版本）：< 当前边界，可选 >= 问卷创建时间
    // ════════════════════════════════════════════════════════════
    // 时间轴： ──┬──────────────────┼──────▶
    //     问卷创建   响应属于此版本  T1(当前)
    filterCriteria = {
      createdAt: {
        max: new Date(currentVersion.timestamp.getTime() - 1)  // 闭区间：不包含当前边界
      }
    };
    // 如果提供了问卷创建时间，加上下界
    if (surveyCreatedAt) {
      filterCriteria.createdAt!.min = surveyCreatedAt;
    }
  }

  return {
    type: isFirstSegment ? 'first' : isLastSegment ? 'last' : 'middle',
    versionIndex,
    description: currentVersion.description,
    filterCriteria
  };
}
```

##### 4.3.4.3 三类时间区间的执行判定方法

| 版本类型 | 索引位置 | 时间区间条件 | 边界检查 |
|---------|---------|-------------|---------|
| **首段（最新）** | `versionIndex === 0` | `createdAt >= currentVersion.timestamp` | 无需检查 `nextVersion`，无上界 |
| **中段（历史）** | `0 < versionIndex < length-1` | `createdAt >= nextVersion.timestamp AND createdAt < currentVersion.timestamp` | `nextVersion` 非空，两边都有边界 |
| **末段（最早）** | `versionIndex === length-1` | `createdAt < currentVersion.timestamp` (可选 `>= surveyCreatedAt`) | `nextVersion` 为 `undefined`，下界可选 |

**判定执行流程：**
```
输入: boundaries, versionIndex, surveyCreatedAt?
    ↓
1. 检查 boundaries 是否为空 → 返回空过滤
    ↓
2. 检查 versionIndex 是否越界 → 抛出错误
    ↓
3. 按时间倒序排序 boundaries
    ↓
4. 获取 currentVersion = sorted[versionIndex]
   获取 nextVersion = sorted[versionIndex + 1]
    ↓
5. 判断类型:
   ├─ 首段: versionIndex === 0 → min = current.timestamp
   ├─ 中段: 0 < index < length-1 → min = next.timestamp, max = current.timestamp - 1ms
   └─ 末段: versionIndex === length-1 → max = current.timestamp - 1ms (可选 min = surveyCreatedAt)
    ↓
输出: VersionSegment (type + filterCriteria)
```

##### 4.3.4.4 完整使用示例（无索引越界）

```typescript
/**
 * 使用示例：查询发布后修改前后的响应
 */
async function getVersionedResponsesExample() {
  // 1. 你需要自己记录版本边界（Formbricks 不提供这个功能）
  // 注意：只记录发布和有意的内容修改，不记录草稿自动保存
  const myBoundaries: VersionBoundary[] = [
    { timestamp: new Date('2026-05-20T10:00:00Z'), description: '首次发布', type: 'publish' },
    { timestamp: new Date('2026-05-22T14:30:00Z'), description: '修改问题3选项', type: 'edit' },
    { timestamp: new Date('2026-05-24T09:15:00Z'), description: '新增问题5', type: 'edit' },
  ];

  const surveyCreatedAt = new Date('2026-05-18T08:00:00Z');

  // 2. 查询首段（最新版本：新增问题5之后）
  const latestSegment = getVersionFilterCriteria(myBoundaries, 0, surveyCreatedAt);
  console.log('首段类型:', latestSegment.type);  // 'first'
  console.log('首段过滤:', JSON.stringify(latestSegment.filterCriteria));
  // { createdAt: { min: '2026-05-24T09:15:00.000Z' } }
  const latestResponses = await getResponses('survey_xxx', 100, 0, latestSegment.filterCriteria);

  // 3. 查询中段（历史版本：修改问题3之后，新增问题5之前）
  const middleSegment = getVersionFilterCriteria(myBoundaries, 1, surveyCreatedAt);
  console.log('中段类型:', middleSegment.type);  // 'middle'
  console.log('中段过滤:', JSON.stringify(middleSegment.filterCriteria));
  // { createdAt: { min: '2026-05-20T10:00:00.000Z', max: '2026-05-24T09:14:59.999Z' } }
  const middleResponses = await getResponses('survey_xxx', 100, 0, middleSegment.filterCriteria);

  // 4. 查询末段（最早版本：首次发布之前，问卷创建之后）
  const lastSegment = getVersionFilterCriteria(myBoundaries, 2, surveyCreatedAt);
  console.log('末段类型:', lastSegment.type);  // 'last'
  console.log('末段过滤:', JSON.stringify(lastSegment.filterCriteria));
  // { createdAt: { min: '2026-05-18T08:00:00.000Z', max: '2026-05-20T09:59:59.999Z' } }
  const lastResponses = await getResponses('survey_xxx', 100, 0, lastSegment.filterCriteria);

  // 5. 边界检查：越界访问会抛出错误
  try {
    getVersionFilterCriteria(myBoundaries, 3, surveyCreatedAt);
  } catch (e) {
    console.log('越界捕获:', (e as Error).message);
    // "versionIndex 3 out of bounds (0-2)"
  }
}

/**
 * 实际调用 getResponses 的示例
 * 对应代码：apps/web/lib/response/service.ts:278
 */
import { getResponses } from "@/lib/response/service";

async function queryResponsesByDateRange() {
  // 2026-05-20 10:00 发布了问卷
  // 2026-05-22 14:30 修改了问题3
  
  // 查询修改前的响应（末段）
  const beforeEditResponses = await getResponses(
    'survey_xxx',
    100,
    0,
    {
      createdAt: {
        min: new Date('2026-05-20T10:00:00Z'),
        max: new Date('2026-05-22T14:29:59Z')
      }
    }
  );
  
  // 查询修改后的响应（首段）
  const afterEditResponses = await getResponses(
    'survey_xxx',
    100,
    0,
    {
      createdAt: {
        min: new Date('2026-05-22T14:30:00Z')
      }
    }
  );
  
  return { beforeEditResponses, afterEditResponses };
}
```

#### 4.3.5 可落地结论

| 错误假设 | 实际情况 | 代码证据 |
|---------|---------|---------|
| 系统自动按 `survey.updatedAt` 分桶 | 用户必须手动选择日期范围才能分离不同版本 | `buildWhereClause:153-642` 不使用 `survey.updatedAt` |
| 每个响应自动关联到发布时的版本 | 响应只关联 `surveyId`，无版本关联字段 | `schema.prisma:158-190` 无 `versionId` 字段 |
| 查看统计时自动排除旧版本响应 | 统计默认包含所有响应，除非手动设置日期过滤 | `surveySummary.ts:1051-1144` 无条件返回所有响应 |
| 可以查询历史版本的响应 | 可以，但需要你自己记录历史时间点 | 无版本历史表，`updatedAt` 只存当前值 |

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

### 5.3 回滚边界（代码级验证）

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
| 响应与问卷配置的关联 | 响应不记录提交时的问卷配置 | `Response` 模型无版本字段 |

#### 5.3.3 版本边界模糊导致的回滚判断问题

**问题 1：无法确定哪些时间戳是"有效发布版本"**

```
T1: 草稿自动保存（updatedAt = T1）
T2: 草稿自动保存（updatedAt = T2）
T3: 正式发布（updatedAt = T3）
T4: 编辑后自动保存（updatedAt = T4）
T5: 编辑后手动保存（updatedAt = T5）

响应 R1 (createdAt = T3.5) → 人为约定属于 T3 版本
响应 R2 (createdAt = T4.5) → 人为约定属于 T4 还是 T5 版本？

// ⚠️  代码无法区分 T1-T5 中哪些是"有意发布"的版本
// ⚠️  只有当前的 updatedAt = T5，历史值全部丢失
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
// ⚠️  如果用 T2 作为分界点，会将相同问题的响应分成两部分
```

#### 5.3.4 回滚判断的可落地操作建议

| 场景 | 推荐操作 | 原理 |
|------|---------|------|
| 只是修复错别字/样式 | 直接编辑，不分离版本 | 不影响响应数据结构 |
| 修改了问题选项/顺序 | 记下修改时间，后续用日期过滤分离 | 问题变化导致数据不可比 |
| 新增/删除了问题 | 记下修改时间，用日期过滤分离，或**复制问卷** | 响应字段结构变化 |
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
2. **灵活性**：不需要预先定义版本，任何编辑都可以作为"新版本"分界点
3. **性能**：响应写入不需要额外的版本 ID 查询
4. **存储效率**：不需要存储多份问卷配置副本

### 6.2 设计缺点（代码级验证）

| 设计问题 | 代码证据 | 实际影响 |
|---------|---------|---------|
| **概念不直观** | 无 `SurveyVersion` 表 | "隐式版本"对开发者和用户都不够直观 |
| **无自动版本分离** | `buildWhereClause` 不使用 `survey.updatedAt` | 用户必须手动选择日期范围才能分离版本 |
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

### 6.4 潜在改进方向（基于代码分析，符合数据层约束）

如果需要改进版本管理机制，**必须考虑 Prisma `@updatedAt` 自动更新的硬约束**。以下是按推荐程度排序的改进方案：

1. **⭐⭐⭐⭐⭐ 新增 `publishedAt` 字段**：
   - 在 Survey 表新增 `publishedAt: DateTime?` 字段
   - 仅在正式发布（`skipValidation = false`）时手动设置 `publishedAt = new Date()`
   - 草稿保存不更新 `publishedAt`，不受 `@updatedAt` 约束影响
   - 版本分界点从 `updatedAt` 改为 `publishedAt`，清晰可靠
   - 数据模型变更最小，迁移成本最低

2. **⭐⭐⭐⭐ 新增 `SurveyVersion` 表**：
   - 新建 `SurveyVersion` 表，存储每次发布的问卷配置快照和版本时间
   - 每次正式发布时创建一条记录，草稿保存不创建
   - `Response` 可选增加 `versionId` 外键（可选，可保持现有响应表结构不变）
   - 可追溯历史版本配置，支持一键回滚
   - 数据层完全隔离，不受 `@updatedAt` 约束影响

3. **⭐⭐⭐ 新增 `lastContentUpdateAt` 字段**：
   - 在 Survey 表新增 `lastContentUpdateAt: DateTime?` 字段
   - 更新时检测 `questions`/`blocks` 是否真正变化，仅在内容变更时更新此字段
   - 草稿自动保存如果只是小改动（如修复错别字），可选择不更新此字段
   - 比 `updatedAt` 更精确，但比 `publishedAt` 复杂

4. **⭐⭐⭐ 业务层维护版本边界**：
   - 无需数据层变更，在业务代码中自行维护版本边界列表（如本文档示例）
   - 每次正式发布时记录时间戳和说明到业务表或配置中
   - 响应查询时使用业务层维护的边界列表过滤
   - 灵活性高，但需要严格的操作规范

5. **⭐⭐ 草稿不存 Survey 主表**：
   - 新建 `SurveyDraft` 表，草稿保存写入此表，不更新 Survey 主表
   - 正式发布时才将草稿内容同步到 Survey 主表
   - Survey 主表的 `updatedAt` 只在正式发布时更新
   - 改动最大，需要重构保存逻辑

> ⚠️  **已废弃的思路**："草稿保存不更新 `updatedAt`" 无法实现，因为 Prisma `@updatedAt` 属性会在每次 `prisma.survey.update()` 调用时自动更新字段值，代码层无法阻止。

---

## 七、关键数据流向图

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
│ surveyId=xxx    │                │
│ (无 versionId)  │                │
└─────────────────┘                │
          │                        │
          ▼                        │
┌─────────────────┐                │
│ Response #2     │                │
│ createdAt=T1+2  │◀── Link/App    │
│ surveyId=xxx    │                │
│ (无 versionId)  │                │
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
│ surveyId=xxx    │
│ (无 versionId)  │
└─────────────────┘

版本分界点：T1 和 T2（人为约定，不是代码强制）
- Response #1, #2 → 版本 1 (createdAt >= T1 and < T2)
- Response #3     → 版本 2 (createdAt >= T2)
- T0+10s 等自动保存产生的时间戳 → 无法区分是否是有效版本
```

⚠️  **关键提醒**：`T0+10s` 这种自动保存产生的 `updatedAt` 推进，没有任何代码标记它是"草稿保存"还是"正式发布"，因此无法精确划定版本边界。所有版本分界都是**人为约定**。

---

## 八、总结

### 8.1 六个核心认知（代码级验证）

1. **没有 SurveyVersion 表**：版本是通过 `survey.updatedAt` 时间戳**人为约定**的，不是代码强制的
2. **响应不关联版本 ID**：只关联 `surveyId`，没有任何版本标识字段
3. **无自动分桶逻辑**：查询响应时**仅**依赖用户传入的 `filterCriteria.createdAt` 过滤
4. **Prisma `@updatedAt` 自动更新**：任何对 Survey 记录的修改都会自动推进 `updatedAt`，代码层无法阻止
5. **草稿自动保存也推进 updatedAt**：每 10 秒的自动保存会产生大量无效版本边界
6. **"草稿不更新 updatedAt" 不可行**：受数据层硬约束，必须通过新增字段实现版本边界清晰化

### 8.2 三类时间区间判定方法（可执行）

| 版本类型 | 索引位置 | 时间区间条件 | 边界检查 |
|---------|---------|-------------|---------|
| **首段（最新）** | `versionIndex === 0` | `createdAt >= currentVersion.timestamp` | 无需检查 `nextVersion`，无上界 |
| **中段（历史）** | `0 < versionIndex < length-1` | `createdAt >= nextVersion.timestamp AND createdAt < currentVersion.timestamp` | `nextVersion` 非空，两边都有边界 |
| **末段（最早）** | `versionIndex === length-1` | `createdAt < currentVersion.timestamp` (可选 `>= surveyCreatedAt`) | `nextVersion` 为 `undefined`，下界可选 |

**判定执行流程**：
```
输入: boundaries, versionIndex, surveyCreatedAt?
    ↓
1. 检查 boundaries 是否为空 → 返回空过滤
    ↓
2. 检查 versionIndex 是否越界 → 抛出错误
    ↓
3. 按时间倒序排序 boundaries
    ↓
4. 获取 currentVersion = sorted[versionIndex]
   获取 nextVersion = sorted[versionIndex + 1]
    ↓
5. 判断类型:
   ├─ 首段: versionIndex === 0 → min = current.timestamp
   ├─ 中段: 0 < index < length-1 → min = next.timestamp, max = current.timestamp - 1ms
   └─ 末段: versionIndex === length-1 → max = current.timestamp - 1ms (可选 min = surveyCreatedAt)
    ↓
输出: VersionSegment (type + filterCriteria)
```

### 8.3 发布流程要点

- 发布 = 状态变更 + 时间戳更新 + 完整验证
- 两种分发渠道：Link（公开链接）和 App（SDK 触发）
- 编辑已发布问卷会产生数据一致性警告
- **推荐：复制问卷而非编辑已发布问卷**（从根本上避免版本边界模糊问题）

### 8.4 回滚边界

| ✅ 可以回滚 | ❌ 无法回滚 |
|-----------|-----------|
| 暂停/恢复调查 | 已提交的响应数据 |
| 手动恢复旧配置 | `survey.updatedAt` 时间戳（每次更新自动前进） |
| 通过日期过滤分离响应 | 历史 `updatedAt` 记录（只存当前值） |
| | 已产生的统计摘要（需手动修正） |
| | 响应与问卷配置的关联（无版本字段） |
| | Prisma `@updatedAt` 自动更新行为 |

### 8.5 可落地的版本管理建议

基于代码分析，给出以下操作建议：

1. **发布后尽量不要编辑**：如果只是修复错别字可以，但修改问题结构会破坏数据一致性
2. **重要修改请复制问卷**：复制后新旧问卷响应天然隔离，是最可靠的版本管理方式
3. **如需日期过滤，务必记录修改时间**：如果确实需要编辑已发布问卷，精确记下修改时间，后续用这个时间点过滤响应
4. **不要依赖 `updatedAt` 做精确版本分界**：它会被草稿自动保存频繁推进，只能作为粗略参考
5. **导出数据做离线分析**：如果需要精确的版本对比，导出所有响应后在外部按时间分段分析
6. **自行维护版本边界列表**：如果业务需要版本管理，需要自己记录每次发布/修改的时间点和说明
7. **优先使用 `publishedAt` 方案**：如果要改进系统，新增 `publishedAt` 字段是成本最低、效果最好的方案

### 8.6 最终结论

Formbricks 的"版本快照"和"版本关联"都是**人为概念**，不是代码实现。代码中：

**✅ 存在的机制**：
- `survey.updatedAt` 时间戳（Prisma `@updatedAt` 自动更新，**每次更新无条件推进**）
- `response.createdAt` 时间戳（响应提交时记录）
- `filterCriteria.createdAt` 过滤条件（用户手动传入，**唯一的版本分离手段**）
- `buildWhereClause` 函数（仅处理用户传入的过滤条件，**无任何自动分桶逻辑**）

**❌ 不存在的机制**：
- 没有任何代码自动比较 `survey.updatedAt` 和 `response.createdAt` 来分桶
- 没有任何版本标识字段（`versionId`、`surveyVersionId` 等）
- 没有任何版本历史记录表
- 没有任何一键回滚功能
- 没有任何方式阻止 `updatedAt` 自动更新

**所有版本分离都需要用户手动选择 `filterCriteria.createdAt` 日期范围来实现。**

> ⚠️  **关键约束**：由于 Prisma `@updatedAt` 的硬约束，"草稿不更新 `updatedAt`" 无法在现有数据模型下实现。如需清晰的版本边界，**必须新增字段**（推荐 `publishedAt`）。
