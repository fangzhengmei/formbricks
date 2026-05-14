# Analytics Query Analysis Report

## 1. 概述

本文档分析了 Formbricks 中 A/B 测试分组机制与漏斗分析查询路径，重点关注：
- 真实分组条件的来源与筛选方式
- 漏斗统计与分组组合查询时的指标输入、条件拼接和结果字段对应关系
- 正常路径与缺省路径处理

## 2. 核心架构

### 2.1 主要文件与职责

| 文件 | 职责 |
|------|------|
| `packages/js-core/src/lib/common/utils.ts` | 客户端流量分配：`shouldDisplayBasedOnPercentage` |
| `packages/js-core/src/lib/survey/widget.ts` | 问卷展示触发：`displayPercentage` 检查 |
| `packages/types/segment.ts` | 细分群体（Segment）过滤器类型定义 |
| `apps/web/lib/response/utils.ts` | 查询条件构建：`buildWhereClause` |
| `apps/web/lib/display/service.ts` | 展示统计服务：`getDisplayCountBySurveyId` |
| `apps/web/app/(app)/environments/[environmentId]/surveys/[surveyId]/(analysis)/summary/lib/surveySummary.ts` | 核心分析查询逻辑汇总 |
| `apps/web/app/(app)/environments/[environmentId]/surveys/[surveyId]/(analysis)/actions.ts` | Server Actions 入口 |

## 3. A/B 分组机制详解

### 3.1 分组条件来源

Formbricks 支持 **三层分组机制**，从流量分配到用户细分逐层过滤：

```
┌─────────────────────────────────────────────────────────┐
│  1. 流量分配层 (Client-side)                            │
│     displayPercentage: 0.01 ~ 100                        │
│     基于加密随机数决定哪些用户进入实验                   │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│  2. 细分群体层 (Server-side)                            │
│     Segment Filters: attribute/device/person/segment     │
│     服务端计算用户是否属于目标群体                       │
└─────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────┐
│  3. 联系人属性层 (Response-level)                       │
│     contactAttributes: 自定义 key-value                 │
│     响应级别细粒度分组（实验变体、用户分层等）           │
└─────────────────────────────────────────────────────────┘
```

### 3.2 分组机制一：流量百分比分配 (displayPercentage)

**实现位置**：`packages/js-core/src/lib/common/utils.ts:208-211`

```typescript
export const shouldDisplayBasedOnPercentage = (displayPercentage: number): boolean => {
  const randomNum = Math.floor(getSecureRandom() * 10000) / 100;
  return randomNum <= displayPercentage;
};
```

**关键特性**：
- **客户端执行**：在 SDK 触发时决定，不存入数据库
- **加密随机**：使用 `crypto.getRandomValues()` 生成安全随机数
- **精度**：两位小数百分比（0.01% 粒度）
- **无状态**：每次触发重新计算，用户可能在不同会话中分组不同

**触发流程**：
```
widget.triggerSurvey()
    ↓
if (survey.displayPercentage) {
  const shouldDisplay = shouldDisplayBasedOnPercentage(displayPercentage);
  if (!shouldDisplay) return;  // 跳过此用户
}
    ↓
继续展示问卷流程
```

### 3.3 分组机制二：Segment 细分群体过滤器

**类型定义**：`packages/types/segment.ts`

支持四种过滤器类型，可通过 `AND/OR` 连接器组合嵌套：

| 过滤器类型 | Root 对象 | 操作符 | 说明 |
|-----------|-----------|--------|------|
| **Attribute** | `{ contactAttributeKey }` | `equals/notEquals/lessThan/greaterThan/contains/startsWith/endsWith/isSet/isNotSet` + 日期操作符 | 按联系人属性过滤 |
| **Person** | `{ personIdentifier }` | `equals/notEquals/contains/startsWith/endsWith/isSet/isNotSet` | 按个人标识过滤 |
| **Segment** | `{ segmentId }` | `userIsIn/userIsNotIn` | 嵌套 Segment 引用 |
| **Device** | `{ deviceType }` | `equals/notEquals` | 按设备类型过滤 |

**Segment 过滤器示例结构**：
```typescript
{
  id: "filter_001",
  connector: "and",  // 组间连接：and/or/null(首组)
  resource: {
    id: "seg_001",
    root: { type: "attribute", contactAttributeKey: "plan" },
    qualifier: { operator: "equals" },
    value: "premium"
  }
}
```

### 3.4 分组机制三：响应级别联系人属性 (contactAttributes)

**存储结构**：
```typescript
// Response 表中 JSONB 字段
contactAttributes: {
  "ab_test_variant": "control",     // A/B 实验分组
  "user_segment": "high_value",     // 用户分层
  "signup_date": "2024-01-15",      // 注册日期
  "custom_key": "custom_value"      // 任意自定义字段
}
```

**过滤查询构建**：`apps/web/lib/response/utils.ts:186-214`

```typescript
// 输入 filterCriteria.contactAttributes
{
  ab_test_variant: { op: "equals", value: "variant_A" }
}

// 输出 Prisma where 条件
{
  AND: [
    { contactAttributes: { path: ["ab_test_variant"], equals: "variant_A" } }
  ]
}
```

### 3.5 三种分组机制的区别与组合

| 维度 | displayPercentage | Segment Filters | contactAttributes |
|------|-------------------|-----------------|--------------------|
| **执行时机** | 客户端触发时 | 服务端计算 | 响应提交时写入 |
| **持久化** | 不存储 | 存储 Segment 定义 | 存储在 Response 中 |
| **粒度** | Survey 级别 | Environment 级别 | Response 级别 |
| **查询时可用** | ❌ 不可用（无状态） | ⚠️ 仅 Segment ID | ✅ 可用 |
| **A/B 测试场景** | 流量切分 | 目标人群筛选 | 实验组标记 |

**典型 A/B 测试组合流程**：
```
1. 设置 displayPercentage = 50 (流量切分 50%)
2. 设置 Segment 筛选目标人群 (e.g. plan = premium)
3. 客户端触发时：
   a. 通过 Segment 检查用户资格
   b. 通过 displayPercentage 随机抽样
4. 响应提交时写入 contactAttributes.ab_test_variant
5. 分析查询时按 contactAttributes 分组对比
```

## 4. 漏斗分析查询机制

### 4.1 核心输入：TResponseFilterCriteria

**完整结构**：`packages/types/responses.ts:202-293`

```typescript
{
  // 基础过滤
  finished?: boolean;                    // 是否已完成
  responseIds?: string[];                // 指定响应 ID 列表
  createdAt?: { min?: Date; max?: Date }; // 创建时间范围
  
  // 分组维度：联系人属性
  contactAttributes?: Record<string, {
    op: "equals" | "notEquals";
    value: string | number;
  }>;
  
  // 分组维度：回答数据
  data?: Record<string, {
    op: "submitted" | "filledOut" | "skipped" |
        "equals" | "notEquals" | "lessThan" | "lessEqual" |
        "greaterThan" | "greaterEqual" | "includesAll" | "includesOne" |
        "uploaded" | "notUploaded" | "accepted" | "clicked" |
        "booked" | "isEmpty" | "isNotEmpty" | "isAnyOf";
    value?: any;
  }>;
  
  // 分组维度：标签
  tags?: { applied?: string[]; notApplied?: string[] };
  
  // 分组维度：其他字段（如语言）
  others?: Record<string, {
    op: "equals" | "notEquals";
    value: string | number;
  }>;
  
  // 分组维度：元数据
  meta?: Record<string, {
    op: "equals" | "notEquals" | "contains" | "doesNotContain" |
        "startsWith" | "doesNotStartWith" | "endsWith" | "doesNotEndWith";
    value?: string;
  }>;
  
  // 配额过滤
  quotas?: Record<string, {
    op: "screenedIn" | "screenedOut" | "screenedOutNotInQuota";
  }>;
}
```

### 4.2 查询组合逻辑：buildWhereClause

**实现位置**：`apps/web/lib/response/utils.ts:153-642`

**条件拼接规则**：所有过滤维度通过 `AND` 连接，维度内部根据操作符构建子条件。

```typescript
return {
  AND: [
    { finished: filterCriteria.finished },          // 完成状态
    { createdAt: { gte, lte } },                    // 时间范围
    { AND: tagFilters },                             // 标签过滤
    { AND: contactAttributesFilters },               // 联系人属性
    { AND: metaFilters },                            // 元数据
    { AND: othersFilters },                          // 其他字段
    { AND: dataFilters },                            // 回答数据
    { id: { in: responseIds } },                     // 指定响应 ID
    { AND: quotaFilters },                           // 配额筛选
  ]
};
```

**关键过滤操作实现**：

| 操作符 | 实现逻辑 | 适用场景 |
|--------|----------|----------|
| **submitted** | `data.path[key] IS NOT NULL` | 检查问题是否被回答 |
| **skipped** | `(IS NULL) OR (= "") OR (= [])` | 检查问题是否被跳过 |
| **includesOne** | 多选任一匹配 → `OR(array_contains, equals)` | 多选题任一选项匹配 |
| **includesAll** | 多选全部匹配 → `array_contains` | 多选题全部选项匹配 |
| **isAnyOf** | 多个值任一匹配 | 支持 "Other" 选项场景 |

**"Other" 选项特殊处理**：当过滤包含 "Other" 时，排除所有预定义选项组合：
```typescript
if (values.includes(otherChoice.label.default)) {
  // 生成所有子集排列组合
  const subsets = generateAllPermutationsOfSubsets(predefinedLabels);
  // 排除这些子集
  NOT: { OR: subsetConditions }
}
```

### 4.3 主查询流程：getSurveySummary

**实现位置**：`apps/web/app/(app)/environments/[environmentId]/surveys/[surveyId]/(analysis)/summary/lib/surveySummary.ts`

```
输入: surveyId + filterCriteria
    ↓
1. 获取 Survey 定义 (问题、欢迎卡配置等)
    ↓
2. 分批获取响应数据 (游标分页)
   ├─ getResponsesForSummary()
   │   └─ buildWhereClause() 构建查询
   └─ 批大小: 5000 条
    ↓
3. 并行获取关联数据
   ├─ getDisplayCountBySurveyId() → 总展示数
   │   └─ 支持 createdAt + responseIds 过滤
   └─ getQuotasSummary() → 配额统计
    ↓
4. 计算漏斗分析数据 → getSurveySummaryDropOff()
    ↓
5. 计算汇总指标 → getSurveySummaryMeta()
    ↓
6. 计算各问题详情 → getElementSummary()
    ↓
输出: TSurveySummary
```

### 4.4 Display 计数的特殊过滤逻辑

当存在 `filterCriteria` 且包含非时间维度过滤时：

```typescript
// 有过滤条件时：先获取符合条件的 responseIds，再关联 display
if (hasFilter) {
  // Step 1: 获取符合条件的响应列表
  const responses = await getResponsesForSummary(surveyId, filterCriteria);
  const responseIds = responses.map(r => r.id);
  
  // Step 2: 使用 responseIds 过滤 display
  const displayCount = await getDisplayCountBySurveyId(surveyId, {
    createdAt: filterCriteria.createdAt,
    responseIds,  // 通过 response 关联计数
  });
} else {
  // 无过滤条件时：直接按时间范围计数
  const displayCount = await getDisplayCountBySurveyId(surveyId, {
    createdAt: filterCriteria?.createdAt
  });
}
```

## 5. 漏斗分析核心算法

### 5.1 可见性判断机制

**核心函数**：`wasElementSeen(response, elementId)` — `surveySummary.ts:130-132`

```typescript
// 判断响应是否真正"看到"了某个问题
wasElementSeen(response, elementId) {
  // 条件 A: 有停留时间 (ttc > 0)
  // 条件 B: 有回答数据 (data[elementId] !== undefined)
  return (response.ttc != null && response.ttc[elementId] > 0) 
         || 
         response.data[elementId] !== undefined;
}
```

**设计决策**：
- ✅ 避免重放问卷逻辑（分支跳转复杂且易错）
- ✅ 基于实际行为数据（更可靠）
- ✅ 支持部分响应场景

### 5.2 流失归因算法

**实现位置**：`surveySummary.ts:134-223`

```typescript
for each response:
  // 遍历所有问题，追踪最后一个被看到的问题
  lastSeenIdx = -1
  for i = 0 to elements.length:
    if wasElementSeen(response, elements[i].id):
      impressions[i]++  // 该问题展示数+1
      lastSeenIdx = i   // 更新最后看到的索引
  
  // 未完成的响应 → 流失归因到最后一个看到的问题
  if (!response.finished && lastSeenIdx >= 0):
    dropOff[lastSeenIdx]++  // 该问题流失数+1
```

### 5.3 首屏特殊处理（无欢迎卡场景）

当 `survey.welcomeCard.enabled = false` 时：

```typescript
// 第一个问题的展示数 = 总展示数
impressions[0] = displayCount;

// 流失数 = 总展示数 - 实际响应数 (impressions from loop)
dropOff[0] = displayCount - actualImpressionsFromLoop;
dropOffPercentage[0] = (dropOff[0] / displayCount) * 100;
```

### 5.4 问题级耗时统计

```typescript
// 基于 block 级 ttc，映射到每个问题
for each block:
  maxElementTtc = max(block.elements.ttc)
  blockTimes[block.id] = maxElementTtc

// 问题级 ttc = 所属 block 的耗时
for each element:
  totalTtc[element.id] += blockTimes[element.blockId]
  avgTtc = totalTtc[element.id] / responseCount
```

## 6. 结果输出结构与字段对应

### 6.1 TSurveySummary 输出结构

```typescript
{
  // ───────────────────────────────────────────────
  // 1. 总体汇总指标
  // ───────────────────────────────────────────────
  meta: {
    displayCount: number;           // 总展示数
    totalResponses: number;         // 总响应数
    startsPercentage: number;       // 开始率 = (响应数/展示数)*100
    
    completedResponses: number;     // 完成数
    completedPercentage: number;    // 完成率 = (完成数/展示数)*100
    
    dropOffCount: number;           // 总流失数
    dropOffPercentage: number;      // 总流失率 = (流失数/响应数)*100
    
    ttcAverage: number;             // 平均完成耗时 (ms)
    
    quotasCompleted: number;        // 已完成配额数
    quotasCompletedPercentage: number;
  },
  
  // ───────────────────────────────────────────────
  // 2. 漏斗流失详情（每个问题）
  // ───────────────────────────────────────────────
  dropOff: [
    {
      elementId: string;
      elementType: string;
      headline: string;
      ttc: number;                  // 该问题平均耗时
      impressions: number;          // 该问题展示数
      dropOffCount: number;         // 该问题流失数
      dropOffPercentage: number;    // 该问题流失率
    }
  ],
  
  // ───────────────────────────────────────────────
  // 3. 各问题回答详情
  // ───────────────────────────────────────────────
  summary: ElementSummary[];        // 按问题类型的详细统计
  
  // ───────────────────────────────────────────────
  // 4. 配额统计
  // ───────────────────────────────────────────────
  quotas: QuotaSummary[];
}
```

### 6.2 输入-输出字段对应关系

| 输入过滤维度 | 影响的输出字段 | 说明 |
|-------------|----------------|------|
| **contactAttributes** | `meta.*`, `dropOff[].*`, `summary` | 按联系人属性分组后计算各组指标 |
| **createdAt** | `meta.displayCount`, `dropOff[].impressions` | 时间范围同时过滤 response 和 display |
| **finished** | `meta.completed*`, 所有流失计算 | 筛选完成/未完成响应 |
| **data** (问题回答) | `summary` 对应问题统计 | 仅计算符合回答条件的响应 |
| **tags** | 所有输出字段 | 按标签过滤后统计 |
| **meta** (userAgent/url等) | 所有输出字段 | 按元数据过滤后统计 |

### 6.3 指标计算公式汇总

```
漏斗层面：
├─ startsPercentage = (totalResponses / displayCount) * 100
├─ completedPercentage = (completedResponses / displayCount) * 100
├─ dropOffPercentage = (dropOffCount / totalResponses) * 100
└─ ttcAverage = sum(response.ttc) / completedResponses

问题层面：
├─ impressions[i] = 该问题被看到的响应数
├─ dropOffCount[i] = 在该问题流失的响应数
└─ dropOffPercentage[i] = (dropOffCount[i] / impressions[i]) * 100
```

## 7. 正常路径 vs 缺省路径

### 7.1 正常路径（有过滤条件）

```
输入: filterCriteria = { contactAttributes: { ab_test_variant: { op: "equals", value: "A" } } }
    ↓
Step 1: buildWhereClause() → 构建包含 contactAttributes 的 AND 条件
    ↓
Step 2: getResponsesForSummary() → 查询符合条件的响应列表
    ↓
Step 3: 提取 responseIds → 用于过滤 display
    ↓
Step 4: getDisplayCountBySurveyId(surveyId, { createdAt, responseIds })
    ↓
Step 5: 基于过滤后的响应集计算漏斗、meta、summary
    ↓
输出: 仅包含 Variant A 用户的分析数据
```

### 7.2 缺省路径（无过滤条件）

```
输入: filterCriteria = undefined
    ↓
Step 1: buildWhereClause() → 仅包含 surveyId 的基础条件
    ↓
Step 2: getResponsesForSummary() → 查询该 survey 的所有响应
    ↓
Step 3: hasFilter = false → 不提取 responseIds
    ↓
Step 4: getDisplayCountBySurveyId(surveyId, { createdAt })
    │  注意：display 仅按时间范围过滤，不关联 response
    ↓
Step 5: 基于全量响应计算漏斗、meta、summary
    ↓
输出: 该 survey 全量用户的分析数据
```

### 7.3 关键差异点

| 场景 | display 过滤方式 | 性能特征 | 适用场景 |
|------|------------------|----------|----------|
| **有过滤条件** | 通过 `responseIds` 关联过滤 | display 查询需 JOIN response，稍慢 | 分组对比、细分分析 |
| **无过滤条件** | 仅按时间范围直接计数 | display 查询无需 JOIN，更快 | 整体概览、大盘统计 |

## 8. A/B 测试分析查询实践

### 8.1 实验组对比查询模式

**场景**：对比 Variant A 和 Variant B 的漏斗表现

```typescript
// 分别查询两组数据
const [summaryA, summaryB] = await Promise.all([
  // Variant A
  getSurveySummary(surveyId, {
    contactAttributes: {
      ab_test_variant: { op: "equals", value: "variant_A" }
    }
  }),
  // Variant B
  getSurveySummary(surveyId, {
    contactAttributes: {
      ab_test_variant: { op: "equals", value: "variant_B" }
    }
  })
]);

// 对比关键指标：
// - summaryA.meta.completedPercentage vs summaryB.meta.completedPercentage
// - summaryA.meta.dropOffPercentage vs summaryB.meta.dropOffPercentage
// - 各问题 dropOff[i].dropOffPercentage 对比
```

### 8.2 多维度交叉分组

支持组合多个过滤维度进行细分分析：

```typescript
const filterCriteria = {
  // A/B 实验分组
  contactAttributes: {
    ab_test_variant: { op: "equals", value: "variant_B" }
  },
  // 用户地域
  meta: {
    country: { op: "equals", value: "US" }
  },
  // 时间范围
  createdAt: {
    min: new Date("2024-01-01"),
    max: new Date("2024-01-31")
  },
  // 仅已完成响应
  finished: true
};
```

### 8.3 按问题回答分组

基于特定问题的回答值进行漏斗对比：

```typescript
// 按 Q1 回答分组
const filterCriteriaQ1_A = {
  data: {
    "question_1_id": { op: "equals", value: "option_A" }
  }
};

const filterCriteriaQ1_B = {
  data: {
    "question_1_id": { op: "equals", value: "option_B" }
  }
};

// 分别查询后对比后续问题的流失率差异
```

## 9. 性能优化策略

### 9.1 游标分页机制

```typescript
// 避免 OFFSET 分页，使用游标基于 ID 排序
take: batchSize,
cursor: lastResponseId,
orderBy: [
  { createdAt: "desc" },
  { id: "desc" }  // 二级排序确保分页一致性
]
```

### 9.2 React Cache 去重

```typescript
export const getSurveySummary = reactCache(async () => { ... });
export const getResponsesForSummary = reactCache(async () => { ... });
export const getDisplayCountBySurveyId = reactCache(async () => { ... });
```

### 9.3 批量处理

```typescript
batchSize = 5000;  // 每批 5000 条响应
// 内存中聚合计算，而非数据库 GROUP BY
// 优势：支持复杂的漏斗逻辑、ttc 计算
// 劣势：大数据量时内存压力大
```

## 10. 设计决策总结

| 决策 | 方案 | 权衡 |
|------|------|------|
| **漏斗计算位置** | 应用层内存计算 | ✅ 灵活支持 ttc + 可见性判断<br>❌ 大数据量时内存压力 |
| **可见性判断** | ttc OR 有回答 | ✅ 避免重放问卷分支逻辑<br>✅ 基于实际行为更可靠 |
| **displayPercentage** | 客户端无状态随机 | ✅ 实现简单<br>❌ 分析查询时无法直接过滤 |
| **分组维度** | contactAttributes 为主 | ✅ 灵活支持自定义字段<br>✅ 查询时直接可用 |
| **display 过滤** | 有条件时通过 responseIds 关联 | ✅ 分组统计准确<br>❌ 需额外 JOIN 查询 |
| **"Other" 选项** | 排除所有预定义选项子集 | ✅ 精确匹配自由输入<br>❌ 子集生成复杂度 O(2^n) |
