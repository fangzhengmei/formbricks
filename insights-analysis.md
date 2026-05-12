# Formbricks 问卷响应分析机制详解

本文档详细说明 Formbricks 如何从原始问卷响应中提取分析洞察，重点涵盖**抽样窗口**、**NPS 推算**、**漏斗分组**三大核心机制。

---

## 1. 抽样窗口（响应数据获取策略）

### 核心实现
文件位置: `apps/web/app/(app)/environments/[environmentId]/surveys/[surveyId]/(analysis)/summary/lib/surveySummary.ts:969-1029`

### 游标分页机制

Formbricks 采用**基于游标（Cursor-based）的分页策略**而非传统的 offset 分页，针对大数据量进行优化：

```typescript
const batchSize = 5000;  // 每批处理 5000 条响应
const responses: TSurveySummaryResponse[] = [];
let cursor: string | undefined = undefined;
let hasMore = true;

while (hasMore) {
  const batch = await getResponsesForSummary(surveyId, batchSize, 0, filterCriteria, cursor);
  responses.push(...batch);
  
  if (batch.length < batchSize) {
    hasMore = false;  // 数据不足一批，终止
  } else {
    cursor = batch[batch.length - 1].id;  // 用最后一条 ID 作为下一批游标
  }
}
```

### 关键设计考量

| 特性 | 说明 |
|------|------|
| **游标条件** | `id: { lt: cursor }` - 获取 ID 小于当前游标的响应（降序） |
| **排序规则** | 主排序 `createdAt: desc`，次排序 `id: desc` 保证分页一致性 |
| **批次大小** | 固定 5000 条，平衡内存占用与数据库查询开销 |
| **过滤支持** | 支持通过 `filterCriteria` 进行时间范围、完成状态等过滤 |

### 数据选择字段

```typescript
select: {
  id: true,              // 响应唯一标识
  data: true,            // 问卷回答数据（JSON）
  updatedAt: true,       // 更新时间
  contact: { ... },      // 关联联系人信息
  contactAttributes: true,  // 联系人属性
  language: true,        // 回答语言
  ttc: true,             // Time-To-Complete 各题耗时
  finished: true,        // 是否完成问卷
}
```

### 过滤器联动

当存在过滤条件时，系统会提取 `responseIds` 并传递给 `getDisplayCountBySurveyId`，确保展示数（displayCount）与响应样本在同一时间维度上对齐，保证转化率计算准确。

---

## 2. NPS 分数推算机制

### 核心实现
文件位置: `apps/web/app/(app)/environments/[environmentId]/surveys/[surveyId]/(analysis)/summary/lib/surveySummary.ts:543-613`

### NPS 标准分组规则

NPS（Net Promoter Score）净推荐值计算遵循行业标准：

| 分组 | 分数范围 | 定义 |
|------|----------|------|
| **Promoters（推荐者）** | 9-10 分 | 忠实用户，积极推荐 |
| **Passives（被动者）** | 7-8 分 | 满意但不热情，易流失 |
| **Detractors（批评者）** | 0-6 分 | 不满意，可能负面宣传 |

### 计算流程

```typescript
const data = {
  promoters: 0,    // 推荐者计数
  passives: 0,     // 被动者计数
  detractors: 0,   // 批评者计数
  dismissed: 0,    // 看到但跳过的用户
  total: 0,        // 总样本
  score: 0,        // NPS 最终分数
};

responses.forEach((response) => {
  const value = response.data[element.id];
  if (typeof value === "number") {
    data.total++;
    scoreCountMap[value]++;  // 记录每个分值的分布
    
    if (value >= 9) data.promoters++;
    else if (value >= 7) data.passives++;
    else data.detractors++;
  } else if (response.ttc && response.ttc[element.id] > 0) {
    data.total++;
    data.dismissed++;  // 有停留时间但没打分 = 跳过
  }
});

// NPS 公式：(推荐者% - 批评者%) × 100
data.score = ((data.promoters - data.detractors) / data.total) * 100;
```

### 关键细节

#### 2.1 Dismissed（跳过）的判定

**双重确认机制**确保统计准确：
- 有 `ttc[elementId] > 0`（在该题上有停留时间）
- 但没有最终的打分数据

这区分了"没看到此题"与"看到了但跳过"两种情况，对分析问卷体验至关重要。

#### 2.2 分值分布明细

除了三大分组，系统还记录了 0-10 每个分值的精确计数与百分比，支持前端绘制完整的分数分布图：

```typescript
const choices = Object.entries(scoreCountMap).map(([rating, count]) => ({
  rating: Number.parseInt(rating),
  count,
  percentage: (count / data.total) * 100,
}));
```

#### 2.3 输出数据结构

```typescript
{
  type: "nps",
  responseCount: 100,
  score: 45,                    // NPS 分数：-100 到 +100
  promoters: { count: 60, percentage: 60 },
  passives: { count: 25, percentage: 25 },
  detractors: { count: 15, percentage: 15 },
  dismissed: { count: 5, percentage: 5 },
  choices: [...]                // 0-10 分的明细分布
}
```

---

## 3. 漏斗分组与流失率分析

### 核心实现
文件位置: `apps/web/app/(app)/environments/[environmentId]/surveys/[surveyId]/(analysis)/summary/lib/surveySummary.ts:134-223`

### 印象（Impressions）判定逻辑

**核心原则**：不重放问卷逻辑，只基于实际交互数据判断用户是否看到题目：

```typescript
const wasElementSeen = (response: TSurveySummaryResponse, elementId: string): boolean => {
  // 有该题的耗时记录，或者有回答数据 = 看到了
  return (response.ttc != null && response.ttc[elementId] > 0) 
         || response.data[elementId] !== undefined;
};
```

这种设计比基于问卷分支逻辑推算更可靠，因为：
- 避免了分支逻辑复杂带来的误判
- 兼容部分响应、中途退出等边界情况
- 真实反映用户实际交互行为

### 流失点归因算法

```typescript
let lastSeenIdx = -1;

// 遍历所有题目，找到用户最后交互到哪一题
for (let i = 0; i < elements.length; i++) {
  if (wasElementSeen(response, elements[i].id)) {
    impressionsArr[i]++;
    lastSeenIdx = i;  // 记录最后看到的题目索引
  }
}

// 未完成的问卷，流失点就记在最后一道看到的题目上
if (!response.finished && lastSeenIdx >= 0) {
  dropOffArr[lastSeenIdx]++;
}
```

**设计要点**：
- 流失（Drop-off）= 看到了此题，但没有继续完成后续
- 每道题的流失率 = 该题流失人数 / 该题曝光人数
- 准确反映问卷中"卡住"用户的具体位置

### 欢迎卡片的特殊校准逻辑

欢迎卡片（Welcome Card）的开关会直接影响首题的统计口径，代码执行顺序非常关键：

```typescript
if (!survey.welcomeCard.enabled) {
  // 步骤 1: 先算真实流失差值（用响应数据算出来的答题人数）
  dropOffArr[0] = displayCount - impressionsArr[0];
  
  // 步骤 2: 计算流失百分比，分母是 displayCount（总展示口径）
  dropOffPercentageArr[0] = impressionsArr[0] >= displayCount
    ? 0
    : ((displayCount - impressionsArr[0]) / displayCount) * 100 || 0;
  
  // 步骤 3: 最后把首题曝光数 回填 = 问卷总展示数（展示口径对齐）
  impressionsArr[0] = displayCount;
} else {
  // 有欢迎卡时，首题流失率按真实曝光口径计算
  dropOffPercentageArr[0] = impressionsArr[0] > 0 
    ? (dropOffArr[0] / impressionsArr[0]) * 100 
    : 0;
}
```

#### 两种口径对比表

| 指标 | 欢迎卡 = 关闭 | 欢迎卡 = 开启 |
|------|--------------|--------------|
| **首题曝光数** | = `displayCount`（问卷总展示数，展示口径） | = 真实看到首题的人数（响应数据口径） |
| **首题流失数** | = `displayCount - 真实开始答题人数`（差值口径） | = 看到首题但没继续的人数（响应数据口径） |
| **首题流失率** | = 流失数 / `displayCount`（总展示为分母） | = 流失数 / 首题真实曝光（首题曝光为分母） |

#### 设计意图说明

1. **关闭欢迎卡** = 用户一进来直接看到第一题，因此"首题曝光"等价于"问卷展示"，统计口径对齐有利于理解"展示到开始"的转化漏斗
2. **开启欢迎卡** = 用户先看到欢迎页，点击后才到第一题，因此首题曝光需要真实统计（排除只看了欢迎页就走的人）

### 耗时（TTC）块级聚合

为避免多题同块导致的重复计数，采用**块级最大耗时**策略：

```typescript
const getBlockTimesForResponse = (response, survey) => {
  return survey.blocks.reduce((acc, block) => {
    // 取该块内所有题目的最大耗时作为块耗时
    const maxElementTtc = block.elements.reduce((maxTtc, element) => {
      return Math.max(maxTtc, response.ttc?.[element.id] ?? 0);
    }, 0);
    acc[block.id] = maxElementTtc;
    return acc;
  }, {});
};
```

这确保了同一页面的多题不会导致总耗时被过度放大。

### 漏斗输出结构

```typescript
{
  elementId: "q1",
  elementType: "openText",
  headline: "您是如何找到我们的？",
  ttc: 12500,                       // 平均答题耗时（毫秒）
  impressions: 1000,                // 曝光人数
  dropOffCount: 150,                // 在此题流失人数
  dropOffPercentage: 15,            // 流失率 %
}
```

---

## 三大分析联动关系

```
问卷展示 (displayCount)
    │
    ├─ 抽样窗口 ──→ 按批次加载响应数据（游标分页）
    │                   │
    │                   ▼
    ├─ 漏斗分析 ──→ 计算各题曝光、流失、耗时
    │                   │
    │                   ▼
    └─ NPS 分析 ──→ 分组统计（推荐者/被动者/批评者）
                        │
                        ▼
                    综合洞察报告
```

### 关键一致性保障

1. **时间对齐**：通过 `createdAt` 过滤确保 displayCount 与响应样本在同一时间窗口
2. **样本一致**：NPS、漏斗、元数据统计共享同一批响应数据
3. **跨维度关联**：流失点数据可与 NPS 低分群体联动，识别问卷体验痛点

---

## 总结

Formbricks 的分析引擎设计注重**真实性**、**可靠性**和**性能**三大目标：
- 基于实际交互数据而非问卷逻辑推导，避免失真
- 游标分页而非 offset 分页，应对大数据量场景
- 细致的边界条件处理（dismissed、欢迎卡片等）
- 块级耗时聚合，准确反映真实答题体验

这套机制为产品决策提供了可信的数据基础。
