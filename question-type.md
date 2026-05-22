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
| **MultipleChoiceMulti** | `string[]` | `["选项A", "选项B", "", "其他自定义文本"]` | **存储的是选中的选项标签文本数组，不是ID**；`""` 是"other"的哨兵值，下一个元素为用户输入的自定义文本 |
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

> **修正点**：MultipleChoiceSingle/MultipleChoiceMulti 存储的是**标签文本**，不是选项 ID；Matrix 存储的是 rowLabel → columnLabel。

#### 多选 "other" 存储格式详解

**文件**: `multiple-choice-multi-element.tsx:94-144`

| 格式类型 | 示例 | 说明 |
|---------|------|------|
| **当前格式** | `["选项A", "", "自定义文本"]` | 空字符串 `""` 作为"other"哨兵值，紧跟自定义文本 |
| **历史兼容格式1** | `["选项A", "other", "自定义文本"]` | 使用 `"other"` 作为哨兵值 |
| **历史兼容格式2** | `["选项A", "自定义文本"]` | 无哨兵值，直接存储自定义文本 |

> 向后兼容：组件读取时支持上述 3 种格式；写入时统一使用当前格式（空字符串哨兵）。

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
- **MultipleChoiceMulti**：如选中 "other"（哨兵值 `""`），要求 "其他"文本非空

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

### 5.7 Survey 层逻辑整合

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
| | 多选 "other" 用空字符串 `""` 作为哨兵值，格式为 `["", "用户输入的自定义文本"]` |
| **代码证据** | 单选写回：`multiple-choice-single-element.tsx:94-100` |
| | 多选写回：`multiple-choice-multi-element.tsx:217-236` |
| | 逻辑评估时转换：`logic.ts:107-141` |

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
| **正确事实** | 支持 3 种存储格式（向后兼容）： |
| | 1. **当前格式**：`["选项A", "", "自定义文本"]` - 空字符串哨兵 |
| | 2. **历史格式1**：`["选项A", "other", "自定义文本"]` - "other" 哨兵 |
| | 3. **历史格式2**：`["选项A", "自定义文本"]` - 无哨兵 |
| | 逻辑评估时统一转换为 `["choiceId1", "choiceId2", "other"]` |
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
| **错误描述** | 未清晰说明动态 required 如何传递到验证流程 |
| **正确事实** | 完整衔接流程： |
| | 1. **当前 block 提交前**：`block-conditional.tsx:316` 调用 `validateBlockResponses(block.elements, value, languageCode)`，使用**原始的** `block.elements`（动态 required 未设置） |
| | 2. **当前 block 提交后**：`survey.tsx:697-796` 调用 `evaluateLogicAndGetNextBlockId` → 评估逻辑 → 执行动作 → 收集 `requiredQuestionIds` → 调用 `makeQuestionsRequired(requiredIds)` 修改 `localSurvey` state |
| | 3. **下一个 block 提交时**：使用**修改后的** `element.required` 进行验证 |
| **代码证据** | 验证调用：`block-conditional.tsx:316` |
| | state 修改：`survey.tsx:631-648` |
| | 完整流程：`survey.tsx:697-796` |
