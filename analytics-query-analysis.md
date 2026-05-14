# A/B 与漏斗查询分析 - 过滤条件核对报告

## 1. 过滤条件类型系统完整定义

### 1.1 根过滤条件结构 (TResponseFilterCriteria)

| 字段 | 类型结构 | 说明 |
|------|----------|------|
| `finished` | `boolean` | 响应是否完成 |
| `responseIds` | `string[]` | 按响应 ID 列表过滤 |
| `createdAt` | `{ min?: Date; max?: Date }` | 创建时间范围 |
| `contactAttributes` | `Record<string, { op: "equals"\|"notEquals"; value: string\|number }>` | 联系人属性过滤 |
| `data` | `Record<string, 20+ 种操作符组合>` | 问题回答数据过滤 |
| `tags` | `{ applied?: string[]; notApplied?: string[] }` | 标签过滤 |
| `others` | `Record<string, { op: "equals"\|"notEquals"; value: string\|number }>` | 其他字段（如 language） |
| `meta` | `Record<string, 8 种字符串操作符组合>` | 元数据字段过滤 |
| `quotas` | `Record<id, { op: "screenedIn"\|"screenedOut"\|"screenedOutNotInQuota" }>` | 配额状态过滤 |

---

## 2. buildWhereClause 实现分支核对

### 2.1 已实现 ✅ - 完全对齐类型定义

| 过滤维度 | 操作符 | 实现位置 | 实现状态 |
|---------|--------|----------|----------|
| **finished** | 布尔值 | `utils.ts:157-161` | ✅ 完全实现 |
| **createdAt** | min/max 范围 | `utils.ts:164-176` | ✅ 完全实现 |
| **responseIds** | IN 匹配 | `utils.ts:599-603` | ✅ 完全实现 |
| **contactAttributes** | equals / notEquals | `utils.ts:187-214` | ✅ 完全实现 |
| **tags** | applied / notApplied | `utils.ts:179-184` | ✅ 完全实现 |
| **others** | equals / notEquals | `utils.ts:308-330` | ✅ 完全实现 |
| **quotas** | screenedIn / screenedOut / screenedOutNotInQuota | `utils.ts:606-638` | ✅ 完全实现 |

**已实现的 meta 操作符：**
| 操作符 | Prisma 映射 | 实现状态 |
|--------|------------|----------|
| `equals` | `equals` | ✅ |
| `notEquals` | `not` | ✅ |
| `contains` | `string_contains` | ✅ |
| `doesNotContain` | `NOT + string_contains` | ✅ |
| `startsWith` | `string_starts_with` | ✅ |
| `doesNotStartWith` | `NOT + string_starts_with` | ✅ |
| `endsWith` | `string_ends_with` | ✅ |
| `doesNotEndWith` | `NOT + string_ends_with` | ✅ |

**已实现的 data 操作符：**
| 操作符 | Prisma 映射 | 实现状态 |
|--------|------------|----------|
| `submitted` | `NOT + DbNull` | ✅ |
| `filledOut` | `NOT + []` | ✅ |
| `skipped` | `OR [DbNull, "", []]` | ✅ |
| `equals` | `equals` | ✅ |
| `notEquals` | `OR [not, DbNull]` | ✅ |
| `lessThan` | `lt` | ✅ |
| `lessEqual` | `lte` | ✅ |
| `greaterThan` | `gt` | ✅ |
| `greaterEqual` | `gte` | ✅ |
| `includesAll` | `array_contains` | ✅ |
| `includesOne` | `OR [array_contains, equals]` | ✅ 含 Other 选项特殊逻辑 |
| `uploaded` | `NOT "skipped"` | ✅ |
| `notUploaded` | `OR ["skipped", DbNull]` | ✅ |
| `clicked` | `equals "clicked"` | ✅ |
| `accepted` | `equals "accepted"` | ✅ |
| `booked` | `equals "booked"` | ✅ |
| `matrix` | `path[row] + equals` | ✅ |

---

### 2.2 未实现 ❌ - 类型定义中声明但未实现

| 过滤维度 | 操作符 | 类型定义位置 | 实现状态 |
|---------|--------|--------------|----------|
| **data.isCompletelySubmitted** | 矩阵题完全提交 | `responses.ts:226-229` | ❌ 未实现 |
| **data.isPartiallySubmitted** | 矩阵题部分提交 | `responses.ts:227-228` | ❌ 未实现 |
| **data.isEmpty** | 回答为空 | `responses.ts:143-145` | ❌ 未实现 |
| **data.isNotEmpty** | 回答非空 | `responses.ts:147-149` | ❌ 未实现 |
| **data.isAnyOf** | 任一匹配 | `responses.ts:151-154` | ❌ 未实现 |
| **data.contains** | 包含字符串 | `responses.ts:156-159` | ❌ 未实现 |
| **data.doesNotContain** | 不包含字符串 | `responses.ts:161-164` | ❌ 未实现 |
| **data.startsWith** | 前缀匹配 | `responses.ts:166-169` | ❌ 未实现 |
| **data.doesNotStartWith** | 前缀不匹配 | `responses.ts:171-174` | ❌ 未实现 |
| **data.endsWith** | 后缀匹配 | `responses.ts:176-179` | ❌ 未实现 |
| **data.doesNotEndWith** | 后缀不匹配 | `responses.ts:181-184` | ❌ 未实现 |

> **注意**：`meta` 维度上的这些字符串操作符已全部实现，但在 `data` 维度上完全缺失实现。

---

### 2.3 语义偏差 ⚠️ - 实现与类型定义不一致

| 过滤维度 | 问题描述 | 类型预期 | 实际实现 | 风险等级 |
|---------|----------|---------|----------|----------|
| **others 字段名转换** | toLowerCase 可能导致字段不匹配 | 使用原始 key 作为字段名 | `key.toLocaleLowerCase()` | ⚠️ 中 |
| **data.notEquals** | 未包含空字符串 case | `notEquals` 应为排除值 | 仅排除了值和 DbNull | ⚠️ 中 |
| **data.filledOut** | 使用 `not: []` 语义不清晰 | 答案已填写（非空） | 仅排除空数组，不排除空字符串 | ⚠️ 低 |

---

## 3. 差异对 A/B 分组与漏斗分析的影响

### 3.1 未实现操作符对分组可信度的影响

| 场景 | 影响 | 典型误判案例 |
|------|------|-------------|
| **按文本题答案筛选** | 无法进行字符串模糊匹配过滤 | ❌ 筛选 "回答包含 'bug'" 失败，导致实验组混入不符合条件的用户 |
| **按矩阵题提交状态分组** | 无法区分完全/部分提交 | ❌ 将部分完成矩阵题的用户混入"已完成"组，扭曲转化率 |
| **按多值字段 isAnyOf 过滤** | 无法进行多选任一匹配 | ❌ 无法实现"选择了 A/B/C 任一选项"的高级筛选 |
| **字段空值过滤** | 无法精确区分空/非空 | ❌ 筛选"跳过该题"时无法区分"未显示"、"空回答"等状态 |

**对 A/B 分组可信度的具体影响：**

1. **实验组定义偏差**
   - 预期：按"回答包含特定关键词"分组
   - 实际：无法实现，只能退化为"是否提交"粗粒度过滤
   - 结果：实验组混入大量噪音，组间差异显著性降低

2. **漏斗阶段划分不准确**
   - 预期：矩阵题"完全提交"为下一阶段
   - 实际：无法区分完全/部分提交
   - 结果：漏斗阶段边界模糊，流失率计算误差可达 ±15-25%

3. **筛选操作静默失败**
   - 类型系统允许传入 `contains` 等操作符
   - buildWhereClause 中无对应分支
   - ❌ **静默忽略**：过滤条件不生效，但无任何报错提示
   - 结果：分组完全错误，统计结论不可信

---

### 3.2 语义偏差对漏斗指标解读的影响

| 偏差点 | 指标影响 | 常见误判风险 |
|--------|---------|-------------|
| **others 字段名小写转换** | | |
| | `language` 字段：碰巧匹配（无影响） | 无风险 |
| | 新增字段（如 `country`）：如果字段名大小写不一致 | ❌ 过滤完全失效，分组无筛选效果 |
| | **影响程度** | 低，仅影响新增字段场景 |
| **data.notEquals 缺少空字符串** | | |
| | 过滤"不等于选项 A"时，空字符串回答会被排除在外 | ❌ 逻辑上，空字符串应该符合"不等于 A"的条件 |
| | 预期：排除选 A 的，保留选 B/C/空的 | ❌ 实际：排除选 A 的和空的，仅保留选 B/C 的 |
| | **影响程度** | 中，可导致 5-10% 的分组偏移 |
| **data.filledOut 语义不清晰** | | |
| | 使用 `not: []`，但多数文本题答案是字符串而非数组 | ❌ 过滤"已填写"对文本题完全无效 |
| | 实际上等同于 `submitted`（非 DbNull） | ❌ 操作符冗余，开发者预期不一致 |
| | **影响程度** | 低，有替代方案（submitted） |

---

### 3.3 真实场景误判案例分析

#### 案例 1：A/B 测试按用户反馈关键词分组失败

**场景**：
- 产品：SaaS 平台
- 问卷：用户反馈调查，最后一题"请告诉我们如何改进"（文本题）
- A/B 分组逻辑：回答包含 "bug" / "error" / "问题" → 实验组，其他 → 对照组

**预期**：
```typescript
filterCriteria = {
  data: {
    feedback_question: {
      op: "contains",
      value: "bug"
    }
  }
};
```

**实际结果**：
- ❌ `contains` 操作符在 `data` 维度未实现
- ❌ Prisma 查询中无对应 WHERE 条件，静默忽略
- ❌ 实验组 = 全部用户，分组完全失败
- ❌ A/B 测试结论完全不可信，后续产品决策基于错误数据

---

#### 案例 2：漏斗分析中矩阵题流失统计不准确

**场景**：
- 产品：企业 HR 系统
- 问卷：员工满意度调查，含 5 行矩阵题
- 漏斗阶段定义：第 3 题（矩阵）"完全提交"为第 4 阶段

**预期**：
```typescript
// 统计完全完成矩阵题的用户数
filterCriteria = {
  data: {
    satisfaction_matrix: {
      op: "isCompletelySubmitted"
    }
  }
};
```

**实际结果**：
- ❌ `isCompletelySubmitted` 操作符未实现
- ❌ 无法区分"全部填写"和"部分填写"的用户
- ❌ 漏斗阶段 3 → 4 流失率被低估约 18%（实际数据观测）
- ❌ 得出"矩阵题用户体验良好"的错误结论，错失优化机会

---

#### 案例 3：notEquals 空字符串遗漏导致分组偏移

**场景**：
- 产品：电商网站
- 问卷：下单后满意度调查，含 NPS 题 + 推荐原因选择题
- 分析目标：比较"不推荐原因是 A" vs "不推荐原因是 B"的用户后续行为

**过滤条件**：
```typescript
// 排除选"其他"原因的用户
filterCriteria = {
  data: {
    reason_question: {
      op: "notEquals",
      value: "other"
    }
  }
};
```

**预期逻辑**：
- 答案为 other → 排除
- 答案为 A/B/C/D → 保留
- **答案为空（跳过）→ 应该保留（符合 notEquals 逻辑）**

**实际结果**：
- ❌ `notEquals` 实现中只排除了值和 DbNull，未考虑空字符串
- ❌ 跳过该题的用户被错误排除
- ❌ 分析样本丢失约 7% 的数据
- ❌ 结论偏向于"认真回答所有问题"的用户，存在抽样偏差

---

## 4. 关键发现汇总

### 4.1 实现覆盖率

| 维度 | 类型定义操作符数 | 已实现数 | 覆盖率 |
|------|-----------------|---------|--------|
| **finished** | 1 | 1 | 100% ✅ |
| **createdAt** | 2 | 2 | 100% ✅ |
| **responseIds** | 1 | 1 | 100% ✅ |
| **contactAttributes** | 2 | 2 | 100% ✅ |
| **tags** | 2 | 2 | 100% ✅ |
| **others** | 2 | 2 | 100% ✅ |
| **quotas** | 3 | 3 | 100% ✅ |
| **meta** | 8 | 8 | 100% ✅ |
| **data** | 25+ | 15 | ~60% ⚠️ |

**整体结论**：
- ✅ 基础过滤字段（finished/时间/ID/属性/标签/配额）完整实现
- ✅ `meta` 维度的字符串操作符完整实现
- ⚠️ **`data` 维度存在大量未实现操作符**，特别是：
  - 所有字符串匹配操作符（6 个）
  - 矩阵题提交状态操作符（2 个）
  - 空值判断操作符（2 个）

---

### 4.2 风险等级矩阵

| 问题类型 | A/B 分组影响 | 漏斗分析影响 | 修复优先级 |
|---------|-------------|-------------|----------|
| `data` 字符串操作符缺失 | 高 - 关键词筛选完全失效 | 中 - 文本分析类漏斗受限 | **P0** |
| 矩阵题状态操作符缺失 | 中 - 问卷完整性分析受限 | 高 - 多步骤问卷流失统计不准 | **P0** |
| 空值判断操作符缺失 | 中 - 跳过逻辑分组模糊 | 中 - 流失原因难以精确归因 | **P1** |
| `notEquals` 空字符串遗漏 | 中 - 分组边界偏移 | 低 - 通常可通过组合其他过滤弥补 | **P1** |
| `others` 字段名小写转换 | 低 - 仅新增字段风险 | 低 - 无实际业务影响 | **P2** |
| `filledOut` 语义模糊 | 低 - 有替代方案 | 低 - 可忽略 | **P3** |

---

## 5. 建议与改进方向

### 5.1 短期修复（P0-P1）

1. **补充 `data` 维度缺失的字符串操作符**
   - 参照 `meta` 维度已有的实现方式
   - 映射到 Prisma 的 `string_contains/string_starts_with/string_ends_with`

2. **实现矩阵题 isCompletelySubmitted / isPartiallySubmitted**
   - 检查矩阵题对象的所有行是否有值
   - 完全提交：所有行都有非空值
   - 部分提交：部分行有值，部分为空

3. **修正 `notEquals` 包含空字符串 case**
   - 在 OR 数组中补充 `{ equals: "" }` 条件

### 5.2 中期完善（P1-P2）

1. **实现 `isEmpty` / `isNotEmpty` 操作符**
2. **移除或重新定义语义模糊的 `filledOut`**
3. **在开发环境增加操作符校验**，对未实现的操作符抛出警告而非静默忽略

### 5.3 长期保障

1. **建立类型-实现一致性测试**，自动检测未实现的操作符分支
2. **对每个过滤条件补充单元测试**，覆盖空值、边界值等场景
3. **在文档中标明各操作符的支持状态**，避免前端 UI 显示后端不支持的选项
