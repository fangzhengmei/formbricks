# Formbricks 问卷题型系统架构报告

## 一、题型注册

所有支持的题型通过枚举统一注册管理，类型定义位于 `packages/types` 包中。

### 1.1 题型枚举定义

**文件路径**：`packages/types/surveys/constants.ts`

```typescript
// packages/types/surveys/constants.ts:1-18
export enum TSurveyElementTypeEnum {
  FileUpload = "fileUpload",              // 文件上传
  OpenText = "openText",                  // 开放文本
  MultipleChoiceSingle = "multipleChoiceSingle",  // 单选题
  MultipleChoiceMulti = "multipleChoiceMulti",    // 多选题
  NPS = "nps",                            // NPS 量表
  CTA = "cta",                            // 行动号召
  Rating = "rating",                      // 评分量表
  Consent = "consent",                    // 同意书
  PictureSelection = "pictureSelection",  // 图片选择
  Cal = "cal",                            // 日历预约
  Date = "date",                          // 日期选择
  Matrix = "matrix",                      // 矩阵题
  Address = "address",                    // 地址信息
  Ranking = "ranking",                    // 排名题
  ContactInfo = "contactInfo",            // 联系信息
}
```

### 1.2 题型类型定义

**文件路径**：`packages/types/surveys/elements.ts`

每种题型都有对应的 Zod schema 和 TypeScript 类型定义，继承自基础元素结构：

```typescript
// packages/types/surveys/elements.ts:57-70
export const ZSurveyElementBase = z.object({
  id: ZSurveyElementId,
  type: z.enum(TSurveyElementTypeEnum),    // 关联枚举类型
  headline: ZI18nString,
  subheader: ZI18nString.optional(),
  imageUrl: ZStorageUrl.optional(),
  videoUrl: ZStorageUrl.optional(),
  required: z.boolean(),
  scale: z.enum(["number", "smiley", "star"]).optional(),
  range: z.union([z.literal(5), z.literal(3), z.literal(4), z.literal(7), z.literal(10)]).optional(),
  isDraft: z.boolean().optional(),
});
```

每种题型扩展基础结构添加特有字段，例如矩阵题：

```typescript
// packages/types/surveys/elements.ts:302-310
export const ZSurveyMatrixElement = ZSurveyElementBase.extend({
  type: z.literal(TSurveyElementTypeEnum.Matrix),
  rows: z.array(ZSurveyMatrixElementChoice),
  columns: z.array(ZSurveyMatrixElementChoice),
  shuffleOption: ZShuffleOption.optional().prefault("none"),
  validation: ZValidation.optional(),
});
```

最后通过 union 类型汇总所有题型：

```typescript
// packages/types/surveys/elements.ts:366-384
export const ZSurveyElement = z.union([
  ZSurveyOpenTextElement,
  ZSurveyConsentElement,
  ZSurveyMultipleChoiceSingleElement,
  ZSurveyMultipleChoiceMultiElement,
  ZSurveyNPSElement,
  ZSurveyCTAElement,
  ZSurveyRatingElement,
  ZSurveyPictureSelectionElement,
  ZSurveyDateElement,
  ZSurveyFileUploadElement,
  ZSurveyCalElement,
  ZSurveyMatrixElement,
  ZSurveyAddressElement,
  ZSurveyRankingElement,
  ZSurveyContactInfoElement,
]);
```

---

## 二、按 type 分发

题型分发通过 `ElementConditional` 组件实现，根据 `element.type` 动态选择对应的渲染组件。

### 2.1 分发器核心实现

**文件路径**：`packages/surveys/src/components/general/element-conditional.tsx`

```typescript
// packages/surveys/src/components/general/element-conditional.tsx:113-352
const renderElement = () => {
  switch (element.type) {
    case TSurveyElementTypeEnum.OpenText:
      return (
        <OpenTextElement
          key={element.id}
          element={element}
          value={typeof value === "string" ? value : ""}
          onChange={onChange}
          languageCode={languageCode}
          ttc={ttc}
          setTtc={wrappedSetTtc}
          autoFocusEnabled={autoFocusEnabled}
          currentElementId={currentElementId}
          dir={dir}
          errorMessage={errorMessage}
        />
      );
    case TSurveyElementTypeEnum.MultipleChoiceMulti:
      return (
        <MultipleChoiceMultiElement
          key={element.id}
          element={element}
          value={Array.isArray(value) ? value : []}
          onChange={onChange}
          languageCode={languageCode}
          ttc={ttc}
          setTtc={wrappedSetTtc}
          autoFocusEnabled={autoFocusEnabled}
          currentElementId={currentElementId}
          dir={dir}
          errorMessage={errorMessage}
        />
      );
    case TSurveyElementTypeEnum.Matrix:
      return (
        <MatrixElement
          element={element}
          value={typeof value === "object" && !Array.isArray(value) ? value : {}}
          onChange={onChange}
          languageCode={languageCode}
          ttc={ttc}
          setTtc={wrappedSetTtc}
          currentElementId={currentElementId}
          errorMessage={errorMessage}
          dir={dir}
        />
      );
    // ... 其他 15+ 种题型的分发分支
    default:
      return null;
  }
};
```

### 2.2 分发器的 Props 适配层

分发器负责将统一的响应值转换为各题型组件期望的具体类型：

```typescript
// 单选题：转换为 string | undefined
value={typeof value === "string" ? value : undefined}

// 多选题：转换为 string[]
value={Array.isArray(value) ? value : []}

// 矩阵题：转换为 Record<string, string>
value={typeof value === "object" && !Array.isArray(value) ? value : {}}

// 评分/NPS：转换为 number | undefined
value={typeof value === "number" ? value : undefined}
```

---

## 三、统一提交模型

### 3.1 响应数据模型定义

**文件路径**：`packages/types/responses.ts`

所有题型的答案统一存储在 `TResponseData` 对象中：

```typescript
// packages/types/responses.ts:7-44
export const ZResponseDataValue = z
  .union([z.string(), z.number(), z.array(z.string()), z.record(z.string(), z.string())])
  .optional();

export type TResponseDataValue = z.infer<typeof ZResponseDataValue>;

export const ZResponseData = z.record(z.string(), ZResponseDataValue);
export type TResponseData = z.infer<typeof ZResponseData>;

// 答题时间追踪（Time To Complete）
export const ZResponseTtc = z.record(z.string(), z.number());
export type TResponseTtc = z.infer<typeof ZResponseTtc>;
```

### 3.2 完整调用链

#### 层级 1：题型组件 → onChange 触发

以多选题组件为例：

**文件路径**：`packages/surveys/src/components/elements/multiple-choice-multi-element.tsx`

```typescript
// packages/surveys/src/components/elements/multiple-choice-multi-element.tsx:217-239
const handleMultiSelectChange = (selectedIds: string[]) => {
  const nextLabels: string[] = [];
  const isOtherNowSelected = Boolean(otherOption) && selectedIds.includes(otherOption!.id);

  selectedIds.forEach((id) => {
    if (id === otherOption?.id) return;
    const matchingOption = allOptions.find((opt) => opt.id === id);
    if (matchingOption) nextLabels.push(matchingOption.label);
  });

  if (isOtherNowSelected) {
    nextLabels.push(otherValue);
  }

  // 调用从 ElementConditional 传入的 onChange
  onChange({ [element.id]: nextLabels });

  // 更新答题时间
  const updatedTtcObj = getUpdatedTtc(ttc, element.id, performance.now() - startTime);
  setTtc(updatedTtcObj);
};
```

矩阵题组件的 onChange 实现：

**文件路径**：`packages/surveys/src/components/elements/matrix-element.tsx`

```typescript
// packages/surveys/src/components/elements/matrix-element.tsx:114-123
const handleChange = (newValue: Record<string, string>) => {
  const labelValue = convertValueFromIds(newValue);

  if (Object.values(labelValue).every((val) => val === "")) {
    onChange({ [element.id]: {} });
  } else {
    onChange({ [element.id]: labelValue });
  }
};
```

#### 层级 2：ElementConditional → BlockConditional 透传

**文件路径**：`packages/surveys/src/components/general/element-conditional.tsx`

onChange 从 BlockConditional 传入，透传给题型组件：

```typescript
// packages/surveys/src/components/general/element-conditional.tsx:76-84
// 包装 setTtc 回调，实现同步 TTC 收集
const wrappedSetTtc = (newTtc: TResponseTtc) => {
  setTtc(newTtc);
  // 提取当前元素的 TTC 并调用收集器
  if (onTtcCollect && newTtc[element.id] !== undefined) {
    onTtcCollect(element.id, newTtc[element.id]);
  }
};
```

#### 层级 3：BlockConditional 处理与提交

**文件路径**：`packages/surveys/src/components/general/block-conditional.tsx`

接收单个元素的变化，合并到 Block 级响应数据：

```typescript
// packages/surveys/src/components/general/block-conditional.tsx:91-136
const handleElementChange = (elementId: string, responseData: TResponseData) => {
  if (elementId !== currentElementId) {
    setCurrentElementId(elementId);
  }

  // 清除验证错误
  if (elementErrors[elementId]) {
    setElementErrors((prev: TValidationErrorMap) => {
      const updated = { ...prev };
      delete updated[elementId];
      return updated;
    });
  }

  // 合并当前 Block 内的所有答案
  const mergedValue = { ...value, ...responseData };
  const blockResponses = block.elements.reduce<TResponseData>((acc, element) => {
    const elementValue = mergedValue[element.id];
    if (elementValue !== undefined) {
      acc[element.id] = elementValue;
    }
    return acc;
  }, {});

  // 调用 Survey 传入的 onChange 更新全局状态
  onChange(mergedValue);

  // 自动进度判断：对于单选题等，选择后自动提交
  if (
    shouldTriggerAutoProgress({
      changedElementId: elementId,
      mergedValue,
      autoProgressElement,
      isAlreadyInFlight: autoProgressingInFlightRef.current,
    })
  ) {
    autoProgressingInFlightRef.current = true;
    setTimeout(() => {
      try {
        const blockTtc = collectTtcValues();
        onSubmit(blockResponses, blockTtc);  // 自动提交整个 Block
      } finally {
        autoProgressingInFlightRef.current = false;
      }
    }, AUTO_PROGRESS_SUBMIT_DELAY_MS);
  }
};
```

用户点击提交按钮时的完整提交流程：

```typescript
// packages/surveys/src/components/general/block-conditional.tsx:310-349
const handleBlockSubmit = (e?: Event) => {
  if (e) {
    e.preventDefault();
  }

  // 步骤 1：集中验证所有题目
  const errorMap = validateBlockResponses(block.elements, value, languageCode);

  if (Object.keys(errorMap).length > 0) {
    setElementErrors(errorMap);
    // 滚动到第一个有错误的题目
    const firstErrorElementId = Object.keys(errorMap)[0];
    const form = elementFormRefs.current.get(firstErrorElementId);
    if (form) {
      const scrollTarget = form.querySelector("[data-element-input]") ?? form;
      scrollTarget.scrollIntoView({ behavior: "smooth", block: "center" });
    }
    return;
  }

  // 步骤 2：也对未迁移到集中验证的题目执行传统验证
  const firstInvalidForm = findFirstInvalidForm();
  if (firstInvalidForm) {
    const scrollTarget = firstInvalidForm.querySelector("[data-element-input]") ?? firstInvalidForm;
    scrollTarget.scrollIntoView({ behavior: "smooth", block: "center" });
    return;
  }

  // 步骤 3：清除所有错误
  setElementErrors({});

  // 步骤 4：收集 TTC 和响应数据
  const blockTtc = collectTtcValues();
  const blockResponses = collectBlockResponses();

  // 步骤 5：提交到 Survey 组件
  onSubmit(blockResponses, blockTtc);
};
```

#### 层级 4：Survey 组件的 onSubmit 处理

**文件路径**：`packages/surveys/src/components/general/survey.tsx`

Survey 组件作为顶层状态容器，处理全局提交逻辑：

```typescript
// packages/surveys/src/components/general/survey.tsx:622-630
// 全局 onChange：从 BlockConditional 接收，更新响应数据状态
const onChange = (responseDataUpdate: TResponseData) => {
  const updatedResponseData = { ...responseData, ...responseDataUpdate };
  setResponseData(updatedResponseData);
};
```

Block 提交的 onSubmit 处理：

```typescript
// packages/surveys/src/components/general/survey.tsx:919-979
const onSubmit = async (surveyResponseData: TResponseData, responsettc: TResponseTtc) => {
  isNavigatingBackRef.current = false;

  setLoadingElement(true);

  // 步骤 1：评估逻辑，计算下一个 Block
  const { nextBlockId, calculatedVariables } = evaluateLogicAndGetNextBlockId(surveyResponseData);
  const finished =
    nextBlockId === undefined || !localSurvey.blocks.map((block) => block.id).includes(nextBlockId);

  setIsSurveyFinished(finished);

  const endingId = nextBlockId
    ? localSurvey.endings.find((ending) => ending.id === nextBlockId)?.id
    : undefined;

  // 步骤 2：更新全局响应数据
  onChange(surveyResponseData);
  onChangeVariables(calculatedVariables);

  // 步骤 3：创建或更新响应（通过 ResponseQueue）
  onResponseCreateOrUpdate({
    data: surveyResponseData,
    ttc: responsettc,
    finished,
    variables: calculatedVariables,
    language: selectedLanguage,
    endingId,
  });

  // 步骤 4：导航到下一个 Block
  if (nextBlockId) {
    setBlockId(nextBlockId);
  } else if (finished) {
    const firstEndingId = localSurvey.endings[0]?.id as string | undefined;
    if (firstEndingId) {
      setBlockId(firstEndingId);
    } else {
      setBlockId("end");
    }
  }

  // 添加到历史记录
  const newHistory = [...history, blockId];
  setHistory(newHistory);

  // 离线支持：保存进度
  if (offlinePersistEnabled) {
    const newBlockId = finished ? endingId || localSurvey.endings[0]?.id || "end" : nextBlockId || blockId;
    void saveSurveyProgress({
      surveyId: survey.id,
      blockId: newBlockId,
      responseData: { ...responseData, ...surveyResponseData },
      ttc: { ...ttc, ...responsettc },
      currentVariables: calculatedVariables,
      history: newHistory,
      selectedLanguage,
      surveyStateSnapshot: {
        responseId: surveyState?.responseId ?? null,
        displayId: surveyState?.displayId ?? null,
        surveyId: survey.id,
        singleUseId: surveyState?.singleUseId ?? null,
        userId: surveyState?.userId ?? null,
        contactId: surveyState?.contactId ?? null,
        responseAcc: surveyState?.responseAcc ?? { finished: false, data: {}, ttc: {}, variables: {} },
      },
      updatedAt: Date.now(),
    });
  }

  setLoadingElement(false);
};
```

#### 层级 5：ResponseQueue 最终提交

**文件路径**：`packages/surveys/src/lib/response-queue.ts`

响应队列处理网络提交、重试和离线存储：

```typescript
// packages/surveys/src/components/general/survey.tsx:820-890
const onResponseCreateOrUpdate = useCallback(
  async (responseUpdate: TResponseUpdate) => {
    // 预览模式只触发回调不实际提交
    if (isPreviewMode) {
      onResponseCreated?.();
      if (responseUpdate.finished) {
        setIsResponseSendingFinished(true);
      }
      return;
    }

    if (surveyState && responseQueue) {
      if (contactId) {
        surveyState.updateContactId(contactId);
      }
      if (userId) {
        surveyState.updateUserId(userId);
      }

      responseQueue.updateSurveyState(surveyState);

      // 添加到响应队列
      responseQueue.add({
        data: responseUpdate.data,
        ttc: responseUpdate.ttc,
        finished: responseUpdate.finished,
        language:
          responseUpdate.language === "default"
            ? getDefaultLanguageCode(survey)
            : responseUpdate.language,
        meta: {
          ...getWebSurveyMeta(),
          action,
        },
        variables: responseUpdate.variables,
        displayId: surveyState.displayId,
        endingId: responseUpdate.endingId,
        hiddenFields: hiddenFieldsRecord,
      });

      onResponseCreated?.();
    }
  },
  [
    appUrl,
    environmentId,
    isPreviewMode,
    surveyState,
    responseQueue,
    onResponse,
    onResponseCreated,
    contactId,
    userId,
    survey,
    action,
    hiddenFieldsRecord,
    getWebSurveyMeta,
  ]
);
```

### 3.3 调用链可视化

```
用户交互（点击选项/输入文本）
    ↓
[题型组件 onChange]
    ↓  packages/surveys/src/components/elements/*.tsx
    ↓  例如：multiple-choice-multi-element.tsx:217-239
    ↓  调用：onChange({ [element.id]: value })
[ElementConditional 透传]
    ↓  packages/surveys/src/components/general/element-conditional.tsx
    ↓  接收：onChange prop 来自 BlockConditional
[BlockConditional handleElementChange]
    ↓  packages/surveys/src/components/general/block-conditional.tsx:91-136
    ↓  合并 Block 内所有答案 → onChange(mergedValue)
    ↓  自动进度判断 → 触发 onSubmit
[Survey 组件]
    ↓  packages/surveys/src/components/general/survey.tsx
    ↓  onChange: setResponseData(updatedResponseData) （第 622-630 行）
    ↓  onSubmit: 逻辑评估 → 导航 → onResponseCreateOrUpdate （第 919-979 行）
[ResponseQueue]
    ↓  packages/surveys/src/lib/response-queue.ts
    ↓  add(): 入队 → 网络提交 / 离线存储 → 重试机制
[API 服务器 / IndexedDB]
```

---

## 附录：关键文件索引

| 功能 | 文件路径 |
|------|---------|
| 题型枚举 | `packages/types/surveys/constants.ts` |
| 题型类型定义 | `packages/types/surveys/elements.ts` |
| 响应数据模型 | `packages/types/responses.ts` |
| 题型分发器 | `packages/surveys/src/components/general/element-conditional.tsx` |
| Block 级别提交管理 | `packages/surveys/src/components/general/block-conditional.tsx` |
| Survey 顶层状态管理 | `packages/surveys/src/components/general/survey.tsx` |
| 响应队列与网络提交 | `packages/surveys/src/lib/response-queue.ts` |
