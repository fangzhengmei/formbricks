# 问卷条件跳转（Logic Jump）实现机制分析报告

## 1. 存储结构

Formbricks 的条件跳转采用 **四层嵌套结构** 存储，数据以 JSON 格式存放在 Prisma 的 `Survey.blocks` 字段中。

### 1.1 数据库层

```prisma
// packages/database/schema.prisma:361
model Survey {
  // ...
  blocks  Json[]  @default([])  // 块数组，每个块包含逻辑配置
  endings Json[]  @default([])  // 结束卡，可作为跳转目标
  // ...
}
```

### 1.2 层级结构

```
Survey (问卷)
└── Block[] (块数组)
    ├── id: string                    // 块 ID (CUID)
    ├── name: string                  // 块名称
    ├── elements: SurveyElement[]     // 问题元素
    ├── logic: BlockLogic[]           // 逻辑规则数组 ← 核心
    └── logicFallback?: string        // 无匹配时的回退块 ID
```

### 1.3 BlockLogic 结构

每个块可以配置多条逻辑规则，每条规则定义了「条件 → 动作」的映射。

**位置**: `packages/types/surveys/blocks.ts:115-121`

```typescript
interface BlockLogic {
  id: string;
  conditions: ConditionGroup;  // 条件组（支持嵌套）
  actions: BlockLogicAction[]; // 动作数组
}
```

### 1.4 ConditionGroup 条件组

条件组支持 **嵌套** 和 **AND/OR** 连接符，形成灵活的条件表达式树。

**位置**: `packages/types/surveys/logic.ts:217-231`

```typescript
interface ConditionGroup {
  id: string;
  connector: "and" | "or";  // 连接符
  conditions: (SingleCondition | ConditionGroup)[];  // 支持嵌套
}
```

### 1.5 SingleCondition 单个条件

**位置**: `packages/types/surveys/logic.ts:138-175`

```typescript
interface SingleCondition {
  id: string;
  leftOperand: DynamicLogicFieldValue;  // 左操作数
  operator: LogicConditionsOperator;     // 比较操作符
  rightOperand?: RightOperand;          // 右操作数（部分操作符不需要）
}
```

#### 1.5.1 左操作数类型

左操作数支持三种数据源：

```typescript
// element: 问题/元素
{ type: "element", value: "questionId", meta?: { row: "0" } }

// variable: 变量
{ type: "variable", value: "variableId" }

// hiddenField: 隐藏字段
{ type: "hiddenField", value: "fieldId" }
```

#### 1.5.2 右操作数类型

```typescript
// static: 静态值
{ type: "static", value: "someValue" | 42 | ["opt1", "opt2"] }

// dynamic: 动态值（同左操作数）
{ type: "element", value: "anotherQuestionId" }
```

#### 1.5.3 支持的操作符

共 **38 种** 操作符，分为以下几类：

| 分类 | 操作符 | 说明 |
|------|--------|------|
| 相等 | `equals`, `doesNotEqual` | 等于/不等于 |
| 字符串 | `contains`, `doesNotContain`, `startsWith`, `doesNotStartWith`, `endsWith`, `doesNotEndWith` | 字符串包含/前缀/后缀 |
| 数值比较 | `isGreaterThan`, `isLessThan`, `isGreaterThanOrEqual`, `isLessThanOrEqual` | 数值大于/小于 |
| 日期比较 | `isAfter`, `isBefore` | 日期前后 |
| 数组包含 | `includesAllOf`, `includesOneOf`, `doesNotIncludeAllOf`, `doesNotIncludeOneOf` | 数组包含关系 |
| 状态检查 | `isSubmitted`, `isSkipped`, `isClicked`, `isNotClicked`, `isAccepted`, `isBooked`, `isPartiallySubmitted`, `isCompletelySubmitted` | 提交/跳过/点击状态 |
| 空值检查 | `isSet`, `isNotSet`, `isEmpty`, `isNotEmpty` | 是否设置/为空 |
| 多选 | `equalsOneOf`, `isAnyOf` | 等于多个值之一 |

**无需右操作数的操作符**：`isSubmitted`, `isSkipped`, `isClicked`, `isNotClicked`, `isAccepted`, `isBooked`, `isPartiallySubmitted`, `isCompletelySubmitted`, `isSet`, `isNotSet`, `isEmpty`, `isNotEmpty`

### 1.6 BlockLogicAction 动作

**位置**: `packages/types/surveys/blocks.ts:61-111`

动作分为三类：

#### 1.6.1 jumpToBlock（跳转）

```typescript
{
  id: string;
  objective: "jumpToBlock";
  target: string;  // 目标块 ID 或结束卡 ID
}
```

#### 1.6.2 requireAnswer（强制必填）

```typescript
{
  id: string;
  objective: "requireAnswer";
  target: string;  // 目标元素 ID
}
```

#### 1.6.3 calculate（变量计算）

```typescript
// 文本变量
{
  id: string;
  objective: "calculate";
  variableId: string;
  operator: "assign" | "concat";
  value: { type: "static", value: string } | DynamicLogicFieldValue;
}

// 数值变量
{
  id: string;
  objective: "calculate";
  variableId: string;
  operator: "add" | "subtract" | "multiply" | "divide" | "assign";
  value: { type: "static", value: number } | DynamicLogicFieldValue;
}
```

---

## 2. 运行时条件评估与跳转执行

### 2.1 入口函数

**位置**: `packages/surveys/src/components/general/survey.tsx:698-806`

`evaluateLogicAndGetNextBlockId` 是跳转逻辑的入口，在用户提交块时调用。

```typescript
const evaluateLogicAndGetNextBlockId = (
  data: TResponseData
): { nextBlockId: string | undefined; calculatedVariables: TResponseVariables } => {
  // 1. 合并当前响应数据
  const localResponseData = { ...responseData, ...data };
  let calculationResults = { ...currentVariables };

  // 2. 处理块级逻辑
  const evaluateBlockLogic = () => {
    let firstJumpTarget: string | undefined;
    const allRequiredQuestionIds: string[] = [];

    // 3. 遍历每条逻辑规则
    if (currentBlock.logic && currentBlock.logic.length > 0) {
      for (const logic of currentBlock.logic) {
        const result = processLogicRule(logic, firstJumpTarget, allRequiredQuestionIds);
        firstJumpTarget = result.jumpTarget;
        allRequiredQuestionIds.length = 0;
        allRequiredQuestionIds.push(...result.requiredIds);
        calculationResults = result.updatedCalculations;
      }
    }

    // 4. 使用 fallback（无匹配时）
    if (!firstJumpTarget && currentBlock.logicFallback) {
      firstJumpTarget = currentBlock.logicFallback;
    }

    return { firstJumpTarget, allRequiredQuestionIds };
  };

  // 5. 处理必填问题
  handleRequiredQuestions(allRequiredQuestionIds);

  // 6. 确定下一个块
  const nextBlockId = firstJumpTarget || localSurvey.blocks[currentBlockIndex + 1]?.id;
  return { nextBlockId, calculatedVariables: calculationResults };
};
```

### 2.2 评估顺序

**核心原则**：**按顺序评估，取第一个匹配的跳转目标**

```
流程：
┌─────────────────────────────────────────────────────────────┐
│ 1. 遍历 block.logic 数组（按配置顺序）                       │
│    ├── 对每条规则：                                          │
│    │   ├── 评估 conditions（递归遍历条件树）                 │
│    │   ├── 如果条件满足 → 执行 actions                        │
│    │   │   ├── 执行 calculate 动作（更新变量）               │
│    │   │   ├── 收集 requireAnswer 目标                       │
│    │   │   └── 收集 jumpToBlock 目标（只取第一个）           │
│    │   └── 如果条件不满足 → 跳过                              │
│    └── 继续下一条规则                                         │
│                                                              │
│ 2. 检查是否已有 jumpTarget                                    │
│    ├── 有 → 使用第一个 jumpTarget                            │
│    └── 无 → 检查 logicFallback                               │
│        ├── 有 → 使用 logicFallback                           │
│        └── 无 → 使用顺序下一个块                             │
└─────────────────────────────────────────────────────────────┘
```

### 2.3 条件评估实现

**位置**: `packages/surveys/src/lib/logic.ts:28-48`

```typescript
export const evaluateLogic = (
  localSurvey, data, variablesData, conditions, selectedLanguage
): boolean => {
  const evaluateConditionGroup = (group: ConditionGroup): boolean => {
    // 递归评估所有子条件
    const results = group.conditions.map((condition) => {
      if (isConditionGroup(condition)) {
        return evaluateConditionGroup(condition);  // 递归
      } else {
        return evaluateSingleCondition(...);  // 评估单个条件
      }
    });

    // AND: 所有为真；OR: 任一为真
    return group.connector === "or" ? results.some(r => r) : results.every(r => r);
  };

  return evaluateConditionGroup(conditions);
};
```

### 2.4 动作执行实现

**位置**: `packages/surveys/src/lib/logic.ts:50-82`

```typescript
export const performActions = (survey, actions, data, calculationResults) => {
  let jumpTarget: string | undefined;
  const requiredQuestionIds: string[] = [];
  const calculations = { ...calculationResults };

  actions.forEach((action) => {
    switch (action.objective) {
      case "calculate":
        const result = performCalculation(...);
        if (result !== undefined) calculations[action.variableId] = result;
        break;
      case "requireAnswer":
        requiredQuestionIds.push(action.target);
        break;
      case "jumpToBlock":
        if (!jumpTarget) {  // 只取第一个
          jumpTarget = action.target;
        }
        break;
    }
  });

  return { jumpTarget, requiredQuestionIds, calculations };
};
```

**关键特性**：
- **多条跳转动作只取第一个**（`if (!jumpTarget)`）
- **变量计算按顺序累积**
- **强制必填问题可累积多个**

### 2.5 单个条件评估

**位置**: `packages/surveys/src/lib/logic.ts:206-448`

`evaluateSingleCondition` 函数处理 38 种操作符的具体逻辑。

#### 2.5.1 左操作数取值

根据类型从不同数据源获取值：

```typescript
const getLeftOperandValue = (localSurvey, data, variablesData, leftOperand, selectedLanguage) => {
  switch (leftOperand.type) {
    case "element":
      // 从响应数据中获取，支持：
      // - 数值类型转换
      // - 多选题的选项标签 → ID 映射
      // - 矩阵题的行索引取值
      return data[leftOperand.value];
    case "variable":
      return getVariableValue(variables, leftOperand.value, variablesData);
    case "hiddenField":
      return data[leftOperand.value];
  }
};
```

#### 2.5.2 特殊处理场景

| 场景 | 处理方式 |
|------|----------|
| 多选题比较 | 标签文本自动映射为选项 ID |
| 矩阵题 | 通过 `meta.row` 索引获取指定行的值 |
| 日期比较 | 使用 `Date.getTime()` 比较时间戳 |
| 异常处理 | 任何错误返回 `false`（条件不满足） |

### 2.6 运行时回退逻辑详解

**位置**: `packages/surveys/src/components/general/survey.tsx:762-800`

```typescript
// 第一层级回退：逻辑规则匹配
let firstJumpTarget: string | undefined;

// 遍历所有逻辑规则
for (const logic of currentBlock.logic) {
  const result = processLogicRule(logic, firstJumpTarget, allRequiredQuestionIds);
  firstJumpTarget = result.jumpTarget;  // 只保留第一个匹配的
  // ...
}

// 第二层级回退：logicFallback
if (!firstJumpTarget && currentBlock.logicFallback) {
  firstJumpTarget = currentBlock.logicFallback;
}

// 第三层级回退：顺序下一个块
const nextBlockId = firstJumpTarget || localSurvey.blocks[currentBlockIndex + 1]?.id;
```

**回退优先级**（从高到低）：

```
1. 第一条满足条件的逻辑规则中的 jumpToBlock 动作
   └── 只取第一个满足条件的规则中的第一个跳转动作

2. block.logicFallback（如果配置了且无逻辑规则匹配）
   └── 适用于所有逻辑规则都不满足的场景

3. localSurvey.blocks[currentBlockIndex + 1]?.id（顺序下一个）
   └── 无任何跳转目标时的默认行为
   └── 如果是最后一个块，则为 undefined（问卷结束）
```

**实际运行时决策流程图**：

```
用户提交块
  ↓
评估逻辑规则
  ├── 规则1条件满足？
  │   └── 是 → 执行动作，取第一个 jumpToBlock → 跳转目标确定
  ├── 规则2条件满足？
  │   └── 是 → 执行动作（跳转目标已确定，忽略新的 jumpToBlock）
  ├── ...
  └── 所有规则都不满足？
      └── 是
          ↓
有配置 logicFallback？
  ├── 是 → 使用 logicFallback 作为跳转目标
  └── 否
        ↓
使用顺序下一个块
  ├── 存在 → 跳转到下一个块
  └── 不存在（最后一块）→ 问卷结束
```

---

## 3. 保存校验阶段的死跳防护

Formbricks 在问卷保存时通过多层校验拦截无效跳转配置，确保不会出现运行时死跳。

### 3.1 校验时机

**位置**: `packages/types/surveys/types.ts:3727-3747`

校验在 `ZSurvey.superRefine` 中执行，是问卷创建/更新的必经阶段。

```typescript
const validateBlockLogic = (
  survey: TSurvey,
  blockIndex: number,
  block: TSurveyBlock,
  allElements: Map<string, { block: number; element: number; data: TSurveyElement }>
): z.core.$ZodIssue[] => {
  // 1. 校验 logicFallback
  const logicFallbackIssue = validateBlockLogicFallback(survey, blockIndex, block);

  if (!block.logic || block.logic.length === 0) {
    return logicFallbackIssue ?? [];
  }

  // 2. 校验每条逻辑规则
  const logicIssues = block.logic.map((logicItem, logicIndex) => {
    return [
      ...validateBlockConditions(survey, blockIndex, logicIndex, logicItem.conditions, allElements),
      ...validateBlockActions(survey, blockIndex, logicIndex, logicItem.actions, block, allElements),
    ];
  });

  return [...logicIssues.flat(), ...(logicFallbackIssue ?? [])];
};
```

### 3.2 jumpToBlock 动作校验

**位置**: `packages/types/surveys/types.ts:3638-3660`

```typescript
// action.objective === "jumpToBlock"
const targetBlockId = action.target;
const blockIds = survey.blocks.map((b) => b.id);
const endingIds = survey.endings.map((ending) => ending.id);
const possibleTargets = [...blockIds, ...endingIds];

// 校验1：目标必须存在
if (!possibleTargets.includes(targetBlockId)) {
  return {
    code: "custom",
    message: `Conditional Logic: Block ID ${targetBlockId} does not exist in logic no: ${String(logicIndex + 1)} of block ${String(blockIndex + 1)}`,
    path: ["blocks", blockIndex, "logic", logicIndex],
  };
}

// 校验2：不能跳转到当前块
if (targetBlockId === currentBlock.id) {
  return {
    code: "custom",
    message: `Conditional Logic: Cannot jump to the current block in logic no: ${String(logicIndex + 1)} of block ${String(blockIndex + 1)}`,
    path: ["blocks", blockIndex, "logic", logicIndex],
  };
}
```

### 3.3 logicFallback 校验

**位置**: `packages/types/surveys/types.ts:3679-3725`

```typescript
const validateBlockLogicFallback = (
  survey: TSurvey,
  blockIndex: number,
  block: TSurveyBlock
): z.core.$ZodIssue[] | undefined => {
  if (!block.logicFallback) return;

  // 校验1：有 fallback 但无 logic 规则 → 无意义
  if (!block.logic?.length && block.logicFallback) {
    return [
      {
        code: "custom",
        message: `Conditional Logic: Fallback logic is defined without any logic in block ${String(blockIndex + 1)}`,
        path: ["blocks", blockIndex],
      },
    ];
  }

  // 校验2：fallback 不能指向当前块
  if (block.id === block.logicFallback) {
    return [
      {
        code: "custom",
        message: `Conditional Logic: Fallback logic is defined with the same block in block ${String(blockIndex + 1)}`,
        path: ["blocks", blockIndex],
      },
    ];
  }

  // 校验3：fallback 目标必须存在（其他块或结束卡）
  const possibleFallbackIds: string[] = [];
  survey.blocks.forEach((b, idx) => {
    if (idx !== blockIndex) {
      possibleFallbackIds.push(b.id);
    }
  });
  survey.endings.forEach((e) => {
    possibleFallbackIds.push(e.id);
  });

  if (!possibleFallbackIds.includes(block.logicFallback)) {
    return [
      {
        code: "custom",
        message: `Conditional Logic: Fallback block ID ${block.logicFallback} does not exist in block ${String(blockIndex + 1)}`,
        path: ["blocks", blockIndex],
      },
    ];
  }
};
```

### 3.4 其他动作校验

**位置**: `packages/types/surveys/types.ts:3666-3673`

```typescript
// 同一条逻辑规则中不能有多个 jumpToBlock 动作
const jumpToBlockActions = actions.filter((action) => action.objective === "jumpToBlock");
if (jumpToBlockActions.length > 1) {
  actionIssues.push({
    code: "custom",
    message: `Conditional Logic: Multiple jump actions are not allowed in logic no: ${String(logicIndex + 1)} of block ${String(blockIndex + 1)}`,
    path: ["blocks", blockIndex, "logic"],
  });
}
```

### 3.5 死跳防护校验清单

| 校验项 | 校验内容 | 错误消息 |
|--------|----------|----------|
| **jumpToBlock 目标无效** | target 不在 blocks + endings 中 | `Block ID ${target} does not exist` |
| **跳转到当前块** | target === 当前块 ID | `Cannot jump to the current block` |
| **fallback 无 logic** | 配置了 fallback 但无逻辑规则 | `Fallback logic is defined without any logic` |
| **fallback 指向当前块** | logicFallback === 当前块 ID | `Fallback logic is defined with the same block` |
| **fallback 目标无效** | logicFallback 不在其他块/结束卡中 | `Fallback block ID ${target} does not exist` |
| **多条跳转动作** | 同一条规则中有多个 jumpToBlock | `Multiple jump actions are not allowed` |

---

## 4. 循环/死跳防护机制

### 4.1 循环检测算法

Formbricks 使用 **DFS（深度优先搜索）** 算法在 **验证阶段** 检测循环。

**位置**: `packages/types/surveys/blocks-validation.ts:3-81`

```typescript
export const findBlocksWithCyclicLogic = (blocks: TSurveyBlock[]): string[] => {
  const visited: Record<string, boolean> = {};      // 已访问标记
  const recStack: Record<string, boolean> = {};     // 递归栈（检测回边）
  const cyclicBlocks = new Set<string>();

  const checkForCyclicLogic = (blockId: string): boolean => {
    if (!visited[blockId]) {
      visited[blockId] = true;
      recStack[blockId] = true;

      const block = blocks.find((b) => b.id === blockId);

      // 路径1：检查 logic 中的 jumpToBlock 动作
      if (block?.logic && block.logic.length > 0) {
        for (const logic of block.logic) {
          const jumpActions = findJumpToBlockActions(logic.actions);
          for (const jumpAction of jumpActions) {
            const destination = jumpAction.target;

            // 跳过非块目标（结束卡）
            if (!blocks.find((b) => b.id === destination)) continue;

            if (!visited[destination] && checkForCyclicLogic(destination)) {
              cyclicBlocks.add(blockId);
              recStack[blockId] = false;
              return true;
            } else if (recStack[destination]) {
              // 发现回边 → 存在循环
              cyclicBlocks.add(blockId);
              recStack[blockId] = false;
              return true;
            }
          }
        }
      }

      // 路径2：检查 logicFallback
      if (block?.logicFallback) {
        const fallbackBlockId = block.logicFallback;
        if (blocks.find((b) => b.id === fallbackBlockId)) {
          // 同上，递归检测
        }
      }

      // 路径3：检查默认顺序（下一个块）
      const nextBlockIndex = blocks.findIndex((b) => b.id === blockId) + 1;
      const nextBlock = blocks[nextBlockIndex];
      if (nextBlock) {
        // 同上，递归检测
      }
    }

    recStack[blockId] = false;
    return false;
  };

  // 对每个块启动检测
  for (const block of blocks) {
    checkForCyclicLogic(block.id);
  }

  return Array.from(cyclicBlocks);
};
```

### 4.2 检测的三种路径

循环检测考虑 **所有可能的跳转路径**：

```
可能的跳转路径：
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│  Block A                                                     │
│  ├── logic[0].actions.jumpToBlock → Block X                  │
│  ├── logic[1].actions.jumpToBlock → Block Y                  │
│  ├── logicFallback → Block Z                                 │
│  └── 顺序下一个 → Block B                                     │
│                                                              │
│  所有这四条路径都需要检测是否形成循环                           │
└─────────────────────────────────────────────────────────────┘
```

### 4.3 结束卡豁免

**结束卡（Ending Card）不会形成循环**，因此检测时会跳过：

```typescript
// 跳过非块目标（结束卡）
if (!blocks.find((b) => b.id === destination)) {
  continue;
}
```

### 4.4 校验触发

**位置**: `packages/types/surveys/types.ts:3733-3746`

循环检测在问卷保存时执行，发现循环则返回错误。

---

## 5. 校验顺序与互斥关系

### 5.1 校验函数执行顺序

**位置**: `packages/types/surveys/types.ts:3727-3747`

校验函数按以下顺序执行：

```
validateBlockLogic(survey, blockIndex, block, allElements)
  │
  ├── 第一步：validateBlockLogicFallback(...)  ← fallback 校验
  │     │
  │     ├── 校验1：有 fallback 但无 logic 规则 → 直接返回
  │     ├── 校验2：fallback 指向当前块 → 直接返回
  │     └── 校验3：fallback 目标不存在 → 直接返回
  │
  └── 第二步：遍历 block.logic，对每条规则执行：
        │
        ├── validateBlockConditions(...)  ← 条件校验
        │
        └── validateBlockActions(...)  ← 动作校验
              │
              ├── 遍历每个 action：
              │     │
              │     ├── calculate 类型校验
              │     ├── requireAnswer 类型校验
              │     └── jumpToBlock 类型校验
              │           │
              │           ├── 校验A：目标不存在 → 直接返回（后续校验不再执行）
              │           └── 校验B：目标是当前块 → 直接返回（前提：先通过校验A）
              │
              └── 最后：统计 jumpToBlock 数量
                    └── 校验C：多条跳转动作 → 追加到错误列表
```

### 5.2 互斥关系说明

**关键理解**：单个校验函数内部是**按顺序 `return`**，不是累积所有错误。

#### 5.2.1 fallback 校验的互斥关系

`validateBlockLogicFallback` 内部是按顺序 `return`：

```typescript
// 校验1：有 fallback 但无 logic
if (!block.logic?.length && block.logicFallback) {
  return [/* 错误1 */];  // ← 直接返回，校验2、3不执行
}

// 校验2：fallback 指向当前块
if (block.id === block.logicFallback) {
  return [/* 错误2 */];  // ← 直接返回，校验3不执行
}

// 校验3：fallback 目标不存在
if (!possibleFallbackIds.includes(block.logicFallback)) {
  return [/* 错误3 */];
}
```

**互斥关系**：
| 校验 | 条件 | 与其他校验的关系 |
|------|------|------------------|
| 1: fallback 无 logic | `logic: []` 且有 `logicFallback` | **独占**，触发后校验2、3不执行 |
| 2: fallback 指向当前块 | `logicFallback === 当前块ID` | 前提：有 logic 规则；**与校验3互斥** |
| 3: fallback 目标无效 | `logicFallback` 不在合法列表中 | 前提：有 logic 规则，且不是当前块 |

#### 5.2.2 jumpToBlock 动作校验的互斥关系

`validateBlockActions` 中对单个 `jumpToBlock` action 的校验：

```typescript
// 校验A：目标不存在
if (!possibleTargets.includes(targetBlockId)) {
  return {/* 错误A */};  // ← 直接返回，校验B不执行
}

// 校验B：跳转到当前块
if (targetBlockId === currentBlock.id) {
  return {/* 错误B */};
}
```

**互斥关系**：
| 校验 | 条件 | 与其他校验的关系 |
|------|------|------------------|
| A: jump 目标无效 | target 不在 blocks + endings 中 | **独占**，触发后校验B不执行 |
| B: 跳转到当前块 | target === 当前块ID | 前提：目标存在（校验A通过） |

#### 5.2.3 多条跳转动作校验

"多条跳转动作"（校验C）是在遍历完所有 action 后**单独检查**的：

```typescript
// 在 forEach 之后执行
const jumpToBlockActions = actions.filter((action) => action.objective === "jumpToBlock");
if (jumpToBlockActions.length > 1) {
  actionIssues.push({/* 错误C */});  // ← 追加，不是 return
}
```

这意味着**校验C可以与其他错误同时出现**：
- 示例：两个 jumpToBlock 动作，其中一个目标不存在 → 会同时触发"目标无效" + "多条跳转动作"

---

## 6. 完整示例

### 6.1 合法配置示例

```json
{
  "id": "survey_legal",
  "name": "学生身份调查",
  "blocks": [
    {
      "id": "block1",
      "name": "身份确认",
      "elements": [
        {
          "id": "q1",
          "type": "multipleChoiceSingle",
          "headline": { "default": "你是学生吗？" },
          "required": true,
          "choices": [
            { "id": "yes", "label": { "default": "是" } },
            { "id": "no", "label": { "default": "不是" } }
          ]
        }
      ],
      "logic": [
        {
          "id": "logic1",
          "conditions": {
            "id": "group1",
            "connector": "and",
            "conditions": [
              {
                "id": "cond1",
                "leftOperand": { "type": "element", "value": "q1" },
                "operator": "equals",
                "rightOperand": { "type": "static", "value": "yes" }
              }
            ]
          },
          "actions": [
            {
              "id": "act1",
              "objective": "jumpToBlock",
              "target": "block2"
            }
          ]
        }
      ],
      "logicFallback": "block3"
    },
    {
      "id": "block2",
      "name": "学生优惠",
      "elements": [
        {
          "id": "q2",
          "type": "openText",
          "headline": { "default": "请输入你的学生证号" },
          "required": true,
          "inputType": "text"
        }
      ]
    },
    {
      "id": "block3",
      "name": "普通用户",
      "elements": [
        {
          "id": "q3",
          "type": "openText",
          "headline": { "default": "请输入你的邮箱" },
          "required": true,
          "inputType": "email"
        }
      ]
    }
  ],
  "endings": [
    {
      "id": "ending1",
      "headline": { "default": "感谢参与！" },
      "type": "endScreen"
    }
  ]
}
```

**为什么合法**：
- 所有 `jumpToBlock.target`（block2、block3）都存在
- `logicFallback`（block3）存在且不是当前块
- 无循环（block1 → block2/block3 → 结束，都是单向向前）

### 6.2 非法配置最小示例（每条错误独立触发）

以下每个示例**只触发一个特定错误**，用于演示校验逻辑。

#### 6.2.1 示例1：jumpToBlock 目标无效

```json
{
  "id": "err_jump_target_not_exist",
  "name": "跳转目标不存在",
  "blocks": [
    {
      "id": "block1",
      "name": "块1",
      "elements": [
        {
          "id": "q1",
          "type": "openText",
          "headline": { "default": "问题1" },
          "required": true,
          "inputType": "text"
        }
      ],
      "logic": [
        {
          "id": "logic1",
          "conditions": {
            "id": "group1",
            "connector": "and",
            "conditions": [
              {
                "id": "cond1",
                "leftOperand": { "type": "element", "value": "q1" },
                "operator": "isNotEmpty",
                "rightOperand": { "type": "static", "value": "" }
              }
            ]
          },
          "actions": [
            {
              "id": "act1",
              "objective": "jumpToBlock",
              "target": "nonExistentBlock"
            }
          ]
        }
      ]
    }
  ],
  "endings": []
}
```

**触发的错误**：
| 校验项 | 错误消息 |
|--------|----------|
| jumpToBlock 目标无效 | `Block ID nonExistentBlock does not exist in logic no: 1 of block 1` |

**说明**：`nonExistentBlock` 不在 `blocks` 或 `endings` 中。

---

#### 6.2.2 示例2：跳转到当前块

```json
{
  "id": "err_jump_to_self",
  "name": "跳转到当前块",
  "blocks": [
    {
      "id": "block1",
      "name": "块1",
      "elements": [
        {
          "id": "q1",
          "type": "openText",
          "headline": { "default": "问题1" },
          "required": true,
          "inputType": "text"
        }
      ],
      "logic": [
        {
          "id": "logic1",
          "conditions": {
            "id": "group1",
            "connector": "and",
            "conditions": [
              {
                "id": "cond1",
                "leftOperand": { "type": "element", "value": "q1" },
                "operator": "isNotEmpty",
                "rightOperand": { "type": "static", "value": "" }
              }
            ]
          },
          "actions": [
            {
              "id": "act1",
              "objective": "jumpToBlock",
              "target": "block1"
            }
          ]
        }
      ]
    }
  ],
  "endings": []
}
```

**触发的错误**：
| 校验项 | 错误消息 |
|--------|----------|
| 跳转到当前块 | `Cannot jump to the current block in logic no: 1 of block 1` |

**说明**：`target = "block1"` 与当前块 ID 相同。

---

#### 6.2.3 示例3：多条跳转动作

```json
{
  "id": "err_multiple_jumps",
  "name": "多条跳转动作",
  "blocks": [
    {
      "id": "block1",
      "name": "块1",
      "elements": [
        {
          "id": "q1",
          "type": "openText",
          "headline": { "default": "问题1" },
          "required": true,
          "inputType": "text"
        }
      ],
      "logic": [
        {
          "id": "logic1",
          "conditions": {
            "id": "group1",
            "connector": "and",
            "conditions": [
              {
                "id": "cond1",
                "leftOperand": { "type": "element", "value": "q1" },
                "operator": "isNotEmpty",
                "rightOperand": { "type": "static", "value": "" }
              }
            ]
          },
          "actions": [
            {
              "id": "act1",
              "objective": "jumpToBlock",
              "target": "block2"
            },
            {
              "id": "act2",
              "objective": "jumpToBlock",
              "target": "ending1"
            }
          ]
        }
      ]
    },
    {
      "id": "block2",
      "name": "块2",
      "elements": []
    }
  ],
  "endings": [
    {
      "id": "ending1",
      "headline": { "default": "结束" },
      "type": "endScreen"
    }
  ]
}
```

**触发的错误**：
| 校验项 | 错误消息 |
|--------|----------|
| 多条跳转动作 | `Multiple jump actions are not allowed in logic no: 1 of block 1` |

**说明**：同一条逻辑规则中定义了 2 个 `jumpToBlock` 动作。

---

#### 6.2.4 示例4：fallback 无 logic

```json
{
  "id": "err_fallback_no_logic",
  "name": "有fallback但无logic规则",
  "blocks": [
    {
      "id": "block1",
      "name": "块1",
      "elements": [
        {
          "id": "q1",
          "type": "openText",
          "headline": { "default": "问题1" },
          "required": true,
          "inputType": "text"
        }
      ],
      "logic": [],
      "logicFallback": "ending1"
    }
  ],
  "endings": [
    {
      "id": "ending1",
      "headline": { "default": "结束" },
      "type": "endScreen"
    }
  ]
}
```

**触发的错误**：
| 校验项 | 错误消息 |
|--------|----------|
| fallback 无 logic | `Fallback logic is defined without any logic in block 1` |

**说明**：`logic: []` 但配置了 `logicFallback`，fallback 无意义。

---

#### 6.2.5 示例5：fallback 指向当前块

```json
{
  "id": "err_fallback_to_self",
  "name": "fallback指向当前块",
  "blocks": [
    {
      "id": "block1",
      "name": "块1",
      "elements": [
        {
          "id": "q1",
          "type": "openText",
          "headline": { "default": "问题1" },
          "required": true,
          "inputType": "text"
        }
      ],
      "logic": [
        {
          "id": "logic1",
          "conditions": {
            "id": "group1",
            "connector": "and",
            "conditions": [
              {
                "id": "cond1",
                "leftOperand": { "type": "element", "value": "q1" },
                "operator": "equals",
                "rightOperand": { "type": "static", "value": "yes" }
              }
            ]
          },
          "actions": [
            {
              "id": "act1",
              "objective": "jumpToBlock",
              "target": "block2"
            }
          ]
        }
      ],
      "logicFallback": "block1"
    },
    {
      "id": "block2",
      "name": "块2",
      "elements": []
    }
  ],
  "endings": []
}
```

**触发的错误**：
| 校验项 | 错误消息 |
|--------|----------|
| fallback 指向当前块 | `Fallback logic is defined with the same block in block 1` |

**说明**：`logicFallback = "block1"` 与当前块 ID 相同。

---

#### 6.2.6 示例6：fallback 目标无效

```json
{
  "id": "err_fallback_not_exist",
  "name": "fallback目标不存在",
  "blocks": [
    {
      "id": "block1",
      "name": "块1",
      "elements": [
        {
          "id": "q1",
          "type": "openText",
          "headline": { "default": "问题1" },
          "required": true,
          "inputType": "text"
        }
      ],
      "logic": [
        {
          "id": "logic1",
          "conditions": {
            "id": "group1",
            "connector": "and",
            "conditions": [
              {
                "id": "cond1",
                "leftOperand": { "type": "element", "value": "q1" },
                "operator": "equals",
                "rightOperand": { "type": "static", "value": "yes" }
              }
            ]
          },
          "actions": [
            {
              "id": "act1",
              "objective": "jumpToBlock",
              "target": "block2"
            }
          ]
        }
      ],
      "logicFallback": "nonExistentBlock"
    },
    {
      "id": "block2",
      "name": "块2",
      "elements": []
    }
  ],
  "endings": []
}
```

**触发的错误**：
| 校验项 | 错误消息 |
|--------|----------|
| fallback 目标无效 | `Fallback block ID nonExistentBlock does not exist in block 1` |

**说明**：`nonExistentBlock` 不在其他块或结束卡中。

---

### 6.3 错误触发条件汇总

| 错误类型 | 触发条件 | 互斥情况 |
|----------|----------|----------|
| jumpToBlock 目标无效 | target 不在 blocks + endings 中 | 独占，触发后"跳转到当前块"不检查 |
| 跳转到当前块 | target === 当前块ID | 前提：目标存在 |
| 多条跳转动作 | 同一条规则中 >1 个 jumpToBlock | 可与其他错误同时触发 |
| fallback 无 logic | logic 为空但有 logicFallback | 独占，触发后其他 fallback 校验不执行 |
| fallback 指向当前块 | logicFallback === 当前块ID | 前提：有 logic；与"fallback 目标无效"互斥 |
| fallback 目标无效 | logicFallback 不在合法列表中 | 前提：有 logic 且不是当前块 |

---

## 7. 关键文件索引

| 功能 | 文件路径 |
|------|----------|
| 类型定义 | `packages/types/surveys/logic.ts` |
| 块类型定义 | `packages/types/surveys/blocks.ts` |
| 保存校验（核心） | `packages/types/surveys/types.ts:3679-3747` |
| 运行时评估 | `packages/surveys/src/lib/logic.ts` |
| 问卷组件（入口） | `packages/surveys/src/components/general/survey.tsx:698-806` |
| 循环检测 | `packages/types/surveys/blocks-validation.ts` |
| 数据库 Schema | `packages/database/schema.prisma` |
| 单元测试 | `packages/surveys/src/lib/logic.test.ts` |
