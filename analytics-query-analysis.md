# Analytics Query Analysis Report

## 1. 概述

本文档分析了 Formbricks 中 A/B 测试和漏斗分析的查询路径，重点关注：
- 指标输入结构
- 查询组合逻辑
- 结果返回关系

## 2. 核心架构

### 2.1 主要文件

| 文件 | 职责 |
|------|------|
| `apps/web/app/(app)/environments/[environmentId]/surveys/[surveyId]/(analysis)/summary/lib/surveySummary.ts` | 核心分析查询逻辑，汇总计算 |
| `apps/web/lib/response/utils.ts` | 查询条件构建 (buildWhereClause) |
| `packages/types/responses.ts` | 过滤条件类型定义 |
| `apps/web/app/(app)/environments/[environmentId]/surveys/[surveyId]/(analysis)/actions.ts` | Server Actions 入口 |

## 3. 指标输入 (Input)

### 3.1 过滤条件类型 TResponseFilterCriteria

```typescript
{
  finished?: boolean;              // 是否已完成
  responseIds?: string[];          // 指定响应ID列表
  createdAt?: {                    // 创建时间范围
    min?: Date;
    max?: Date;
  };
  contactAttributes?: Record<string, {   // 联系人属性过滤
    op: "equals" | "notEquals";
    value: string | number;
  }>;
  data?: Record<string, {           // 问题回答数据过滤
    op: ResponseFilterCondition;
    value?: any;
  }>;
  tags?: {                          // 标签过滤
    applied?: string[];
    notApplied?: string[];
  };
  others?: Record<string, {         // 其他字段（如 language）
    op: "equals" | "notEquals";
    value: string | number;
  }>;
  meta?: Record<string, {           // 元数据过滤
    op: equals | notEquals | contains | doesNotContain | startsWith | doesNotStartWith | endsWith | doesNotEndWith;
    value?: string;
  }>;
  quotas?: Record<string, {         // 配额过滤
    op: "screenedIn" | "screenedOut" | "screenedOutNotInQuota";
  }>;
}
```

### 3.2 支持的过滤操作符

| 操作符 | 适用场景 | 说明 |
|--------|----------|------|
| `equals` / `notEquals` | 通用 | 精确匹配/不匹配 |
| `lessThan` / `lessEqual` | 数值 | 小于/小于等于 |
| `greaterThan` / `greaterEqual` | 数值 | 大于/大于等于 |
| `includesAll` / `includesOne` | 多选 | 包含全部/包含任一 |
| `accepted` / `clicked` / `submitted` | 特殊题型 | 同意/点击/提交 |
| `skipped` / `booked` | 特殊题型 | 跳过/预约 |
| `uploaded` / `notUploaded` | 文件上传 | 已上传/未上传 |
| `isEmpty` / `isNotEmpty` | 通用 | 空/非空 |
| `contains` / `doesNotContain` | 字符串 | 包含/不包含 |
| `startsWith` / `endsWith` | 字符串 | 前缀/后缀匹配 |
| `isAnyOf` | 多选 | 多个值中任一匹配 |

## 4. 查询组合逻辑

### 4.1 主查询流程

```
getSurveySummary(surveyId, filterCriteria)
    │
    ├─→ 获取 Survey 定义
    │
    ├─→ 分批获取响应数据（游标分页）
    │   │
    │   └─→ getResponsesForSummary()
    │        │
    │        └─→ buildWhereClause() 构建 Prisma 查询
    │
    ├─→ 获取显示次数 (getDisplayCountBySurveyId)
    │
    ├─→ 获取配额信息
    │
    ├─→ 计算漏斗分析数据 (getSurveySummaryDropOff)
    │   │
    │   ├─→ 基于 ttc (time-to-complete) 计算可见性
    │   ├─→ 追踪每个问题的展示次数
    │   └─→ 计算每个环节的流失率
    │
    ├─→ 计算元数据 (getSurveySummaryMeta)
    │   │
    │   ├─→ 总响应数 / 完成数
    │   ├─→ 开始率 / 完成率
    │   ├─→ 流失数 / 流失率
    │   └─→ 平均完成时间
    │
    └─→ 计算各问题摘要 (getElementSummary)
        │
        ├─→ 文本题：样本收集
        ├─→ 选择题：选项分布统计
        ├─→ 评分题：平均值 / CSAT 计算
        ├─→ NPS 题：Promoter/Passive/Detractor 分组
        └─→ 矩阵题：行列交叉统计
```

### 4.2 buildWhereClause 构建逻辑

构建 SQL WHERE 子句，所有条件通过 `AND` 连接：

```typescript
AND: [
  { finished: true/false },           // 完成状态
  { createdAt: { lte?, gte? } },      // 时间范围
  { AND: tagFilters },                // 标签过滤
  { AND: contactAttributesFilters },  // 联系人属性
  { AND: metaFilters },               // 元数据
  { AND: othersFilters },             // 其他字段
  { AND: dataFilters },               // 回答数据
  { id: { in: responseIds } },        // 指定响应ID
  { AND: quotaFilters },              // 配额筛选
]
```

### 4.3 数据过滤关键点

#### 4.3.1 "Other" 选项特殊处理

当过滤条件包含 "other" 选项时，系统排除所有预定义选项的组合：

```typescript
// 包含 "其他" 选择时
if (values.includes(otherChoice.label.default)) {
  // 收集所有用户未选择的预定义选项
  const predefinedLabels = ...;
  
  // 生成所有子集排列组合
  const subsets = generateAllPermutationsOfSubsets(predefinedLabels);
  
  // 排除这些子集
  NOT: { OR: subsetConditions }
}
```

#### 4.3.2 矩阵题过滤

```typescript
{
  data: {
    path: [elementId, rowLabel],  // 嵌套路径
    equals: value
  }
}
```

## 5. 结果返回关系

### 5.1 TSurveySummary 返回结构

```typescript
{
  meta: {
    displayCount: number;           // 总展示次数
    totalResponses: number;         // 总响应数
    startsPercentage: number;       // 开始率 %
    completedResponses: number;     // 完成数
    completedPercentage: number;    // 完成率 %
    dropOffCount: number;           // 流失数
    dropOffPercentage: number;      // 流失率 %
    ttcAverage: number;             // 平均完成时间(ms)
    quotasCompleted: number;        // 已完成配额数
    quotasCompletedPercentage: number;
  },
  
  dropOff: [
    {
      elementId: string;
      elementType: string;
      headline: string;
      ttc: number;                  // 该问题平均耗时
      impressions: number;          // 展示次数
      dropOffCount: number;         // 流失数
      dropOffPercentage: number;    // 流失率 %
    }
  ],
  
  summary: ElementSummary[];       // 各问题详情
  quotas: QuotaSummary[];
}
```

### 5.2 漏斗分析计算逻辑 (getSurveySummaryDropOff)

#### 核心算法：基于 ttc + data 判断可见性

```typescript
// 判断响应是否真正看到了某个问题
wasElementSeen(response, elementId) {
  // 1. 有完成时间 (ttc > 0)，说明用户看到并停留过
  // 2. 有回答数据 (data[elementId] !== undefined)，说明用户作答了
  return ttc > 0 || hasAnswer;
}
```

#### 流失归因

```typescript
for each response:
  // 找到用户实际看到的最后一个问题索引
  lastSeenIdx = -1
  for i = 0 to elements.length:
    if wasElementSeen(response, elements[i].id):
      lastSeenIdx = i
  
  // 未完成的响应 -> 流失归因到最后一个看到的问题
  if !response.finished && lastSeenIdx >= 0:
    dropOffArr[lastSeenIdx]++
```

#### 首屏特殊处理

```typescript
if (!survey.welcomeCard.enabled) {
  // 无欢迎卡时，第一个问题的展示数 = 总展示数
  impressions[0] = displayCount
  dropOff[0] = displayCount - responsesCount
}
```

### 5.3 指标关系图

```
displayCount (总展示)
    │
    ├─→ startsPercentage = (totalResponses / displayCount) * 100
    │
    └─→ dropOff at element 0 (当无欢迎卡时)

totalResponses (总响应)
    │
    ├─→ completedResponses (已完成)
    │   └─→ completedPercentage
    │
    └─→ dropOffCount (未完成)
        └─→ dropOffPercentage

dropOff 数组 (每个问题的流失)
    │
    ├─→ impressions: 该问题被多少人看到
    ├─→ dropOffCount: 在该问题流失的人数
    └─→ dropOffPercentage: (dropOffCount / impressions) * 100
```

## 6. A/B 测试与分析查询关联

### 6.1 A/B 测试通过 Contact Attributes 实现

```typescript
// 过滤条件中通过联系人属性区分实验组
filterCriteria = {
  contactAttributes: {
    ab_test_variant: {
      op: "equals",
      value: "variant_A"  // "control" / "variant_A" / "variant_B"
    }
  }
}

// 分别查询各组数据
summary_A = getSurveySummary(surveyId, filterCriteria_A)
summary_B = getSurveySummary(surveyId, filterCriteria_B)

// 对比关键指标：
// - completedPercentage 完成率
// - ttcAverage 平均耗时
// - 各问题 dropOffPercentage 流失率差异
```

### 6.2 分组统计维度

| 维度 | 过滤方式 | 指标 |
|------|----------|------|
| 版本/Variant | contactAttributes | 完成率、耗时、NPS |
| 时间/Date | createdAt | 趋势分析 |
| 渠道/Source | meta.source | 渠道效果对比 |
| 地域/Country | meta.country | 地域差异 |
| 设备/Device | meta.userAgent.device | 设备适配 |

## 7. 性能考虑

### 7.1 游标分页

```typescript
// 避免 OFFSET 分页，使用游标基于 ID 排序
take: batchSize
cursor: lastResponseId
orderBy: [{ createdAt: desc }, { id: desc }]
```

### 7.2 缓存策略

```typescript
// React Cache 去重
export const getSurveySummary = reactCache(async () => { ... })
export const getResponsesForSummary = reactCache(async () => { ... })
```

### 7.3 数据批处理

```typescript
// 5000 条为一批处理
batchSize = 5000
```

## 8. 总结

### 8.1 输入 → 输出 数据流

```
Input: TResponseFilterCriteria
    ↓ [构建查询]
buildWhereClause() → Prisma WHERE
    ↓ [执行查询]
Responses (分批游标获取)
    ↓ [聚合计算]
    ├─→ meta (汇总指标)
    ├─→ dropOff (漏斗分析)
    └─→ summary (各问题统计)
    ↓
Output: TSurveySummary
```

### 8.2 关键设计决策

1. **内存聚合 vs 数据库聚合**：在应用层进行聚合计算，提供更灵活的统计逻辑
2. **ttc 优先判断可见性**：比重新回放问卷逻辑更可靠的用户行为追踪
3. **"Other" 选项排除法**：通过排除所有预定义选项组合来精确匹配自由输入
4. **分批处理大数据**：游标分页 + 批量计算支持大规模响应数据
