# 问卷题型渲染管线与回答数据结构耦合分析

## 一、整体架构概览

Formbricks 的问卷系统采用**元数据驱动**的设计模式，通过 Zod Schema 定义题型元数据，前端根据元数据动态渲染控件，回答数据结构与题型 schema 强耦合，跨题分支逻辑直接依赖回答数据进行条件判断。

核心模块关系图：

```
┌─────────────────────────────────────────────────────────────────┐
│                     元数据定义层 (packages/types)                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌───────────────┐  │
│  │  elements.ts    │  │  types.ts      │  │  logic.ts     │  │
│  │  题型Schema    │  │  问卷/区块定义 │  │  逻辑条件定义 │  │
│  └─────────────────┘  └─────────────────┘  └───────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    前端渲染层 (packages/surveys)                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌───────────────┐  │
│  │ element-conditional │  │  各题型组件    │  │  logic.ts     │  │
│  │  类型分发器     │  │  (17种题型渲染  │  │  逻辑评估器   │  │
│  └─────────────────┘  └─────────────────┘  └───────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    数据层 (packages/types)                   │
│  ┌─────────────────┐  ┌─────────────────┐                      │
│  │ responses.ts    │  │ validation/        │                      │
│  │  回答数据结构   │  │ evaluator.ts     │                      │
│  └─────────────────┘  └─────────────────┘                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、题型元数据定义

### 2.1 题型枚举定义

**文件**: `packages/types/surveys/constants.ts`

```typescript
export enum TSurveyElementTypeEnum {
  FileUpload = "fileUpload",
  OpenText = "openText",
  MultipleChoiceSingle = "multipleChoiceSingle",
  MultipleChoiceMulti = "multipleChoiceMulti",
  NPS = "nps",
  CTA = "cta",
  Rating = "rating",
  Consent = "consent",
  PictureSelection = "pictureSelection",
  Cal = "cal",
  Date = "date",
  Matrix = "matrix",
  Address = "address",
  Ranking = "ranking",
  ContactInfo = "contactInfo",
  CSAT = "csat",
  CES = "ces",
}
```

### 2.2 基础元素 Schema

**文件**: `packages/types/surveys/elements.ts:57-70

```typescript
export const ZSurveyElementBase = z.object({
  id: ZSurveyElementId,                    // 元素唯一标识，用户可编辑
  type: z.enum(TSurveyElementTypeEnum),   // 题型枚举
  headline: ZI18nString,                    // 题目标题（多语言）
  subheader: ZI18nString.optional(),        // 题目描述（多语言）
  imageUrl: ZStorageUrl.optional(),         // 图片URL
  videoUrl: ZStorageUrl.optional(),         // 视频URL
  required: z.boolean(),                   // 是否必填
  scale: z.enum(["number", "smiley", "star"]).optional(),  // 评分尺度
  range: z.union([z.literal(5), z.literal(3), z.literal(4), z.literal(7), z.literal(10)]).optional(),
  isDraft: z.boolean().optional(),
});
```

### 2.3 各题型扩展字段

每种题型基于 `ZSurveyElementBase` 扩展特有字段：

| 题型 | 特有字段 | 数据类型 |
|------|---------|---------|
| **OpenText** | `placeholder`, `longAnswer`, `inputType` (text/email/url/number/phone), `charLimit`, `validation` | 见 [elements.ts:76-118 |
| **MultipleChoiceSingle** | `choices[]`, `shuffleOption`, `otherOptionPlaceholder`, `displayType` (list/dropdown) | 见 [elements.ts:152-160 |
| **MultipleChoiceMulti** | 同上 + `validation` | 见 [elements.ts:163-172 |
| **NPS** | `lowerLabel`, `upperLabel`, `isColorCodingEnabled` | 见 [elements.ts:183-188 |
| **Rating** | `scale` (必填), `range` (必填), `lowerLabel`, `upperLabel`, `isColorCodingEnabled` | 见 [elements.ts:232-239 |
| **Consent** | `label` (checkbox标签), `validation` | 见 [elements.ts:123-129 |
| **PictureSelection** | `allowMulti`, `choices[]` (含imageUrl), `validation` | 见 [elements.ts:251-258 |
| **Date** | `format` (M-d-y/d-M-y/y-M-d), `validation` | 见 [elements.ts:263-268 |
| **FileUpload** | `allowMultipleFiles`, `maxSizeInMB`, `allowedFileExtensions[]`, `validation` | 见 [elements.ts:273-279 |
| **Matrix** | `rows[]`, `columns[]`, `shuffleOption`, `validation` | 见 [elements.ts:302-308 |
| **Address** | `addressLine1/2`, `city`, `state`, `zip`, `country` (各含show/required/placeholder), `validation` | 见 [elements.ts:321-330 |
| **Ranking** | `choices[]`, `otherOptionPlaceholder`, `shuffleOption`, `validation` | 见 [elements.ts:335-348 |
| **ContactInfo** | `firstName`, `lastName`, `email`, `phone`, `company` (各含show/required/placeholder), `validation` | 见 [elements.ts:353-361 |
| **CSAT/CES** | 同 Rating (CSAT强制range=5, CES可选5/7) | 见 [elements.ts:366-387 |
| **CTA** | `buttonExternal`, `buttonUrl`, `ctaButtonLabel` | 见 [elements.ts:193-227 |
| **Cal** | `calUserName`, `calHost` | 见 [elements.ts:284-290 |

### 2.4 区块（Block）模型

问卷采用 Block-Element 二级结构，一个 Block 可包含多个 Element：

**文件**: `packages/types/surveys/blocks.ts:124-151`

```typescript
export const ZSurveyBlock = z
  .object({
    id: ZSurveyBlockId, // CUID 格式
    name: z.string().min(1, {
      error: "Block name is required",
    }), // 必填，用于编辑器
    elements: ZSurveyElements.min(1, {
      error: "Block must have at least one element",
    }),
    logic: z.array(ZSurveyBlockLogic).optional(),
    logicFallback: ZSurveyBlockId.optional(), // 必须是有效的 block ID
    buttonLabel: ZI18nString.optional(),
    backButtonLabel: ZI18nString.optional(),
  })
  .superRefine((block, ctx) => {
    // 校验 element ID 在 block 内唯一
    const elementIds = block.elements.map((e) => e.id);
    const uniqueElementIds = new Set(elementIds);
    if (uniqueElementIds.size !== elementIds.length) {
      ctx.addIssue({
        code: "custom",
        message: "Element IDs must be unique within a block",
        path: [elementIds.findIndex((id, index) => elementIds.indexOf(id) !== index), "id"],
      });
    }
  });
```

> **修正点**：Block 没有 `type` 字段，有必填的 `name` 字段；`id` 是 CUID 格式；`elements` 至少需要 1 个元素；`logicFallback` 必须是有效的 block ID。

---

## 三、元数据驱动前端控件渲染

### 3.1 类型分发器：ElementConditional

**文件**: `packages/surveys/src/components/general/element-conditional.tsx`

核心是一个大型 `switch-case` 分发器，根据 `element.type` 动态选择对应的渲染组件：

```typescript
const renderElement = () => {
  switch (element.type) {
    case TSurveyElementTypeEnum.OpenText:
      return <OpenTextElement element={element} ... />;
    case TSurveyElementTypeEnum.MultipleChoiceSingle:
      return <MultipleChoiceSingleElement element={element} ... />;
    case TSurveyElementTypeEnum.MultipleChoiceMulti:
      return <MultipleChoiceMultiElement element={element} ... />;
    case TSurveyElementTypeEnum.NPS:
      return <NPSElement element={element} ... />;
    case TSurveyElementTypeEnum.Rating:
    case TSurveyElementTypeEnum.CSAT:
    case TSurveyElementTypeEnum.CES:
      return <RatingElement element={element} ... />;
    // ... 17种题型
    default:
      return null;
  }
};
```

**关键耦合点**：

1. **值类型转换** (`element-conditional.tsx:121-348`):
   - `OpenText` → `string`
   - `MultipleChoiceSingle` → `string` (选项ID或标签文本)
   - `MultipleChoiceMulti` → `string[]`
   - `NPS/Rating/CSAT/CES` → `number`
   - `Matrix` → `Record<string, string>` (rowLabel → columnLabel)
   - `Address/ContactInfo` → `string[]` (按字段顺序)
   - `Ranking/PictureSelection/FileUpload` → `string[]`
   - `Consent/CTA/Date/Cal` → `string`

2. **17种题型组件映射**：
   - 每个组件接收 `element` (含完整元数据)、`value` (当前答案值)、`onChange` (回写回调)
   - 组件内部根据元数据字段渲染不同UI变体

### 3.2 典型题型组件示例：OpenTextElement

**文件**: `packages/surveys/src/components/elements/open-text-element.tsx`

```typescript
export function OpenTextElement({
  element,      // 完整元数据
  value,      // 当前值: string
  onChange,  // 回写回调: (data: TResponseData) => void
  languageCode,
  ...
}) {
  const handleChange = (inputValue: string) => {
    // 直接以 element.id 为 key 写回
    onChange({ [element.id]: inputValue });
  };

  // 根据元数据渲染不同变体
  const getInputType = (): "text" | "email" | "url" | "phone" | "number" => {
    if (element.inputType === "phone") return "phone";
    if (element.inputType === "email") return "email";
    // ...
    return "text";
  };

  return (
    <form onSubmit={handleOnSubmit}>
      <OpenText
        headline={getLocalizedValue(element.headline, languageCode)}
        placeholder={getLocalizedValue(element.placeholder, languageCode)}
        value={value}
        onChange={handleChange}
        required={element.required}
        longAnswer={element.longAnswer}
        inputType={getInputType()}
        charLimit={element.inputType === "text" ? element.charLimit : undefined}
        // ...
      />
    </form>
  );
}
```

### 3.3 区块渲染流程

**文件**: `packages/surveys/src/components/general/block-conditional.tsx

```
Block 渲染流程：
1. 遍历 block.elements 数组
2. 为每个 element 渲染 ElementConditional
3. 收集所有 element 的回答 → blockResponses
4. 提交时统一校验 → 传给 survey.tsx 的 onSubmit
```

---

## 四、回答数据结构与写回校验

### 4.1 回答数据结构定义

**文件**: `packages/types/responses.ts:7-44

```typescript
// 单个答案值类型：支持字符串、数字、字符串数组、对象
export const ZResponseDataValue = z
  .union([z.string(), z.number(), z.array(z.string()), z.record(z.string(), z.string())])
  .optional();

// 完整回答数据：key = element.id, value = 该题答案
export const ZResponseData = z.record(z.string(), ZResponseDataValue);
export type TResponseData = z.infer<typeof ZResponseData>;

// 完整 Response 结构
export const ZResponse = z.object({
  id: z.cuid2(),
  surveyId: z.cuid2(),
  finished: z.boolean(),
  data: ZResponseData,              // 核心回答数据
  variables: ZResponseVariables,       // 变量计算结果
  ttc: ZResponseTtc.optional(), // 每题耗时
  // ...
});
```

### 4.2 各题型答案值映射表

| 题型 | 值类型 | 示例值 | 说明 |
|------|--------|--------|------|
| **OpenText** | `string` | `"hello"` | 直接文本内容 |
| **MultipleChoiceSingle** | `string` | `"是"` | **存储的是选项标签文本(label)，不是选项ID**；`""` 表示选中"其他"但未填写 |
| **MultipleChoiceMulti** | `string[]` | `["选项A", "选项B", "其他自定义文本"]` | **存储的是选中的选项标签文本数组，不是ID**；"other" 选项直接追加自定义文本，**没有空字符串哨兵** |
| **NPS** | `number` | `9` | 0-10 评分 |
| **Rating** | `number` | `3` | 1-5/3/4/6/7/10 评分 |
| **Consent** | `string` | `"accepted"` 或 `""` | `"accepted"` 表示同意 |
| **CTA** | `string` | `"clicked"` 或 `""` | `"clicked"` 表示点击 |
| **Date** | `string` | `"2024-01-15"` | ISO格式日期 |
| **PictureSelection** | `string[]` | `["pic1", "pic2"]` | 选中的图片ID |
| **FileUpload** | `string[]` | `["https://.../file1.pdf"]` | 上传文件URL数组 |
| **Matrix** | `Record<string, string>` | `{"满意度": "非常满意", "易用性": "一般"}` | **rowLabel → columnLabel，不是rowId → columnId** |
| **Address** | `string[]` | `["123 Main St", "", "Beijing", "", "100000", "China"]` | 按 [addressLine1, addressLine2, city, state, zip, country] 顺序 |
| **Ranking** | `string[]` | `["choice2", "choice1", "choice3"]` | 按排名顺序的选项ID |
| **ContactInfo** | `string[]` | `["Zhang", "San", "zhang@example.com", "", ""]` | 按 [firstName, lastName, email, phone, company] 顺序 |
| **CSAT/CES** | `number` | `4` | 1-5 或 1-7 评分 |
| **Cal** | `string` | `"booked"` 或 `""` | `"booked"` 表示已预约 |

> **修正点**：MultipleChoiceSingle/MultipleChoiceMulti 存储的是**标签文本**，不是选项 ID；Matrix 存储的是 rowLabel → columnLabel；多选 other 写回时**没有空字符串哨兵**，哨兵格式仅用于历史兼容读取，不会被现行代码写入。

#### 多选 "other" 存储格式详解

**统一口径**：现行写入代码始终产生**无哨兵**格式；空字符串 `""`、`"other"` ID 等哨兵格式仅用于向后兼容读取，不会被写入。

##### 现行写入格式（两段写回路径）

**文件**: `multiple-choice-multi-element.tsx:168-175, 217-236`

```typescript
// 路径1：用户在"其他"输入框输入时
const handleOtherValueChange = (newOtherValue: string) => {
  setOtherValue(newOtherValue);
  const baseLabels = getNormalizedSelectedLabels();  // 已选标签（不含哨兵、不含other）
  const nextValue = [...baseLabels, newOtherValue];  // 直接追加自定义文本，无哨兵
  onChange({ [element.id]: nextValue });
};

// 路径2：用户选择/取消选择选项时
const handleMultiSelectChange = (selectedIds: string[]) => {
  const nextLabels: string[] = [];
  const isOtherNowSelected = Boolean(otherOption) && selectedIds.includes(otherOption!.id);

  selectedIds.forEach((id) => {
    if (id === otherOption?.id) return;  // 跳过 other ID
    const matchingOption = allOptions.find((opt) => opt.id === id);
    if (matchingOption) nextLabels.push(matchingOption.label);  // 存标签
  });

  if (isOtherNowSelected) {
    nextLabels.push(otherValue);  // 直接追加 otherValue，无哨兵
  }

  onChange({ [element.id]: nextLabels });
};
```

**现行写入格式示例**：`["产品A", "产品B", "用户输入的自定义文本"]`

##### 历史兼容读取格式（三种，仅读取不写入）

**文件**: `multiple-choice-multi-element.tsx:94-144`

| 格式类型 | 示例 | 检测方式 | 说明 |
|---------|------|---------|------|
| **格式1：空字符串哨兵** | `["产品A", "", "自定义文本"]` | `value.includes("")` | 注释标记为"Current"，但实际代码不写入 |
| **格式2：other ID 哨兵** | `["产品A", "other", "自定义文本"]` | `value.includes(otherOption.id)` | 历史格式，仅读取 |
| **格式3：无哨兵** | `["产品A", "自定义文本"]` | 检测未知值（不是已知标签也不是已知ID） | 与现行写入格式一致 |

**读取 other 值的提取逻辑**：
```typescript
// 格式1：["", "<custom>"] → 取哨兵后一位
const sentinelIndex = value.indexOf("");
if (sentinelIndex !== -1) {
  setOtherValue(value[sentinelIndex + 1] ?? "");
}

// 格式2：["other", "<custom>"] → 取 other ID 后一位
const otherIdIndex = value.indexOf(otherOption.id);
if (otherIdIndex !== -1) {
  setOtherValue(value[otherIdIndex + 1] ?? "");
}

// 格式3：["<custom>"] → 取第一个未知值
const unknown = value.find((v) => v !== "" && !knownLabels.has(v) && !knownIds.has(v));
setOtherValue(unknown ?? "");
```

##### 逻辑评估时的统一转换

**文件**: `logic.ts:124-140`

三种格式在逻辑评估时都会被统一转换为 choice ID 数组（含 `"other"` 标记）：

```typescript
// 遍历存储值（无论哪种格式）
responseValue.forEach((value) => {
  const foundChoice = currentQuestion.choices.find((choice) => {
    return getLocalizedValue(choice.label, selectedLanguage) === value;
  });

  if (foundChoice) {
    choices.push(foundChoice.id);  // 匹配标签 → choice ID
  } else if (isOthersEnabled) {
    choices.push("other");  // 不匹配（空字符串/other ID/自定义文本）→ "other"
  }
});

// 去重后返回：["choice_abc123", "other"]
return Array.from(new Set(choices));
```

> **关键统一口径**：
> - ✅ 写入：始终无哨兵，`["标签1", "标签2", "自定义文本"]`
> - ✅ 读取：兼容3种格式（空字符串哨兵、other ID 哨兵、无哨兵）
> - ✅ 逻辑评估：统一转换为 choice ID 数组，`"other"` 作为特殊标记
> - ❌ 现行代码**不写入**空字符串哨兵，注释与代码存在不一致，以实际代码为准

### 4.3 答案写回流程

**核心流程**（survey.tsx → block-conditional.tsx → element-conditional.tsx → 各题型组件

```
用户输入
    ↓
[题型组件] onChange({ [element.id]: value })
    ↓
[BlockConditional] handleElementChange()
    ├─ 合并到 blockResponses = { ...value, ...responseData }
    ├─ 触发自动进度判断
    └─ onChange(mergedValue) → 更新 Survey 级 responseData
    ↓
[Survey] onChange() → setResponseData(updatedResponseData)
    ↓
用户点击提交
    ↓
[BlockConditional] handleBlockSubmit()
    ├─ 1. validateBlockResponses() → 集中式校验
    ├─ 2. findFirstInvalidForm() → 传统HTML校验（兼容旧逻辑）
    ├─ 3. collectTtcValues() → 收集每题耗时
    ├─ 4. collectBlockResponses() → 收集本Block所有答案
    └─ onSubmit(blockResponses, blockTtc) → 提交到 Survey
    ↓
[Survey] onSubmit()
    ├─ evaluateLogicAndGetNextBlockId() → 逻辑评估
    ├─ onResponseCreateOrUpdate() → 发送到服务器/入队
    └─ setBlockId(nextBlockId) → 跳转到下一题
```

### 4.4 校验机制

#### 4.4.1 集中式校验器（新架构）

**文件**: `packages/surveys/src/lib/validation/evaluator.ts`

```typescript
// 验证规则定义见 validation-rules.ts:1-55
export const ZValidationRuleType = z.enum([
  "minLength", "maxLength", "pattern", "email", "url", "phone",
  "equals", "doesNotEqual", "contains", "doesNotContain",
  "minValue", "maxValue", "isGreaterThan", "isLessThan",
  "minSelections", "maxSelections",
  "minRanked", "rankAll",
  "minRowsAnswered", "answerAllRows",
  "isLaterThan", "isEarlierThan", "isBetween", "isNotBetween",
  "fileExtensionIs", "fileExtensionIsNot",
]);

// 验证对象结构见 elements.ts:50-55
export const ZValidation = z.object({
  rules: ZValidationRules,    // 规则数组
  logic: ZValidationLogic.prefault("and"),  // 组合逻辑: and/or
});
```

**校验流程** (`evaluator.ts:366-404):

```typescript
export const validateElementResponse = (
  element, value, languageCode) => {
  // 1. 必填校验（独立于 validation rules）
  const requiredError = checkRequiredField(element, value, t);

  // 2. 添加隐式规则（根据 element 类型自动添加
  rules = addImplicitOpenTextRules(element, rules);
  // 如：OpenText inputType=email → 自动添加 email 规则
  // 如：ContactInfo email.show → 自动添加 email 字段级 email 规则

  // 3. 执行规则
  if (validationLogic === "or") {
    return executeOrLogic(rules, ...);
  }
  return executeAndLogic(rules, ...);
};
```

**各题型可用规则**（validation-rules.ts:289-299):

| 题型 | 可用规则 |
|------|---------|
| openText | minLength, maxLength, pattern, email, url, phone, equals, doesNotEqual, contains, doesNotContain, minValue, maxValue, isGreaterThan, isLessThan |
| multipleChoiceMulti | minSelections, maxSelections |
| date | isLaterThan, isEarlierThan, isBetween, isNotBetween |
| matrix | minRowsAnswered, answerAllRows |
| ranking | minRanked, rankAll |
| fileUpload | fileExtensionIs, fileExtensionIsNot |
| address / contactInfo | minLength, maxLength, pattern, email, url, phone, equals, doesNotEqual, contains, doesNotContain |
| pictureSelection | minSelections, maxSelections |

#### 4.4.2 必填校验的特殊处理

- **CTA 元素**：即使 required=true 也不阻止跳转（仅作信息展示）
- **Ranking**：required 表示至少 1 项被排名
- **Matrix**：required 表示至少 1 行被回答
- **MultipleChoiceMulti**：如选中 "other"，要求 "其他"文本非空（`otherValue.trim() !== ""`），与哨兵格式无关

#### 4.4.3 传统HTML校验（兼容层）

**文件**: `block-conditional.tsx:219-257`

```typescript
const validateElementForm = (element, form) => {
  if (element.type === "address" || element.type === "contactInfo") {
    return form.checkValidity();  // HTML5 原生校验
  }
  if (element.type === "ranking") {
    return validateRankingElement(...);  // 自定义排名校验
  }
  if (element.type === "matrix") {
    // 至少1行回答
  }
  // 其他：检查 required + 空值
};
```

---

## 五、跨题分支逻辑与 Schema 依赖

### 5.1 逻辑条件定义

**文件**: `packages/types/surveys/logic.ts`

```typescript
// 条件操作符（共 32 种）
export const ZSurveyLogicConditionsOperator = z.enum([
  "equals", "doesNotEqual", "contains", "doesNotContain",
  "startsWith", "doesNotStartWith", "endsWith", "doesNotEndWith",
  "isSubmitted", "isSkipped", "isGreaterThan", "isLessThan",
  "isGreaterThanOrEqual", "isLessThanOrEqual",
  "equalsOneOf", "includesAllOf", "includesOneOf",
  "doesNotIncludeOneOf", "doesNotIncludeAllOf",
  "isClicked", "isNotClicked", "isAccepted",
  "isBefore", "isAfter", "isBooked",
  "isPartiallySubmitted", "isCompletelySubmitted",
  "isSet", "isNotSet", "isEmpty", "isNotEmpty", "isAnyOf",
]);

// 动态操作数
const ZDynamicElement = z.object({
  type: z.literal("element"),
  value: z.string(),      // element.id
  meta: z.record(z.string(), z.string()).optional(),  // 如 matrix 的 row 索引
});

// 单个条件
export const ZSingleCondition = z.object({
  id: ZId,
  leftOperand: ZDynamicLogicFieldValue,  // 左操作数：element/variable/hiddenField
  operator: ZSurveyLogicConditionsOperator,
  rightOperand: ZRightOperand.optional(),  // 右操作数：static/element/variable/hiddenField
});

// 条件组（支持嵌套）
export const ZConditionGroup = z.object({
  id: ZId,
  connector: ZConnector,  // and/or
  conditions: z.array(z.union([ZSingleCondition, ZConditionGroup])),
});

// 动作类型
export const ZSurveyBlockLogicAction = z.discriminatedUnion("objective", [
  z.object({
    id: ZId,
    objective: z.literal("jumpToBlock"),
    target: z.string(),  // 目标 block.id
  }),
  z.object({
    id: ZId,
    objective: z.literal("requireAnswer"),
    target: z.string(),  // 设为必填的 element.id
  }),
  z.object({
    id: ZId,
    objective: z.literal("calculate"),
    variableId: z.string(),
    operator: ZActionNumberVariableCalculateOperator | ZActionTextVariableCalculateOperator,
    value: z.union([ZRightOperandStatic, ZDynamicLogicFieldValue]),
  }),
]);
```

### 5.2 逻辑评估流程

**文件**: `packages/surveys/src/lib/logic.ts:28-48`

> **修正点**：`evaluateLogic` 仅负责**评估条件是否成立**，返回 `boolean`；**动作执行**由独立的 `performActions` 函数完成。两者分开调用：先 `evaluateLogic`，条件成立后再 `performActions`。

```typescript
// evaluateLogic 仅返回 boolean，表示条件组是否成立
export const evaluateLogic = (
  localSurvey, data, variablesData, conditions, selectedLanguage) => {
  const evaluateConditionGroup = (group) => {
    const results = group.conditions.map(condition => {
      if (isConditionGroup(condition)) {
        return evaluateConditionGroup(condition);  // 递归嵌套
      } else {
        return evaluateSingleCondition(localSurvey, data, variablesData, condition, selectedLanguage);
      }
    });
    return group.connector === "or"
      ? results.some(r => r)
      : results.every(r => r);
  };
  return evaluateConditionGroup(conditions);
};

// performActions 负责执行动作，返回执行结果
export const performActions = (survey, actions, data, calculationResults) => {
  let jumpTarget;
  const requiredQuestionIds = [];
  const calculations = { ...calculationResults };

  actions.forEach(action => {
    switch (action.objective) {
      case "calculate":
        const result = performCalculation(survey, action, data, calculations);
        if (result !== undefined) calculations[action.variableId] = result;
        break;
      case "requireAnswer":
        requiredQuestionIds.push(action.target);
        break;
      case "jumpToBlock":
        if (!jumpTarget) jumpTarget = action.target;
        break;
    }
  });

  return { jumpTarget, requiredQuestionIds, calculations };
};
```

### 5.3 左操作数取值与答案数据结构的耦合

**文件**: `logic.ts:84-180`

> **核心耦合点**：从 `TResponseData` 取出原始值后，必须根据 `element.type` 进行**类型转换**后才能用于逻辑比较。

#### 5.3.1 单选题（MultipleChoiceSingle）取值转换

```typescript
// 存储值（标签文本）: { "q1": "选项A" }
// 转换后（选项ID）: "choice_abc123" 或 "other"
if (currentQuestion.type === "multipleChoiceSingle" || currentQuestion.type === "multipleChoiceMulti") {
  const isOthersEnabled = currentQuestion.choices.some((c) => c.id === "other");

  if (typeof responseValue === "string") {
    // 通过标签文本查找对应的选项ID
    const choice = currentQuestion.choices.find((choice) => {
      return getLocalizedValue(choice.label, selectedLanguage) === responseValue;
    });

    if (!choice) {
      return isOthersEnabled ? "other" : undefined;
    }
    return choice.id;  // 返回选项ID，用于后续比较
  }
}
```

#### 5.3.2 多选题（MultipleChoiceMulti）取值转换

```typescript
// 存储值（标签数组）: { "q1": ["选项A", "选项B", "", "自定义文本"] }
// 转换后（选项ID数组）: ["choice_abc123", "choice_def456", "other"]
else if (Array.isArray(responseValue)) {
  let choices: string[] = [];
  responseValue.forEach((value) => {
    const foundChoice = currentQuestion.choices.find((choice) => {
      return getLocalizedValue(choice.label, selectedLanguage) === value;
    });

    if (foundChoice) {
      choices.push(foundChoice.id);
    } else if (isOthersEnabled) {
      choices.push("other");
    }
  });
  return Array.from(new Set(choices));  // 去重后的选项ID数组
}
```

#### 5.3.3 矩阵题（Matrix）取值转换

**必须通过 `leftOperand.meta.row` 指定行索引**：

```typescript
// 存储值: { "q2": {"满意度": "非常满意", "易用性": "一般"} }
// meta.row = "0"  → 取第1行的值 → 返回列索引的字符串形式: "2"
if (currentQuestion.type === "matrix" && typeof responseValue === "object") {
  if (leftOperand.meta && leftOperand.meta?.row !== undefined) {
    const rowIndex = Number(leftOperand.meta.row);
    if (isNaN(rowIndex) || rowIndex < 0 || rowIndex >= currentQuestion.rows.length) {
      return undefined;
    }

    const rowLabel = getLocalizedValue(currentQuestion.rows[rowIndex].label, selectedLanguage);
    const rowValue = responseValue[rowLabel];  // "非常满意"

    if (rowValue) {
      const columnIndex = currentQuestion.columns.findIndex((column) => {
        return getLocalizedValue(column.label, selectedLanguage) === rowValue;
      });
      return columnIndex === -1 ? undefined : columnIndex.toString();  // 返回 "2"
    }
  }
}
```

#### 5.3.4 日期题（Date）特殊比较逻辑

**文件**: `logic.ts:264-270, 309-315, 415-418`

存储值是 ISO 格式字符串（如 `"2024-01-15"`），在 `evaluateSingleCondition` 中进行特殊转换：

```typescript
case "equals":
  if ((leftField as TSurveyElement).type === TSurveyElementTypeEnum.Date) {
    // 字符串 → Date 对象 → 时间戳比较
    return new Date(leftValue).getTime() === new Date(rightValue).getTime();
  }
case "isAfter":
  return new Date(String(leftValue)) > new Date(String(rightValue));
case "isBefore":
  return new Date(String(leftValue)) < new Date(String(rightValue));
```

#### 5.3.5 完整取值流程

```typescript
const getLeftOperandValue = (localSurvey, data, variablesData, leftOperand, selectedLanguage) => {
  switch (leftOperand.type) {
    case "element":
      const questions = getElementsFromSurveyBlocks(localSurvey.blocks);
      const currentQuestion = questions.find(q => q.id === leftOperand.value);
      if (!currentQuestion) return undefined;

      const responseValue = data[leftOperand.value];  // 从 TResponseData 中取值

      // OpenText number 类型转数字
      if (currentQuestion.type === "openText" && currentQuestion.inputType === "number") {
        if (responseValue === undefined) return undefined;
        if (typeof responseValue === "string" && responseValue.trim() === "") return undefined;
        const numberValue = typeof responseValue === "number" ? responseValue : Number(responseValue);
        return isNaN(numberValue) ? undefined : numberValue;
      }

      // 单选/多选：标签 → 选项ID
      if (currentQuestion.type === "multipleChoiceSingle" || currentQuestion.type === "multipleChoiceMulti") {
        // ... 如上所示
      }

      // 矩阵：通过 meta.row 索引取特定行 → 返回列索引字符串
      if (currentQuestion.type === "matrix") {
        // ... 如上所示
      }

      return responseValue;  // 其他类型返回原始值

    case "variable":
      return getVariableValue(variables, leftOperand.value, variablesData);
    case "hiddenField":
      return data[leftOperand.value];
    default:
      return undefined;
  }
};
```

### 5.4 操作符与题型映射表

| 操作符 | 适用题型/类型 | 说明 |
|--------|---------------|------|
| `equals` / `doesNotEqual` | 所有 | 值相等/不等 |
| `contains` / `doesNotContain` | 文本类 | 字符串包含 |
| `startsWith` / `doesNotStartWith` | 文本类 | 开头匹配 |
| `endsWith` / `doesNotEndWith` | 文本类 | 结尾匹配 |
| `isSubmitted` | 所有 | 答案非空/已提交 |
| `isSkipped` | 所有 | 答案为空/跳过 |
| `isGreaterThan` / `isLessThan` | 数字类 (NPS, Rating, CSAT, CES, OpenText(number) | 数值比较 |
| `isGreaterThanOrEqual` / `isLessThanOrEqual` | 数字类 | 数值比较 |
| `equalsOneOf` | 单选类 | 等于多个值之一 |
| `includesAllOf` / `includesOneOf` | 多选类 (MultipleChoiceMulti, PictureSelection, Ranking | 数组包含 |
| `doesNotIncludeOneOf` / `doesNotIncludeAllOf` | 多选类 | 数组不包含 |
| `isClicked` / `isNotClicked` | CTA | 按钮点击状态 |
| `isAccepted` | Consent | 同意状态 |
| `isBefore` / `isAfter` | Date | 日期比较 |
| `isBooked` | Cal | 预约状态 |
| `isPartiallySubmitted` / `isCompletelySubmitted` | Address, ContactInfo | 复合字段提交状态 |
| `isSet` / `isNotSet` | 所有 | 值存在/不存在 |
| `isEmpty` / `isNotEmpty` | 文本类 | 空/非空 |
| `isAnyOf` | 所有 | 等于多个值之一（数组匹配） |

### 5.5 动作执行与数据回写

**文件**: `logic.ts:50-82`

```typescript
export const performActions = (survey, actions, data, calculationResults) => {
  let jumpTarget;
  const requiredQuestionIds = [];
  const calculations = { ...calculationResults };

  actions.forEach(action => {
    switch (action.objective) {
      case "calculate":
        // 变量计算：支持 static/element/variable/hiddenField 作为操作数
        const result = performCalculation(survey, action, data, calculations);
        if (result !== undefined) calculations[action.variableId] = result;
        break;
      case "requireAnswer":
        // 动态设为必填
        requiredQuestionIds.push(action.target);
        break;
      case "jumpToBlock":
        // 跳转到指定 block
        if (!jumpTarget) jumpTarget = action.target;
        break;
    }
  });

  return { jumpTarget, requiredQuestionIds, calculations };
};
```

### 5.6 validateBlockResponses 与 required 判定的衔接关系

> **修正点**：动态 required 影响的是**下一个 block** 的验证，不是当前 block。

#### 完整时序流程

```
用户在 Block A 填写答案
    ↓
用户点击 "Next" 按钮
    ↓
[BlockConditional.handleBlockSubmit]
    ├─ 1. validateBlockResponses(block.elements, value, languageCode)
    │   │  使用的是**原始的** block.elements（动态 required 尚未设置）
    │   └─ 内部调用 checkRequiredField(element, value, t)
    │       └─ 检查 element.required（静态值）
    ├─ 2. 如有错误 → 显示错误并停止
    └─ 3. 验证通过 → 调用 onSubmit(blockResponses, blockTtc)
            ↓
[Survey.handleBlockSubmit]
    ├─ 1. evaluateLogicAndGetNextBlockId(surveyResponseData)
    │   ├─ 遍历 Block A 的所有 logic 规则
    │   ├─ 对每条规则：
    │   │   ├─ evaluateLogic() → boolean（条件是否成立）
    │   │   └─ 条件成立 → performActions()
    │   │       ├─ jumpTarget（跳转目标）
    │   │       ├─ requiredQuestionIds（需要设为必填的 element ID）
    │   │       └─ calculations（变量计算结果）
    │   ├─ handleRequiredQuestions(allRequiredQuestionIds)
    │   │   └─ makeQuestionsRequired(requiredIds)
    │   │       └─ 修改 localSurvey state: element.required = true
    │   └─ 返回 nextBlockId
    ├─ 2. onResponseCreateOrUpdate() → 发送到服务器
    └─ 3. setBlockId(nextBlockId) → 跳转到 Block B
            ↓
用户在 Block B 填写答案
    ↓
用户点击 "Next" 按钮
    ↓
[BlockConditional.handleBlockSubmit]
    └─ 1. validateBlockResponses(block.elements, value, languageCode)
        └─ 使用的是**修改后的** block.elements（element.required = true）
```

#### 关键代码验证

**文件**: `survey.tsx:631-648` (makeQuestionsRequired 修改 state)

```typescript
const makeQuestionsRequired = (requiredQuestionIds: string[]): void => {
  const updateElementIfRequired = (element: TSurveyElement) => {
    if (requiredQuestionIds.includes(element.id)) {
      return { ...element, required: true };  // 修改 element.required
    }
    return element;
  };

  const updateBlockElements = (block: TSurveyBlock) => ({
    ...block,
    elements: block.elements.map(updateElementIfRequired),
  });

  // 更新 localSurvey state
  setlocalSurvey((prevSurvey) => ({
    ...prevSurvey,
    blocks: prevSurvey.blocks.map(updateBlockElements),
  }));
};
```

**文件**: `block-conditional.tsx:310-349` (先验证再提交)

```typescript
const handleBlockSubmit = (e?: Event) => {
  // 第一步：集中式校验（此时动态 required 还未设置）
  const errorMap = validateBlockResponses(block.elements, value, languageCode);
  if (Object.keys(errorMap).length > 0) {
    setElementErrors(errorMap);
    return;  // 验证失败，不提交
  }

  // 第二步：传统HTML校验（兼容层）
  const firstInvalidForm = findFirstInvalidForm();
  if (firstInvalidForm) {
    return;  // 验证失败，不提交
  }

  // 第三步：验证通过后才提交
  const blockTtc = collectTtcValues();
  const blockResponses = collectBlockResponses();
  onSubmit(blockResponses, blockTtc);  // 触发 Survey 层逻辑
};
```

### 5.6.1 前进与回退路径的 required 状态变化

> **新增内容**：动态 required 有完整的前进设置与回退恢复机制，涉及三个核心变量的联动。

#### 三个核心数据结构

| 变量 | 类型 | 初始化时机 | 作用 |
|------|------|-----------|------|
| `originalQuestionRequiredStates` | `Record<string, boolean>` | 组件初始化时，基于 `survey.blocks` 计算 | 保存所有题目的**原始 required 状态**（静态配置值），用于回退时恢复 |
| `questionRequiredByMap` | `useRef<Record<string, string[]>>` | 初始化为空对象 `{}` | 记录**哪个 block 的逻辑导致了哪些题被设为必填**，key 是 block 第一个 element.id，value 是被设为必填的 element.id 数组 |
| `localSurvey` state | 包含 `blocks: TSurveyBlock[]` | 初始化为 `survey` prop | 运行时的 survey 状态，`element.required` 可能被动态修改 |

**文件**: `survey.tsx:208-217`

```typescript
// 保存原始 required 状态（基于 survey.blocks，即静态配置）
const originalQuestionRequiredStates = useMemo(() => {
  return questions.reduce<Record<string, boolean>>((acc, question) => {
    acc[question.id] = question.required;
    return acc;
  }, {});
}, [survey.blocks]);  // 仅依赖原始 survey prop

// 记录逻辑 → required 变更的映射
const questionRequiredByMap = useRef<Record<string, string[]>>({});
```

#### 前进路径（Next 按钮）- 设置 required

**文件**: `survey.tsx:786-796`

```typescript
// 前进时：在 evaluateLogicAndGetNextBlockId 中调用
const handleRequiredQuestions = (requiredIds: string[]) => {
  if (requiredIds.length > 0) {
    // 记录：当前 block 的第一个 element.id → 被设为必填的题目列表
    if (currentBlock.elements[0]) {
      questionRequiredByMap.current[currentBlock.elements[0].id] = requiredIds;
    }
    // 修改 localSurvey state：将这些题的 required 设为 true
    makeQuestionsRequired(requiredIds);
  }
};
```

#### 回退路径（Back 按钮）- 恢复 required

**文件**: `survey.tsx:1007-1030`

```typescript
const onBack = (): void => {
  isNavigatingBackRef.current = true;

  // 从 history 获取前一个 block ID
  let prevBlockId: string | undefined;
  if (history.length > 0) {
    const newHistory = [...history];
    prevBlockId = newHistory.pop();
    setHistory(newHistory);
  } else {
    prevBlockId = localSurvey.blocks[currentBlockIndex - 1]?.id;
  }

  popVariableState();

  // 关键：回退时恢复 required 状态
  const prevBlock = localSurvey.blocks.find((b) => b.id === prevBlockId);
  if (prevBlock?.elements[0]) {
    // 用前一个 block 的第一个 element.id 作为 key，查找哪些题需要恢复
    revertRequiredChangesByQuestion(prevBlock.elements[0].id);
  }

  setBlockId(prevBlockId);
};
```

#### revertRequiredChangesByQuestion 实现

**文件**: `survey.tsx:650-677`

```typescript
const revertRequiredChangesByQuestion = (questionId: string): void => {
  // 从 map 中获取由该 question 所在 block 逻辑导致的必填题列表
  const questionsToRevert = questionRequiredByMap.current[questionId] || [];

  if (questionsToRevert.length > 0) {
    const revertElementIfNeeded = (element: TSurveyElement) => {
      if (questionsToRevert.includes(element.id)) {
        return {
          ...element,
          // 恢复为原始 required 状态，而不是简单设为 false
          required: originalQuestionRequiredStates[element.id] ?? element.required,
        };
      }
      return element;
    };

    const updateBlockElements = (block: TSurveyBlock) => ({
      ...block,
      elements: block.elements.map(revertElementIfNeeded),
    });

    setlocalSurvey((prevSurvey) => ({
      ...prevSurvey,
      blocks: prevSurvey.blocks.map(updateBlockElements),
    }));

    // 清理 map，避免重复恢复
    delete questionRequiredByMap.current[questionId];
  }
};
```

#### 完整时序对比

| 阶段 | 前进路径（Next） | 回退路径（Back） |
|------|-----------------|-----------------|
| **触发点** | 用户点击 Next 按钮 | 用户点击 Back 按钮 |
| **第一步** | 验证当前 block（用原始 required） | `isNavigatingBackRef.current = true` |
| **第二步** | 验证通过 → 提交答案 | 从 history 栈弹出 prevBlockId |
| **第三步** | 评估逻辑 → 收集 requiredQuestionIds | `popVariableState()` 恢复变量 |
| **第四步** | `questionRequiredByMap.current[currentBlockFirstElementId] = requiredIds` | `revertRequiredChangesByQuestion(prevBlockFirstElementId)` |
| **第五步** | `makeQuestionsRequired(requiredIds)` 修改 state | 恢复 `element.required = originalQuestionRequiredStates[id]` |
| **第六步** | 跳转到下一个 block | `delete questionRequiredByMap.current[questionId]` |
| **第七步** | 下一个 block 验证时使用修改后的 required | 跳转到前一个 block |

### 5.7 逻辑配置值与响应存储值的映射契约

> **新增内容**：为什么配置校验看 choice ID，运行时要把标签映射回 ID？

#### 两层不同的存储体系

| 层级 | 存储内容 | 数据类型 | 原因 |
|------|---------|---------|------|
| **逻辑配置层**（编辑器） | choice ID | 如 `"choice_abc123"` | ID 是稳定的，标签可被用户修改，用 ID 确保逻辑引用不失效 |
| **响应存储层**（运行时） | 标签文本 | 如 `"是"` | 标签是用户可见的最终值，便于数据分析和导出，不需要依赖 schema |

#### 配置时用 ID 校验

**文件**: `apps/web/modules/survey/editor/lib/utils.tsx:1502-1554`

```typescript
// 检查某个选项是否在逻辑中被使用（配置时）
export const findOptionUsedInLogic = (
  survey: TSurvey,
  elementId: string,
  optionId: string,  // 传入的是 choice ID
  checkInLeftOperand: boolean = false
): number => {
  const isUsedInOperand = (condition: TSingleCondition): boolean => {
    if (condition.leftOperand.type === "element" && condition.leftOperand.value === elementId) {
      if (!checkInLeftOperand && condition.rightOperand && condition.rightOperand.type === "static") {
        if (Array.isArray(condition.rightOperand.value)) {
          // 用 choice ID 进行匹配
          return condition.rightOperand.value.includes(optionId);
        } else {
          return condition.rightOperand.value === optionId;
        }
      }
    }
    return false;
  };
  // ...
};
```

**使用场景**：删除选项时检查是否在逻辑中使用 → 用 ID 匹配

#### 运行时标签 → ID 映射

**文件**: `packages/surveys/src/lib/logic.ts:107-141`

```typescript
// 左操作数取值转换（运行时）
if (currentQuestion.type === "multipleChoiceSingle" || currentQuestion.type === "multipleChoiceMulti") {
  const isOthersEnabled = currentQuestion.choices.some((c) => c.id === "other");

  if (typeof responseValue === "string") {
    // responseValue 是存储的标签文本，如 "是"
    // 通过标签查找对应的 choice ID
    const choice = currentQuestion.choices.find((choice) => {
      return getLocalizedValue(choice.label, selectedLanguage) === responseValue;
    });

    if (!choice) {
      if (isOthersEnabled) {
        return "other";
      }
      return undefined;
    }

    // 返回 choice ID，用于和逻辑配置中的 ID 比较
    return choice.id;
  }
}
```

#### 为什么需要这样的映射契约？

| 问题 | 解答 |
|------|------|
| **配置时为什么存 ID？** | 标签可修改（如多语言切换、编辑时重命名），ID 是 CUID 永不变化。如果存标签，修改标签后逻辑会失效。 |
| **运行时为什么存标签？** | 响应数据需要独立可读，便于导出、分析。如果存 ID，没有 schema 上下文无法理解数据含义。 |
| **为什么运行时要映射回 ID？** | 逻辑配置存的是 ID，运行时必须把标签转成 ID 才能正确比较 `equals` / `includesOneOf` 等操作。 |
| **多语言场景如何处理？** | 映射时使用 `getLocalizedValue(choice.label, selectedLanguage)` 按当前语言匹配。 |

### 5.8 完整示例链路

#### 5.8.1 单选题完整示例链路

**场景**：
- 题目："您的职业是？"（elementId: `"q1"`）
- 选项：`[{ id: "choice_abc123", label: "学生" }, { id: "choice_def456", label: "工程师" }, { id: "other", label: "其他" }]`
- 逻辑：如果 q1 等于 "工程师"（choice ID: `"choice_def456"`），则跳转到 Block B

**完整链路**：

```
1. 【配置时】编辑器保存逻辑条件
   rightOperand: { type: "static", value: "choice_def456" }
   ↑ 存的是 choice ID

2. 【运行时】用户选择 "工程师"
   ↓
   MultipleChoiceSingleElement.handleChange("choice_def456")
   ↓ 第99-100行：ID → 标签
   onChange({ q1: "工程师" })
   ↓
   responseData = { q1: "工程师" }
   ↑ 存的是标签文本

3. 【提交时】用户点击 Next
   ↓
   validateBlockResponses() → 验证通过
   ↓
   evaluateLogicAndGetNextBlockId()
   ↓
   evaluateLogic()
     ↓
     getLeftOperandValue("q1")
       ↓ 第107-124行：标签 → ID
       responseValue = "工程师"
       查找 choice.label === "工程师" → 找到 id = "choice_def456"
       return "choice_def456"
     ↓
     evaluateSingleCondition("equals")
       leftValue = "choice_def456"  (映射后的 ID)
       rightValue = "choice_def456" (逻辑配置的 ID)
       → true
     ↓
   performActions()
     jumpTarget = "block_b"
     ↓
   跳转到 Block B
```

**文件索引**：
- 写回：`multiple-choice-single-element.tsx:94-100`
- 映射：`logic.ts:107-124`
- 比较：`logic.ts:260-294`

#### 5.8.2 多选题完整示例链路

**场景**：
- 题目："您使用过以下哪些产品？"（elementId: `"q2"`）
- 选项：`[{ id: "choice_xxx", label: "产品A" }, { id: "choice_yyy", label: "产品B" }, { id: "other", label: "其他" }]`
- 逻辑：如果 q2 包含 "产品A"（choice ID: `"choice_xxx"`）或 "产品B"（choice ID: `"choice_yyy"`），则 q3 设为必填

**完整链路**：

```
1. 【配置时】编辑器保存逻辑条件
   rightOperand: { type: "static", value: ["choice_xxx", "choice_yyy"] }
   ↑ 存的是 choice ID 数组

2. 【运行时】用户选择 "产品A" 和 "其他"，输入 "产品C"
   ↓
   MultipleChoiceMultiElement.handleMultiSelectChange([ "choice_xxx", "other" ])
   ↓ 第218-235行：ID → 标签
   跳过 "other" ID
   nextLabels = ["产品A"]
   isOtherNowSelected = true → 追加 otherValue("产品C")
   onChange({ q2: ["产品A", "产品C"] })
   ↓
   responseData = { q2: ["产品A", "产品C"] }
   ↑ 存的是标签文本数组，无哨兵

3. 【提交时】用户点击 Next
   ↓
   validateBlockResponses() → 验证通过
   ↓
   evaluateLogicAndGetNextBlockId()
   ↓
   evaluateLogic()
     ↓
     getLeftOperandValue("q2")
       ↓ 第124-141行：标签数组 → ID 数组
       responseValue = ["产品A", "产品C"]
       遍历：
         "产品A" → 找到 id = "choice_xxx"
         "产品C" → 未匹配标签，启用了 other → 返回 "other"
       return ["choice_xxx", "other"]
     ↓
     evaluateSingleCondition("includesOneOf")
       leftValue = ["choice_xxx", "other"]  (映射后的 ID 数组)
       rightValue = ["choice_xxx", "choice_yyy"] (逻辑配置的 ID 数组)
       → true（包含 "choice_xxx"）
     ↓
   performActions()
     requiredQuestionIds = ["q3"]
     ↓
   questionRequiredByMap.current[currentBlockFirstElementId] = ["q3"]
   makeQuestionsRequired(["q3"])
   ↓
   跳转到下一个 block，q3.required 已变为 true

4. 【回退时】用户点击 Back
   ↓
   onBack()
     ↓
     revertRequiredChangesByQuestion(prevBlockFirstElementId)
       questionsToRevert = ["q3"]
       q3.required = originalQuestionRequiredStates["q3"] → 恢复原始值
       delete questionRequiredByMap.current[prevBlockFirstElementId]
     ↓
   回到上一个 block，q3.required 已恢复
```

**文件索引**：
- 写回：`multiple-choice-multi-element.tsx:218-235`
- 兼容读取：`multiple-choice-multi-element.tsx:94-113`
- 映射：`logic.ts:124-141`
- 比较：`logic.ts:391-396`
- 前进设置：`survey.tsx:786-796`
- 回退恢复：`survey.tsx:650-677, 1007-1030`

### 5.9 Survey 层逻辑整合

**文件**: `survey.tsx:697-805`

```typescript
const evaluateLogicAndGetNextBlockId = (data) => {
  // 1. 遍历当前 block 的所有 logic 规则
  let allRequiredQuestionIds: string[] = [];

  if (currentBlock.logic && currentBlock.logic.length > 0) {
    for (const logic of currentBlock.logic) {
      // 2. 评估条件（仅返回 boolean）
      const isLogicMet = evaluateLogic(
        localSurvey, localResponseData, calculationResults, logic.conditions, selectedLanguage);

      if (isLogicMet) {
        // 3. 执行动作（分离关注点）
        const { jumpTarget, requiredQuestionIds, calculations } = performActions(...);
        // 4. 收集需要动态设为必填的题目
        allRequiredQuestionIds = [...allRequiredQuestionIds, ...requiredQuestionIds];
        // 5. 设置跳转目标（第一个命中的优先）
        firstJumpTarget = firstJumpTarget ?? jumpTarget;
      }
    }
  }

  // 6. 修改 localSurvey state（影响下一个 block）
  handleRequiredQuestions(allRequiredQuestionIds);

  // 7. 无匹配时使用 logicFallback
  if (!firstJumpTarget && currentBlock.logicFallback) {
    firstJumpTarget = currentBlock.logicFallback;
  }

  // 8. 返回下一个 block ID
  return firstJumpTarget || localSurvey.blocks[currentBlockIndex + 1]?.id;
};
```

---

## 六、耦合关键点总结

### 6.1 元数据 → 控件 → 数据 强耦合链

```
ZSurveyElement (元数据定义
    ↓ type 字段
ElementConditional switch-case → 选择组件
    ↓ 组件根据元数据字段渲染UI
    ↓ onChange({ [element.id]: value )
TResponseData (key=element.id, value=题型对应类型
    ↓
evaluateLogic() 从 TResponseData 取值
    ↓ 根据 element.type 进行类型转换
    ↓ operator 匹配
逻辑条件成立 → 执行动作
```

### 6.2 关键设计模式

1. **判别联合类型**：Zod discriminated union 实现类型安全的题型定义
2. **访问者模式**：ElementConditional 作为分发器，各题型组件作为访问者
3. **策略模式**：验证规则和逻辑操作符均为可扩展策略
4. **解释器模式**：逻辑条件组递归解释器模式递归求值
5. **责任链模式**：多层验证（必填→隐式规则→自定义规则

### 6.3 数据一致性保证

1. **ID 作为唯一键**：element.id 贯穿元数据定义、前端渲染、数据存储、逻辑判断的唯一纽带
2. **类型一致性**：元数据定义值类型与逻辑评估时类型转换严格对应
3. **多语言处理**：标签比较时统一用 `getLocalizedValue()` 处理
4. **向后兼容**：标签和标签文本同时支持选项 ID 两种存储同时支持标签文本→ID 双向映射

### 6.4 扩展点

1. **新题型添加**：
   - 在 `TSurveyElementTypeEnum` 添加枚举
   - 在 `elements.ts` 定义 Zod Schema
   - 在 `element-conditional.tsx` 添加 case
   - 创建对应渲染组件
   - 在 `validation-rules.ts` 定义可用规则
   - 在 `logic.ts` 添加取值和操作符适配

2. **新验证规则**：
   - 在 `validation-rules.ts` 添加 ruleType 和 params
   - 在 `validators.ts` 实现 check 和 getDefaultMessage
   - 在 `APPLICABLE_RULES` 映射到适用题型

3. **新逻辑操作符**：
   - 在 `logic.ts` 的 `ZSurveyLogicConditionsOperator` 添加
   - 在 `evaluateSingleCondition` switch 添加 case

---

## 七、核心文件索引

| 模块 | 文件路径 | 核心功能 |
|------|---------|---------|
| 题型枚举 | `packages/types/surveys/constants.ts | TSurveyElementTypeEnum |
| 元素Schema | `packages/types/surveys/elements.ts` | 各题型 Zod 定义 |
| 逻辑条件 | `packages/types/surveys/logic.ts` | 条件/动作 Schema |
| 验证规则 | `packages/types/surveys/validation-rules.ts` | 验证规则类型定义 |
| 回答数据 | `packages/types/responses.ts` | TResponseData 等 |
| 类型分发器 | `packages/surveys/src/components/general/element-conditional.tsx` | 17种题型分发 |
| 区块渲染 | `packages/surveys/src/components/general/block-conditional.tsx` | Block级渲染与校验 |
| 主控制器 | `packages/surveys/src/components/general/survey.tsx` | Survey 状态机与提交流程 |
| 逻辑评估 | `packages/surveys/src/lib/logic.ts` | evaluateLogic / performActions |
| 验证评估 | `packages/surveys/src/lib/validation/evaluator.ts` | validateElementResponse / validateBlockResponses |
| OpenText组件 | `packages/surveys/src/components/elements/open-text-element.tsx` | 文本输入示例 |
| 单选组件 | `packages/surveys/src/components/elements/multiple-choice-single-element.tsx` | 单选示例 |

---

## 八、已修正清单（错误点 → 正确代码事实）

本小节逐条列出初始分析中的错误点，以及经过代码核对后的正确结论。

### 8.1 Block Schema 定义错误

| 项目 | 内容 |
|------|------|
| **错误描述** | `ZSurveyBlock` 有 `type: z.literal("Default("question")` 字段 |
| **正确事实** | Block 没有 `type` 字段，有必填的 `name: z.string().min(1)` 字段（用于编辑器）；`id` 是 CUID 格式；`elements` 至少需要 1 个元素；`logicFallback` 必须是有效的 block ID |
| **代码证据** | `packages/types/surveys/blocks.ts:124-151` |

### 8.2 单选/多选答案存储格式错误

| 项目 | 内容 |
|------|------|
| **错误描述** | 单选答案存储选项 ID，多选答案存储选项 ID 数组 |
| **正确事实** | 单选和多选答案存储的都是**标签文本(label)**，不是选项 ID |
| | 单选：`{ [elementId]: "选项A的标签文本" }` |
| | 多选：`{ [elementId]: ["选项A标签", "选项B标签", ...] }` |
| | 多选 "other" 写回时**直接追加自定义文本，无空字符串哨兵**；空字符串哨兵仅用于读取兼容 |
| **代码证据** | 单选写回：`multiple-choice-single-element.tsx:94-100` |
| | 多选写回：`multiple-choice-multi-element.tsx:168-175, 217-236` |
| | 逻辑评估时转换：`logic.ts:107-141` |
| | 兼容读取：`multiple-choice-multi-element.tsx:94-113` |

### 8.3 逻辑条件操作符数量错误

| 项目 | 内容 |
|------|------|
| **错误描述** | 有 38 种逻辑条件操作符 |
| **正确事实** | `ZSurveyLogicConditionsOperator` 定义了 **32 种**操作符 |
| **代码证据** | `packages/types/surveys/logic.ts:5-38` |

### 8.4 动态 required 的生效时机错误

| 项目 | 内容 |
|------|------|
| **错误描述** | 动态 required 在当前 block 提交时的 validateBlockResponses 中生效 |
| **正确事实** | 动态 required 通过修改 `localSurvey` state 中的 `element.required` 字段，影响的是**下一个 block** 的验证，不是当前 block |
| **时序说明** | 1. 当前 block 提交 → 2. validateBlockResponses（使用原始 required）→ 3. 验证通过 → 4. evaluateLogic → 5. performActions → 6. makeQuestionsRequired 修改 state → 7. 跳转到下一个 block → 8. 下一个 block 提交时使用修改后的 required |
| **代码证据** | 修改 state：`survey.tsx:631-648` |
| | 先验证后提交：`block-conditional.tsx:310-349` |

### 8.5 evaluateLogic 与 performActions 职责划分错误

| 项目 | 内容 |
|------|------|
| **错误描述** | `evaluateLogic` 同时负责条件评估和动作执行 |
| **正确事实** | `evaluateLogic` 仅返回 `boolean`（条件组是否成立）；`performActions` 负责执行动作，返回 `{ jumpTarget, requiredQuestionIds, calculations }`；两者是**分离调用**的：先 `evaluateLogic`，条件成立后再 `performActions` |
| **代码证据** | `logic.ts:28-48`（evaluateLogic） |
| | `logic.ts:50-82`（performActions） |
| | 调用顺序：`survey.tsx:727-748` |

### 8.6 多选 "other" 存储格式不完整

| 项目 | 内容 |
|------|------|
| **错误描述** | 仅提到用空字符串 `""` 作为哨兵值 |
| **正确事实** | 统一口径： |
| | ✅ **现行写入**：始终无哨兵，`["选项A", "自定义文本"]` |
| | ✅ **历史兼容读取**：支持3种格式（仅读取不写入） |
| | &nbsp;&nbsp;&nbsp;1. 空字符串哨兵：`["选项A", "", "自定义文本"]` |
| | &nbsp;&nbsp;&nbsp;2. other ID 哨兵：`["选项A", "other", "自定义文本"]` |
| | &nbsp;&nbsp;&nbsp;3. 无哨兵：`["选项A", "自定义文本"]`（与现行写入一致） |
| | ✅ **逻辑评估**：统一转换为 `["choiceId1", "choiceId2", "other"]` |
| **代码证据** | 读取检测：`multiple-choice-multi-element.tsx:94-113` |
| | 逻辑转换：`logic.ts:107-141` |

### 8.7 日期题条件判断比较方式不完整

| 项目 | 内容 |
|------|------|
| **错误描述** | 未提及日期题的特殊比较逻辑 |
| **正确事实** | 日期题在 `evaluateSingleCondition` 中有特殊处理，统一转换为 `Date` 对象比较： |
| | `equals`/`doesNotEqual`：`new Date(leftValue).getTime() === new Date(rightValue).getTime()` |
| | `isAfter`/`isBefore`：`new Date(String(leftValue)) > new Date(String(rightValue))` |
| **代码证据** | `logic.ts:264-270, 309-315, 415-418` |

### 8.8 矩阵题取值转换细节不完整

| 项目 | 内容 |
|------|------|
| **错误描述** | 仅提到存储格式是 rowLabel → columnLabel |
| **正确事实** | 补充： |
| | 1. 必须通过 `leftOperand.meta.row` 指定**行索引**（数字字符串，如 `"0"`） |
| | 2. 转换后返回的是**列索引的字符串形式**（如 `"2"` 表示第 3 列） |
| | 3. 行索引越界或列标签不匹配时返回 `undefined` |
| **代码证据** | `logic.ts:143-170` |

### 8.9 validateBlockResponses 与 required 判定的衔接关系不清晰

| 项目 | 内容 |
|------|------|
| **错误描述** | 未清晰说明动态 required 如何传递到验证流程，以及回退路径的状态恢复 |
| **正确事实** | 完整衔接流程涉及三个核心数据结构的联动： |
| | 1. `originalQuestionRequiredStates`（useMemo）- 保存原始静态配置的 required 状态 |
| | 2. `questionRequiredByMap`（useRef）- 记录哪个 block 逻辑导致哪些题被设为必填 |
| | 3. `localSurvey` state - 运行时状态，element.required 可能被动态修改 |
| | **前进路径**：evaluateLogic → performActions → handleRequiredQuestions → makeQuestionsRequired（修改 state） |
| | **回退路径**：onBack → revertRequiredChangesByQuestion（恢复 originalQuestionRequiredStates[id]）→ delete map entry |
| | 动态 required 影响的是**下一个 block** 的验证，不是当前 block |
| **代码证据** | 核心数据结构：`survey.tsx:208-217` |
| | 前进设置：`survey.tsx:631-648, 786-796` |
| | 回退恢复：`survey.tsx:650-677, 1007-1030` |
| | 先验证后提交：`block-conditional.tsx:310-349` |

### 8.10 逻辑配置值与响应存储值的映射契约缺失

| 项目 | 内容 |
|------|------|
| **错误描述** | 未说明为什么配置校验看 choice ID、运行时要把标签映射回 ID |
| **正确事实** | 两层不同的存储体系： |
| | **逻辑配置层（编辑器）**：存 choice ID（稳定，标签可修改，确保逻辑引用不失效） |
| | **响应存储层（运行时）**：存标签文本（独立可读，便于分析导出，不依赖 schema） |
| | **运行时映射原因**：逻辑配置存 ID，运行时必须把标签转成 ID 才能正确比较 |
| | **多语言处理**：映射时使用 `getLocalizedValue(choice.label, selectedLanguage)` 按当前语言匹配 |
| **代码证据** | 配置校验：`apps/web/modules/survey/editor/lib/utils.tsx:1502-1554` |
| | 运行时映射：`packages/surveys/src/lib/logic.ts:107-141` |

### 8.11 完整示例链路缺失

| 项目 | 内容 |
|------|------|
| **错误描述** | 未提供从配置到运行时的完整端到端示例 |
| **正确事实** | 补充了单选题和多选题各一条完整示例链路，包括： |
| | 1. 配置时保存 choice ID 到逻辑条件 |
| | 2. 运行时用户选择 → 组件 ID→标签转换 → 存储标签 |
| | 3. 提交时标签→ID 映射 → 与逻辑配置 ID 比较 |
| | 4. 逻辑成立 → 动态设置 required → 影响下一个 block |
| | 5. 回退时恢复原始 required 状态 |
| **代码证据** | 见文档 5.8 节完整示例 |

---

## 九、交叉校验清单：数据形态描述 ↔ 代码证据

本节列出文档中所有"数据形态描述"与对应代码证据的一一映射，确保全文前后一致，无自相矛盾。

### 9.1 答案存储格式类

| 序号 | 数据形态描述 | 代码证据 | 文档位置 |
|------|-------------|---------|---------|
| 1 | 单选题存储**标签文本**，不是选项 ID | `multiple-choice-single-element.tsx:99-100` <br> `const foundChoice = allOptions.find((opt) => opt.id === selectedId);` <br> `onChange({ [element.id]: getLocalizedValue(foundChoice.label, languageCode) });` | 4.2 节映射表 |
| 2 | 多选题存储**标签文本数组**，不是选项 ID | `multiple-choice-multi-element.tsx:222-226` <br> `selectedIds.forEach((id) => {` <br> `  if (id === otherOption?.id) return;` <br> `  const matchingOption = allOptions.find((opt) => opt.id === id);` <br> `  if (matchingOption) nextLabels.push(matchingOption.label);` <br> `});` | 4.2 节映射表 |
| 3 | 多选 other **现行写入无哨兵**，直接追加自定义文本 | `multiple-choice-multi-element.tsx:168-175` <br> `const nextValue = [...baseLabels, newOtherValue];` <br> `onChange({ [element.id]: nextValue });` <br><br> `multiple-choice-multi-element.tsx:228-231` <br> `if (isOtherNowSelected) {` <br> `  nextLabels.push(otherValue);` <br> `}` | 4.2.4 节 |
| 4 | 多选 other **历史兼容读取3种格式** | `multiple-choice-multi-element.tsx:94-113`（isOtherSelected） <br> `multiple-choice-multi-element.tsx:116-144`（useEffect 提取 other 值） | 4.2.4 节 |
| 5 | 多选 other 逻辑评估时**统一转换为 choice ID 数组**，含 `"other"` 标记 | `logic.ts:124-140` <br> `responseValue.forEach((value) => {` <br> `  const foundChoice = currentQuestion.choices.find(...);` <br> `  if (foundChoice) choices.push(foundChoice.id);` <br> `  else if (isOthersEnabled) choices.push("other");` <br> `});` | 4.2.4 节 |
| 6 | Matrix 存储 **rowLabel → columnLabel**，不是 ID | `matrix-element.tsx:130-140` <br> `const handleRowChange = (rowLabel, columnLabel) => {` <br> `  setValue((prev) => ({ ...prev, [rowLabel]: columnLabel }));` <br> `};` | 4.2 节映射表 |

### 9.2 逻辑配置与运行时类

| 序号 | 数据形态描述 | 代码证据 | 文档位置 |
|------|-------------|---------|---------|
| 7 | 逻辑配置层**存 choice ID**，确保标签修改后逻辑不失效 | `apps/web/modules/survey/editor/lib/utils.tsx:1502-1554` <br> `findOptionUsedInLogic(survey, elementId, optionId)` <br> 用 `optionId` 匹配 `condition.rightOperand.value` | 5.7 节 |
| 8 | 运行时**标签 → ID 映射**，用于逻辑比较 | `logic.ts:107-141` <br> 单选：`getLocalizedValue(choice.label, selectedLanguage) === responseValue` → `choice.id` <br> 多选：遍历数组，标签匹配 → choice ID，不匹配 → `"other"` | 5.3 节、5.7 节 |
| 9 | 日期题**Date 对象比较**，不是字符串比较 | `logic.ts:264-270`（equals） <br> `new Date(leftValue).getTime() === new Date(rightValue).getTime()` <br><br> `logic.ts:309-315`（isAfter） <br> `new Date(String(leftValue)) > new Date(String(rightValue))` | 5.3.4 节 |
| 10 | 矩阵题**meta.row 行索引**，返回**列索引字符串** | `logic.ts:148-169` <br> `const rowIndex = Number(leftOperand.meta.row);` <br> `const rowLabel = getLocalizedValue(currentQuestion.rows[rowIndex].label, selectedLanguage);` <br> `const columnIndex = currentQuestion.columns.findIndex(...);` <br> `return columnIndex.toString();` | 5.3.3 节 |

### 9.3 required 状态管理类

| 序号 | 数据形态描述 | 代码证据 | 文档位置 |
|------|-------------|---------|---------|
| 11 | `originalQuestionRequiredStates` 保存**原始静态配置** | `survey.tsx:208-214` <br> `useMemo(() => questions.reduce((acc, q) => {` <br> `  acc[q.id] = q.required;` <br> `  return acc;` <br> `}, {}), [survey.blocks]);` | 5.6.1 节 |
| 12 | `questionRequiredByMap` 记录**哪个 block 逻辑导致哪些题必填** | `survey.tsx:217` <br> `useRef<Record<string, string[]>>({});` <br><br> `survey.tsx:789-791` <br> `questionRequiredByMap.current[currentBlock.elements[0].id] = requiredIds;` | 5.6.1 节 |
| 13 | 前进路径：**先验证后设置 required**，影响下一个 block | `block-conditional.tsx:310-349`（先 validateBlockResponses） <br> `survey.tsx:631-648`（makeQuestionsRequired 修改 state） <br> `survey.tsx:796`（handleRequiredQuestions 在 evaluateLogic 之后调用） | 5.6 节 |
| 14 | 回退路径：**恢复 originalQuestionRequiredStates**，不是简单设为 false | `survey.tsx:650-677` <br> `revertElementIfNeeded(element) {` <br> `  return { ...element, required: originalQuestionRequiredStates[element.id] ?? element.required };` <br> `}` | 5.6.1 节 |
| 15 | 回退时**清理 questionRequiredByMap** 记录 | `survey.tsx:675` <br> `delete questionRequiredByMap.current[questionId];` | 5.6.1 节 |

### 9.4 操作符与枚举类

| 序号 | 数据形态描述 | 代码证据 | 文档位置 |
|------|-------------|---------|---------|
| 16 | 题型枚举共 **17 种** | `packages/types/surveys/constants.ts:3-19` <br> `FileUpload`, `OpenText`, `MultipleChoiceSingle`, `MultipleChoiceMulti`, <br> `NPS`, `CTA`, `Rating`, `Consent`, `PictureSelection`, `Cal`, <br> `Date`, `Matrix`, `Address`, `Ranking`, `ContactInfo`, `CSAT`, `CES` | 2.3 节 |
| 17 | 逻辑条件操作符共 **32 种** | `packages/types/surveys/logic.ts:5-37` <br> `equals`, `doesNotEqual`, `contains`, `doesNotContain`, <br> `startsWith`, `doesNotStartWith`, `endsWith`, `doesNotEndWith`, <br> `isSubmitted`, `isSkipped`, `isGreaterThan`, `isLessThan`, <br> `isGreaterThanOrEqual`, `isLessThanOrEqual`, `equalsOneOf`, `includesAllOf`, <br> `includesOneOf`, `doesNotIncludeOneOf`, `doesNotIncludeAllOf`, `isClicked`, <br> `isNotClicked`, `isAccepted`, `isBefore`, `isAfter`, `isBooked`, <br> `isPartiallySubmitted`, `isCompletelySubmitted`, `isSet`, `isNotSet`, <br> `isEmpty`, `isNotEmpty`, `isAnyOf` | 5.1 节 |
| 18 | `evaluateLogic` 仅返回 **boolean**，动作由 `performActions` 执行 | `logic.ts:28-48`（evaluateLogic 返回 boolean） <br> `logic.ts:50-82`（performActions 返回 `{ jumpTarget, requiredQuestionIds, calculations }`） | 5.2 节 |

### 9.5 Block Schema 类

| 序号 | 数据形态描述 | 代码证据 | 文档位置 |
|------|-------------|---------|---------|
| 19 | Block 无 `type` 字段，有**必填 `name` 字段** | `packages/types/surveys/blocks.ts:124-151` <br> `name: z.string().min(1, { error: "Block name is required" })` | 2.4 节 |
| 20 | Block `id` 是 **CUID 格式** | `packages/types/surveys/blocks.ts:125` <br> `id: ZSurveyBlockId`（ZSurveyBlockId 是 cuid2 格式） | 2.4 节 |
| 21 | Block `elements` 至少需要 **1 个元素** | `packages/types/surveys/blocks.ts:131-133` <br> `elements: ZSurveyElements.min(1, { error: "Block must have at least one element" })` | 2.4 节 |
| 22 | Block 内 element ID **必须唯一** | `packages/types/surveys/blocks.ts:138-150` <br> `superRefine((block, ctx) => {` <br> `  const uniqueElementIds = new Set(elementIds);` <br> `  if (uniqueElementIds.size !== elementIds.length) { /* add issue */ }` <br> `})` | 2.4 节 |

### 9.6 一致性校验说明

| 校验项 | 状态 |
|--------|------|
| 多选 other 写入格式全文一致 | ✅ 统一为"无哨兵，直接追加" |
| 多选 other 读取格式全文一致 | ✅ 统一为"3种历史兼容" |
| 单选/多选存储类型全文一致 | ✅ 统一为"标签文本，不是 ID" |
| 动态 required 生效时机全文一致 | ✅ 统一为"影响下一个 block" |
| 逻辑操作符数量全文一致 | ✅ 统一为 32 种 |
| Block Schema 字段全文一致 | ✅ 无 type，有 name，至少 1 个 element |
